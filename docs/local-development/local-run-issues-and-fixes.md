# Local Run Issues And Fixes

Date: 2026-08-04

Context: this note documents the issues found while running the microservices locally without Docker Compose/Kubernetes, plus the working fixes used to get the full app running on a local machine.

## Goal

Run the full application locally, service by service, while understanding:

- Which services must start first.
- Which ports each service should use locally.
- Which environment variables are required.
- Which services are gRPC vs HTTP.
- Which failures were caused by local port conflicts, missing config, profiler settings, or stale process configuration.

## Final Working Local Service Map

| Service                      | Runtime              |           Local port | Start notes                                                                  |
| ---------------------------- | -------------------- | -------------------: | ---------------------------------------------------------------------------- |
| `productcatalogservice`    | Go                   |             `3550` | gRPC service. Start before recommendation, checkout, and frontend.           |
| `currencyservice`          | Node.js              |             `7001` | Moved from`7000` because local port `7000` was already occupied.         |
| `cartservice`              | .NET                 |             `5000` | Actual local listener was`5000`, not `7070`. Requires Redis.             |
| `redis`                    | Redis                |             `6379` | Required by cartservice.                                                     |
| `shippingservice`          | Go                   |            `50051` | gRPC service.                                                                |
| `paymentservice`           | Node.js              |            `50052` | Moved to avoid conflict with shippingservice on`50051`.                    |
| `emailservice`             | Python               |             `5005` | Moved from`5000` because `5000` was already occupied.                    |
| `recommendationservice`    | Python               |             `8082` | Depends on productcatalogservice.                                            |
| `adservice`                | Java/Gradle          |             `9555` | gRPC service.                                                                |
| `checkoutservice`          | Go                   |             `5050` | Depends on product, shipping, payment, email, currency, and cart.            |
| `frontend`                 | Go                   |             `8088` | HTTP service. Moved from`8081` because Redis Commander was using `8081`. |
| `shoppingassistantservice` | Python/HTTP optional | `8083` placeholder | Frontend required env var even when assistant was not actually running.      |

## Final Working Frontend Command

Run this from `src/frontend`:

```bash
PORT=8088 \
PRODUCT_CATALOG_SERVICE_ADDR=localhost:3550 \
CURRENCY_SERVICE_ADDR=localhost:7001 \
CART_SERVICE_ADDR=localhost:5000 \
RECOMMENDATION_SERVICE_ADDR=localhost:8082 \
SHIPPING_SERVICE_ADDR=localhost:50051 \
CHECKOUT_SERVICE_ADDR=localhost:5050 \
AD_SERVICE_ADDR=localhost:9555 \
SHOPPING_ASSISTANT_SERVICE_ADDR=localhost:8083 \
./frontend
```

Validate:

```bash
curl -i http://localhost:8088/_healthz
curl -i http://localhost:8088/
```

Open in browser:

```text
http://localhost:8088
```

## Final Working Checkout Command

Run this from `src/checkoutservice`:

```bash
PRODUCT_CATALOG_SERVICE_ADDR=localhost:3550 \
SHIPPING_SERVICE_ADDR=localhost:50051 \
PAYMENT_SERVICE_ADDR=localhost:50052 \
EMAIL_SERVICE_ADDR=localhost:5005 \
CURRENCY_SERVICE_ADDR=localhost:7001 \
CART_SERVICE_ADDR=localhost:5000 \
./checkoutservice
```

Important: restart checkoutservice whenever any dependency address changes. The service reads environment variables only at startup.

## Issue 1: `pip` Command Not Found

Service affected:

- `recommendationservice`
- `emailservice`

Error:

```text
zsh: command not found: pip
```

Root cause:

The machine did not have a standalone `pip` command available on `PATH`.

Fix:

Use Python module execution:

```bash
python3 -m pip install -r requirements.txt
```

Recommended isolated setup:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

## Issue 2: Incorrect Python Pip Command

Service affected:

- `emailservice`

Bad command:

```bash
python3 pip install -r requirements.txt
```

Error:

```text
can't open file '.../src/emailservice/pip': [Errno 2] No such file or directory
```

Root cause:

`python3 pip ...` tells Python to run a file named `pip`. The correct way is `python3 -m pip`.

Fix:

```bash
python3 -m pip install -r requirements.txt
```

## Issue 3: Recommendation Service Missing Product Catalog Address

Service affected:

- `recommendationservice`

Error:

```text
Exception: PRODUCT_CATALOG_SERVICE_ADDR environment variable not set
```

Root cause:

Recommendation service depends on Product Catalog service and requires its gRPC address at startup.

Fix:

Start `productcatalogservice` first, then start recommendation:

```bash
PORT=8082 \
PRODUCT_CATALOG_SERVICE_ADDR=localhost:3550 \
python recommendation_server.py
```

Dependency:

```text
recommendationservice -> productcatalogservice
```

## Issue 4: Ad Service Gradle Wrapper Permission Denied

Service affected:

- `adservice`

Error:

```text
zsh: permission denied: ./gradlew
```

Root cause:

The Gradle wrapper file existed but did not have executable permission.

Fix:

```bash
chmod +x gradlew
./gradlew installDist
```

Alternative without changing file mode:

```bash
bash gradlew installDist
```

## Issue 5: Ad Service Build Fails On Google Java Format

Service affected:

- `adservice`

Error:

```text
Execution failed for task ':verifyGoogleJavaFormat'.
class com.google.googlejavaformat.java.RemoveUnusedImports ... cannot access class com.sun.tools.javac.tree.JCTree$JCImport
```

Root cause:

The formatter task is incompatible with the current local JDK module access rules. This is a formatting verification problem, not necessarily an app runtime problem.

Fix for local run:

```bash
./gradlew installDist
PORT=9555 ./build/install/hipstershop/bin/AdService
```

If building locally and the formatter blocks the build:

```bash
./gradlew build -x verifyGoogleJavaFormat
```

## Issue 6: Checkout Service Missing Dependency Environment Variables

Service affected:

- `checkoutservice`

Error:

```text
panic: environment variable "SHIPPING_SERVICE_ADDR" not set
```

Root cause:

Checkout service is an orchestrator and requires all downstream service addresses at startup.

Required env vars:

```text
PRODUCT_CATALOG_SERVICE_ADDR
SHIPPING_SERVICE_ADDR
PAYMENT_SERVICE_ADDR
EMAIL_SERVICE_ADDR
CURRENCY_SERVICE_ADDR
CART_SERVICE_ADDR
```

Fix:

Start all dependencies first, then run checkout with the final working command shown above.

## Issue 7: Currency Service Could Not Bind To Port `7000`

Service affected:

- `currencyservice`

Error:

```text
E No address added out of total 1 resolved
Error: server must be bound in order to start
```

Root cause:

Port `7000` was already in use by `ControlCe`.

Evidence:

```bash
lsof -nP -iTCP:7000 -sTCP:LISTEN
```

Output showed:

```text
ControlCe ... TCP *:7000 (LISTEN)
```

Fix:

Run currency on another port:

```bash
DISABLE_PROFILER=1 PORT=7001 node server.js
```

Update dependents:

```text
CURRENCY_SERVICE_ADDR=localhost:7001
```

## Issue 8: Currency And Payment Services Failed Because Profiler Was Enabled

Services affected:

- `currencyservice`
- `paymentservice`

Error:

```text
Error: Project ID must be specified in the configuration
```

Root cause:

The Node.js services tried to start Google Cloud Profiler locally, but no GCP project ID was configured.

Fix:

Disable profiler locally.

Currency:

```bash
DISABLE_PROFILER=1 PORT=7001 node server.js
```

Payment:

```bash
DISABLE_PROFILER=1 DISABLE_TRACING=1 PORT=50052 node index.js
```

## Issue 9: Email Service Could Not Bind To Port `5000`

Service affected:

- `emailservice`

Error:

```text
RuntimeError: Failed to bind to address [::]:5000
```

Root cause:

Port `5000` was already in use by `cartservice` and `ControlCe`.

Evidence:

```bash
lsof -nP -iTCP:5000 -sTCP:LISTEN
```

Output showed:

```text
cartservi ... TCP 127.0.0.1:5000 (LISTEN)
ControlCe ... TCP *:5000 (LISTEN)
```

Fix:

Run email on another port:

```bash
DISABLE_PROFILER=1 PORT=5005 python email_server.py
```

Update checkout:

```text
EMAIL_SERVICE_ADDR=localhost:5005
```

## Issue 10: Cart Service Is A .NET App

Service affected:

- `cartservice`

Incorrect assumption:

```bash
REDIS_ADDR=localhost:6379 ./cartservice
```

Root cause:

Cart service is a .NET application. The direct binary may not exist unless it was published.

Fix for local development:

```bash
cd src/cartservice/src
REDIS_ADDR=localhost:6379 dotnet run
```

Published run option:

```bash
dotnet publish -c Release -o ./publish
REDIS_ADDR=localhost:6379 dotnet ./publish/cartservice.dll
```

Observed local listener:

```text
cartservice: localhost:5000
```

Therefore dependents must use:

```text
CART_SERVICE_ADDR=localhost:5000
```

## Issue 11: Frontend Missing Required Environment Variables

Service affected:

- `frontend`

Initial error:

```text
panic: environment variable "PRODUCT_CATALOG_SERVICE_ADDR" not set
```

Later error:

```text
panic: environment variable "SHOPPING_ASSISTANT_SERVICE_ADDR" not set
```

Root cause:

Frontend requires backend service addresses at startup. This local version also requires `SHOPPING_ASSISTANT_SERVICE_ADDR`.

Fix:

Run frontend with all required env vars:

```bash
PORT=8088 \
PRODUCT_CATALOG_SERVICE_ADDR=localhost:3550 \
CURRENCY_SERVICE_ADDR=localhost:7001 \
CART_SERVICE_ADDR=localhost:5000 \
RECOMMENDATION_SERVICE_ADDR=localhost:8082 \
SHIPPING_SERVICE_ADDR=localhost:50051 \
CHECKOUT_SERVICE_ADDR=localhost:5050 \
AD_SERVICE_ADDR=localhost:9555 \
SHOPPING_ASSISTANT_SERVICE_ADDR=localhost:8083 \
./frontend
```

