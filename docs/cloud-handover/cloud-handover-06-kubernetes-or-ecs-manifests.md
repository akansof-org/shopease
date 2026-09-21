# Cloud Handover Checklist: 6. Kubernetes Or ECS Manifests

Scenario: the development team has handed this application to the platform engineering team. We are preparing it for cloud infrastructure, instrumentation, and deployment to Kubernetes or ECS.

This note covers the sixth checklist item only: **Kubernetes Or ECS Manifests**.

## Objective

Prepare the deployment objects for the target platform.

This stage answers:

- Which Kubernetes objects already exist?
- Which Kubernetes objects are missing or optional?
- Which objects are required before this app is production-ready?
- How would the Kubernetes deployment model translate to ECS?
- Which deployment objects are app-specific vs platform/environment-specific?

## Kubernetes Checklist

- [ ] Namespace.
- [x] Deployment or StatefulSet.
- [x] Service.
- [ ] ConfigMap.
- [ ] Secret.
- [x] ServiceAccount.
- [x] Ingress, Gateway, or LoadBalancer service.
- [ ] PersistentVolumeClaim if needed.
- [ ] HorizontalPodAutoscaler if needed.
- [ ] PodDisruptionBudget if needed.
- [ ] NetworkPolicy if required.
- [x] Resource requests and limits.
- [x] Readiness and liveness probes.
- [x] Security context.

Note:

- Items marked checked exist in the default Kubernetes manifests.
- Items unchecked are either absent, optional, or environment/platform decisions.

## ECS Checklist

- [ ] Cluster.
- [ ] Task definition.
- [ ] Container definitions.
- [ ] Service definition.
- [ ] Target group.
- [ ] Load balancer listener/rules.
- [ ] Security groups.
- [ ] IAM task role.
- [ ] IAM execution role.
- [ ] CloudWatch logs.
- [ ] Service discovery or Service Connect.
- [ ] Desired count.
- [ ] CPU and memory settings.
- [ ] Health check grace period.

Note:

- This repo is Kubernetes-first.
- ECS deployment objects are not present in the repo and would need to be created separately.

## Current Kubernetes Manifest Layout

This repo has multiple Kubernetes manifest paths.

| Path | Purpose | Notes |
| --- | --- | --- |
| `kubernetes-manifests/` | Skaffold-oriented manifests | Uses placeholder image names. Do not apply directly without Skaffold image replacement. |
| `release/kubernetes-manifests.yaml` | Prebuilt release manifest | Uses public prebuilt images. Useful for quick demo deployment. |
| `kustomize/base/` | Base Kustomize manifests | Similar base app objects. |
| `kustomize/components/` | Optional variations | Network policies, cloud ops, Memorystore, Spanner, AlloyDB, Istio, image tag/registry customization, etc. |
| `kustomize/kustomization.yaml` | Kustomize entrypoint | Used with `kubectl apply -k kustomize/`. |

Platform note:

```text
For owned cloud environments, prefer building your own images and deploying rendered manifests with approved image references.
Use release manifests mainly for fast demo validation.
```

## Default Kubernetes Objects Present

The default app includes Kubernetes objects for the main services.

| Component | Deployment | Service | ServiceAccount | Public exposure |
| --- | --- | --- | --- | --- |
| `frontend` | Yes | Yes | Yes | Yes, `frontend-external` LoadBalancer |
| `productcatalogservice` | Yes | Yes | Yes | No |
| `currencyservice` | Yes | Yes | Yes | No |
| `cartservice` | Yes | Yes | Yes | No |
| `redis-cart` | Yes | Yes | No dedicated ServiceAccount | No |
| `shippingservice` | Yes | Yes | Yes | No |
| `paymentservice` | Yes | Yes | Yes | No |
| `emailservice` | Yes | Yes | Yes | No |
| `checkoutservice` | Yes | Yes | Yes | No |
| `recommendationservice` | Yes | Yes | Yes | No |
| `adservice` | Yes | Yes | Yes | No |
| `loadgenerator` | Yes | No | Yes | No |
| `shoppingassistantservice` | Optional component | Optional component | Optional component | No |

## Kubernetes Object Inventory

### Deployments

All default app workloads are modeled as `Deployment`.

There are no `StatefulSet` objects in the default manifests.

