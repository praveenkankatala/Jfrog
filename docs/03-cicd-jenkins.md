# Module 3: Enterprise CI/CD Integration (Jenkins-focused)

[← Back to Index](../README.md) | [← Module 2](02-storage-metadata-bom.md) | [Module 4 →](04-promotion-release-lifecycle.md)

---

## 3.1 The Industry Standard: JFrog CLI Inside Declarative Pipelines

Modern Jenkins + Artifactory integration favors the **JFrog CLI**, wrapped by the **JFrog Jenkins Plugin's** pipeline steps (`rtServer`, `rtNpmInstall`, `rtMavenRun`, `rtDockerPush`, `rtXrayScan`, etc.), over legacy freestyle job configuration. This keeps all pipeline logic version-controlled inside a `Jenkinsfile` rather than buried in opaque UI settings that are hard to audit or replicate across jobs.

Advantages of this approach:
- **Portable** — the same `Jenkinsfile` logic works whether the agent is a Docker container, a VM, or an ephemeral Kubernetes pod.
- **Consistent Build Info generation** — every `rt*` step automatically contributes to a shared, structured Build Info object for the job.
- **Ecosystem-agnostic core pattern** — the same authenticate → resolve → build → publish → scan → ledger flow applies whether you're building npm, Maven, Gradle, Go, or Docker artifacts; only the specific `rt*` step names change.

## 3.2 Pipeline Architecture — "Build Once, Deploy Anywhere"

```
[Git Commit] → [Jenkins Triggered]
       │
       ├─ 1. rtServer: authenticate to Artifactory
       ├─ 2. Resolve deps through Virtual Repo (npm-virtual / maven-virtual)
       ├─ 3. Compile / package the binary  (ONE TIME ONLY)
       ├─ 4. Push binary → dev-local repo
       ├─ 5. rtCollectEnv + rtPublishBuildInfo → ledger recorded
       ├─ 6. rtXrayScan → security/license gate
       │
       ▼
  [Same immutable binary promoted: dev-local → qa-local → prod-local]
       (no recompilation at any stage — only metadata changes, see Module 4)
```

The core discipline this enforces: **the artifact that gets security-tested, QA-tested, and eventually deployed to production is byte-for-byte identical** to what was compiled in step 3. Nothing downstream ever triggers a second compile.

## 3.3 Hands-On Lab: Production-Grade Jenkinsfile (npm example)

```groovy
pipeline {
    agent any

    environment {
        SERVER_ID    = 'my-artifactory-server'
        VIRTUAL_REPO = 'npm-virtual'
        LOCAL_REPO   = 'npm-dev-local'
        BUILD_NAME   = "${env.JOB_NAME}"
        BUILD_NUMBER = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Initialize JFrog') {
            steps {
                // Establishes authenticated handshake with the Artifactory server
                // defined in Jenkins Global Tool Configuration.
                rtServer (
                    id: "${SERVER_ID}",
                    url: 'https://your-company.jfrog.io/artifactory',
                    credentialsId: 'jfrog-credentials-id'
                )
            }
        }

        stage('Checkout Code') {
            steps {
                cleanWs()
                checkout scm
            }
        }

        stage('Configure & Install') {
            steps {
                // All npm installs will resolve packages through the virtual repo,
                // which transparently blends internal + public dependencies.
                rtNpmResolver (
                    id: "${SERVER_ID}",
                    repo: "${VIRTUAL_REPO}"
                )
                rtNpmInstall (
                    id: "${SERVER_ID}",
                    args: '--verbose'
                )
            }
        }

        stage('Test & Build') {
            steps {
                sh 'npm run test'
                sh 'npm run build'
            }
        }

        stage('Publish Artifacts') {
            steps {
                // Deployer targets the private local repo — this is the ONE TIME
                // the compiled artifact is uploaded.
                rtNpmDeployer (
                    id: "${SERVER_ID}",
                    repo: "${LOCAL_REPO}"
                )
                rtNpmPublish (
                    id: "${SERVER_ID}"
                )
            }
        }

        stage('Scan & Collect Build Info') {
            steps {
                // Captures agent environment variables into Build Info.
                rtCollectEnv (id: "${SERVER_ID}")

                // Runs an Xray scan against the just-published artifact.
                // failOnScanFailure hard-stops the pipeline on policy violation.
                rtXrayScan (
                    id: "${SERVER_ID}",
                    failOnScanFailure: true
                )

                // Publishes the complete, immutable Build Info JSON ledger.
                rtPublishBuildInfo (id: "${SERVER_ID}")
            }
        }
    }

    post {
        always  { cleanWs() }
        success { echo "Build ${BUILD_NAME} #${BUILD_NUMBER} published to Artifactory!" }
        failure { echo "Pipeline failed. Check Artifactory access logs or Xray alerts." }
    }
}
```

