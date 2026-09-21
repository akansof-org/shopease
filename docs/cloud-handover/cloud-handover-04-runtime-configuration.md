# Cloud Handover Checklist: 4. Runtime Configuration

Scenario: the development team has handed this application to the platform engineering team. We are preparing it for cloud infrastructure, instrumentation, and deployment to Kubernetes or ECS.

This note covers the fourth checklist item only: **Runtime Configuration**.

## Objective

Document every variable needed at runtime.

This stage answers:

- Which environment variables are required?
- Which variables are optional?
- Which values differ between local, dev, staging, and production?
- Which values are secrets?
- Which values are service addresses?
- Which values are feature flags?
- Which values configure telemetry?
- Are any environment-specific values hardcoded in source code?

## Checklist

- [x] List required environment variables.
- [x] List optional environment variables.
- [x] Identify default values.
- [x] Identify secrets.
- [x] Identify config that differs per environment.
- [x] Identify service addresses.
- [x] Identify feature flags.
- [x] Identify telemetry configuration.
- [x] Confirm no environment-specific value is hardcoded in source code.

Note:

- Most values are environment-variable based.
- Some optional cloud integrations introduce secrets and cloud-specific configuration.
- A few hardcoded or inconsistent defaults need platform attention before production use.

## Recommended Config Sources

### Kubernetes

Use:

- `ConfigMap` for non-secret config.
- `Secret` for secret values.
- External Secrets Operator or equivalent for cloud secret manager integration.
- Sealed Secrets or SOPS if secrets must be stored in Git.
- Cloud-native identity where possible instead of static credentials.

### ECS

Use:

- Task definition environment variables for non-secret config.
- AWS Secrets Manager for secrets.
- SSM Parameter Store for config values.
- IAM task roles for cloud permissions.

## Configuration Categories

| Category | Examples | Recommended source |
| --- | --- | --- |
| Service ports | `PORT`, `ASPNETCORE_HTTP_PORTS` | Deployment/task env |
| Internal service addresses | `PRODUCT_CATALOG_SERVICE_ADDR`, `REDIS_ADDR` | ConfigMap or task env |
| Feature flags | `ENABLE_ASSISTANT`, `CYMBAL_BRANDING` | ConfigMap or task env |
| Telemetry | `ENABLE_TRACING`, `COLLECTOR_SERVICE_ADDR` | ConfigMap or task env |
| Secrets | database passwords, API keys | Secret manager, Kubernetes Secret, ECS secrets |
| Cloud resource identifiers | `PROJECT_ID`, `REGION`, `ALLOYDB_*` | ConfigMap plus Secret for sensitive values |

## Core Runtime Configuration

These variables are required for the default app deployment.

