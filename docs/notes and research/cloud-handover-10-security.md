# Cloud Handover Checklist: 10. Security

Scenario: the development team has handed this application to the platform engineering team. We are preparing it for cloud infrastructure, instrumentation, and deployment to Kubernetes or ECS.

This note covers the tenth checklist item only: **Security**.

## Objective

Security should be built into the deployment, not added later.

This stage answers:

- Do containers run with least privilege?
- Are Linux capabilities dropped?
- Is privilege escalation disabled?
- Is the root filesystem read-only where possible?
- Are secrets kept out of Git and images?
- Are IAM/RBAC permissions minimal?
- Are images scanned and base images pinned?
- Are network restrictions configured?
- Is public traffic protected with TLS?
- Are supply-chain controls defined?

## Checklist

- [x] Run containers as non-root.
- [x] Drop Linux capabilities.
- [x] Disable privilege escalation.
- [x] Use read-only root filesystem where possible.
- [x] Avoid mounting Docker socket.
- [x] Avoid privileged containers.
- [ ] Store secrets outside Git.
- [ ] Use least-privilege IAM/RBAC.
- [ ] Scan images for vulnerabilities.
- [ ] Pin base image versions.
- [x] Avoid unnecessary tools in runtime images.
- [ ] Configure network restrictions.
- [ ] Configure TLS for public traffic.
- [ ] Review supply chain requirements.

Note:

- Items marked checked are confirmed by reviewing the repo's Kubernetes manifests and Dockerfiles.
- Items unchecked are platform/security controls that must be implemented or validated in the target environment.

## Current Security Posture Summary

The default Kubernetes manifests already include good baseline hardening:

- workloads run as non-root
- `runAsUser: 1000`
- `runAsGroup: 1000`
- `fsGroup: 1000`
- privilege escalation disabled
- Linux capabilities dropped
- containers are not privileged
- root filesystem is read-only for core services
- service accounts are defined per app service
- backend services are private `ClusterIP`
- default public exposure is frontend only

Important gaps or decisions:

- TLS is not configured by default.
- NetworkPolicies are optional, not enabled by default.
- No application Secrets are present by default, but optional cloud features require secret handling.
- RBAC/IAM bindings are not fully defined for production cloud integrations.
- Image scanning is not configured in the repo.
- Some base images are digest-pinned, but not every image reference is pinned.
- Supply-chain signing/attestation policy is not defined.

## Kubernetes Security Context

The core service manifests use this pattern:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000
```

Container-level pattern:

```yaml
securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
  privileged: false
  readOnlyRootFilesystem: true