### Prerequisites

1. Install the **JFrog Plugin** via *Manage Jenkins → Plugins*.
2. Configure the server under *Manage Jenkins → System → JFrog Platform Instances* — assign the Server ID (must match `SERVER_ID` above) and store credentials in Jenkins Credentials Manager.
3. Ensure the ecosystem manifest exists at repo root: `package.json` for npm, `pom.xml` for Maven, `Dockerfile` for Docker builds.

### Variant: Maven Jenkinsfile Stage

```groovy
stage('Maven Build & Deploy') {
    steps {
        rtMavenDeployer (
            id: "${SERVER_ID}",
            releaseRepo: 'maven-dev-local',
            snapshotRepo: 'maven-dev-local'
        )
        rtMavenResolver (
            id: "${SERVER_ID}",
            releaseRepo: 'maven-virtual',
            snapshotRepo: 'maven-virtual'
        )
        rtMavenRun (
            id: "${SERVER_ID}",
            tool: 'maven3',
            goals: 'clean install'
        )
    }
}
```

### Variant: Docker Build & Push Stage

```groovy
stage('Docker Build & Push') {
    steps {
        script {
            def rtDocker = Artifactory.docker(server: Artifactory.server("${SERVER_ID}"))
            def buildInfo = rtDocker.push(
                "my-registry.jfrog.io/docker-dev-local/my-app:${BUILD_NUMBER}",
                "docker-dev-local"
            )
            buildInfo.env.collect()
            server.publishBuildInfo(buildInfo)
        }
    }
}
```

The architectural sequence (authenticate → resolve → build → publish → scan → ledger) is **identical** across all three ecosystems — only the ecosystem-specific `rt*` step names change.

## 3.4 Best Practices & Interview Nuances

- Store `credentialsId` in Jenkins Credentials Manager — **never** hardcode tokens in the `Jenkinsfile` itself.
- Use **scoped identity tokens**, not personal passwords, for CI service accounts — they're independently revocable and don't couple to a human's password rotation.
- Always run `rtXrayScan` **before** any promotion stage — fail fast, before a vulnerable artifact advances further down the pipeline.
- Keep `Jenkinsfile`s ecosystem-agnostic where possible by parameterizing repo names via `environment {}` blocks, so the same pipeline template can be reused across services.
- Interview trap: *"Why not just `docker push` directly to a registry?"* → You lose automatic Build Info correlation, dependency graph capture, and centralized promotion tracking that the `rt*` steps provide for free — `docker push` alone gives Artifactory no linkage back to the git commit or CI job that produced the image.
- Interview trap: *"What happens if `rtXrayScan` finds a violation but `failOnScanFailure` is false?"* → The pipeline continues, but the violation is still recorded against the artifact in Xray — useful for a "warn, don't block" rollout phase before enforcing hard gates.

---

[← Module 2: Storage & Metadata](02-storage-metadata-bom.md) | [Module 4: Artifact Promotion & Release Lifecycles →](04-promotion-release-lifecycle.md)
