# Waldo

Lost and found with image-based matching, delivered through an end-to-end DevSecOps and MLOps pipeline.

Runs on a local kind cluster on a second MacBook, reached over Tailscale. A push to GitHub triggers Jenkins, which tests, builds, pushes to Docker Hub and deploys with Ansible and Helm.

## Status

Phase P0 (walking skeleton): a hello-world frontend goes through the whole pipeline. The version in the page footer changes on every deploy.

## Layout

```
services/frontend/   nginx page with a version footer (more services come in P1)
deploy/helm/waldo/   Helm chart
deploy/ansible/      site.yml and roles (helm_deploy so far)
deploy/kind/         kind cluster config
ci/Jenkinsfile       pipeline
docs/runbook.md      server setup and operations
```

## Run locally

```
docker compose up --build
open http://localhost:8080
```

## Contributing

- No direct pushes to `main`. Open a pull request, get one review, wait for a green Jenkins check.
- Branches: `feat/<area>-<thing>`, `fix/...`, `chore/...`, `docs/...`.
- Conventional Commits, squash merge, PRs under about 300 changed lines.
- Never commit secrets.

Server setup is in [docs/runbook.md](docs/runbook.md).
