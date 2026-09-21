# Cloud Handover Checklist: 13. Deployment Strategy

Scenario: the development team has handed this application to the platform engineering team. We are preparing it for cloud infrastructure, instrumentation, and deployment to Kubernetes or ECS.

This note covers the thirteenth checklist item only: **Deployment Strategy**.

## Objective

Define how changes move safely through environments.

This stage answers:

- Which environments exist?
- How does a change move from development to production?
- Which deployment strategy will be used?
- How do we roll back quickly?
- Which image tag or digest is safe to roll back to?
- How are database migrations handled?
- What must be verified after release?
- Who approves production deployments?

## Checklist

- [ ] Define environments: dev, staging, production.
- [ ] Define promotion process.
- [ ] Define deployment strategy: rolling, blue/green, canary.
- [x] Define rollback command/process.
- [ ] Define image tag used for rollback.
- [ ] Define database migration process.
- [ ] Define release verification steps.
- [ ] Define who approves production deploys.

Note:

- The repo contains deployment tooling, but it does not define your organization's environment promotion policy.
- `skaffold.yaml` uses Git commit based image tagging.
- Kubernetes Deployments default to rolling updates unless another strategy is configured.
- Production approval, rollback ownership, and promotion rules must be defined by the platform/product team.

## Current Repo Deployment Options

| Option | Location | Use case | Notes |
| --- | --- | --- | --- |
| Skaffold app deployment | `skaffold.yaml` | Build, tag, render, and deploy services from source | Recommended for owned dev/staging environments. |
| Skaffold Google Cloud Build profile | `skaffold.yaml` profile `gcb` | Build images remotely without local Docker | Useful when CI/CD should build in cloud. |
| Kustomize manifests | `kubernetes-manifests/` | Base Kubernetes app definitions | Intended to be rendered by Skaffold so images are replaced correctly. |
| Release manifests | `release/kubernetes-manifests.yaml` | Quick deployment with prebuilt public images | Good for demo validation, not ideal for owned production. |
| Helm release artifact | `release/onlineboutique-*.tgz` when generated | Chart-based deployment | Useful if your platform standard is Helm. |
| Docker Compose | `docker-compose.yml` | Local development/demo | Not a production cluster deployment strategy. |

Important repo detail:

```yaml
tagPolicy:
  gitCommit: {}
```

Skaffold tags images from Git commit state. For platform usage, record the final pushed image tag and digest for every deployed service.

## Environment Definition

At minimum, define these environments before production:

| Environment | Purpose | Typical access | Deployment source | Notes |
| --- | --- | --- | --- | --- |
| `dev` | Fast integration testing | Engineering/platform | Latest merged or feature branch build | Can tolerate more frequent changes. |
| `staging` | Production-like verification | Product/platform/QA | Release candidate image tags | Should mirror production config as closely as possible. |
| `production` | Customer/user traffic | Restricted | Approved immutable release tag or digest | Requires approval, rollback plan, monitoring, and support coverage. |

Decide whether environments are separated by:

- Separate clusters.
- Separate namespaces in one cluster.
- Separate cloud accounts/projects/subscriptions.
- Separate ECS clusters/services.
- Separate registries or registry paths.
- Separate Kustomize overlays, Helm values, or task definitions.

Recommended platform rule:

```text
Production should not deploy directly from a developer laptop.
Production should deploy from CI/CD or GitOps using reviewed, immutable artifacts.
```

## Promotion Process

Promotion means moving the same tested artifact through environments.

Recommended flow:

```text
code merge -> build image once -> tag image immutably -> deploy to dev
          -> promote same image tag/digest to staging
          -> approve release
          -> promote same image tag/digest to production
```

Avoid this:

```text
build separate image for dev
build separate image for staging
build separate image for production
```

Why:

- Separate builds can produce different artifacts.
- Rollbacks become harder to reason about.
- Debugging becomes confusing because the same Git commit may not mean the same container image.

