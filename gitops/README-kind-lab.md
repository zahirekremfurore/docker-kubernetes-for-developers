# Kind + Flux + Weave GitOps Lab

This repository now includes a local Flux sync target at `gitops/clusters/local-kind`.

## What changed

- `gitops/clusters/local-kind` is the Flux path for the kind-based lab.
- `gitops/infrastructure/controllers/local` installs the local controller set:
  - Capacitor
  - Weave GitOps OSS
  - CloudNativePG
  - MariaDB Operator
  - MetalLB
  - a local-friendly Istio gateway stack
- `gitops/infrastructure/configs/local` contains local-only shared config.
- `gitops/apps/local-kind` deploys `product-service` and `user-service` without Vault/ExternalSecret dependencies.
- Container images are expected under `ghcr.io/zahirekremfurore/*`.

## Local apply flow

If you want to practice locally without connecting Flux to a Git repository, use this flow:

```powershell
kind create cluster --config iac/kind/cluster.yaml
kubectl config use-context kind-docker-k8s-training
flux check --pre
flux install
kubectl apply -k gitops/infrastructure/controllers/local
kubectl apply -k gitops/infrastructure/configs/local
kubectl apply -k gitops/apps/local-kind
```

Verify:

```powershell
kubectl get pods -A
kubectl get helmreleases -A
kubectl get svc -A
```

## Optional Git bootstrap

Create the cluster:

```powershell
kind create cluster --config iac/kind/cluster.yaml
kubectl config use-context kind-docker-k8s-training
kubectl get nodes
```

Bootstrap Flux against your own fork or branch:

```powershell
flux bootstrap github `
  --owner <github-user-or-org> `
  --repository <repo-name> `
  --branch <branch-name> `
  --path gitops/clusters/local-kind `
  --personal
```

Verify Flux:

```powershell
flux check
kubectl get pods -n flux-system
flux get sources all -A
flux get kustomizations -A
flux get helmreleases -A
```

## UI access

Capacitor:

```powershell
kubectl -n flux-system port-forward svc/capacitor 9000:9000
```

Weave GitOps:

```powershell
kubectl -n flux-system port-forward svc/ww-gitops-weave-gitops 9001:9001
```

Weave GitOps needs an emergency user secret for local login. For a local-only lab, create it before or after applying the Weave manifests:

```powershell
kubectl create secret generic cluster-user-auth `
  -n flux-system `
  --from-literal=username=admin `
  --from-literal=password='<bcrypt-hash>' `
  --dry-run=client -o yaml | kubectl apply -f -
```

If the pod started before the secret existed, restart it:

```powershell
kubectl rollout restart deployment ww-gitops-weave-gitops -n flux-system
```

## Notes

- `gitops/clusters/production/kustomization.yaml` was added so the production path is a valid Flux target as well.
- The local overlays intentionally avoid Vault, External Secrets, and Cloudflare-driven cert-manager flows.
- `gitops/infrastructure/configs/local/metallb-pool.yaml` assumes the usual kind Docker subnet `172.18.0.0/16`. If `docker network inspect kind` shows a different subnet, update the pool before reconciling the gateway service.
- The local manifest set can be applied directly without creating a Flux `GitRepository`.
