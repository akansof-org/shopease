# Cloud Handover Checklist: 11. Observability

Scenario: the development team has handed this application to the platform engineering team. We are preparing it for cloud infrastructure, instrumentation, and deployment to Kubernetes or ECS.

This note covers the eleventh checklist item only: **Observability**.

## Objective

You should be able to explain what is happening after deployment.

This stage answers:

- Where do logs go?
- Are logs structured?
- Are metrics collected?
- Are traces emitted across service calls?
- Are dashboards available?
- Are alerts defined for user-impacting and infrastructure failures?
- Can we see error rate, latency, saturation, restarts, and deployment events?
- Can platform engineering debug a failure without guessing?

## Checklist

- [x] Logs are written to stdout/stderr.
- [x] Logs are structured where possible.
- [ ] Metrics are exposed or collected.
- [ ] Traces are emitted for request flows if supported.
- [ ] Dashboards exist for service health.
- [ ] Alerts exist for user-impacting failures.
- [ ] Alerts exist for infrastructure failures.
- [ ] Error rate is visible.
- [ ] Latency is visible.
- [ ] Saturation is visible: CPU, memory, disk, network.
- [x] Pod/task restarts are visible.
- [x] Deployment events are visible.

Note:

- Logs are mostly application-ready.
- Kubernetes pod restarts and deployment events are available from the platform.
- Tracing, metrics, profiler, dashboards, and alerts require platform setup.
- The repo includes an optional Google Cloud Operations Kustomize component.

## Observability Layers

| Layer | What it answers | Example signals |
| --- | --- | --- |
| Logs | What happened? | request logs, errors, startup messages |
| Metrics | How much/how often/how saturated? | request count, error rate, latency, CPU, memory |
| Traces | Where did time go across services? | frontend -> checkout -> payment spans |
| Events | What changed in the platform? | deploys, restarts, scheduling failures |
| Alerts | What needs human attention? | high error rate, crash loops, saturation |

## Current Repo Observability State

| Area | Current state | Platform action |
| --- | --- | --- |
| Logs | Services log to stdout/stderr; many use structured JSON logging | Route container logs to central logging backend |
| Metrics | Optional collector component exists, but metrics support is noted as incomplete/coming | Define metrics backend and required app/platform metrics |
| Traces | App has OpenTelemetry hooks; disabled by default | Enable collector and tracing env vars |
| Profiler | Hooks exist in several services; disabled in default manifests | Enable only if profiler backend/IAM is ready |
| Dashboards | Not included as production dashboards | Build service and platform dashboards |
| Alerts | Not included | Define alert rules |
| Pod restarts | Available through Kubernetes | Surface in dashboards/alerts |
| Deployment events | Available through Kubernetes/Skaffold/kubectl | Capture in CI/CD and event monitoring |

## Logging

### Logging behavior

Most services write logs to stdout/stderr, which is the correct container pattern.

Structured logging examples:

| Service/runtime | Logging style |
| --- | --- |
| Go services | `logrus` JSON formatter |
| Node.js services | `pino` JSON logger |
| Python services | `python-json-logger` |
| Java adservice | Log4j2 JSON-style configuration |
| .NET cartservice | Console logging |

Platform expectation:

```text
The container runtime captures stdout/stderr.
The cluster or ECS logging agent forwards logs to the central logging backend.
```

Kubernetes options:

- Cloud provider managed container logging.
- Fluent Bit / Fluentd / Vector / OpenTelemetry Collector logs pipeline.
- Loki, Elasticsearch/OpenSearch, Splunk, Datadog, New Relic, Cloud Logging.

ECS options:

- `awslogs` log driver to CloudWatch Logs.
- FireLens with Fluent Bit.
- OpenTelemetry Collector sidecar/daemon.

### Logging checklist

