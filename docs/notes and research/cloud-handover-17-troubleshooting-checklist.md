# Cloud Handover Checklist: 17. Troubleshooting Checklist

Scenario: the development team has handed this application to the platform engineering team. We are preparing it for cloud infrastructure, instrumentation, and deployment to Kubernetes or ECS.

This note covers the seventeenth checklist item only: **Troubleshooting Checklist**.

## Objective

When something fails, isolate by layer.

This stage answers:

- Is the image pull failing?
- Is the container starting?
- Is the app process listening on the expected port?
- Are environment variables present?
- Are secrets mounted or injected?
- Are readiness/liveness probes failing?
- Is DNS resolving?
- Is the Service selector matching pod labels?
- Are endpoints populated?
- Are network policies or security groups blocking traffic?
- Is the app incorrectly calling `localhost` for another service?
- Are downstream dependencies healthy?
- Are CPU or memory limits causing throttling, restarts, or OOM kills?

## Checklist

- [ ] Is the image pull failing?
- [ ] Is the container starting?
- [ ] Is the process listening on the expected port?
- [ ] Are env vars present?
- [ ] Are secrets mounted/injected?
- [ ] Are probes failing?
- [ ] Is DNS resolving?
- [ ] Is the service selector matching pod labels?
- [ ] Are endpoints populated?
- [ ] Are network policies/security groups blocking traffic?
- [ ] Is the app trying to call `localhost` incorrectly?
- [ ] Are downstream dependencies healthy?
- [ ] Are resource limits causing OOM kills or throttling?

## Common Commands

```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
kubectl exec -it <pod-name> -- sh
kubectl get events --sort-by=.lastTimestamp
kubectl get endpoints <service-name>
```

With namespace:

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous
kubectl exec -it <pod-name> -n <namespace> -- sh
kubectl get events -n <namespace> --sort-by=.lastTimestamp
kubectl get endpoints <service-name> -n <namespace>
```

## Troubleshooting Order

Use this order when debugging:

1. Deployment/rollout status.
2. Pod/task status.
3. Image pull.
4. Container startup.
5. App logs.
6. Environment variables and secrets.
7. Process port/listener.
8. Probes.
9. Service selectors and endpoints.
10. DNS resolution.
11. Internal service traffic.
12. Network policies/security groups.
13. Downstream dependencies.
14. Resource pressure.
15. Public load balancer/ingress.

Platform rule:

```text
Start from the failing symptom, then move one layer inward or outward.
Do not guess across layers.
```

## Quick Triage Commands

Run these first:

```bash
kubectl get deployments -n <namespace>
kubectl get pods -n <namespace> -o wide
kubectl get svc -n <namespace>
kubectl get endpoints -n <namespace>
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

Check a specific deployment:

```bash
kubectl rollout status deployment/<deployment-name> -n <namespace>
kubectl describe deployment <deployment-name> -n <namespace>
kubectl logs deployment/<deployment-name> -n <namespace> --tail=100
```

Check recently failed containers:

```bash
kubectl logs <pod-name> -n <namespace> --previous
kubectl describe pod <pod-name> -n <namespace>
```

## Layer 1: Image Pull

Symptoms:

- Pod stuck in `ImagePullBackOff`.
- Pod stuck in `ErrImagePull`.
- Events show `pull access denied`.
- Events show `manifest unknown`.
- Events show architecture mismatch.

