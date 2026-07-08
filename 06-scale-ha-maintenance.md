# Module 6: Global Scale, High Availability & Maintenance

[← Back to Index](../README.md) | [← Module 5](05-devsecops-security.md)

---

## 6.1 High Availability — Active-Active Clustering

A production Artifactory HA cluster runs multiple **active** nodes behind a load balancer, all sharing the same database and filestore (typically object storage, so any node can serve any request):

```
                     ┌───────────────┐
                     │ Load Balancer  │
                     └───────┬───────┘
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        ┌───────────┐  ┌───────────┐  ┌───────────┐
        │  Node A    │  │  Node B    │  │  Node C    │   (all active)
        └─────┬─────┘  └─────┬─────┘  └─────┬─────┘
              └──────────────┼──────────────┘
                              ▼
                 Shared PostgreSQL + Shared Filestore (S3)
```

If Node A fails, the load balancer routes to B/C with zero data loss, since state lives in the shared DB/filestore, not on individual nodes. This is only possible *because* of the metadata/binary split covered in Module 2 — no node holds unique local state.

**Minimal `system.yaml` HA snippet** (illustrative):

```yaml
shared:
  database:
    type: postgresql
    driver: org.postgresql.Driver
    url: "jdbc:postgresql://db.internal:5432/artifactory"
    username: artifactory
    password: "${DB_PASSWORD}"
  node:
    id: "node-a"
    ip: "10.0.1.11"
    haEnabled: true
  extraJavaOpts: "-Xms2g -Xmx8g"
```

Each node references the same shared `database` block but has a unique `node.id`/`node.ip` — this is the config-level expression of "shared state, independent compute."

## 6.2 Multi-Site Topology: Federated Repositories vs. Replication

| | **Federated Repositories** | **Replication (Push/Pull)** |
|---|---|---|
| Sync direction | Bi-directional, near-real-time mesh | Configurable (one-way or scheduled) |
| Use case | Global dev teams needing local-speed reads/writes everywhere | DR (disaster recovery), or selective mirroring |
| Conflict handling | Automatic, event-driven propagation | Scheduled batch sync, less real-time |
| Failure mode | A regional outage doesn't stop other regions from writing | A replication target being down just delays sync |

```
   US Site (local-us)  ⇄  EU Site (local-eu)  ⇄  APAC Site (local-apac)
      (Federated: every push in one region propagates to all others
       automatically, over an optimized binary-diff protocol)
```

**Example: configuring a Federated repository** (via REST API, simplified):

```bash
curl -X PUT "http://us-artifactory:8081/artifactory/api/repositories/npm-federated-local" \
  -u admin:password \
  -H "Content-Type: application/json" \
  -d '{
        "rclass": "federated",
        "packageType": "npm",
        "key": "npm-federated-local",
        "members": [
          { "url": "http://eu-artifactory:8081/artifactory/npm-federated-local", "enabled": true },
          { "url": "http://apac-artifactory:8081/artifactory/npm-federated-local", "enabled": true }
        ]
      }'
```

**Example: configuring one-way Replication** (Push, used for DR):

```bash
curl -X PUT "http://localhost:8081/artifactory/api/repositories/npm-dev-local/replicate" \
  -u admin:password \
  -H "Content-Type: application/json" \
  -d '{
        "url": "http://dr-site:8081/artifactory/npm-dev-local",
        "cronExp": "0 0 2 * * ?",
        "enabled": true,
        "syncDeletes": true,
        "syncProperties": true
      }'
```

## 6.3 Distribution & Edge Nodes

- **JFrog Distribution**: securely broadcasts *Release Bundles* (not loose artifacts, see Module 4) to remote datacenters, retail POS systems, or IoT fleets.
- **Edge Nodes**: lightweight, typically read-only Artifactory instances deployed inside the consuming environment (e.g., a regional Kubernetes cluster). They pull from the central Artifactory once, then serve locally — avoiding repeated cross-region bandwidth costs every time a pod pulls an image.

```
   Central Artifactory (source of truth)
            │
            ▼  Distribution broadcasts signed Release Bundle
   ┌────────┴────────┬────────────────┐
   ▼                 ▼                ▼
 Edge Node (US-K8s) Edge Node (EU-K8s) Edge Node (Factory IoT)
   │                 │                │
   ▼                 ▼                ▼
 Pods pull locally  Pods pull locally  Devices pull locally
 (no repeated       (no repeated       (works even with
  cross-region        cross-region       intermittent WAN)
  bandwidth)          bandwidth)
```

