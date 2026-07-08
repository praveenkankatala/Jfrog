# Module 2: Enterprise Storage & Metadata Management

[← Back to Index](../README.md) | [← Module 1](01-foundational-architecture.md) | [Module 3 →](03-cicd-jenkins.md)

---

## 2.1 Checksum-Based Storage (Deduplication)

Artifactory never stores a file according to its logical repository path. Every uploaded binary is hashed (SHA-1 **and** SHA-256, and MD5 for legacy compatibility), and the **physical object on disk is named after its checksum**, not its filename or folder location.

```
Logical view (what users/CI see):
  npm-dev-local/lodash/-/lodash-4.17.21.tgz
  npm-team2-local/vendor/lodash-4.17.21.tgz
  docker-local/base-images/node:18-alpine (shared base layer)

Physical view (what's actually on disk / object storage):
  filestore/
    ab/
      ab34f9c2...   ← the actual 200 KB binary, stored ONCE

Database (simplified):
  path_a  → sha256:ab34f9c2...
  path_b  → sha256:ab34f9c2...   (different logical location, same blob)
```

**Why this matters in production:**
- If 50 teams upload the identical 100 MB dependency, physical storage consumption is **100 MB, not 5 GB**.
- Docker image layers are checksum-addressed too — a common base layer (`node:18-alpine`) shared across 200 images is stored once, not 200 times.
- **Deletion is deferred and reference-counted.** Deleting a logical path only removes the database pointer immediately. The physical blob is only eligible for removal once *no other path references that checksum* — and even then it goes to a trash can before final Garbage Collection.
- **Copies between local repos are instant** — a "copy" operation just creates a new DB pointer to the existing checksum; no bytes move across the network or disk. This is also the foundation of instantaneous artifact promotion (see Module 4).

## 2.2 The Metadata / Binary Split — Database vs. Filestore

Artifactory's architecture cleanly separates **what things are** (metadata) from **the bytes themselves** (binary content):

```
┌───────────────────────────────┐        ┌───────────────────────────────┐
│      RELATIONAL DATABASE       │        │        BINARY FILESTORE        │
│   (PostgreSQL / Oracle / MySQL)│        │  (Local disk / S3 / GCS /      │
│                                │        │       Azure Blob Storage)      │
│  • Repository configurations   │        │                                │
│  • Users, groups, permissions  │        │  • Actual binary content,      │
│  • Logical folder hierarchy    │◄──────►│    addressed by SHA-256        │
│  • Build Info (JSON documents) │  maps  │    checksum (flat namespace,   │
│  • Package index metadata      │  paths │    no folder hierarchy)        │
│    (npm package.json, Maven    │  to    │                                │
│    POM data, Docker manifests) │  hashes│  • Durable, horizontally       │
│  • Properties / custom tags    │        │    scalable when backed by     │
│  • Audit logs, access logs     │        │    an object store             │
└───────────────────────────────┘        └───────────────────────────────┘
```

**Practical implications of this split:**

