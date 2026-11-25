# Waldo project plan

Lost and found platform with image-based matching, delivered through a full DevSecOps and MLOps pipeline. This is the SPE (CSE 816) final project. The requirements live in [srs/README.md](srs/README.md); this file says how and in what order we build them.

Where this plan and the SRS disagree, this plan wins. The differences are:

- Docker Hub instead of GHCR (the course rubric asks for Docker Hub).
- ELK added for logs (rubric). Prometheus and Grafana stay for metrics.
- HashiCorp Vault for runtime secrets (rubric). Ansible Vault only for bootstrap variables.
- Jenkins runs on the server Mac and deploys locally. Provisioning roles run from a dev laptop over Tailscale SSH.

## Rubric mapping

| Rubric item | Where it is covered |
| --- | --- |
| Jenkins with GitHub hook trigger | P0, webhook through Tailscale Funnel |
| Docker and Compose | P1 |
| Ansible playbooks, roles for the advanced mark | P0 `helm_deploy`, P1 roles, P6 cleanup |
| Kubernetes | kind on the server Mac, Helm charts |
| ELK with a Kibana dashboard | P5 |
| Push to Docker Hub | P0 |
| UI change visible on refresh | P0 exit gate, demo step 7 |
| Vault | P6 |
| HPA | P6 |
| Live patching | P6, rolling update with readiness probes |
| Innovation | matcher model gate and canary release (P3), signed images with Kyverno (P4) |
| MLOps and DevSecOps domain marks | P3 and P4 |

## Infrastructure

- **Server Mac** (M5 Pro, 16 GB): Jenkins (native, port 8081), Colima capped at 10 GB, kind, Vault, ELK, Prometheus, Grafana, Tailscale with Funnel. Funnel only exposes `/github-webhook/`.
- **Dev laptops**: code and open PRs only. Images are arm64.
- Everything reaches the server over Tailscale. Setup steps are in [runbook.md](runbook.md).

## Stack (proposed, to confirm with Dheeraj)

- Frontend: React with Vite, served by nginx
- API: FastAPI, modular monolith
- Worker: RQ on Redis
- Matcher: FastAPI service with a small pretrained ONNX embedding model
- Data: Postgres with pgvector, MinIO for images
- Auth: cookie sessions with CSRF protection

## Pipeline

Runs on every push. Stages after "Push to Docker Hub" only run on `main`.

1. Gitleaks (secrets)
2. Lint and unit tests, Hadolint on Dockerfiles
3. Model gate (P3 onwards)
4. Semgrep
5. Trivy filesystem and config scan
6. Build images
7. Trivy image scan
8. Syft SBOM
9. Push to Docker Hub, tag is the git SHA
10. Cosign sign and attest (main)
11. Ansible and Helm deploy (main)
12. Smoke test (main)
13. ZAP baseline (main)

Kyverno `verifyImages` on the cluster rejects unsigned images. Deploys are rolling (`maxUnavailable: 0`) with readiness probes, which is what makes live patching work.

## Phases

### P0: Walking skeleton

Repo, CODEOWNERS and PR template are done, along with a hello-world frontend, Helm chart, Ansible `helm_deploy` role and a Jenkinsfile.

Still to do: on the server Mac, set up Tailscale on all machines, Colima, kind, Jenkins, Funnel, the Docker Hub token and the GitHub webhook. In the repo, set branch protection once Jenkins reports a check, and write ADR-005.

- SRS: NFR-004, NFR-014, NFR-020, AC-11, ADR-005
- Exit: a merged PR changes the footer version on Dheeraj's laptop with no manual step.

### P1: App skeletons

Every service gets a multi-stage Dockerfile, `.dockerignore`, non-root user, pinned base image, health and metrics endpoints and a Helm subchart.

- Services: api (`/health/live`, `/health/ready`, `/metrics`), worker, matcher (`/health`, `/ready`, `/model-info`, `/metrics`), Postgres with pgvector, Redis, MinIO.
- Also: compose file for local dev, DB migration tool, structured JSON logs with request ids, Hadolint in CI, unit-test scaffolding, Makefile, reduced-resource profile doc, ADR-001.
- Ansible roles: `docker`, `kind`, `jenkins`, `helm_deploy`.
- SRS: FR-027, FR-028 (endpoints), NFR-013, NFR-015, NFR-016, NFR-020, AC-10
- Exit: `docker compose up` works locally and the pipeline deploys all services to kind with probes green.

### P2: Core product

Two sub-phases, each shipped as small vertical slices.

**P2a: accounts, listings, search**

- Roles (visitor, user, moderator, admin), cookie sessions, CSRF, rate limits.
- Listings with the lifecycle, upload validation and re-encoding, MinIO storage, browse, search, filters, pagination.
- Guided create form with photo preview, profile, account deletion, accessibility basics.
- ADR-003, ADR-004, ADR-006.
- SRS: FR-001 to FR-011, FR-030, NFR-001, 002, 007, 011, 012, 017, 018, 019. AC-01, 02, 03.

