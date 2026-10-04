# 🛍️ 🚀 Revive E-Commerce Platform — App & CI

> **Status:** Active, in-progress project. Built incrementally; see the engineering journal for session-by-session history.

A microservices e-commerce application (fork of AWS's `retail-store-sample-app`), rebuilt with a self-owned, security-first CI pipeline as a DevSecOps portfolio project. This repo owns the application source and GitHub Actions CI; its companion repo, [`cyprien-ecommerce-project-automation`](https://github.com/cyprientemateu/cyprien-ecommerce-project-automation), owns the Helm chart and GitOps/CD side.

## 🧭 Project Overview

This project was originally built during a DevOps training program on the program's shared infrastructure (self-hosted Jenkins, SonarQube, Slack). It's being rebuilt end to end on personal resources — GitHub Actions, SonarCloud, a personal Docker Hub account, a local Kubernetes cluster — and re-engineered with a genuine DevSecOps mindset, motivated by the maintainer's ongoing cybersecurity certification track.

It has evolved from a set of hand-maintained Jenkins pipelines into:
- A single, consolidated GitHub Actions CI pipeline (replacing three overlapping Jenkinsfiles)
- Secrets scanning, dual SAST, per-language SCA, container image scanning, SBOM generation, and image signing on every build
- A companion GitOps repo driving deployment via Helm + ArgoCD onto a local Kubernetes cluster
- A documented engineering journal capturing each phase, including the bugs found in the legacy pipeline and how they were fixed

> Full session-by-session history, findings, and the roadmap live in [`docs/ENGINEERING_JOURNAL.md`](docs/ENGINEERING_JOURNAL.md).

---

## 🚀 Key Features

### ✅ CI Pipeline (GitHub Actions)
- Consolidated single pipeline (`.github/workflows/ci.yml`) replacing three overlapping legacy Jenkinsfiles
- Per-service unit test matrix — Go (`catalog`), Maven (`ui`/`cart`/`orders`), Node (`checkout`) — with a legacy `-DskipTests=true` bug fixed so tests actually run
- Git-SHA image tagging, `latest` on `main`, semver on release tags

### ✅ DevSecOps Tooling
- **Secrets scanning:** gitleaks, gating every other job
- **SAST:** SonarCloud (enforcing) + Semgrep (`p/ci` + `p/owasp-top-ten`, **report-only** for now)
- **SCA:** govulncheck (Go), OWASP Dependency-Check (Maven), and `npm audit` (Node) all run **report-only**; GitHub-native dependency review on PRs is enforcing
- **Container security:** Trivy image scanning (**report-only**) + Syft CycloneDX SBOM generation on every build
- **Supply chain:** keyless image signing with cosign
- **CodeQL:** scheduled + PR-triggered, kept out of the main CI path to avoid slowing every push

> Semgrep, Trivy, govulncheck, OWASP Dependency-Check, and npm audit are deliberately non-blocking right now — each has a real backlog of pre-existing findings to triage first. See the journal's roadmap for the plan to flip them to hard gates.

### ✅ GitOps/CD (companion repo)
- Helm chart deployed via ArgoCD onto a local kind cluster
- Environment-overlay values (`dev` auto-syncs, `production` requires manual approval)
- `repository_dispatch` handoff from this repo's CI instead of a broad cross-repo push token

### ✅ Documentation
- This README
- [`docs/ENGINEERING_JOURNAL.md`](docs/ENGINEERING_JOURNAL.md) — phase-by-phase engineering journal, current capabilities snapshot, and roadmap

---

## 🏗 Architecture

```text
┌──────────────────────────────┐   repository_dispatch   ┌───────────────────────────────┐
│  cyprien-ecommerce-project   │ ───────────────────────▶│ cyprien-ecommerce-project-     │
│  (this repo: app + CI)       │   {tag, environment}     │ automation (Helm + GitOps/CD)  │
│                               │                          │                                │
│  test → SAST/SCA →            │                          │  umbrella Helm chart           │
│  build → Trivy/SBOM/cosign →  │                          │  ArgoCD Application (dev/prod)│
│  push to Docker Hub           │                          └───────────────┬────────────────┘
└───────────────┬───────────────┘                                          │ sync
                │ push images                                              ▼
                ▼                                                ┌───────────────────────┐
       ┌──────────────────┐          pulls images                │ kind                   │
       │   Docker Hub      │◀─────────────────────────────────── │ ui, catalog, cart,     │
       │ cyprientemateu/*  │                                     │ orders, checkout,      │
       └──────────────────┘                                     │ assets + datastores    │
                                                                  └───────────────────────┘
```

### Services

| Service | Language / Framework | Datastore | Purpose |
| --- | --- | --- | --- |
| `ui` | Java / Spring Boot | — | Storefront web front-end |
| `catalog` | Go | MariaDB | Product catalog API |
| `cart` | Java / Spring Boot | DynamoDB (local) | Shopping cart API |
| `orders` | Java / Spring Boot | PostgreSQL + RabbitMQ | Order processing API |
| `checkout` | Node / NestJS | Redis | Checkout orchestration |
| `assets` | Static / nginx | — | Static asset serving |

> **Naming note:** the cart service's source directory and Docker image are `cart` (singular), but its Helm chart and Kubernetes resources are named `carts` (plural) — a naming split inherited from the original AWS sample app, not a typo.

---

## 🧰 Technologies Used

- **CI:** GitHub Actions
- **SAST:** SonarCloud, Semgrep, CodeQL
- **SCA:** govulncheck, OWASP Dependency-Check, npm audit, GitHub dependency review
- **Container security:** Trivy, Syft (SBOM), cosign (signing)
- **Secrets scanning:** gitleaks
- **Registry:** Docker Hub
- **CD:** Helm, ArgoCD, kind (see companion automation repo)
- **App stack:** Go, Java/Spring Boot, Node/NestJS, MariaDB, DynamoDB, RabbitMQ, Redis

---

## 📁 Project Structure

```text
cyprien-ecommerce-project/
│
├── docs/
│   └── ENGINEERING_JOURNAL.md       # phase-by-phase journal, capabilities, roadmap
│
├── .github/
│   └── workflows/
│       ├── ci.yml                   # main pipeline: test → SAST/SCA → build → scan/SBOM/sign → dispatch
│       └── codeql.yml               # scheduled + PR CodeQL analysis
│
├── do-it-yourself/
│   ├── src/
│   │   ├── ui/            # Java/Spring storefront
│   │   ├── catalog/       # Go catalog API
│   │   ├── cart/          # Java/Spring cart API
│   │   ├── orders/        # Java/Spring orders API
│   │   ├── checkout/      # Node/NestJS checkout API
│   │   ├── assets/        # static assets + nginx
│   │   ├── e2e/           # end-to-end tests
│   │   └── load-generator/
│   └── helm-chart/        # per-service Helm charts (consumed as dependencies
│                           # by the automation repo's umbrella chart)
│
├── docker-compose.yml      # local dev only — not a deployment path
├── sonar-project.properties
└── README.md
```

---

## ⚙️ How It Works

1. **Push or PR** — a change to `main` or a pull request triggers `ci.yml`.
2. **Gate** — gitleaks scans for secrets before anything else runs.
3. **Test** — each service's real unit tests run in a matrix (Go/Maven/Node).
4. **Analyze** — SonarCloud and Semgrep run SAST; per-language SCA tools check dependencies.
5. **Build & secure** — on push to `main`/tags, each service image is built, scanned with Trivy, given a Syft SBOM, and signed with cosign, then pushed to Docker Hub.
6. **Hand off** — a `repository_dispatch` event notifies the automation repo with the new image tag and target environment.
7. **Deploy** — image tag bumps in the automation repo's Helm values are currently manual; a receiving workflow that consumes the `repository_dispatch` event and updates `values-dev.yaml` automatically is planned but not yet built. Once a tag is bumped (by hand or by that future workflow), ArgoCD syncs the change onto the cluster.

---

## ✅ Current Capabilities

See [`docs/ENGINEERING_JOURNAL.md`](docs/ENGINEERING_JOURNAL.md#-current-capabilities-as-of-session-7) for the full, kept-up-to-date snapshot. As of the latest session:

- Consolidated CI pipeline, replacing three overlapping legacy Jenkinsfiles — **verified with a fully green, end-to-end run**
- Real per-service unit tests, including `cart`'s DynamoDB-dependent tests (a legacy test-skipping bug is fixed)
- Secrets scanning, dual SAST, per-language SCA, container image scanning, SBOM generation, and image signing all running on every push; all 11 image variants build and push successfully to Docker Hub
- `repository_dispatch` successfully notifies the automation repo on every push to `main`
- **Full deployment pipeline working end to end**: a local kind cluster + ArgoCD run all 11 services from personal Docker Hub images, managed declaratively from the automation repo — see that repo's README for the chart/GitOps details
- Next up: the `repository_dispatch` receiving workflow (image tag bumps are still manual), Sealed Secrets for the two remaining plaintext-ish datastore credentials, and triaging the report-only findings — see the journal's roadmap

---

## ⚙️ Key Concepts Learned

### A Helm chart with valid scaffolding isn't necessarily a real template
The automation repo's chart had a correct `Chart.yaml` and `_helpers.tpl` with proper label helpers defined — but the actual `templates/deploy.yaml` was a one-time hand-flattened dump of rendered output, with only image-tag lines manually edited to look parameterized. The helpers were never `{{ include }}`'d anywhere. Lesson: check whether a template actually calls its own helpers before trusting it's maintainable.

### `-DskipTests=true` in CI is a silent no-op, not a passing test suite
The legacy pipeline ran `mvn test -DskipTests=true` for two of the five services — meaning "tests" had been compiling, not executing, for an unknown period. A green CI badge only means what the pipeline actually checks.

### Committed base64 is not encryption
Two Kubernetes `Secret` objects with real database passwords (base64-encoded, per the Kubernetes Secret spec) were committed directly to a template file. Base64 is an encoding, not encryption — anyone with read access to the repo has the plaintext credential. Treat any such exposure as a compromised credential regardless of whether git history is later scrubbed, and rotate it.

### An EOL runtime is a free security finding
The `checkout` service's CI test image was pinned to `node:14`, end-of-life since April 2023 — an easy first hit for any dependency/image scanner and a reminder to check runtime support windows during any pipeline migration, not just application dependencies.

### Cross-repo automation tokens should be scoped to the action, not the repo
Rather than giving this repo's CI a token that can `git push` into the automation repo (the legacy pattern: clone, `yq` edit, commit, push), the rebuilt pipeline uses `repository_dispatch` with a token scoped only to triggering that one event type — smaller blast radius if the token ever leaks.

---

## 🔐 Best Practices Used

- Fix silently-broken checks (skipped tests) rather than just porting them forward
- Treat any exposed secret as compromised immediately, independent of whatever remediation follows
- Scope cross-repo automation tokens to the narrowest action that accomplishes the goal
- Roll out new hard-fail security gates in report-only mode first, then flip to blocking once the baseline is triaged
- Keep hygiene commits (dead file removal) separate from functional pipeline changes
- Document every phase, including bugs found and their root cause — not just the end state

---

## 🚀 Future Improvements

- Wire the `repository_dispatch` receiving workflow in the automation repo so image tag bumps stop being manual
- Rotate the mariadb/postgresql credentials still inherited from the vendored charts' defaults and replace them with Sealed Secrets
- Triage the report-only findings (Semgrep, Trivy, govulncheck, OWASP Dependency-Check, npm audit) and decide a remediation plan before flipping any of them to blocking
- Fix the Spring Boot Actuator over-exposure (`include: '*'`) on `cart`/`orders`/`ui`
- Write real unit test coverage for `checkout` (currently zero `*.spec.ts` files)
- Build at least one service's Docker image from local source rather than relabeling a pre-built upstream image
- Add cluster-side cosign signature verification, not just sign-and-publish
- OWASP ZAP baseline DAST scan against the running app
- Full roadmap: see [`docs/ENGINEERING_JOURNAL.md`](docs/ENGINEERING_JOURNAL.md#-planned-roadmap)

---

## 🧪 Learning Outcome

This project demonstrates:
- DevSecOps pipeline design (SAST, SCA, container scanning, SBOM, signing, secrets scanning) as an integrated whole rather than bolted-on tools
- Root-cause debugging of both infrastructure issues (a corrupted local git repo) and pipeline defects (silently skipped tests)
- Secrets-management maturity: recognizing encoded-but-unencrypted credentials as an active exposure
- GitOps and Kubernetes deployment patterns split cleanly across an app/CI repo and a GitOps/CD repo
- Disciplined, phase-by-phase engineering documentation

---

## ✍️ Maintainer
Cyprien Temateu
DevOps / DevSecOps — Cybersecurity Track
`Revive E-Commerce Platform`
