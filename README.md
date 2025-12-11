# Skool MVP GitOps

ArgoCD and environment configuration repo. Holds ArgoCD Applications plus per-environment values/overlays for deploying the Skool MVP app and supporting stacks (e.g., observability).

## Planned layout

- `applications/` – ArgoCD Application manifests (app-of-apps or per-app).
- `environments/dev/` – values/overlays for dev (app + observability).
- `environments/prod/` – pattern only (optional for MVP).

## Status

- Planning; structure stubbed here. Will be populated once charts/infra are ready.