| Workload | Kubernetes kind | Notes |
| --- | --- | --- |
| `frontend` | Deployment | Public-facing HTTP app. |
| `productcatalogservice` | Deployment | Internal gRPC service. |
| `currencyservice` | Deployment | Internal gRPC service. |
| `cartservice` | Deployment | Internal gRPC service, depends on Redis. |
| `redis-cart` | Deployment | In-cluster Redis with `emptyDir`. |
| `shippingservice` | Deployment | Internal gRPC service. |
| `paymentservice` | Deployment | Internal gRPC service. |
| `emailservice` | Deployment | Internal gRPC service. |
| `checkoutservice` | Deployment | Internal gRPC orchestration service. |
| `recommendationservice` | Deployment | Internal gRPC service. |
| `adservice` | Deployment | Internal gRPC service. |
| `loadgenerator` | Deployment | Optional traffic generator. |

Platform note:

```text
redis-cart is a Deployment with emptyDir, not a StatefulSet with persistent storage.
That is acceptable for a demo but should be reviewed for production.
```

### Services

All request-serving components have Kubernetes Services.

| Service | Type | Port | Target port | Protocol expectation |
| --- | --- | ---: | ---: | --- |
| `frontend` | ClusterIP | `80` | `8080` | HTTP |
| `frontend-external` | LoadBalancer | `80` | `8080` | HTTP |
| `productcatalogservice` | ClusterIP | `3550` | `3550` | gRPC |
| `currencyservice` | ClusterIP | `7000` | `7000` | gRPC |
| `cartservice` | ClusterIP | `7070` | `7070` | gRPC |
| `redis-cart` | ClusterIP | `6379` | `6379` | TCP |
| `shippingservice` | ClusterIP | `50051` | `50051` | gRPC |
| `paymentservice` | ClusterIP | `50051` | `50051` | gRPC |
| `emailservice` | ClusterIP | `5000` | `8080` | gRPC |
| `checkoutservice` | ClusterIP | `5050` | `5050` | gRPC |
| `recommendationservice` | ClusterIP | `8080` | `8080` | gRPC |
| `adservice` | ClusterIP | `9555` | `9555` | gRPC |

Important:

```text
emailservice has service port 5000 and container targetPort 8080.
Clients should use emailservice:5000.
```

### Namespace

The default manifests do not define a `Namespace`.

Deployment behavior:

- `kubectl apply -f ...` deploys to the current namespace unless `-n` is used.
- `skaffold run` deploys to the current Kubernetes context namespace unless configured otherwise.

Platform recommendation:

```text
Create a dedicated namespace per environment.
Example: online-boutique-dev, online-boutique-staging, online-boutique-prod.
```

Example:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: online-boutique-dev
```

Deploy:

```sh
kubectl apply -n online-boutique-dev -k kustomize/
```

### ConfigMaps

The default manifests mostly inline environment variables directly into Deployments.

Default app:

- No central application `ConfigMap`.
- Service addresses and feature flags are embedded in Deployment env sections.

Optional components:

- `kustomize/components/google-cloud-operations/otel-collector.yaml` includes an OpenTelemetry Collector `ConfigMap`.

Platform recommendation:

```text
For owned environments, centralize non-secret config in ConfigMaps or platform-specific configuration.
This makes per-environment overlays cleaner.
```

Candidate ConfigMap values:

- service addresses
- feature flags
- platform/env labels
- telemetry collector address
- Redis address if not secret

### Secrets

The default manifests do not include application Secrets.

Default app:

- No application secret required for basic demo deployment.
- Redis runs without auth in the default in-cluster deployment.

Optional integrations may require secrets:

- AlloyDB database password.
- Spanner connection credentials if not identity-based.
- Gemini/API credentials if not identity-based.
- Private registry image pull secret if needed.

Platform recommendation:

```text
Use cloud secret manager integration or external secret tooling.
Do not commit secret values into manifests.
```

### ServiceAccounts

Most workloads define a ServiceAccount.

Present:

- `frontend`
- `productcatalogservice`
- `currencyservice`
- `cartservice`
- `shippingservice`
- `paymentservice`
- `emailservice`
- `checkoutservice`
- `recommendationservice`
- `adservice`
- `loadgenerator`
- optional `shoppingassistantservice`

Notable:

- `redis-cart` does not define a dedicated ServiceAccount in the default manifest.

Platform recommendation:

```text
If using cloud APIs, bind workload identities/IAM roles to service accounts deliberately.
For services with no cloud API access, keep permissions minimal.
```

### Public Exposure

Default public exposure:

```text
frontend-external: LoadBalancer
```

Alternative exposure options:

- Ingress
- Gateway API
- Istio Gateway
- internal-only frontend using `non-public-frontend` component

Kustomize components include:

- `service-mesh-istio`
- `non-public-frontend`

Platform recommendation:

```text
Choose one public exposure model per environment.
For production, prefer controlled ingress/gateway with TLS, DNS, WAF/security controls, and access logging.
```

### PersistentVolumeClaim

No `PersistentVolumeClaim` is present in the default manifests.

Redis storage:

```text
redis-cart uses emptyDir.
```

Impact:

- Cart data is lost when the Redis pod is recreated.
- This is fine for demos but not usually acceptable for production state.

Platform options:

- Keep `emptyDir` for non-production demos.
- Replace Redis with managed Redis/Memorystore/ElastiCache.
- Use a proper StatefulSet/PVC-backed Redis if running Redis in-cluster.

### HorizontalPodAutoscaler

No HPA is present in the default manifests.

Platform decision:

- Add HPA if services need autoscaling.
- Start with frontend, checkoutservice, productcatalogservice, currencyservice, and cartservice if traffic-driven scaling is needed.

Example skeleton:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### PodDisruptionBudget

No PDB is present in the default manifests.

Platform decision:

- Add PDBs for critical services before production.
- PDBs matter most once replicas are greater than one.

Example:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: frontend
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: frontend
```