Note:

`SHOPPING_ASSISTANT_SERVICE_ADDR=localhost:8083` was used as a placeholder because the assistant service was not required for the normal shopping flow.

## Issue 12: Frontend Port `8081` Was Actually Redis Commander

Service affected:

- `frontend`

Symptom:

```bash
curl -i http://localhost:8081/
```

Returned:

```html
<title>Redis Commander: Home</title>
```

Root cause:

Port `8081` was already being used by Redis Commander. The frontend was not actually running there.

Fix:

Run frontend on `8088`:

```bash
PORT=8088 ... ./frontend
```

Validate:

```bash
curl -i http://localhost:8088/_healthz
curl -i http://localhost:8088/
```

## Issue 13: Checkout Failed During Place Order Because It Had Stale Addresses

Service affected:

- `checkoutservice`

Symptom:

The frontend loaded correctly, but checkout failed when navigating to:

```text
http://localhost:8088/cart/checkout
```

Checkout logs showed stale configuration:

```text
cartSvcAddr:localhost:7070
currencySvcAddr:localhost:7000
emailSvcAddr:localhost:5000
paymentSvcAddr:localhost:50052
```

Root cause:

Checkout was started before the final local port map was corrected. It still had old dependency addresses in memory.

Fix:

Stop checkout and restart it with the final working dependency addresses:

```bash
PRODUCT_CATALOG_SERVICE_ADDR=localhost:3550 \
SHIPPING_SERVICE_ADDR=localhost:50051 \
PAYMENT_SERVICE_ADDR=localhost:50052 \
EMAIL_SERVICE_ADDR=localhost:5005 \
CURRENCY_SERVICE_ADDR=localhost:7001 \
CART_SERVICE_ADDR=localhost:5000 \
./checkoutservice
```

Lesson:

```text
Changing a service's port means every dependent service must be restarted with the new address.
```

## Recommended Local Startup Order

Start services in this order:

1. `redis`
2. `productcatalogservice`
3. `currencyservice`
4. `cartservice`
5. `shippingservice`
6. `paymentservice`
7. `emailservice`
8. `adservice`
9. `recommendationservice`
10. `checkoutservice`
11. `frontend`

Why:

- `recommendationservice` needs `productcatalogservice`.
- `cartservice` needs Redis.
- `checkoutservice` needs product, shipping, payment, email, currency, and cart.
- `frontend` needs almost all backend services.

## Useful Local Debug Commands

Check what is using a port:

```bash
lsof -nP -iTCP:<port> -sTCP:LISTEN
```

Check frontend:

```bash
curl -i http://localhost:8088/_healthz
curl -i http://localhost:8088/
```

Check gRPC health:

```bash
grpcurl -plaintext localhost:<port> grpc.health.v1.Health/Check
```

Check Product Catalog:

```bash
grpcurl -plaintext \
  -proto protos/demo.proto \
  -d '{}' \
  localhost:3550 \
  hipstershop.ProductCatalogService/ListProducts
```

Check Recommendation:

```bash
grpcurl -plaintext \
  -proto protos/demo.proto \
  -d '{"user_id":"test-user","product_ids":["OLJCESPC7Z"]}' \
  localhost:8082 \
  hipstershop.RecommendationService/ListRecommendations
```

Check Payment:

```bash
grpcurl -plaintext \
  -proto protos/demo.proto \
  -d '{
    "amount": {
      "currency_code": "USD",
      "units": 10,
      "nanos": 0
    },
    "credit_card": {
      "credit_card_number": "4111111111111111",
      "credit_card_cvv": 123,
      "credit_card_expiration_year": 2030,
      "credit_card_expiration_month": 12
    }
  }' \
  localhost:50052 \
  hipstershop.PaymentService/Charge
```

## Lessons Learned

- Local runs need a clear port map because Kubernetes service ports do not always translate cleanly to local host ports.
- Several services are gRPC, so Postman must use gRPC mode or `grpcurl`.
- Many startup failures are missing environment variables, not code bugs.
- Node services may need profiler disabled outside GCP.
- Port conflicts should be handled by moving local service ports instead of fighting background macOS processes.
- If a backend port changes, every dependent service must be restarted.
- The frontend health endpoint is `/_healthz`.
- A successful homepage load does not prove checkout works; checkout validates a deeper service path.
- The fastest troubleshooting loop is: check process listening port, check env vars, check service logs, test gRPC/HTTP directly, then retry the frontend flow.

## Final Result

The application was successfully run locally after:

- Assigning non-conflicting local ports.
- Disabling local profiler for services that expected GCP metadata.
- Starting services in dependency order.
- Supplying all required environment variables.
- Restarting checkout with corrected dependency addresses.
- Running frontend on an unused port.

Final browser entrypoint:

```text
http://localhost:8088
```