Commands:

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl get events -n <namespace> --sort-by=.lastTimestamp
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[*].image}'
```

What to check:

- Image exists in registry.
- Tag is correct.
- Digest is correct.
- Registry credentials are configured.
- Cluster nodes/tasks can access the registry.
- Image was built for the node CPU architecture.
- `imagePullSecrets` exist if required.

Common fixes:

| Cause | Fix |
| --- | --- |
| Wrong tag | Deploy the correct immutable tag or digest. |
| Image not pushed | Push image to target registry. |
| Private registry auth missing | Add `imagePullSecrets` or cloud IAM pull permissions. |
| Wrong architecture | Build `amd64`, `arm64`, or multi-arch image. |
| Registry unreachable | Check network egress, registry region, firewall, or VPC endpoint. |

## Layer 2: Container Startup

Symptoms:

- `CrashLoopBackOff`.
- `RunContainerError`.
- `CreateContainerConfigError`.
- Container exits immediately.
- Logs show missing file, bad command, bad config, or missing dependency.

Commands:

```bash
kubectl get pods -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous
```

What to check:

- Entrypoint command is correct.
- Required files are in the image.
- Runtime dependencies are installed.
- Environment variables are set.
- Secret/config references exist.
- Container filesystem permissions allow startup.
- Non-root user can read required files and bind required port.

Common fixes:

| Cause | Fix |
| --- | --- |
| Missing env var | Add env var to Deployment, ConfigMap, Secret, or task definition. |
| Missing Secret/ConfigMap | Create object in correct namespace or update manifest reference. |
| Bad command/args | Fix Dockerfile `CMD`/`ENTRYPOINT` or manifest command. |
| Permission denied | Fix file ownership, security context, or non-root runtime permissions. |
| Missing runtime file | Copy required file into image during build. |

## Layer 3: Process Listening On Expected Port

Symptoms:

- Pod is running but readiness fails.
- Service has endpoints but requests fail.
- Connection refused.
- Load balancer health check fails.

Commands:

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl exec -it <pod-name> -n <namespace> -- sh
```

Inside the pod, if tools exist:

```bash
netstat -tuln
ss -tuln
wget -qO- http://localhost:<port>/_healthz
```

If the image is minimal and has no shell/tools, use a debug pod:

```bash
kubectl run curl-test \
  --rm -i --restart=Never \
  -n <namespace> \
  --image=curlimages/curl \
  -- curl -i http://frontend:80/_healthz
```

Current app port reminders:

| Service | Protocol | Service port | Container port |
| --- | --- | ---: | ---: |
| `frontend` | HTTP | `80` | `8080` |
| `frontend-external` | HTTP | `80` | `8080` |
| `productcatalogservice` | gRPC | `3550` | `3550` |
| `currencyservice` | gRPC | `7000` | `7000` |
| `cartservice` | gRPC | `7070` | `7070` |
| `redis-cart` | TCP | `6379` | `6379` |
| `shippingservice` | gRPC | `50051` | `50051` |
| `paymentservice` | gRPC | `50051` | `50051` |
| `emailservice` | gRPC | `5000` | `8080` |
| `checkoutservice` | gRPC | `5050` | `5050` |
| `recommendationservice` | gRPC | `8080` | `8080` |
| `adservice` | gRPC | `9555` | `9555` |

Important:

```text
emailservice exposes Service port 5000, but the container listens on 8080.
This is expected in the default manifests.
```

## Layer 4: Environment Variables

Symptoms:

- App starts but cannot find dependencies.
- Frontend shows errors.
- Logs show empty address, wrong host, or wrong port.
- App tries to call `localhost`.

Commands:

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl exec -it <pod-name> -n <namespace> -- env
kubectl get deployment <deployment-name> -n <namespace> -o yaml
```

What to check:

- Required env vars exist.
- Env var names match the app source.
- Service addresses use Kubernetes/ECS service discovery names.
- Ports match Service ports, not only container ports.
- No local-only values are present.

Current app dependency env vars to watch:

| Service | Important env vars |
| --- | --- |
| `frontend` | `PRODUCT_CATALOG_SERVICE_ADDR`, `CURRENCY_SERVICE_ADDR`, `CART_SERVICE_ADDR`, `RECOMMENDATION_SERVICE_ADDR`, `SHIPPING_SERVICE_ADDR`, `CHECKOUT_SERVICE_ADDR`, `AD_SERVICE_ADDR` |
| `checkoutservice` | `PRODUCT_CATALOG_SERVICE_ADDR`, `SHIPPING_SERVICE_ADDR`, `PAYMENT_SERVICE_ADDR`, `EMAIL_SERVICE_ADDR`, `CURRENCY_SERVICE_ADDR`, `CART_SERVICE_ADDR` |
| `recommendationservice` | `PRODUCT_CATALOG_SERVICE_ADDR` |
| `cartservice` | `REDIS_ADDR` |
| `loadgenerator` | `FRONTEND_ADDR`, `USERS`, `RATE` |

Localhost rule:

```text
localhost inside a container means that same container or pod/task.
It does not mean another service.
```

## Layer 5: Secrets Mounted Or Injected

Symptoms:

- `CreateContainerConfigError`.
- Logs show missing credentials.
- App cannot authenticate to database, cache, API, or telemetry backend.
- External secret controller reports sync errors.

Commands:

```bash
kubectl get secret -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
kubectl get deployment <deployment-name> -n <namespace> -o yaml
```

What to check:

- Secret exists in the same namespace as the workload.
- Secret key names match manifest references.
- Secret sync is complete if using External Secrets.
- Runtime ServiceAccount has permission to read cloud secret manager values if used.
- Mounted secret path matches app expectation.

Do not print secret values into terminal output or logs during normal troubleshooting.

## Layer 6: Probes

Symptoms:

- Pod restarts repeatedly.
- Pod stays unready.
- Events show readiness or liveness failures.
- Load balancer has no healthy targets.

Commands:

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl get pod <pod-name> -n <namespace> -o yaml
```

