# Flux GitOps Repository

This repository contains Flux configurations to automatically deploy and manage:
- cert-manager (for Let's Encrypt certificates)
- OpenSearch Operator

## Prerequisites

- Kubernetes cluster (v1.24+)
- kubectl configured to access your cluster
- Flux CLI installed ([installation guide](https://fluxcd.io/flux/installation/))
- Git repository (this repo) pushed to GitHub, GitLab, or any Git provider

## Repository Structure

```
.
├── clusters/
│   └── production/              # Production cluster configuration
│       ├── flux-system/         # Flux system components (auto-generated)
│       └── infrastructure.yaml  # Infrastructure deployment definition
├── infrastructure/
│   ├── controllers/             # Kubernetes controllers
│   │   ├── cert-manager/        # cert-manager Helm chart configuration
│   │   └── opensearch-operator/ # OpenSearch Operator Helm chart
│   └── configs/                 # Configuration resources
│       └── cert-manager/        # Let's Encrypt ClusterIssuers
└── README.md
```

## Installation Steps

### 1. Configure Let's Encrypt Email

Before deploying, update the email address in the Let's Encrypt configurations:

Edit the following files and replace `your-email@example.com` with your actual email:
- `infrastructure/configs/cert-manager/letsencrypt-staging.yaml`
- `infrastructure/configs/cert-manager/letsencrypt-production.yaml`

### 2. Push Repository to Git

```bash
git init
git add .
git commit -m "Initial Flux configuration"
git remote add origin <your-git-repo-url>
git branch -M main
git push -u origin main
```

### 3. Bootstrap Flux on Your Cluster

For GitHub:
```bash
flux bootstrap github \
  --owner=<github-username> \
  --repository=<repo-name> \
  --branch=main \
  --path=clusters/production \
  --personal
```

For GitLab:
```bash
flux bootstrap gitlab \
  --owner=<gitlab-username> \
  --repository=<repo-name> \
  --branch=main \
  --path=clusters/production \
  --personal
```

For generic Git:
```bash
flux bootstrap git \
  --url=<git-repo-url> \
  --branch=main \
  --path=clusters/production
```

### 4. Verify Deployment

Check Flux is running:
```bash
flux check
```

Watch the reconciliation:
```bash
flux get kustomizations --watch
```

Check cert-manager deployment:
```bash
kubectl get pods -n cert-manager
kubectl get clusterissuers
```

Check OpenSearch Operator deployment:
```bash
kubectl get pods -n opensearch-operator-system
```

## What Gets Deployed

### cert-manager
- **Namespace**: `cert-manager`
- **Purpose**: Manages SSL/TLS certificates
- **Version**: Latest v1.13.x
- **Includes**:
  - cert-manager controller
  - Let's Encrypt staging issuer (for testing)
  - Let's Encrypt production issuer

### OpenSearch Operator
- **Namespace**: `opensearch-operator-system`
- **Purpose**: Manages OpenSearch clusters
- **Version**: Latest v2.x
- **Resources**:
  - CPU: 100m-500m
  - Memory: 128Mi-512Mi

## Using Let's Encrypt Certificates

Once deployed, you can request certificates in your Ingress resources:

### Staging (for testing):
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-staging"
spec:
  tls:
    - hosts:
        - example.com
      secretName: example-tls
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: example-service
                port:
                  number: 80
```

### Production:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-production"
spec:
  tls:
    - hosts:
        - example.com
      secretName: example-tls
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: example-service
                port:
                  number: 80
```

## Deployment Order

Flux automatically handles dependencies:
1. **infrastructure-controllers** deploys first (cert-manager + OpenSearch Operator)
2. **infrastructure-configs** deploys after controllers are ready (Let's Encrypt issuers)

## Customization

### Change cert-manager version
Edit `infrastructure/controllers/cert-manager/helmrelease.yaml` and modify the version constraint.

### Change OpenSearch Operator version
Edit `infrastructure/controllers/opensearch-operator/helmrelease.yaml` and modify the version constraint.

### Add more infrastructure
Create new directories under `infrastructure/controllers/` or `infrastructure/configs/` and add them to the respective `kustomization.yaml` files.

## Troubleshooting

Check Flux logs:
```bash
flux logs --all-namespaces --follow
```

Check specific Kustomization:
```bash
flux get kustomizations infrastructure-controllers
flux describe kustomization infrastructure-controllers
```

Check HelmRelease status:
```bash
flux get helmreleases --all-namespaces
```

Force reconciliation:
```bash
flux reconcile kustomization infrastructure-controllers --with-source
```

## Uninstall

To remove everything:
```bash
flux uninstall --namespace=flux-system
```

Note: This will not remove the CRDs or PVCs. To completely clean up:
```bash
kubectl delete namespace cert-manager
kubectl delete namespace opensearch-operator-system
```
