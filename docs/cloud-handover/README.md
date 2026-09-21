# Cloud handover guide

[Documentation home](../README.md)

These 18 chapters expand the [platform deployment readiness checklist](readiness-checklist.md). They were written around a handover scenario: a development team supplies an application, and a platform engineer determines how to package, deploy, observe, secure, and recover it.

They are useful because they connect each checklist item to questions, repository observations, example commands, risks, and expected evidence. Use them as a study guide and assessment aid when onboarding ShopEase, then reuse the questions for other products.

## How to use this material

1. Read the [summary checklist](readiness-checklist.md) to understand the scope.
2. Consult the chapters that explain your current task; you do not need to complete all 18 in one sitting.
3. For each applicable check, record the actual result, evidence, unresolved gap, and next action. Mark non-applicable checks with a reason.
4. Use chapters 15 and 16 around a deployment, chapter 17 during diagnosis, and chapter 18 when deciding readiness.

The chapters contain Kubernetes and ECS alternatives and broader production-readiness topics. ShopEase's current delivery direction is local k3d through PlatformOne GitOps. Treat ECS sections as comparative reference. Configuration examples in these notes do not replace the authoritative manifests in `platformone-gitops/apps/shopease/`.

Existing repository observations and example versions describe the research context. Revalidate them against the current checkout before using them as evidence. The presence of a chapter does not mean that its controls have been implemented or tested.

## Chapter map

| Chapter | When to use it | What it helps establish |
| --- | --- | --- |
| [1. Application Inventory](cloud-handover-01-application-inventory.md) | Understand the application | Identify services, owners, dependencies, and the business journey. |
| [2. Containerization](cloud-handover-02-containerization.md) | Package each service | Check Dockerfiles, build contexts, runtime behavior, and container constraints. |
| [3. Image Registry And Tagging](cloud-handover-03-image-registry-and-tagging.md) | Identify release artifacts | Decide registry access, tags, digests, and image traceability. |
| [4. Runtime Configuration](cloud-handover-04-runtime-configuration.md) | Define runtime inputs | Inventory environment variables, secrets, feature flags, and environment differences. |
| [5. Service Discovery](cloud-handover-05-service-discovery.md) | Connect services | Check service names, ports, addressing, and dependency resolution. |
| [6. Kubernetes Or ECS Manifests](cloud-handover-06-kubernetes-or-ecs-manifests.md) | Describe deployment resources | Review Kubernetes and ECS examples and identify required workload configuration. |
| [7. Health Checks](cloud-handover-07-health-checks.md) | Detect unhealthy services | Assess readiness, liveness, startup behavior, and dependency failures. |
| [8. Storage And State](cloud-handover-08-storage-and-state.md) | Protect state | Identify mutable data, persistence, restart behavior, and recovery requirements. |
| [9. Networking And Exposure](cloud-handover-09-networking-and-exposure.md) | Control network exposure | Plan public entry points, private traffic, DNS, TLS, and network restrictions. |
| [10. Security](cloud-handover-10-security.md) | Review security controls | Assess identities, secrets, image risks, privileges, and policy controls. |
| [11. Observability](cloud-handover-11-observability.md) | Observe the application | Identify logging, metrics, traces, dashboards, and alerting gaps. |
| [12. Resource Management](cloud-handover-12-resource-management.md) | Size the workloads | Review requests, limits, replicas, capacity, and scaling evidence. |
| [13. Deployment Strategy](cloud-handover-13-deployment-strategy.md) | Release and recover | Choose rollout, promotion, and rollback behavior. |
| [14. CI/CD Pipeline](cloud-handover-14-cicd-pipeline.md) | Automate delivery | Map source changes through testing, image publication, deployment, and verification. |
| [15. Pre-Deployment Validation](cloud-handover-15-pre-deployment-validation.md) | Check before deployment | Validate builds, rendered configuration, permissions, and platform prerequisites. |
| [16. Post-Deployment Validation](cloud-handover-16-post-deployment-validation.md) | Check after deployment | Verify runtime behavior, connectivity, health, and the shopper journey. |
| [17. Troubleshooting Checklist](cloud-handover-17-troubleshooting-checklist.md) | Troubleshoot by layer | Investigate image pulls, startup, configuration, network paths, and resources. |
| [18. Final Go/No-Go Checklist](cloud-handover-18-final-go-no-go-checklist.md) | Make the readiness decision | Bring evidence, remaining gaps, and acceptance decisions together. |

## Suggested reading groups

- **Understand and package:** chapters 1–5.
- **Configure and operate:** chapters 6–12.
- **Deliver changes:** chapters 13–14.
- **Validate and recover:** chapters 15–18.

For the business journey and initial design choices, also read the [onboarding research](<../notes and research/onboarding-research.md>). For the actual request dependencies, read [interservice communication flows](<../notes and research/interservice-communication-flows.md>).
