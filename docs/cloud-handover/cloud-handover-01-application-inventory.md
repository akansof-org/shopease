# Cloud Handover Checklist: 1. Application Inventory

Scenario: the development team has handed this application to the platform engineering team. We are preparing it for cloud infrastructure, instrumentation, and deployment to Kubernetes or ECS.

This note covers the first checklist item only: **Application Inventory**.

## Objective

Before containerizing, instrumenting, or deploying anything, identify what the application is made of:

- services
- runtimes
- entrypoints
- startup commands
- ports
- protocols
- internal dependencies
- external dependencies
- public/private exposure
- background jobs or optional workers

This prevents common deployment mistakes such as exposing the wrong service, misconfiguring service discovery, missing a cache/database, or using `localhost` where cluster DNS is required.

## Checklist

- [X] List every service/application component.
- [X] Identify the language/runtime for each service.
- [X] Identify the application entrypoint.
- [X] Identify the startup command.
- [X] Identify exposed ports.
- [X] Identify protocol per port: HTTP, gRPC, TCP, UDP.
- [X] Identify internal service dependencies.
- [X] Identify external dependencies: database, cache, queue, object storage, third-party API.
- [X] Identify background workers, cron jobs, or one-off migration jobs.
- [X] Identify which service receives public traffic.
- [X] Identify which services must stay private.

## High-Level Application Summary

This application is a microservices-based e-commerce demo.

The public user-facing component is:

- `frontend`

Most backend services communicate using gRPC and should remain private inside the cluster.

The main user request flow is:

```text
User Browser
  -> frontend
  -> productcatalogservice
  -> currencyservice
  -> cartservice
  -> recommendationservice
  -> checkoutservice
  -> adservice
```

The checkout flow fans out to several backend services:

```text
checkoutservice
  -> productcatalogservice
  -> cartservice
  -> currencyservice
  -> shippingservice
  -> paymentservice
  -> emailservice
```

Cart state is stored in Redis:

```text
cartservice
  -> redis-cart
```

## Application Components

| Component                    | Type                          | Runtime       | Entrypoint                                                   | Startup command                                                |                                               Port | Protocol    | Public?                        |
| ---------------------------- | ----------------------------- | ------------- | ------------------------------------------------------------ | -------------------------------------------------------------- | -------------------------------------------------: | ----------- | ------------------------------ |
| `frontend`                 | Web application               | Go            | `src/frontend/main.go`                                     | `/src/server` in container, or `go run .` locally          | `8080` container, exposed as service port `80` | HTTP        | Yes                            |
| `productcatalogservice`    | Backend service               | Go            | `src/productcatalogservice/server.go`                      | `/src/server` in container, or `go run .` locally          |                                           `3550` | gRPC        | No                             |
| `currencyservice`          | Backend service               | Node.js       | `src/currencyservice/server.js`                            | `node server.js`                                             |                                           `7000` | gRPC        | No                             |
| `cartservice`              | Backend service               | .NET/C#       | `src/cartservice/src/Program.cs`                           | `/app/cartservice` in container, or `dotnet run` locally   |                                           `7070` | gRPC        | No                             |
| `shippingservice`          | Backend service               | Go            | `src/shippingservice/main.go`                              | `/src/shippingservice` in container, or `go run .` locally |                                          `50051` | gRPC        | No                             |
| `paymentservice`           | Backend service               | Node.js       | `src/paymentservice/index.js`                              | `node index.js`                                              |                                          `50051` | gRPC        | No                             |
| `emailservice`             | Backend service               | Python        | `src/emailservice/email_server.py`                         | `python email_server.py`                                     | `8080` in Kubernetes, `5000` in Docker Compose | gRPC        | No                             |
| `recommendationservice`    | Backend service               | Python        | `src/recommendationservice/recommendation_server.py`       | `python recommendation_server.py`                            |                                           `8080` | gRPC        | No                             |
| `adservice`                | Backend service               | Java          | `src/adservice/src/main/java/hipstershop/AdService.java`   | `/app/build/install/hipstershop/bin/AdService`               |                                           `9555` | gRPC        | No                             |
| `redis-cart`               | Cache/state store             | Redis         | Redis image entrypoint                                       | default Redis startup                                          |                                           `6379` | TCP         | No                             |
| `loadgenerator`            | Background traffic generator  | Python/Locust | `src/loadgenerator/locustfile.py`                          | `locust --host=http://${FRONTEND_ADDR} --headless ...`       |                                       none exposed | HTTP client | No                             |
| `shoppingassistantservice` | Optional AI assistant service | Python/Flask  | `src/shoppingassistantservice/shoppingassistantservice.py` | `python shoppingassistantservice.py`                         |                                           `8080` | HTTP        | No, called through`frontend` |

