# How to initialize the cluster

This guide describes the initial setup and how to deploy everything in this repository with ArgoCD.

## Install k3s

```
curl -sfL https://get.k3s.io |  sh -s - server \
  --disable traefik \
  --disable servicelb
```

## Copy kube config

Copy `cat /etc/rancher/k3s/k3s.yaml` and save it under `~/.kube/config`. Change the ip inside the config to the actual ip. You should then be able to use `kubectl`.

## Restore SealedSecret key (only on restore)

```
kubectl create secret tls sealed-secrets-key \
  --cert=/home/david/Documents/Keys/Server/Proxmox/backup/sealed-secrets/tls.crt \
  --key=/home/david/Documents/Keys/Server/Proxmox/backup/sealed-secrets/tls.key \
  -n kube-system

kubectl label secret sealed-secrets-key -n kube-system sealedsecrets.bitnami.com/sealed-secrets-key=active
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
