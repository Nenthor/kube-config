# How to initialize the cluster

This guide describes the initial setup and how to deploy everything in this repository with ArgoCD.

## Install k3s

On first master:

```
curl -sfL https://get.k3s.io | K3S_TOKEN="secret_join_token" sh -s - server \
  --cluster-init \
  --disable local-storage \
  --disable traefik \
  --disable servicelb
```

On other masters:

```
curl -sfL https://get.k3s.io | K3S_TOKEN="secret_join_token" sh -s - server \
  --server https://<IP_OF_NODE_1>:6443 \
  --disable local-storage \
  --disable traefik \
  --disable servicelb
```

## Setup ArgoCD

```
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl wait --namespace argocd --for=condition=ready pod --selector=app.kubernetes.io/name=argocd-server --timeout=90s

kubectl create secret generic git-repo-secret -n argocd \
  --from-literal=type=git \
  --from-literal=url=ssh://git@git.nentwich.dev:2222/nentwich/kube-config.git \
  --from-file=sshPrivateKey=/home/david/Documents/Keys/Server/Proxmox/backup/argocd-deploy-keys/id_rsa

kubectl label secret git-repo-secret -n argocd argocd.argoproj.io/secret-type=repository

kubectl apply -f root-app.yaml
```

## Logging into ArgoCD UI

**Get initial admin password:**

```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

**Port forward to [http://127.0.0.1:8080](http://127.0.0.1:8080):**

```
kubectl port-forward svc/argocd-server -n argocd 8080:80
```

## Setup sealed-secrets

Install kubeseal cli:

```
curl -OL "https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.37.0/kubeseal-0.37.0-linux-amd64.tar.gz"
tar -xvzf kubeseal-0.37.0-linux-amd64.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

Encrypt `secret.yaml` to `sealed-secret.yaml`:

```
kubeseal -o=yaml < secret.yaml > sealed-secret.yaml
```

**BACKUP sealing certificate:**

1. User kubernetes vscode extension
2. Go to `kube-system` namespace
3. find `sealed-secrets-keyxxx` under `Configurations/Secrets`
4. Export `tls.crt` and `tls.key` to somewhere save

## Verify setup

Go to ArgoCD UI and everything should be Synced and Healthy
