# Platform Deployment Readiness Checklist

Use this checklist when a product team hands over an application that must be containerized, instrumented, and deployed to Kubernetes, ECS, or another container platform.

The goal is to answer three questions:

1. Can I build and package every service?
2. Can the services run correctly in a cluster?
3. Can I operate, observe, secure, and recover the application after deployment?

## 1. Application Inventory

Create a complete service inventory before touching deployment manifests.

Checklist:

- [ ] List every service/application component.
- [ ] Identify the language/runtime for each service.
- [ ] Identify the application entrypoint.
- [ ] Identify the startup command.
- [ ] Identify exposed ports.
- [ ] Identify protocol per port: HTTP, gRPC, TCP, UDP.
- [ ] Identify internal service dependencies.
- [ ] Identify external dependencies: database, cache, queue, object storage, third-party API.
- [ ] Identify background workers, cron jobs, or one-off migration jobs.
- [ ] Identify which service receives public traffic.
- [ ] Identify which services must stay private.

Service inventory table:

| Service | Runtime | Entrypoint | Port | Protocol | Depends on | Public? |
| --- | --- | --- | --- | --- | --- | --- |
|  |  |  |  |  |  |  |

## 2. Containerization

Each service should be independently buildable and runnable as a container.

Checklist:

- [ ] Confirm each service has a Dockerfile.
- [ ] Confirm the Dockerfile builds from a clean checkout.
- [ ] Confirm dependencies are installed during build.
- [ ] Confirm the correct files are copied into the image.
- [ ] Confirm the container starts the correct process.
- [ ] Confirm the container exposes or documents the correct port.
- [ ] Confirm config comes from environment variables or mounted config, not hardcoded values.
- [ ] Confirm the image does not depend on local files outside the build context.
- [ ] Confirm the image can run without Docker Compose.
- [ ] Confirm the image runs as non-root where possible.
- [ ] Confirm unnecessary package managers/build tools are not present in runtime images.
- [ ] Confirm image size is reasonable.

Common Dockerfile patterns:

| Runtime | Packaging pattern |
| --- | --- |
| Go | Build static binary, copy binary into small runtime/distroless image |
| Node.js | Install with `npm ci`, copy app, run `node` or package script |
| Python | Install `requirements.txt`, copy app, run Python entrypoint or WSGI/ASGI server |
| Java | Build with Maven/Gradle, run JAR or generated start script |
| .NET | `dotnet publish`, run published app in runtime image |

Basic image commands:

```sh
docker build -t <service-name>:<tag> ./path/to/service
docker run --rm -p <host-port>:<container-port> <service-name>:<tag>
```

## 3. Image Registry And Tagging

The cluster must be able to pull every image.

Checklist:

- [ ] Choose target registry: ECR, Artifact Registry, Docker Hub, ACR, private registry.
- [ ] Define image naming convention.
- [ ] Define tag strategy: Git SHA, semantic version, build number, environment tag.
- [ ] Build images for the correct CPU architecture: `amd64`, `arm64`, or multi-arch.
- [ ] Push images to registry.
- [ ] Confirm cluster nodes/tasks can pull images.
- [ ] Configure image pull secrets if required.
- [ ] Enable image vulnerability scanning if available.
- [ ] Avoid `latest` for production deployments.

Example:

```sh
docker build -t <registry>/<app>/<service>:<git-sha> ./src/<service>
docker push <registry>/<app>/<service>:<git-sha>
```

## 4. Runtime Configuration

Document every variable needed at runtime.

Checklist:

- [ ] List required environment variables.
- [ ] List optional environment variables.
- [ ] Identify default values.
- [ ] Identify secrets.
- [ ] Identify config that differs per environment.
- [ ] Identify service addresses.
- [ ] Identify feature flags.
- [ ] Identify telemetry configuration.
- [ ] Confirm no environment-specific value is hardcoded in source code.

Configuration table:

| Name | Required? | Example | Source | Secret? | Notes |
| --- | --- | --- | --- | --- | --- |
| `PORT` | Yes | `8080` | env | No | Service listen port |
|  |  |  |  |  |  |

Recommended config sources:

- Kubernetes: `ConfigMap`, `Secret`, External Secrets, sealed secrets, cloud secret manager integration.
- ECS: task definition environment variables, AWS Secrets Manager, SSM Parameter Store.

## 5. Service Discovery

Cluster networking is different from local development.

Checklist:

- [ ] Replace `localhost` dependencies with service discovery names.
- [ ] Confirm internal service DNS names.
- [ ] Confirm ports match container, service, and client config.
- [ ] Confirm public services and private services are separated.
- [ ] Confirm services can resolve each other by DNS.
- [ ] Confirm protocol expectations: HTTP vs gRPC vs raw TCP.

Important rule:

```text
localhost inside a container means the same container or pod/task.
It does not mean another service.
```

Kubernetes examples:

```text
service-name:8080
service-name.namespace.svc.cluster.local:8080
```

ECS examples:

