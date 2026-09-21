# Cloud Handover Checklist: 7. Health Checks

Scenario: the development team has handed this application to the platform engineering team. We are preparing it for cloud infrastructure, instrumentation, and deployment to Kubernetes or ECS.

This note covers the seventh checklist item only: **Health Checks**.

## Objective

Health checks must reflect whether the service can receive traffic.

This stage answers:

- What health endpoint or health RPC does each component expose?
- Is readiness configured?
- Is liveness configured?
- Is a startup probe needed?
- Do checks use the right protocol: HTTP, gRPC, TCP, or command?
- Do checks require authentication?
- Do checks depend on fragile downstream services?
- Will the target platform support the configured health check style?

## Checklist

- [x] Identify health endpoint or health RPC.
- [x] Configure readiness checks.
- [x] Configure liveness checks.
- [ ] Configure startup checks for slow-starting services if needed.
- [x] Confirm health checks use the correct protocol.
- [x] Confirm health checks do not require authentication unless intentionally handled.
- [x] Confirm health checks do not depend on fragile downstream services unless appropriate.

Note:

- The default Kubernetes manifests include readiness and liveness checks for request-serving services.
- No startup probes are defined in the default manifests.
- `loadgenerator` has an init container that waits for frontend, but no readiness/liveness probe.

## Common Probe Types

| Protocol | Health check style |
| --- | --- |
| HTTP | `GET /health`, `/ready`, `/_healthz` |
| gRPC | gRPC health checking protocol |
| TCP | TCP socket check |
| Worker | Custom command or process check |

## Health Check Inventory

| Component | Protocol | Health implementation | Readiness probe | Liveness probe | Startup probe |
| --- | --- | --- | --- | --- | --- |
| `frontend` | HTTP | `GET /_healthz` returns `ok` | HTTP `/_healthz` on port `8080` | HTTP `/_healthz` on port `8080` | None |
| `productcatalogservice` | gRPC | gRPC health service | gRPC port `3550` | gRPC port `3550` | None |
| `currencyservice` | gRPC | gRPC health service | gRPC port `7000` | gRPC port `7000` | None |
| `cartservice` | gRPC | gRPC health service checks cart store ping | gRPC port `7070` | gRPC port `7070` | None |
| `redis-cart` | TCP | TCP port availability | TCP port `6379` | TCP port `6379` | None |
| `shippingservice` | gRPC | gRPC health service | gRPC port `50051` | gRPC port `50051` | None |
| `paymentservice` | gRPC | gRPC health service | gRPC port `50051` | gRPC port `50051` | None |
| `emailservice` | gRPC | gRPC health service | gRPC container port `8080` | gRPC container port `8080` | None |
| `checkoutservice` | gRPC | gRPC health service | gRPC port `5050` | gRPC port `5050` | None |
| `recommendationservice` | gRPC | gRPC health service | gRPC port `8080` | gRPC port `8080` | None |
| `adservice` | gRPC | gRPC health service | gRPC port `9555` | gRPC port `9555` | None |
| `loadgenerator` | Worker/client | Init container waits for frontend HTTP 200 | None | None | None |
| `shoppingassistantservice` | HTTP | No default manifest in core app | Not in default manifests | Not in default manifests | None |

## Probe Configuration Summary

### HTTP probe

`frontend` uses HTTP health checks:

```text
GET /_healthz on container port 8080
```

The handler returns:

```text
ok
```

Platform notes:

- Probe does not require authentication.
- Probe does not call downstream services.
- This is a lightweight process-level health check.

### gRPC probes

Most backend services use Kubernetes native gRPC probes.

Services:

- `productcatalogservice`
- `currencyservice`
- `cartservice`
- `shippingservice`
- `paymentservice`
- `emailservice`
- `checkoutservice`
- `recommendationservice`
- `adservice`

Platform notes:

- Confirm the target Kubernetes version supports native gRPC probes.
- If native gRPC probes are unsupported, use `grpc-health-probe` or expose HTTP health endpoints.
- gRPC probes use container ports, not service ports.

