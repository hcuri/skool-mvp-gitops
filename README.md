# Skool MVP GitOps

Built by **Hector Curi** as a personal project to demonstrate end-to-end DevOps/SRE skills (GitOps, ArgoCD, Helm, Kubernetes).

ArgoCD Applications and environment-specific Helm values for deploying the Skool MVP API to EKS using GitOps.

## How this fits into the Skool MVP
- ArgoCD Applications and environment values for the Skool MVP API backend
- API code + chart are in `skool-mvp-api` (https://github.com/hcuri/skool-mvp-api)
- AWS infra (EKS, RDS, VPC) is in `skool-mvp-infra` (https://github.com/hcuri/skool-mvp-infra).

## Layout
- `applications/`
  - `skool-mvp-api.yaml` – ArgoCD Application pointing to the API chart.
  - `aws-load-balancer-controller.yaml` – Installs AWS Load Balancer Controller (ALB ingress).
  - `ingress.yaml` – Applies dev Ingress manifests (host-based routing + TLS).
  - `environments/`
  - `dev/skool-mvp-api/values.yaml` – Dev values (image tag, replicas, service type, env secret name).
  - `prod/skool-mvp-api/values.yaml` – Reference prod-style values (not deployed yet).
  - `dev/ingress/` – Ingress manifests for `api.skoo1.com` and `grafana.skoo1.com`.

## How ArgoCD uses this repo
- Multi-repo setup: ArgoCD pulls the chart from `skool-mvp-api` and the values from this repo using a second source (`$skool-mvp-gitops/.../values.yaml`).
- Updating the image tag in the values file and committing here causes ArgoCD to roll out a new version (images are tagged with Git SHAs).

## Deploying with ArgoCD (manual apply)
1) Install ArgoCD in the cluster (namespace `argocd`).
2) Apply the Application:
   ```bash
   kubectl apply -f applications/skool-mvp-api.yaml
   ```
3) In ArgoCD UI/CLI, sync `skool-mvp-api`:
   - Source: `skool-mvp-app` repo (`charts/skool-mvp-api`) with `environments/dev/...` values from this repo.
   - Destination: same cluster, namespace `apps` (created if missing).

## Workflow example
1) Code change in `skool-mvp-api` → build & push image tagged with Git SHA (repo `hcuri/skool-mvp-api`).
2) CI opens a PR in this repo updating `environments/dev/skool-mvp-api/values.yaml` `image.tag` to that SHA; approve/merge.
3) ArgoCD syncs and deploys to the `apps` namespace in EKS.

## Deployment strategies
Right now `skool-mvp-api` is deployed as a standard Kubernetes Deployment and exposed via a Service of type `LoadBalancer`. ArgoCD applies the rendered Helm manifests, and Kubernetes handles the rollout strategy.

### Current strategy – Rolling updates (default)

- ArgoCD applies a new image tag via GitOps; Kubernetes:
  - Creates a new ReplicaSet for the updated Pod template.
  - Gradually scales up the new ReplicaSet and scales down the old one.
- Old ReplicaSets are kept with 0 replicas as rollout history (`revisionHistoryLimit` on the Deployment).
- Benefits: no full outage during deploys; rollback via `kubectl rollout undo` or reverting in Git and letting ArgoCD sync.
- Chart can expose `revisionHistoryLimit`, `maxSurge`, and `maxUnavailable` as values to tune per environment.

### Blue/green (how we would do it here)

- Define two Deployments in the chart: `skool-mvp-api-blue` and `skool-mvp-api-green`.
- Use a single Service `skool-mvp-api` whose selector targets either blue or green (e.g., `version=blue` or `version=green`).
- Add a value such as `activeColor` in the Helm values (managed via GitOps), e.g.:
  ```
  activeColor: blue
  ```
Flow:
1) Blue live (`activeColor=blue`), Green idle.
2) Deploy new version to Green via GitOps (ArgoCD syncs).
3) Validate Green (health, smoke tests).
4) Flip `activeColor` to `green` in GitOps values; merge PR.
5) ArgoCD syncs; Service selector switches from blue to green; traffic moves instantly.
6) Optionally scale down Blue after confidence; keep config for rollback.
Rollback: change `activeColor` back to `blue` and let ArgoCD sync.

### Canary (future extension)

- Introduce Argo Rollouts and replace the Deployment with a Rollout for `skool-mvp-api`.
- Configure canary steps (e.g., 5% → 25% → 50% → 100%) with pauses/metric checks.
- Use an ingress/mesh that supports weighting (NGINX Ingress, AWS ALB Ingress Controller, Istio/Linkerd).
- Manage Rollout spec and image tags via GitOps; ArgoCD syncs desired state; Rollouts controller shifts traffic.
For this MVP, we stick with rolling updates but can add blue/green or canary incrementally as needed.

## Observability
- ArgoCD deploys `kube-prometheus-stack` (Prometheus, Alertmanager, Grafana) via `applications/observability.yaml` into the `observability` namespace.
- `skool-mvp-api` exposes `/metrics` (Prometheus format) and is scraped via a ServiceMonitor (enabled in `environments/dev/skool-mvp-api/values.yaml`).
- Grafana is exposed via ALB Ingress at `https://grafana.skoo1.com` (Service is `ClusterIP`).

## Ingress + TLS (AWS-native)
The dev environment uses an AWS-native ingress setup:

- **Ingress controller**: AWS Load Balancer Controller (installed via `applications/aws-load-balancer-controller.yaml`).
- **Load balancer**: a single internet-facing **ALB** shared via Ingress group `skool-mvp-dev`.
- **TLS termination**: ALB terminates TLS using an ACM certificate in `us-west-2`.
- **HTTP→HTTPS**: handled at ALB using `alb.ingress.kubernetes.io/ssl-redirect: "443"`.
- **Host-based routing**:
  - `api.skoo1.com` → `skool-mvp-api` (namespace `apps`)
  - `grafana.skoo1.com` → Grafana (namespace `observability`)
- **ArgoCD**: intentionally not exposed publicly (use port-forward only).

### DNS (Cloudflare)
The domain `skoo1.com` is hosted in Cloudflare DNS for this demo.

- ACM DNS validation records are created **manually** in Cloudflare based on Terraform outputs from `skool-mvp-infra`.
- Once the ALB exists, create **DNS-only (grey cloud)** CNAMEs in Cloudflare:
  - `api.skoo1.com` → `<alb-dns-name>`
  - `grafana.skoo1.com` → `<alb-dns-name>`
  - (later) `web.skoo1.com` → `<alb-dns-name>`

Important: keep these records **DNS-only** so TLS terminates at the ALB using ACM (not at Cloudflare).

## CI/CD note
- API repo (`skool-mvp-api`): GitHub Actions builds/pushes `hcuri/skool-mvp-api:${{ github.sha }}` on `main` and opens a PR here to bump the dev image tag.
- GitOps repo (`skool-mvp-gitops`): review/merge the bump PR to roll the deployment via ArgoCD.

## Security / secrets
- Secrets are not stored here. `DATABASE_URL` and other sensitive data live in Kubernetes Secrets created separately (e.g., `skool-mvp-db`).
