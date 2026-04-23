# taskapp-argocd

GitOps configuration for deploying the taskapp stack to Kubernetes using ArgoCD.

## Overview

This repository implements the [App-of-Apps](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/) pattern. A single root `Application` points ArgoCD at a Helm chart (`apps/`) that bootstraps all child applications. Environment-specific configuration is handled via separate Helm value files for `dev` and `prod`.

Helm charts for the individual components live in the companion repo: [`taskapp-helmcharts`](https://github.com/boicotaz/taskapp-helmcharts).

## Repository Structure

```
taskapp-argocd/
├── root-dev.yaml               # Root ArgoCD Application for dev
├── root-prod.yaml              # Root ArgoCD Application for prod
└── apps/
    ├── Chart.yaml              # Helm chart metadata
    ├── values.yaml             # Shared default values
    ├── values-dev.yaml         # Dev environment overrides
    ├── values-prod.yaml        # Prod environment overrides
    └── templates/
        ├── external-secrets-app.yaml         # External Secrets Operator (sync wave 0)
        ├── external-secrets-config-app.yaml  # ESO ClusterSecretStore / config (sync wave 1)
        ├── database-app.yaml                 # Database (sync wave 2)
        ├── backend-app.yaml                  # Backend service (sync wave 2)
        └── frontend-app.yaml                 # Frontend service (sync wave 2)
```

## Applications

| App | Namespace | Sync Wave | Description |
|---|---|---|---|
| `external-secrets` | `external-secrets` | 0 | External Secrets Operator (v0.14.4) |
| `external-secrets-config` | `external-secrets` | 1 | ClusterSecretStore and ESO configuration |
| `taskapp-database` | `default` | 2 | Database deployment |
| `taskapp-backend` | `default` | 2 | Backend API |
| `taskapp-frontend` | `default` | 2 | Frontend UI |

Sync waves ensure External Secrets Operator and its configuration are ready before the application components are deployed.

## Environments

| Environment | Secret Path | Root Manifest |
|---|---|---|
| `dev` | `taskapp/dev/database` | `root-dev.yaml` |
| `prod` | `taskapp/prod/database` | `root-prod.yaml` |

## Bootstrap

Apply the root manifest for the target environment to bootstrap the entire stack:

```bash
# Dev
kubectl apply -f root-dev.yaml

# Prod
kubectl apply -f root-prod.yaml
```

ArgoCD will pick up the root `Application` and automatically create all child applications. All apps are configured with `automated` sync, `selfHeal: true`, and `prune: true`.

## Notifications

Deployment events are sent to the `#deployments` Slack channel via [ArgoCD Notifications](https://argocd-notifications.readthedocs.io/). The following events are subscribed for backend and database apps:

- `on-sync-succeeded`
- `on-sync-failed`
- `on-smoke-test-failed` (backend only)