```text
Cloud Map service discovery name
Internal load balancer DNS name
Service Connect endpoint
```

## 6. Kubernetes Or ECS Manifests

Prepare the deployment objects for the target platform.

Kubernetes checklist:

- [ ] Namespace.
- [ ] Deployment or StatefulSet.
- [ ] Service.
- [ ] ConfigMap.
- [ ] Secret.
- [ ] ServiceAccount.
- [ ] Ingress, Gateway, or LoadBalancer service.
- [ ] PersistentVolumeClaim if needed.
- [ ] HorizontalPodAutoscaler if needed.
- [ ] PodDisruptionBudget if needed.
- [ ] NetworkPolicy if required.
- [ ] Resource requests and limits.
- [ ] Readiness and liveness probes.
- [ ] Security context.

ECS checklist:

- [ ] Cluster.
- [ ] Task definition.
- [ ] Container definitions.
- [ ] Service definition.
- [ ] Target group.
- [ ] Load balancer listener/rules.
- [ ] Security groups.
- [ ] IAM task role.
- [ ] IAM execution role.
- [ ] CloudWatch logs.
- [ ] Service discovery or Service Connect.
- [ ] Desired count.
- [ ] CPU and memory settings.
- [ ] Health check grace period.

## 7. Health Checks

Health checks must reflect whether the service can receive traffic.

Checklist:

- [ ] Identify health endpoint or health RPC.
- [ ] Configure readiness checks.
- [ ] Configure liveness checks.
- [ ] Configure startup checks for slow-starting services if needed.
- [ ] Confirm health checks use the correct protocol.
- [ ] Confirm health checks do not require authentication unless intentionally handled.
- [ ] Confirm health checks do not depend on fragile downstream services unless appropriate.

Common probe types:

| Protocol | Health check style |
| --- | --- |
| HTTP | `GET /health`, `/ready`, `/_healthz` |
| gRPC | gRPC health checking protocol |
| TCP | TCP socket check |
| Worker | Custom command or process check |

## 8. Storage And State

Stateful components need special attention.

Checklist:

- [ ] Identify databases, caches, queues, and persistent volumes.
- [ ] Decide managed service vs in-cluster service.
- [ ] Confirm persistence requirements.
- [ ] Confirm backup and restore plan.
- [ ] Confirm migration strategy.
- [ ] Confirm data retention requirements.
- [ ] Confirm storage class.
- [ ] Confirm volume size.
- [ ] Confirm access mode.
- [ ] Confirm what happens during pod/task restart.

Questions to answer:

- Can this data be lost?
- Does the app need migrations before deploy?
- Is this cache or source-of-truth storage?
- What is the recovery process?

## 9. Networking And Exposure

Only expose what must be exposed.

Checklist:

- [ ] Identify public entrypoint.
- [ ] Keep backend services private.
- [ ] Configure ingress, gateway, load balancer, or API gateway.
- [ ] Configure TLS certificates.
- [ ] Configure DNS.
- [ ] Configure allowed origins/CORS if applicable.
- [ ] Configure network policies or security groups.
- [ ] Confirm service mesh requirements if applicable.
- [ ] Confirm timeout settings for HTTP/gRPC.
- [ ] Confirm load balancer health checks.

Kubernetes exposure options:

- `ClusterIP` for internal services.
- `LoadBalancer` for simple public exposure.
- `Ingress` or Gateway API for HTTP routing.
- Service mesh gateway for mesh-managed ingress.

ECS exposure options:

- Application Load Balancer.
- Network Load Balancer.
- Internal load balancer.
- API Gateway in front of ALB/NLB.

## 10. Security

Security should be built into the deployment, not added later.

Checklist:

- [ ] Run containers as non-root.
- [ ] Drop Linux capabilities.
- [ ] Disable privilege escalation.
- [ ] Use read-only root filesystem where possible.
- [ ] Avoid mounting Docker socket.
- [ ] Avoid privileged containers.
- [ ] Store secrets outside Git.
- [ ] Use least-privilege IAM/RBAC.
- [ ] Scan images for vulnerabilities.
- [ ] Pin base image versions.
- [ ] Avoid unnecessary tools in runtime images.
- [ ] Configure network restrictions.
- [ ] Configure TLS for public traffic.
- [ ] Review supply chain requirements.

