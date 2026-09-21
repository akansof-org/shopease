# ShopEase onboarding research

Date documented: 2026-09-21  
Status: Draft — awaiting reflection and decision review  
Scope: Research supporting Stage 7 onboarding

This document records the research discussed before the implementation plan. Recommendations are proposals, not accepted architecture decisions or completed implementation. After review, capture the agreed conclusions and personal reflections in the learning journal and carry accepted decisions into the relevant product documentation.

Source observations below come from the downloaded upstream checkout inspected during the discussion and the linked upstream documentation. Live release information reflects that research session; it has not been rechecked for this transcription.

## 1. What business journey will you demonstrate?

### Findings and sources

Online Boutique supports browsing products, managing a cart, and completing a simulated purchase. Payment, shipping, and email are mocked. The default frontend uses shopper sessions without requiring account registration.

Source: [Downloaded upstream README](../../README.md).

### Proposed answer

Demonstrate a shopper completing this journey:

> Open the storefront → view a product → add it to the cart → review quantity and price → enter synthetic checkout details → place the order → see an order confirmation → verify the cart is empty.

Success means:

- The selected product and quantity survive navigation.
- Prices, currency, and totals are consistent.
- Checkout returns an order ID and confirmation.
- Purchased items disappear from the cart.

The journey exercises frontend, catalog, currency, cart, Redis, checkout, payment, and shipping dependencies. Recommendations and advertising support the experience. Describe the outcome as a simulated purchase, without implying real payment processing or fulfilment.

### Tradeoffs

This is a useful end-to-end business demonstration, but it does not prove production commerce capabilities such as durable orders, inventory management, refunds, or real transaction settlement.

### Status and validation

**Proposed.** Agree the exact assertions and test data, then verify the journey manually and through automated tests.

## 2. Which upstream version will you use?

### Findings and sources

The upstream release page identified `v0.10.6` as the latest release at the time of research.

- Proposed release: `v0.10.6`
- Release commit: `5b3a712ab85ccb8f6f7cd5b720d36ba9a8d041eb`
- Inspected local checkout: `9a4616e77f0f9cbcbecaf27d711c38890dda1404`

The local checkout's commit message references the release, but it is not the release tag's commit. A comparison of committed files under `src/` and `kubernetes-manifests/` showed no differences between these two commits. This comparison does not establish that every repository file is identical or that a build has been validated.