Probe inventory for this app:

| Component | Probe type |
| --- | --- |
| `frontend` | HTTP `/_healthz` on container port `8080` |
| Most backend services | Native gRPC health probes |
| `redis-cart` | TCP socket on port `6379` |
| `loadgenerator` | Init container waits for frontend |

Things to check:

- Probe protocol matches app protocol.
- Probe checks container port, not wrong Service port.
- Initial delay is long enough.
- Timeout is not too aggressive.
- gRPC probes are supported by cluster version.
- Liveness probe is not killing slow-starting app too early.

Test frontend health:

```bash
kubectl port-forward svc/frontend 8080:80 -n <namespace>
curl -i http://localhost:8080/_healthz
```

Test gRPC health by port-forward:

```bash
kubectl port-forward svc/productcatalogservice 3550:3550 -n <namespace>
grpcurl -plaintext localhost:3550 grpc.health.v1.Health/Check
```

## Layer 7: DNS Resolution

Symptoms:

- Logs show `no such host`.
- App cannot find another service.
- Service names work locally but not in cluster.
- Fully qualified DNS works but short name does not.

Commands:

```bash
kubectl run dns-test \
  --rm -i --restart=Never \
  -n <namespace> \
  --image=busybox:1.36 \
  -- nslookup productcatalogservice
```

Test fully qualified name:

```bash
kubectl run dns-test \
  --rm -i --restart=Never \
  -n <namespace> \
  --image=busybox:1.36 \
  -- nslookup productcatalogservice.<namespace>.svc.cluster.local
```

What to check:

- Service exists.
- Service is in the expected namespace.
- App and target service are in the same namespace if using short DNS names.
- CoreDNS is healthy.
- Network policy does not block DNS egress.

CoreDNS check:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100
```

## Layer 8: Service Selector And Endpoints

Symptoms:

- Service exists but traffic fails.
- `kubectl get endpoints` shows no addresses.
- Load balancer has no healthy backend pods.

Commands:

```bash
kubectl get svc <service-name> -n <namespace> -o yaml
kubectl get endpoints <service-name> -n <namespace> -o yaml
kubectl get pods -n <namespace> --show-labels
```

What to compare:

```text
Service spec.selector == Pod metadata.labels
```

Example:

```bash
kubectl get svc frontend -n <namespace> -o jsonpath='{.spec.selector}'
kubectl get pods -n <namespace> -l app=frontend
```

Common causes:

| Cause | Result |
| --- | --- |
| Wrong selector label | Service has no endpoints. |
| Pods not ready | Endpoints may be missing or not ready. |
| Wrong targetPort | Endpoints exist, but traffic fails. |
| Deployment changed labels | Service no longer selects pods. |

## Layer 9: Network Policies Or Security Groups

Symptoms:

- DNS resolves but connections time out.
- One service can call another locally but not in cluster.
- Frontend reachable, but backend calls fail.
- Health checks fail only when network policies are enabled.

Kubernetes commands:

```bash
kubectl get networkpolicy -n <namespace>
kubectl describe networkpolicy <policy-name> -n <namespace>
```

Connectivity test:

```bash
kubectl run curl-test \
  --rm -i --restart=Never \
  -n <namespace> \
  --image=curlimages/curl \
  -- curl -i --connect-timeout 5 http://frontend:80/_healthz