Kubernetes examples:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
```

## 11. Observability

You should be able to explain what is happening after deployment.

Checklist:

- [ ] Logs are written to stdout/stderr.
- [ ] Logs are structured where possible.
- [ ] Metrics are exposed or collected.
- [ ] Traces are emitted for request flows if supported.
- [ ] Dashboards exist for service health.
- [ ] Alerts exist for user-impacting failures.
- [ ] Alerts exist for infrastructure failures.
- [ ] Error rate is visible.
- [ ] Latency is visible.
- [ ] Saturation is visible: CPU, memory, disk, network.
- [ ] Pod/task restarts are visible.
- [ ] Deployment events are visible.

Minimum useful alerts:

- High error rate.
- High latency.
- Pod/task crash looping.
- Readiness failures.
- CPU or memory saturation.
- Disk saturation for stateful services.
- External dependency failures.

## 12. Resource Management

Right-size workloads before production.

Checklist:

- [ ] Set CPU requests.
- [ ] Set memory requests.
- [ ] Set CPU limits if appropriate.
- [ ] Set memory limits.
- [ ] Observe actual usage under test.
- [ ] Configure HPA or autoscaling if needed.
- [ ] Configure min/max replicas.
- [ ] Configure PodDisruptionBudget for critical services.
- [ ] Confirm node capacity.
- [ ] Confirm cluster autoscaler settings.

Kubernetes example:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

## 13. Deployment Strategy

Define how changes move safely through environments.

Checklist:

- [ ] Define environments: dev, staging, production.
- [ ] Define promotion process.
- [ ] Define deployment strategy: rolling, blue/green, canary.
- [ ] Define rollback command/process.
- [ ] Define image tag used for rollback.
- [ ] Define database migration process.
- [ ] Define release verification steps.
- [ ] Define who approves production deploys.

Kubernetes rollout commands:

```sh
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name>
```

## 14. CI/CD Pipeline

The pipeline should produce repeatable deployments.

Checklist:

- [ ] Run tests.
- [ ] Build images.
- [ ] Scan images.
- [ ] Push images.
- [ ] Render manifests.
- [ ] Validate manifests.
- [ ] Deploy to dev.
- [ ] Run smoke tests.
- [ ] Promote to staging/prod.
- [ ] Record image tags and deployment metadata.
- [ ] Support rollback.

Pipeline stages:

```text
source
  -> test
  -> build image
  -> scan image
  -> push image
  -> render manifests
  -> deploy
  -> smoke test
  -> promote
```

## 15. Pre-Deployment Validation

Validate before touching a shared cluster.

Checklist:

- [ ] Build every image locally or in CI.
- [ ] Run every container locally with required env vars.
- [ ] Validate manifests with `kubectl apply --dry-run=server` or equivalent.
- [ ] Confirm namespace exists.
- [ ] Confirm registry pull access.
- [ ] Confirm secrets/config exist.
- [ ] Confirm ingress/load balancer prerequisites exist.
- [ ] Confirm storage class exists.
- [ ] Confirm resource quotas are not exceeded.
- [ ] Confirm cluster version supports required features.

Useful commands:

```sh
kubectl apply --dry-run=server -f <manifest.yaml>
kubectl diff -f <manifest.yaml>
kubectl get namespace
kubectl get storageclass
kubectl get resourcequota -A
```

## 16. Post-Deployment Validation

After deployment, prove the app works from the cluster outward.

Checklist:

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

Kubernetes commands:

```sh
kubectl get pods
kubectl get svc
kubectl get endpoints
kubectl describe pod <pod-name>
kubectl logs deployment/<deployment-name>
kubectl rollout status deployment/<deployment-name>
```

Smoke test examples:

```sh
curl -i http://<public-url>/health
curl -i http://<public-url>/
```

## 17. Troubleshooting Checklist

When something fails, isolate by layer.

Checklist:

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

Common commands:

```sh
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
kubectl exec -it <pod-name> -- sh
kubectl get events --sort-by=.lastTimestamp
kubectl get endpoints <service-name>
```

## 18. Final Go/No-Go Checklist

Use this before declaring the application cluster-ready.

- [ ] Every service is inventoried.
- [ ] Every service has a container image.
- [ ] Every image builds successfully.
- [ ] Every image is in an accessible registry.
- [ ] Runtime config is documented.
- [ ] Secrets are handled securely.
- [ ] Internal service discovery is configured.
- [ ] Public entrypoint is defined.
- [ ] Backends are private.
- [ ] Health checks are configured.
- [ ] Resource requests and limits are configured.
- [ ] Security context is configured.
- [ ] Stateful dependencies are understood.
- [ ] Observability is configured.
- [ ] Rollback process is documented.
- [ ] Smoke tests pass.
- [ ] Operational owner knows how to debug failures.

## Quick Reusable Summary

```text
[ ] Inventory services, ports, protocols, and dependencies
[ ] Confirm or create Dockerfile per service
[ ] Build and run images locally
[ ] Push images to registry
[ ] Configure env vars, ConfigMaps, and Secrets
[ ] Create Deployments/Services or ECS task/service definitions
[ ] Configure service discovery
[ ] Configure storage/stateful dependencies
[ ] Add readiness/liveness health checks
[ ] Set requests/limits or CPU/memory reservations
[ ] Apply security hardening
[ ] Expose only the public entrypoint
[ ] Configure TLS/DNS/ingress/load balancer
[ ] Configure logs, metrics, traces, dashboards, alerts
[ ] Validate manifests/task definitions
[ ] Deploy to dev/staging
[ ] Run smoke tests
[ ] Verify rollback process
```