## Core Services

These are required for the normal storefront to work:

- `frontend`
- `productcatalogservice`
- `currencyservice`
- `cartservice`
- `redis-cart`
- `shippingservice`
- `paymentservice`
- `emailservice`
- `checkoutservice`
- `recommendationservice`
- `adservice`

## Optional Or Non-Core Components

### `loadgenerator`

Purpose:

- Generates synthetic traffic against the frontend.
- Useful for demos, load testing, observability validation, and autoscaling tests.

Deployment status:

- Present in the repo.
- Not required for serving users.
- In `docker-compose.yml`, it is behind the `loadtest` profile.
- In `kubernetes-manifests/kustomization.yaml`, it is commented out for normal deployment.

Platform note:

- Do not deploy this by default to production-like environments unless you intentionally want background traffic.

### `shoppingassistantservice`

Purpose:

- Optional AI assistant feature.
- Provides an HTTP endpoint consumed by the frontend assistant flow.

Deployment status:

- Present in the repo.
- Not part of the default Kubernetes kustomization.
- Requires Google Cloud/Gemini/AlloyDB-related configuration.

Platform note:

- Treat this as an optional feature integration.
- Do not enable `ENABLE_ASSISTANT=true` on the frontend unless this service and its cloud dependencies are configured.

## Public And Private Exposure

### Public Traffic

Only this service should receive public traffic:

| Service      | Reason                         |
| ------------ | ------------------------------ |
| `frontend` | Browser-facing web application |

In Kubernetes, the repo exposes frontend in two ways:

- internal `frontend` service using `ClusterIP`
- external `frontend-external` service using `LoadBalancer`

### Private Services

These should stay private inside the cluster:

- `productcatalogservice`
- `currencyservice`
- `cartservice`
- `redis-cart`
- `shippingservice`
- `paymentservice`
- `emailservice`
- `checkoutservice`
- `recommendationservice`
- `adservice`
- `shoppingassistantservice`
- `loadgenerator`

Platform rule:

```text
Only expose the frontend publicly.
Everything else should be internal service-to-service traffic.
```

## Service Dependency Map

```text
frontend
  -> productcatalogservice
  -> currencyservice
  -> cartservice
  -> recommendationservice
  -> shippingservice
  -> checkoutservice
  -> adservice
  -> shoppingassistantservice optional

recommendationservice
  -> productcatalogservice

checkoutservice
  -> productcatalogservice
  -> cartservice
  -> currencyservice
  -> shippingservice
  -> paymentservice
  -> emailservice

cartservice
  -> redis-cart

loadgenerator
  -> frontend

shoppingassistantservice
  -> Google Secret Manager
  -> AlloyDB
  -> Gemini / Google Generative AI APIs
```

## Internal Service Addresses

These are the expected in-cluster service addresses.

