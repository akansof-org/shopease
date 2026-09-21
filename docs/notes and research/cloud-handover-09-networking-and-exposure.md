# Cloud Handover Checklist: 9. Networking And Exposure

Scenario: the development team has handed this application to the platform engineering team. We are preparing it for cloud infrastructure, instrumentation, and deployment to Kubernetes or ECS.

This note covers the ninth checklist item only: **Networking And Exposure**.

## Objective

Only expose what must be exposed.

This stage answers:

- What is the public entrypoint?
- Which services must remain private?
- Should the app use a LoadBalancer, Ingress, Gateway, API Gateway, or service mesh gateway?
- How will TLS and DNS be configured?
- Are network policies/security groups required?
- Are there CORS requirements?
- Are HTTP/gRPC timeouts understood?
- Do load balancer health checks match the app?

## Checklist

- [x] Identify public entrypoint.
- [x] Keep backend services private.
- [x] Configure ingress, gateway, load balancer, or API gateway.
- [ ] Configure TLS certificates.
- [ ] Configure DNS.
- [ ] Configure allowed origins/CORS if applicable.
- [ ] Configure network policies or security groups.
- [ ] Confirm service mesh requirements if applicable.
- [ ] Confirm timeout settings for HTTP/gRPC.
- [x] Confirm load balancer health checks.

Note:

- The default Kubernetes manifests expose `frontend` publicly through a `LoadBalancer`.
- TLS, DNS, CORS, production ingress rules, and network policy enforcement are platform decisions not fully configured in the default manifests.

## Kubernetes Exposure Options

| Option | Use case |
| --- | --- |
| `ClusterIP` | Internal-only services |
| `LoadBalancer` | Simple public exposure |
| `Ingress` | HTTP routing through ingress controller |
| Gateway API | Modern Kubernetes traffic routing |
| Service mesh gateway | Mesh-managed ingress and traffic policy |

## ECS Exposure Options

| Option | Use case |
| --- | --- |
| Application Load Balancer | Public HTTP/HTTPS frontend |
| Network Load Balancer | TCP/gRPC or high-performance network routing |
| Internal load balancer | Private service-to-service or private app access |
| API Gateway in front of ALB/NLB | API management, auth, throttling, edge controls |

## Current App Exposure Model

Default public path:

```text
User Browser
  -> cloud LoadBalancer
  -> Kubernetes Service frontend-external:80
  -> frontend pod container port 8080
```

Internal frontend path:

```text
Cluster workloads
  -> Kubernetes Service frontend:80
  -> frontend pod container port 8080
```

Backend path:

```text
frontend
  -> private ClusterIP services
```

## Public Entrypoint

| Public service | Kubernetes type | Port | Target port | Protocol | Notes |
| --- | --- | ---: | ---: | --- | --- |
| `frontend-external` | `LoadBalancer` | `80` | `8080` | HTTP | Default public entrypoint. |

The app also has an internal frontend service:

| Internal service | Kubernetes type | Port | Target port | Protocol | Notes |
| --- | --- | ---: | ---: | --- | --- |
| `frontend` | `ClusterIP` | `80` | `8080` | HTTP | Used by internal clients like `loadgenerator`. |

Platform decision:

```text
Decide whether frontend-external LoadBalancer is acceptable, or whether to replace it with Ingress, Gateway API, or a service mesh gateway.
```

## Private Services

All backend services should remain private.

| Service | Type | Port | Protocol |
| --- | --- | ---: | --- |
| `productcatalogservice` | ClusterIP | `3550` | gRPC |
| `currencyservice` | ClusterIP | `7000` | gRPC |
| `cartservice` | ClusterIP | `7070` | gRPC |
| `redis-cart` | ClusterIP | `6379` | TCP |
| `shippingservice` | ClusterIP | `50051` | gRPC |
| `paymentservice` | ClusterIP | `50051` | gRPC |
| `emailservice` | ClusterIP | `5000` | gRPC |
| `checkoutservice` | ClusterIP | `5050` | gRPC |
| `recommendationservice` | ClusterIP | `8080` | gRPC |
| `adservice` | ClusterIP | `9555` | gRPC |

Rule:

```text
Do not expose backend services through public LoadBalancers, public ingresses, or public ECS target groups.
```

## Default LoadBalancer

The default frontend external service is:

```yaml
kind: Service
metadata:
  name: frontend-external
spec:
  type: LoadBalancer
  ports:
  - name: http
    port: 80
    targetPort: 8080
```

Pros:

- Simple.
- Works quickly on cloud Kubernetes clusters.
- Good for demos and initial smoke testing.

Cons:

