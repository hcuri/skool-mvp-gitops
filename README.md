# Skool MVP GitOps

ArgoCD + environment configuration for Skool MVP. Holds ArgoCD `Application` manifests and per-environment Helm values.

## Layout

- `applications/`
  - `skool-mvp-app.yaml` – ArgoCD Application pointing to the app Helm chart in `skool-mvp-app` repo.
- `environments/`
  - `dev/skool-mvp-app/values.yaml` – Dev values (image tag, replicas, service type, env secret name).
  - `prod/skool-mvp-app/values.yaml` – Reference prod-style values (not deployed yet).

## How to deploy with ArgoCD

1) Install ArgoCD in the cluster (namespace `argocd`).
2) Apply the Application:
   ```bash
   kubectl apply -f applications/skool-mvp-app.yaml
   ```
3) In ArgoCD UI/CLI, sync `skool-mvp-app`:
   - Source: `skool-mvp-app` repo (`charts/skool-mvp-app`) with `environments/dev/...` values.
   - Destination: same cluster, namespace `apps` (created if missing).

## Notes / Next steps
- Secrets: `skool-mvp-db` is expected to exist in the target namespace; not managed here.
- Image bumps: update `environments/dev/skool-mvp-app/values.yaml` (or prod) with a new tag; ArgoCD sync will roll the deployment.
- Future: add observability Application (e.g., kube-prometheus-stack) and additional services; optionally add an app-of-apps/ApplicationSet for multiple apps.