- [ ] Confirm every container writes logs to stdout/stderr.
- [ ] Confirm logs are collected from all namespaces/services.
- [ ] Confirm logs include service name, severity, timestamp.
- [ ] Confirm error logs are searchable.
- [ ] Confirm request logs exist for frontend and critical backends.
- [ ] Confirm sensitive data is not logged.
- [ ] Confirm log retention policy.
- [ ] Confirm log access controls.

## Tracing

### Default state

Tracing is disabled by default.

The repo includes OpenTelemetry instrumentation in several services and an optional Kustomize component:

```text
kustomize/components/google-cloud-operations
```

Enable:

```sh
cd kustomize
kustomize edit add component components/google-cloud-operations
kubectl apply -k .
```

This component adds:

- `opentelemetrycollector` Deployment
- `opentelemetrycollector` Service on port `4317`
- collector ConfigMap
- env vars such as `ENABLE_TRACING=1`
- env vars such as `COLLECTOR_SERVICE_ADDR=opentelemetrycollector:4317`

### Trace path

Expected trace flow:

```text
instrumented service
  -> OTLP gRPC
  -> opentelemetrycollector:4317
  -> observability backend
```

For Google Cloud Operations, the collector exports to Google Cloud.

### Services with tracing hooks

| Service | Tracing support |
| --- | --- |
| `frontend` | OpenTelemetry tracing support |
| `productcatalogservice` | OpenTelemetry tracing support |
| `checkoutservice` | OpenTelemetry tracing support |
| `currencyservice` | OpenTelemetry tracing support |
| `paymentservice` | OpenTelemetry tracing support |
| `emailservice` | OpenTelemetry tracing support |
| `recommendationservice` | OpenTelemetry tracing support |
| `shippingservice` | Tracing code path marked temporarily unavailable |
| `adservice` | Tracing code path marked temporarily unavailable |
| `cartservice` | No obvious custom tracing setup in reviewed files |

Platform caution:

```text
Do not assume every service emits complete traces.
Validate trace coverage after enabling telemetry.
```

### Tracing checklist

- [ ] Deploy OpenTelemetry Collector or equivalent.
- [ ] Configure `COLLECTOR_SERVICE_ADDR`.
- [ ] Enable tracing env vars.
- [ ] Confirm collector can export to backend.
- [ ] Configure IAM/permissions for backend export.
- [ ] Validate a frontend request creates trace spans.
- [ ] Validate service names are correct.
- [ ] Validate sampling policy.
- [ ] Validate trace retention.

## Metrics

### Default state

Application metrics are not fully production-ready by default.

The Google Cloud Operations README notes:

```text
Currently only trace is supported. Support for metrics, and more is coming soon.
```

However, platform metrics are still available from Kubernetes/ECS:

- CPU
- memory
- restarts
- pod readiness
- deployment status
- node saturation
- network/disk metrics depending on platform

### Metrics to collect

| Metric category | Examples |
| --- | --- |
| Traffic | request count, gRPC calls, frontend requests |
| Errors | HTTP 5xx, gRPC error codes, failed checkout attempts |
| Latency | frontend request latency, checkout latency, backend RPC latency |
| Saturation | CPU, memory, disk, network, Redis memory |
| Availability | pod readiness, health probe failures, service endpoints |
| Runtime | restarts, OOM kills, crash loops |
| Storage | Redis availability, Redis memory, DB connection errors |

### Metrics checklist

- [ ] Confirm platform metrics collection is enabled.
- [ ] Confirm app metrics source.
- [ ] Confirm gRPC metrics strategy.
- [ ] Confirm HTTP metrics strategy for frontend.
- [ ] Confirm Redis metrics strategy.
- [ ] Confirm node/task resource metrics.
- [ ] Confirm metrics labels include service, namespace/environment, pod/task.
- [ ] Confirm retention period.

## Profiler

Several services include profiler hooks.

Default manifests usually disable profiler:

- `DISABLE_PROFILER=1`
- `ENABLE_PROFILER=0`

The Google Cloud Operations component enables profiler for some services.

