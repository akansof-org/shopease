# Cloud Handover Checklist: 18. Final Go/No-Go Checklist

Scenario: the development team has handed this application to the platform engineering team. We are preparing it for cloud infrastructure, instrumentation, and deployment to Kubernetes or ECS.

This note covers the eighteenth checklist item only: **Final Go/No-Go Checklist**.

## Objective

Use this before declaring the application cluster-ready.

This stage answers:

- Is the application inventory complete?
- Are images buildable, pushable, pullable, and runnable?
- Is runtime configuration documented and environment-safe?
- Are secrets handled securely?
- Does service discovery work inside the target platform?
- Is public/private networking clear?
- Are health checks, resources, security, state, and observability ready?
- Is rollback documented?
- Do smoke tests pass?
- Can the operational owner debug failures?

## Final Checklist

- [ ] Every service is inventoried.
- [ ] Every service has a container image.
- [ ] Every image builds successfully.
- [ ] Every image is in an accessible registry.
- [ ] Runtime config is documented.
- [ ] Secrets are handled securely.
- [ ] Internal service discovery is configured.
- [ ] Public entrypoint is defined.
- [ ] Backends are private.
- [ ] Health checks are configured.
- [ ] Resource requests and limits are configured.
- [ ] Security context is configured.
- [ ] Stateful dependencies are understood.
- [ ] Observability is configured.
- [ ] Rollback process is documented.
- [ ] Smoke tests pass.
- [ ] Operational owner knows how to debug failures.

## Decision Rule

Use this rule:

```text
If a missing item can cause outage, data loss, security exposure, or unclear rollback,
the answer is No-Go until it is fixed or explicitly accepted as a risk.
```

For dev environments, some gaps may be acceptable.

For production, every critical item should be complete or formally risk-accepted.

## Go/No-Go Summary

| Area | Go condition | No-Go condition |
| --- | --- | --- |
| Inventory | All services, ports, protocols, owners, and dependencies are known | Unknown component, unknown owner, or unknown dependency |
| Containerization | Every required service image builds and starts | Image missing, build fails, or container cannot start |
| Registry | Images are pushed and pullable by cluster/tasks | Pull access untested or failing |
| Configuration | Runtime env vars/config are documented per environment | Hardcoded or unknown environment-specific values |
| Secrets | Secrets are stored outside Git and injected securely | Secrets in Git, local files, or manual undocumented injection |
| Service discovery | Services use cluster discovery names | Apps call `localhost` for other services |
| Networking | Public entrypoint and private backends are clear | Backends exposed unintentionally |
| Health checks | Readiness/liveness/target health checks are valid | Traffic can route to unhealthy workloads |
| State | Stateful dependencies and recovery plan are known | Data durability or migration behavior is unknown |
| Observability | Logs, metrics, traces, dashboards, and alerts are ready enough for operation | Team cannot see failures after deployment |
| Resources | Requests/limits and capacity are validated | Workloads cannot schedule or are likely to OOM |
| Security | Runtime security controls and IAM/RBAC are acceptable | Containers are overly privileged or access is too broad |
| Deployment | Rollout and rollback process is known | Rollback target or command is unknown |
| Validation | Smoke tests pass | User path cannot be proven healthy |
| Operations | Owner can debug failures | No one knows how to operate the app |

## Application-Specific Final Checks

For this microservices app, confirm:

- [ ] `frontend` is the only intended public traffic entrypoint.
- [ ] `frontend-external`, Ingress, Gateway, ALB, or API Gateway is configured intentionally.
- [ ] Backend gRPC services remain private.
- [ ] `frontend` uses `/_healthz` for HTTP health checks.
- [ ] Backend services use gRPC health checks or a platform-approved equivalent.
- [ ] `redis-cart` state expectations are understood.
- [ ] `loadgenerator` is not deployed to production unless explicitly intended.
- [ ] `kubernetes-manifests/` is rendered through Skaffold before apply, or release manifests/Helm/Kustomize output are used correctly.
- [ ] Image tags/digests for every deployed service are recorded.
- [ ] Rollback target is known before deployment.

## Production Readiness Evidence

Fill this table before approval:

| Evidence | Status | Link/path | Owner | Notes |
| --- | --- | --- | --- | --- |
| Application inventory |  |  |  |  |
| Image build results |  |  |  |  |
| Image scan results |  |  |  |  |
| Registry push/pull validation |  |  |  |  |
| Runtime config inventory |  |  |  |  |
| Secret handling review |  |  |  |  |
| Rendered manifests |  |  |  |  |
| Dry-run/diff results |  |  |  |  |
| Health check validation |  |  |  |  |
| Resource sizing validation |  |  |  |  |
| Security review |  |  |  |  |
| Observability validation |  |  |  |  |
| Smoke test results |  |  |  |  |
| Rollback plan |  |  |  |  |
| Troubleshooting owner/runbook |  |  |  |  |

## Final Decision Record

Use this before deployment:

| Field | Value |
| --- | --- |
| Application |  |
| Environment |  |
| Cluster/platform |  |
| Release version/Git SHA |  |
| Image registry |  |
| Manifest artifact |  |
| Deployment strategy |  |
| Rollback target |  |
| Risk level |  |
| Open risks accepted |  |
| Approver |  |
| Operational owner |  |
| Decision | Go / No-Go |
| Decision time |  |

## No-Go Triggers

Do not declare the app cluster-ready if any of these are true:

- [ ] A required service is not inventoried.
- [ ] A required image does not build.
- [ ] The cluster cannot pull required images.
- [ ] Required config or secrets are missing.
- [ ] A backend service is accidentally public.
- [ ] Readiness checks are missing or incorrect for traffic-serving services.
- [ ] Resource requests are missing for production workloads.
- [ ] Stateful dependency behavior is unknown.
- [ ] Logs are unavailable.
- [ ] There is no rollback target.
- [ ] Smoke tests fail.
- [ ] No operational owner is assigned.

## Go Criteria

Declare Go only when:

- [ ] The platform team can deploy the app repeatably.
- [ ] The cluster can schedule and run the workloads.
- [ ] The app can receive traffic through the intended entrypoint.
- [ ] Internal services can communicate through service discovery.
- [ ] Failures are visible through logs, events, metrics, and alerts.
- [ ] The app can be rolled back to a known-good version.
- [ ] The operational owner can troubleshoot the common failure modes.

## Notes For Future Apps

Use these 18 checklist sections as a reusable handover pattern:

1. Application Inventory
2. Containerization
3. Image Registry And Tagging
4. Runtime Configuration
5. Service Discovery
6. Kubernetes Or ECS Manifests
7. Health Checks
8. Storage And State
9. Networking And Exposure
10. Security
11. Observability
12. Resource Management
13. Deployment Strategy
14. CI/CD Pipeline
15. Pre-Deployment Validation
16. Post-Deployment Validation
17. Troubleshooting Checklist
18. Final Go/No-Go Checklist

This final checklist is not a replacement for the previous notes. It is the decision page that summarizes whether the previous work is complete enough to deploy safely.