| Consumer                  | Environment variable                | Value                                                                        |
| ------------------------- | ----------------------------------- | ---------------------------------------------------------------------------- |
| `frontend`              | `PRODUCT_CATALOG_SERVICE_ADDR`    | `productcatalogservice:3550`                                               |
| `frontend`              | `CURRENCY_SERVICE_ADDR`           | `currencyservice:7000`                                                     |
| `frontend`              | `CART_SERVICE_ADDR`               | `cartservice:7070`                                                         |
| `frontend`              | `RECOMMENDATION_SERVICE_ADDR`     | `recommendationservice:8080`                                               |
| `frontend`              | `SHIPPING_SERVICE_ADDR`           | `shippingservice:50051`                                                    |
| `frontend`              | `CHECKOUT_SERVICE_ADDR`           | `checkoutservice:5050`                                                     |
| `frontend`              | `AD_SERVICE_ADDR`                 | `adservice:9555`                                                           |
| `frontend`              | `SHOPPING_ASSISTANT_SERVICE_ADDR` | `shoppingassistantservice:80`                                              |
| `checkoutservice`       | `PRODUCT_CATALOG_SERVICE_ADDR`    | `productcatalogservice:3550`                                               |
| `checkoutservice`       | `SHIPPING_SERVICE_ADDR`           | `shippingservice:50051`                                                    |
| `checkoutservice`       | `PAYMENT_SERVICE_ADDR`            | `paymentservice:50051`                                                     |
| `checkoutservice`       | `EMAIL_SERVICE_ADDR`              | `emailservice:8080` in Kubernetes, `emailservice:5000` in Docker Compose |
| `checkoutservice`       | `CURRENCY_SERVICE_ADDR`           | `currencyservice:7000`                                                     |
| `checkoutservice`       | `CART_SERVICE_ADDR`               | `cartservice:7070`                                                         |
| `recommendationservice` | `PRODUCT_CATALOG_SERVICE_ADDR`    | `productcatalogservice:3550`                                               |
| `cartservice`           | `REDIS_ADDR`                      | `redis-cart:6379`                                                          |
| `loadgenerator`         | `FRONTEND_ADDR`                   | `frontend:80` in Kubernetes, `frontend:8080` in Docker Compose           |

Important:

```text
Do not use localhost for service-to-service communication in Kubernetes or ECS.
localhost means the same container/pod/task, not another service.
```

## External Dependencies

| Component                       | External dependency           | Required for default deployment? | Notes                                                                                          |
| ------------------------------- | ----------------------------- | -------------------------------- | ---------------------------------------------------------------------------------------------- |
| `cartservice`                 | Redis                         | Yes                              | Default repo deploys`redis-cart` in-cluster.                                                 |
|  `productcatalogservice`      | AlloyDB                       | No                               | Optional. If AlloyDB env vars are set, catalog loads from AlloyDB instead of`products.json`. |
| `cartservice`                 | Spanner                       | No                               | Optional alternative cart store.                                                               |
| `cartservice`                 | AlloyDB                       | No                               | Optional alternative cart store.                                                               |
| `shoppingassistantservice`    | Google Secret Manager         | Yes, if assistant enabled        | Reads database password from Secret Manager.                                                   |
| `shoppingassistantservice`    | AlloyDB                       | Yes, if assistant enabled        | Uses AlloyDB vector store.                                                                     |
| `shoppingassistantservice`    | Gemini / Google Generative AI | Yes, if assistant enabled        | Uses Google Generative AI models.                                                              |
| telemetry-enabled services      | OpenTelemetry Collector       | No by default                    | Required if enabling tracing.                                                                  |
| cloud profiler-enabled services | Cloud Profiler or equivalent  | No by default                    | Profiling is disabled in default manifests for most services.                                  |

## Background Workers, Cron Jobs, And One-Off Jobs

No required cron jobs or migration jobs are present in the default deployment.

Known non-request-serving component:

| Component         | Type                         | Required? | Notes                                                            |
| ----------------- | ---------------------------- | --------- | ---------------------------------------------------------------- |
| `loadgenerator` | Background traffic generator | No        | Generates synthetic frontend traffic. Use for testing/demo only. |

Potential migration/setup concern:

- If optional AlloyDB, Spanner, or shopping assistant features are enabled, the platform team must confirm database schema/table setup and secret provisioning. The default local/static catalog path does not require migrations.

## Protocol Inventory

| Protocol | Components                                                                                                                                                                                |
| -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| HTTP     | `frontend`, `shoppingassistantservice`                                                                                                                                                |
| gRPC     | `productcatalogservice`, `currencyservice`, `cartservice`, `shippingservice`, `paymentservice`, `emailservice`, `checkoutservice`, `recommendationservice`, `adservice` |
| TCP      | `redis-cart`                                                                                                                                                                            |

## Port Inventory