```

What to check:

- Egress from source pod is allowed.
- Ingress to destination pod is allowed.
- DNS egress to kube-dns/CoreDNS is allowed.
- Service mesh authorization policies allow traffic.
- Load balancer/security group allows public traffic.

ECS checks:

- Task security group allows inbound from ALB or allowed callers.
- Task security group allows outbound to dependencies.
- ALB security group allows inbound public/private traffic.
- Subnet routing and NACLs allow traffic.
- Service Connect or Cloud Map configuration is correct.

## Layer 10: Downstream Dependencies

Symptoms:

- App starts and health passes, but user flow fails.
- Frontend page loads with errors.
- Checkout, cart, recommendation, or currency calls fail.
- Logs show connection refused, timeout, unavailable, or permission denied.

Dependency map:

| Service | Depends on |
| --- | --- |
| `frontend` | product catalog, currency, cart, recommendation, shipping, checkout, ad |
| `checkoutservice` | product catalog, shipping, payment, email, currency, cart |
| `recommendationservice` | product catalog |
| `cartservice` | Redis or external cart database |
| `loadgenerator` | frontend |

What to check:

- Target service pod/task is running.
- Target service has endpoints.
- Target service DNS resolves.
- Target port/protocol is correct.
- Credentials are valid.
- Network policy allows traffic.
- External dependency is healthy.

Start with frontend errors:

```bash
kubectl logs deployment/frontend -n <namespace> --tail=200
```

Then follow the dependency named in the logs.

## Layer 11: Resource Limits, OOM, And Throttling

Symptoms:

- `OOMKilled`.
- Container restarts under load.
- Latency rises without obvious app error.
- Java/Node/Python services become slow.
- CPU throttling causes readiness timeouts.

Commands:

```bash
kubectl top pods -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.status.containerStatuses[*].lastState}'
```

Check resource settings:

```bash
kubectl get deployment <deployment-name> -n <namespace> -o yaml
```

What to check:

- Memory limit is too low.
- CPU limit is causing throttling.
- Request is too low for realistic scheduling.
- Node is under pressure.
- HPA is missing or not scaling.
- JVM/.NET/Node/Python runtime needs memory tuning.

Useful events:

```bash
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

Look for:

- `OOMKilled`
- `Evicted`
- `FailedScheduling`
- `Back-off restarting failed container`
- `Readiness probe failed`
- `Liveness probe failed`

## Layer 12: Public Endpoint And Load Balancer

Symptoms:

- Internal frontend works, public endpoint fails.
- LoadBalancer IP stays pending.
- DNS resolves to wrong address.
- TLS certificate not ready.
- ALB/NLB target group unhealthy.

Kubernetes commands:

```bash
kubectl get svc frontend-external -n <namespace>
kubectl describe svc frontend-external -n <namespace>
kubectl get ingress -A
kubectl describe ingress <ingress-name> -n <namespace>
```

Test from outside:

```bash
curl -i http://<external-ip>/_healthz
curl -i http://<public-host>/
```

Test from inside:

```bash
kubectl run curl-test \
  --rm -i --restart=Never \
  -n <namespace> \
  --image=curlimages/curl \
  -- curl -i http://frontend:80/_healthz
```

Interpretation:

| Internal test | External test | Likely area |
| --- | --- | --- |
| Fails | Fails | App, Service, endpoints, or network policy. |
| Passes | Fails | Load balancer, ingress, DNS, TLS, firewall/security group. |
| Passes | Passes | Public path is healthy. |

## ECS Troubleshooting Commands

Service and task status:

```bash
aws ecs describe-services --cluster <cluster-name> --services <service-name>
aws ecs list-tasks --cluster <cluster-name> --service-name <service-name>
aws ecs describe-tasks --cluster <cluster-name> --tasks <task-arn>
```

Logs:

```bash
aws logs tail <log-group-name> --follow
```

Load balancer target health:

```bash
aws elbv2 describe-target-health --target-group-arn <target-group-arn>
```

What to check:

- Task stopped reason.
- Container exit code.
- Image pull errors.
- Secrets Manager or SSM permission errors.
- Execution role permissions.
- Task role permissions.
- Security group rules.
- Target group health check path and port.
- Desired count vs running count.
- Deployment circuit breaker rollback.

