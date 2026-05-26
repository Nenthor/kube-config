# Kube Config

This repository holds the entire infrastructure of the kubernetes cluster and is used by ArgoCD to maintain and deploy it.

## Specific Documentation

- [Setup](docs/setup.md)

## Common Commands

```
kubeseal -o yaml < secret.yaml > sealed-secret.yaml
```