| Because of the split... | You get... |
|---|---|
| Logical hierarchy lives only in the DB | Promotion between repos = a DB write, not a byte copy (0ms operation) |
| Filestore is flat and checksum-addressed | You can swap the storage backend (local disk → S3) without touching a single logical path |
| DB is small relative to filestore | DB backups can run frequently (minutes); filestore relies on the durability of the underlying object store (e.g., S3's 11 nines) |
| Metadata and binaries scale independently | A DB read replica can serve UI/API load while filestore scales via object storage throughput, with no coupling |

### The Binary Provider Chain

In production, Artifactory typically layers multiple binary providers for caching and resilience:

```
Request for artifact
        │
        ▼
  ┌──────────────┐
  │  Cache Layer  │  (in-memory / local SSD cache of hot objects)
  └──────┬───────┘
         │ miss
         ▼
  ┌──────────────┐
  │  Eventual /    │  (local disk buffer, smooths bursty writes
  │  Filesystem    │   before they land in the object store)
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │  S3 / GCS /    │  (durable, long-term binary storage)
  │  Azure Blob    │
  └──────────────┘
```

This is configured in `binarystore.xml` on the Artifactory server — production deployments almost always terminate in an object store rather than raw local disk, since local disk doesn't support the shared access required by HA clusters (see Module 6).

## 2.3 The Bill of Materials (BOM)

The term **Bill of Materials** in the JFrog context refers to the **complete, structured inventory of every component that makes up a built artifact** — direct dependencies, transitive dependencies, their exact versions, checksums, and licenses. It is effectively the manufacturing parts-list for software.

```
my-app-1.0.42.jar
│
├── BOM (Bill of Materials)
│   ├── express@4.18.2        (direct dependency)
│   │     └── body-parser@1.20.1   (transitive)
│   │            └── bytes@3.1.2   (transitive, 2 levels deep)
│   ├── lodash@4.17.21        (direct dependency)
│   └── ... (every package pulled in at build time)
```

A BOM is normally expressed in an industry-standard format such as **CycloneDX** or **SPDX**, both of which Artifactory/Xray can generate and consume. The BOM is what makes component-level vulnerability scanning possible — Xray doesn't scan the whole compiled binary as a black box, it scans **every entry in the BOM individually**, including nested dependencies inside a Docker image or a fat JAR.

**Why the BOM matters operationally:**
- **Incident response** — when a new CVE drops for `xyz@2.1.0`, you query "which of our BOMs include xyz@2.1.0?" and instantly know every affected artifact/service, without re-scanning anything.
- **License compliance** — the BOM lists every component's declared license, letting you enforce policy (e.g., "no GPL-licensed code in commercial products") automatically.
- **Regulatory / supply-chain requirements** — frameworks like the U.S. Executive Order on cybersecurity and various SBOM (Software Bill of Materials) mandates require exactly this artifact-level inventory to be produced and retained.

Artifactory's Build Info (below) effectively *is* the BOM, wrapped with additional build provenance context (who built it, from what commit, when).

## 2.4 Build Info & Traceability — The Legal Ledger

**Build Info** is a structured JSON document — published to Artifactory by the JFrog CLI or a CI plugin — that captures the **complete provenance** of a single build execution:

```json
{
  "version": "1.0.1",
  "name": "my-app",
  "number": "42",
  "type": "GENERIC",
  "started": "2026-07-08T10:15:00.000+0000",
  "buildAgent": { "name": "Jenkins", "version": "2.462" },
  "vcs": [
    {
      "revision": "a1b2c3d4e5f6789",
      "url": "https://github.com/org/my-app.git",
      "branch": "main"
    }
  ],
  "modules": [
    {
      "id": "my-app:1.0.42",
      "artifacts": [
        {
          "type": "tgz",
          "sha1": "3f786850e387550fdab836ed7e6dc881de23001b",
          "sha256": "a1b2c3...",
          "md5": "d41d8cd98f00b204e9800998ecf8427e",
          "name": "my-app-1.0.42.tgz"
        }
      ],
      "dependencies": [
        {
          "id": "express:4.18.2",
          "type": "npm",
          "scopes": ["production"],
          "sha1": "..."
        },
        {
          "id": "lodash:4.17.21",
          "type": "npm",
          "scopes": ["production"],
          "sha1": "..."
        }
      ]
    }
  ],
  "properties": {
    "buildAgent": "Jenkins",
    "ciUrl": "https://jenkins.internal/job/my-app/42/",
    "triggeredBy": "jane.doe"
  }
}
```

**Why "legal ledger" is the right mental model:** once published, Build Info is treated as **immutable** — it cannot be silently edited after the fact. This gives you:

1. **Exact reproducibility** — given any production binary, you can trace back to the *exact* git commit, CI job, and full dependency tree that produced it — down to the SHA-1 of every transitive dependency.
2. **Non-repudiation** — because it records `triggeredBy`, timestamps, and the CI URL, Build Info functions as an audit trail suitable for compliance/security review, similar to a chain-of-custody record.
3. **Forensic incident response** — "Is `log4j-core:2.14.1` anywhere in our production estate?" becomes a single query against Build Info/BOM records across every build ever published, instead of re-scanning every running service.
4. **Diff-ability** — you can diff Build Info between two release numbers to see *exactly* what dependency versions changed between releases, which is invaluable for regression triage.

## 2.5 Hands-On Lab: Publish a Package + Full Build Info via JFrog CLI

### Step 1 — Install the JFrog CLI

```bash
curl -fL https://install-cli.jfrog.io | sh
jf --version
```

### Step 2 — Configure the CLI against your server

```bash
jf config add my-server \
  --artifactory-url=http://localhost:8081/artifactory \
  --user=admin \
  --password=password \
  --interactive=false
```

### Step 3 — Create a dummy package and upload it under a Build Name/Number

```bash
mkdir -p /tmp/my-app && cd /tmp/my-app
echo '{"name":"my-app","version":"1.0.42"}' > package.json
tar -czf my-app-1.0.42.tgz package.json

jf rt upload my-app-1.0.42.tgz npm-dev-local/my-app/1.0.42/ \
  --build-name=my-app --build-number=42
```

### Step 4 — Collect environment variables into Build Info

```bash
jf rt build-collect-env my-app 42
```

This captures the current shell's environment variables (CI agent name, JAVA_HOME, PATH, etc.) as part of the build context — useful for diagnosing "works on my machine" discrepancies later.

### Step 5 — Attach VCS/git metadata automatically

```bash
jf rt build-add-git my-app 42
```

This reads the local `.git` directory and injects the current commit SHA, branch, and remote URL into Build Info — the critical link back to source.

### Step 6 — (Optional) Manually append custom dependency/module data

For ecosystems the CLI doesn't auto-detect, you can inject a manual Build Info fragment:

```bash
cat > partial-build-info.json << 'EOF'
{
  "modules": [
    {
      "id": "my-app:1.0.42",
      "dependencies": [
        { "id": "custom-lib:2.0.0", "type": "generic", "sha1": "deadbeef..." }
      ]
    }
  ]
}
EOF

jf rt build-append my-app 42 partial-build-info.json
```

### Step 7 — Publish the completed Build Info to Artifactory

```bash
jf rt build-publish my-app 42
```

### Step 8 — Verify

Navigate to *Artifactory UI → Builds → my-app → #42*. You should see:
- The uploaded artifact with its checksums.
- The full dependency list.
- The linked git commit and branch.
- Captured environment variables.

You can also fetch it directly via API to confirm the full JSON:

```bash
curl -s -u admin:password \
  "http://localhost:8081/artifactory/api/build/my-app/42" | jq .
```

## 2.6 Best Practices & Interview Nuances

- **Publish Build Info on every CI run**, not just tagged releases — it's your audit trail and the backbone of impact analysis during security incidents.
- **Never manually delete files directly from the filestore.** Because of checksum-based deduplication, a physical blob may be referenced by paths you don't know about — always delete via the Artifactory API/UI so reference counts stay consistent.
- **Externalize the filestore to S3/GCS/Azure Blob for any production deployment.** Local disk doesn't support HA clustering and complicates backup/restore.
- **Retain Build Info at least as long as your compliance window requires** — some regulated industries mandate multi-year retention of build provenance.
- Interview trap: *"Why is promotion instant?"* → Because of the metadata/binary split: promoting an artifact is a database pointer update referencing an existing checksum in the filestore — no bytes are copied.
- Interview trap: *"What's the difference between Build Info and a BOM?"* → A BOM is the component inventory (what's inside); Build Info is the BOM **plus** provenance metadata (who/when/how it was built, git commit, CI job) — Build Info is a superset that makes the BOM traceable back to source.

---

[← Module 1: Foundational Architecture](01-foundational-architecture.md) | [Module 3: CI/CD Integration →](03-cicd-jenkins.md)