### NetworkPolicy

No NetworkPolicy is included in the default manifest set.

However, this repo includes optional network policies:

```text
kustomize/components/network-policies/
```

Skaffold also has a `network-policies` profile.

Example:

```sh
skaffold run -p network-policies --default-repo=<registry>/<repo>
```

Platform recommendation:

```text
Use NetworkPolicies if the cluster CNI enforces them and the environment requires network segmentation.
Start with explicit allow paths based on the service dependency map.
```

### Resource Requests And Limits

The default manifests include resource requests and limits for app containers.

Platform decision:

- Treat current values as demo defaults.
- Validate with load testing and real metrics before production.

Check:

```sh
kubectl describe deployment frontend
kubectl top pods
```

### Readiness And Liveness Probes

The default manifests include probes.

Probe types:

| Component type | Probe style |
| --- | --- |
| `frontend` | HTTP `/_healthz` |
| gRPC backends | gRPC probes |
| `redis-cart` | TCP socket probes |

Platform note:

```text
Confirm your Kubernetes cluster version supports native gRPC probes.
If not, use grpc-health-probe or an HTTP sidecar/adapter.
```

### Security Context

The default manifests include pod and container security contexts.

Common controls:

- `runAsNonRoot`
- `runAsUser: 1000`
- `runAsGroup: 1000`
- `fsGroup: 1000`
- `allowPrivilegeEscalation: false`
- `capabilities.drop: ALL`
- `readOnlyRootFilesystem: true`

Platform note:

```text
Security context is already reasonably strong for a demo app.
Validate it against your cluster Pod Security Admission, Gatekeeper, Kyverno, or internal policy controls.
```

## Kubernetes Deployment Approaches

### Skaffold

Use when building and deploying from this repo:

```sh
skaffold run --default-repo=<registry>/<repo>
```

Skaffold:

- builds images
- tags images
- rewrites image references
- renders manifests
- applies manifests with `kubectl`

### Kustomize

Render:

```sh
kubectl kustomize kustomize/
```

Apply:

```sh
kubectl apply -k kustomize/
```

Use Kustomize components for:

- network policies
- Google Cloud Operations
- Memorystore
- Spanner
- AlloyDB
- image registry/tag customization
- non-public frontend
- Istio service mesh

### Release manifest

Quick demo deployment:

```sh
kubectl apply -f release/kubernetes-manifests.yaml
```

This uses prebuilt public images.

Platform recommendation:

```text
Use release manifests for quick validation.
Use owned images and rendered manifests for team-owned environments.
```

## ECS Translation

This repo does not provide ECS task definitions or services.

If deploying to ECS, create the following:

| ECS object | Purpose | App mapping |
| --- | --- | --- |
| Cluster | ECS runtime environment | One per environment or shared by platform |
| Task definition | Pod-like spec | One task definition per service |
| Container definition | Container settings | Image, port, env vars, health check, logs |
| Service definition | Keeps tasks running | One ECS service per app service |
| Target group | Load balancer backend | Usually frontend only |
| Load balancer listener/rules | Public/private routing | Public listener routes to frontend |
| Security groups | Network firewall | Allow only required service paths |
| IAM task role | App permissions | Needed for cloud APIs/secrets |
| IAM execution role | ECS agent permissions | Pull images, write logs, fetch secrets |
| CloudWatch logs | Log collection | One log group or service-specific groups |
| Service discovery/Service Connect | Internal DNS | Replaces Kubernetes service DNS |
| Desired count | Replica count | Per service |
| CPU/memory settings | Scheduling and limits | Per task/container |
| Health check grace period | Startup tolerance | Especially important for slower services |