## 6.4 Artifactory Query Language (AQL)

AQL is a JSON-like query DSL for searching artifacts by metadata, properties, checksums, dates, and more — far more expressive than simple path-based browsing.

**Example 1 — find all artifacts older than 30 days in a repo that were never promoted:**

```
items.find({
  "repo": "npm-dev-local",
  "created": {"$before": "30d"},
  "@release.status": {"$ne": "promoted"}
}).include("name", "repo", "path", "created")
```

**Example 2 — find all Docker images containing a specific vulnerable base layer checksum:**

```
items.find({
  "repo": {"$eq": "docker-prod-local"},
  "actual_sha256": {"$eq": "a1b2c3d4e5f6..."}
}).include("name", "repo", "path")
```

**Example 3 — find artifacts by custom property (e.g., team ownership tag):**

```
items.find({
  "repo": "maven-local",
  "@team": {"$eq": "payments"}
}).include("name", "path", "property.*")
```

## 6.5 Hands-On Lab: AQL-Driven Cleanup Cron Job

### Step 1 — AQL query script (`find-stale.aql`)

```
items.find(
  {
    "repo": "npm-dev-local",
    "created": {"$before": "30d"},
    "@build.status": {"$ne": "promoted"}
  }
).include("name","repo","path")
```

### Step 2 — Bash wrapper to run the query and delete matches

```bash
#!/usr/bin/env bash
set -euo pipefail

ARTIFACTORY_URL="http://localhost:8081/artifactory"
AUTH="admin:password"

echo "Querying stale, unpromoted artifacts older than 30 days..."

RESULTS=$(curl -s -X POST "${ARTIFACTORY_URL}/api/search/aql" \
  -u "${AUTH}" \
  -H "Content-Type: text/plain" \
  --data-binary @find-stale.aql)

echo "$RESULTS" | jq -r '.results[] | "\(.repo)/\(.path)/\(.name)"' | while read -r ARTIFACT_PATH; do
  echo "Deleting: ${ARTIFACT_PATH}"
  curl -s -X DELETE "${ARTIFACTORY_URL}/${ARTIFACT_PATH}" -u "${AUTH}"
done

echo "Cleanup complete."
```

### Step 3 — Schedule via cron

```bash
# Run every day at 02:00 AM
0 2 * * * /opt/scripts/cleanup-stale-artifacts.sh >> /var/log/artifactory-cleanup.log 2>&1
```

### Step 4 — Reinforce with Garbage Collection

Deleting via API moves items to Artifactory's internal trash can (not immediately freed on disk). Trigger GC to reclaim space immediately:

```bash
curl -X POST "http://localhost:8081/artifactory/api/system/storage" \
  -u admin:password
```

Or lower trash-can retention under *Administration → General → Trash Can Settings* (e.g., from 14 days to 2 days) for faster reclamation long-term.

## 6.6 Best Practices & Interview Nuances

- Prefer **Federated Repositories** for active multi-region development teams; prefer **Replication** for DR/backup scenarios where near-real-time bi-directional sync isn't required.
- Externalize the filestore to S3/GCS/Azure Blob before scaling to HA — local disk doesn't support multi-node shared access.
- Automate retention policies (AQL + cron, or JFrog Pipelines) — manual cleanup does not scale and is the #1 cause of "Artifactory ran out of disk" incidents.
- Use **Edge Nodes** anywhere network latency or intermittent connectivity would otherwise slow down or block deployments (retail, factory floor IoT, disconnected environments).
- Interview trap: *"Why does deleting an artifact not immediately free disk space?"* → Because of checksum-based storage (Module 2): the physical blob may still be referenced by other logical paths, so Artifactory only removes the DB pointer immediately and defers physical GC until it's safe (no remaining references) or the trash-can retention expires.
- Interview trap: *"How would you diagnose sudden replication lag between US and EU sites?"* → Check network throughput/latency between sites first, then check the replication/federation queue depth in the Artifactory UI, then check for large binary uploads saturating the sync channel.

---

[← Module 5: DevSecOps & Security](05-devsecops-security.md) | [Back to Index](../README.md)
