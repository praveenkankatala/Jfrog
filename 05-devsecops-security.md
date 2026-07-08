# Module 5: DevSecOps & Software Supply Chain Security

[← Back to Index](../README.md) | [← Module 4](04-promotion-release-lifecycle.md) | [Module 6 →](06-scale-ha-maintenance.md)

---

## 5.1 Shift-Left: Component Analysis vs. Runtime Scanning

Traditional security scans code or containers at **runtime** — after the software is already deployed. **Component analysis** (Software Composition Analysis, SCA) inspects every dependency **at build/publish time**, catching known-vulnerable or license-incompatible components before they ever reach a running environment.

```
Traditional (runtime-only):
  [Build] → [Deploy] → [Runtime Scanner finds CVE] → Incident response, hot patch under pressure

Shift-Left (component analysis at publish time):
  [Build] → [Xray scans BOM at publish] → [Blocked before it ever reaches prod] → No incident
```

Shifting left doesn't replace runtime monitoring — it dramatically reduces the volume and severity of issues that ever reach that stage.

## 5.2 JFrog Xray — Deep Dive

Xray indexes every artifact stored in Artifactory **recursively**, including nested dependencies inside a Docker image layer or a fat JAR, and cross-references each component against continuously updated vulnerability intelligence (NVD, VulnDB, and Xray's own research feed).

**Key capabilities:**

- **Vulnerability scanning** — CVE detection with CVSS severity scoring, down to the individual transitive dependency.
- **License compliance** — flags copyleft (e.g., GPL) or otherwise disallowed licenses per your organization's policy.
- **Impact analysis graph** — shows exactly which of *your* deployed services/builds are affected when a new CVE is published for a shared dependency (critical during incidents like Log4Shell, where knowing "which of our 400 services use log4j-core 2.14.1" in seconds rather than days was the difference between a controlled response and chaos).
- **Watches & Policies** — declarative rules like "block any artifact with CVSS ≥ 8.5 from being downloaded," or "warn but don't block for CVSS 4.0–7.9."

```
   Artifact uploaded → Xray indexes recursively (full BOM, see Module 2)
                              │
                 ┌────────────┴────────────┐
                 ▼                         ▼
         Vulnerability DB match     License compliance check
                 │                         │
                 └────────────┬────────────┘
                              ▼
                    Policy evaluation
                 (block / warn / notify)
```

**Re-scanning is continuous, not one-time.** When a new CVE is published for a component already stored in Artifactory, Xray retroactively flags every existing artifact containing that component — you don't need to re-upload anything to discover you're newly exposed.

## 5.3 JFrog Curation — Perimeter Defense

Xray protects artifacts *already inside* your Artifactory instance. **Curation** sits one layer earlier — at the remote repository boundary — intercepting a developer's or CI pipeline's request for a public package **before** it's even cached, blocking known-malicious or typosquatted packages (e.g., a package named `reqeusts` impersonating `requests`) from ever reaching your network.

| Layer | Tool | When it acts |
|---|---|---|
| Perimeter (before entry) | JFrog Curation | At request time, against public registries, before caching |
| Post-ingestion | JFrog Xray | After the artifact is already stored in Artifactory |

Curation policies commonly block on:
- Packages published very recently with no established reputation ("cool-down" period rules).
- Known typosquatting patterns against popular package names.
- Packages with zero or single maintainers and no prior version history.
- Explicit deny-lists maintained by your security team.

## 5.4 Hands-On Lab: Security Policy Blocking on CVSS ≥ 8.5

### Step 1 — Create a Watch on your Docker local repo

```bash
curl -X POST "http://localhost:8081/xray/api/v2/watches" \
  -u admin:password \
  -H "Content-Type: application/json" \
  -d '{
        "general_data": { "name": "prod-docker-watch", "description": "Blocks critical CVEs" },
        "project_resources": {
          "resources": [
            { "type": "repository", "name": "docker-prod-local" }
          ]
        }
      }'
```

### Step 2 — Create a Security Policy with a CVSS threshold rule

```bash
curl -X POST "http://localhost:8081/xray/api/v2/policies" \
  -u admin:password \
  -H "Content-Type: application/json" \
  -d '{
        "name": "block-critical-cve",
        "type": "security",
        "rules": [
          {
            "name": "block-cvss-8.5",
            "criteria": { "cvss_range": { "from": 8.5, "to": 10 } },
            "actions": { "block_download": { "active": true, "unscanned": true } }
          }
        ]
      }'
```

### Step 3 — Add a License Compliance Rule (bonus)

```bash
curl -X POST "http://localhost:8081/xray/api/v2/policies" \
  -u admin:password \
  -H "Content-Type: application/json" \
  -d '{
        "name": "block-gpl-license",
        "type": "license",
        "rules": [
          {
            "name": "no-gpl",
            "criteria": {
              "banned_licenses": ["GPL-2.0", "GPL-3.0", "AGPL-3.0"],
              "allow_unknown": false
            },
            "actions": { "block_download": { "active": true } }
          }
        ]
      }'
```

### Step 4 — Attach the policies to the watch

Via UI: *Xray → Watches → prod-docker-watch → Assign Policies → select `block-critical-cve` and `block-gpl-license`*.

### Step 5 — Wire it into Jenkins

With the `rtXrayScan (failOnScanFailure: true)` step already present in the Module 3 Jenkinsfile, any image pulling in a component with CVSS ≥ 8.5 or a banned license will now hard-fail the pipeline at the scan stage — before it ever reaches `docker-prod-local`, and long before it reaches a Kubernetes cluster.

### Step 6 — Verify Enforcement

Attempt to pull a known-vulnerable base image and confirm the block:

```bash
docker pull localhost:8081/docker-prod-local/vulnerable-test-image:latest
# Expected: 403 Forbidden, with an Xray policy violation message in the response body
```

## 5.5 Best Practices & Interview Nuances

- **Scan early and often** — in CI (blocking, preventative) *and* continuously on already-stored artifacts (Xray re-scans retroactively when new CVEs are published, catching artifacts that were "clean" at publish time but aren't anymore).
- Curation policies should be tuned per-ecosystem — npm and PyPI see far more typosquatting/supply-chain attacks than, say, Maven Central, which has stricter publishing controls.
- Start security policies in **"warn" mode** before flipping to **"block"** — this avoids surprise pipeline breakage across dozens of teams on day one, while still surfacing the data needed to prioritize remediation.
- Interview trap: *"If Xray already scans everything, why do we need Curation?"* → Xray only sees what's already inside Artifactory; Curation prevents malicious packages from being cached/ingested in the first place, closing the window between "published maliciously upstream" and "detected by your scanner."
- Interview trap: *"How does Xray handle a CVE published for a component after the artifact was already deployed to production?"* → Xray's index is continuously updated; it re-evaluates stored artifacts against new CVE data automatically and will flag/alert on previously-clean artifacts — you don't need to manually re-trigger a scan.

---

[← Module 4: Artifact Promotion](04-promotion-release-lifecycle.md) | [Module 6: Global Scale, HA & Maintenance →](06-scale-ha-maintenance.md)