## ECS Service Design

Recommended ECS shape:

```text
frontend ECS service
  -> public ALB target group

backend ECS services
  -> private networking
  -> service discovery or Service Connect

redis
  -> managed ElastiCache preferred
```

Do not expose backend services publicly.

## ECS Service Discovery Mapping

Kubernetes service names must become ECS-resolvable names.

Examples:

| App dependency | Kubernetes value | ECS equivalent example |
| --- | --- | --- |
| product catalog | `productcatalogservice:3550` | `productcatalogservice.online-boutique.local:3550` |
| currency | `currencyservice:7000` | `currencyservice.online-boutique.local:7000` |
| cart | `cartservice:7070` | `cartservice.online-boutique.local:7070` |
| checkout | `checkoutservice:5050` | `checkoutservice.online-boutique.local:5050` |
| Redis | `redis-cart:6379` | ElastiCache endpoint or Cloud Map name |

Platform note:

```text
The app does not care whether it is Kubernetes or ECS.
It only needs reachable dependency addresses in env vars.
```

## ECS Health Checks

For ECS:

- ALB health check can test frontend HTTP `/_healthz`.
- Internal gRPC services may need container health checks or sidecar-compatible checks.
- ECS/ALB does not handle gRPC health exactly like Kubernetes native gRPC probes by default.

Platform decision:

```text
Define how ECS will check gRPC service health.
Options include command health checks, grpc-health-probe binary, app-level HTTP health endpoint, or service-level smoke tests.
```

## Manifest Validation Commands

Render Kustomize:

```sh
kubectl kustomize kustomize/
```

Server-side dry run:

```sh
kubectl apply --dry-run=server -k kustomize/
```

Diff:

```sh
kubectl diff -k kustomize/
```

Skaffold render:

```sh
skaffold render --default-repo=<registry>/<repo>
```

Validate deployed objects:

```sh
kubectl get deploy
kubectl get svc
kubectl get sa
kubectl get pods
kubectl get endpoints
```

Check missing object types:

```sh
kubectl get configmap
kubectl get secret
kubectl get hpa
kubectl get pdb
kubectl get networkpolicy
kubectl get pvc
```

## Platform Gaps And Decisions

| Area | Current state | Decision needed |
| --- | --- | --- |
| Namespace | Not defined | Create per environment namespace. |
| ConfigMap | Not central by default | Decide whether to centralize env config. |
| Secret | Not required for default app | Add if using private registry, external Redis, AlloyDB, Spanner, assistant, or telemetry credentials. |
| Public exposure | `frontend-external` LoadBalancer | Keep, replace with Ingress/Gateway, or make internal-only. |
| Redis persistence | `emptyDir` | Keep demo mode or replace with managed/persistent Redis. |
| HPA | Not present | Add if autoscaling is needed. |
| PDB | Not present | Add for production availability. |
| NetworkPolicy | Optional component exists | Enable if cluster policy requires segmentation. |
| Observability | Optional Kustomize component exists | Enable/configure collector if tracing/metrics required. |
| ECS support | Not present | Create task definitions/services if ECS is target. |

## Kubernetes Or ECS Manifests Output

At the end of this stage, platform engineering should have:

- [x] Workload object inventory.
- [x] Service object inventory.
- [x] Public exposure model documented.
- [x] Security context status documented.
- [x] Probe status documented.
- [x] Resource request/limit status documented.
- [x] Optional Kustomize components identified.
- [x] Missing Kubernetes objects documented.
- [x] ECS translation requirements documented.
- [ ] Namespace strategy finalized.
- [ ] ConfigMap/Secret strategy finalized.
- [ ] Persistence strategy finalized.
- [ ] Autoscaling/PDB strategy finalized.
- [ ] NetworkPolicy decision finalized.
- [ ] Manifests rendered and validated against target cluster.

## Ready For Next Checklist Item

The next stage is:

```text
7. Health Checks
```

In that stage, validate readiness and liveness behavior for HTTP, gRPC, and TCP components, and confirm the target platform can run the configured health checks correctly.

