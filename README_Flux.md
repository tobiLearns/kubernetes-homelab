[<p style="text-align:right;">back to main README</p>](./README.md)
## GitOps with Flux

This cluster is managed by [Flux](https://fluxcd.io/), a GitOps operator that runs inside the cluster and continuously reconciles its state with this repository. There is no manual `kubectl apply` — every change is made by editing files in the repo and pushing to `main`.

### How it works

1. Flux watches the Git repository via a `GitRepository` source object.
2. `Kustomization` objects in `clusters/staging/` define which paths in the repo Flux should apply to the cluster, and at what interval.
3. Flux renders the Kustomize overlays and applies the resulting manifests. If the cluster drifts from the desired state, Flux corrects it automatically.
4. Secrets are encrypted with SOPS before being committed. Flux decrypts them during reconciliation using an age key stored in the cluster as a Kubernetes Secret (`sops-age`).

### Repository structure

```
clusters/staging/    # Flux entrypoint — Kustomization objects pointing at app paths
apps/base/           # Base manifests, shared across environments
apps/staging/        # Staging overlays — environment-specific config and secrets
```

[<p style="text-align:right;">back to main README</p>](./README.md)
