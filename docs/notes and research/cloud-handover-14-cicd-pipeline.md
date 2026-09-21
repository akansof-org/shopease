# Cloud Handover Checklist: 14. CI/CD Pipeline

Scenario: the development team has handed this application to the platform engineering team. We are preparing it for cloud infrastructure, instrumentation, and deployment to Kubernetes or ECS.

This note covers the fourteenth checklist item only: **CI/CD Pipeline**.

## Objective

The pipeline should produce repeatable deployments.

This stage answers:

- Are tests run automatically?
- Are all service images built the same way every time?
- Are images scanned before deployment?
- Are images pushed to a registry the cluster can pull from?
- Are manifests rendered and validated before apply?
- Is dev deployment automatic?
- Are smoke tests part of the pipeline?
- How are staging and production promotions controlled?
- Are image tags, digests, manifests, and deployment metadata recorded?
- Can the pipeline support rollback?

## Checklist

- [x] Run tests.
- [x] Build images.
- [ ] Scan images.
- [x] Push images.
- [x] Render manifests.
- [x] Validate manifests.
- [x] Deploy to dev.
- [x] Run smoke tests.
- [ ] Promote to staging/prod.
- [ ] Record image tags and deployment metadata.
- [ ] Support rollback.

Note:

- The repo has GitHub Actions workflows for tests, PR deployments, smoke tests, manifest validation, Helm validation, Terraform validation, cleanup, and manual release building.
- The repo does not fully define your production promotion, approval, image scanning, deployment metadata, or rollback automation policy.
- Treat the existing workflows as a strong starting point, then adapt them to your target registry, cluster, IAM, and release process.

## Pipeline Stages

```text
source
  -> test
  -> build image
  -> scan image
  -> push image
  -> render manifests
  -> validate manifests
  -> deploy
  -> smoke test
  -> promote
  -> record metadata
```

Platform interpretation:

```text
A deployment should be reproducible from source code, immutable image references,
versioned configuration, rendered manifests, and recorded release metadata.
```

## Current Repo CI/CD Inventory

| Workflow/script | Location | What it does | Platform interpretation |
| --- | --- | --- | --- |
| Pull request CI | `.github/workflows/ci-pr.yaml` | Runs Go and C# tests, builds/deploys PR images to a GKE namespace, waits for pods, runs smoke test | Good PR validation pattern. |
| Main/release CI | `.github/workflows/ci-main.yaml` | Runs tests, deploys built images to a GKE namespace, runs smoke test | Good integration validation pattern. |
| Manual release builder | `.github/workflows/make-release.yaml` | Builds/pushes release images, regenerates manifests and Helm chart, creates release branch/tag/PR | Good release artifact process. |
| Kustomize build CI | `.github/workflows/kustomize-build-ci.yaml` | Runs `kubectl kustomize` and test builds | Validates Kustomize output. |
| Helm chart CI | `.github/workflows/helm-chart-ci.yaml` | Runs `helm lint`, `helm template`, and Kustomize build checks for chart scenarios | Validates Helm packaging paths. |
| Kubevious manifest CI | `.github/workflows/kubevious-manifests-ci.yaml` | Validates Kubernetes manifests, Helm chart, and Kustomize config | Static Kubernetes manifest validation. |
| Terraform validation | `.github/workflows/terraform-validate-ci.yaml` | Runs `terraform init -backend=false` and `terraform validate` | Validates infrastructure code syntax. |
| Cleanup | `.github/workflows/cleanup.yaml` | Cleans up PR namespace after PR close | Prevents test environment buildup. |
| Release scripts | `docs/releasing/*.sh` | Build images, generate release artifacts, package Helm chart, tag release | Useful for release automation or CI migration. |

## Existing PR Pipeline Behavior

The current PR workflow does this:

1. Checks out the code.
2. Sets up Go and .NET runtimes.
3. Runs unit tests for selected services.
4. Authenticates to Google Cloud using workload identity.
5. Configures Docker/GKE/Skaffold.
6. Builds and deploys images into a PR-specific namespace.
7. Waits for each Deployment to become available.
8. Gets the frontend external IP.
9. Uses the load generator logs as a smoke test.
10. Comments the staged URL back on the PR.

