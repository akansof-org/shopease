# ShopEase Stage 7: actions guide

Recorded: 2026-09-21  
Purpose: Explain each action, what already exists, and what will count as done.

[Documentation home](../README.md) · [Onboarding research](onboarding-research.md) · [Cloud handover guide](../cloud-handover/README.md)

## Where the work belongs

We use two repositories:

- **`akansof-org/shopease`**: the fork, application code, Dockerfiles, tests, CI, and product documentation.
- **`akansof-org/platformone-gitops`**: deployment manifests under `apps/shopease/`, plus Argo CD Application and AppProject definitions.

GitHub stores the code and configuration. **Argo CD deploys the application into Kubernetes**, initially the local k3d cluster.

The team is complete based on your confirmation. Repository and file status below comes from the local checkout; live CI runs and cluster state have not been verified for this guide. Existing examples and research do not count as completed ShopEase deployment evidence.

## 1. Create the ShopEase team

**Status: Created — confirmed by you.**

The team identifies who owns ShopEase. Check that it has the intended access to the fork and that deployment changes have an agreed review owner in PlatformOne GitOps.

**Done when:** ownership and repository access are clear.

## 2. Create ShopEase repositories

**Status: Done for the agreed structure.**

The ShopEase fork and the existing PlatformOne GitOps repository provide the two homes we need. Local `origin` points to your fork and `upstream` points to Google's original project. No separate ShopEase docs, tests, or platform-config repository is needed.

**Done when:** application changes can be pushed to the fork and deployment changes have a home in PlatformOne GitOps. The application branch has already been pushed successfully.

## 3. Add upstream attribution

**Status: Complete — 2026-09-21.**

The [application README](../../README.md) now credits Google and the upstream contributors. [UPSTREAM.md](../../UPSTREAM.md) records the actual starting commit, its distinction from the release tag, Akansof's changes, the retained licence/notices, and the upstream update process.

“Upstream” means the original project ShopEase is based on: **Google's Online Boutique (`GoogleCloudPlatform/microservices-demo`)**. Attribution means clearly crediting that source and distinguishing your work from the original application.

The fork relationship, upstream README, and Apache 2.0 [LICENSE](../../LICENSE) provide provenance. The ShopEase README and root `UPSTREAM.md` now record:

- Original project name and URL.
- The actual upstream baseline tag and commit used.
- The retained licence and notices.
- What Akansof has added or changed, and how future upstream updates will be reviewed.

Example wording:

> ShopEase is based on Google's Online Boutique microservices demo. Akansof's work adapts it for local development and PlatformOne onboarding. See UPSTREAM.md for the source baseline and modification history.

Preserve upstream licence and copyright notices. `UPSTREAM.md` explains provenance; it does not replace those files. The research proposed `v0.10.6`, but record the actual baseline rather than claiming that proposal has already been applied.

**Done when:** a reader can identify the original authors, source version, and your contribution.

## 4. Create the product overview

**Status: Research exists; a dedicated ShopEase overview is pending.**

Write a short `docs/product-overview.md`: what ShopEase does, who the shopper is, who owns it, and what the demonstration covers. State that payments, shipping, and email are simulated. The inherited upstream purpose document is not this ShopEase overview.

**Done when:** someone unfamiliar with the project understands it in a few minutes.

## 5. Define the critical user journey

**Status: Proposed in the onboarding research; needs an agreed test definition.**

Document: **browse → view product → add to cart → review totals → checkout → order confirmation → empty cart**. Use synthetic shopper details. Specify expected items, quantities, totals, confirmation, and resulting cart state.

**Done when:** the same journey can be repeated with clear pass/fail checks, initially manually and then automatically.

## 6. Create a product architecture diagram

**Status: Upstream diagram and communication notes exist; ShopEase deployment view is pending.**

Use the [communication study](interservice-communication-flows.md) and [upstream diagram](../img/architecture-diagram.png) as references. Create a simple Mermaid diagram showing the shopper, ingress, frontend, service dependencies, and Redis. Mark the external entry point and where state lives. Show the delivery pipeline separately if useful.

**Done when:** the diagram explains the deployed design and agrees with its service connections.