Important `emailservice` detail:

```text
emailservice Kubernetes Service exposes port 5000.
emailservice container listens on port 8080.
The gRPC probe correctly checks container port 8080.
Clients call emailservice:5000.
```

### TCP probes

`redis-cart` uses TCP socket checks:

```text
tcpSocket port 6379
```

Platform notes:

- TCP probe confirms Redis port is open.
- It does not prove Redis auth, data durability, or command success.
- For production Redis, prefer managed service health checks and application-level validation.

### Worker/init check

`loadgenerator` uses an init container named `frontend-check`.

Behavior:

- Attempts to reach `http://frontend:80`.
- Retries 12 times.
- Sleeps 10 seconds between retries.
- Main loadgenerator container starts only after frontend returns HTTP `200`.

Platform notes:

- This is dependency gating, not a liveness/readiness probe.
- `loadgenerator` is optional and should not be deployed by default in production.

## Readiness vs Liveness

### Readiness

Readiness means:

```text
Can this pod receive traffic right now?
```

If readiness fails:

- Kubernetes removes the pod from service endpoints.
- The pod keeps running.
- Traffic stops going to that pod until it becomes ready again.

### Liveness

Liveness means:

```text
Is this process stuck or unrecoverable?
```

If liveness fails:

- Kubernetes restarts the container.

Platform warning:

```text
Do not make liveness checks too dependent on downstream services.
If a database or dependency fails, restarting every app pod may make the outage worse.
```

## Downstream Dependency Behavior

| Component | Health depends on downstream service? | Notes |
| --- | --- | --- |
| `frontend` | No | `/_healthz` returns local `ok`; it does not test all backends. |
| `productcatalogservice` | No for local JSON mode | Health server is separate from catalog operations. |
| `currencyservice` | No | Health check returns serving. |
| `cartservice` | Yes, indirectly | Health service calls cart store `Ping()`, so Redis/store health can affect readiness/liveness. |
| `redis-cart` | Local process only | TCP check only. |
| `checkoutservice` | No | Health check returns serving; startup can fail if required gRPC addresses are missing. |
| `recommendationservice` | No for health RPC | Runtime request path depends on product catalog. |
| `shippingservice` | No | Health check returns serving. |
| `paymentservice` | No | Health check returns serving. |
| `emailservice` | No | Health check returns serving. |
| `adservice` | No | Uses gRPC health status manager. |

Platform note:

```text
Most health checks are process-level checks, not full dependency checks.
Use separate smoke tests or synthetic checks for full user journey validation.
```

## Kubernetes Probe Inventory

| Component | Readiness details | Liveness details |
| --- | --- | --- |
| `frontend` | HTTP `/_healthz`, port `8080`, initial delay `10s` | HTTP `/_healthz`, port `8080`, initial delay `10s` |
| `productcatalogservice` | gRPC port `3550` | gRPC port `3550` |
| `currencyservice` | gRPC port `7000` | gRPC port `7000` |
| `cartservice` | gRPC port `7070`, initial delay `15s` | gRPC port `7070`, initial delay `15s`, period `10s` |
| `redis-cart` | TCP port `6379`, period `5s` | TCP port `6379`, period `5s` |
| `shippingservice` | gRPC port `50051`, period `5s` | gRPC port `50051` |
| `paymentservice` | gRPC port `50051` | gRPC port `50051` |
| `emailservice` | gRPC port `8080`, period `5s` | gRPC port `8080`, period `5s` |
| `checkoutservice` | gRPC port `5050` | gRPC port `5050` |
| `recommendationservice` | gRPC port `8080`, period `5s` | gRPC port `8080`, period `5s` |
| `adservice` | gRPC port `9555`, initial delay `20s`, period `15s` | gRPC port `9555`, initial delay `20s`, period `15s` |

## Startup Probe Decision

No startup probes are currently configured.

When to add startup probes:

- service has slow cold starts
- Java/.NET service takes longer under low CPU
- dependency initialization delays startup
- image pulls are fast, but app boot is slow
- liveness probe kills the app before startup completes

