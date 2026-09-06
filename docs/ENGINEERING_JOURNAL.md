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

# SESSION 4 — 2026-09-05 — First Live CI Run: SonarCloud Setup & Real Bugs Found

## Objective
Finish wiring `ci.yml` to a real SonarCloud project, push, trigger the first-ever live run of the migrated pipeline, and fix whatever it surfaces.

## What I Did

### SonarCloud project setup
Created the SonarCloud organization and project through its GitHub Actions-based wizard rather than "Automatic Analysis" (the two conflict — automatic analysis actively rejects a CI-driven scan). Corrected `sonar-project.properties` to the project's real assigned keys (`sonar.organization=cyprientemateu`, `sonar.projectKey=cyprientemateu_cyprien-ecommerce-project` — different from the placeholder guessed in Session 3). Also fixed a latent problem the wizard comparison surfaced: `sonar.java.binaries` pointed at a path with no compiled classes at all — added a Maven compile step ahead of the scan and pointed the property at each service's real `target/classes`.

### First live run — five real failures, all root-caused
Pushed and watched the first full run. Five jobs failed; none were flukes:

| Job | Root cause | Fix |
|---|---|---|
| `mvn test` (ui/cart/orders) | `mvnw` wrapper scripts were committed with mode `100644` (not executable) — an artifact of the repo's Windows-authored history | `git update-index --chmod=+x` on all three |
| `SCA (catalog)` — govulncheck | `go install govulncheck@latest` needs a modern Go toolchain; `setup-go` was matching `catalog/go.mod`'s declared `go 1.18`, too old to build it | Pinned `setup-go` to `1.23` for that step only (govulncheck still analyzes the older module fine) |
| `Unit tests (checkout)` | This fork ships **zero** `*.spec.ts` unit test files for checkout — only an e2e spec requiring a running app. Jest exits 1 on "no tests found" by default. Never caught before because the old Jenkins pipeline only ran `node --version`, never `npm test` | Added `--passWithNoTests`, documented as a known gap rather than silently faked |
| `SAST (Semgrep)` | Not a bug — the scan succeeded and found 36 real findings, then exited 1 because `--error` is designed to fail the build on findings | Switched to report-only (SARIF → GitHub code scanning tab), same rollout approach already used for Trivy; real findings triaged below |
| — (would have failed later) | `actions/dependency-review-action@v4` **does not exist** (only goes up to `v3`); `aquasecurity/trivy-action@0.24.0` was missing its `v` prefix (real tag: `v0.36.0`) | Both caught proactively while resolving action versions, before those jobs ever ran, and fixed |