| Service | Name | Required? | Example | Source | Secret? | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| `frontend` | `PORT` | Yes | `8080` | env | No | HTTP listen port. Defaults to `8080` in code. |
| `frontend` | `PRODUCT_CATALOG_SERVICE_ADDR` | Yes | `productcatalogservice:3550` | env/ConfigMap | No | Required by startup. |
| `frontend` | `CURRENCY_SERVICE_ADDR` | Yes | `currencyservice:7000` | env/ConfigMap | No | Required by startup. |
| `frontend` | `CART_SERVICE_ADDR` | Yes | `cartservice:7070` | env/ConfigMap | No | Required by startup. |
| `frontend` | `RECOMMENDATION_SERVICE_ADDR` | Yes | `recommendationservice:8080` | env/ConfigMap | No | Required by startup. |
| `frontend` | `SHIPPING_SERVICE_ADDR` | Yes | `shippingservice:50051` | env/ConfigMap | No | Required by startup. |
| `frontend` | `CHECKOUT_SERVICE_ADDR` | Yes | `checkoutservice:5050` | env/ConfigMap | No | Required by startup. |
| `frontend` | `AD_SERVICE_ADDR` | Yes | `adservice:9555` | env/ConfigMap | No | Required by startup. |
| `frontend` | `SHOPPING_ASSISTANT_SERVICE_ADDR` | Yes | `shoppingassistantservice:80` | env/ConfigMap | No | Required by startup even when assistant UI is disabled. |
| `productcatalogservice` | `PORT` | Yes | `3550` | env | No | gRPC listen port. Defaults to `3550` in code. |
| `currencyservice` | `PORT` | Yes | `7000` | env | No | gRPC listen port. No safe app-level default observed. |
| `cartservice` | `REDIS_ADDR` | Yes for clustered deployment | `redis-cart:6379` | env/ConfigMap | No | If unset, app falls back to in-memory cart store. |
| `shippingservice` | `PORT` | Yes | `50051` | env | No | gRPC listen port. Defaults to `50051` in code. |
| `paymentservice` | `PORT` | Yes | `50051` | env | No | gRPC listen port. No safe app-level default observed. |
| `emailservice` | `PORT` | Yes | `8080` | env | No | gRPC listen port. Defaults to `8080` in code. Docker Compose uses `5000`. |
| `checkoutservice` | `PORT` | Yes | `5050` | env | No | gRPC listen port. Defaults to `5050` in code. |
| `checkoutservice` | `PRODUCT_CATALOG_SERVICE_ADDR` | Yes | `productcatalogservice:3550` | env/ConfigMap | No | Required by startup. |
| `checkoutservice` | `SHIPPING_SERVICE_ADDR` | Yes | `shippingservice:50051` | env/ConfigMap | No | Required by startup. |
| `checkoutservice` | `PAYMENT_SERVICE_ADDR` | Yes | `paymentservice:50051` | env/ConfigMap | No | Required by startup. |
| `checkoutservice` | `EMAIL_SERVICE_ADDR` | Yes | `emailservice:8080` | env/ConfigMap | No | Required by startup. Docker Compose uses `emailservice:5000`. |
| `checkoutservice` | `CURRENCY_SERVICE_ADDR` | Yes | `currencyservice:7000` | env/ConfigMap | No | Required by startup. |
| `checkoutservice` | `CART_SERVICE_ADDR` | Yes | `cartservice:7070` | env/ConfigMap | No | Required by startup. |
| `recommendationservice` | `PORT` | Yes | `8080` | env | No | gRPC listen port. Defaults to `8080`. |
| `recommendationservice` | `PRODUCT_CATALOG_SERVICE_ADDR` | Yes | `productcatalogservice:3550` | env/ConfigMap | No | Required by startup. |
| `adservice` | `PORT` | Yes | `9555` | env | No | gRPC listen port. Defaults to `9555`. |

## Optional Runtime Configuration

These values enable optional features, cloud integrations, demos, or local variants.

| Service | Name | Required? | Example | Source | Secret? | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| `frontend` | `BASE_URL` | No | `/shop` | env/ConfigMap | No | Prefixes frontend routes. Empty by default. |
| `frontend` | `LISTEN_ADDR` | No | `0.0.0.0` | env/ConfigMap | No | Bind address. Empty means listen on all interfaces when combined with `:<port>`. |
| `frontend` | `ENV_PLATFORM` | No | `gcp`, `aws`, `azure`, `local` | env/ConfigMap | No | UI platform indicator. Defaults to local unless GCP metadata detected. |
| `frontend` | `FRONTEND_MESSAGE` | No | `Welcome` | env/ConfigMap | No | Optional banner/message. |
| `frontend` | `CYMBAL_BRANDING` | No | `true` | env/ConfigMap | No | Branding feature flag. |
| `frontend` | `ENABLE_ASSISTANT` | No | `true` | env/ConfigMap | No | Enables assistant UI/flow. Requires assistant service to be configured. |
| `frontend` | `BANNER_COLOR` | No | `blue` | env/ConfigMap | No | Demo/canary visual config. |
| `frontend` | `ENABLE_SINGLE_SHARED_SESSION` | No | `true` | env/ConfigMap | No | Session behavior flag. |
| `frontend` | `PACKAGING_SERVICE_URL` | No | `http://packaging.example` | env/ConfigMap | No | Optional packaging service integration. |
| `productcatalogservice` | `EXTRA_LATENCY` | No | `250ms` | env/ConfigMap | No | Injects artificial latency. Useful for demos/testing. |
| `loadgenerator` | `FRONTEND_ADDR` | Yes if deployed | `frontend:80` | env/ConfigMap | No | Target frontend address. |
| `loadgenerator` | `USERS` | No | `10` | env/ConfigMap | No | Number of simulated users. Defaults to `10` in Docker entrypoint. |
| `loadgenerator` | `RATE` | No | `1` | env/ConfigMap | No | Spawn rate. Defaults to `1` in Docker entrypoint. |