For this app, a good promotion artifact set is:

| Artifact | Why it matters |
| --- | --- |
| Git SHA | Links the deployment to source code. |
| Image tag | Human-friendly deploy reference. |
| Image digest | Immutable proof of the exact image content. |
| Rendered manifest | Shows exactly what was applied to the cluster. |
| Config version | Shows which environment configuration was used. |
| Release notes | Explains what changed and what to verify. |

## Deployment Strategy Options

| Strategy | How it works | Pros | Risks |
| --- | --- | --- | --- |
| Rolling update | Gradually replaces old pods/tasks with new ones | Simple, default Kubernetes behavior, low operational overhead | Bad release can affect users while rollout is in progress. |
| Blue/green | Runs old and new versions separately, then switches traffic | Fast rollback, clear separation | Requires extra capacity and traffic switching mechanism. |
| Canary | Sends small percentage of traffic to new version first | Safer for risky changes | Requires ingress, service mesh, weighted routing, or progressive delivery tooling. |
| Recreate | Stops old version before starting new version | Simple for special cases | Causes downtime; avoid for user-facing services. |

Recommended starting point for this app:

- Use rolling updates in `dev` and `staging`.
- Use rolling updates in production only after readiness probes, resource requests, minimum replicas, and rollback commands are confirmed.
- Use canary for production if the platform has service mesh, Gateway API, Argo Rollouts, Flagger, ALB weighted routing, or equivalent tooling.
- Use blue/green if releases must be switched or rolled back at the traffic layer.

## Kubernetes Deployment Strategy

Kubernetes Deployments use rolling updates by default.

Example explicit rolling strategy:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

Platform interpretation:

- `maxUnavailable: 0` keeps existing capacity while new pods start.
- `maxSurge: 1` allows one extra pod above desired replicas during rollout.
- Readiness probes decide when new pods can receive traffic.
- Resource requests help the scheduler place new pods during rollout.

For production, avoid single replica deployments for critical user-facing services. A single replica can make rolling updates and node maintenance more fragile.

## Kubernetes Rollout Commands

Check rollout status:

```bash
kubectl rollout status deployment/<name>
```

View rollout history:

```bash
kubectl rollout history deployment/<name>
```

Roll back to the previous revision:

```bash
kubectl rollout undo deployment/<name>
```

Roll back to a specific revision:

```bash
kubectl rollout undo deployment/<name> --to-revision=<revision>
```

Check the image currently running:

```bash
kubectl get deployment <name> -o jsonpath='{.spec.template.spec.containers[*].image}'
```

Check rollout events:

```bash
kubectl describe deployment <name>
kubectl get events --sort-by=.lastTimestamp
```

## Skaffold Deployment Flow

Build and deploy to a cluster:

```bash
skaffold run --default-repo=<registry>/<repo>
```

Build remotely with Google Cloud Build:

```bash
skaffold run -p gcb --default-repo=<registry>/<repo>
```

Render manifests before applying:

```bash
skaffold render --default-repo=<registry>/<repo>
```

Recommended release habit:

```bash
skaffold render --default-repo=<registry>/<repo> > rendered-manifests.yaml
kubectl apply --dry-run=server -f rendered-manifests.yaml
kubectl apply -f rendered-manifests.yaml
```

This gives the platform team a reviewable artifact before anything changes in the cluster.

## Release Manifest Flow

The repo also provides prebuilt release manifests:

```bash
kubectl apply -f release/kubernetes-manifests.yaml
```

Use this for:

- Quick demo deployment.
- Validating cluster basics.
- Learning the app topology.

Avoid using this as the main owned production process unless:

- You trust the public prebuilt images.
- You have reviewed the image provenance.
- You accept the exact release manifest content.
- You have a rollback plan to a known previous release manifest.

For your own cloud deployment, prefer building and pushing your own images to your own registry.

