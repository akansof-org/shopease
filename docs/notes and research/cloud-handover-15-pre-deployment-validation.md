# Cloud Handover Checklist: 15. Pre-Deployment Validation

Scenario: the development team has handed this application to the platform engineering team. We are preparing it for cloud infrastructure, instrumentation, and deployment to Kubernetes or ECS.

This note covers the fifteenth checklist item only: **Pre-Deployment Validation**.

## Objective

Validate before touching a shared cluster.

This stage answers:

- Can every image build successfully?
- Can every container start with only runtime configuration?
- Are manifests rendered correctly before apply?
- Does the target namespace exist?
- Can the cluster pull the images?
- Are required secrets and config present?
- Are ingress, load balancer, DNS, and TLS prerequisites ready?
- Are storage prerequisites ready?
- Will resource quotas block deployment?
- Does the cluster version support the required Kubernetes features?

## Checklist

- [ ] Build every image locally or in CI.
- [ ] Run every container locally with required env vars.
- [ ] Validate manifests with `kubectl apply --dry-run=server` or equivalent.
- [ ] Confirm namespace exists.
- [ ] Confirm registry pull access.
- [ ] Confirm secrets/config exist.
- [ ] Confirm ingress/load balancer prerequisites exist.
- [ ] Confirm storage class exists.
- [ ] Confirm resource quotas are not exceeded.
- [ ] Confirm cluster version supports required features.

Note:

- Pre-deployment validation is the last safety check before changing a shared environment.
- This stage should fail fast and clearly.
- Do not use production as the first place where image, config, manifest, or IAM problems are discovered.

## Useful Commands

```bash
kubectl apply --dry-run=server -f <manifest.yaml>
kubectl diff -f <manifest.yaml>
kubectl get namespace
kubectl get storageclass
kubectl get resourcequota -A
```

Additional commands:

```bash
kubectl version
kubectl api-resources
kubectl auth can-i create deployments -n <namespace>
kubectl auth can-i create services -n <namespace>
kubectl get secret -n <namespace>
kubectl get configmap -n <namespace>
kubectl get ingressclass
kubectl get nodes -o wide
```

## Current Repo Pre-Deployment Facts

| Area | Current state | Validation action |
| --- | --- | --- |
| Image build | `skaffold.yaml` defines all main service images and build contexts | Run `skaffold build` or CI build. |
| Image tagging | Skaffold uses Git commit based tags | Confirm final tag/digest after push. |
| Manifest rendering | `kubernetes-manifests/` is intended for Skaffold image injection | Do not apply base manifests directly without rendering. |
| Release manifest | `release/kubernetes-manifests.yaml` uses prebuilt public images | Useful for demo validation. |
| Kustomize | Kustomize base/components exist | Run `kustomize build` or `skaffold render`. |
| Helm | Helm chart exists | Run `helm lint` and `helm template`. |
| Namespace | Default manifests do not force a dedicated environment namespace | Create/confirm namespace before deploy. |
| Registry access | Depends on chosen registry | Confirm node/task image pull permissions. |
| Storage | Default `redis-cart` uses in-cluster Redis without persistent volume | Storage class may only matter for optional persistent components. |
| Ingress/LB | Default frontend has external exposure via Kubernetes Service | Confirm cloud load balancer prerequisites and quotas. |

Important:

```text
kubernetes-manifests/ is not directly deployable as-is.
Use Skaffold to inject images, or use release/kubernetes-manifests.yaml for prebuilt public demo images.
```

## Validation Order

Use this sequence before deploying to a shared cluster:

1. Validate source and dependencies.
2. Build all images.
3. Run containers locally or in an isolated CI environment.
4. Push images to the target registry.
5. Confirm image digests.
6. Render manifests for the target environment.
7. Validate manifests offline.
8. Validate manifests against the target cluster API.
9. Confirm target namespace and permissions.
10. Confirm config, secrets, registry pull access, networking, and storage prerequisites.
11. Run `kubectl diff` or platform equivalent.
12. Get approval to deploy.

Platform rule:

```text
Render and validate the exact artifact you intend to deploy.
```

## Build Every Image

For this repo, Skaffold can build the service images defined in `skaffold.yaml`.

Build all app images:

```bash
skaffold build --default-repo=<registry>/<repo>
```

Build and deploy with remote build profile:

```bash
skaffold run -p gcb --default-repo=<registry>/<repo>
```

Build a single service manually:

```bash
docker build -t <service-name>:<tag> ./src/<service-name>
```

Checklist:

- [ ] All service images build from a clean checkout.
- [ ] Build does not depend on uncommitted local files.
- [ ] Build does not depend on files outside the build context.
- [ ] Build works on the CI runner.
- [ ] Build works for target CPU architecture.
- [ ] Image tags are immutable.
- [ ] Image digests are captured.

Services to account for:

| Service | Build context |
| --- | --- |
| `frontend` | `src/frontend` |
| `checkoutservice` | `src/checkoutservice` |
| `productcatalogservice` | `src/productcatalogservice` |
| `cartservice` | `src/cartservice/src` |
| `currencyservice` | `src/currencyservice` |
| `paymentservice` | `src/paymentservice` |
| `shippingservice` | `src/shippingservice` |
| `emailservice` | `src/emailservice` |
| `recommendationservice` | `src/recommendationservice` |
| `adservice` | `src/adservice` |
| `shoppingassistantservice` | `src/shoppingassistantservice` |
| `loadgenerator` | `src/loadgenerator` |

## Run Every Container Locally Or In CI

Container startup validation catches missing environment variables, missing files, bad entrypoints, and runtime dependency errors before cluster deployment.

Basic pattern:

```bash
docker run --rm <image>
```

With ports and environment variables:

```bash
docker run --rm \
  -p <host-port>:<container-port> \
  -e PORT=<port> \
  -e SOME_DEPENDENCY_ADDR=<host-or-service> \
  <image>
```

What to validate:

- [ ] Container process starts.
- [ ] Container does not exit immediately.
- [ ] Logs go to stdout/stderr.
- [ ] Required environment variables are accepted.
- [ ] Required files are present in the image.
- [ ] Container listens on expected port.
- [ ] Health endpoint or health RPC works if available.
- [ ] Container does not require Docker Compose to start.

For services that require dependencies, use one of:

- A local dependency container.
- A test/mock dependency.
- A CI integration test environment.
- A short-lived Kubernetes namespace.

Do not mark a service production-ready only because the image built successfully. Build success does not prove runtime startup.

## Render Manifests Before Validation

You must validate the final rendered manifest, not only the template.

Skaffold render:

```bash
skaffold render --default-repo=<registry>/<repo> > rendered-manifests.yaml
```

Kustomize render:

```bash
kustomize build kustomize/ > rendered-manifests.yaml
```

Helm render:

```bash
helm template online-boutique helm-chart/ -n <namespace> > rendered-manifests.yaml
```

Release manifest:

```bash
cp release/kubernetes-manifests.yaml rendered-manifests.yaml
```

Checklist:

- [ ] Rendered manifests include the expected namespace.
- [ ] Rendered manifests include the expected image registry.
- [ ] Rendered manifests include immutable tags or digests.
- [ ] Rendered manifests include expected environment-specific config.
- [ ] Rendered manifests do not include placeholder values.
- [ ] Rendered manifests are saved as a CI artifact.

## Validate Manifests

Server-side dry run:

```bash
kubectl apply --dry-run=server -f rendered-manifests.yaml
```

Diff against the target cluster:

```bash
kubectl diff -f rendered-manifests.yaml
```

Kustomize server-side validation:

```bash
kubectl apply --dry-run=server -k kustomize/
kubectl diff -k kustomize/
```

Helm validation:

```bash
helm lint helm-chart/
helm template online-boutique helm-chart/ -n <namespace> > rendered-manifests.yaml
kubectl apply --dry-run=server -f rendered-manifests.yaml
```

Checklist:

- [ ] YAML is syntactically valid.
- [ ] Kubernetes API accepts the resources.
- [ ] Required CRDs exist if custom resources are used.
- [ ] Deprecated API versions are not used.
- [ ] Security policies are satisfied.
- [ ] Resource requests and limits are present.
- [ ] Probes are configured.
- [ ] Services point to the correct ports.
- [ ] Image references are correct.

## Confirm Namespace Exists

Check namespaces:

```bash
kubectl get namespace
kubectl get namespace <namespace>
```

Create namespace if required:

```bash
kubectl create namespace <namespace>
```

Set context namespace:

```bash
kubectl config set-context --current --namespace=<namespace>
```

Checklist:

- [ ] Namespace exists.
- [ ] Namespace name matches environment.
- [ ] Namespace labels are correct.
- [ ] Service mesh injection labels are correct if using mesh.
- [ ] Network policies account for the namespace.
- [ ] Resource quotas and limit ranges are known.

Example namespace labels to check:

```bash
kubectl get namespace <namespace> --show-labels
```

