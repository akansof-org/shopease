# Cloud Handover Checklist: 12. Resource Management

Scenario: the development team has handed this application to the platform engineering team. We are preparing it for cloud infrastructure, instrumentation, and deployment to Kubernetes or ECS.

This note covers the twelfth checklist item only: **Resource Management**.

## Objective

Right-size workloads before production.

This stage answers:

- Are CPU requests set?
- Are memory requests set?
- Are CPU and memory limits set appropriately?
- Have we observed actual usage under realistic traffic?
- Do services need autoscaling?
- How many replicas should each service run?
- Do critical services need PodDisruptionBudgets?
- Does the cluster have enough capacity?
- Is cluster autoscaling configured?

## Checklist

- [x] Set CPU requests.
- [x] Set memory requests.
- [x] Set CPU limits if appropriate.
- [x] Set memory limits.
- [ ] Observe actual usage under test.
- [ ] Configure HPA or autoscaling if needed.
- [ ] Configure min/max replicas.
- [ ] Configure PodDisruptionBudget for critical services.
- [ ] Confirm node capacity.
- [ ] Confirm cluster autoscaler settings.

Note:

- Default manifests include resource requests and limits.
- Current values should be treated as demo baselines, not production sizing.
- No default HPAs or PodDisruptionBudgets are present.

## Kubernetes Resource Example

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

## Current Resource Inventory

| Component | CPU request | Memory request | CPU limit | Memory limit | Notes |
| --- | ---: | ---: | ---: | ---: | --- |
| `frontend` | `100m` | `64Mi` | `200m` | `128Mi` | Public HTTP frontend. |
| `productcatalogservice` | `100m` | `64Mi` | `200m` | `128Mi` | Product catalog gRPC service. |
| `currencyservice` | `100m` | `64Mi` | `200m` | `128Mi` | Currency gRPC service. |
| `cartservice` | `200m` | `64Mi` | `300m` | `128Mi` | Cart gRPC service. |
| `redis-cart` | `70m` | `200Mi` | `125m` | `256Mi` | In-cluster Redis. |
| `shippingservice` | `100m` | `64Mi` | `200m` | `128Mi` | Shipping gRPC service. |
| `paymentservice` | `100m` | `64Mi` | `200m` | `128Mi` | Payment gRPC service. |
| `emailservice` | `100m` | `64Mi` | `200m` | `128Mi` | Email gRPC service. |
| `checkoutservice` | `100m` | `64Mi` | `200m` | `128Mi` | Checkout orchestration service. |
| `recommendationservice` | `100m` | `220Mi` | `200m` | `450Mi` | Higher memory than most services. |
| `adservice` | `200m` | `180Mi` | `300m` | `300Mi` | Java service; higher baseline. |
| `loadgenerator` | `300m` | `256Mi` | `500m` | `512Mi` | Optional load generator. |

## Resource Baseline Interpretation

Current values are enough to start and demo the app, but they are not proof of production readiness.

Platform interpretation:

```text
These are initial scheduling hints and guardrails.
They must be validated with real traffic, load tests, and observed usage.
```

Services needing extra attention:

| Service | Why |
| --- | --- |
| `frontend` | Public traffic entrypoint; user-visible latency and saturation matter. |
| `checkoutservice` | Fan-out service; depends on many backends. |
| `cartservice` | Stateful dependency path; Redis latency affects cart flow. |
| `currencyservice` | Often high-QPS in this app. |
| `recommendationservice` | Higher memory allocation; Python service. |
| `adservice` | Java service; higher memory and startup profile. |
| `redis-cart` | In-cluster state; memory limit directly affects cart storage capacity. |

## Requests vs Limits

### Requests

Requests are used by Kubernetes scheduler.

```text
requests = capacity reserved/scheduled for the pod
```

If requests are too low:

- pods may be packed too tightly
- CPU/memory contention increases
- latency becomes noisy

If requests are too high:

- cluster capacity is wasted
- pods may fail to schedule
- autoscaler may add nodes unnecessarily

### Limits

Limits cap resource usage.

```text
limits = maximum allowed resource usage
```

If CPU limit is too low:

