# Skool MVP GitOps

Built by **Hector Curi** as a personal project to demonstrate end-to-end DevOps/SRE skills (GitOps, ArgoCD, Helm, Kubernetes).

ArgoCD Applications and environment-specific Helm values for deploying the Skool MVP API to EKS using GitOps.

## Role in the Skool MVP
- Contains ArgoCD `Application` manifests (e.g., `applications/skool-mvp-api.yaml`).
- Contains env-specific values under `environments/dev/skool-mvp-api/values.yaml` (and a prod reference).
- App code + chart are in `skool-mvp-app` (API backend); AWS infra (EKS, RDS, VPC) is in `skool-mvp-infra`.

## How ArgoCD uses this repo
- Multi-repo setup: ArgoCD pulls the chart from `skool-mvp-app` and the values from this repo using a second source (`$skool-mvp-gitops/.../values.yaml`).
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
1) Code change in `skool-mvp-app` → build & push image tagged with Git SHA (repo `hcuri/skool-mvp-api`).
2) Update `environments/dev/skool-mvp-api/values.yaml` `image.tag` to that SHA and commit/push to this repo.
3) ArgoCD syncs and deploys to the `apps` namespace in EKS.

## Security / secrets
- Secrets are not stored here. `DATABASE_URL` and other sensitive data live in Kubernetes Secrets created separately (e.g., `skool-mvp-db`).

## License & attribution
- Licensed under the MIT License (see `LICENSE`).
- You are welcome to copy/adapt the code. If you reuse it, please preserve the original copyright and license notices so attribution to the author, Hector Curi, is retained.