This is a strong pattern because it validates more than code compilation. It proves the app can build, deploy, start, receive traffic, and run a basic user flow.

## Gaps To Close For Your Platform Pipeline

| Gap | Why it matters | Recommended action |
| --- | --- | --- |
| Image scanning not clearly enforced | Vulnerable images can reach the cluster | Add registry scanning, Trivy, Grype, Snyk, Prisma, or cloud-native scanner gate. |
| Production promotion not fully defined | Main branch validation is not the same as production release control | Add staging/prod jobs with approvals. |
| Deployment metadata not fully recorded | Rollback and audit need exact artifact history | Record Git SHA, image tag, digest, manifest version, env, actor, and timestamp. |
| Rollback automation not fully defined | Manual rollback during incident is slower | Add rollback job or documented command that redeploys previous artifact. |
| Tests are selective | Some service languages may lack complete test coverage | Expand per-service test matrix over time. |
| Environment config promotion is not fully modeled | Same image may behave differently due to config drift | Version ConfigMaps, Secrets references, Helm values, or Kustomize overlays. |
| Production approvals are organization-specific | CI cannot know who owns risk by default | Use protected environments or change management approval. |

## Recommended Pipeline Design

Use separate workflows or jobs for:

| Stage | Trigger | Output |
| --- | --- | --- |
| Pull request validation | PR opened/updated | Test result, rendered manifest validation, optional preview namespace. |
| Main branch build | Merge to `main` | Immutable images pushed to registry. |
| Dev deploy | Merge to `main` or manual trigger | Dev environment updated and smoke tested. |
| Staging promotion | Manual or release branch/tag | Same image promoted to staging and verified. |
| Production promotion | Manual approval after staging passes | Same image promoted to production. |
| Rollback | Manual incident trigger | Previous known-good image or manifest redeployed. |

Recommended flow:

```text
pull request
  -> unit tests
  -> manifest validation
  -> preview deploy
  -> smoke test

merge to main
  -> build images
  -> scan images
  -> push images
  -> render manifests
  -> deploy to dev
  -> smoke test

release candidate
  -> promote same image digests to staging
  -> run integration/smoke/performance checks
  -> approval
  -> promote same image digests to production
  -> verify
```

## Source Stage

The source stage decides what code version enters the pipeline.

Checklist:

- [ ] Require pull request review before merge.
- [ ] Protect `main` or release branches.
- [ ] Require CI checks before merge.
- [ ] Require signed commits or verified commits if your org requires it.
- [ ] Capture Git SHA for every build.
- [ ] Capture branch, PR number, release tag, and actor.

Useful metadata:

```text
GIT_SHA
GIT_BRANCH
GITHUB_RUN_ID
GITHUB_RUN_ATTEMPT
PR_NUMBER
RELEASE_VERSION
```

## Test Stage

Current repo tests:

- Go unit tests for selected services.
- C# unit tests for `cartservice`.
- Manifest-focused tests for Kustomize and Helm.
- Deployment smoke tests through the load generator.

Recommended test expansion:

| Service/runtime | Test command pattern |
| --- | --- |
| Go | `go test ./...` |
| Java | `./gradlew test` or `mvn test` depending on project structure |
| Node.js | `npm ci && npm test` |
| Python | `pip install -r requirements.txt && pytest` |
| .NET | `dotnet test` |
| Manifests | `kubectl kustomize`, `kustomize build`, `helm lint`, `helm template` |

Platform checklist:

- [ ] Tests run before image build.
- [ ] Tests fail the pipeline on error.
- [ ] Tests cover all services over time.
- [ ] Tests are not dependent on a developer laptop.
- [ ] Integration tests are separate from unit tests.
- [ ] Slow tests have clear timeouts.

## Build Image Stage

This repo uses Skaffold to build service images:

```bash
skaffold build --default-repo=<registry>/<repo>
```

For deployment:

```bash
skaffold run --default-repo=<registry>/<repo>
```

The `skaffold.yaml` file defines service build contexts and uses Git commit based tagging:

```yaml
tagPolicy:
  gitCommit: {}
```

Build checklist:

- [ ] Build every required service image.
- [ ] Build from a clean checkout.
- [ ] Build for target CPU architecture: `amd64`, `arm64`, or multi-arch.
- [ ] Use BuildKit/build cache where appropriate.
- [ ] Fail the pipeline if any image fails to build.
- [ ] Capture image tag and digest after push.
- [ ] Avoid mutable production tags like `latest`.

## Scan Image Stage

Image scanning should happen before production deployment.

Scanner options:

| Option | Example |
| --- | --- |
| Registry-native scanning | ECR enhanced scanning, Artifact Registry scanning, ACR Defender, Docker Hub Scout |
| CLI scanners | Trivy, Grype, Snyk, Docker Scout |
| Enterprise scanners | Prisma Cloud, Aqua, Wiz, Lacework |

Checklist:

- [ ] Scan every service image.
- [ ] Fail on critical vulnerabilities according to policy.
- [ ] Decide whether high vulnerabilities block production.
- [ ] Ignore or accept findings only through a documented exception process.
- [ ] Scan base images and application dependencies.
- [ ] Store scan results as pipeline artifacts.
- [ ] Re-scan images periodically even if code has not changed.

Example with Trivy:

```bash
trivy image --severity CRITICAL,HIGH --exit-code 1 <registry>/<repo>/<service>:<tag>
```

## Push Image Stage

The pipeline must push images to a registry the cluster can pull from.

Checklist:

- [ ] Authenticate to registry using CI identity.
- [ ] Push all service images.
- [ ] Confirm push succeeded.
- [ ] Capture image digest.
- [ ] Confirm cluster pull identity has access.
- [ ] Do not push production images only to a developer-local registry.

Example:

```bash
docker push <registry>/<app>/<service>:<git-sha>
docker inspect --format='{{index .RepoDigests 0}}' <registry>/<app>/<service>:<git-sha>
```

Skaffold usually handles build and push together when targeting a remote cluster:

```bash
skaffold build --default-repo=<registry>/<repo>
```

## Render Manifests Stage

Rendering manifests turns templates into concrete Kubernetes YAML.

Skaffold render:

```bash
skaffold render --default-repo=<registry>/<repo> > rendered-manifests.yaml
```

Kustomize build:

```bash
kustomize build kustomize/ > rendered-manifests.yaml
```

Helm template:

```bash
helm template online-boutique helm-chart/ -n online-boutique > rendered-manifests.yaml
```

Checklist:

- [ ] Render manifests before deployment.
- [ ] Render with the correct environment overlay or values.
- [ ] Render with immutable image tags or digests.
- [ ] Store rendered manifests as pipeline artifacts.
- [ ] Review rendered manifests for production changes.

## Validate Manifests Stage

The repo already validates Kustomize, Helm, and Kubernetes manifests.

Recommended validation layers:

| Validation | Example command/tool |
| --- | --- |
| Syntax/rendering | `kustomize build`, `helm template`, `helm lint` |
| Server-side validation | `kubectl apply --dry-run=server -f rendered-manifests.yaml` |
| Diff | `kubectl diff -f rendered-manifests.yaml` |
| Policy | Kyverno, OPA Gatekeeper, Conftest |
| Kubernetes static analysis | kubeconform, kube-score, kubevious |
| Security posture | Kubescape, Polaris |

Example:

```bash
kubectl apply --dry-run=server -f rendered-manifests.yaml
kubectl diff -f rendered-manifests.yaml
```

Checklist:

- [ ] Validate YAML syntax.
- [ ] Validate Kubernetes schema.
- [ ] Validate against cluster API version.
- [ ] Validate policy requirements.
- [ ] Validate security context requirements.
- [ ] Validate resource requests and limits.
- [ ] Validate probes.
- [ ] Validate image references.

## Deploy Stage

Kubernetes deployment options:

```bash
skaffold run --default-repo=<registry>/<repo> --namespace=<namespace>
kubectl apply -f rendered-manifests.yaml
kubectl apply -k kustomize/
helm upgrade --install online-boutique helm-chart/ -n online-boutique
```

