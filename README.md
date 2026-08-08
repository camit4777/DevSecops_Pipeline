# End-to-End DevSecOps CI/CD using GitHub Actions

This project recreates the DevSecOps pipeline for the Spring PetClinic app shown
in the "End to End DevSecOps CI Pipeline using GitHub Action" walkthrough:
a **self-hosted GitHub Actions runner**, a set of **reusable workflows** chained
together, **SonarCloud** for SAST/code-quality, **Snyk** for dependency/image
scanning, **JFrog Artifactory** for artifact storage, and **Docker Hub** for the
final container image — with an optional CD stage into Kubernetes via Argo CD.

## DevSecOps CI/CD reference flow

```
 Agile Teams ⇄ Jira
        │
        ▼
       Git ──► Detect Secret ──► SBOM Scan ──► SAST ──► Unit-Test ──► Dockerfile Scan ──► Build
                                                                                            │
                                                                                            ▼
                                                                              Container Image scanning
                                                                                            │
                                                                                            ▼
                                                                            Container Validation Test
                                                                                            │
                                                                                            ▼
                                                                              Container Signing
                                                                                            │
        ┌───────────────────────────────────────────────────────────────────────────────────┘
        ▼
K8-Manifest Scanning ──► K8 CIS Scan ──► Deploy ──► Smoke Test / API Testing ──► DAST
                                            │
                              ┌─────────────┴─────────────┐
                              ▼                            ▼
                          Test Env                     Prod Env  ──► Monitoring

  Any red-path failure (Code Analysis, Unit Test, Dockerfile Scan, Image Scan,
  Image Validation, K8 CIS Scan, K8-Manifest Scan, Test) is filed back to Jira as a defect.
```

## Tools used and why

| Concern | Tool | Notes |
|---|---|---|
| Build | Maven (`mvn clean install`) | Spring PetClinic, Java 17 |
| SAST / code quality | **SonarCloud** | free account at https://sonarcloud.io |
| SCA / dependency & image scanning | **Snyk** | SAST (`SNYK_API_TOKEN`) — optional stage, wired in but scaffolded |
| Artifact storage | **JFrog Artifactory** | free trial account, stores the built JAR |
| Container registry | **Docker Hub** | stores/pushes the final application image |
| CI orchestration | **GitHub Actions** (self-hosted runner) | runner label `java_build` |
| CD / GitOps | **Argo CD** (optional, see `k8s/`) | deploy stage, ties into the ArgoCD project from this series |

**Snyk vs SonarCloud**, per the walkthrough:
- Snyk mostly works on security aspects like SAST and SCA (Software Composition Analysis).
- SonarCloud/SonarQube focuses on code coverage, code quality, etc.
- Both offer a range of services; DAST (OWASP ZAP, Nikto) can be layered on top.

## Repository layout

```
DevSecOps_Pipeline/
├── .github/workflows/
│   ├── feature.yml      # orchestrator — runs on push to any feature branch
│   ├── production.yml   # orchestrator — runs on push to master (adds deploy)
│   ├── javabuild.yml    # reusable: checkout, build, cache, upload to JFrog
│   ├── sonar.yml        # reusable: SonarCloud scan
│   ├── test.yml         # reusable: run the test suite
│   ├── imagepush.yml    # reusable: docker build/tag/push to Docker Hub
│   └── snyk.yml         # reusable: Snyk SCA + container scan (optional stage)
├── Dockerfile
├── docker-compose.yml    # local run: app + MySQL
├── k8s/                  # manifests for the CD/deploy stage
│   ├── deployment.yaml
│   ├── service.yaml
│   └── argocd-application.yaml
├── pom.xml                (from the forked Spring PetClinic app — not reproduced here)
├── src/                    (Spring PetClinic source — not reproduced here)
└── README.md
```

> This deliverable focuses on the **pipeline** (the part the video is actually
> teaching). The application source itself is the stock
> [Spring PetClinic](https://github.com/spring-projects/spring-petclinic) app —
> fork that repo and drop these workflow/Docker/k8s files on top of it.

## Step-by-step: how to reproduce this

### Step 1 — Fork the sample app
```bash
git clone https://github.com/<you>/DevSecops_Pipeline.git
cd DevSecops_Pipeline
```
(Fork of `spring-projects/spring-petclinic`, per the video.)

### Step 2 — Stand up a self-hosted runner
1. In GitHub: **repo → Settings → Actions → Runners → New self-hosted runner**.
2. Pick **Linux / x64**, then on your build server (an EC2 instance in the video):
```bash
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.320.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.320.0/actions-runner-linux-x64-2.320.0.tar.gz
tar xzf ./actions-runner-linux-x64-2.320.0.tar.gz
./config.sh --url https://github.com/<you>/DevSecops_Pipeline --token <REGISTRATION_TOKEN>
# when prompted for runner labels/name, keep the default group and add the label: java_build
./run.sh
# (or install as a service: sudo ./svc.sh install && sudo ./svc.sh start)
```
3. Install the software the build needs directly on that server: `java` (JDK 17), `maven`, `docker`.

### Step 3 — Add repository secrets
**Settings → Secrets and variables → Actions → New repository secret**:

| Secret | Used by |
|---|---|
| `SONAR_HOST_URL` | sonar.yml — `https://sonarcloud.io` |
| `SONAR_PROJECT_KEY` | sonar.yml |
| `SONAR_TOKEN` | sonar.yml — generate at sonarcloud.io → My Account → Security |
| `JFROG_TOKEN` | javabuild.yml |
| `JFROG_URL` | javabuild.yml — your JFrog instance URL |
| `DOCKER_USERNAME` | imagepush.yml |
| `DOCKER_PASSWORD` | imagepush.yml |
| `SYNK_API_TOKEN` | snyk.yml (optional stage) |
| `DEVSECOPS_PIPELINE_TOKEN` | used if any workflow needs to call the GitHub API (PAT) |

### Step 4 — Create the pipeline
Drop the 6 workflow files from `.github/workflows/` (below) into your repo and push.
Every commit to a feature branch runs `feature.yml`; every push to `master` runs
`production.yml`, which adds the deploy stage.

### Step 5 — Watch it run
**Actions tab** → pick the run → each reusable workflow shows as its own job
(`build`, `sonar`, `test`, `imagepush`, ...) chained by `needs:`.

### Step 6 — Verify the outputs
- **SonarCloud dashboard** — code coverage / quality gate for the project.
- **JFrog Artifactory** — the uploaded JAR under `petjfrog-repo/`.
- **Docker Hub** — the pushed image, tagged with the commit SHA.

### Step 7 — (Optional) Deploy stage
`production.yml` finishes by applying `k8s/argocd-application.yaml`, which
points Argo CD at the `k8s/` manifests so the new image gets rolled out —
this is the same GitOps pattern from the End-to-End CD Using ArgoCD walkthrough
earlier in this series.

## Notes / honesty about reconstruction

Everything in `.github/workflows/feature.yml`, `javabuild.yml`, `sonar.yml`,
and `imagepush.yml` was read directly off-screen in the video, so those four
files are close to verbatim (secrets/tokens redacted). `test.yml`,
`production.yml`'s deploy job, `snyk.yml`, and the `k8s/` manifests were not
fully readable on screen — I've filled those in following the same
conventions (`runs-on: java_build`, `workflow_call`, `secrets: inherit`) so
the pipeline is complete and internally consistent. Review before using in
a real environment, especially the Snyk stage — the video left it commented
out in a scratch file (`build.txt`) rather than actually wired into a workflow.