- container gets throttled
- latency can increase

If memory limit is too low:

- container can be OOMKilled
- pod restarts

Platform note:

```text
Memory limits are usually important.
CPU limits require more care because throttling can hurt latency-sensitive services.
```

## Current Replica Configuration

Default manifests mostly do not set explicit `replicas`.

Kubernetes default:

```text
replicas: 1
```

Explicit default:

| Component | Explicit replicas |
| --- | ---: |
| `loadgenerator` | `1` |

Everything else implicitly runs one replica by default unless modified.

Production consideration:

```text
Single-replica services are not highly available.
For production-like environments, run at least 2 replicas for critical stateless services.
```

Candidates for multiple replicas:

- `frontend`
- `checkoutservice`
- `productcatalogservice`
- `currencyservice`
- `cartservice`
- `shippingservice`
- `paymentservice`
- `emailservice`
- `recommendationservice`
- `adservice`

Redis requires a separate HA design if using in-cluster Redis.

## Autoscaling

No HorizontalPodAutoscaler is present by default.

Autoscaling options:

| Platform | Autoscaling option |
| --- | --- |
| Kubernetes | HorizontalPodAutoscaler |
| Kubernetes | KEDA for event/custom metric scaling |
| Kubernetes | Vertical Pod Autoscaler for recommendations or controlled resizing |
| ECS | Service autoscaling |
| ECS | Target tracking scaling on CPU/memory/ALB request count/custom metrics |

Recommended starting point:

- Add HPA first for `frontend`.
- Consider HPA for `checkoutservice`, `currencyservice`, and `productcatalogservice`.
- Avoid autoscaling Redis as a simple Deployment unless using a proper Redis HA/operator/managed service.

Example HPA:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

## Min/Max Replica Decisions

Fill this table after load testing and availability planning.

| Service | Min replicas | Max replicas | Scaling metric | Notes |
| --- | ---: | ---: | --- | --- |
| `frontend` | `2` recommended | TBD | CPU, HTTP RPS, latency | Public entrypoint. |
| `checkoutservice` | `2` recommended | TBD | CPU, gRPC RPS, latency | Critical checkout flow. |
| `productcatalogservice` | `2` recommended | TBD | CPU, gRPC RPS | Frequently called. |
| `currencyservice` | `2` recommended | TBD | CPU, gRPC RPS | Potentially high-QPS. |
| `cartservice` | `2` recommended | TBD | CPU, gRPC RPS | Depends on cart backend. |
| `recommendationservice` | `2` if critical | TBD | CPU/memory, gRPC RPS | Can degrade product page recommendations. |
| `adservice` | `2` if critical | TBD | CPU/memory, gRPC RPS | Java memory baseline. |
| `redis-cart` | n/a | n/a | n/a | Use managed Redis or HA design. |

## PodDisruptionBudget

No PodDisruptionBudget is present by default.

PDBs help keep enough replicas available during voluntary disruptions such as:

- node drains
- cluster upgrades
- planned maintenance

Add PDBs after increasing replicas above one.