Likely candidates if startup issues appear:

- `adservice`, because Java services can have slower startup
- `cartservice`, because .NET startup plus Redis/store check may need time
- `shoppingassistantservice`, if enabled, because cloud clients and model/vector store setup may be slow

Example startup probe:

```yaml
startupProbe:
  grpc:
    port: 9555
  failureThreshold: 30
  periodSeconds: 5
```

## Kubernetes Validation Commands

Check probe config:

```sh
kubectl describe deployment frontend
kubectl describe deployment productcatalogservice
kubectl describe deployment cartservice
```

Check pod readiness:

```sh
kubectl get pods
```

Check failing probe events:

```sh
kubectl describe pod <pod-name>
```

Look for messages like:

```text
Readiness probe failed
Liveness probe failed
Back-off restarting failed container
```

Check service endpoints:

```sh
kubectl get endpoints
```

If readiness fails, endpoints may be missing.

## Manual Health Testing

### Frontend HTTP health

Port-forward:

```sh
kubectl port-forward svc/frontend 8080:80
```

Test:

```sh
curl -i http://localhost:8080/_healthz
```

Expected:

```text
HTTP 200
ok
```

### gRPC health check

Port-forward a service:

```sh
kubectl port-forward svc/productcatalogservice 3550:3550
```

Test with `grpcurl`:

```sh
grpcurl -plaintext localhost:3550 grpc.health.v1.Health/Check
```

If reflection is unavailable, use the health proto or a debug image with `grpcurl`.

### Redis TCP health

From a debug pod:

```sh
nc -vz redis-cart 6379
```

## ECS Health Check Translation

ECS does not map one-to-one with Kubernetes probes.

### HTTP frontend

Use an ALB target group health check:

```text
Path: /_healthz
Port: traffic port
Expected status: 200
```

### gRPC backends

Options:

- container health check command using `grpc-health-probe`
- app-level HTTP health endpoint
- ECS Service Connect health behavior plus app smoke tests
- internal load balancer health checks if services are behind target groups

Example container health command:

```json
{
  "healthCheck": {
    "command": [
      "CMD-SHELL",
      "grpc-health-probe -addr=:3550 || exit 1"
    ],
    "interval": 10,
    "timeout": 5,
    "retries": 3,
    "startPeriod": 30
  }
}
```

Platform note:

```text
If using ECS for gRPC services, decide early how health checks will work.
Native Kubernetes gRPC probes do not automatically translate to ECS.
```

## Health Check Risks And Follow-Ups

| Area | Risk | Follow-up |
| --- | --- | --- |
| Kubernetes version | Native gRPC probes require modern Kubernetes support | Confirm target cluster version. |
| Startup probes | None defined | Add if slow-starting services are killed before ready. |
| Cart health | `cartservice` health depends on cart store ping | Confirm behavior during Redis outage. |
| Frontend health | `/_healthz` does not validate backend dependencies | Add synthetic checks for full user journey. |
| Redis health | TCP check only confirms port open | Use stronger Redis health if needed. |
| ECS migration | gRPC health checks need translation | Add `grpc-health-probe` or HTTP health endpoint. |
| Assistant service | Optional assistant lacks default health manifest | Add health endpoint/probes if enabled. |

## Health Check Output

At the end of this stage, platform engineering should have:

- [x] Health endpoints/RPCs identified.
- [x] Readiness probes documented.
- [x] Liveness probes documented.
- [x] Protocol-specific probe styles documented.
- [x] Downstream dependency behavior documented.
- [x] Startup probe decision points documented.
- [x] ECS translation notes documented.
- [ ] Target Kubernetes version confirmed for gRPC probes.
- [ ] Probe behavior validated after deployment.
- [ ] Decision made on startup probes.
- [ ] Synthetic end-to-end health checks defined if needed.

## Ready For Next Checklist Item

The next stage is:

```text
8. Storage And State
```

In that stage, identify stateful dependencies, persistence requirements, backup/restore needs, managed service options, and what happens when pods or tasks restart.

