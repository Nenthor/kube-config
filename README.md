# Kube Config

This repository holds the entire infrastructure of the kubernetes cluster and is used by ArgoCD to maintain and deploy it.

## Specific Documentation

- [Setup](docs/setup.md)
- [Database](docs/database.md)

## Common Commands

**Seal a Secret:**

```
kubeseal -o yaml < secret.yaml > sealed-secret.yaml
```

**Restart a service:**

```
kubectl rollout restart deployment <SERVICE> -n <NAMESPACE>
```

**Port forward the database to your local machine:**

```
kubectl port-forward -n cnpg-database svc/postgres-rw 5432:5432
```

**Get latest version of a helm chart:**

```
helm show chart oci://ghcr.io/grafana-community/helm-charts/loki
```

## Helpful Links

- schema.yaml: https://github.com/datreeio/CRDs-catalog
