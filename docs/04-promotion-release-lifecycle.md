# Module 4: Artifact Promotion & Advanced Release Lifecycles

[← Back to Index](../README.md) | [← Module 3](03-cicd-jenkins.md) | [Module 5 →](05-devsecops-security.md)

---

## 4.1 Why Rebuilding Per Environment Is an Anti-Pattern

Rebuilding the same source code separately for dev, QA, and prod risks:

- A transitive dependency resolving to a **different (newer) version** between builds ("dependency drift") — e.g., a `^1.2.0` semver range in `package.json` silently pulls `1.2.5` today and `1.3.0` next week.
- Subtle environment differences (a different Node/JDK patch version on the build agent) producing a **functionally different binary** despite identical source.
- Loss of a single point of truth for "what was actually tested" — if QA tested one binary and a *different* rebuilt binary ships to prod, your test coverage guarantees nothing about what's actually live.

The fix is the **"Build Once, Deploy Anywhere"** principle: compile exactly once, then move that exact binary through environments by changing its *metadata status*, not its bytes.

## 4.2 Metadata-Based Promotion (0ms Because No Bytes Move)

```
   dev-local                qa-local               prod-local
 ┌───────────┐  promote  ┌───────────┐  promote  ┌───────────┐
 │ app-1.0.42 │ ────────► │ app-1.0.42 │ ────────► │ app-1.0.42 │
 └───────────┘           └───────────┘           └───────────┘
   (same physical checksum blob referenced at every stage —
    only the DB path/status pointer changes — see Module 2)
```

This works because of the checksum-based storage model from Module 2 — the promotion API call simply tells Artifactory "create a new logical path in `qa-local` pointing at the same checksum currently in `dev-local`," which is a database write, not a network transfer. This is why promotion between repos on the *same* Artifactory instance completes in milliseconds regardless of artifact size.

## 4.3 Release Bundles (v2) — Immutable, Signed, Multi-Artifact Units

A single release is often *multiple* artifacts together — e.g., a backend Docker image, a frontend static bundle, and a Helm chart. A **Release Bundle** packages these into one signed, versioned, immutable unit:

- Once created, a Release Bundle's contents **cannot be edited, added to, or removed from** — this is what distinguishes it from a mutable repository folder.
- It's cryptographically signed, so downstream consumers (Distribution, Edge Nodes) can verify authenticity and detect tampering.
- It's the unit that gets distributed to remote sites/edge/IoT devices — not individual loose artifacts, which would risk drift between components of the same release.

```
   ┌────────────────────────────────────┐
   │           RELEASE BUNDLE v2.3.0     │
   │  (signed, immutable, versioned)     │
   │                                     │
   │   - backend-service:2.3.0 (docker)  │
   │   - frontend-app-2.3.0.zip          │
   │   - helm-chart-2.3.0.tgz            │
   └────────────────────────────────────┘
                    │
                    ▼
      JFrog Distribution → Edge Nodes (regional/K8s/IoT)
```

### Release Bundle vs. Standard Repository — Key Differences

| | Standard Repository | Release Bundle v2 |
|---|---|---|
| Mutability | Mutable — files can be added/overwritten | Immutable once created |
| Scope | Single artifact / package | Multi-artifact, versioned unit |
| Signing | Not inherent | Cryptographically signed |
| Distribution target | Not distributed as a unit | Designed for Distribution → Edge Nodes |
| Use case | Day-to-day storage & CI publishing | Formal release, multi-region rollout |

## 4.4 Hands-On Lab: Scripted Promotion Gate

### Bash Version