|      Port | Component                                | Protocol | Notes                                       |
| --------: | ---------------------------------------- | -------- | ------------------------------------------- |
|    `80` | `frontend` Kubernetes service          | HTTP     | Internal service port.                      |
|    `80` | `frontend-external` Kubernetes service | HTTP     | Public LoadBalancer service.                |
|  `8080` | `frontend` container                   | HTTP     | Actual app listen port.                     |
|  `3550` | `productcatalogservice`                | gRPC     | Product catalog API.                        |
|  `7000` | `currencyservice`                      | gRPC     | Currency API.                               |
|  `7070` | `cartservice`                          | gRPC     | Cart API.                                   |
|  `6379` | `redis-cart`                           | TCP      | Redis cache/cart store.                     |
| `50051` | `shippingservice`                      | gRPC     | Shipping API.                               |
| `50051` | `paymentservice`                       | gRPC     | Payment API. Separate pod, same port is OK. |
|  `8080` | `emailservice` in Kubernetes           | gRPC     | Docker Compose uses`5000`.                |
|  `5050` | `checkoutservice`                      | gRPC     | Checkout orchestration API.                 |
|  `8080` | `recommendationservice`                | gRPC     | Recommendation API.                         |
|  `9555` | `adservice`                            | gRPC     | Ads API.                                    |
|  `8080` | `shoppingassistantservice` container   | HTTP     | Optional service.                           |

## Entrypoint And Startup Command Details

| Component                    | Source entrypoint                                            | Container startup                                                                  |
| ---------------------------- | ------------------------------------------------------------ | ---------------------------------------------------------------------------------- |
| `frontend`                 | `src/frontend/main.go`                                     | `/src/server`                                                                    |
| `productcatalogservice`    | `src/productcatalogservice/server.go`                      | `/src/server`                                                                    |
| `checkoutservice`          | `src/checkoutservice/main.go`                              | `/src/checkoutservice`                                                           |
| `shippingservice`          | `src/shippingservice/main.go`                              | `/src/shippingservice`                                                           |
| `currencyservice`          | `src/currencyservice/server.js`                            | `node server.js`                                                                 |
| `paymentservice`           | `src/paymentservice/index.js`                              | `node index.js`                                                                  |
| `emailservice`             | `src/emailservice/email_server.py`                         | `python email_server.py`                                                         |
| `recommendationservice`    | `src/recommendationservice/recommendation_server.py`       | `python recommendation_server.py`                                                |
| `cartservice`              | `src/cartservice/src/Program.cs`                           | `/app/cartservice`                                                               |
| `adservice`                | `src/adservice/src/main/java/hipstershop/AdService.java`   | `/app/build/install/hipstershop/bin/AdService`                                   |
| `loadgenerator`            | `src/loadgenerator/locustfile.py`                          | `locust --host=http://${FRONTEND_ADDR} --headless -u ${USERS:-10} -r ${RATE:-1}` |
| `shoppingassistantservice` | `src/shoppingassistantservice/shoppingassistantservice.py` | `python shoppingassistantservice.py`                                             |

## Platform Engineering Notes

### What is already known from inventory

- This is a polyglot microservice app.
- The app is already split into independently deployable services.
- Service-to-service communication is mostly gRPC.
- The main public boundary is HTTP through `frontend`.
- Redis is the default state dependency for carts.
- Optional cloud dependencies exist but are not required for the default deployment.
- Load generation is included but should not be treated as a production app component.

### What needs special attention in later checklist stages

- Image builds for multiple runtimes.
- Correct service DNS and environment variables.
- gRPC health probes.
- Redis persistence decision.
- Public exposure only through `frontend`.
- Observability collector setup if tracing is required.
- Optional AI/AlloyDB assistant should be explicitly included or excluded.
- Difference between Docker Compose and Kubernetes ports for `emailservice`.

## Application Inventory Output

At the end of this stage, platform engineering should have:

- [X] A list of all services and supporting components.
- [X] A runtime/language map.
- [X] Entrypoints and startup commands.
- [X] Port and protocol inventory.
- [X] Dependency map.
- [X] External dependency list.
- [X] Public/private exposure decision.
- [X] Optional component list.
- [X] Notes for follow-up stages.

## Ready For Next Checklist Item

The next stage is:

```text
2. Containerization
```

In that stage, verify or create one container image per service, confirm Dockerfiles are correct, validate build contexts, and prove each container can start with only environment-based configuration.
