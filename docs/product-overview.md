# ShopEase product overview

ShopEase is Akansof's demonstration online store, based on Google's Online Boutique. A shopper can browse products, add items to a cart, and complete a simulated purchase. It is the first flagship application used to demonstrate how PlatformOne supports application delivery and operation.

## Who it serves

- **Demo shoppers** use the storefront to explore products and complete a purchase with synthetic details.
- **The ShopEase team** maintains the application and uses it to practice building, testing, and releasing changes.
- **Platform engineers and reviewers** use the project to understand and assess deployment, failure detection, and recovery on PlatformOne.

## What the shopper can do

The central journey is:

> Browse the catalog → view a product → add it to the cart → review quantities and totals → check out → see an order confirmation → verify the cart is empty.

The application also provides currency conversion, recommendations, advertising, and shipping estimates. Shoppers use sessions without creating an account. Payment, shipping, and email are simulated; the demo does not charge real cards, dispatch goods, or deliver real confirmation emails.

The [critical user journey](critical-user-journey.md) defines repeatable test data, steps, and pass/fail checks. Its runtime execution is pending; the definition is not completed test evidence.

## Why Akansof is onboarding it

ShopEase gives PlatformOne a realistic application with multiple services and dependencies. The onboarding goal is to show that an application change can be built, tested, scanned, deployed through GitOps, and verified from the shopper's perspective.

The first end-to-end demonstration should show:

- A successful simulated purchase.
- A normal release traced from source commit to running image.
- A controlled bad release and its observed shopper impact.
- Recovery through a Git-based rollback, followed by a successful purchase.

## Ownership and repository responsibilities

| Area | Responsible team | Home |
| --- | --- | --- |
| Application behavior, source, Dockerfiles, tests, CI, and product docs | ShopEase | `akansof-org/shopease` |
| Deployment integration and platform configuration | PlatformOne, with ShopEase input | `akansof-org/platformone-gitops`, under `apps/shopease/` |
| Argo CD registration and deployment permissions | PlatformOne | GitOps repository's `applications/` and `appprojects/` directories |

GitHub stores the source and deployment configuration. Argo CD will reconcile that configuration into Kubernetes. The initial target is the local k3d environment; published release images are intended to come from ECR.

## Initial scope and limits

The initial scope is a non-production learning and demonstration workload. It does not yet carry a production availability or support commitment. The default application has Redis-backed carts and a file-based catalog, but does not provide a durable order database or payment ledger.

Real payments, fulfilment, production customer data, and a production commerce launch are outside this onboarding scope. Cart persistence and recovery behavior still need implementation decisions and runtime verification.

## Current state

The ShopEase team and fork exist. Local development changes, research, and upstream attribution are documented. PlatformOne GitOps has a ShopEase folder scaffold. ShopEase CI, release images, deployment, operational controls, and release/recovery evidence remain to be completed or verified.

Use the [Stage 7 actions guide](<notes and research/onboarding-actions-guide.md>) for the current checklist rather than treating this overview as proof of deployment readiness.

## Source and supporting documentation

The original Online Boutique application was developed by Google and its upstream contributors. Akansof's adaptations are documented in [UPSTREAM.md](../UPSTREAM.md), alongside the retained [Apache 2.0 licence](../LICENSE).

- [Local development](local-development/README.md)
- [Onboarding research](<notes and research/onboarding-research.md>)
- [Service communication flows](<notes and research/interservice-communication-flows.md>)
- [Cloud handover guide](cloud-handover/README.md)