Platform checklist:

- [ ] Decide if profiler is required.
- [ ] Confirm profiler backend.
- [ ] Confirm IAM permissions.
- [ ] Confirm overhead is acceptable.
- [ ] Enable only in approved environments.

## Dashboards

Dashboards are not included as production-ready assets in the default repo.

Minimum dashboard set:

### Application overview

- frontend availability
- request rate
- error rate
- latency
- checkout success/failure
- top failing services
- deployment version/image tag

### Service dashboard

For each service:

- pod/task status
- restart count
- CPU/memory
- request/RPC rate
- error rate
- latency
- readiness/liveness failures
- logs link
- trace link

### Infrastructure dashboard

- node/task capacity
- CPU saturation
- memory saturation
- network saturation
- disk saturation
- load balancer health
- Redis/database health

### Dependency dashboard

- Redis availability
- external cloud APIs
- OpenTelemetry Collector health
- managed DB/cache availability

## Alerts

Minimum useful alerts:

- High error rate.
- High latency.
- Pod/task crash looping.
- Readiness failures.
- CPU or memory saturation.
- Disk saturation for stateful services.
- External dependency failures.

Recommended alert table:

| Alert | Signal | Severity | Notes |
| --- | --- | --- | --- |
| Frontend unavailable | frontend ready pods = 0 or LB unhealthy | Critical | User-facing outage |
| High frontend 5xx | HTTP 5xx rate above threshold | Critical | User-facing errors |
| High checkout failures | checkout gRPC errors above threshold | Critical | Purchase flow broken |
| High latency | p95/p99 above SLO | Warning/Critical | Tune threshold by service |
| CrashLoopBackOff | pod restart loop | Critical | Workload cannot stay up |
| Readiness failures | ready pods below expected | Warning/Critical | Service capacity reduced |
| CPU saturation | CPU near limits/request for sustained window | Warning | Scaling/resource issue |
| Memory saturation/OOM | OOMKilled or memory near limit | Critical | Stability issue |
| Redis unavailable | Redis endpoint unhealthy or cart errors | Critical | Cart flow broken |
| Collector down | OpenTelemetry Collector unavailable | Warning | Observability degraded |
| Image pull failures | pods stuck ImagePullBackOff | Critical | Deployment failure |

Platform note:

```text
Alert on symptoms first, causes second.
User-facing availability and error rate matter more than raw CPU alone.
```

## SLI/SLO Starting Point

Suggested starting SLIs:

| User journey | SLI |
| --- | --- |
| Browse homepage | HTTP success rate and latency |
| View product page | HTTP success rate and latency |
| Add to cart | HTTP/gRPC success rate |
| Checkout | checkout success rate and latency |

Suggested service-level indicators:

- availability: successful requests / total requests
- latency: p95 and p99
- error rate: HTTP 5xx and gRPC non-OK
- saturation: CPU/memory/disk/network

Example starting SLOs for non-production:

```text
Frontend availability: 99%
Checkout availability: 99%
Frontend p95 latency: under 500ms
Checkout p95 latency: under 1500ms
```

Adjust with product expectations and real baseline data.

## Deployment Events And Restarts

Kubernetes visibility:

```sh
kubectl get pods
kubectl describe pod <pod>
kubectl get events --sort-by=.lastTimestamp
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
```

Signals to surface:

- deployment started
- deployment completed
- deployment failed
- pod scheduled
- pod restarted
- image pull failed
- readiness failed
- liveness restarted pod
- OOMKilled
- CrashLoopBackOff

ECS visibility:

- ECS service events
- task stopped reasons
- CloudWatch Logs
- target group health
- deployment circuit breaker events
- CloudTrail for deployment actions

## OpenTelemetry Collector Operations

If using the provided collector:

Objects:

- Deployment: `opentelemetrycollector`
- Service: `opentelemetrycollector`
- Port: `4317`
- ConfigMap: `collector-gateway-config-template`