```

This is present across the default workloads:

| Component | Non-root | Drop capabilities | Privileged false | Read-only root FS |
| --- | --- | --- | --- | --- |
| `frontend` | Yes | Yes | Yes | Yes |
| `productcatalogservice` | Yes | Yes | Yes | Yes |
| `currencyservice` | Yes | Yes | Yes | Yes |
| `cartservice` | Yes | Yes | Yes | Yes |
| `redis-cart` | Yes | Yes | Yes | Yes |
| `shippingservice` | Yes | Yes | Yes | Yes |
| `paymentservice` | Yes | Yes | Yes | Yes |
| `emailservice` | Yes | Yes | Yes | Yes |
| `checkoutservice` | Yes | Yes | Yes | Yes |
| `recommendationservice` | Yes | Yes | Yes | Yes |
| `adservice` | Yes | Yes | Yes | Yes |
| `loadgenerator` | Yes | Yes | Yes | Yes |
| `shoppingassistantservice` | Yes | Yes | Yes | No |

Note:

```text
shoppingassistantservice sets readOnlyRootFilesystem: false in its optional component.
Review this before enabling the assistant feature.
```

## Container Runtime Security

### Docker socket

No manifest mounts the Docker socket.

Good:

```text
No /var/run/docker.sock mount found.
```

Risk to avoid:

```text
Mounting the Docker socket gives the container broad control over the node/container runtime.
```

### Privileged containers

Default manifests set:

```yaml
privileged: false
```

No app workload requires privileged mode.

### Runtime images

The Dockerfiles generally use multi-stage builds:

- builder image installs build tools
- runtime image contains only runtime artifacts

Examples:

| Runtime | Pattern |
| --- | --- |
| Go services | Build static binary, run in distroless static image |
| Node.js services | Build dependencies in Node Alpine, run in Alpine with Node.js |
| Python services | Build dependencies in builder stage, copy Python libs to runtime |
| Java service | Build with JDK, run with JRE |
| .NET service | Publish with SDK, run with runtime-deps image |

Platform note:

```text
This is a good baseline. Still scan final images to confirm build tools are absent from runtime layers.
```

## Base Image Pinning

Many Dockerfiles pin base images by digest.

Examples of digest-pinned base images:

- Go Alpine builder images in several services
- Node Alpine builder image
- Alpine runtime image for Node services
- Python Alpine/slim images
- .NET SDK/runtime-deps images
- Java Eclipse Temurin images
- BusyBox init container image
- OpenTelemetry Collector optional image

Known gaps:

| Image reference | Issue | Recommendation |
| --- | --- | --- |
| `src/frontend/Dockerfile` uses `golang:1.25.0-alpine` | Tag-pinned but not digest-pinned | Pin by digest. |
| Go runtime uses `gcr.io/distroless/static` | Tag/reference is not digest-pinned in service Dockerfiles | Pin by digest or enforce digest pinning in CI. |
| `redis:alpine` | Public image tag, not digest-pinned in manifests | Pin digest or mirror to private registry. |
| Placeholder images in `kubernetes-manifests/` | Skaffold replaces these | Ensure rendered images use immutable tags/digests. |

Platform recommendation:

```text
Use immutable image references for production:
<image>@sha256:<digest>
or at least <image>:<git-sha> with recorded digest.
```

## Secrets Handling

Default deployment:

- No application Kubernetes Secrets are required for the basic demo.
- Redis has no password by default.
- Email/payment services are mock/dummy implementations.

Optional features introduce secrets:

| Feature | Secret need | Recommended handling |
| --- | --- | --- |
| AlloyDB cart/product catalog | DB password | Cloud Secret Manager + workload identity or External Secrets |
| Shopping assistant | DB password and Google API key if used | Secret Manager/External Secrets; avoid committing API keys |
| Spanner | Usually identity-based; connection string may be sensitive | Workload identity/IAM; Secret if connection string includes credentials |
| Private image registry | Pull credentials if needed | Node identity, workload identity, or `imagePullSecrets` |

Security rule:

```text
Do not store secret values in Git, Dockerfiles, ConfigMaps, or plain manifests.
```

## IAM And RBAC

Default manifests create Kubernetes ServiceAccounts per service.

Examples:

- `frontend`
- `cartservice`
- `checkoutservice`
- `productcatalogservice`
- `paymentservice`
- `emailservice`
- `recommendationservice`
- `shippingservice`
- `currencyservice`
- `adservice`
- `loadgenerator`

Default manifests do not define Kubernetes Roles or RoleBindings for app services.

That is fine for the basic app because services do not need Kubernetes API access.

Optional cloud integrations need IAM:

| Integration | Service account | Required access |
| --- | --- | --- |
| AlloyDB cart | `cartservice` | AlloyDB client, Secret Manager secret accessor |
| Product catalog from AlloyDB | `productcatalogservice` | AlloyDB client, Secret Manager secret accessor |
| Shopping assistant | `shoppingassistantservice` | AlloyDB, Secret Manager, Generative AI/API access |
| Spanner cart | `cartservice` | Spanner database user |
| Telemetry to cloud services | instrumented services or collector | telemetry backend permissions |

Platform recommendation:

```text
Bind cloud IAM only to the Kubernetes ServiceAccount that needs it.
Do not give broad project/account permissions to all workloads.
```

## Image Vulnerability Scanning

The repo does not configure an image scanning gate.

Add scanning in registry and/or CI:

| Platform | Options |
| --- | --- |
| AWS | ECR scanning, Amazon Inspector |
| GCP | Artifact Analysis |
| Azure | Microsoft Defender for Cloud, ACR integrations |
| Generic | Trivy, Grype, Snyk, Anchore, Aqua, Prisma |

Example:

```sh
trivy image <registry>/online-boutique/frontend:<git-sha>
```

Recommended policy:

- scan every image
- fail builds on critical vulnerabilities unless exception approved
- track base image CVEs
- rebuild images regularly after base image updates
- scan both app images and third-party images like Redis

## Network Restrictions

Default:

```text
No NetworkPolicies are enabled by default.
```

Repo provides optional network policies:

```text
kustomize/components/network-policies
```

Enable:

```sh
cd kustomize
kustomize edit add component components/network-policies
kubectl apply -k .
```

Or with Skaffold:

```sh
skaffold run -p network-policies --default-repo=<registry>/<repo>
```

Important:

- NetworkPolicies require an enforcing CNI such as Calico, Cilium, or GKE Dataplane V2.
- The repo notes egress is intentionally wide open in provided policies.

Security recommendation:

```text
Use default-deny plus explicit allow rules for production if your platform requires pod-level segmentation.
```

## TLS For Public Traffic

Default:

```text
frontend-external exposes HTTP on port 80.
No TLS certificate is configured.
```

Production requirement:

- configure HTTPS
- terminate TLS at ingress/gateway/load balancer
- redirect HTTP to HTTPS if required
- monitor certificate expiration

Options:

- Kubernetes Ingress with cert-manager
- Gateway API with managed certificates
- cloud load balancer managed certificate
- Istio/Cloud Service Mesh gateway
- AWS ALB with ACM certificate for ECS/EKS

Security recommendation:

```text
Do not expose production frontend over plain HTTP only.
```

## Supply Chain Controls

Supply-chain controls are not fully defined in this repo.

Recommended controls:

- build in trusted CI
- use immutable image tags/digests
- generate SBOMs
- sign images
- verify signatures at deploy time
- scan dependencies and images
- restrict who can push to production registry
- require review for manifest changes
- keep provenance/attestations
- pin base image digests

Possible tools:

| Control | Tools |
| --- | --- |
| SBOM | Syft, Trivy, Docker BuildKit, cloud-native tooling |
| Image signing | Cosign/Sigstore, Notation |
| Admission policy | Kyverno, OPA Gatekeeper, ValidatingAdmissionPolicy |
| Provenance | SLSA provenance, Tekton Chains, GitHub/GitLab attestations |
| Dependency scanning | Dependabot, Renovate, Snyk, Trivy, Grype |

## Kubernetes Policy Validation

Check security contexts:

```sh
kubectl get deploy -o yaml | rg "securityContext|runAsNonRoot|allowPrivilegeEscalation|readOnlyRootFilesystem|capabilities|privileged"
```

Check service accounts:

```sh
kubectl get serviceaccount
```

Check RBAC:

```sh
kubectl get role,rolebinding,clusterrole,clusterrolebinding
```

Check network policies:

```sh
kubectl get networkpolicy
```

Check public exposure:

```sh
kubectl get svc
```

Check image references:

```sh
kubectl get deploy -o jsonpath='{range .items[*]}{.metadata.name}{" -> "}{.spec.template.spec.containers[*].image}{"\n"}{end}'
```

## ECS Security Translation

For ECS, translate Kubernetes security controls into:

| Kubernetes concept | ECS equivalent |
| --- | --- |
| ServiceAccount/IAM | Task role and execution role |
| NetworkPolicy | Security groups, subnets, NACLs |
| Secret | AWS Secrets Manager / SSM Parameter Store |
| Pod securityContext | Container user, readonly root filesystem, Linux capabilities |
| Image pull secret | Task execution role for ECR or repository credentials |
| Ingress/LoadBalancer TLS | ALB/NLB listener with ACM certificate |
| Logs | CloudWatch log driver |

ECS checklist:

- [ ] Use least-privilege task role.
- [ ] Use execution role only for image pull/logs/secrets retrieval.
- [ ] Keep backend tasks in private subnets.
- [ ] Expose only frontend through ALB/API Gateway.
- [ ] Configure security groups per service boundary.
- [ ] Store secrets in Secrets Manager/SSM.
- [ ] Enable CloudWatch logs.
- [ ] Scan images in ECR/CI.
- [ ] Use TLS on public listeners.

## Security Risks And Follow-Ups

| Area | Current state | Risk | Follow-up |
| --- | --- | --- | --- |
| TLS | Not configured by default | Public traffic is HTTP only | Add TLS via ingress/gateway/load balancer. |
| Network policies | Optional, not default | Backends rely on cluster-private services only | Enable/enforce NetworkPolicies or cloud firewall/security groups. |
| Secrets | No default app secrets, optional features need them | Secrets could be placed in manifests during setup | Use Secret Manager/External Secrets. |
| IAM/RBAC | ServiceAccounts exist, but IAM binding is optional | Optional cloud features may get overbroad permissions | Bind least privilege per workload. |
| Image scanning | Not configured | Vulnerable images may deploy | Add scanning gate. |
| Base image pinning | Mostly digest-pinned, but gaps exist | Mutable base image references | Pin all base/runtime images by digest. |
| Redis | Public image, no auth, ephemeral | Demo-grade state/security | Use managed Redis or harden Redis for production. |
| Assistant service | `readOnlyRootFilesystem: false` | Weaker filesystem hardening | Review need before enabling. |
| Supply chain | Not fully defined | Weak provenance/signing controls | Add SBOM/signing/admission policy if required. |

## Security Output

At the end of this stage, platform engineering should have:

- [x] Container security context reviewed.
- [x] Non-root execution confirmed.
- [x] Privilege escalation disabled.
- [x] Linux capabilities dropped.
- [x] Privileged containers avoided.
- [x] Docker socket mounts avoided.
- [x] Runtime image hardening reviewed.
- [x] Secret-bearing integrations identified.
- [x] IAM/RBAC needs documented.
- [x] TLS and network restriction gaps documented.
- [ ] Image scanning enabled.
- [ ] TLS configured for public traffic.
- [ ] NetworkPolicies/security groups finalized.
- [ ] Secret management implementation confirmed.
- [ ] Least-privilege IAM/RBAC implemented.
- [ ] Supply-chain controls defined.

## Ready For Next Checklist Item

The next stage is:

```text
11. Observability
```

In that stage, define logs, metrics, traces, dashboards, alerts, service-level indicators, and how platform engineering will detect and debug failures after deployment.