ECS deployment options:

```bash
aws ecs register-task-definition --cli-input-json file://task-definition.json
aws ecs update-service --cluster <cluster> --service <service> --task-definition <task-def-revision>
```

Checklist:

- [ ] Deploy to the correct environment.
- [ ] Confirm namespace, cluster, account, and region.
- [ ] Use environment-specific configuration.
- [ ] Use least-privilege CI identity.
- [ ] Wait for rollout to complete.
- [ ] Fail the pipeline if rollout fails.

Kubernetes rollout checks:

```bash
kubectl rollout status deployment/frontend -n <namespace>
kubectl get pods -n <namespace>
kubectl get svc -n <namespace>
```

## Smoke Test Stage

The repo uses the load generator logs as a smoke test in CI. That is useful because it checks a real request path instead of only confirming pods are running.

Minimum smoke tests:

- [ ] Frontend responds.
- [ ] Health endpoint responds.
- [ ] Basic product listing works.
- [ ] Add-to-cart path works if testable.
- [ ] Checkout path works in a controlled test mode if safe.
- [ ] Internal services have no obvious startup errors.
- [ ] Load generator or synthetic test shows no elevated errors.

Example:

```bash
curl -f http://<frontend-host>/
curl -f http://<frontend-host>/_healthz
```

Kubernetes internal test pattern:

```bash
kubectl run smoke-test \
  --rm -i --restart=Never \
  --image=curlimages/curl \
  -- curl -f http://frontend:80/_healthz
```

## Promotion Stage

Promotion should move the same artifact forward.

Recommended:

```text
dev image digest == staging image digest == production image digest
```

Promotion checklist:

- [ ] Promote by image digest or immutable tag.
- [ ] Do not rebuild for staging/prod.
- [ ] Use environment-specific config only where needed.
- [ ] Require approval before production.
- [ ] Require staging smoke tests before production.
- [ ] Record who approved and when.
- [ ] Record what was promoted.

Promotion models:

| Model | How it works | Good for |
| --- | --- | --- |
| CI/CD job promotion | Workflow manually promotes artifact to next env | Teams using GitHub Actions, GitLab CI, Jenkins, CircleCI |
| GitOps promotion | Pipeline updates environment repo or overlay | Argo CD, Flux, audited environment changes |
| Release branch/tag | Release tag triggers staging/prod workflow | Versioned release management |
| Manual release artifact | Human applies approved manifest/chart | Small teams or early migration |

## Metadata Recording

Record deployment metadata automatically.

Minimum metadata:

| Field | Example |
| --- | --- |
| App | `online-boutique` |
| Service | `frontend` |
| Environment | `staging` |
| Git SHA | `9f3a2c1...` |
| Image tag | `frontend:9f3a2c1` |
| Image digest | `sha256:...` |
| Manifest artifact | `rendered-manifests.yaml` |
| Pipeline run | GitHub Actions run ID |
| Actor | User or service account |
| Deployment time | Timestamp |
| Approval | Approver identity |
| Rollback target | Previous known-good image/digest |

Store metadata in one or more:

- GitHub deployment environments.
- Release notes.
- CI artifacts.
- Change management ticket.
- GitOps commit.
- Kubernetes annotations.
- Artifact registry metadata.
- Deployment dashboard.

Useful Kubernetes annotations:

```yaml
metadata:
  annotations:
    app.kubernetes.io/version: "<git-sha-or-release>"
    deployment.platform.example.com/pipeline-run: "<run-id>"
    deployment.platform.example.com/image-digest: "<digest>"
```

## Rollback Support

The pipeline should make rollback boring and fast.

Kubernetes rollback:

```bash
kubectl rollout history deployment/<name> -n <namespace>
kubectl rollout undo deployment/<name> -n <namespace>
kubectl rollout status deployment/<name> -n <namespace>
```

Rollback by applying previous manifest:

```bash
kubectl apply -f rendered-manifests-previous-known-good.yaml
```

ECS rollback:

```bash
aws ecs update-service \
  --cluster <cluster-name> \
  --service <service-name> \
  --task-definition <previous-task-definition-revision>
```

