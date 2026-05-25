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

```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
kubectl port-forward svc/argocd-server -n argocd 8080:443

```

## Setup sealed-secrets

Install kubeseal cli:

```
curl -OL "https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.37.0/kubeseal-0.37.0-linux-amd64.tar.gz"
tar -xvzf kubeseal-0.37.0-linux-amd64.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

deploy sealed-secrets on git

```
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update
helm upgrade --install sealed-secrets sealed-secrets/sealed-secrets -n kube-system --set fullnameOverride=sealed-secrets-controller
```

Create secret:

```
kubectl create secret generic my-secret --from-literal=password=supersecret --dry-run=client -o yaml > secret.yaml
```

Encrypt the secret:

```
kubeseal --format=yaml -n NAMESPACE < secret.yaml > sealed-secret.yaml
```

## Run ArgoCD

```
kubectl apply root-app.yaml
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