Example:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: frontend
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: frontend
```

Recommended PDB candidates:

- `frontend`
- `checkoutservice`
- `cartservice`
- `productcatalogservice`
- `currencyservice`
- `paymentservice`
- `shippingservice`
- `emailservice`

## Load Testing And Observation

The repo includes `loadgenerator`, which can help produce traffic.

Default loadgenerator config:

| Env var | Default |
| --- | --- |
| `USERS` | `10` |
| `RATE` | `1` |
| `FRONTEND_ADDR` | `frontend:80` in Kubernetes |

Use load testing to observe:

- CPU usage
- memory usage
- latency
- error rate
- pod restarts
- HPA behavior if enabled
- Redis memory usage
- checkout flow stability

Kubernetes commands:

```sh
kubectl top pods
kubectl top nodes
kubectl get hpa
kubectl describe hpa <name>
kubectl get events --sort-by=.lastTimestamp
```

Platform note:

```text
Do not tune requests/limits only from idle metrics.
Tune from realistic traffic and failure scenarios.
```

## Cluster Capacity

Total default requested capacity for core app is approximately:

```text
CPU requests: about 1.37 vCPU including redis-cart, excluding loadgenerator
Memory requests: about 1.12 GiB including redis-cart, excluding loadgenerator
```

Including `loadgenerator`:

```text
CPU requests: about 1.67 vCPU
Memory requests: about 1.37 GiB
```

These are only requests, not limits.

Capacity checklist:

- [ ] Confirm total CPU requests fit available nodes.
- [ ] Confirm total memory requests fit available nodes.
- [ ] Confirm room for system pods.
- [ ] Confirm room for observability components.
- [ ] Confirm room for service mesh sidecars if enabled.
- [ ] Confirm room for surge during rolling updates.
- [ ] Confirm room for HPA scale-out.
- [ ] Confirm node architecture matches images.

If using service mesh:

```text
Add sidecar CPU/memory overhead to every meshed pod.
```

## Cluster Autoscaler

Cluster autoscaling is not configured in app manifests.

Platform decision:

- GKE Autopilot handles node provisioning automatically.
- GKE Standard requires cluster autoscaler/node pool config.
- EKS/AKS require cluster autoscaler or Karpenter/equivalent.
- ECS/Fargate abstracts node capacity.
- ECS on EC2 needs capacity providers/autoscaling.

Checklist:

- [ ] Confirm cluster autoscaler or equivalent is enabled.
- [ ] Confirm node pools can scale to max HPA demand.
- [ ] Confirm resource quotas do not block scheduling.
- [ ] Confirm PodDisruptionBudgets do not block node drains unexpectedly.
- [ ] Confirm scale-up latency is acceptable.

## ECS Resource Translation

For ECS, map Kubernetes resource decisions to:

| Kubernetes concept | ECS equivalent |
| --- | --- |
| CPU request/limit | Task/container CPU units |
| Memory request/limit | Task/container memory reservation/hard limit |
| replicas | ECS service desired count |
| HPA | ECS service autoscaling |
| PDB | Deployment minimum healthy percent/maximum percent |
| cluster autoscaler | Capacity providers, Fargate, EC2 autoscaling |

ECS checklist:

- [ ] Set task CPU.
- [ ] Set task memory.
- [ ] Set container memory limits/reservations.
- [ ] Set desired count per service.
- [ ] Configure service autoscaling.
- [ ] Configure min/max task count.
- [ ] Confirm ALB target group can handle scale-out.
- [ ] Confirm capacity provider/Fargate capacity.

## Resource Management Risks

| Area | Risk | Follow-up |
| --- | --- | --- |
| Demo-sized resources | Defaults may be too low for production | Load test and tune. |
| Single replica | Most services run one replica by default | Set replicas/HPA for critical services. |
| No HPA | Traffic spikes may overload services | Add autoscaling where useful. |
| No PDB | Voluntary disruptions can take services down | Add PDBs after replica count > 1. |
| CPU limits | Low CPU limits can throttle services | Review after latency testing. |
| Memory limits | Low memory limits cause OOMKills | Watch memory and restart metrics. |
| Redis memory | Redis memory limit constrains cart capacity | Use managed Redis or tune memory/eviction policy. |
| Service mesh overhead | Sidecars increase resource needs | Add sidecar overhead to sizing. |
| Observability overhead | Collector/profiler/tracing add resource usage | Re-test after enabling telemetry. |

## Resource Management Output

At the end of this stage, platform engineering should have:

- [x] Current requests and limits documented.
- [x] Current replica behavior documented.
- [x] Autoscaling gaps documented.
- [x] PDB gaps documented.
- [x] Cluster capacity considerations documented.
- [x] ECS resource translation documented.
- [ ] Load test results collected.
- [ ] Requests/limits tuned from observed usage.
- [ ] Replica counts finalized.
- [ ] HPA/autoscaling configured if needed.
- [ ] PDBs configured for critical services.
- [ ] Cluster autoscaler/capacity provider confirmed.

## Ready For Next Checklist Item

The next stage is:

```text
13. Deployment Strategy
```

In that stage, define environments, promotion process, rollout strategy, rollback process, migration handling, release verification, and production approval flow.