Source: [Online Boutique v0.10.6 release](https://github.com/GoogleCloudPlatform/microservices-demo/releases/tag/v0.10.6).

### Proposed answer

Start from `v0.10.6` pinned to its full release commit. Record upstream identity separately from ShopEase's own release identity, for example `shopease-v0.1.0`.

### Tradeoffs

A pinned release provides a reproducible baseline and reviewable upgrades. It also creates an ongoing responsibility to review upstream changes and dependency vulnerabilities. A release label does not prove that an image is vulnerability-free.

### Status and validation

**Proposed; baseline needs build and scan validation.** Record the selected commit in `UPSTREAM.md` and verify the build before declaring the baseline accepted.

## 3. Will you fork, mirror, or consume upstream directly?

### Findings and sources

| Approach | Fit for ShopEase |
| --- | --- |
| Fork | Supports application changes, custom CI, release history, and visible upstream attribution. |
| Mirror | Useful for maintaining a copy; ongoing mirroring is awkward when maintaining independent application changes. |
| Consume upstream directly | Appropriate for deploying an unchanged demo, but provides less scope to demonstrate ownership of application builds and releases. |

Source: [GitHub repository duplication and mirroring](https://docs.github.com/en/repositories/creating-and-managing-repositories/duplicating-a-repository).

### Proposed answer

Create a GitHub fork named `akansof-org/shopease-app`, establishing the application baseline at the chosen release commit.

Preserve upstream history, licence, and copyright notices. Add `UPSTREAM.md` identifying the source, baseline, modifications, and upgrade policy. Review inherited workflows before enabling them because upstream automation includes Google-specific release infrastructure.

### Tradeoffs

A fork makes provenance clear and supports customization, but requires deliberate upstream integration and maintenance of the ShopEase changes.

### Status and validation

**Proposed.** Confirm repository naming and ownership before creation. No repository creation is recorded as complete here.

## 4. How will application images be built and tagged?

### Findings and sources

PlatformOne's existing registry strategy uses local image import for development and Amazon ECR for release and GitOps evidence. Upstream provides service Dockerfiles and build contexts in `skaffold.yaml`.

Sources:

- [PlatformOne local registry strategy](../../../../02-platformone/platformone-local-lab/registry/local-registry-strategy.md)
- [Upstream build configuration](../../skaffold.yaml)
- [ECR tag immutability](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-tag-mutability.html)
- [ECR authentication](https://docs.aws.amazon.com/AmazonECR/latest/userguide/registry_auth.html)

### Proposed answer

Use GitHub Actions to build one image per application service from the fork's checked-out commit. Run relevant tests, scan the resulting images, publish candidates to ECR, and deploy the exact scanned artifacts by digest.

Example naming:

```text
Repository: shopease/frontend
Tag:        sha-<commit>-run-<build-id>
Deployment: <registry>/shopease/frontend@sha256:<digest>
```

Enable ECR tag immutability. Include a build identifier to distinguish rebuilds of the same source revision. Record the source commit, build run, scan results, and service digests together as release evidence.

Initially build all application services for each release; optimize changed-service builds later while accounting for shared protobuf and dependency changes. Promote the same digest between environments. Use a pinned external Redis image rather than treating Redis as a custom application build.

### Tradeoffs

Building every service is simpler to reason about but consumes more CI time. Selective builds are faster but require correct dependency tracking. Multi-architecture builds add time and complexity, so first verify the cluster architecture.

ECR authentication is also an operational dependency: authorization tokens last 12 hours, so local k3d pull credentials require renewal. CI authentication and cluster image-pull authentication are separate concerns.

### Status and validation

**Proposed.** Confirm target architecture, build contexts, scan gate and exception policy, registry access, and credential renewal. Prove the cluster can pull the published digest.

## 5. Which services require independent scaling?

### Findings and sources

The services handle different types of demand. The following are scaling candidates, not measured capacity requirements.

| Service group | Why demand can differ |
| --- | --- |
| Frontend | Handles shopper HTTP traffic and aggregates downstream requests. |
| Catalog and currency | Browsing produces repeated reads and conversions. |
| Cart service | Handles cart reads and changes separately from checkout volume. |
| Checkout, payment, shipping | Demand follows purchase activity. |
| Recommendations and advertising | Have distinct latency and resource characteristics. |
| Email | Demand follows completed orders. |
| Load generator | Produces test demand and must be controlled independently. |

Source: [Upstream service descriptions](../../README.md).

### Proposed answer

Keep service deployments independently configurable. Start with one replica per service for the local baseline and measure request rate, latency, CPU, memory, and errors before introducing autoscaling.

Treat Redis separately: adding replicas to its Deployment does not establish Redis replication, consistent shared state, or failover.

### Tradeoffs

One replica reduces laptop resource consumption but does not provide service redundancy. Adding replicas without observing bottlenecks can waste capacity and leave the actual limiting dependency unchanged.

### Status and validation

**Needs measurement.** Run controlled load tests before assigning scaling thresholds or claiming that any service requires autoscaling.

## 6. Which services require persistent state?

### Findings and sources

Redis is the main mutable state dependency in the default application. The upstream cart manifest uses an `emptyDir` volume for Redis; replacing that Pod can therefore lose carts.

Other relevant distinctions:

- Cart service stores state in Redis; its own Pods do not require persistent volumes.
- The default catalog is supplied through a JSON file.
- The application does not provide a durable order database or payment ledger.
- Payment, shipping, and email are simulated.

Sources:

- [Cart and Redis manifest](../../kubernetes-manifests/cartservice.yaml)
- [Catalog data](../../src/productcatalogservice/products.json)
- [Checkout implementation](../../src/checkoutservice/main.go)

### Proposed answer

Adopt this initial requirement:

> A shopper's cart should survive replacement of the Redis Pod.

Use a persistent volume and explicitly configured Redis persistence, then test recovery. A PVC alone does not prove durability.

### Tradeoffs

Persistence adds storage configuration and recovery responsibilities. Local persistent storage can demonstrate Pod replacement recovery, but does not establish survival of laptop loss or cluster deletion. Persistence also does not create a durable order history that the application does not implement.

### Status and validation

**Proposed.** Choose persistence settings and storage behavior, document the intended data-loss tolerance, and verify that the same shopper session retains its cart after Redis Pod replacement.

## 7. Which configuration differs by environment?

### Findings and sources

Environment differences affect deployment behavior and integrations. The same application artifact should be reusable across environments.

Sources:

- [Upstream frontend configuration](../../kubernetes-manifests/frontend.yaml)
- [PlatformOne GitOps repository structure](../../../../02-platformone/platformone-gitops/docs/repository-structure.md)

### Proposed answer

| Configuration | Examples |
| --- | --- |
| Routing | Hostname, ingress class, TLS |
| Capacity | Replicas, requests, limits, autoscaling |
| Storage | Storage class, volume size, Redis endpoint |
| Access | Secret references, registry credentials, service accounts |
| Connectivity | Network policies and external dependency endpoints |
| Observability | Telemetry endpoints, sampling, log verbosity |
| Test behavior | Load generator enabled, user count, test duration |
| Release selection | Approved image digests |
| Presentation | Environment banner and optional feature flags |

Keep credentials outside Git. Keep application code consistent and promote the same image digest. Internal service names can remain consistent when environments use separate namespaces.

### Tradeoffs

Overlays make differences explicit but can drift if too much configuration is duplicated. Define shared defaults and introduce environment overrides only where needed.

### Status and validation

**Proposed.** Implement the local environment first. Define other environment values when their requirements are known rather than inventing production settings now.

## 8. What is the initial service tier?

### Findings and sources

The inspected PlatformOne glossary defines a service tier as a classification of importance and expected operational controls. The research did not find an established numbered tier policy.

Source: [PlatformOne glossary](../../../../02-platformone/platformone-docs/docs/planning/glossary.md).

### Proposed answer

Use this descriptive classification initially:

> Non-production demonstration service, supported during scheduled lab and demo sessions.

Provide a named owner, health checks, monitoring, a tested recovery procedure, and a defined demo availability window. This classification does not imply 24/7 support.

Checkout remains the highest-priority business journey within the service. Operational tier and journey criticality are separate decisions.

### Tradeoffs

A descriptive tier is honest about current operating commitments, but must eventually map to a shared platform classification if a numbered policy is introduced. Assigning a number now would invent a standard.

### Status and validation

**Proposed.** Confirm ownership, support expectations, demo windows, and the mapping to any subsequently adopted tier policy.

## 9. What failures matter to a shopper?

### Findings and sources

| Shopper experience | Likely dependency or failure |
| --- | --- |
| Store will not open | Routing, frontend, or a required downstream dependency |
| Products or prices fail to load | Catalog or currency service |
| Cart cannot be updated or disappears | Cart service, Redis availability, or lost state |
| Checkout fails or hangs | Checkout, payment, shipping, or network connectivity |
| Confirmation is ambiguous | Failure after a checkout step has already succeeded |
| Purchase succeeds but cart remains populated | Cart-clearing failure |
| Recommendations or adverts disappear | Supporting service failure |
| Confirmation email is unavailable | Email service failure |

The inspected checkout code calls payment before shipping. A subsequent shipping error can therefore produce an ambiguous outcome. These operations are mocked, but their ordering exposes a transaction-design limitation worth documenting.

Checkout ignores the returned cart-clearing error and logs email errors without failing the completed order. The frontend also has explicit error-tolerant handling for recommendation failures. Behavior should be verified on each tested route rather than assuming all supporting-service failures are harmless.

Sources:

- [Checkout implementation](../../src/checkoutservice/main.go)
- [Frontend handlers](../../src/frontend/handlers.go)

### Proposed answer

Prioritize purchase completion, correct totals, retained cart contents before purchase, and correct cart state afterward. Treat missing supporting content separately from inability to shop.

Use journey assertions alongside infrastructure health checks. Healthy Pods or a successful HTTP response alone do not prove a successful purchase.

For failure research, distinguish:

- A rollout failure that leaves the previous healthy version serving shoppers.
- A dependency failure that causes a shopper-visible business failure.
- A state-loss event that changes cart contents.
- A supporting-service failure that degrades the experience without blocking purchase.

### Tradeoffs

Infrastructure checks are inexpensive and useful but cannot establish business correctness. End-to-end tests provide stronger evidence, but require controlled data, repeatable sessions, and explicit expected outcomes.

### Status and validation

**Proposed priorities; failure behavior needs runtime validation.** Record the observed shopper impact, detection method, recovery steps, and post-recovery journey result for each controlled experiment.

## Review before accepting the decisions

- Confirm the business journey and its assertions.
- Accept or revise the upstream baseline and fork strategy.
- Agree image naming, scan gates, and registry authentication responsibilities.
- Validate scaling assumptions through measurement.
- Accept the cart persistence requirement and its recovery limits.
- Agree the environment configuration boundaries and service classification.
- Select the first controlled failure scenarios.

After reflection, record the agreed conclusions in the learning journal and update product documentation and decision records. This draft does not mark the research proposals or implementation work as completed.

## Repository layout decision — accepted 2026-09-21

Use one ShopEase repository, `shopease`, for application source, Dockerfiles, CI, tests, and documentation. ShopEase research lives under `docs/notes and research/`, alongside preserved upstream documentation.

Deployment desired state belongs in the existing `platformone-gitops/apps/shopease/` directory, with common manifests in `base/` and local overrides in `overlays/local/`. Argo CD registration belongs in that repository’s `applications/` and `appprojects/` directories. Separate ShopEase docs, tests, integrations, and platform-configuration repositories are not required.

The existing downloaded checkout was relocated intact into `04-products/shopease/`, preserving its Git history and local changes. Its upstream remote and current commit were retained; creation of the GitHub fork and selection of the proposed release baseline remain pending. Existing upstream deployment examples remain reference material; PlatformOne GitOps will own the deployed ShopEase configuration.

### Documentation consolidation

The onboarding research was moved to `docs/notes and research/onboarding-research.md`. The redundant `docs/shopease/` wrapper and its README were removed; the repository ownership and layout decisions are retained here.
