# Build the laptop Kubernetes foundation

## Primary laptop distribution

arm64

## Primary laptop distro decision record in ADR



## Laptop resource profiles



## Platform namespaces

Decision: use purpose-based namespaces managed from `02-platformone/platformone-local-lab/manifests/platform-namespaces.yaml`.

Managed namespaces:

- `platformone-system`
- `observability`
- `security`
- `gitops`
- `apps`
- `sandbox`

Labels:

- `platformone.io/managed-by=platformone-local-lab`
- `platformone.io/environment=local`
- `platformone.io/purpose=<purpose>`
- `platformone.io/owner=<owner>`

Ownership:

| Namespace | Owner |
| --- | --- |
| `platformone-system` | `platform-engineering` |
| `observability` | `sre` |
| `security` | `security-engineering` |
| `gitops` | `platform-engineering` |
| `apps` | `product-engineering` |
| `sandbox` | `platform-engineering` |

Source of truth:

- Manifest: `02-platformone/platformone-local-lab/manifests/platform-namespaces.yaml`
- Local-lab strategy: `02-platformone/platformone-local-lab/namespaces/local-namespace-strategy.md`
- Docs page: `02-platformone/platformone-docs/docs/engineering/local-namespace-strategy.md`

Validated commands:

```bash
cd 02-platformone/platformone-local-lab
make namespaces-apply
make namespaces-verify
kubectl get ns platformone-system observability security gitops apps sandbox
```



## Namespace resource guardrails

Decision: scaffold local `ResourceQuota` and `LimitRange` guardrails for the managed platform namespaces.

Source of truth:

- Manifests: `02-platformone/platformone-local-lab/manifests/resource-guardrails/`
- Local-lab strategy: `02-platformone/platformone-local-lab/namespaces/local-resource-guardrails.md`
- Docs page: `02-platformone/platformone-docs/docs/engineering/local-resource-guardrails.md`

Standard commands:

```bash
cd 02-platformone/platformone-local-lab
make guardrails-apply
make guardrails-verify
```

Guardrail objects:

- `ResourceQuota` caps total namespace usage.
- `LimitRange` sets container and PVC defaults and bounds.



## Initial RBAC roles

Decision: use namespace-scoped RBAC roles for managed platform namespaces.

Role model:

- `platformone-namespace-admin`
- `platformone-namespace-editor`
- `platformone-namespace-viewer`

Group model:

- `platformone:platform-engineering`
- `platformone:sre`
- `platformone:security-engineering`
- `platformone:product-engineering`

Current implementation:

- `02-platformone/platformone-local-lab/manifests/rbac/` implements the namespace-folder pattern for all managed namespaces.
- Non-platform-owned namespaces include explicit platform engineering viewer bindings.

Source of truth:

- Manifests: `02-platformone/platformone-local-lab/manifests/rbac/`
- Local-lab strategy: `02-platformone/platformone-local-lab/namespaces/local-rbac-strategy.md`
- Docs page: `02-platformone/platformone-docs/docs/engineering/local-rbac-strategy.md`

Standard commands:

```bash
cd 02-platformone/platformone-local-lab
make rbac-apply
make rbac-verify
```

TODO:

- Add `rbac-can-i` after local identity and group simulation are defined.



## Cluster lifecycle workflow

Source of truth:

- Automation: `02-platformone/platformone-local-lab/Makefile`
- Scripts: `02-platformone/platformone-local-lab/scripts/cluster/`
- Runbook: `02-platformone/platformone-docs/docs/engineering/local-cluster-lifecycle.md`

Standard commands:

```bash
cd 02-platformone/platformone-local-lab
make cluster-prereqs
make cluster-up
make cluster-verify
make cluster-down
make cluster-reset
```



## Local registry strategy

Decision: hybrid k3d + AWS ECR.

Default dev path:

- Build locally.
- Import into k3d with `k3d image import`.
- Deploy using local image names such as `shopease/cartservice:dev`.

Release/proof path:

- Build and tag with a unique tag.
- Push to Amazon ECR.
- Deploy using the ECR image reference.
- For local k3d, create an `imagePullSecret` such as `ecr-pull-secret` in the workload namespace.

Source of truth:

- Local-lab strategy: `02-platformone/platformone-local-lab/registry/local-registry-strategy.md`
- Docs page: `02-platformone/platformone-docs/docs/engineering/local-registry-strategy.md`

Completion gate:

- Local path works: build, import, deploy, and observe a visible change.
- ECR path works: build, push, deploy, delete the pod, and confirm Kubernetes pulls it again.
- Tradeoff is explainable: k3d import is fast; ECR is AWS-realistic but requires auth, unique tags, and network access.

## Local DNS and TLS approaches

Decision:

- DNS: hosts-file based `*.platformone.local`.
- TLS: HTTP-only in the current local baseline.
- Planned HTTPS enhancement: local CA plus cert-manager.

Default hosts target:

```text
127.0.0.1
```

Example hostnames:

- `test.platformone.local`
- `shopease.platformone.local`
- `argocd.platformone.local`
- `grafana.platformone.local`

Source of truth:

- Local-lab strategy: `02-platformone/platformone-local-lab/networking/local-dns-tls-strategy.md`
- Draft access runbook: `02-platformone/platformone-local-lab/networking/access-services-locally-runbook.md`
- Smoke manifest: `02-platformone/platformone-local-lab/manifests/smoke-test.yaml`
- Docs page: `02-platformone/platformone-docs/docs/engineering/local-dns-tls-strategy.md`
- Docs runbook: `02-platformone/platformone-docs/docs/engineering/access-services-locally.md`

Runbook status:

- Draft, not yet tested end to end.
- Current examples use port `8080` because `k3d/cluster-dev.k3d.yaml` maps host `8080` to cluster port `80`.

Smoke test commands:

```bash
cd 02-platformone/platformone-local-lab
make smoke-apply
make smoke-verify
make smoke-delete
```

Completion gate:

- `test.platformone.local` is documented as the first test hostname.
- At least one `*.platformone.local` service is reachable over HTTP.
- The same hostname can be restored after k3d cluster deletion and recreation.
- TLS is clearly documented as planned, not implemented in the current local baseline.

## What's used for local storage

Decision: use the default k3s `local-path` StorageClass for development-grade PVC behavior.

Operating model:

- Pod recreation should preserve data when the same PVC is retained.
- Full k3d cluster deletion and recreation does not guarantee data persistence.
- Local storage is for development, demos, manifest validation, and backup/restore practice.
- Local storage does not claim production durability, replication, or failure-domain tolerance.

Source of truth:

- Local-lab strategy: `02-platformone/platformone-local-lab/storage/local-storage-strategy.md`
- Smoke manifest: `02-platformone/platformone-local-lab/manifests/storage-smoke-test.yaml`
- Docs page: `02-platformone/platformone-docs/docs/engineering/local-storage-strategy.md`

Smoke test commands:

```bash
cd 02-platformone/platformone-local-lab
make storage-apply
make storage-verify
make storage-restart
make storage-verify
make storage-delete
```

## Cluster-Health checks

Decision: use layered local cluster health checks.

Health tiers:

- Quick check: `kubectl get nodes` and `kubectl get pods -n kube-system`
- Standard check: `make cluster-verify`
- Full local foundation check: `make cluster-health`

Health domains:

- Kubernetes API access
- Node readiness
- Core `kube-system` add-ons
- Cluster DNS
- Traefik ingress
- Local PVC-backed storage

Source of truth:

- Local-lab runbook: `02-platformone/platformone-local-lab/health/cluster-health-checks.md`
- Docs page: `02-platformone/platformone-docs/docs/engineering/local-cluster-health-checks.md`

Standard commands:

```bash
cd 02-platformone/platformone-local-lab
make cluster-verify
make cluster-health
```

## Cleanup and rebuild process



## Test run for complete cluster deletion and recreation



#### Baseline cluster resource usage
