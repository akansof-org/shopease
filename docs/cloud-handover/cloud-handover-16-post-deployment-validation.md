# Cloud Handover Checklist: 16. Post-Deployment Validation

Scenario: the development team has handed this application to the platform engineering team. We are preparing it for cloud infrastructure, instrumentation, and deployment to Kubernetes or ECS.

This note covers the sixteenth checklist item only: **Post-Deployment Validation**.

## Objective

After deployment, prove the app works from the cluster outward.

This stage answers:

- Are pods or tasks running?
- Are readiness checks passing?
- Do Services, endpoints, target groups, and load balancers point to healthy workloads?
- Can services resolve each other through cluster/service discovery DNS?
- Can internal service-to-service traffic flow?
- Does the public endpoint work?
- Are logs, metrics, and traces flowing?
- Do smoke tests pass?
- Is a basic load test needed before declaring the deployment healthy?

## Checklist

- [ ] Confirm pods/tasks are running.
- [ ] Confirm readiness probes pass.
- [ ] Confirm services/endpoints exist.
- [ ] Confirm internal DNS resolution.
- [ ] Confirm internal service-to-service traffic.
- [ ] Confirm public endpoint works.
- [ ] Confirm logs are flowing.
- [ ] Confirm metrics are flowing.
- [ ] Confirm traces are flowing if enabled.
- [ ] Run smoke tests.
- [ ] Run basic load test if required.

Note:

- A successful `kubectl apply` is not proof that the app works.
- A running pod is not proof that the app can receive traffic.
- A healthy frontend process is not proof that all downstream services work.
- Post-deployment validation should check platform health, application health, and user-facing behavior.

## Kubernetes Commands

```bash
kubectl get pods
kubectl get svc
kubectl get endpoints
kubectl describe pod <pod-name>
kubectl logs deployment/<deployment-name>
kubectl rollout status deployment/<deployment-name>
```

More complete command set:

```bash
kubectl get deployments
kubectl get pods -o wide
kubectl get svc
kubectl get endpoints
kubectl get events --sort-by=.lastTimestamp
kubectl rollout status deployment/frontend
kubectl logs deployment/frontend --tail=100
```

With namespace:

```bash
kubectl get pods -n <namespace>
kubectl get svc -n <namespace>
kubectl get endpoints -n <namespace>
kubectl rollout status deployment/frontend -n <namespace>
kubectl logs deployment/frontend -n <namespace> --tail=100
```

## Smoke Test Examples

Generic examples:

```bash
curl -i http://<public-url>/health
curl -i http://<public-url>/
```

For this app, use the actual frontend health endpoint:

```bash
curl -i http://<public-url>/_healthz
curl -i http://<public-url>/
```

Expected frontend health response:

```text
ok
```

## Current App Validation Map

| Component | What to validate | Example |
| --- | --- | --- |
| `frontend` | Public HTTP endpoint and internal service calls | `curl http://<public-url>/_healthz`, browse homepage |
| `frontend-external` | Public LoadBalancer service | `kubectl get svc frontend-external` |
| Backend gRPC services | Pods ready, endpoints present, gRPC health passing | `kubectl get endpoints`, `grpcurl` from debug pod |
| `redis-cart` | Pod ready, Redis port reachable, cart path works | TCP probe, cart smoke test |
| `loadgenerator` | Optional traffic generator running and producing successful requests | Check loadgenerator logs |
| Observability collector | Collector running if enabled | `kubectl logs deployment/opentelemetrycollector` |

## Confirm Pods Or Tasks Are Running

Kubernetes:

```bash
kubectl get pods -n <namespace>
kubectl get deployments -n <namespace>
kubectl get replicasets -n <namespace>
```

What to look for:

- `READY` should show expected containers ready.
- `STATUS` should be `Running` or `Completed` for jobs.
- `RESTARTS` should not continuously increase.
- `AGE` should match the deployment window.

Common bad states:

| State | Meaning |
| --- | --- |
| `ImagePullBackOff` | Image cannot be pulled. |
| `ErrImagePull` | Image pull failed immediately. |
| `CrashLoopBackOff` | Container starts and repeatedly crashes. |
| `Pending` | Scheduler cannot place the pod. |
| `CreateContainerConfigError` | Missing config, secret, or invalid container setup. |
| `RunContainerError` | Runtime failed to start the container. |
| `OOMKilled` | Container exceeded memory limit. |

ECS:

```bash
aws ecs describe-services --cluster <cluster-name> --services <service-name>
aws ecs list-tasks --cluster <cluster-name> --service-name <service-name>
aws ecs describe-tasks --cluster <cluster-name> --tasks <task-arn>
```

Check:

- Desired count equals running count.
- No repeated task stops.
- Service events do not show placement, health check, image pull, or IAM errors.

## Confirm Readiness Probes Pass

Kubernetes:

```bash
kubectl get pods -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
```

Look for:

- `Ready=True`.
- No repeated readiness probe failures.
- Deployment available condition is true.

Rollout check:

```bash
kubectl rollout status deployment/<deployment-name> -n <namespace>
```

For this app:

- `frontend` uses HTTP `/_healthz`.
- Most backend services use gRPC health checks.
- `redis-cart` uses TCP health checks.
- `loadgenerator` has an init container that waits for frontend, but no readiness/liveness probe.

## Confirm Services And Endpoints Exist

Kubernetes Services provide stable addresses. Endpoints prove the Service has ready backend pods.

Commands:

```bash
kubectl get svc -n <namespace>
kubectl get endpoints -n <namespace>
kubectl describe svc frontend -n <namespace>
kubectl describe endpoints frontend -n <namespace>
```

Checklist:

- [ ] Each internal service has a `ClusterIP` service.
- [ ] `frontend-external` or replacement public entrypoint exists.
- [ ] Endpoints are populated for each Service.
- [ ] Service ports match app/client expectations.
- [ ] No Service points to zero ready endpoints.

Important interpretation:

```text
Service exists but endpoints are empty usually means labels do not match,
pods are not ready, or pods are not running.
```

## Confirm Internal DNS Resolution

Use a temporary debug pod:

```bash
kubectl run dns-test \
  --rm -i --restart=Never \
  -n <namespace> \
  --image=busybox:1.36 \
  -- nslookup frontend
```

Test service DNS names:

```bash
kubectl run dns-test \
  --rm -i --restart=Never \
  -n <namespace> \
  --image=busybox:1.36 \
  -- nslookup productcatalogservice
```

Names to validate:

```text
frontend
productcatalogservice
currencyservice
cartservice
redis-cart
shippingservice
paymentservice
emailservice
checkoutservice
recommendationservice
adservice
```

If services are in another namespace, test the fully qualified service name:

```text
service-name.namespace.svc.cluster.local
```

## Confirm Internal Service-To-Service Traffic

HTTP example for frontend health inside the cluster:

```bash
kubectl run curl-test \
  --rm -i --restart=Never \
  -n <namespace> \
  --image=curlimages/curl \
  -- curl -i http://frontend:80/_healthz
```

gRPC example from a debug pod with `grpcurl`:

```bash
grpcurl -plaintext productcatalogservice:3550 grpc.health.v1.Health/Check
```

If running from your local machine, port-forward first:

```bash
kubectl port-forward svc/productcatalogservice 3550:3550 -n <namespace>
grpcurl -plaintext localhost:3550 grpc.health.v1.Health/Check
```

Checklist:

- [ ] Frontend can reach backend services.
- [ ] gRPC services respond on expected ports.
- [ ] Redis/cart path works.
- [ ] Network policies do not block required traffic.
- [ ] Service mesh policy does not block required traffic.
- [ ] DNS names match runtime configuration.

## Confirm Public Endpoint Works

For default Kubernetes exposure:

```bash
kubectl get service frontend-external -n <namespace>
```

Get external IP:

```bash
kubectl get service frontend-external \
  -n <namespace> \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

Test:

```bash
curl -i http://<external-ip>/_healthz
curl -i http://<external-ip>/
```

If using DNS/TLS:

```bash
curl -i https://<public-host>/_healthz
curl -i https://<public-host>/
```

Checklist:

- [ ] Load balancer has an external address.
- [ ] DNS resolves to the correct load balancer.
- [ ] TLS certificate is valid if HTTPS is used.
- [ ] Frontend health endpoint works.
- [ ] Homepage loads.
- [ ] No obvious frontend errors.
- [ ] Public exposure matches intended access model.

## Confirm Logs Are Flowing

Kubernetes:

```bash
kubectl logs deployment/frontend -n <namespace> --tail=100
kubectl logs deployment/checkoutservice -n <namespace> --tail=100
kubectl logs deployment/cartservice -n <namespace> --tail=100
```

Previous crashed container logs:

```bash
kubectl logs <pod-name> -n <namespace> --previous
```

Checklist:

- [ ] Logs are visible with `kubectl logs`.
- [ ] Logs reach centralized logging backend.
- [ ] Logs include service name, severity, and timestamps where possible.
- [ ] No repeated startup errors.
- [ ] No repeated dependency connection failures.
- [ ] No crash loops or OOM messages.

ECS:

```bash
aws logs tail <log-group-name> --follow
```

Check CloudWatch Logs for:

- Application startup.
- Request errors.
- Dependency errors.
- Task restarts.
- Health check failures.

## Confirm Metrics Are Flowing

Metrics validation depends on your platform stack.

Kubernetes platform metrics:

```bash
kubectl top pods -n <namespace>
kubectl top nodes
```

Checklist:

- [ ] CPU usage visible.
- [ ] Memory usage visible.
- [ ] Pod restart count visible.
- [ ] Request rate visible if app/platform metrics are configured.
- [ ] Error rate visible.
- [ ] Latency visible.
- [ ] Dashboards show the new deployment.
- [ ] Alerts are evaluating.

If `kubectl top` fails, metrics-server may not be installed or ready.

## Confirm Traces Are Flowing If Enabled

This repo has optional OpenTelemetry support. If the collector is enabled, validate it.

Objects to check:

- Deployment: `opentelemetrycollector`
- Service: `opentelemetrycollector`
- Port: `4317`
- ConfigMap: `collector-gateway-config-template`

Commands:

```bash
kubectl get deploy opentelemetrycollector -n <namespace>
kubectl get svc opentelemetrycollector -n <namespace>
kubectl logs deployment/opentelemetrycollector -n <namespace> --tail=100
```

Checklist:

- [ ] Collector is running.
- [ ] Collector logs show successful export.
- [ ] App telemetry environment variables are set.
- [ ] Trace backend receives spans.
- [ ] A frontend request produces trace data.
- [ ] Traces include key service hops.

Common trace issues:

| Symptom | Likely cause |
| --- | --- |
| No traces | Tracing env vars missing, collector disabled, app instrumentation disabled. |
| Collector errors | Collector config or IAM/backend permission issue. |
| Partial traces | Some services not instrumented or context propagation gap. |
| `localhost:4317` errors | Collector address configured for local development instead of cluster service DNS. |

## Run Smoke Tests

Minimum smoke test set:

- [ ] Public frontend health returns success.
- [ ] Homepage returns success.
- [ ] Product listing page loads.
- [ ] Product detail page loads.
- [ ] Cart can be used if test data/session allows.
- [ ] Checkout path works in a safe test mode if enabled.
- [ ] No errors appear in frontend logs.
- [ ] No errors appear in backend logs.

Simple curl checks:

```bash
curl -f http://<public-url>/_healthz
curl -f http://<public-url>/
```

In-cluster smoke test:

```bash
kubectl run smoke-test \
  --rm -i --restart=Never \
  -n <namespace> \
  --image=curlimages/curl \
  -- curl -f http://frontend:80/_healthz