Rollback checklist:

- [ ] Previous known-good image is recorded.
- [ ] Previous rendered manifest is stored.
- [ ] Previous task definition or Helm release revision is available.
- [ ] Rollback command is tested in staging.
- [ ] Rollback job requires the right approval.
- [ ] Rollback verifies service health after completion.
- [ ] Rollback handles migrations safely.

## Required CI/CD Secrets And Permissions

Typical secrets and identities:

| Need | Kubernetes/GKE example | ECS/AWS example |
| --- | --- | --- |
| Registry push | Artifact Registry writer | ECR push permissions |
| Cluster deploy | GKE deployer via workload identity | ECS service deploy role |
| Secret access | Secret Manager accessor | Secrets Manager/SSM read |
| Manifest deploy | Kubernetes RBAC for target namespace | IAM permissions for ECS/ALB/task definitions |
| Release creation | GitHub token or app token | GitHub token or app token |

Recommended:

- Use OIDC/workload identity instead of long-lived static cloud keys.
- Use separate identities for dev, staging, and production.
- Restrict production deploy identity.
- Protect production environment in CI/CD.
- Audit every deployment.

## Example GitHub Actions Pipeline Shape

This is a simplified target shape, not a drop-in workflow:

```yaml
name: deploy

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: go test ./...

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: skaffold build --default-repo=<registry>/<repo>

  validate:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: skaffold render --default-repo=<registry>/<repo> > rendered-manifests.yaml
      - run: kubectl apply --dry-run=server -f rendered-manifests.yaml

  deploy-dev:
    needs: validate
    runs-on: ubuntu-latest
    environment: dev
    steps:
      - run: kubectl apply -f rendered-manifests.yaml
      - run: kubectl rollout status deployment/frontend -n online-boutique-dev
```

For production, add protected environments, approvals, immutable image digests, smoke tests, and rollback support.

## Platform Decision Table

| Decision | Recommendation |
| --- | --- |
| CI system | GitHub Actions is already present; continue unless your org standard differs. |
| Build tool | Use Skaffold for multi-service image build/render/deploy. |
| Registry | Use your cloud registry: ECR, Artifact Registry, ACR, or equivalent. |
| Image tag | Use Git SHA and record image digest. |
| Manifest model | Use Kustomize overlays, Helm values, or GitOps environment repo. |
| Dev deploy | Automatic after merge to `main`. |
| Staging deploy | Manual or release candidate promotion. |
| Production deploy | Manual approval with protected environment. |
| Rollback | Previous manifest/image digest or Kubernetes rollout undo. |

## CI/CD Risks

| Risk | Why it matters | Mitigation |
| --- | --- | --- |
| Pipeline deploys mutable tags | Cannot know what is running | Use Git SHA and digest. |
| No image scanning | Known vulnerabilities can ship | Add scanner gate. |
| Rebuild per environment | Artifacts differ between environments | Build once, promote. |
| No manifest validation | Bad YAML can fail at deploy time | Render and validate before deploy. |
| No smoke test | Broken deployment may appear healthy at pod level | Test real request path. |
| No production approval | Risky changes can deploy silently | Use protected environments. |
| No metadata | Hard to audit or roll back | Record deployment evidence. |
| No rollback job | Incident response is slower | Store previous known-good artifact and automate rollback. |

## Output Checklist

Before moving to pre-deployment validation, produce:

- [ ] CI/CD workflow diagram.
- [ ] Service test commands.
- [ ] Image build and push commands.
- [ ] Image scanning policy.
- [ ] Manifest render and validation commands.
- [ ] Dev deployment workflow.
- [ ] Staging and production promotion workflow.
- [ ] Smoke test commands.
- [ ] Deployment metadata format.
- [ ] Rollback workflow.
- [ ] Required secrets and IAM/RBAC list.

## Ready For Next Checklist

After CI/CD pipeline design is defined, move to:

```text
15. Pre-Deployment Validation
```

In that stage, validate the app, images, manifests, configuration, dependencies, security controls, and target environment before a real deployment.
