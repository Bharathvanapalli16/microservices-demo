# Trivy Image Scanning — Plan

Add container-image vulnerability scanning with [Trivy](https://aquasecurity.github.io/trivy/)
to the existing CI/CD pipeline (`.github/workflows/build.yml`), as a security
gate before images are deployed via ArgoCD.

> Status: **Plan / proposal.** Extends Phase 5 of `docs/ci-cd-argocd.md`.
> Nothing is wired up yet.

---

## 1. Goals

- Scan every service image we build for known OS + library CVEs.
- **Block the pipeline on CRITICAL vulnerabilities** (so a vulnerable image is
  never pushed to ECR or deployed by ArgoCD).
- Surface all findings (HIGH + CRITICAL) in the **GitHub Security tab** for
  tracking and triage.
- Keep it fast (cache the Trivy vulnerability DB) and low-noise (only gate on
  CVEs that actually have a fix available).

## 2. Decisions (agreed)

| Decision | Choice | Notes |
|---|---|---|
| Gate severity | **Fail on CRITICAL only** | `--ignore-unfixed` so we only block when a patch exists. |
| Report severity | **HIGH + CRITICAL** | Reported via SARIF even though only CRITICAL blocks. |
| Results destination | **GitHub Security tab (SARIF)** | `github/codeql-action/upload-sarif`. Free on this **public** repo; would need GitHub Advanced Security if the repo becomes private. |
| Scope | **Container images only** | OS packages + app dependencies. IaC/misconfig scanning is a later add-on (§9). |
| Where in pipeline | **In the `build` job, between build and push** | Build locally → scan/gate → push only if it passes. |

## 3. Where it fits in the pipeline

Today the `build` matrix job builds and pushes in one step
(`docker/build-push-action` with `push: true`). To gate **before** the image
reaches ECR, we split that into **build (load) → scan → push**:

```
detect ─► build (matrix, per changed service)
            ├─ docker build  (load image into local daemon, cached)
            ├─ trivy scan → SARIF  ──► upload to Security tab   (never fails)
            ├─ trivy scan → gate   ──► exit 1 on CRITICAL        (fails job)
            └─ docker push  (only reached if the gate passed)
          ─► deploy (write-back)   # needs: build → skipped if any build leg failed
```

Because `deploy` has `needs: [detect, build]`, a failed scan on **any** changed
service fails the `build` job and therefore **skips the write-back/deploy** —
no `kustomization.yaml` bump, so ArgoCD never sees the bad image. (Trade-off:
one service's CRITICAL CVE blocks the deploy of all services changed in that
push. Acceptable for this POC.)

## 4. Concrete workflow changes

### 4.1 Permissions
The `build` job needs to upload SARIF, so add `security-events: write` to it
(keep it job-scoped for least privilege; the workflow default stays
`contents: read` + `id-token: write`):

```yaml
  build:
    permissions:
      contents: read
      id-token: write          # OIDC to AWS (already required)
      security-events: write   # NEW: upload Trivy SARIF
```

### 4.2 Replace the single build-push step
Pin the Trivy action to a release tag (e.g. `@0.24.0`) and the DB is cached
automatically by recent versions; we also cache `~/.cache/trivy`.

```yaml
      - name: Build image (load to local daemon)
        uses: docker/build-push-action@v6
        with:
          context: ${{ steps.ctx.outputs.context }}
          load: true                     # build single-arch into docker, don't push yet
          tags: ${{ env.ECR_REGISTRY }}/${{ matrix.service }}:${{ needs.detect.outputs.sha }}
          provenance: false
          cache-from: type=gha,scope=${{ matrix.service }}
          cache-to: type=gha,scope=${{ matrix.service }},mode=max

      - name: Trivy scan → SARIF (report, non-blocking)
        uses: aquasecurity/trivy-action@0.24.0
        with:
          image-ref: ${{ env.ECR_REGISTRY }}/${{ matrix.service }}:${{ needs.detect.outputs.sha }}
          format: sarif
          output: trivy-${{ matrix.service }}.sarif
          severity: HIGH,CRITICAL
          ignore-unfixed: true
          exit-code: '0'                 # never fail here; this run is just for reporting

      - name: Upload SARIF to Security tab
        uses: github/codeql-action/upload-sarif@v3
        if: always()                     # upload even if the gate below fails
        with:
          sarif_file: trivy-${{ matrix.service }}.sarif
          category: trivy-${{ matrix.service }}   # per-service category → no overwrite

      - name: Trivy gate (fail on fixable CRITICAL)
        uses: aquasecurity/trivy-action@0.24.0
        with:
          image-ref: ${{ env.ECR_REGISTRY }}/${{ matrix.service }}:${{ needs.detect.outputs.sha }}
          format: table
          severity: CRITICAL
          ignore-unfixed: true
          exit-code: '1'                 # ← this is the gate

      - name: Push image
        run: docker push ${{ env.ECR_REGISTRY }}/${{ matrix.service }}:${{ needs.detect.outputs.sha }}
```

Notes:
- Two Trivy invocations share the cached DB, so the second is fast. The first
  produces SARIF and never fails; the second is the gate.
- `docker push` works because the image was `load`ed into the daemon and ECR
  login already happened earlier in the job.
- `load: true` builds a single platform (amd64) — fine for this project.

### 4.3 Optional: cache the Trivy DB explicitly
`trivy-action` caches the DB by default; to be safe across runners:

```yaml
      - uses: actions/cache@v4
        with:
          path: ~/.cache/trivy
          key: trivy-db-${{ runner.os }}
```

## 5. Triage & suppression

- Findings appear under **Security → Code scanning alerts**, grouped by the
  per-service `category`.
- To accept a specific CVE (false positive or risk-accepted), add it to a
  repo-root **`.trivyignore`** file (one CVE ID per line, with a comment and
  ideally an expiry note). Trivy reads it automatically.
- Prefer fixing via base-image bumps in the service `Dockerfile` over ignoring.

## 6. Performance / cost

- Vulnerability DB download is a few tens of MB; cached after first run.
- Adds roughly 20–60s per service leg (matrix runs in parallel, so wall-clock
  impact is small since we only build changed services).
- No AWS cost — scanning runs on the GitHub runner against the local image.

## 7. Rollout

- [ ] Add the steps above to `build.yml` (gate on CRITICAL, SARIF upload).
- [ ] **Soft-launch option:** first ship with the gate step's `exit-code: '0'`
      for one push to confirm SARIF appears in the Security tab and the noise
      level is acceptable, then flip to `exit-code: '1'` to enforce.
- [ ] Trigger a build (e.g. touch `src/frontend`) and confirm: SARIF appears in
      Security tab; a forced CRITICAL (test image) blocks push + deploy.
- [ ] Add a `.trivyignore` only if a legitimate, unfixable CRITICAL blocks work.

## 8. Verification checklist

- Security tab shows `trivy-<service>` categories with HIGH/CRITICAL findings.
- A clean image: `build` passes, image pushed, write-back PR merged, ArgoCD syncs.
- A CRITICAL-laden image: `build` fails at the gate, **no** push, `deploy`
  skipped, ArgoCD unchanged.

## 9. Future enhancements (out of scope here)

- **Trivy config/IaC scan** of `Dockerfile`s and `kubernetes-manifests/` for
  misconfigurations (non-root, resource limits, etc.).
- **Scheduled re-scan** of currently-deployed image tags to catch CVEs
  disclosed after deploy (cron workflow scanning the tags in `kustomization.yaml`).
- **ECR enhanced scanning** (Amazon Inspector) as a defense-in-depth comparison.
- SBOM generation (`trivy image --format cyclonedx`) stored as a build artifact.
