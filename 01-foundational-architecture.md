# Module 1: Foundational Architecture & The Repository Ecosystem

[← Back to Index](../README.md) | [Module 2 →](02-storage-metadata-bom.md)

---

## 1.1 The "Why" — The Problem Artifactory Solves

Git is a **version control system for source code** — text files, diffs, and history. It is not designed to store, index, or serve large compiled binaries (`.jar`, `.war`, Docker layers, `.whl`, `.zip`) efficiently. Attempting to do so causes:

- **Repository bloat** — binary diffs can't be compressed the way text diffs can, so `.git` history balloons.
- **No dependency resolution** — Git has no concept of "give me version 2.3.1 of this package and its transitive dependencies."
- **No promotion semantics** — Git branches model *code lineage*, not *release maturity* (dev → QA → prod).
- **No governance layer** — no built-in vulnerability scanning, license compliance, or immutability guarantees for a deployable unit.

Artifactory exists to be the **single source of truth for what was actually built**, as opposed to Git, which is the source of truth for *what was intended to be built*.

| Concern | Git | Artifactory |
|---|---|---|
| Stores | Source code, text diffs | Compiled binaries, packages, containers, ML models |
| Unit of truth | Commit / branch | Immutable artifact + Build Info |
| Answers | "What changed in the code?" | "What is actually running in production, and how was it built?" |
| Dependency resolution | None | Native (npm, Maven, PyPI, Go, etc.) |
| Promotion model | Merge / branch | Metadata-based promotion (no rebuild) |

## 1.2 The Triad: Local, Remote, and Virtual Repositories

```
                     ┌─────────────────────────────────────────┐
                     │            VIRTUAL REPOSITORY             │
                     │   npm-virtual  (single endpoint used by   │
                     │        developers & CI pipelines)         │
                     └───────────────────┬───────────────────────┘
                                         │  resolves requests against
                     ┌───────────────────┼────────────────────────┐
                     ▼                                            ▼
        ┌───────────────────────┐                    ┌───────────────────────┐
        │   LOCAL REPOSITORY     │                    │  REMOTE REPOSITORY    │
        │   npm-dev-local        │                    │  npm-remote           │
        │   (your own builds,    │                    │  (caching proxy of    │
        │   internal packages)   │                    │   registry.npmjs.org) │
        └───────────────────────┘                    └───────────────────────┘
```

**Local Repository**
- Physically hosted on your Artifactory instance.
- Holds artifacts *your organization* produces: internal libraries, proprietary Docker images, compiled services.
- Fully private by default; access governed by permission targets.

**Remote Repository**
- Not a copy of the internet — it's a **caching proxy**. The first request for a package is fetched from the real upstream (Maven Central, npm registry, Docker Hub, PyPI) and cached locally.
- Subsequent requests are served from cache — fast, and resilient if the upstream is down or a package is yanked (this actually happened industry-wide with the `left-pad` npm incident in 2016 — teams with a caching proxy were unaffected).

**Virtual Repository**
- A logical aggregation layer. It does not store anything itself — it's a resolution rule that says "check local-repo-A first, then remote-repo-B, then remote-repo-C."
- This is the **only** endpoint developers and CI tools should ever be configured to point at. It decouples client configuration from backend repository topology — admins can add/remove/reorder backend repos without any client reconfiguration.

## 1.3 Universal Package Ecosystem Support

Artifactory is **package-type aware**, not just a dumb file store. For each package type (Docker, npm, Maven, PyPI, Go, Helm, NuGet, Conan, Cargo, Hugging Face/GGUF model weights, and 50+ others) it understands the package's metadata format, index files, and protocol semantics — so tools like `npm install`, `docker pull`, or `pip install` work exactly as if talking to the public registry itself.

## 1.4 Hands-On Lab: Local Docker Setup + npm Virtual Repository Pattern

### Step 1 — Run Artifactory OSS/Pro locally via Docker

```bash
docker run --name artifactory -d -p 8081:8081 -p 8082:8082 \
  releases-docker.jfrog.io/jfrog/artifactory-oss:latest
```

- Port `8081`: Artifactory UI/API
- Port `8082`: Router service (required in newer versions)

Wait ~60 seconds, then browse to `http://localhost:8081` and complete the onboarding wizard (create admin user, accept EULA).

### Step 2 — Create the Remote Repository (proxy to npm registry)

Via UI: *Administration → Repositories → Remote → New Remote Repository*

Or via REST API:

```bash
curl -X PUT "http://localhost:8081/artifactory/api/repositories/npm-remote" \
  -u admin:password \
  -H "Content-Type: application/json" \
  -d '{
        "rclass": "remote",
        "packageType": "npm",
        "url": "https://registry.npmjs.org",
        "key": "npm-remote"
      }'
```

### Step 3 — Create the Local Repository

```bash
curl -X PUT "http://localhost:8081/artifactory/api/repositories/npm-dev-local" \
  -u admin:password \
  -H "Content-Type: application/json" \
  -d '{
        "rclass": "local",
        "packageType": "npm",
        "key": "npm-dev-local"
      }'
```

### Step 4 — Create the Virtual Repository (aggregates both)

```bash
curl -X PUT "http://localhost:8081/artifactory/api/repositories/npm-virtual" \
  -u admin:password \
  -H "Content-Type: application/json" \
  -d '{
        "rclass": "virtual",
        "packageType": "npm",
        "repositories": ["npm-dev-local", "npm-remote"],
        "key": "npm-virtual"
      }'
```

### Step 5 — Point your local npm client at the virtual repo

`.npmrc` (project-level or `~/.npmrc`):

```ini
registry=http://localhost:8081/artifactory/api/npm/npm-virtual/
always-auth=true
//localhost:8081/artifactory/api/npm/npm-virtual/:_authToken=<YOUR_IDENTITY_TOKEN>
```

Then run:

```bash
npm install express
```

Check the Artifactory UI — `express` and its transitive dependencies now appear cached inside `npm-remote`, resolved transparently through `npm-virtual`.

### Equivalent for Python (`pip.conf`)

```ini
[global]
index-url = https://<user>:<identity-token>@localhost:8081/artifactory/api/pypi/pypi-virtual/simple
```

### Equivalent for Maven (`settings.xml`)

```xml
<settings>
  <servers>
    <server>
      <id>central</id>
      <username>admin</username>
      <password>${env.ARTIFACTORY_TOKEN}</password>
    </server>
  </servers>
  <mirrors>
    <mirror>
      <id>central</id>
      <mirrorOf>*</mirrorOf>
      <url>http://localhost:8081/artifactory/maven-virtual</url>
    </mirror>
  </mirrors>
</settings>
```

## 1.5 Best Practices & Interview Nuances

- **Never point clients directly at local or remote repos** — always go through virtual. This is the single most-tested interview concept.
- Repository keys should be **lowercase, hyphen-separated**, no underscores or special characters (avoids URL-encoding issues at CDN/edge layers).
- One virtual repo per package ecosystem per environment tier is a common pattern (e.g., `npm-virtual`, `maven-virtual`, `docker-virtual`).
- Remote repositories should have **sensible cache retention** — don't cache forever if upstream artifacts are mutable (e.g., `latest` Docker tags).
- Interview trap: *"What happens if the public registry deletes a package your remote repo already cached?"* → Nothing — Artifactory continues serving the cached copy, which is exactly the resilience benefit of the caching-proxy model.

---

[Module 2: Enterprise Storage & Metadata Management →](02-storage-metadata-bom.md)