### Supply-chain hardening, prompted by Semgrep's own findings
Semgrep's 36 findings included several categories:
- **5 "mutable action tag" findings** — every `uses: action@vN` reference in `ci.yml`/`codeql.yml` uses a floating tag. Fixed by pinning every action to its exact commit SHA (resolved via `git ls-remote --tags` against each action's repo), with the version kept as a trailing comment for readability.
- **2 "shell injection" findings** — `${{ github.ref }}` / `${{ github.ref_name }}` were interpolated directly into `run:` shell blocks. Fixed by routing them through `env:` first, per Semgrep's own recommended pattern.
- **3 findings in application source** (`cart`/`orders`/`ui` `application.yml`: Spring Boot Actuator fully exposed via `include: '*'`) and **1 in `do-it-yourself/src/load-generator/manifest.yml`** (missing `securityContext.allowPrivilegeEscalation: false`) — real, but application/config security posture decisions, not CI plumbing. Deliberately left alone rather than changed unilaterally; tracked below as a backlog item.

## Capabilities at this point
- `ci.yml` has now actually executed on GitHub, not just been authored
- Every previously-failing job has a root-caused fix, not a suppressed symptom
- SonarCloud project is real and correctly keyed
- Every GitHub Action reference across both workflow files is SHA-pinned
- Two genuine, previously-undetected latent bugs (`dependency-review-action@v4`, `trivy-action@0.24.0`) were caught and fixed *before* they ever executed
- Two real application-level security findings are now visible and tracked (Spring Actuator exposure, missing pod `securityContext` on the load generator) — not fixed yet, by design (see roadmap)

## Lessons Learned
- A workflow file that parses as valid YAML and "looks right" can still reference a non-existent action version or a mistyped tag — resolving every `uses:` line against the real upstream tag list (`git ls-remote --tags`) is worth doing once, not just trusting memory of common version numbers.
- `git ls-remote --tags` distinguishes annotated tags (which show a second `^{}` line with the real commit SHA) from lightweight tags (single line, already the commit) — pinning to the wrong one silently pins to a tag object instead of a commit.
- A repo's file-mode bits can be wrong for reasons that predate any of the current work (Windows-authored history, `core.filemode=false`) — `git ls-files -s <path>` is the fast way to check the actual tracked mode without trusting what's on disk.
- A security scanner correctly doing its job (Semgrep exiting 1 on real findings) looks identical in a CI failure list to a scanner that's broken — the fix isn't always "make it pass," sometimes it's "decide the right enforcement posture," which is a different question than "why did this fail."
- Catching a broken action reference before it ever runs (by resolving tags proactively) is strictly better than finding out when that job finally executes days/weeks later on some other trigger path (a git-tag push, in this case).

---

# SESSION 5 — 2026-09-05 — Second Live Run: Real Findings vs. Real Bugs

## Objective
Push the Session 4 fixes, watch the second live run, and separate "the pipeline is broken" from "the pipeline correctly found something real."

## What I Did

### Progress from Session 4's fixes
`checkout`, `ui`, `orders`, and `catalog` unit tests all passed this run — confirming the `mvnw` executable-bit fix and the `--passWithNoTests` fix both worked.

### `cart` unit tests — a second real bug
`DynamoDBCartServiceTests` (4 tests) failed with `SdkClientException: Unable to load AWS credentials from any provider in the chain`. Traced it to the actual code:
- `DynamoDBConfiguration.amazonDynamoDB()` only overrides the client's endpoint if `carts.dynamodb.endpoint` is set — otherwise it builds a client pointed at real AWS.
- The test's `application.properties` sets `aws.accessKeyId`/`aws.secretKey`, but those are **Spring properties in a test resource file**, not JVM system properties — the AWS SDK v1 credential chain only reads actual `System.getProperty("aws.accessKeyId")`/`aws.secretKey`, which Spring's environment never populates. The test's credential setup has never actually worked; it was masked by `-DskipTests=true` in the old Jenkins pipeline.
- This exactly mirrors what `docker-compose.yml` already does for local dev (`CARTS_DYNAMODB_ENDPOINT`, dummy `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`) — the pattern already exists in the project, it just was never wired into the test run.

**Fix**: added a conditional step (only for the `cart` matrix leg) that starts `amazon/dynamodb-local:1.20.0` via `docker run`, waits for it to accept connections, then runs `mvnw test` with `CARTS_DYNAMODB_ENDPOINT=http://localhost:8000` (env var, relaxed-bound to `carts.dynamodb.endpoint`) and `-Daws.accessKeyId=test -Daws.secretKey=test` (real JVM system properties this time, which Surefire forwards to the forked test JVM by default).

### Semgrep — a workflow bug, not a code bug
Semgrep's scan itself succeeded (0 exit) after Session 4's switch to `--sarif` output, but the **new** `upload-sarif` step failed: `Error: Resource not accessible by integration`. Root cause: uploading SARIF to GitHub's code scanning API requires the `security-events: write` permission, which this job didn't declare (GitHub's default `GITHUB_TOKEN` permissions don't include it). Fixed by adding an explicit `permissions:` block to the `sast-semgrep` job. Proactively added the same block — plus `id-token: write`, which keyless cosign signing needs for its Sigstore OIDC flow — to `build-and-push` too, since it has the same Trivy-upload-sarif step and hasn't run yet.

### `govulncheck` and `npm audit` — real findings, not bugs
Both tools now run successfully (the Go-toolchain fix from Session 4 worked) and both exit non-zero because they found **real, extensive, pre-existing vulnerabilities**: 39 findings from `govulncheck` (mostly Go standard-library CVEs only fixed in newer Go patch releases than this module's declared `go 1.18`), and 96 from `npm audit` (`10 low, 54 moderate, 26 high, 6 critical`, frozen dependency versions from the original 2022-era AWS sample fork). Applied the same report-only rollout already used for Trivy and Semgrep: `continue-on-error: true` on both steps, to be flipped to blocking once each baseline is triaged.

## Capabilities at this point
- Every previously-failing job now has a diagnosed, correct fix in place (pushed, not yet re-verified with a clean run)
- Three categories of "failure" are now clearly distinguished in the journal: infrastructure/config bugs (mvnw perms, wrong action refs, missing job permissions) vs. missing test setup (cart's DynamoDB dependency) vs. real, extensive pre-existing findings that need a separate triage effort (Go stdlib CVEs, npm audit backlog) rather than blocking this migration

## Lessons Learned
- A test class's credential setup can look complete (values are present in a properties file) while being silently wrong (wrong property namespace entirely) — always check *which* runtime actually reads the property, not just whether a property with a plausible name exists somewhere.
- Maven Surefire forwards the launching JVM's system properties to the forked test JVM by default (no explicit `<systemPropertyVariables>` needed) unless the project's POM overrides that — worth confirming per-project rather than assuming either way.
- `upload-sarif` and any GitHub code-scanning integration needs `security-events: write` explicitly declared per job; it is not part of the default `GITHUB_TOKEN` permission set.
- Cosign's keyless signing flow needs `id-token: write` for its Sigstore OIDC exchange — worth granting proactively before the job that needs it ever runs, rather than waiting to discover the gap live.
- Three different tools (Semgrep, govulncheck, npm audit) all needed the exact same policy decision on their first real run: report-only until the pre-existing finding baseline is triaged. Recognizing that as one repeated pattern, not three separate problems, kept the fixes consistent.

---

# SESSION 6 — 2026-09-06 — First Fully Green Run: Two Credential-Scope Bugs

## Objective
Resolve whatever remained from Session 5's push and get `ci.yml` to a genuine, fully green, end-to-end run.

## What I Did

### `build-and-push` — Docker Hub token scope
All 11 image builds failed identically at the push step:
```
failed to push .../a1cyprien_do_it_yourself_assets:8864787: failed to authorize:
failed to fetch oauth token: ... 401 Unauthorized: access token has insufficient scopes
```
Login succeeded (the base image pull worked fine), only the push was rejected — the signature of a Docker Hub access token created with **Read-only** scope rather than **Read & Write**. Fixed on Docker Hub's side by regenerating `DOCKERHUB_TOKEN` with write access. Not a workflow bug; a credential-provisioning gap from initial secret setup.

### `trigger-cd-update` — wrong PAT permission (my error)
`repository_dispatch` failed: `Error: Resource not accessible by personal access token`. Root cause: my original guidance for `CD_REPO_DISPATCH_TOKEN` (back when repo secrets were first set up) specified `Actions: Read and write` — but the `POST /repos/{owner}/{repo}/dispatches` endpoint actually requires **Contents: Read and write** on a fine-grained PAT. `Actions` permission covers workflow-run operations (re-run, cancel, read logs), not triggering dispatch events. Fixed by editing the token's permissions on GitHub (repository access stayed scoped to just the automation repo; only the permission changed, not the token string).

### `SCA (dependency review, PRs only)` — skip, not a bug
User asked why this job showed as skipped. By design: `if: github.event_name == 'pull_request'` — the GitHub `dependency-review-action` diffs a PR's dependency changes against its base branch, which only makes sense on an actual pull request, not a direct push to `main`. It will run for real on the next PR opened against `main`.

## Result
**First fully green run, end to end**: secrets scan → all 5 services' unit tests → both SAST tools → all SCA jobs (report-only ones included) → all 11 image builds, scans, SBOMs, and signings → the `repository_dispatch` notification to the automation repo. Phase 1 (GitHub Actions CI migration) is complete and verified, not just authored.

## Lessons Learned
- Docker Hub access tokens are scoped at creation (Read-only / Read & Write / Read-Write-Delete) — a token that successfully authenticates can still be rejected on `push` specifically if scoped read-only. The "401 insufficient scopes" error text is the tell, distinct from a plain "unauthorized" login failure.
- GitHub fine-grained PAT permission names don't always map intuitively to REST API operations — `repository_dispatch` sits under **Contents**, not **Actions**. When granting a PAT for a specific API call, check that endpoint's documented required permission directly rather than guessing from the feature area it seems to belong to. (This was my own mistake in the original secret-setup guidance — worth remembering for the next token this project needs.)
- A job showing "Skipped" in the Actions UI isn't inherently a signal something's wrong — check the job's `if:` condition against what actually triggered the run before assuming it's broken.

---

# 📊 Current Capabilities (as of Session 6)

## Done
- Both repos on stable, corruption-free git, outside OneDrive sync
- Dead files removed from both repos
- Consolidated, single-source-of-truth CI pipeline authored **and verified with a fully green live run** for the app repo
- Real unit tests running per service, including `cart`'s DynamoDB-dependent tests (skip-tests bug fixed, mvnw permissions fixed)
- Secrets scanning (gitleaks), dual SAST (SonarCloud + Semgrep), per-language SCA, PR dependency review — all with correct, resolvable action references and correct credential scopes
- All 11 image variants build, get scanned (Trivy), get an SBOM (Syft), get signed (cosign), and push successfully to Docker Hub
- `repository_dispatch` successfully notifies the automation repo on every push to `main`
- CodeQL scheduled scanning
- SonarCloud project created and correctly keyed; `sonar-project.properties` matches the real project
- Every GitHub Action pinned to a commit SHA (Semgrep-driven supply-chain hardening)
- Semgrep, Trivy, govulncheck, and npm audit all running in report-only mode with findings visible in GitHub's code scanning tab / job logs

## Not Yet Done
- No receiving workflow in the automation repo for `repository_dispatch` yet (the event fires successfully, but nothing there listens for it)
- Automation repo's Helm chart is still the broken hand-flattened `deploy.yaml`, still with two committed base64 "secrets"
- No kind/minikube cluster or ArgoCD installed yet
- No NetworkPolicies, Pod Security Admission, or cluster-side signature verification yet
- No DAST scanning yet (needs a live deployed target)
- Old Jenkinsfiles in the automation repo (docker-compose-based deploy path) still present
- Spring Boot Actuator fully exposed on `cart`/`orders`/`ui`; `load-generator` Kubernetes manifest missing `allowPrivilegeEscalation: false`; `checkout` service has no unit test coverage at all (only an untested-in-CI e2e spec)
- Report-only findings (Semgrep, Trivy, govulncheck: 39, npm audit: 96) not yet triaged

---

# 🚀 Planned Roadmap

## Short-Term (next few working sessions)
- Triage the report-only findings (Semgrep, Trivy, govulncheck, npm audit) via the GitHub code scanning tab and each tool's own report: fix the Spring Boot Actuator over-exposure (`include: '*'`) in `cart`/`orders`/`ui`'s `application.yml`, add `securityContext.allowPrivilegeEscalation: false` to `do-it-yourself/src/load-generator/manifest.yml`, and decide on a remediation plan for the frozen npm/Go dependency baselines (39 + 96 findings) before flipping any of these gates to blocking
- Write real unit test coverage for the `checkout` service (currently zero `*.spec.ts` files — `--passWithNoTests` is a documented gap, not a fix)
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