```bash
#!/usr/bin/env bash
set -euo pipefail

BUILD_NAME="my-app"
BUILD_NUMBER="42"
SOURCE_REPO="npm-dev-local"
TARGET_REPO="npm-qa-local"
ARTIFACTORY_URL="http://localhost:8081/artifactory"
AUTH="admin:password"

# --- Mock Quality Gate ---
COVERAGE=$(cat coverage-report.txt | grep -oP '\d+(?=%)')
echo "Test coverage: ${COVERAGE}%"

if [ "$COVERAGE" -lt 80 ]; then
  echo "❌ Coverage gate failed (${COVERAGE}% < 80%). Promotion blocked."
  exit 1
fi

echo "✅ Coverage gate passed. Promoting build..."

curl -s -X POST \
  "${ARTIFACTORY_URL}/api/build/promote/${BUILD_NAME}/${BUILD_NUMBER}" \
  -u "${AUTH}" \
  -H "Content-Type: application/json" \
  -d '{
        "status": "qa-approved",
        "comment": "Promoted after passing automated coverage gate",
        "ci_user": "jenkins",
        "sourceRepo": "'"${SOURCE_REPO}"'",
        "targetRepo": "'"${TARGET_REPO}"'",
        "copy": false
      }'

echo "Promotion request submitted for ${BUILD_NAME} #${BUILD_NUMBER}."
```

### Python Version (using `requests`)

```python
#!/usr/bin/env python3
"""
Promote a build in JFrog Artifactory only if a mock quality gate passes.
Usage: python promote.py --coverage-file coverage-report.txt
"""
import argparse
import re
import sys
import requests

ARTIFACTORY_URL = "http://localhost:8081/artifactory"
AUTH = ("admin", "password")

BUILD_NAME = "my-app"
BUILD_NUMBER = "42"
SOURCE_REPO = "npm-dev-local"
TARGET_REPO = "npm-qa-local"
COVERAGE_THRESHOLD = 80


def read_coverage(path: str) -> int:
    with open(path) as f:
        content = f.read()
    match = re.search(r"(\d+)%", content)
    if not match:
        raise ValueError("Could not parse coverage percentage from report")
    return int(match.group(1))


def promote_build():
    url = f"{ARTIFACTORY_URL}/api/build/promote/{BUILD_NAME}/{BUILD_NUMBER}"
    payload = {
        "status": "qa-approved",
        "comment": "Promoted after passing automated coverage gate",
        "ci_user": "jenkins",
        "sourceRepo": SOURCE_REPO,
        "targetRepo": TARGET_REPO,
        "copy": False,
    }
    resp = requests.post(url, json=payload, auth=AUTH)
    resp.raise_for_status()
    print(f"Promotion request submitted for {BUILD_NAME} #{BUILD_NUMBER}.")
    print(resp.json())


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--coverage-file", required=True)
    args = parser.parse_args()

    coverage = read_coverage(args.coverage_file)
    print(f"Test coverage: {coverage}%")

    if coverage < COVERAGE_THRESHOLD:
        print(f"❌ Coverage gate failed ({coverage}% < {COVERAGE_THRESHOLD}%). Promotion blocked.")
        sys.exit(1)

    print("✅ Coverage gate passed. Promoting build...")
    promote_build()


if __name__ == "__main__":
    main()
```

### Equivalent Using JFrog CLI Directly

```bash
jf rt build-promote my-app 42 npm-qa-local \
  --status=qa-approved \
  --comment="Promoted after passing coverage gate" \
  --source-repo=npm-dev-local
```

## 4.5 Best Practices & Interview Nuances

- Prefer `"copy": false` (move via new pointer) over physically copying when promoting between local repos of the *same* storage backend — it's what makes promotion instantaneous.
- Use **immutable, semantic version tags** for anything promoted to `prod-local` — never promote a mutable `latest` tag, since a future overwrite of `latest` would silently break traceability back to a specific Build Info record.
- Gate every promotion behind an automated check (test coverage, Xray scan result, manual QA sign-off recorded via API) rather than manual drag-and-drop in the UI, so the promotion history itself becomes part of the audit trail.
- Interview trap: *"What's the difference between repository promotion and Release Bundle promotion?"* → Repository promotion moves a single build's artifacts between repo tiers; Release Bundle promotion moves a signed, multi-artifact, versioned unit through a distribution lifecycle — often across organizational or geographic boundaries.
- Interview trap: *"Can promotion fail halfway and leave inconsistent state?"* → Because it's a single atomic database transaction per artifact (not a byte-copy process), promotion of an individual artifact either fully succeeds or fully fails — there's no "half-copied" binary state to worry about.

---

[← Module 3: CI/CD Integration](03-cicd-jenkins.md) | [Module 5: DevSecOps & Security →](05-devsecops-security.md)