## Telemetry Configuration

Telemetry is mostly opt-in or disabled by default in manifests.

| Service | Name | Required? | Example | Source | Secret? | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| `frontend` | `ENABLE_TRACING` | No | `1` | env/ConfigMap | No | Enables OpenTelemetry tracing. |
| `frontend` | `COLLECTOR_SERVICE_ADDR` | Required if tracing enabled | `otel-collector:4317` | env/ConfigMap | No | OTLP gRPC collector address. |
| `frontend` | `ENABLE_PROFILER` | No | `0` | env/ConfigMap | No | Profiler flag. Default manifests set `0`. |
| `productcatalogservice` | `ENABLE_TRACING` | No | `1` | env/ConfigMap | No | Enables tracing. |
| `productcatalogservice` | `COLLECTOR_SERVICE_ADDR` | Required if tracing enabled | `otel-collector:4317` | env/ConfigMap | No | Required by tracing setup. |
| `productcatalogservice` | `DISABLE_PROFILER` | No | `1` | env/ConfigMap | No | Disables profiler when set. Default manifests set this. |
| `currencyservice` | `ENABLE_TRACING` | No | `1` | env/ConfigMap | No | Enables tracing. |
| `currencyservice` | `COLLECTOR_SERVICE_ADDR` | Required if tracing enabled | `otel-collector:4317` | env/ConfigMap | No | OTLP gRPC collector URL/address. |
| `currencyservice` | `OTEL_SERVICE_NAME` | No | `currencyservice` | env/ConfigMap | No | Overrides telemetry service name. |
| `currencyservice` | `DISABLE_PROFILER` | No | `1` | env/ConfigMap | No | Disables profiler when set. |
| `paymentservice` | `ENABLE_TRACING` | No | `1` | env/ConfigMap | No | Enables tracing. |
| `paymentservice` | `COLLECTOR_SERVICE_ADDR` | Required if tracing enabled | `otel-collector:4317` | env/ConfigMap | No | OTLP gRPC collector URL/address. |
| `paymentservice` | `OTEL_SERVICE_NAME` | No | `paymentservice` | env/ConfigMap | No | Overrides telemetry service name. |
| `paymentservice` | `DISABLE_PROFILER` | No | `1` | env/ConfigMap | No | Disables profiler when set. |
| `checkoutservice` | `ENABLE_TRACING` | No | `1` | env/ConfigMap | No | Enables tracing. |
| `checkoutservice` | `COLLECTOR_SERVICE_ADDR` | Required if tracing enabled | `otel-collector:4317` | env/ConfigMap | No | Required by tracing setup. |
| `checkoutservice` | `ENABLE_PROFILER` | No | `0` | env/ConfigMap | No | Enables profiler only when set to `1`. |
| `recommendationservice` | `ENABLE_TRACING` | No | `1` | env/ConfigMap | No | Enables tracing. |
| `recommendationservice` | `COLLECTOR_SERVICE_ADDR` | No | `otel-collector:4317` | env/ConfigMap | No | Defaults to `localhost:4317` if tracing enabled and not set. |
| `recommendationservice` | `DISABLE_PROFILER` | No | `1` | env/ConfigMap | No | Disables profiler when set. |
| `emailservice` | `ENABLE_TRACING` | No | `1` | env/ConfigMap | No | Enables tracing. |
| `emailservice` | `COLLECTOR_SERVICE_ADDR` | No | `otel-collector:4317` | env/ConfigMap | No | Defaults to `localhost:4317` if tracing enabled and not set. |
| `emailservice` | `DISABLE_PROFILER` | No | `1` | env/ConfigMap | No | Disables profiler when set. |
| `shippingservice` | `DISABLE_TRACING` | No | `1` | env/ConfigMap | No | Tracing is enabled unless this is set, but implementation is marked temporarily unavailable. |
| `shippingservice` | `DISABLE_PROFILER` | No | `1` | env/ConfigMap | No | Disables profiler when set. |
| `shippingservice` | `DISABLE_STATS` | No | `1` | env/ConfigMap | No | Disables stats path. |
| `adservice` | `DISABLE_TRACING` | No | `1` | env/ConfigMap | No | Disables tracing. |
| `adservice` | `DISABLE_STATS` | No | `1` | env/ConfigMap | No | Disables stats. |