- No TLS by default.
- No DNS by default.
- Limited HTTP routing control.
- Public exposure may not meet production security requirements.
- Cloud load balancer settings are mostly implicit.

## Non-Public Frontend Option

The repo includes a Kustomize component to remove public exposure:

```text
kustomize/components/non-public-frontend
```

Enable it:

```sh
cd kustomize
kustomize edit add component components/non-public-frontend
kubectl apply -k .
```

Use this when:

- app should be accessible only through port-forwarding
- app is behind another gateway
- app is internal-only
- exposure is handled by service mesh or platform ingress

## Ingress Or Gateway Option

For production-style Kubernetes exposure, prefer an explicit ingress or gateway model.

Typical design:

```text
DNS name
  -> cloud load balancer
  -> Ingress/Gateway
  -> frontend Service
  -> frontend pods
```

Required decisions:

- hostname
- TLS certificate source
- ingress controller or gateway class
- allowed paths
- HTTP to HTTPS redirect
- load balancer type: public or internal
- WAF/security policy
- access logs
- timeout settings

Example conceptual route:

```text
https://shop.example.com/
  -> frontend:80
```

## Service Mesh Option

The repo includes an Istio service mesh component:

```text
kustomize/components/service-mesh-istio
```

This component:

- removes the default `frontend-external` service
- creates Gateway API resources
- routes traffic to `frontend`
- includes mesh-related resources

Gateway shape:

```text
Gateway istio-gateway
  listener: HTTP port 80
HTTPRoute frontend-route
  backendRef: frontend:80
```

Enable:

```sh
cd kustomize
kustomize edit add component components/service-mesh-istio
kubectl apply -k .
```

Platform notes:

- Do not combine `service-mesh-istio` with `non-public-frontend`; the repo notes they both patch public frontend exposure.
- Namespace sidecar injection must be configured if using Istio sidecars.
- Probe rewrite annotation already exists on frontend and loadgenerator manifests.
- Confirm gRPC traffic behavior and timeouts through the mesh.
- Confirm egress rules for Google APIs if optional cloud features or telemetry are enabled.

## TLS Certificates

Default manifests:

```text
No TLS certificate is configured.
```

Production options:

- cloud managed certificate
- cert-manager
- ACM certificate for AWS ALB/API Gateway
- Azure managed certificate
- service mesh certificate management

Checklist:

- [ ] Choose certificate authority/source.
- [ ] Define hostname.
- [ ] Configure HTTPS listener.
- [ ] Redirect HTTP to HTTPS if required.
- [ ] Define renewal process.
- [ ] Confirm cert ownership and expiration alerting.

## DNS

Default manifests:

```text
No DNS record is configured.
```

For demos:

```sh
kubectl get service frontend-external
```

Use the external IP directly.

For production:

- create DNS name
- point DNS to ingress/load balancer
- manage TTL
- validate certificate hostname
- document ownership

Example:

```text
shop.example.com -> frontend ingress/load balancer
```

## CORS And Allowed Origins

Default app:

- Browser traffic is served by `frontend`.
- Backend gRPC services are not browser-facing.
- No explicit CORS configuration is apparent in the default manifests.

Platform decision:

```text
If only the frontend serves browser traffic from the same origin, CORS may not be needed.
If APIs are exposed separately or frontend assets are served from another domain, define allowed origins explicitly.
```

Checklist:

- [ ] Identify browser-facing APIs.
- [ ] Identify frontend domain.
- [ ] Identify API domain if different.
- [ ] Configure allowed origins if needed.
- [ ] Avoid wildcard origins for authenticated production traffic.

## Network Policies

Default manifests:

```text
No NetworkPolicies are enabled by default.
```

The repo includes:

```text
kustomize/components/network-policies
```

Enable:

```sh
cd kustomize
kustomize edit add component components/network-policies
kubectl apply -k .
```

Or with Skaffold profile:

```sh
skaffold run -p network-policies --default-repo=<registry>/<repo>
```

Important repo note:

```text
The provided NetworkPolicies intentionally leave egress wide open.
```

Platform notes:

- NetworkPolicies require a CNI that enforces them.
- Examples: GKE Dataplane V2, Calico, Cilium.
- Default Kind/minikube networking may not enforce them unless configured.
- Use dependency map from checklist 5 to define allowed ingress paths.

## Security Groups For ECS

If deploying to ECS, replace Kubernetes NetworkPolicies with security groups and routing controls.

Suggested boundary:

```text
ALB security group
  -> allows internet HTTPS/HTTP
  -> forwards only to frontend service

frontend task security group
  -> allows inbound only from ALB
  -> allows outbound to backend services

backend task security groups
  -> allow inbound only from required calling services

Redis/DB security group
  -> allow inbound only from cartservice or required app services
```