## 7. Create the CI workflow

**Status: Upstream workflows exist; a verified ShopEase workflow is pending.**

Review inherited workflows before adapting them. Configure checks for pull requests and a trusted release path that tests, builds, scans, and publishes service images. Keep cluster deployment with Argo CD; CI should produce artifacts and release metadata.

**Done when:** a ShopEase change runs the intended checks and a failure blocks release publication or promotion as designed.

## 8. Build and scan images

**Status: Local build work exists; release image and scan evidence is pending.**

Build the application services for the target architecture, scan them, and publish accepted candidates to ECR. Define which scan findings block promotion and how exceptions are recorded. Use unique tags and record immutable digests; pin the external Redis image too.

**Done when:** source commit, build, scan report, and deployable image digest can be traced, and the cluster can pull the image.

## 9. Create the platform configuration

**Status: Folder scaffolding exists; deployable configuration is pending.**

Populate `platformone-gitops/apps/shopease/base/` and `overlays/local/` with the workload configuration: Deployments, Services, ingress, image digests, runtime settings, secret references, resources, probes, and storage. Keep credentials outside Git.

**Done when:** the local overlay renders successfully and describes the intended deployment without manual edits.

## 10. Deploy through GitOps

**Status: ShopEase registration and deployment are pending.**

Add an Argo CD AppProject and Application for ShopEase. Set the permitted repository and namespace, point the Application to the local overlay, and sync it to the cluster.

**Done when:** Argo CD reports `Synced / Healthy` and the shopper journey passes. A healthy deployment alone is not proof that checkout works.

## 11. Add health checks and resource settings

**Status: Upstream examples exist; local suitability is unverified.**

Review and adapt existing readiness/liveness probes, add startup probes where needed, and set CPU/memory requests and limits that fit the local cluster. Include these in the configuration before the first deployment, then tune from observations.

**Done when:** workloads start reliably, health checks detect the intended failures, and resource behavior has been observed under normal test load.

## 12. Add network policies

**Status: ShopEase-specific enforcement is pending.**

Use the service dependency map to allow necessary traffic: ingress to frontend, service calls, DNS, and required telemetry or external destinations. Apply default-deny policies with the needed allow rules in the chosen namespace.

**Done when:** required traffic works, an unauthorized connection is demonstrably blocked, and the shopper journey still passes.

## 13. Test a normal release

**Status: Pending.**

Make a small visible frontend change. Follow it through CI, the published image, a GitOps image-digest update, Argo CD reconciliation, and the shopper journey.

**Done when:** the running change is visible and its commit, image, configuration change, and test result are linked.

## 14. Test a bad release

**Status: Pending.**

In the local environment, introduce a controlled readiness failure in a new revision. Observe detection and whether the previous healthy version continues serving. Separately test a checkout dependency failure to confirm the journey test catches shopper-visible errors.

**Done when:** the failure, actual shopper impact, and recovery are recorded. A blocked rollout does not necessarily cause an outage.

## 15. Document rollback

**Status: PlatformOne has general guidance; a ShopEase runbook and proof are pending.**

Write the runbook before testing a bad release. Identify the last good configuration, revert the bad GitOps change, let Argo CD reconcile, and rerun the journey. Keep the previous image available. Explain that application rollback does not restore lost data.

**Done when:** the written steps have restored ShopEase successfully and include the verification result.

## 16. Record a first end-to-end demo

**Status: Pending.**

Record a short walkthrough: product purpose, successful purchase, normal release, controlled failure, rollback, and successful purchase afterward. Link the recording to the commits, CI results, and deployment evidence used.

**Done when:** another person can watch the full delivery-and-recovery story and trace its evidence.

## Suggested order from here

1. Add attribution, product overview, journey definition, and architecture.
2. Establish CI and build/scan/publish the images.
3. Create platform configuration, including health checks, resources, and network policies; deploy and verify it.
4. Write the rollback steps, test good and bad releases, and record actual recovery.
5. Record the demo.

Use the [cloud handover guide](../cloud-handover/README.md) when a step needs deeper investigation. This action guide is the short path through Stage 7, not a replacement for evidence that each action worked.
