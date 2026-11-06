# OpenChoreo GitOps synchronization

This repository is the source of truth for the OpenChoreo configuration committed
here. After the Flux bootstrap change is merged, Flux reconciles `main` every
minute. Application source code and image builds remain in the separate
`biocare-web` repository.

## Ownership

Flux applies the OpenChoreo intent resources listed in the root
`kustomization.yaml`: the default `Project`, its `ComponentRelease`, the
biocare-web staging and production `ReleaseBinding`s, and the existing
`DeploymentPipeline`. The staging replica count is intentionally set to the
currently observed value of one. OpenChoreo controllers render those resources
and own the resulting data-plane resources, including Kubernetes `Deployment`s.
Do not add controller-generated data-plane resources to this Kustomize target.

Automatic pruning is disabled. Removing a manifest from this repository does
not delete its Kubernetes resource.

## Bootstrap

Install the Flux CLI and only its source and Kustomize controllers, then apply
the bootstrap identity and permissions before applying the source and
Kustomization. Check that the repository's desired state is appropriate for the
target cluster before enabling the Kustomization:

```sh
flux install --namespace=flux-system \
  --components=source-controller,kustomize-controller \
  --version=v2.10.0 \
  --watch-all-namespaces=false
kubectl apply -f clusters/k3d-openchoreo/flux-system/bootstrap-rbac.yaml
kubectl apply -f clusters/k3d-openchoreo/flux-system/backstage-flux-reader.yaml
kubectl apply -f clusters/k3d-openchoreo/flux-system/gotk-sync.yaml
```

The GitHub repository is public, so Flux fetches it anonymously and no GitHub
credential is required. Do not put credentials in this repository. If the
repository becomes private, create a Kubernetes Secret in `flux-system` using
`flux create secret git` and add a `secretRef` to the `GitRepository`.

The Flux Kustomization waits for both biocare-web `ReleaseBinding`s to become
Ready. The OpenChoreo CRDs and controllers must already be healthy; this setup
does not install or modify OpenChoreo Helm releases. The `prune` setting is
disabled, so removing a manifest will not delete the corresponding resource.

## Monitor

```sh
flux get sources git -n flux-system
flux get kustomizations -n flux-system
flux logs --all-namespaces --level=error
kubectl get releasebinding biocare-web-staging -n default -o yaml
kubectl get renderedrelease biocare-web-staging -n default -o yaml
kubectl get deployments -n dp-default-default-staging-1362f732
```

For source or build failures, inspect the `GitRepository` and `Kustomization`
conditions. For an apply or application rollout failure, inspect the Flux
Kustomization first, then the OpenChoreo `ReleaseBinding` and `RenderedRelease`
conditions. Flux does not own application-source builds or deployment artifacts.