**P2b: claims and moderation**

- Claim flow with private proof; accept, reject or ask for info; resolve.
- Report listing, moderator queue, audit trail, in-app inbox, admin settings.
- Object-level authorization tests.
- SRS: FR-018 to FR-026, NFR-008 to 010. AC-07, 08, 09.

Exit: the full lost, found, claim, resolve and moderation story works on the cluster.

### P3: Matching and MLOps

- Pretrained embedding model. ADR-002 for model and licence, ADR-007 for weights and thresholds.
- Async match jobs with bounded retries, idempotency, and invalidation when a listing is edited.
- pgvector candidate ranking, score and confidence bands in the UI, feedback buttons.
- Match notifications (FR-017).
- Labelled set of 30 to 100 pairs, eval script, CI model gate.
- Inference metrics, versioned model layer, canary matcher release.
- SRS: FR-012 to FR-017, FR-029, NFR-003, NFR-006. AC-04, 05, 06, 14.
- Exit: the gate fails on a deliberately bad model and passes on the good one. A similar photo ranks high and a dissimilar one does not.

### P4: DevSecOps gates

One gate per PR, report-only first and then blocking: Gitleaks, Semgrep, Trivy fs, Trivy config, Hadolint, Trivy image, Syft, Cosign, Kyverno, ZAP.

- Hardening: read-only root filesystem, dropped capabilities, resource limits, NetworkPolicies (Calico, or document the limitation).
- ADR-008. Scan reports archived in Jenkins.
- SRS: section 9.2, section 10.1
- Exit: an unsigned image and an image with a known CRITICAL CVE are both rejected.

### P5: Observability

- ELK: Filebeat, Elasticsearch (single node, 512 MB heap), Kibana log dashboard.
- Prometheus and Grafana: application dashboard, infrastructure dashboard, alerts for service down, high error rate and queue backlog.
- Metrics from the SRS: request counts, matching jobs, inference duration, queue depth, pod health.
- ADR-009.
- SRS: NFR-015, section 10.4. AC-12.
- Exit: Kibana shows live app logs, Grafana shows traffic and matcher latency, and an injected failure fires the alert.

### P6: Advanced and resilience

- Vault with AppRole for Jenkins and Ansible. ADR-010.
- Ansible roles cleaned up.
- HPA on API and matcher, with a load test.
- Live patch: a deploy during traffic with zero failed requests.
- Failure injection from the SRS: matcher, storage, DB, worker, bad release.
- SRS: NFR-004, NFR-005, section 11. AC-13.
- Exit: the load test scales pods up and back down, and each failure scenario behaves as the SRS says.

### P7: Freeze and demo

- Feature freeze.
- k6 or Locust run against NFR-001 and NFR-002, backup and restore test, rebuild-from-scratch test of the server Mac.
- Demo script rehearsed twice, backup demo recording, final report with rubric and SRS traceability.
- SRS: AC-15, AC-16.
- Exit: a clean rebuild reaches a working demo and the recording exists.

**Cut order if time runs out:** Should-priority items go first (FR-003, FR-016, FR-025, FR-026, FR-029, FR-030, and the Should NFRs). The mandatory rubric items stay.

## Git and PR workflow

- Trunk-based. Short branches off `main`, PR-only merges.
- One review from the other person, and a green Jenkins check.
- Squash merge. Conventional Commit style, short messages.
- PRs under about 300 lines.
- CODEOWNERS makes each of us review the other's area.
- Branch protection gets switched on once Jenkins reports a status check.

## Work split

- MJTheGreat3: Ansible, Jenkins, Kyverno, Vault, server Mac, CI.
- Dheeraj-Murthy: services, ML, frontend.
- Helm charts and docs are shared.

## Risks

| Risk | Plan |
| --- | --- |
| 16 GB is tight for ELK plus kind plus Jenkins | Colima cap of 10 GB, ES heap 512 MB, scale ELK to zero when not demoing |
| Server Mac sleeps or Tailscale drops | `pmset` and `caffeinate`, keep on power, runbook for manual deploy |
| Funnel path restriction unavailable | small nginx in front forwarding only `/github-webhook/` |
| Matcher model too heavy or inaccurate | small ONNX model, labelled set and gate catch regressions early |
| Time runs out | cut order above, mandatory rubric items protected |

## Demo script

1. Show the app and Grafana.
2. User A reports a lost item.
3. User B reports a found item.
4. Async match runs and a ranked suggestion appears.
5. Claim, accept, resolve.
6. Open a PR that changes the footer, merge it.
7. Show the Jenkins stages, then refresh and see the new version with no downtime.
8. Kibana shows the logs.
9. Matcher version and inference metrics.
10. Delete the matcher pod and watch it recover.
11. Kyverno rejects an unsigned image.
12. Load test, HPA scales.
13. Vault and the canary release.
14. Close with the architecture and the Git history.

## Open decisions

Confirm with Dheeraj: React, FastAPI, cookie sessions, MinIO, Redis with RQ, pgvector, and the embedding model.