Do not allow public inbound traffic to backend task security groups.

## Timeout Settings

Default manifests:

```text
No explicit ingress/load balancer timeout settings are defined.
```

Timeouts to confirm:

- frontend HTTP request timeout
- load balancer idle timeout
- ingress/gateway timeout
- gRPC client deadlines/timeouts
- service mesh route timeout
- connection draining/termination grace period

Special attention:

- checkout flow calls multiple services.
- gRPC calls may fail differently through proxies/load balancers if HTTP/2 is not handled correctly.
- service mesh and ingress defaults may be too short or too long.

Platform decision:

```text
Define timeout standards for HTTP and gRPC before production.
```

## Load Balancer Health Checks

Default public load balancer path:

```text
frontend-external -> frontend pods
```

Kubernetes Service type `LoadBalancer` relies on cloud provider behavior and pod readiness.

Frontend readiness:

```text
GET /_healthz on container port 8080
```

Expected response:

```text
HTTP 200
ok
```

If using ALB/Ingress/Gateway:

- configure health check path as `/_healthz`
- expect HTTP `200`
- health check the frontend only
- do not route external health checks directly to gRPC backends

## Managed Storage Networking

Storage choices affect networking.

| Storage option | Network requirement |
| --- | --- |
| In-cluster Redis | Pod-to-Service TCP `redis-cart:6379` |
| Managed Redis/Memorystore | Cluster must reach Redis private IP/network |
| ElastiCache | ECS/EKS subnet and security groups must allow Redis port |
| Spanner | App identity and Google API/private access path |
| AlloyDB | VPC/private connectivity and DB security rules |
| Shopping assistant | AlloyDB, Secret Manager, Gemini/Generative AI API access |

Platform note:

```text
Networking is not only frontend exposure.
Managed databases and cloud APIs also need explicit network paths and IAM.
```

## Validation Commands

Check services:

```sh
kubectl get svc
```

Expected:

- `frontend-external` is `LoadBalancer`
- backend services are `ClusterIP`

Check external IP:

```sh
kubectl get service frontend-external
```

Port-forward internal frontend:

```sh
kubectl port-forward svc/frontend 8080:80
curl -i http://localhost:8080/_healthz
```

Check endpoints:

```sh
kubectl get endpoints
```

Check NetworkPolicies:

```sh
kubectl get networkpolicy
```

Check Gateway API resources if using mesh/gateway:

```sh
kubectl get gateway
kubectl get httproute
```

Check Ingress if using ingress:

```sh
kubectl get ingress
kubectl describe ingress <name>
```

Test public app:

```sh
curl -i http://<external-ip>/_healthz
curl -i http://<external-ip>/
```

## Networking And Exposure Risks

| Area | Risk | Follow-up |
| --- | --- | --- |
| Public exposure | Default LoadBalancer exposes frontend over HTTP only | Add TLS/DNS or replace with Ingress/Gateway. |
| Backend exposure | Accidentally exposing backend gRPC services publicly | Keep all backends as ClusterIP/private target groups. |
| TLS | No default TLS certificate | Configure cert-manager/cloud managed cert/ACM. |
| DNS | No default DNS record | Create controlled DNS record for the public endpoint. |
| Network policy | Not enabled by default | Enable if environment requires segmentation and CNI supports it. |
| Service mesh | Istio component changes exposure model | Decide before production; test gRPC behavior. |
| Timeouts | Not explicitly configured in default manifests | Define HTTP/gRPC timeout standards. |
| Managed storage | External DB/cache needs network access | Validate VPC/subnet/security group/private access. |
| Assistant service | Requires external Google APIs and AlloyDB | Do not enable until network/IAM dependencies work. |

## Networking And Exposure Output

At the end of this stage, platform engineering should have:

- [x] Public entrypoint documented.
- [x] Private backend boundary documented.
- [x] Default LoadBalancer model documented.
- [x] Alternative Ingress/Gateway/service mesh models documented.
- [x] NetworkPolicy option documented.
- [x] ECS security group/load balancer translation documented.
- [x] Load balancer health check expectation documented.
- [ ] TLS approach finalized.
- [ ] DNS approach finalized.
- [ ] CORS decision finalized.
- [ ] NetworkPolicy/security group rules finalized.
- [ ] Timeout settings finalized.
- [ ] Service mesh decision finalized.
- [ ] Public endpoint validated after deployment.

## Ready For Next Checklist Item

The next stage is:

```text
10. Security
```

In that stage, validate container security, pod/task security, secrets handling, RBAC/IAM, image scanning, network restrictions, and supply-chain controls.