Platform note:

```text
Do not enable tracing until an OpenTelemetry Collector exists and COLLECTOR_SERVICE_ADDR is configured consistently.
```

## Optional Database And Cloud Configuration

These values are not needed for the default deployment, but they become important if optional cloud-backed features are enabled.

### Product catalog backed by AlloyDB

`productcatalogservice` loads from local `products.json` by default. If `ALLOYDB_CLUSTER_NAME` is set, it switches to AlloyDB mode.

| Service | Name | Required? | Example | Source | Secret? | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| `productcatalogservice` | `PROJECT_ID` | Required for AlloyDB mode | `my-gcp-project` | ConfigMap/env | No | Cloud project. |
| `productcatalogservice` | `REGION` | Required for AlloyDB mode | `us-central1` | ConfigMap/env | No | AlloyDB region. |
| `productcatalogservice` | `ALLOYDB_CLUSTER_NAME` | Enables/required for AlloyDB mode | `catalog-cluster` | ConfigMap/env | No | If set, app loads from AlloyDB. |
| `productcatalogservice` | `ALLOYDB_INSTANCE_NAME` | Required for AlloyDB mode | `catalog-primary` | ConfigMap/env | No | AlloyDB instance. |
| `productcatalogservice` | `ALLOYDB_DATABASE_NAME` | Required for AlloyDB mode | `products` | ConfigMap/env | No | Database name. |
| `productcatalogservice` | `ALLOYDB_TABLE_NAME` | Required for AlloyDB mode | `products` | ConfigMap/env | No | Table name. |
| `productcatalogservice` | `ALLOYDB_SECRET_NAME` | Required for AlloyDB mode | `catalog-db-password` | ConfigMap/env | No | Secret Manager secret name. The secret payload is sensitive. |

### Cart storage alternatives

`cartservice` defaults to Redis when `REDIS_ADDR` is set. It also has optional Spanner and AlloyDB paths.

| Service | Name | Required? | Example | Source | Secret? | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| `cartservice` | `SPANNER_PROJECT` | No | `my-gcp-project` | ConfigMap/env | No | Enables Spanner path with project-based config. |
| `cartservice` | `SPANNER_CONNECTION_STRING` | No | connection string | Secret/env | Possibly | Alternative Spanner config. Treat as sensitive if credentials included. |
| `cartservice` | `ALLOYDB_PRIMARY_IP` | No | `10.0.0.5` | ConfigMap/env | No | Enables AlloyDB cart store path. |
| `cartservice` | `PROJECT_ID` | Required for AlloyDB cart path | `my-gcp-project` | ConfigMap/env | No | Used for Secret Manager. |
| `cartservice` | `ALLOYDB_SECRET_NAME` | Required for AlloyDB cart path | `cart-db-password` | ConfigMap/env | No | Secret Manager secret name. Payload is sensitive. |
| `cartservice` | `ALLOYDB_DATABASE_NAME` | Required for AlloyDB cart path | `cart` | ConfigMap/env | No | Database name. |
| `cartservice` | `ALLOYDB_TABLE_NAME` | Required for AlloyDB cart path | `cart_items` | ConfigMap/env | No | Table name. |

### Shopping assistant

`shoppingassistantservice` is optional and requires cloud integrations.

| Service | Name | Required? | Example | Source | Secret? | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| `shoppingassistantservice` | `PROJECT_ID` | Yes if deployed | `my-gcp-project` | ConfigMap/env | No | Google Cloud project. |
| `shoppingassistantservice` | `REGION` | Yes if deployed | `us-central1` | ConfigMap/env | No | Region. |
| `shoppingassistantservice` | `ALLOYDB_DATABASE_NAME` | Yes if deployed | `assistant` | ConfigMap/env | No | AlloyDB database. |
| `shoppingassistantservice` | `ALLOYDB_TABLE_NAME` | Yes if deployed | `product_embeddings` | ConfigMap/env | No | Vector table. |
| `shoppingassistantservice` | `ALLOYDB_CLUSTER_NAME` | Yes if deployed | `assistant-cluster` | ConfigMap/env | No | AlloyDB cluster. |
| `shoppingassistantservice` | `ALLOYDB_INSTANCE_NAME` | Yes if deployed | `assistant-primary` | ConfigMap/env | No | AlloyDB instance. |
| `shoppingassistantservice` | `ALLOYDB_SECRET_NAME` | Yes if deployed | `assistant-db-password` | ConfigMap/env | No | Secret Manager secret name. Payload is sensitive. |