## Common Failure Matrix

| Symptom | Likely layer | First command |
| --- | --- | --- |
| Pod stuck `ImagePullBackOff` | Image/registry | `kubectl describe pod <pod>` |
| Pod stuck `CreateContainerConfigError` | Config/Secret | `kubectl describe pod <pod>` |
| Pod `CrashLoopBackOff` | App startup | `kubectl logs <pod> --previous` |
| Pod running but not ready | Probe/app port | `kubectl describe pod <pod>` |
| Service has no endpoints | Selector/readiness | `kubectl get endpoints <svc>` |
| DNS fails | Service/CoreDNS/network policy | `nslookup <service>` from debug pod |
| DNS works but connect times out | Network policy/security group | connectivity test from debug pod |
| Connection refused | Process not listening or wrong port | logs, port check, Service targetPort |
| Frontend page errors | Backend dependency | `kubectl logs deployment/frontend` |
| Cart errors | Redis/cart store | `kubectl logs deployment/cartservice` |
| Public URL fails only | Ingress/LB/DNS/TLS | `kubectl describe svc/ingress` |
| Restarts under load | Resource pressure | `kubectl top pods`, `kubectl describe pod` |
| No logs | Logging pipeline | `kubectl logs`, logging agent/CloudWatch |
| No metrics | Metrics pipeline | `kubectl top pods`, Prometheus/collector |
| No traces | Telemetry config | collector logs, env vars |

## Debug Pod Patterns

HTTP test:

```bash
kubectl run curl-test \
  --rm -i --restart=Never \
  -n <namespace> \
  --image=curlimages/curl \
  -- curl -i http://frontend:80/_healthz
```

DNS test:

```bash
kubectl run dns-test \
  --rm -i --restart=Never \
  -n <namespace> \
  --image=busybox:1.36 \
  -- nslookup frontend
```

Shell test:

```bash
kubectl run debug-shell \
  --rm -it --restart=Never \
  -n <namespace> \
  --image=busybox:1.36 \
  -- sh
```

If network policies are enabled, debug pods may not have the same labels as app pods. A debug pod may be blocked even when real app traffic is allowed, or allowed when real app traffic is blocked. Match labels when needed.

## Incident Notes Template

Use this while troubleshooting:

| Field | Value |
| --- | --- |
| Environment |  |
| Namespace/cluster |  |
| Service affected |  |
| User-facing impact |  |
| First detected by |  |
| Recent deployment/change |  |
| Current image tag/digest |  |
| Previous known-good image |  |
| Failing symptom |  |
| First failing layer |  |
| Logs/events link |  |
| Owner |  |
| Decision: fix forward or rollback |  |

## Troubleshooting Go/No-Go

Continue investigating if:

- [ ] Impact is limited to dev/staging.
- [ ] Root cause is visible.
- [ ] Fix is low risk.
- [ ] Rollback target is available.
- [ ] Service is stable enough to observe safely.

Roll back if:

- [ ] Production user impact is active.
- [ ] Error rate or latency is unacceptable.
- [ ] Critical service is crash looping.
- [ ] Data path is broken.
- [ ] Root cause is unknown and rollout caused the issue.
- [ ] Fix forward is riskier than rollback.

Escalate if:

- [ ] Multiple critical services are affected.
- [ ] Data loss or corruption is suspected.
- [ ] Security issue is suspected.
- [ ] Cloud provider/network/storage dependency is failing.
- [ ] You need app team context to interpret business logic failure.

## Output Checklist

Before moving to final go/no-go, produce:

- [ ] Failing layer identified.
- [ ] Evidence gathered from logs/events/status.
- [ ] Impacted service identified.
- [ ] Dependency path checked.
- [ ] Rollback option confirmed.
- [ ] Fix-forward option identified if appropriate.
- [ ] Owner assigned.
- [ ] Decision recorded.

## Ready For Next Checklist

After troubleshooting guidance is defined, move to:

```text
18. Final Go/No-Go Checklist
```

In that stage, decide whether the app is truly ready for production deployment based on inventory, containerization, registry, config, discovery, manifests, health, state, networking, security, observability, resources, deployment strategy, CI/CD, validation, and troubleshooting readiness.
