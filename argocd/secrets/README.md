# Secrets — README

## Overview

No real secrets are stored in Git. All secrets are injected into the
cluster **before** ArgoCD syncs.

---

## Single-Repo Architecture

Everything lives in `shopmesh/shopmesh-infra`:

```
shopmesh-infra/
├── helm/                 ← Helm umbrella chart (Deployments, StatefulSet, etc.)
│   ├── environments/dev/values.yaml   ← CI writes dev image tags here (develop branch)
│   └── environments/prod/values.yaml  ← CI writes prod image tags here (main branch)
└── argocd/               ← ArgoCD Applications, AppProjects, bootstrap
```

ArgoCD only needs **one repo** registered.

---

## Pre-Deployment Checklist

### 1. Register the repo with ArgoCD

```bash
argocd login localhost:8080 --username admin --password <PASSWORD> --insecure

argocd repo add https://github.com/shopmesh/shopmesh-infra.git \
  --username YOUR_GITHUB_USERNAME \
  --password YOUR_GITHUB_PAT
```

---

### 2. Update NFS Configuration

The MongoDB chart requires an NFS server for persistent storage.
**Update before deploying** — edit both files:

```
helm/environments/dev/values.yaml  → mongodb.nfs.server and mongodb.nfs.path
helm/environments/prod/values.yaml → mongodb.nfs.server and mongodb.nfs.path
```

Example:
```yaml
mongodb:
  nfs:
    server: "172.31.24.10"       # ← your NFS server IP
    path: "/exports/mongodb-dev" # ← your NFS export path
```

---

### 3. Update Image Tags and Credentials

The environment values files ship with placeholder base64-encoded credentials.
Before deploying to production, re-encode real credentials:

```bash
# Encode a secret value
echo -n 'your-real-jwt-secret' | base64

# Encode a MongoDB URI
echo -n 'mongodb://admin:realpass@mongodb-0.mongodb-headless.prod.svc.cluster.local:27017/authdb?authSource=admin' | base64
```

Update the `secrets:` blocks in `helm/environments/prod/values.yaml`.

---

### 4. (Optional) GHCR Image Pull Secret

Only needed if your images are private on GHCR.
The current images (`priyatham753/*`) are on Docker Hub.
If you migrate to GHCR, create this in each namespace:

```bash
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=YOUR_GITHUB_USERNAME \
  --docker-password=YOUR_GITHUB_PAT \
  --namespace dev

kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=YOUR_GITHUB_USERNAME \
  --docker-password=YOUR_GITHUB_PAT \
  --namespace prod
```

---

## Recommended: Replace Inline Secrets with Sealed Secrets

The current approach stores base64-encoded secrets in `values.yaml`.
Base64 is **not encryption** — anyone with repo access can decode them.

For production, migrate to one of:

| Tool | Description |
|------|-------------|
| [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) | Encrypt secrets with a cluster-side key, commit ciphertext to Git |
| [External Secrets Operator](https://external-secrets.io/) | Sync from AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault |
| [ArgoCD Vault Plugin](https://argocd-vault-plugin.readthedocs.io/) | Inject secrets during ArgoCD sync from Vault |

---

## CI — How to Update Image Tags

Your CI pipeline must do the following after building and pushing an image:

```bash
# Example for auth-service on the develop branch (dev deploy)
SERVICE=auth-service
NEW_TAG="dev-$(git rev-parse --short HEAD)"
ENV_FILE="helm/environments/dev/values.yaml"

git clone https://x-access-token:${INFRA_PAT}@github.com/shopmesh/shopmesh-infra.git
cd shopmesh-infra
git checkout develop

# Update the image tag (requires yq installed)
yq e ".\"${SERVICE}\".image.tag = \"${NEW_TAG}\"" -i ${ENV_FILE}

git config user.email "ci@shopmesh.io"
git config user.name "ShopMesh CI"
git add ${ENV_FILE}
git commit -m "ci: update ${SERVICE} image to ${NEW_TAG} [skip ci]"
git push origin develop
```

ArgoCD detects the push within 3 minutes and auto-syncs the dev namespace.

For **prod**, replace `develop` → `main` and `dev` → `prod`.