## ECS Deployment Strategy

ECS has similar decisions, but the objects differ.

| Kubernetes concept | ECS equivalent |
| --- | --- |
| Deployment rollout | ECS service deployment |
| Pod template change | New task definition revision |
| Image tag/digest | Container image in task definition |
| Rollback | Revert service to previous task definition revision |
| Blue/green | CodeDeploy blue/green deployment |
| Canary/weighted traffic | ALB weighted target groups, CodeDeploy, App Mesh, or API Gateway |

ECS strategy checklist:

- Define whether ECS services use rolling update or CodeDeploy blue/green.
- Keep previous task definition revisions available.
- Record image tags and digests used in each task definition.
- Configure ALB target group health checks.
- Configure deployment circuit breaker if using ECS rolling deployments.
- Confirm rollback process before production.

Example ECS rollback idea:

```bash
aws ecs update-service \
  --cluster <cluster-name> \
  --service <service-name> \
  --task-definition <previous-task-definition-revision>
```

## Image Tag Used For Rollback

Rollback must target a known-good artifact.

Do not rely on:

```text
latest
```

Use one or more of:

| Tag type | Example | Notes |
| --- | --- | --- |
| Git SHA | `checkoutservice:9f3a2c1` | Good default for CI/CD. |
| Semantic version | `frontend:v1.2.3` | Good for product releases. |
| Build number | `cartservice:build-1042` | Useful when CI assigns build IDs. |
| Image digest | `frontend@sha256:...` | Strongest immutable reference. |

Recommended rollback record per service:

| Service | Previous known-good image | Current image | Deployment time | Owner |
| --- | --- | --- | --- | --- |
| `frontend` |  |  |  |  |
| `checkoutservice` |  |  |  |  |
| `productcatalogservice` |  |  |  |  |
| `cartservice` |  |  |  |  |
| `currencyservice` |  |  |  |  |
| `paymentservice` |  |  |  |  |
| `shippingservice` |  |  |  |  |
| `emailservice` |  |  |  |  |
| `recommendationservice` |  |  |  |  |
| `adservice` |  |  |  |  |
| `redis-cart` |  |  |  |  |

## Database Migration Process

The default demo deployment does not include application database migration jobs.

Current state:

- `redis-cart` stores cart state and has no schema migration.
- Default services mostly use in-memory/static/configured data.
- Optional cloud database components such as Spanner or AlloyDB require extra setup outside the default app path.
- The shopping assistant path can require additional infrastructure and configuration.

Platform rule:

```text
Schema changes must be deployed through an explicit migration process, not hidden inside ordinary app startup.
```

Migration checklist:

- [ ] Identify whether the release changes database schema.
- [ ] Identify whether migrations are backward compatible.
- [ ] Define whether migrations run before, during, or after app deployment.
- [ ] Define whether migrations are automatic jobs or manual approved steps.
- [ ] Define rollback behavior if app rollback meets migrated data.
- [ ] Back up data before destructive migrations.
- [ ] Test migration against staging data.

Recommended migration pattern:

```text
expand schema -> deploy app compatible with old and new schema -> migrate data -> contract old schema later
```

Avoid destructive schema changes in the same deployment that introduces application code depending on the new schema.

## Release Verification Steps

Pre-deployment verification:

- [ ] Confirm image builds completed successfully.
- [ ] Confirm images were pushed to the target registry.
- [ ] Confirm vulnerability scan results are acceptable.
- [ ] Confirm manifest rendering succeeds.
- [ ] Confirm server-side dry run succeeds.
- [ ] Confirm environment variables and secrets are present.
- [ ] Confirm target namespace, cluster, or ECS service is correct.
- [ ] Confirm rollback artifact is known.
- [ ] Confirm on-call owner is aware of production deployment.

Kubernetes commands:

```bash
skaffold render --default-repo=<registry>/<repo>
kubectl apply --dry-run=server -f rendered-manifests.yaml
kubectl diff -f rendered-manifests.yaml
```

Post-deployment verification:

- [ ] Confirm rollout completed.
- [ ] Confirm all pods/tasks are running.
- [ ] Confirm readiness checks are passing.
- [ ] Confirm public frontend is reachable.
- [ ] Confirm critical user flows work.
- [ ] Confirm logs show no startup errors.
- [ ] Confirm error rate did not increase.
- [ ] Confirm latency did not increase.
- [ ] Confirm restart count is stable.
- [ ] Confirm dashboards and alerts are healthy.

Kubernetes commands:

```bash
kubectl get deployments
kubectl get pods
kubectl get svc
kubectl rollout status deployment/frontend
kubectl logs deployment/frontend --tail=100
```

Smoke test examples:

```bash
curl -I http://<frontend-host>/
curl http://<frontend-host>/_healthz
```

For internal gRPC services, use `grpcurl` from a debug pod or an allowed network location.

## Production Approval

Define who can approve production deployments before the first real release.

Typical approvers:

| Area | Example owner | Approves |
| --- | --- | --- |
| Product | Product owner or engineering lead | User-facing release readiness. |
| Platform/SRE | Platform engineer or SRE on call | Operational readiness and rollback plan. |
| Security | Security owner | High-risk security or compliance changes. |
| Database | DBA/data owner | Schema migrations and data changes. |
| Incident response | On-call lead | Timing, freeze windows, and support coverage. |

Minimum production approval questions:

- Is this release tied to an approved change?
- Is the rollback path documented?
- Is the previous known-good image recorded?
- Are dashboards and alerts available?
- Is someone watching the rollout?
- Are migrations required?
- Is this inside an allowed deployment window?

## Recommended Deployment Strategy For This App

For a first cloud/platform deployment:

1. Use `dev` to validate image builds, service discovery, health checks, and basic traffic.
2. Use `staging` to validate production-like config, secrets, resource sizing, observability, and rollback.
3. Use rolling updates initially, because the app already uses Kubernetes Deployments.
4. Make `frontend`, `checkoutservice`, `cartservice`, and other critical services run with more than one replica before production.
5. Add readiness and liveness probes before relying on rolling updates.
6. Record image digests for every deployed service.
7. Promote the same image artifacts from staging to production.
8. Use canary or blue/green later if the platform supports weighted traffic or service mesh routing.

## Deployment Strategy Risks

| Risk | Why it matters | Mitigation |
| --- | --- | --- |
| Deploying from laptop to production | Hard to audit and reproduce | Use CI/CD or GitOps. |
| Rebuilding per environment | Same code can produce different artifacts | Build once, promote same artifact. |
| Using `latest` | Rollback target is ambiguous | Use Git SHA, release tag, and digest. |
| No readiness probes | Rolling update may send traffic too early | Add readiness checks for every traffic-serving service. |
| Single replica critical services | Rollout or node drain can cause interruption | Increase replicas and add PodDisruptionBudgets. |
| No migration plan | Rollback can break when schema changed | Use backward-compatible migrations. |
| No post-deploy smoke test | Failed deploy may be discovered by users | Automate smoke tests in pipeline. |
| No approval process | Production changes become informal | Define required approvers and evidence. |

## Output Checklist

Before moving to CI/CD pipeline design, produce:

- [ ] Environment list.
- [ ] Promotion flow diagram or written process.
- [ ] Deployment strategy decision.
- [ ] Rollback commands.
- [ ] Previous known-good image tracking method.
- [ ] Migration process.
- [ ] Release verification checklist.
- [ ] Production approval policy.

## Ready For Next Checklist

After deployment strategy is defined, move to:

```text
14. CI/CD Pipeline
```

In that stage, define how builds, tests, image pushes, manifest rendering, approvals, deployments, and rollback automation happen in the delivery pipeline.
