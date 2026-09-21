# Run Locally With Docker Compose

This repository is the Online Boutique microservices demo. The upstream local
development path uses a local Kubernetes cluster plus Skaffold, but you can run
the core app locally with Docker Compose first.

## What Runs

The app is a set of small services that talk over gRPC. Docker Compose gives
each service a DNS name matching its service name, so addresses like
`productcatalogservice:3550` work the same way they do in Kubernetes.

| Service | Runtime | Port inside Compose | Role |
| --- | --- | --- | --- |
| `frontend` | Go | `8080` | Website HTTP server |
| `productcatalogservice` | Go | `3550` | Product list and search |
| `currencyservice` | Node.js | `7000` | Currency conversion |
| `cartservice` | C#/.NET | `7070` | Cart API |
| `redis-cart` | Redis | `6379` | Cart storage |
| `recommendationservice` | Python | `8080` | Product recommendations |
| `checkoutservice` | Go | `5050` | Checkout orchestration |
| `paymentservice` | Node.js | `50051` | Mock payment |
| `shippingservice` | Go | `50051` | Mock shipping quotes/orders |
| `emailservice` | Python | `5000` | Mock order email |
| `adservice` | Java | `9555` | Contextual ads |

Only the frontend is published to your host machine:

```sh
docker compose up --build
```

Then open:

```text
http://localhost:8080
```

Stop it with:

```sh
docker compose down
```

## Optional Load Generator

The load generator is kept behind a Compose profile so it does not start during
normal local development.

```sh
docker compose --profile loadtest up --build
```

## Notes

- The first build can be slow because this project uses Go, Node.js, Python,
  Java, .NET, Redis, and several base images.
- On Apple Silicon, build the services for the native Docker target
  architecture. The local Dockerfiles avoid forcing `amd64` for the Go and .NET
  services because emulated x86 builds can crash or hang.
- If frontend compilation appears stuck, check Docker Desktop resources. Existing
  local Kubernetes clusters or other control-plane containers can consume enough
  CPU to make the Go build look frozen.
- The frontend has an optional shopping assistant integration. The Compose file
  sets `SHOPPING_ASSISTANT_SERVICE_ADDR` because the frontend requires the env
  var at startup, but `ENABLE_ASSISTANT` is not enabled.
- This setup is for local learning and smoke testing. The Kubernetes manifests
  are still the source of truth for the upstream deployment shape.
