# JFrog Artifactory & Platform — Zero to Advanced Enterprise Mastery

A complete, practical, production-grade learning path covering JFrog Artifactory and the wider JFrog Platform: repository architecture, storage internals, CI/CD integration, release governance, DevSecOps, and global scale operations.

> Use this as a personal reference, a study guide for interviews, or a portfolio piece demonstrating hands-on binary/DevOps management skills.

Each module lives in its own file under [`docs/`](docs/) so it's easy to navigate, link to individually, and extend over time.

---

## Modules

| # | Module | Covers |
|---|---|---|
| 1 | [Foundational Architecture & The Repository Ecosystem](docs/01-foundational-architecture.md) | Why Git isn't enough, Local/Remote/Virtual repos, universal package support, Docker + npm/PyPI lab |
| 2 | [Enterprise Storage & Metadata Management](docs/02-storage-metadata-bom.md) | Checksum-based storage, DB/filestore split, Bill of Materials (BOM), Build Info & traceability, JFrog CLI lab |
| 3 | [Enterprise CI/CD Integration (Jenkins-focused)](docs/03-cicd-jenkins.md) | JFrog CLI + declarative pipelines, "Build Once, Deploy Anywhere," full Jenkinsfile (npm/Maven/Docker) |
| 4 | [Artifact Promotion & Advanced Release Lifecycles](docs/04-promotion-release-lifecycle.md) | Metadata-based promotion, Release Bundles v2, scripted quality-gated promotion (Bash + Python) |
| 5 | [DevSecOps & Software Supply Chain Security](docs/05-devsecops-security.md) | Shift-left security, JFrog Xray, JFrog Curation, CVSS-based blocking policy lab |
| 6 | [Global Scale, High Availability & Maintenance](docs/06-scale-ha-maintenance.md) | HA clustering, Federated Repos vs. Replication, Distribution & Edge Nodes, AQL, automated cleanup lab |

---

## Quick-Reference Cheat Sheet

| Concept | One-Line Summary |
|---|---|
| Local repo | Your own built artifacts |
| Remote repo | Caching proxy of a public registry |
| Virtual repo | Single URL aggregating local + remote |
| Checksum storage | Same file stored once, referenced by many paths |
| BOM | Full structured inventory of every component in a build |
| Build Info | Immutable JSON ledger of how a build was made (BOM + provenance) |
| Promotion | Metadata pointer update, not a byte copy |
| Release Bundle | Immutable, signed, multi-artifact release unit |
| Xray | Post-ingestion vulnerability & license scanning |
| Curation | Perimeter blocking of malicious packages before ingestion |
| Federated repos | Real-time bi-directional multi-site sync |
| Edge Nodes | Read-only local caches inside remote runtime environments |
| AQL | Query language for artifact metadata search & automation |

---

## Suggested Portfolio Projects

1. **Automated Project Onboarding** — Terraform/Python script that provisions Local/Remote/Virtual repos + RBAC groups for a new team via the Artifactory REST API.
2. **Secure Docker DevSecOps Pipeline** — Jenkins + Xray policy that blocks Kubernetes deployment of any image with CVSS ≥ 8.5.
3. **Multi-Region Federated Repo Demo** — Two Artifactory instances (Docker Compose) configured as federated mirrors, demonstrating bi-directional sync.

---

## Repository Structure

```
.
├── README.md                              ← this file (index)
└── docs/
    ├── 01-foundational-architecture.md
    ├── 02-storage-metadata-bom.md
    ├── 03-cicd-jenkins.md
    ├── 04-promotion-release-lifecycle.md
    ├── 05-devsecops-security.md
    └── 06-scale-ha-maintenance.md
```

*Structured for direct use as a GitHub repository. Contributions and corrections welcome via PR.*
