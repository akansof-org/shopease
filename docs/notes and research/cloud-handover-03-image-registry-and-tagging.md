# Cloud Handover Checklist: 3. Image Registry And Tagging

Scenario: the development team has handed this application to the platform engineering team. We are preparing it for cloud infrastructure, instrumentation, and deployment to Kubernetes or ECS.

This note covers the third checklist item only: **Image Registry And Tagging**.

## Objective

The cluster must be able to pull every container image needed by the application.

This stage answers:

- Where will images be stored?
- How will images be named?
- How will images be tagged?
- Which CPU architecture will images support?
- How will images be pushed?
- How will Kubernetes nodes or ECS tasks authenticate to pull images?
- How will image scanning and promotion work?

## Checklist

- [ ] Choose target registry: ECR, Artifact Registry, Docker Hub, ACR, private registry.
- [ ] Define image naming convention.
- [ ] Define tag strategy: Git SHA, semantic version, build number, environment tag.
- [ ] Build images for the correct CPU architecture: `amd64`, `arm64`, or multi-arch.
- [ ] Push images to registry.
- [ ] Confirm cluster nodes/tasks can pull images.
- [ ] Configure image pull secrets if required.
- [ ] Enable image vulnerability scanning if available.
- [ ] Avoid `latest` for production deployments.

## Current Repo State

This repo already defines image build metadata in `skaffold.yaml`.

Important current settings:

| Item | Current state |
| --- | --- |
| Build tool | Skaffold |
| Image artifacts | Defined in `skaffold.yaml` |
| Tag policy | Git commit based |
| Target platforms | `linux/amd64`, `linux/arm64` |
| Local build engine | Docker CLI with BuildKit |
| Kubernetes manifests | Use placeholder image names |
| Release manifests | Use prebuilt public images |

Important repo behavior:

```text
kubernetes-manifests/ are not meant to be applied directly.
They contain short image names like frontend, cartservice, paymentservice.
Skaffold rewrites those image names with the selected registry and tag.
```

For direct cluster deployment without building images yourself, the repo also includes:

```text
release/kubernetes-manifests.yaml
```

That file references prebuilt public images. For a real platform handover, prefer building and pushing your own approved images to your organization registry.

## Image Inventory

These are the images that need to exist in the target registry for the default app.

| Image | Build context | Required for default app? |
| --- | --- | --- |
| `frontend` | `src/frontend` | Yes |
| `productcatalogservice` | `src/productcatalogservice` | Yes |
| `currencyservice` | `src/currencyservice` | Yes |
| `cartservice` | `src/cartservice/src` | Yes |
| `shippingservice` | `src/shippingservice` | Yes |
| `paymentservice` | `src/paymentservice` | Yes |
| `emailservice` | `src/emailservice` | Yes |
| `checkoutservice` | `src/checkoutservice` | Yes |
| `recommendationservice` | `src/recommendationservice` | Yes |
| `adservice` | `src/adservice` | Yes |
| `loadgenerator` | `src/loadgenerator` | No, testing/demo only |
| `shoppingassistantservice` | `src/shoppingassistantservice` | No, optional AI feature |
| `redis` | public `redis:alpine` image | Yes, unless replaced with managed Redis |

## Choose Target Registry

Choose the registry based on the cloud platform and organization standards.

| Platform | Common registry |
| --- | --- |
| AWS ECS/EKS | Amazon ECR |
| Google GKE | Artifact Registry |
| Azure AKS | Azure Container Registry |
| Generic Kubernetes | Docker Hub, Harbor, Quay, GitLab registry, private OCI registry |

Decision template:

```text
Target cloud/platform:
Target registry:
Registry region:
Registry project/account/subscription:
Registry access model:
Image scanning tool:
Retention policy:
```

Example choices:

```text
AWS:
<aws-account-id>.dkr.ecr.<region>.amazonaws.com/online-boutique/<service>:<tag>

GCP:
<region>-docker.pkg.dev/<project-id>/online-boutique/<service>:<tag>

Azure:
<registry-name>.azurecr.io/online-boutique/<service>:<tag>

Docker Hub:
docker.io/<org>/online-boutique-<service>:<tag>
```

## Image Naming Convention

Use a predictable naming convention.

Recommended pattern:

```text
<registry>/<team-or-app>/<service>:<tag>
```

Example:

```text
us-docker.pkg.dev/acme-platform/online-boutique/frontend:git-abc1234
us-docker.pkg.dev/acme-platform/online-boutique/cartservice:git-abc1234
```

Alternative flat pattern:

```text
<registry>/<service>:<tag>
```

Example:

```text
123456789012.dkr.ecr.us-east-1.amazonaws.com/frontend:git-abc1234
```

Recommended for this app:

```text
<registry>/online-boutique/<service>:<git-sha>
```

Reason:

- Keeps all service images grouped under one app/repository namespace.
- Makes it easy to identify ownership.
- Works well for Kubernetes, ECS, and CI/CD promotion.

## Tag Strategy

Avoid `latest` for production.

Recommended tags:

| Tag type | Example | Use |
| --- | --- | --- |
| Git SHA | `3f2a9c1` | Best immutable deploy reference |
| Full Git SHA | `3f2a9c19...` | Maximum traceability |
| Semantic version | `v1.4.2` | Release-oriented teams |
| Build number | `build-1042` | CI-oriented tracking |
| Environment tag | `dev`, `staging`, `prod` | Convenience pointer, not immutable |

Recommended production approach:

```text
Deploy immutable tags, usually Git SHA or release version.
Use environment tags only as metadata or convenience pointers.
Do not deploy latest.
```

This repo's Skaffold config already uses:

```text
tagPolicy:
  gitCommit: {}
```

That means Skaffold tags images using Git commit information.

## CPU Architecture

The cluster's node architecture must match the images.

This repo's Skaffold config requests:

```text
linux/amd64
linux/arm64
```

Checklist:

- [ ] Confirm the cluster node architecture.
- [ ] Confirm local builders or CI builders support required architecture.
- [ ] Confirm base images support required architecture.
- [ ] Confirm native dependencies can build for target architecture.
- [ ] Confirm multi-arch manifests are pushed if supporting both `amd64` and `arm64`.

Common cases:

| Cluster/node type | Image architecture needed |
| --- | --- |
| Most x86 cloud nodes | `linux/amd64` |
| Graviton/ARM nodes | `linux/arm64` |
| Mixed node pools | Multi-arch images |

## Build And Push With Skaffold

Preferred repo-native flow:

```sh
skaffold build --default-repo=<registry>/<repo>
```

Example:

```sh
skaffold build --default-repo=us-docker.pkg.dev/my-project/online-boutique
```

Build and deploy:

```sh
skaffold run --default-repo=<registry>/<repo>
```

Example:

```sh
skaffold run --default-repo=us-docker.pkg.dev/my-project/online-boutique
```

Google Cloud Build profile:

```sh
skaffold run -p gcb --default-repo=us-docker.pkg.dev/my-project/online-boutique
```

Use the cloud build profile when:

- local Docker is unavailable
- local machine is too small
- organization requires builds inside cloud CI
- you need a cleaner build environment

## Build And Push Manually

Manual pattern:

```sh
docker build -t <registry>/<app>/<service>:<git-sha> ./src/<service>
docker push <registry>/<app>/<service>:<git-sha>
```

Example:

```sh
docker build -t <registry>/online-boutique/frontend:<git-sha> ./src/frontend
docker push <registry>/online-boutique/frontend:<git-sha>
```

For this app, manual commands would look like:

```sh
docker build -t <registry>/online-boutique/frontend:<git-sha> ./src/frontend
docker build -t <registry>/online-boutique/productcatalogservice:<git-sha> ./src/productcatalogservice
docker build -t <registry>/online-boutique/currencyservice:<git-sha> ./src/currencyservice
docker build -t <registry>/online-boutique/cartservice:<git-sha> ./src/cartservice/src
docker build -t <registry>/online-boutique/shippingservice:<git-sha> ./src/shippingservice
docker build -t <registry>/online-boutique/paymentservice:<git-sha> ./src/paymentservice
docker build -t <registry>/online-boutique/emailservice:<git-sha> ./src/emailservice
docker build -t <registry>/online-boutique/checkoutservice:<git-sha> ./src/checkoutservice
docker build -t <registry>/online-boutique/recommendationservice:<git-sha> ./src/recommendationservice
docker build -t <registry>/online-boutique/adservice:<git-sha> ./src/adservice
```

Then push:

```sh
docker push <registry>/online-boutique/frontend:<git-sha>
docker push <registry>/online-boutique/productcatalogservice:<git-sha>
docker push <registry>/online-boutique/currencyservice:<git-sha>
docker push <registry>/online-boutique/cartservice:<git-sha>
docker push <registry>/online-boutique/shippingservice:<git-sha>
docker push <registry>/online-boutique/paymentservice:<git-sha>
docker push <registry>/online-boutique/emailservice:<git-sha>
docker push <registry>/online-boutique/checkoutservice:<git-sha>
docker push <registry>/online-boutique/recommendationservice:<git-sha>
docker push <registry>/online-boutique/adservice:<git-sha>
```

Optional images:

```sh
docker build -t <registry>/online-boutique/loadgenerator:<git-sha> ./src/loadgenerator
docker build -t <registry>/online-boutique/shoppingassistantservice:<git-sha> ./src/shoppingassistantservice

docker push <registry>/online-boutique/loadgenerator:<git-sha>
docker push <registry>/online-boutique/shoppingassistantservice:<git-sha>
```

## Registry Authentication

The build system needs push access. The cluster needs pull access.

### Kubernetes

Registry pull access options:

- Node service account or instance role has registry pull permission.
- Workload identity or managed identity grants pull access.
- `imagePullSecrets` are configured in the namespace.
- Registry is public.

Checklist:

- [ ] Confirm build identity can push images.
- [ ] Confirm node identity can pull images.
- [ ] Confirm private registry auth is configured if needed.
- [ ] Confirm target namespace has required `imagePullSecrets`.
- [ ] Confirm deployments reference the correct image registry.

Kubernetes pull secret example:

```sh
kubectl create secret docker-registry regcred \
  --docker-server=<registry> \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=<email>
```

Deployment reference:

```yaml
imagePullSecrets:
  - name: regcred
```

### ECS

Registry pull access options:

- ECS task execution role can pull from ECR.
- Private registry credentials are stored in Secrets Manager.
- Public registry does not require authentication.

Checklist:

- [ ] Confirm task execution role exists.
- [ ] Confirm task execution role can pull images.
- [ ] Confirm private registry credentials are configured if needed.
- [ ] Confirm image URI is reachable from the ECS task network path.

## Vulnerability Scanning

Enable image scanning before production.

Checklist:

- [ ] Enable registry-native vulnerability scanning.
- [ ] Run CI image scanning if required.
- [ ] Define severity threshold for blocking deployment.
- [ ] Track base image vulnerabilities.
- [ ] Track app dependency vulnerabilities.
- [ ] Rebuild images when base images are patched.

Common scanning options:

| Platform | Scanning options |
| --- | --- |
| AWS | ECR basic/enhanced scanning, Inspector |
| GCP | Artifact Analysis |
| Azure | Microsoft Defender for Cloud / ACR scanning integrations |
| Generic | Trivy, Grype, Snyk, Prisma, Aqua, Anchore |

Example:

```sh
trivy image <registry>/online-boutique/frontend:<git-sha>
```

## Manifest Image Replacement

This repo has two deployment styles.

### Skaffold-managed deployment

Use:

```sh
skaffold run --default-repo=<registry>/<repo>
```

Skaffold:

- builds images
- tags images
- pushes images
- renders Kubernetes manifests
- replaces short image names with full image references
- deploys with `kubectl`

### Prebuilt release manifests

Use:

```sh
kubectl apply -f release/kubernetes-manifests.yaml
```

This uses public prebuilt images.

Platform recommendation:

```text
For owned environments, build and deploy your own images.
Use release manifests for quick demo validation only.
```

## Production Tagging Rules

Recommended rules:

- Do not deploy `latest`.
- Do not overwrite immutable release tags.
- Use Git SHA or semantic version for deployments.
- Record the image digest for each deployment.
- Promote the same image across environments instead of rebuilding for each environment.
- Keep environment differences in config, not in rebuilt images.

Good:

```text
frontend:3f2a9c1
frontend:v1.2.0
frontend@sha256:<digest>
```

Avoid:

```text
frontend:latest
frontend:prod
frontend:stable
```

Environment tags can exist as convenience aliases, but the deployment record should always include the immutable tag or digest.

## Image Registry Decision Record

Fill this in during handover.

```text
Target platform:
Registry:
Registry region:
App/repository namespace:
Required architectures:
Tag format:
Image scanning tool:
Build system:
Push identity:
Cluster pull identity:
imagePullSecret required:
Retention policy:
Promotion strategy:
```

## Validation Commands

Confirm images exist:

```sh
docker pull <registry>/online-boutique/frontend:<git-sha>
```

Confirm image architecture:

```sh
docker buildx imagetools inspect <registry>/online-boutique/frontend:<git-sha>
```

Confirm Kubernetes can pull:

```sh
kubectl run image-pull-test \
  --image=<registry>/online-boutique/frontend:<git-sha> \
  --restart=Never
```

Check result:

```sh
kubectl get pod image-pull-test
kubectl describe pod image-pull-test
```

Clean up:

```sh
kubectl delete pod image-pull-test
```

Confirm rendered manifest images:

```sh
skaffold render --default-repo=<registry>/<repo>
```

Look for fully qualified image names, not placeholder names.

## Risks And Follow-Ups

| Area | Risk | Follow-up |
| --- | --- | --- |
| Registry choice | Images may be pushed to a registry the cluster cannot access | Confirm cluster pull permissions early |
| Image tags | Mutable tags make rollback and audit harder | Use Git SHA or digest |
| Architecture | ARM cluster cannot run AMD-only image | Build multi-arch or target correct platform |
| Manifest rendering | Applying `kubernetes-manifests/` directly leaves placeholder image names | Use Skaffold or release manifests |
| Optional components | Building all images includes optional services not deployed by default | Decide whether to include `loadgenerator` and `shoppingassistantservice` |
| Public base images | `redis:alpine` is pulled from public registry | Mirror to private registry if required by org policy |
| Scanning | Vulnerabilities may be missed if scanning is not enabled | Add registry or CI scanning gate |

## Image Registry And Tagging Output

At the end of this stage, platform engineering should have:

- [ ] Target registry selected.
- [ ] Image naming convention documented.
- [ ] Tagging strategy documented.
- [ ] Required architectures confirmed.
- [ ] Images built and pushed.
- [ ] Cluster pull access confirmed.
- [ ] Pull secrets configured if needed.
- [ ] Image scanning enabled.
- [ ] Decision made about optional images.
- [ ] Decision made about mirroring public images like Redis.
- [ ] Manifest rendering strategy confirmed.

## Ready For Next Checklist Item

The next stage is:

```text
4. Runtime Configuration
```

In that stage, document every environment variable, secret, service address, feature flag, and per-environment config value needed to run the app safely.