## Confirm Registry Pull Access

Kubernetes must be able to pull every image.

Checklist:

- [ ] Images are pushed to target registry.
- [ ] Cluster nodes or workload identity can pull images.
- [ ] `imagePullSecrets` exist if required.
- [ ] ServiceAccount references `imagePullSecrets` if required.
- [ ] Registry region is reachable from the cluster.
- [ ] Image architecture matches node architecture.

Commands:

```bash
kubectl get secret -n <namespace>
kubectl get serviceaccount -n <namespace>
kubectl describe serviceaccount <service-account> -n <namespace>
```

Optional pull test:

```bash
kubectl run image-pull-test \
  --rm -i --restart=Never \
  -n <namespace> \
  --image=<registry>/<repo>/<service>:<tag> \
  --command -- true
```

Common failure symptoms:

| Symptom | Likely cause |
| --- | --- |
| `ImagePullBackOff` | Image missing, bad tag, auth failure, registry unreachable. |
| `ErrImagePull` | Pull failed before retry loop. |
| `no matching manifest for linux/amd64` | Image built for wrong CPU architecture. |
| `unauthorized` | Missing registry permission or pull secret. |

## Confirm Secrets And Config Exist

Check ConfigMaps:

```bash
kubectl get configmap -n <namespace>
```

Check Secrets:

```bash
kubectl get secret -n <namespace>
```

Describe expected object:

```bash
kubectl describe configmap <name> -n <namespace>
kubectl describe secret <name> -n <namespace>
```

Checklist:

- [ ] Required ConfigMaps exist.
- [ ] Required Secrets exist.
- [ ] Secret values are created outside Git.
- [ ] Environment-specific values are correct.
- [ ] External secret sync has completed if using External Secrets.
- [ ] Secret names match manifest references.
- [ ] ConfigMap names match manifest references.

For this app:

- Default manifests mostly rely on environment variables embedded in Deployments.
- Optional cloud integrations can require secrets, IAM annotations, and external configuration.
- Validate config carefully if using Spanner, AlloyDB, external Redis, OpenTelemetry, service mesh, or cloud operations components.

## Confirm Ingress And Load Balancer Prerequisites

The default app exposes `frontend` publicly through a Kubernetes Service in many deployment paths. Your platform may instead use Ingress, Gateway API, service mesh gateway, or an internal load balancer.

Checklist:

- [ ] Ingress controller exists if using Ingress.
- [ ] Gateway controller exists if using Gateway API.
- [ ] LoadBalancer service support exists in the cluster.
- [ ] Cloud load balancer quotas are available.
- [ ] TLS certificate exists or can be issued.
- [ ] DNS zone and record process are ready.
- [ ] Firewall/security group rules allow expected traffic.
- [ ] Internal services remain private.
- [ ] Health check path and port match the service.

Commands:

```bash
kubectl get ingressclass
kubectl get ingress -A
kubectl get svc -A --field-selector spec.type=LoadBalancer
kubectl get gatewayclass
kubectl get gateway -A
```

Cloud-specific checks:

- GKE: confirm load balancer quota, firewall rules, static IPs, managed certificates if used.
- EKS: confirm AWS Load Balancer Controller, subnets, security groups, ACM certificate, target group permissions.
- AKS: confirm ingress add-on/controller, public IP, NSG rules, managed identity permissions.
- ECS: confirm ALB/NLB, target groups, listeners, certificates, security groups, task subnet routing.

## Confirm Storage Class Exists

Default `redis-cart` in this repo does not use a PersistentVolumeClaim in the base Kubernetes demo. Storage validation becomes important if you add persistence or use optional cloud database/storage components.

Commands:

```bash
kubectl get storageclass
kubectl describe storageclass <storage-class>
kubectl get pvc -A
kubectl get pv
```

Checklist:

- [ ] Required StorageClass exists.
- [ ] Default StorageClass is understood.
- [ ] Volume binding mode is acceptable.
- [ ] Access mode matches workload need.
- [ ] Requested volume size fits quota.
- [ ] Backup/restore expectations are defined.

For ECS:

- Confirm EFS exists if using shared persistent storage.
- Confirm task role can mount/access storage.
- Confirm security groups allow NFS if using EFS.
- Confirm persistent data is not stored only on ephemeral task disk unless acceptable.

## Confirm Resource Quotas Are Not Exceeded

Commands:

```bash
kubectl get resourcequota -A
kubectl describe resourcequota -n <namespace>
kubectl get limitrange -n <namespace>
kubectl describe limitrange -n <namespace>
```

