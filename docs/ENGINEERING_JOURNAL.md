# 🛡️ Revive E-Commerce Platform — DevSecOps Rebuild Journal

## 🧭 Purpose

This document is the engineering journal for the rebuild of the Revive e-commerce platform's delivery pipeline. It records the project's evolution session by session across **both** repositories that make up the system:

- [`cyprien-ecommerce-project`](https://github.com/cyprientemateu/cyprien-ecommerce-project) (this repo) — application source + CI
- [`cyprien-ecommerce-project-automation`](https://github.com/cyprientemateu/cyprien-ecommerce-project-automation) — Helm chart + GitOps/CD

It tracks:

- why the rebuild was needed and what decisions were made, and why
- what was done each session, with real commands and real findings
- bugs found in the pre-existing pipeline, their root cause, and the fix
- capabilities gained at each stage
- lessons learned
- the roadmap toward full completion

This file is updated whenever a phase completes or a meaningful engineering decision is made.

---

# 📌 Project Summary

Revive is a microservices e-commerce app (fork of AWS's `retail-store-sample-app`) with six services — `ui`, `catalog`, `cart`, `orders`, `checkout`, `assets` — each with its own datastore. It was originally built and operated on a former organization's infrastructure (self-hosted Jenkins, self-hosted SonarQube, that org's Slack workspace). This rebuild moves the entire delivery pipeline onto personal resources and layers in a genuine DevSecOps posture, motivated by the maintainer's ongoing cybersecurity certification track — the goal is a pipeline that demonstrates security engineering judgment, not just a restored CI/CD pipeline.

**Decisions locked in before implementation began:**

| Decision | Choice | Why |
|---|---|---|
| CI platform | GitHub Actions | No self-hosted Jenkins server to maintain |
| CD target | Local Kubernetes (kind/minikube) + ArgoCD | No cloud spend required for a portfolio/learning deployment |
| Security depth | Full DevSecOps pass | SAST + SCA + image scanning + secrets scanning + SBOM + DAST, not a checkbox pass |
| Static analysis | SonarCloud | Free for public repos, replaces the org's self-hosted SonarQube |

---

# SESSION 1 — 2026-09-04 — Discovery, Risk Assessment & Planning

## Objective
Before changing anything, understand exactly what the existing two-repo pipeline does, what's tied to the former organization, and what a rebuilt, security-conscious version should look like.

## What I Did
- Read through both repositories end to end: every Jenkinsfile (6 total across both repos — 3 evolving variants in each), both `docker-compose.yml` files, the Helm chart in the automation repo, the vendored-but-unused per-service Helm charts, `sonar-project.properties`, and both READMEs.
- Identified every resource still pointing at the former organization (self-hosted SonarQube host, one org-owned Docker Hub image reference, an org Slack channel).
- Discovered the automation repo's `chart/templates/deploy.yaml` is not a real Helm template — it's a one-time hand-flattened dump of ~6 services' manifests, with only image-tag lines manually edited to look templated. Its own `_helpers.tpl` label helpers are defined but never actually used.
- Found two real base64-encoded Kubernetes `Secret` objects (`catalog-db`, `orders-db` passwords) committed directly to that file.
- Found a bug in the existing Jenkins pipeline: `mvn test -DskipTests=true` for the cart and orders services — meaning their unit tests had never actually been executed by CI, only compiled.
- Confirmed this repo's `checkout` service pinned its CI test image to `node:14`, which has been end-of-life since April 2023.

## Key Finding
The vendored `do-it-yourself/helm-chart/<service>/` directory already contains well-built, correctly parameterized per-service Helm charts (real templates, security contexts with `cap_drop: ALL` + `readOnlyRootFilesystem` + `runAsNonRoot`, resource limits) — but they're unused and still point at a stale third-party image (`prinsoo/revive-app`). This became the anchor decision for the CD rebuild: repoint and reuse these charts as umbrella-chart dependencies, rather than hand-writing new templates from scratch.

## Decisions Made
Confirmed with the maintainer: GitHub Actions for CI, local kind/minikube + ArgoCD for CD, a full DevSecOps tooling pass, and SonarCloud for static analysis (see summary table above).

## Capabilities at this point
- **None changed yet** — this was a read-only discovery and planning session. Both repos still ran (in principle) on the old Jenkins pipelines, still had org-tied resources, and still had the two committed secrets.

## Lessons Learned
- A Helm chart directory with a valid `Chart.yaml` and `_helpers.tpl` is not proof the chart is actually templated — always check whether the templates reference the helpers, and whether a "template" file is actually static rendered output.
- `-DskipTests=true` flags in a CI pipeline are a red flag worth grepping for; a "passing" pipeline can be silently not testing anything.
- Before touching a legacy pipeline, map every hostname/credential/Slack channel/registry reference back to "whose infrastructure is this," not just "does it work."

---

# SESSION 2 — 2026-09-05 — Phase 0: Repository Recovery & Hygiene

## Objective
Fix a critical environment problem discovered while starting implementation, then clear out dead files from both repos before any pipeline changes land.

## What I Did

### The corruption
Both local repo copies lived under an OneDrive-synced folder (`OneDrive\Desktop\TCC-TEST\...`). Running `git status` in the automation repo failed outright:
```text
$ git log --oneline -3
fatal: bad object HEAD
```
Diagnosis showed `HEAD` → `refs/heads/main` → a commit hash with **no corresponding object** in `.git/objects` at all — the classic failure mode of OneDrive's file-locking/renaming interfering with git's content-addressed loose objects.

### The fix
```bash
mkdir -p C:/dev
cd C:/dev
git clone git@github.com:cyprientemateu/cyprien-ecommerce-project.git
git clone git@github.com:cyprientemateu/cyprien-ecommerce-project-automation.git

cd C:/dev/cyprien-ecommerce-project-automation
git fsck                 # clean
git log --oneline -3     # HEAD resolves correctly again
```
Before discarding the corrupted OneDrive copy, diffed it against the fresh clone to make sure no uncommitted work would be lost:
```bash
diff -rq "<OneDrive copy>" "C:/dev/cyprien-ecommerce-project-automation" --exclude=.git
# Only difference: validate.sh — a CRLF/LF line-ending difference, not a content change.
```
No work was lost. Both repos now live at `C:\dev\...`, outside any synced folder, eliminating the corruption risk going forward.

### Cleanup
Removed dead files that had no reference anywhere in either pipeline (verified by grepping all Jenkinsfiles/docs for each filename before deleting):

| Repo | File removed | What it was |
|---|---|---|
| app | `Jenkins Built-in environment Variables.yml` | Reference doc, irrelevant post-Jenkins |
| app | `do-it-yourself/helm-chart/check.yaml` | A pasted `helm install --dry-run` terminal output, not a real template |
| app | `do-it-yourself/src/checkout/a.sh` | Unrelated debug scratch script (`name=maryam` test snippet) |
| automation | `chart/a.sh` | One-line manual `yq` scratch command |
| automation | `chart/data.yaml` | Orphaned tag file, never referenced by any Jenkinsfile |

Both cleanups were committed and pushed as their own commits, kept separate from any functional pipeline change.

## Capabilities at this point
- Both repos are on a stable, corruption-free local git setup.
- Both working trees are free of dead scratch files and stale reference docs.
- No pipeline behavior has changed yet — Jenkinsfiles are still present and would still run as before if pointed at a Jenkins server.

## Lessons Learned
- Syncing a `.git` directory through OneDrive/Dropbox-style file sync is a known corruption risk — loose objects are immutable and content-addressed, and sync clients that rename/lock files mid-write can leave refs pointing at objects that never fully landed.
- Always diff a suspect working tree against a fresh clone before discarding it — don't assume "the remote has everything" without checking.
- Cleanup commits are worth keeping separate from functional changes; it keeps the diff of the "real" change reviewable.

---

# SESSION 3 — 2026-09-05 — Phase 1: GitHub Actions CI Migration

## Objective
Replace all three Jenkinsfiles in this repo with a single, consolidated GitHub Actions pipeline, fixing the known test-skipping bug along the way and laying in the first layer of DevSecOps tooling.

## What I Did
Authored `.github/workflows/ci.yml` consolidating `Jenkinsfile`, `Jenkinsfile3`, and `Jenkinsfile5` into one pipeline:

- **`secrets-scan`** (gitleaks) — gates every other job.
- **`test`** — matrix over `catalog(go) / ui(maven) / cart(maven) / orders(maven) / checkout(node)`. The `mvn test -DskipTests=true` bug from Session 1 is fixed — tests run for real now. `checkout`'s runtime bumped from `node:14` (EOL) to `node:20`.
- **`sast-sonarcloud`** + **`sast-semgrep`** — SonarCloud (`p/ci` default rules) plus Semgrep with `p/ci` + `p/owasp-top-ten` rulesets for security-specific patterns Sonar under-emphasizes.
- **`sca-dependency-review`** (PR-only, GitHub-native) + **`sca-per-language`** — `govulncheck` for catalog, OWASP Dependency-Check for the three Maven services, `npm audit --audit-level=high` for checkout.
- **`build-and-push`** — matrix over all 11 image variants (services + their `-db`/`-dynamodb`/`-rabbit-mq` companions), tagged with short git SHA (mirroring the one part of the old Jenkins pipelines that was already solid), `latest` on `main`, semver on `v*.*.*` tags. Each build immediately runs a **Trivy** scan (report-only for now — `exit-code: 0` — until the initial finding baseline is triaged), generates a **Syft** CycloneDX SBOM, and signs the image **keylessly with cosign**.
- **`trigger-cd-update`** — replaces the old Jenkins pattern of cloning the automation repo and running `yq`+`git push` directly from this repo's pipeline. Uses `repository_dispatch` instead, so this repo only needs a narrowly-scoped dispatch token rather than a token capable of pushing arbitrary commits to the other repo.
- **`validate-release-tag`** — inlines the same semver check `validate.sh` (in the automation repo) already does, run whenever the trigger is a `v*.*.*` tag push.

Also added `.github/workflows/codeql.yml` as a separate, lighter-weight scan (PR + weekly schedule) so CodeQL doesn't add latency to every push.

Updated `sonar-project.properties`: removed `sonar.host.url` (the former org's self-hosted SonarQube), added a `sonar.organization` placeholder for SonarCloud.

Deleted `Jenkinsfile`, `Jenkinsfile3`, `Jenkinsfile5` — fully superseded.

## Capabilities at this point
- A single, git-diffable CI definition exists, replacing three overlapping Jenkins pipelines.
- Every service's unit tests run for real (the skip-tests bug is fixed).
- Secrets scanning, two SAST tools, per-language SCA, container image scanning, SBOM generation, and image signing are all wired into the pipeline definition.
- **Not yet verified against a live run** — this workflow has not executed on GitHub yet. It needs four repo secrets configured first (`DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `SONAR_TOKEN`, `CD_REPO_DISPATCH_TOKEN`) and has not yet been committed/pushed.
- The automation repo has no receiving workflow yet for the `repository_dispatch` event this pipeline sends — that lands in Phase 5.

## Lessons Learned
- Folding Trivy + Syft + cosign into the same matrix leg as the build (rather than separate jobs that re-pull the image) avoids redundant image pulls and keeps per-image results co-located in one job's log.
- Introducing a hard-fail security gate (`exit-code: 1` on Trivy findings) on day one of a migration is likely to block on a backlog of pre-existing base-image CVEs unrelated to this change; starting in report-only mode and flipping to blocking once the baseline is triaged is a more realistic rollout.
- A `repository_dispatch` token scoped to just "trigger a workflow in repo X" has a much smaller blast radius than a token that can `git push` — worth the small extra plumbing of having the receiving repo own its own mutation logic.

---

# 📊 Current Capabilities (as of Session 3 / Phase 1)

## Done
- Both repos on stable, corruption-free git, outside OneDrive sync
- Dead files removed from both repos
- Consolidated, single-source-of-truth CI pipeline authored for the app repo
- Real unit tests running per service (skip-tests bug fixed)
- Secrets scanning (gitleaks), dual SAST (SonarCloud + Semgrep), per-language SCA, PR dependency review
- Container image scanning (Trivy), SBOM generation (Syft), keyless image signing (cosign) wired into every build
- CodeQL scheduled scanning
- `sonar-project.properties` retargeted at SonarCloud

## Not Yet Done
- CI workflow not yet committed/pushed or run live (pending repo secrets)
- No receiving workflow in the automation repo for `repository_dispatch` yet
- Automation repo's Helm chart is still the broken hand-flattened `deploy.yaml`, still with two committed base64 "secrets"
- No kind/minikube cluster or ArgoCD installed yet
- No NetworkPolicies, Pod Security Admission, or cluster-side signature verification yet
- No DAST scanning yet (needs a live deployed target)
- Old Jenkinsfiles in the automation repo (docker-compose-based deploy path) still present

---

# 🚀 Planned Roadmap

## Short-Term (next few working sessions)
- Get `ci.yml` its required secrets and confirm a real green run on GitHub, including a look at the first Trivy/SBOM artifacts and the SonarCloud dashboard
- Convert at least one service's Dockerfile (catalog is the easiest — Go, straightforward multi-stage build) to actually compile from local source, since most services today just relabel AWS's pre-built images rather than shipping what CI tested
- Stand up a local kind cluster and ArgoCD; fix the automation repo's Helm chart by turning it into an umbrella chart over the already-well-built (but currently unused) per-service charts; delete the broken `deploy.yaml`
- Rotate the two exposed database credentials and replace the committed plaintext-ish secrets with Sealed Secrets

## Mid-Term
- Wire the `repository_dispatch` → automation-repo receiving workflow so a commit in the app repo flows untouched-by-hand into a running pod in kind
- Add NetworkPolicies (with an explicit note on kind's default CNI not enforcing them without Calico), Pod Security Admission labels on namespaces, and fix the `assets` service's inconsistent security context
- Flip the Trivy scan from report-only to a real CRITICAL/HIGH gate once the initial finding baseline is triaged
- Retire the automation repo's docker-compose-based deploy Jenkinsfile now that ArgoCD sync supersedes it

## Long-Term
- OWASP ZAP baseline DAST scan against the running kind `ui` service (scheduled/manual, since it needs a live target)
- Cluster-side cosign signature verification (admission-controller enforcement), not just sign-and-publish
- Production promotion path: manual-sync ArgoCD Application gated by the semver release tag flow
- Full README/documentation polish in both repos reflecting the finished architecture, replacing any remaining copy-pasted generic docs
- Consider whether a low-cost managed Kubernetes target (vs. local kind) is worth adding for a more "always-on" portfolio demo

---

# ✍️ Maintainer
Cyprien Temateu
DevOps / DevSecOps — Cybersecurity Track
`Revive E-Commerce Platform — DevSecOps Rebuild`
