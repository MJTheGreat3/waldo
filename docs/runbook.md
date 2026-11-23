# Runbook

## P0: set up the server (M5 MacBook Pro)

Do these on the server Mac. Every manual step here should later move into an Ansible role.

### 1. Keep the Mac awake

```
sudo pmset -a sleep 0 disablesleep 1
caffeinate -dimsu &
```

Keep it on power.

### 2. Tools

```
brew install colima docker kubectl kind helm ansible tailscale jenkins-lts openjdk@21 gh
```

Start the container runtime with a memory cap that leaves room for macOS:

```
colima start --cpu 6 --memory 10 --disk 60 --arch aarch64
docker info | grep -i architecture   # expect aarch64
```

### 3. Tailscale

Sign in on all three machines. Confirm the teammate can reach the server:

```
tailscale status
ssh <user>@<server-tailscale-name>
```

### 4. kind cluster

```
kind create cluster --config deploy/kind/kind-config.yaml
kubectl config use-context kind-waldo
kubectl get nodes
```

The frontend NodePort 30080 is published on the Mac at port 8080, so `http://<server-tailscale-name>:8080` works from any tailnet machine.

### 5. Jenkins

```
brew services start jenkins-lts
```

Open `http://localhost:8080` is taken by the app, so move Jenkins to 8081:
edit `~/Library/LaunchAgents/homebrew.mxcl.jenkins-lts.plist`, set `--httpPort=8081`, then `brew services restart jenkins-lts`.

In Jenkins:

1. Install suggested plugins, plus GitHub, Pipeline: Multibranch, Credentials Binding.
2. Add credential `dockerhub` (username with password): the Docker Hub user and an access token, not the password.
3. New item, Multibranch Pipeline, source GitHub, repo `waldo`, script path `ci/Jenkinsfile`.
4. Enable "GitHub hook trigger for GITScm polling".
5. Make sure the `jenkins` user can run `docker`, `kubectl`, `helm` and `ansible-playbook` (PATH is set in the Jenkinsfile for Homebrew), and has the kubeconfig for `kind-waldo`.

### 6. Webhook through Tailscale Funnel

Expose only the webhook path, not the whole Jenkins UI:

```
tailscale funnel --bg --set-path /github-webhook/ http://localhost:8081/github-webhook/
tailscale funnel status
```

In GitHub, repo Settings, Webhooks:

- Payload URL: `https://<server>.<tailnet>.ts.net/github-webhook/`
- Content type: `application/json`
- Events: push and pull requests
- Set a secret and add the same secret in Jenkins.

If Funnel path restriction is not available in your Tailscale version, put a small nginx in front that only forwards `/github-webhook/`.

### 7. Acceptance test for P0

1. Edit `services/frontend/public/index.html` on a branch, open a PR, get a review, merge.
2. Jenkins builds, pushes `<dockerhub-user>/waldo-frontend:<sha>`, deploys with Ansible and Helm.
3. Refresh `http://<server-tailscale-name>:8080` from the teammate's laptop. The footer shows the new commit hash.

## Manual deploy (when Jenkins is down)

```
cd deploy/ansible
ansible-playbook site.yml -e dockerhub_user=<user> -e image_tag=<tag>
```

## Rebuild from scratch

```
kind delete cluster --name waldo
kind create cluster --config deploy/kind/kind-config.yaml
```

Then rerun the Jenkins job on `main`.