Platform note:

```text
For cloud secret manager references, the secret name itself is usually not secret.
The secret value fetched at runtime is secret.
The workload identity/IAM permission to read it must be tightly controlled.
```

## Feature Flags

| Service | Name | Default | Notes |
| --- | --- | --- | --- |
| `frontend` | `ENABLE_ASSISTANT` | false | Enables assistant UI. Requires optional assistant service. |
| `frontend` | `CYMBAL_BRANDING` | false | Switches branding. |
| `frontend` | `ENABLE_SINGLE_SHARED_SESSION` | false | Changes session behavior. |
| `frontend` | `BANNER_COLOR` | empty | Demo/canary UI config. |
| `frontend` | `FRONTEND_MESSAGE` | empty | Optional UI message. |
| `productcatalogservice` | `EXTRA_LATENCY` | `0` | Injects artificial latency. |

## Config That Differs Per Environment

These values commonly change between local, dev, staging, and production.

| Config | Local | Kubernetes/ECS | Notes |
| --- | --- | --- | --- |
| Internal service addresses | Docker Compose service names or localhost ports | Kubernetes service names or ECS service discovery | Must not use localhost for cross-service calls. |
| `ENV_PLATFORM` | `local` | `gcp`, `aws`, `azure`, `onprem`, etc. | Mainly frontend UI/platform display. |
| `FRONTEND_ADDR` | `frontend:8080` or `localhost:8080` | `frontend:80` | Used by loadgenerator. |
| `EMAIL_SERVICE_ADDR` | `emailservice:5000` in Docker Compose | `emailservice:8080` in Kubernetes | Important port difference. |
| `REDIS_ADDR` | `redis-cart:6379` | `redis-cart:6379` or managed Redis endpoint | Managed Redis endpoint differs by environment. |
| `COLLECTOR_SERVICE_ADDR` | local collector or unset | `otel-collector:4317` | Required if tracing enabled. |
| Cloud resource IDs | usually unset | project/account/region-specific | Required for optional cloud integrations. |

## Secrets

Default deployment has minimal explicit secrets.

Secret-bearing or sensitive areas:

| Area | Secret? | Notes |
| --- | --- | --- |
| Redis password | No in default deployment | Default `redis-cart` has no password. Revisit for production. |
| AlloyDB password | Yes | Retrieved from Secret Manager using `ALLOYDB_SECRET_NAME`. |
| Spanner connection string | Possibly | Treat as secret if it contains credentials. |
| Gemini/API credentials | Yes/identity-based | Prefer workload identity or cloud-native identity, not static keys. |
| Registry credentials | Yes | Covered in checklist 3. |

Platform recommendation:

```text
Do not store secret values in ConfigMaps, plain manifests, Dockerfiles, or source code.
Use cloud secret managers or Kubernetes Secrets integrated with external secret tooling.
```

## Hardcoded Or Risky Configuration Findings

| Finding | Impact | Recommendation |
| --- | --- | --- |
| `shoppingassistantservice` Dockerfile sets `PORT=8080`, but the Flask app runs on hardcoded `8080` | Env `PORT` does not actually control the Flask port | Update app to read `PORT` if this service will be deployed generally. |
| `emailservice` uses `8080` in Kubernetes but `5000` in Docker Compose | Confusing between environments | Document clearly and standardize if possible. |
| `frontend` always requires `SHOPPING_ASSISTANT_SERVICE_ADDR` even when assistant is disabled | Frontend startup requires a value for optional service | Keep dummy/internal value configured, or change app to require it only when assistant is enabled. |
| `currencyservice` and `paymentservice` rely on `PORT` env without clear fallback | Missing `PORT` may cause startup/bind failure | Always set `PORT` in manifests/task definitions. |
| Some telemetry defaults use `localhost:4317` if collector address is missing | In cluster, localhost points to same pod/container | Always set `COLLECTOR_SERVICE_ADDR` when tracing is enabled. |
| Default Redis has no auth and uses in-cluster public image | Fine for demo, weak for production | Use managed Redis or secure Redis deployment for production. |

