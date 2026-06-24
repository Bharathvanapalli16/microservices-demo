# CI/CD with GitHub Actions + ArgoCD (GitOps) — Design

This document describes how we will implement CI/CD for the Online Boutique
microservices and integrate it with the existing ArgoCD GitOps deployment.

> Status: **Design / proposal**. Nothing in this document is wired up yet —
> it is the agreed plan before we add workflows under `.github/workflows/`.

---

## 1. Goals

- Automatically **build, test, and push** container images for the 11
  microservices when their source changes.
- Keep **Git as the single source of truth** for what is deployed. No
  `kubectl apply` from CI, no direct cluster access from the pipeline.
- Close the loop: a successful build updates the image tag in
  `kubernetes-manifests/`, and **ArgoCD** rolls it out to the cluster.
- Use **immutable, traceable** image tags (Git commit SHA).
- Build **only the services that changed** to keep pipelines fast and cheap.
- **No long-lived AWS credentials** in GitHub — use OIDC federation.

## 2. Current environment (what already exists)

| Piece | Value |
|---|---|
| Git repo | `github.com/Bharathvanapalli16/microservices-demo` (branch `main`) |
| Container registry | AWS ECR, account `209822909485`, region `ap-south-1` |
| Image ref format | `209822909485.dkr.ecr.ap-south-1.amazonaws.com/<service>:<tag>` |
| Deploy tool | ArgoCD `Application` (`argocd-app.yaml`), watches `main`, path `kubernetes-manifests` |
| Sync policy | `automated` with `prune: true` and `selfHeal: true` |
| Cluster | minikube on EC2, ingress-nginx, host `boutique.13.127.18.166.nip.io` |

ArgoCD is already configured to auto-sync from `kubernetes-manifests/`, so the
deploy half of the loop works. This design adds the **build + write-back** half.

## 3. High-level flow

```
 Developer push to main (src/** changed)
              │
              ▼
   ┌────────────────────────┐
   │  GitHub Actions (CI)    │
   │  1. detect changed svc  │
   │  2. build + unit test   │
   │  3. auth to AWS via OIDC │
   │  4. docker build/push   │  ──►  AWS ECR  (image:<git-sha>)
   │  5. kustomize set image │
   │  6. commit tag to Git   │ ──┐
   └────────────────────────┘   │ pushes manifest change to main
                                 ▼
                    kubernetes-manifests/kustomization.yaml
                                 │ (Git = source of truth)
                                 ▼
                    ┌────────────────────────┐
                    │        ArgoCD           │
                    │  detects new commit     │
                    │  auto-sync + prune      │  ──►  minikube cluster
                    │  self-heal              │
                    └────────────────────────┘
```

Two distinct commits exist per change:
1. The **developer's source commit** (touches `src/<service>/...`).
2. An **automated tag-bump commit** made by the CI bot (touches only
   `kubernetes-manifests/`). ArgoCD reacts to commit #2.

## 4. Decisions (agreed)

| Decision | Choice | Rationale |
|---|---|---|
| CI platform | **GitHub Actions** | Native to the repo; OIDC to AWS; no runners to maintain. |
| GitOps write-back | **CI commits the tag to Git** | Explicit, auditable; Git stays the source of truth. No extra controller. |
| Image tag | **Git commit short SHA** (e.g. `:a1b2c3d`) | Immutable, traceable to exact code, ideal for GitOps diffs/rollback. |
| Build scope | **Only changed services** | Faster, cheaper; path-filtered build matrix. |
| AWS auth | **GitHub OIDC → IAM role** | No static access keys stored in GitHub secrets. |

## 5. Prerequisite refactor: centralize image tags in kustomize

Today each `kubernetes-manifests/<service>.yaml` hardcodes its image, e.g.:

```yaml
image: 209822909485.dkr.ecr.ap-south-1.amazonaws.com/frontend:v0.10.5
```

Editing 11 files from CI is brittle. Instead, we will let
`kubernetes-manifests/kustomization.yaml` own the tags via the `images:`
transformer, so CI only edits **one file**:

```yaml
# kubernetes-manifests/kustomization.yaml
images:
  - name: 209822909485.dkr.ecr.ap-south-1.amazonaws.com/frontend
    newTag: a1b2c3d
  - name: 209822909485.dkr.ecr.ap-south-1.amazonaws.com/cartservice
    newTag: a1b2c3d
  # ... one entry per service
```

CI then runs, for each changed service:

```bash
cd kubernetes-manifests
kustomize edit set image \
  209822909485.dkr.ecr.ap-south-1.amazonaws.com/<service>=...<service>:<git-sha>
```

This keeps the deployment YAMLs stable and makes the tag-bump commit a tiny,
reviewable diff.

## 6. Pipeline stages (per push to `main`, paths `src/**`)

1. **Detect changes** — compute which `src/<service>` directories changed in the
   push (e.g. `dorny/paths-filter` or `git diff --name-only`). Output a matrix
   of affected services. If none changed, the build job is skipped.
