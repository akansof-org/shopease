# Cloud Handover Checklist: 5. Service Discovery

Scenario: the development team has handed this application to the platform engineering team. We are preparing it for cloud infrastructure, instrumentation, and deployment to Kubernetes or ECS.

This note covers the fifth checklist item only: **Service Discovery**.

## Objective

Cluster networking is different from local development.

This stage answers:

- How does each service find another service?
- Are any service-to-service calls using `localhost` incorrectly?
- Do service names, ports, protocols, and client configuration match?
- Which services are public?
- Which services are private?
- How do we validate DNS and connectivity inside the cluster?

## Checklist

- [x] Replace `localhost` dependencies with service discovery names.
- [x] Confirm internal service DNS names.
- [x] Confirm ports match container, service, and client config.
- [x] Confirm public services and private services are separated.
- [ ] Confirm services can resolve each other by DNS in the deployed cluster.
- [x] Confirm protocol expectations: HTTP vs gRPC vs raw TCP.

Note:

- Items marked checked are confirmed by reviewing manifests and application configuration.
- DNS resolution must be validated in a real cluster after deployment.

## Important Rule

```text
localhost inside a container means the same container, pod, or task.
It does not mean another service.
```

Good examples:

```text
productcatalogservice:3550
cartservice:7070
redis-cart:6379
```

Bad examples for service-to-service calls inside a cluster:

```text
localhost:3550
127.0.0.1:7070
```

## Service Discovery Model

This app uses environment variables to pass dependency addresses into each service.

The values are Kubernetes service names plus ports:

```text
<service-name>:<service-port>
```

Examples:

```text
productcatalogservice:3550
currencyservice:7000
checkoutservice:5050
```

In Kubernetes, short names resolve within the same namespace.

Fully qualified form:

```text
service-name.namespace.svc.cluster.local:port
```

Example:

```text
productcatalogservice.default.svc.cluster.local:3550
```

## Public vs Private Services

Only the frontend should receive public traffic.

| Service | Kubernetes service type | Public? | Notes |
| --- | --- | --- | --- |
| `frontend-external` | `LoadBalancer` | Yes | Public entrypoint. Routes to frontend pods. |
| `frontend` | `ClusterIP` | No | Internal frontend service. |
| `productcatalogservice` | `ClusterIP` | No | Internal gRPC backend. |
| `currencyservice` | `ClusterIP` | No | Internal gRPC backend. |
| `cartservice` | `ClusterIP` | No | Internal gRPC backend. |
| `redis-cart` | `ClusterIP` | No | Internal TCP Redis. |
| `shippingservice` | `ClusterIP` | No | Internal gRPC backend. |
| `paymentservice` | `ClusterIP` | No | Internal gRPC backend. |
| `emailservice` | `ClusterIP` | No | Internal gRPC backend. |
| `checkoutservice` | `ClusterIP` | No | Internal gRPC backend. |
| `recommendationservice` | `ClusterIP` | No | Internal gRPC backend. |
| `adservice` | `ClusterIP` | No | Internal gRPC backend. |
| `loadgenerator` | No service | No | Background traffic generator. |
| `shoppingassistantservice` | Optional | No | Not part of default manifests. |

Platform rule:

```text
Expose frontend only.
Keep all backend services private.
```

## Internal DNS And Port Inventory

These are the expected in-cluster names and ports.

| DNS name | Service port | Target container port | Protocol | Notes |
| --- | ---: | ---: | --- | --- |
| `frontend` | `80` | `8080` | HTTP | Internal frontend service. |
| `frontend-external` | `80` | `8080` | HTTP | Public LoadBalancer frontend service. |
| `productcatalogservice` | `3550` | `3550` | gRPC | Product catalog API. |
| `currencyservice` | `7000` | `7000` | gRPC | Currency API. |
| `cartservice` | `7070` | `7070` | gRPC | Cart API. |
| `redis-cart` | `6379` | `6379` | TCP | Redis cart store. |
| `shippingservice` | `50051` | `50051` | gRPC | Shipping API. |
| `paymentservice` | `50051` | `50051` | gRPC | Payment API. |
| `emailservice` | `5000` | `8080` | gRPC | Service port differs from container port. |
| `checkoutservice` | `5050` | `5050` | gRPC | Checkout orchestration API. |
| `recommendationservice` | `8080` | `8080` | gRPC | Recommendation API. |
| `adservice` | `9555` | `9555` | gRPC | Ads API. |