Validate:

```sh
kubectl get deploy opentelemetrycollector
kubectl get svc opentelemetrycollector
kubectl logs deployment/opentelemetrycollector
```

Common issues:

| Symptom | Likely cause |
| --- | --- |
| No traces | tracing env vars missing, collector unreachable, app not instrumented |
| Collector PermissionDenied | missing cloud IAM role |
| Collector crash | config error |
| Partial traces | not all services instrumented or context propagation gap |
| `localhost:4317` errors | missing `COLLECTOR_SERVICE_ADDR` in cluster |

## Validation Commands

Check logs:

```sh
kubectl logs deployment/frontend
kubectl logs deployment/checkoutservice
kubectl logs deployment/cartservice
```

Check previous crashed container logs:

```sh
kubectl logs <pod-name> --previous
```

Check events:

```sh
kubectl get events --sort-by=.lastTimestamp
```

Check pod restarts:

```sh
kubectl get pods
```

Check resource usage:

```sh
kubectl top pods
kubectl top nodes
```

Check rollouts:

```sh
kubectl rollout status deployment/frontend
kubectl rollout history deployment/frontend
```

Render telemetry-enabled manifests:

```sh
cd kustomize
kustomize edit add component components/google-cloud-operations
kubectl kustomize .
```

## ECS Observability Translation

For ECS:

| Need | ECS/AWS option |
| --- | --- |
| Logs | CloudWatch Logs via `awslogs` or FireLens |
| Metrics | CloudWatch Container Insights |
| Traces | AWS X-Ray or OpenTelemetry Collector |
| Deployment events | ECS service events, EventBridge |
| Alerts | CloudWatch Alarms, PagerDuty/Opsgenie integration |
| Dashboards | CloudWatch dashboards, Grafana, Datadog, New Relic |
| Load balancer health | ALB/NLB target group metrics |

ECS checklist:

- [ ] Configure log group per service or app.
- [ ] Configure log retention.
- [ ] Enable Container Insights.
- [ ] Configure tracing sidecar or collector.
- [ ] Add target group health alarms.
- [ ] Add ECS task restart/stopped alarms.
- [ ] Add CPU/memory alarms.
- [ ] Add ALB 5xx/latency alarms.

## Observability Risks And Follow-Ups

| Area | Risk | Follow-up |
| --- | --- | --- |
| Tracing disabled | No distributed request visibility | Enable collector and tracing vars. |
| Metrics incomplete | App-level error/latency may be missing | Add HTTP/gRPC metrics instrumentation or mesh/sidecar metrics. |
| No dashboards | Operators rely on kubectl/logs only | Build app, service, infrastructure dashboards. |
| No alerts | Failures discovered by users | Add alert rules before production. |
| Partial instrumentation | Some services have unavailable or missing tracing | Validate trace coverage per service. |
| Collector IAM | Collector cannot export telemetry | Configure cloud IAM/workload identity. |
| Logs may contain sensitive data | Payment/cart/order logs may include request details | Review log redaction. |
| Optional services | Assistant/loadgenerator may add noise | Separate dashboards/alerts for optional components. |

## Observability Output

At the end of this stage, platform engineering should have:

- [x] Logging behavior documented.
- [x] Telemetry component identified.
- [x] Tracing enablement path documented.
- [x] Metrics gaps documented.
- [x] Dashboard requirements documented.
- [x] Minimum useful alerts documented.
- [x] Kubernetes/ECS validation commands documented.
- [ ] Centralized logging configured.
- [ ] Metrics collection configured.
- [ ] Tracing enabled and validated.
- [ ] Dashboards created.
- [ ] Alerts created and tested.
- [ ] SLI/SLOs agreed with product/service owners.

## Ready For Next Checklist Item

The next stage is:

```text
12. Resource Management
```

In that stage, define CPU/memory requests and limits, replica counts, autoscaling, PodDisruptionBudgets, and capacity planning for the app.