Checklist:

- [ ] Namespace CPU quota can fit requested CPU.
- [ ] Namespace memory quota can fit requested memory.
- [ ] Object count quota allows Deployments, Services, Secrets, ConfigMaps, etc.
- [ ] LoadBalancer quota allows public service if used.
- [ ] PVC/storage quota allows requested volumes.
- [ ] LimitRange defaults do not conflict with app resources.

Rough validation idea:

```text
sum(requested CPU/memory for desired replicas) <= namespace quota and cluster capacity
```

Also confirm node capacity:

```bash
kubectl top nodes
kubectl describe nodes
```

If metrics server is not installed, `kubectl top` may not work.

## Confirm Cluster Version And API Support

Commands:

```bash
kubectl version
kubectl api-versions
kubectl api-resources
```

Checklist:

- [ ] Cluster Kubernetes version is supported by your platform.
- [ ] Manifest API versions are supported.
- [ ] Required CRDs are installed.
- [ ] Ingress/Gateway API versions match the cluster.
- [ ] Native gRPC probe support is available if using native gRPC health checks.
- [ ] Pod security admission settings are understood.
- [ ] NetworkPolicy support exists if using network policies.

Static validation options:

```bash
kubeconform -strict rendered-manifests.yaml
kube-score score rendered-manifests.yaml
kubescape scan framework nsa rendered-manifests.yaml
```

Use whichever tools match your organization.

## Confirm RBAC Or IAM Permissions

Before deploy, confirm the pipeline identity can do exactly what it needs.

Kubernetes examples:

```bash
kubectl auth can-i get pods -n <namespace>
kubectl auth can-i create deployments -n <namespace>
kubectl auth can-i patch deployments -n <namespace>
kubectl auth can-i create services -n <namespace>
kubectl auth can-i create secrets -n <namespace>
```

Checklist:

- [ ] CI identity can deploy required resources.
- [ ] CI identity cannot modify unrelated namespaces.
- [ ] Runtime ServiceAccounts exist.
- [ ] Runtime ServiceAccounts have least privilege.
- [ ] Cloud IAM bindings exist if workload identity is used.

For ECS:

- CI can register task definitions.
- CI can update ECS services.
- CI can pass the required task roles.
- ECS execution role can pull images and write logs.
- ECS task role has only application-required permissions.

## Confirm External Dependencies

The app can deploy successfully and still fail at runtime if dependencies are missing.

Checklist:

- [ ] Redis, database, or external cache is reachable if externalized.
- [ ] Third-party APIs are reachable.
- [ ] Cloud service APIs are enabled.
- [ ] DNS resolution works from cluster/network.
- [ ] Egress policies allow required outbound calls.
- [ ] Certificates and trust stores are available if TLS is used.
- [ ] Credentials are available through the approved secret mechanism.

For this app, pay special attention to:

- `redis-cart` or external Redis.
- Optional Spanner/AlloyDB cart database.
- Optional OpenTelemetry collector/exporter endpoint.
- Optional Google Cloud Operations integration.
- Optional shopping assistant dependencies.
- Frontend-to-backend gRPC service addresses.

## Pre-Deployment Evidence Template

Use this table before deploying to a shared cluster:

| Validation item | Result | Evidence/link | Owner | Notes |
| --- | --- | --- | --- | --- |
| Images built |  |  |  |  |
| Images pushed |  |  |  |  |
| Image scan passed |  |  |  |  |
| Containers started |  |  |  |  |
| Manifests rendered |  |  |  |  |
| Server-side dry run passed |  |  |  |  |
| Diff reviewed |  |  |  |  |
| Namespace confirmed |  |  |  |  |
| Registry pull access confirmed |  |  |  |  |
| Secrets/config confirmed |  |  |  |  |
| Networking prerequisites confirmed |  |  |  |  |
| Storage prerequisites confirmed |  |  |  |  |
| Quota/capacity confirmed |  |  |  |  |
| Cluster API support confirmed |  |  |  |  |
| Approval received |  |  |  |  |

## Kubernetes Pre-Deployment Command Set

Replace placeholders before running.

