# Critical user journey: complete a simulated purchase

Journey ID: `SE-CUJ-001`

Owner: ShopEase team

Status: Defined from the current source; runtime execution and automation are pending.

[Product overview](product-overview.md) · [Stage 7 actions](<notes and research/onboarding-actions-guide.md>)

## Shopper goal

A shopper finds a product, buys the intended quantity at the displayed total, receives an order confirmation, and returns to an empty cart.

> Browse → view product → add to cart → review totals → checkout → confirm order → check cart.

This is the primary acceptance check for the first deployment, normal releases, and recovery after rollback. Healthy Pods alone do not establish that it works.

## Before starting

- Record the environment URL and application/configuration revisions being tested.
- Use a fresh browser session with cookies enabled and an empty cart. Keep that same session throughout the journey.
- Select **USD** using the currency selector. Keep currency fixed for this baseline test.
- Use the default catalog and shipping implementation described below. If a release intentionally changes either, review and update the expected values first.
- Run against the simulated ShopEase application using the synthetic details below.

## Repeatable test data

| Field | Value |
| --- | --- |
| Product | Sunglasses, SKU `OLJCESPC7Z` |
| Quantity | 2 |
| Unit price | USD 19.99 |
| Item subtotal | USD 39.98 |
| Shipping | USD 8.99 for the current non-empty-cart shipping implementation |
| Expected total | **USD 48.97** |
| Email | `shopper@example.com` |
| Street / city / state | `123 Demo Street` / `Mountain View` / `CA` |
| ZIP / country | `94043` / `United States` |
| Mock card number | `4111111111111111` |
| Expiration | December of the year after the test is run; choose that year in the form |
| CVV | `123` |

Payments, shipping, and email are mocked. No real charge, shipment, or inbox delivery is expected. The card value is a synthetic test number accepted by the mock's format checks.

## Steps and pass criteria

| Step | Shopper action | Required result |
| --- | --- | --- |
| 1. Browse | Open the storefront and select USD. | Catalog renders; Sunglasses is available at USD 19.99. |
| 2. View | Open Sunglasses. | Product name, SKU, price, and quantity selector are present. |
| 3. Add | Select quantity 2 and add to cart once. | Cart contains Sunglasses with quantity 2 and an item total of USD 39.98. |
| 4. Retain | Navigate back to the storefront, then reopen the cart. | The same item and quantity remain; shipping is USD 8.99 and total is USD 48.97. |
| 5. Checkout | Fill the synthetic details and submit the order once. | Checkout finishes without a validation or server error and shows the order confirmation page. |
| 6. Confirm | Read the confirmation. | “Your order is complete!”, a non-empty confirmation/order ID, a non-empty tracking ID, and Total Paid of USD 48.97 are displayed. |
| 7. Verify cart | Use Continue Shopping and open the cart again in the same session. | “Your shopping cart is empty!” is displayed; Sunglasses is no longer in the cart. |

The order page does not currently list purchased items, so verify item identity and quantity in the cart before submitting. Its email-success message does not establish that an email was delivered.

## What counts as a failure

Fail the journey if any required result is missing: pages do not load, cart state changes unexpectedly, quantities or totals are wrong, checkout errors, confirmation is incomplete, or the cart remains populated after success.

For repeatable execution, use an initial **30-second timeout per navigation or checkout action**. A timeout fails that run; capture it rather than automatically retrying checkout. This is a test timeout, not a production latency SLO. Treat a run that cannot start because its environment is unavailable as blocked, never passed.

Missing recommendations or adverts should be recorded as degradation; they do not fail this purchase journey if every required result still passes. Do not automatically resubmit an ambiguous checkout: the current flow performs payment before shipping, and transaction-wide retry safety is not established.

Cart recovery after Redis replacement, multiple currencies, invalid input, concurrent shoppers, and load testing are separate tests. This journey does not prove durable orders, exactly-once payment, or cart persistence across infrastructure failures.

## Evidence to record

Use this short record for each manual or automated run:

```text
Journey: SE-CUJ-001
Date/time and tester:
Environment URL:
Application commit / image digest(s):
GitOps configuration commit (or local Compose configuration revision):
Reason: first deployment / normal release / failure test / rollback
Result: PASS / FAIL / BLOCKED
Failed step and observed behavior, if any:
Order ID and total paid, if available:
Cart state after checkout:
Screenshots or test-report links:
Related logs/events and recovery notes:
```

Record PASS only when every required assertion passes. Capture at least the cart with totals, confirmation page, and empty cart. No execution result is claimed by this document.

## Automation and source references

The future journey test should use one browser context and assert the same results. The application routes are `GET /`, `GET /product/OLJCESPC7Z`, `POST /cart`, `GET /cart`, and `POST /cart/checkout`; prepend any configured base path. HTTP-only tests must retain cookies and inspect response content, not just status codes.

The existing load generator produces varied traffic; it does not assert all of this journey's outcomes and is not a substitute for this acceptance test.

- [Catalog fixture](../src/productcatalogservice/products.json)
- [Shipping quote](../src/shippingservice/quote.go)
- [Frontend routes](../src/frontend/main.go) and [handlers](../src/frontend/handlers.go)
- [Cart template](../src/frontend/templates/cart.html) and [confirmation template](../src/frontend/templates/order.html)
- [Checkout orchestration](../src/checkoutservice/main.go)
- [Existing load generator](../src/loadgenerator/locustfile.py)