## Recommended Kubernetes Config Layout

Suggested split:

### ConfigMap: `online-boutique-config`

Non-secret values:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: online-boutique-config
data:
  PRODUCT_CATALOG_SERVICE_ADDR: productcatalogservice:3550
  CURRENCY_SERVICE_ADDR: currencyservice:7000
  CART_SERVICE_ADDR: cartservice:7070
  RECOMMENDATION_SERVICE_ADDR: recommendationservice:8080
  SHIPPING_SERVICE_ADDR: shippingservice:50051
  CHECKOUT_SERVICE_ADDR: checkoutservice:5050
  AD_SERVICE_ADDR: adservice:9555
  SHOPPING_ASSISTANT_SERVICE_ADDR: shoppingassistantservice:80
  REDIS_ADDR: redis-cart:6379
  ENV_PLATFORM: gcp
```

### Secret: `online-boutique-secrets`

Only if optional cloud/database integrations are enabled:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: online-boutique-secrets
type: Opaque
stringData:
  SPANNER_CONNECTION_STRING: ""
```

Prefer External Secrets or cloud secret manager integration for real secret values.

## Recommended ECS Config Layout

For ECS:

- Put non-secret values in task definition `environment`.
- Put secret values in task definition `secrets`.
- Store secrets in AWS Secrets Manager or SSM Parameter Store.
- Use task execution role for image pull.
- Use task role for app access to AWS APIs.

Example conceptual split:

```text
environment:
  PRODUCT_CATALOG_SERVICE_ADDR=productcatalogservice:3550
  CURRENCY_SERVICE_ADDR=currencyservice:7000
  CART_SERVICE_ADDR=cartservice:7070

secrets:
  SPANNER_CONNECTION_STRING=<Secrets Manager ARN>
```

## Validation Commands

Inspect env vars in Kubernetes manifests:

```sh
kubectl set env deployment/frontend --list
```

Check a running pod's environment:

```sh
kubectl exec deployment/frontend -- printenv
```

Check service DNS from a debug pod:

```sh
kubectl run netcheck --rm -it --image=busybox:1.38.0 -- sh
nslookup productcatalogservice
nslookup cartservice
```

Validate config rendering before deploy:

```sh
skaffold render --default-repo=<registry>/<repo>
```

Search source for env usage:

```sh
rg "os\\.Getenv|os\\.LookupEnv|process\\.env|Configuration\\[|environ\\[|environ\\.get|getenv" src
```

## Runtime Configuration Risks And Follow-Ups

| Area | Risk | Follow-up |
| --- | --- | --- |
| Service addresses | Wrong DNS/port breaks startup or runtime calls | Centralize addresses in ConfigMap/task env. |
| Optional assistant | Frontend references assistant address even when disabled | Keep configured placeholder or adjust code. |
| Email port mismatch | Compose and Kubernetes use different email ports | Standardize or document per environment. |
| Telemetry | Enabling tracing without collector breaks or degrades services | Deploy collector before enabling tracing. |
| Secrets | Optional cloud integrations need secret manager access | Define identity/IAM and secret ownership. |
| Redis | Default Redis config is demo-grade | Choose managed Redis or harden in-cluster Redis. |
| Hardcoded values | `shoppingassistantservice` ignores `PORT` env | Fix before treating it as portable. |

## Runtime Configuration Output

At the end of this stage, platform engineering should have:

- [x] Required env vars documented.
- [x] Optional env vars documented.
- [x] Service address map documented.
- [x] Feature flags documented.
- [x] Telemetry config documented.
- [x] Optional cloud/database config documented.
- [x] Secret-bearing config identified.
- [x] Environment-specific differences documented.
- [x] Hardcoded/risky config findings documented.
- [ ] ConfigMap/Secret or ECS task config created.
- [ ] Secret manager/IAM approach confirmed.
- [ ] Runtime config validated in deployed environment.

## Ready For Next Checklist Item

The next stage is:

```text
5. Service Discovery
```

In that stage, verify how services find each other inside Kubernetes or ECS, confirm DNS names, ports, protocols, and ensure no service-to-service call uses local-only assumptions.