```bash
export NAMESPACE=<namespace>
export REGISTRY=<registry>/<repo>

skaffold render --default-repo=$REGISTRY > rendered-manifests.yaml

kubectl get namespace $NAMESPACE
kubectl apply --dry-run=server -f rendered-manifests.yaml -n $NAMESPACE
kubectl diff -f rendered-manifests.yaml -n $NAMESPACE

kubectl get storageclass
kubectl get resourcequota -A
kubectl get limitrange -n $NAMESPACE
kubectl auth can-i create deployments -n $NAMESPACE
kubectl auth can-i create services -n $NAMESPACE
kubectl get secret -n $NAMESPACE
kubectl get configmap -n $NAMESPACE
```

## ECS Pre-Deployment Command Set

Replace placeholders before running.

```bash
aws ecs describe-clusters --clusters <cluster-name>
aws ecs describe-services --cluster <cluster-name> --services <service-name>
aws ecr describe-images --repository-name <repo-name> --image-ids imageTag=<tag>
aws iam simulate-principal-policy \
  --policy-source-arn <ci-role-arn> \
  --action-names ecs:RegisterTaskDefinition ecs:UpdateService iam:PassRole
aws elbv2 describe-target-groups --names <target-group-name>
aws logs describe-log-groups --log-group-name-prefix <prefix>
```

ECS checklist:

- [ ] Cluster exists.
- [ ] Service exists or creation is planned.
- [ ] Task execution role can pull image and write logs.
- [ ] Task role has application permissions.
- [ ] Task definition renders correctly.
- [ ] Target group exists.
- [ ] Listener/rules exist.
- [ ] Security groups allow expected traffic.
- [ ] Subnets have required routing.
- [ ] Secrets Manager/SSM parameters exist.

## Common Pre-Deployment Failures

| Failure | Likely cause | What to check |
| --- | --- | --- |
| Dry run fails | Invalid manifest or unsupported API | `kubectl apply --dry-run=server`, `kubectl version`. |
| Diff shows unexpected namespace | Manifest rendered with wrong environment | Kustomize overlay, Helm values, Skaffold namespace. |
| Image pull will fail | Image not pushed or auth missing | Registry, tag, digest, pull secret/IAM. |
| Pods cannot schedule | Resource requests exceed capacity or quota | ResourceQuota, node capacity, requests/limits. |
| LoadBalancer stuck pending | Cloud LB quota/controller/subnet issue | Cloud controller logs, quotas, subnets, annotations. |
| Secrets missing | Secret creation skipped or wrong namespace | `kubectl get secret -n <namespace>`. |
| Service mesh injection fails | Namespace label/webhook problem | Namespace labels, webhook health, mesh control plane. |
| Network policies block traffic | Policies too strict or missing allow rules | NetworkPolicy objects and service dependencies. |
| Storage PVC pending | StorageClass or quota issue | StorageClass, PVC events, cloud disk quota. |

## Go/No-Go For Deployment

Go only if:

- [ ] Images are built, pushed, scanned, and pullable.
- [ ] Containers can start with required configuration.
- [ ] Manifests are rendered for the correct environment.
- [ ] Server-side dry run passes.
- [ ] Diff is reviewed and expected.
- [ ] Namespace and permissions are ready.
- [ ] Secrets and config exist.
- [ ] Networking and exposure prerequisites are ready.
- [ ] Storage prerequisites are ready or confirmed unnecessary.
- [ ] Resource quotas and cluster capacity are sufficient.
- [ ] Cluster API support is confirmed.
- [ ] Rollback artifact is known.

No-go if:

- [ ] Image tag is missing or mutable.
- [ ] Dry run fails.
- [ ] Diff includes unexpected destructive changes.
- [ ] Required secrets are missing.
- [ ] Registry pull access is unverified.
- [ ] Required ingress/load balancer prerequisites are missing.
- [ ] Required storage or database prerequisites are missing.
- [ ] Rollback target is unknown.

## Output Checklist

Before moving to post-deployment validation, produce:

- [ ] Build evidence.
- [ ] Runtime startup evidence.
- [ ] Rendered manifest artifact.
- [ ] Dry-run result.
- [ ] Diff review result.
- [ ] Namespace/RBAC confirmation.
- [ ] Registry pull confirmation.
- [ ] Secrets/config confirmation.
- [ ] Networking prerequisite confirmation.
- [ ] Storage prerequisite confirmation.
- [ ] Quota/capacity confirmation.
- [ ] Cluster version/API confirmation.
- [ ] Rollback artifact reference.

## Ready For Next Checklist

After pre-deployment validation is complete, move to:

```text
16. Post-Deployment Validation
```

In that stage, prove the deployed app actually works from the cluster outward: pods/tasks, readiness, services, DNS, internal traffic, public endpoint, logs, metrics, traces, smoke tests, and basic load tests.
