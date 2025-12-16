# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a GitOps repository managing Kubernetes infrastructure for the TaskTime application using ArgoCD and Helm charts. The repository follows a declarative infrastructure approach where all infrastructure components are defined as Helm charts and deployed via ArgoCD Applications.

## Architecture

### Repository Structure

- `argo-cd/`: ArgoCD installation and project configuration
  - `project.yaml`: Defines the ArgoCD AppProject "tasktime-cd" with source repos and destination namespaces
  - Helm chart wrapping the official ArgoCD chart (version 9.1.6)

- `postgres/`: PostgreSQL database infrastructure
  - `app.yaml`: ArgoCD Application manifest for postgres-dev
  - Helm chart wrapping Bitnami PostgreSQL (version 18.1.13)
  - Deployed to `tasktime-cd` namespace
  - Service name: `postgres-dev-postgresql.tasktime-cd.svc.cluster.local`

- `tasktime/`: TaskTime application deployment
  - `app.yaml`: ArgoCD Application manifest for tasktime-dev
  - Custom Helm templates for Deployment, Service, and ConfigMap
  - Deployed to `tasktime` namespace
  - Uses images from `ghcr.io/clarenced/`
  - Service exposed via NodePort on port 31000

### GitOps Workflow

All infrastructure is managed through ArgoCD Applications pointing to this repository:
- Source: `https://github.com/clarenced/tasktime-cd`
- Target branch: `main`
- Automated sync with prune and self-heal enabled

### Namespace Organization

- `argo-cd`: ArgoCD control plane
- `tasktime-cd`: Infrastructure components (PostgreSQL)
- `tasktime`: Application workloads

### Database Configuration

PostgreSQL configuration for dev environment:
- Database name: `tasktime-database` (note: there's a duplicate `database` key in postgres/values-dev.yaml:14,16)
- Username: `tasktime-user`
- Connection URL: `jdbc:postgresql://postgres-dev-postgresql.tasktime-cd.svc.cluster.local:5432/tasktime`
- 5Gi persistent volume
- Backup disabled in dev

## Common Commands

### Managing Helm Dependencies

```bash
# Update Helm dependencies for a chart
cd argo-cd/ && helm dependency update
cd postgres/ && helm dependency update

# Build dependencies (downloads .tgz files to charts/ directory)
helm dependency build
```

### Working with ArgoCD

```bash
# Apply ArgoCD project
kubectl apply -f argo-cd/project.yaml

# Apply application manifests
kubectl apply -f postgres/app.yaml
kubectl apply -f tasktime/app.yaml

# Check ArgoCD application status
kubectl get applications -n argo-cd

# Manually sync an application
argocd app sync postgres-dev
argocd app sync tasktime-dev
```

### Testing Helm Templates Locally

```bash
# Render templates without installing
helm template postgres ./postgres -f postgres/values-dev.yaml
helm template tasktime ./tasktime -f tasktime/templates/values-dev.yaml

# Validate manifests
helm lint ./postgres
helm lint ./tasktime
```

### Debugging

```bash
# Check ArgoCD application details
kubectl describe application postgres-dev -n argo-cd
kubectl describe application tasktime-dev -n argo-cd

# View PostgreSQL pods
kubectl get pods -n tasktime-cd

# View TaskTime application pods
kubectl get pods -n tasktime

# Access TaskTime service (NodePort)
# Application is exposed on port 31000
```

## Important Notes

- Helm chart dependencies are committed as .tgz files in `charts/` directories (excluded from git via .gitignore)
- The tasktime/app.yaml references `values-dev.yml` but the actual file is at `tasktime/templates/values-dev.yaml`
- ConfigMap template uses Helm templating but values file is in templates/ directory (non-standard location)
- Database credentials are currently hardcoded in values files (not using secrets)