Important detail:

```text
emailservice listens on container port 8080.
Its Kubernetes Service exposes port 5000 and forwards to targetPort 8080.
Clients should call emailservice:5000.
```

## Dependency Address Map

### frontend dependencies

| Env var | Value | Protocol |
| --- | --- | --- |
| `PRODUCT_CATALOG_SERVICE_ADDR` | `productcatalogservice:3550` | gRPC |
| `CURRENCY_SERVICE_ADDR` | `currencyservice:7000` | gRPC |
| `CART_SERVICE_ADDR` | `cartservice:7070` | gRPC |
| `RECOMMENDATION_SERVICE_ADDR` | `recommendationservice:8080` | gRPC |
| `SHIPPING_SERVICE_ADDR` | `shippingservice:50051` | gRPC |
| `CHECKOUT_SERVICE_ADDR` | `checkoutservice:5050` | gRPC |
| `AD_SERVICE_ADDR` | `adservice:9555` | gRPC |
| `SHOPPING_ASSISTANT_SERVICE_ADDR` | `shoppingassistantservice:80` | HTTP |

Note:

- `SHOPPING_ASSISTANT_SERVICE_ADDR` is configured even though `shoppingassistantservice` is not deployed by default.
- Do not enable assistant features unless the optional assistant service exists and is reachable.

### checkoutservice dependencies

| Env var | Value | Protocol |
| --- | --- | --- |
| `PRODUCT_CATALOG_SERVICE_ADDR` | `productcatalogservice:3550` | gRPC |
| `SHIPPING_SERVICE_ADDR` | `shippingservice:50051` | gRPC |
| `PAYMENT_SERVICE_ADDR` | `paymentservice:50051` | gRPC |
| `EMAIL_SERVICE_ADDR` | `emailservice:5000` | gRPC |
| `CURRENCY_SERVICE_ADDR` | `currencyservice:7000` | gRPC |
| `CART_SERVICE_ADDR` | `cartservice:7070` | gRPC |

### recommendationservice dependencies

| Env var | Value | Protocol |
| --- | --- | --- |
| `PRODUCT_CATALOG_SERVICE_ADDR` | `productcatalogservice:3550` | gRPC |

### cartservice dependencies

| Env var | Value | Protocol |
| --- | --- | --- |
| `REDIS_ADDR` | `redis-cart:6379` | TCP |

### loadgenerator dependencies

| Env var | Value | Protocol |
| --- | --- | --- |
| `FRONTEND_ADDR` | `frontend:80` in Kubernetes | HTTP |

## Protocol Expectations

| Protocol | Services |
| --- | --- |
| HTTP | `frontend`, optional `shoppingassistantservice`, `loadgenerator` as HTTP client |
| gRPC | `productcatalogservice`, `currencyservice`, `cartservice`, `shippingservice`, `paymentservice`, `emailservice`, `checkoutservice`, `recommendationservice`, `adservice` |
| TCP | `redis-cart` |

Platform note:

```text
Do not test gRPC services with normal curl.
Use grpcurl, Postman gRPC, BloomRPC, or application smoke tests.
```

## Kubernetes Examples

Short service name in same namespace:

```text
productcatalogservice:3550
```

Fully qualified DNS:

```text
productcatalogservice.default.svc.cluster.local:3550
```

HTTP frontend:

```text
frontend:80
```

Redis:

```text
redis-cart:6379
```

## ECS Examples

In ECS, service discovery may come from:

- AWS Cloud Map service discovery name
- Internal load balancer DNS name
- ECS Service Connect endpoint
- App Mesh virtual service name

Example patterns:

```text
productcatalogservice.online-boutique.local:3550
cartservice.online-boutique.local:7070
redis.internal.example:6379
```

or:

```text
productcatalogservice:3550
```

if using ECS Service Connect names.

Platform note:

```text
The names do not have to match Kubernetes names, but the application env vars must be set to whatever ECS can resolve.
```

## Local vs Cluster Discovery

| Environment | Discovery style | Example |
| --- | --- | --- |
| Direct local process | localhost ports | `localhost:3550` |
| Docker Compose | Compose service names | `productcatalogservice:3550` |
| Kubernetes | Kubernetes service names | `productcatalogservice:3550` |
| ECS | Cloud Map, Service Connect, or internal LB names | `productcatalogservice:3550` or FQDN |

Important:

- `localhost` is valid only when the target service is running in the same network namespace.
- For separate containers, pods, or ECS tasks, use service discovery names.

## Validation: Kubernetes DNS

Run a temporary debug pod:

```sh
kubectl run netcheck --rm -it --image=busybox:1.38.0 -- sh
```

Inside the pod:

```sh
nslookup frontend
nslookup productcatalogservice
nslookup currencyservice
nslookup cartservice
nslookup redis-cart
nslookup checkoutservice
```

Test HTTP service:

```sh
wget -qO- http://frontend/_healthz
```

Test TCP Redis port:

```sh
nc -vz redis-cart 6379
```

BusyBox images may not include every tool depending on tag. If needed, use a richer debug image:

```sh
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash
```

## Validation: Kubernetes Services And Endpoints

Confirm Services exist:

```sh
kubectl get svc
```

Confirm endpoints exist:

```sh
kubectl get endpoints
```

Check one service:

```sh
kubectl describe svc productcatalogservice
kubectl get endpoints productcatalogservice
```

Expected:

- Service exists.
- Selector matches pod labels.
- Endpoints are populated with pod IPs.
- Port and targetPort are correct.

## Validation: gRPC Connectivity

Use `grpcurl` from a debug image that has it installed, or port-forward locally.

Port-forward example:

```sh
kubectl port-forward svc/productcatalogservice 3550:3550
```

Then from local machine:

```sh
grpcurl -plaintext -proto protos/demo.proto -d '{}' \
  localhost:3550 hipstershop.ProductCatalogService/ListProducts
```

For in-cluster gRPC testing, use a debug container with `grpcurl`:

```sh
grpcurl -plaintext productcatalogservice:3550 list
```

If reflection is not enabled on a service, provide the proto file or test through the application path.

## Common Service Discovery Failure Modes

| Symptom | Likely cause | Check |
| --- | --- | --- |
| `connection refused` | Wrong port or app not listening | Container port, service targetPort, app logs |
| `no such host` | DNS/service name wrong | `kubectl get svc`, `nslookup` |
| Service has no endpoints | Selector mismatch or pods not ready | `kubectl get endpoints`, pod labels |
| HTTP works but gRPC fails | Wrong protocol, proxy, ingress, or health check style | Service port names, mesh config, client protocol |
| Frontend starts but page errors | Backend service address wrong or backend unhealthy | Frontend logs, backend logs |
| Checkout fails | One of cart/catalog/currency/shipping/payment/email unreachable | Checkout logs and dependency endpoints |
| Redis errors | `REDIS_ADDR` wrong or Redis service unavailable | `redis-cart` service/endpoints |
| Assistant feature fails | `shoppingassistantservice` not deployed or not reachable | `ENABLE_ASSISTANT`, assistant service DNS |

## Service Discovery Risks And Follow-Ups

| Area | Risk | Follow-up |
| --- | --- | --- |
| Optional assistant | Frontend has assistant address configured but service is not deployed by default | Keep assistant disabled or deploy/configure service. |
| Email port mapping | `emailservice` service port is `5000`, container port is `8080` | Keep `EMAIL_SERVICE_ADDR=emailservice:5000` in Kubernetes. |
| gRPC protocol | gRPC backends may not be testable with normal HTTP tools | Use gRPC-aware probes and clients. |
| Direct manifest changes | Changing service names breaks env var references | Update both Service names and client env vars. |
| Namespace changes | Short service names only resolve in same namespace | Use FQDN or deploy all services together. |
| Service mesh | Mesh may require protocol naming and probe handling | Keep port names accurate and review mesh annotations. |
| ECS migration | Kubernetes DNS names do not automatically exist in ECS | Map env vars to Cloud Map, Service Connect, or internal LB names. |

## Service Discovery Output

At the end of this stage, platform engineering should have:

- [x] Internal DNS names documented.
- [x] Service port and targetPort map documented.
- [x] Client env var dependency map documented.
- [x] Public/private separation documented.
- [x] Protocol expectations documented.
- [x] Known discovery risks documented.
- [ ] In-cluster DNS resolution validated.
- [ ] In-cluster service connectivity validated.
- [ ] gRPC connectivity validated for key services.

## Ready For Next Checklist Item

The next stage is:

```text
6. Kubernetes Or ECS Manifests
```

In that stage, verify the actual deployment objects: Deployments, Services, task definitions, service accounts, probes, resources, security context, and exposure configuration.