```

Use the repo's `loadgenerator` as a stronger smoke test when deployed:

```bash
kubectl delete pod -l app=loadgenerator -n <namespace>
kubectl logs -l app=loadgenerator -n <namespace> --tail=100
```

The existing GitHub Actions smoke test watches `loadgenerator` logs for aggregated request count and error count.

## Run Basic Load Test If Required

Use load testing when:

- Deploying to staging before production.
- Changing resource requests/limits.
- Changing autoscaling settings.
- Changing service mesh/networking policy.
- Changing database/cache configuration.
- Preparing for production traffic.

Basic checks during load:

- [ ] Request success rate is acceptable.
- [ ] Error rate stays low.
- [ ] Latency stays within target.
- [ ] CPU and memory stay within expected range.
- [ ] Pods/tasks do not restart.
- [ ] HPA/autoscaling behaves as expected if enabled.
- [ ] Logs do not show dependency saturation.

Kubernetes commands during load:

```bash
kubectl top pods -n <namespace>
kubectl get pods -n <namespace>
kubectl get hpa -n <namespace>
kubectl logs -l app=loadgenerator -n <namespace> --tail=100
```

Do not run the demo `loadgenerator` against production unless the team explicitly approves the traffic pattern.

## ECS Post-Deployment Validation

ECS equivalent checklist:

- [ ] ECS service desired count equals running count.
- [ ] Tasks are healthy.
- [ ] Target group targets are healthy.
- [ ] CloudWatch logs are flowing.
- [ ] ALB/NLB endpoint responds.
- [ ] Service discovery or Service Connect endpoints resolve.
- [ ] Task CPU and memory are stable.
- [ ] Deployment circuit breaker did not roll back.

Commands:

```bash
aws ecs describe-services --cluster <cluster-name> --services <service-name>
aws ecs list-tasks --cluster <cluster-name> --service-name <service-name>
aws ecs describe-tasks --cluster <cluster-name> --tasks <task-arn>
aws elbv2 describe-target-health --target-group-arn <target-group-arn>
aws logs tail <log-group-name> --follow
```

Smoke tests:

```bash
curl -i http://<alb-dns-name>/_healthz
curl -i http://<alb-dns-name>/
```

## Validation Evidence Template

Use this after every shared environment deployment:

| Validation item | Result | Evidence/link | Owner | Notes |
| --- | --- | --- | --- | --- |
| Rollout complete |  |  |  |  |
| Pods/tasks running |  |  |  |  |
| Readiness passing |  |  |  |  |
| Services/endpoints healthy |  |  |  |  |
| Internal DNS works |  |  |  |  |
| Internal traffic works |  |  |  |  |
| Public endpoint works |  |  |  |  |
| Logs flowing |  |  |  |  |
| Metrics flowing |  |  |  |  |
| Traces flowing |  |  |  |  |
| Smoke tests passed |  |  |  |  |
| Load test passed if required |  |  |  |  |

## Common Post-Deployment Failures

| Symptom | Likely cause | First checks |
| --- | --- | --- |
| `kubectl apply` worked but app unavailable | Rollout failed, readiness failed, Service has no endpoints | `kubectl get pods`, `kubectl get endpoints`, `kubectl describe pod`. |
| Public IP pending | Load balancer provisioning issue | Service events, cloud quotas, subnet/controller config. |
| Frontend health works but page errors | Backend dependency failure | Frontend logs, backend logs, service DNS. |
| Backend pods ready but frontend cannot call them | Wrong service address, port mismatch, network policy | Env vars, Service ports, NetworkPolicy. |
| gRPC health fails | Wrong port/protocol or service not serving | gRPC probe config, app logs, `grpcurl`. |
| Cart fails | Redis unreachable or cart store error | `cartservice` logs, Redis pod/service, network policy. |
| Logs missing | Logging agent/sidecar/permissions issue | Node logging agent, CloudWatch/collector config. |
| Metrics missing | Metrics server or collector missing | `kubectl top`, Prometheus/Cloud Monitoring config. |
| Traces missing | Collector disabled or env vars missing | Collector logs, telemetry config. |
| Restarts increasing | Crash, liveness failure, OOM, dependency error | `kubectl describe pod`, `kubectl logs --previous`. |

## Post-Deployment Go/No-Go

Go if:

- [ ] All expected workloads are running.
- [ ] Rollouts completed successfully.
- [ ] Readiness checks pass.
- [ ] Services and endpoints are populated.
- [ ] Internal DNS resolves.
- [ ] Internal traffic works.
- [ ] Public endpoint works.
- [ ] Logs are available.
- [ ] Metrics are available.
- [ ] Traces are available if enabled.
- [ ] Smoke tests pass.
- [ ] Error rate and latency are acceptable.
- [ ] No unexpected restarts or crash loops.

No-go if:

- [ ] Critical pods/tasks are not ready.
- [ ] Rollout is stuck or failed.
- [ ] Service endpoints are empty.
- [ ] Frontend cannot reach required backends.
- [ ] Public endpoint fails.
- [ ] Logs are unavailable during rollout.
- [ ] Error rate is elevated.
- [ ] Latency is unacceptable.
- [ ] Restarts are increasing.
- [ ] Required smoke tests fail.

## Output Checklist

Before moving to troubleshooting, produce:

- [ ] Rollout status evidence.
- [ ] Pod/task health evidence.
- [ ] Readiness evidence.
- [ ] Service/endpoint evidence.
- [ ] Internal DNS test result.
- [ ] Internal traffic test result.
- [ ] Public endpoint test result.
- [ ] Log flow confirmation.
- [ ] Metric flow confirmation.
- [ ] Trace flow confirmation if enabled.
- [ ] Smoke test result.
- [ ] Load test result if required.
- [ ] Decision: continue, roll back, or investigate.

## Ready For Next Checklist

After post-deployment validation is complete, move to:

```text
17. Troubleshooting Checklist
```

In that stage, diagnose failures such as pods not starting, image pulls failing, readiness probes failing, service discovery breaking, load balancers not becoming ready, missing logs/metrics/traces, and broken downstream dependencies.