2. **Build & test** (matrix, one job per changed service) — run each service's
   native build/test (Go `go test`, Java `./gradlew test`, Node `npm test`,
   Python unit tests, C# `dotnet test`). Fail fast on test failure.
3. **Authenticate to AWS** — `aws-actions/configure-aws-credentials` with OIDC,
   assuming the CI IAM role; then `aws-actions/amazon-ecr-login`.
4. **Build & push image** — `docker build` the service and push to ECR tagged
   with the **short commit SHA**. (Optionally also push `latest` for humans —
   never deploy from `latest`.)
5. **Write back tag** — in a final single job (after the matrix), run
   `kustomize edit set image` for each built service, commit to
   `kubernetes-manifests/kustomization.yaml`, and push to `main`.
6. **ArgoCD deploys** — ArgoCD detects the new commit and syncs automatically
   (already configured). No action from CI.

## 7. Avoiding pipeline loops

The build workflow triggers **only** on `paths: ['src/**']`. The CI bot's
tag-bump commit touches only `kubernetes-manifests/**`, so it does **not**
re-trigger the build workflow. As defense in depth, the bot commit message
includes `[skip ci]` and uses a dedicated bot identity.

## 8. AWS / IAM setup (OIDC, no static keys)

- Create an **IAM OIDC identity provider** for
  `token.actions.githubusercontent.com`.
- Create an **IAM role** trusted by the GitHub OIDC provider, scoped to this
  repo (and ideally the `main` branch / `ref`), with a policy allowing:
  `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`,
  `ecr:InitiateLayerUpload`, `ecr:UploadLayerPart`,
  `ecr:CompleteLayerUpload`, `ecr:PutImage`, `ecr:BatchGetImage`.
- Store the role ARN as a GitHub repo variable (not a secret — it's not
  sensitive). No `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` in GitHub.
- Ensure an **ECR repository per service** exists (create on first use or
  pre-create via Terraform/console).

## 9. Git write-back permissions

`main` is **branch-protected**, so the CI bot cannot push directly to it. The
tag-bump must therefore go through a **pull request** that is then merged
(auto-merge enabled, or merged after required checks pass).

**Decision (for now): use a fine-grained PAT.**
- Store a fine-grained **PAT** (scoped to this repo, `contents:write` +
  `pull_requests:write`) as a GitHub Actions secret, e.g. `GITOPS_PAT`.
- The write-back job uses it to create a branch, open a PR with the tag bump,
  and auto-merge it once checks pass. (The default `GITHUB_TOKEN` cannot
  satisfy branch protection on its own and can't trigger downstream workflows.)

**Later:** move to a **GitHub App** installation token (least privilege, no
personal account coupling). GitHub access from the local Claude Code
environment will be via the **GitHub MCP server**; that is for interactive
work, not the pipeline credential.

## 10. Branching & promotion (initial)

For this POC we deploy from `main` directly (single environment). When a
staging/prod split is needed later, the natural extension is:

- One ArgoCD `Application` per environment, each pointing at a different path
  or overlay (e.g. `kustomize/overlays/staging`, `.../prod`).
- CI writes the SHA into the **staging** overlay automatically; promotion to
  prod is a separate, reviewed commit/PR.

## 11. Rollback

Because every deploy is a Git commit with an immutable SHA tag:

- **Revert** the tag-bump commit (`git revert <sha>`) — ArgoCD rolls back.
- Or use the **ArgoCD UI/CLI** (`argocd app rollback`) to a previous synced
  revision. Git remains the source of truth, so prefer the Git revert.

## 12. Security considerations

- No long-lived cloud credentials in CI (OIDC only).
- CI has **no kubeconfig / cluster access** — it only pushes images and commits
  Git. The cluster pulls via ArgoCD, reducing blast radius.
- Image tags are immutable SHAs; avoid mutable `latest` in manifests.
- Scope the IAM role's trust policy to this repo/branch to prevent other repos
  assuming it.
- Consider adding image scanning (ECR scan-on-push or Trivy in CI) as a later
  enhancement.

## 13. Implementation plan (phased)

- [x] **Phase 0 — Refactor:** image tags centralized in
      `kubernetes-manifests/kustomization.yaml` `images:` block; ArgoCD renders
      identically. (merged)
- [x] **Phase 1 — AWS:** OIDC provider + role
      `github-actions-online-boutique` (trust scoped to
      `repo:Bharathvanapalli16/microservices-demo:*`, inline `ecr-push`
      policy) created; all 10 ECR repos already exist.
- [x] **Phase 2 — Build CI:** `.github/workflows/build.yml` `build` job —
      path-filtered change detection (`dorny/paths-filter`), per-service matrix,
      OIDC auth, `docker build` + push to ECR tagged with the short SHA.
- [x] **Phase 3 — Write-back:** `deploy` job — `kustomize edit set image` for
      each built service, then an auto-merged PR (`peter-evans/create-pull-request`
      + `gh pr merge --auto`). Loop-prevented by the `src/**` path filter +
      `[skip ci]`.
- [ ] **Phase 4 — Verify end-to-end:** push a trivial `src/frontend` change,
      watch CI build → push → write-back PR → merge → ArgoCD sync → new pod.
- [ ] **Phase 5 — Hardening (later):** per-language unit tests in CI, image
      scanning, PR-based build checks, staging/prod overlays.

### Required GitHub configuration (one-time, manual)

The non-secret values (role ARN, region, registry) are hardcoded in the
workflow `env:`. Only two things must be set in the repo by hand:

1. **Secret `GITOPS_PAT`** — a fine-grained PAT for this repo with
   `contents:write` + `pull_requests:write`. Used to check out, open the
   write-back PR, and enable auto-merge (the default `GITHUB_TOKEN` can't push
   to protected `main` or satisfy protection).
   *Settings → Secrets and variables → Actions → New repository secret.*
2. **Enable "Allow auto-merge"** — *Settings → General → Pull Requests.*
   Also ensure branch protection on `main` lets the PAT user's PR merge once
   checks pass (no blocking required-reviewer that the bot can't satisfy).

## 14. Open items

- ~~Confirm whether `main` has branch protection.~~ **Confirmed: protected.**
  Write-back goes via auto-merged PR using a fine-grained PAT (§9).
- ~~Decide GitHub App vs PAT vs `GITHUB_TOKEN`.~~ **PAT now; GitHub App later.**
- Decide whether to also build on PRs (build + test only, no push/commit) for
  pre-merge validation.
