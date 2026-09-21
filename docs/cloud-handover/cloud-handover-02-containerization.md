# Cloud Handover Checklist: 2. Containerization

Scenario: the development team has handed this application to the platform engineering team. We are preparing it for cloud infrastructure, instrumentation, and deployment to Kubernetes or ECS.

This note covers the second checklist item only: **Containerization**.

## Objective

Each service should be independently buildable and runnable as a container.

For platform engineering, the containerization phase answers:

- Does every application component have a container image definition?
- Can each image build from a clean checkout?
- Does each image contain the correct runtime artifact?
- Does each container start the correct process?
- Does runtime config come from environment variables or mounted config?
- Can each container run outside Docker Compose?
- Is the image appropriate for Kubernetes or ECS?

## Checklist

- [X] Confirm each service has a Dockerfile.
- [ ] Confirm the Dockerfile builds from a clean checkout.
- [X] Confirm dependencies are installed during build.
- [X] Confirm the correct files are copied into the image.
- [X] Confirm the container starts the correct process.
- [X] Confirm the container exposes or documents the correct port.
- [X] Confirm config comes from environment variables or mounted config, not hardcoded values.
- [X] Confirm the image does not depend on local files outside the build context.
- [ ] Confirm the image can run without Docker Compose.
- [X] Confirm the image runs as non-root where possible.
- [X] Confirm unnecessary package managers/build tools are not present in runtime images.
- [ ] Confirm image size is reasonable.

Note:

- Items marked checked are confirmed by reviewing this repo's Dockerfiles and manifests.
- Items still unchecked require actual image builds/runs in CI or locally.

## Common Dockerfile Patterns

| Runtime | Packaging pattern                                                                      |
| ------- | -------------------------------------------------------------------------------------- |
| Go      | Build static binary, copy binary into small runtime/distroless image                   |
| Node.js | Install with`npm ci` or production install, copy app, run `node` or package script |
| Python  | Install`requirements.txt`, copy app, run Python entrypoint or WSGI/ASGI server       |
| Java    | Build with Maven/Gradle, run JAR or generated start script                             |
| .NET    | `dotnet publish`, run published app in runtime image                                 |

Basic image commands:

```sh
docker build -t <service-name>:<tag> ./path/to/service
docker run --rm -p <host-port>:<container-port> <service-name>:<tag>
```

## Containerization Status Summary

This application is already containerized.

Each application service has a Dockerfile in its service directory. `redis-cart` does not have a project Dockerfile because it uses the public `redis:alpine` image.

| Component                    | Dockerfile present? | Build context                    | Runtime pattern                              | Status                                |
| ---------------------------- | ------------------- | -------------------------------- | -------------------------------------------- | ------------------------------------- |
| `frontend`                 | Yes                 | `src/frontend`                 | Go multi-stage build, distroless runtime     | Ready for build validation            |
| `productcatalogservice`    | Yes                 | `src/productcatalogservice`    | Go multi-stage build, distroless runtime     | Ready for build validation            |
| `checkoutservice`          | Yes                 | `src/checkoutservice`          | Go multi-stage build, distroless runtime     | Ready for build validation            |
| `shippingservice`          | Yes                 | `src/shippingservice`          | Go multi-stage build, distroless runtime     | Ready for build validation            |
| `currencyservice`          | Yes                 | `src/currencyservice`          | Node.js builder, Alpine runtime with Node.js | Ready for build validation            |
| `paymentservice`           | Yes                 | `src/paymentservice`           | Node.js builder, Alpine runtime with Node.js | Ready for build validation            |
| `emailservice`             | Yes                 | `src/emailservice`             | Python builder, Python Alpine runtime        | Ready for build validation            |
| `recommendationservice`    | Yes                 | `src/recommendationservice`    | Python builder, Python Alpine runtime        | Ready for build validation            |
| `cartservice`              | Yes                 | `src/cartservice/src`          | .NET publish, runtime-deps image             | Ready for build validation            |
| `adservice`                | Yes                 | `src/adservice`                | Gradle build, Java JRE runtime               | Ready for build validation            |
| `loadgenerator`            | Yes                 | `src/loadgenerator`            | Python/Locust image                          | Optional, ready for build validation  |
| `shoppingassistantservice` | Yes                 | `src/shoppingassistantservice` | Python builder, Python slim runtime          | Optional, requires external config    |
| `redis-cart`               | Not applicable      | Public image                     | `redis:alpine`                             | External/public base image dependency |

## Service Build Commands

These commands build each service image directly using Docker.

Run from the repo root.

```sh
docker build -t frontend:local ./src/frontend
docker build -t productcatalogservice:local ./src/productcatalogservice
docker build -t checkoutservice:local ./src/checkoutservice
docker build -t shippingservice:local ./src/shippingservice
docker build -t currencyservice:local ./src/currencyservice
docker build -t paymentservice:local ./src/paymentservice
docker build -t emailservice:local ./src/emailservice
docker build -t recommendationservice:local ./src/recommendationservice
docker build -t cartservice:local ./src/cartservice/src
docker build -t adservice:local ./src/adservice
docker build -t loadgenerator:local ./src/loadgenerator
docker build -t shoppingassistantservice:local ./src/shoppingassistantservice
```

Skaffold can build the full app using the build contexts defined in `skaffold.yaml`:

```sh
skaffold build --default-repo=<registry>/<repo>
```

For build and deploy:

```sh
skaffold run --default-repo=<registry>/<repo>
```

## Runtime Packaging Details

| Component                    | Dependencies installed during build                    | Runtime files copied                           | Container entrypoint                             | Exposed port |
| ---------------------------- | ------------------------------------------------------ | ---------------------------------------------- | ------------------------------------------------ | -----------: |
| `frontend`                 | `go mod download`                                    | compiled Go binary,`templates/`, `static/` | `/src/server`                                  |     `8080` |
| `productcatalogservice`    | `go mod download`                                    | compiled Go binary,`products.json`           | `/src/server`                                  |     `3550` |
| `checkoutservice`          | `go mod download`                                    | compiled Go binary                             | `/src/checkoutservice`                         |     `5050` |
| `shippingservice`          | `go mod download`                                    | compiled Go binary                             | `/src/shippingservice`                         |    `50051` |
| `currencyservice`          | `npm install --only=production`                      | `node_modules`, app source                   | `node server.js`                               |     `7000` |
| `paymentservice`           | `npm install --only=production`                      | `node_modules`, app source                   | `node index.js`                                |    `50051` |
| `emailservice`             | `pip install -r requirements.txt`                    | Python packages, app source, templates         | `python email_server.py`                       |     `8080` |
| `recommendationservice`    | `pip install -r requirements.txt`                    | Python packages, app source                    | `python recommendation_server.py`              |     `8080` |
| `cartservice`              | `dotnet restore`, `dotnet publish`                 | published .NET app                             | `/app/cartservice`                             |     `7070` |
| `adservice`                | `./gradlew downloadRepos`, `./gradlew installDist` | Java distribution                              | `/app/build/install/hipstershop/bin/AdService` |     `9555` |
| `loadgenerator`            | `pip install -r requirements.txt`                    | Python packages,`locustfile.py`              | `locust --host=... --headless ...`             |         none |
| `shoppingassistantservice` | `pip install -r requirements.txt`                    | Python packages, app source                    | `python shoppingassistantservice.py`           |     `8080` |

## Runtime Images

| Runtime | Services                                                                          | Runtime base image                        |
| ------- | --------------------------------------------------------------------------------- | ----------------------------------------- |
| Go      | `frontend`, `productcatalogservice`, `checkoutservice`, `shippingservice` | `gcr.io/distroless/static`              |
| Node.js | `currencyservice`, `paymentservice`                                           | `alpine` with `nodejs` installed      |
| Python  | `emailservice`, `recommendationservice`, `loadgenerator`                    | `python:3.14.6-alpine`                  |
| Python  | `shoppingassistantservice`                                                      | `python:3.14.6-slim`                    |
| .NET    | `cartservice`                                                                   | `mcr.microsoft.com/dotnet/runtime-deps` |
| Java    | `adservice`                                                                     | `eclipse-temurin` JRE Alpine            |
| Redis   | `redis-cart`                                                                    | `redis:alpine`                          |

## Config Handling

Runtime configuration is mostly environment-variable based.

Examples:

| Component                    | Runtime config                                               |
| ---------------------------- | ------------------------------------------------------------ |
| `frontend`                 | `PORT`, service dependency addresses, feature flags        |
| `checkoutservice`          | `PORT`, service dependency addresses                       |
| `recommendationservice`    | `PORT`, `PRODUCT_CATALOG_SERVICE_ADDR`                   |
| `cartservice`              | `REDIS_ADDR`, optional database settings                   |
| `productcatalogservice`    | `PORT`, optional AlloyDB settings                          |
| `loadgenerator`            | `FRONTEND_ADDR`, `USERS`, `RATE`                       |
| `shoppingassistantservice` | Google Cloud project, region, AlloyDB, Secret Manager config |

Platform validation:

- Containers should not require local files outside the image.
- Service addresses should be provided through env vars.
- Secrets should not be baked into images.
- Environment-specific config should not be hardcoded.

## Can Images Run Without Docker Compose?

The goal is yes, but each service has different dependency requirements.

| Component                    | Can run standalone? | Notes                                                                                                           |
| ---------------------------- | ------------------- | --------------------------------------------------------------------------------------------------------------- |
| `productcatalogservice`    | Yes                 | No required internal dependencies.                                                                              |
| `currencyservice`          | Yes                 | No required internal dependencies.                                                                              |
| `shippingservice`          | Yes                 | No required internal dependencies.                                                                              |
| `paymentservice`           | Yes                 | No required internal dependencies.                                                                              |
| `emailservice`             | Yes                 | Runs in dummy mode by default.                                                                                  |
| `adservice`                | Yes                 | No required internal dependencies.                                                                              |
| `cartservice`              | Partially           | Needs Redis for the default clustered deployment, though app can fall back to memory if`REDIS_ADDR` is unset. |
| `recommendationservice`    | No                  | Requires`PRODUCT_CATALOG_SERVICE_ADDR`.                                                                       |
| `checkoutservice`          | No                  | Requires product, cart, currency, shipping, payment, and email services.                                        |
| `frontend`                 | No                  | Requires backend service addresses.                                                                             |
| `loadgenerator`            | No                  | Requires`FRONTEND_ADDR`.                                                                                      |
| `shoppingassistantservice` | No                  | Requires Google Cloud/Gemini/AlloyDB configuration.                                                             |

## Example Standalone Container Runs

These examples are useful for validating image startup before deploying to Kubernetes or ECS.

### productcatalogservice

```sh
docker run --rm -p 3550:3550 \
  -e PORT=3550 \
  -e DISABLE_PROFILER=1 \
  productcatalogservice:local
```

### currencyservice

```sh
docker run --rm -p 7000:7000 \
  -e PORT=7000 \
  -e DISABLE_PROFILER=1 \
  currencyservice:local
```

### shippingservice

```sh
docker run --rm -p 50051:50051 \
  -e PORT=50051 \
  -e DISABLE_PROFILER=1 \
  shippingservice:local
```

### paymentservice

```sh
docker run --rm -p 50052:50051 \
  -e PORT=50051 \
  -e DISABLE_PROFILER=1 \
  paymentservice:local
```

Note:

- The container listens on `50051`.
- The host maps it to `50052` here to avoid conflicts with `shippingservice`.

### adservice

```sh
docker run --rm -p 9555:9555 \
  -e PORT=9555 \
  adservice:local
```

### cartservice With Redis

```sh
docker network create boutique-local

docker run --rm --name redis-cart \
  --network boutique-local \
  redis:alpine
```

In another terminal:

```sh
docker run --rm -p 7070:7070 \
  --network boutique-local \
  -e REDIS_ADDR=redis-cart:6379 \
  cartservice:local
```

### recommendationservice

Requires `productcatalogservice`.

```sh
docker network create boutique-local
```

Run product catalog:

```sh
docker run --rm --name productcatalogservice \
  --network boutique-local \
  -e PORT=3550 \
  -e DISABLE_PROFILER=1 \
  productcatalogservice:local
```

Run recommendation service:

```sh
docker run --rm -p 8081:8080 \
  --network boutique-local \
  -e PORT=8080 \
  -e PRODUCT_CATALOG_SERVICE_ADDR=productcatalogservice:3550 \
  -e DISABLE_PROFILER=1 \
  recommendationservice:local
```

### frontend

`frontend` should be validated after backend services are running, because it requires service addresses at startup.

Minimum required env vars:

```text
PRODUCT_CATALOG_SERVICE_ADDR
CURRENCY_SERVICE_ADDR
CART_SERVICE_ADDR
RECOMMENDATION_SERVICE_ADDR
SHIPPING_SERVICE_ADDR
CHECKOUT_SERVICE_ADDR
AD_SERVICE_ADDR
SHOPPING_ASSISTANT_SERVICE_ADDR
```

## Non-Root And Runtime Hardening

Container-level findings:

| Component              | Non-root configured in Dockerfile? | Notes                                                    |
| ---------------------- | ---------------------------------- | -------------------------------------------------------- |
| `cartservice`        | Yes                                | Dockerfile sets`USER 1000`.                            |
| Go distroless services | Implicit/minimal runtime           | Kubernetes manifests set pod/container security context. |
| Node.js services       | Not explicitly in Dockerfile       | Kubernetes manifests set pod/container security context. |
| Python services        | Not explicitly in Dockerfile       | Kubernetes manifests set pod/container security context. |
| `adservice`          | Not explicitly in Dockerfile       | Kubernetes manifests set pod/container security context. |
| `redis-cart`         | Public image                       | Kubernetes manifests set pod/container security context. |

Kubernetes manifests harden containers with:

- `runAsNonRoot`
- `runAsUser: 1000`
- `runAsGroup: 1000`
- `allowPrivilegeEscalation: false`
- `capabilities.drop: ALL`
- `readOnlyRootFilesystem: true`

Platform note:

- For Kubernetes, runtime hardening is mostly enforced in manifests.
- For ECS, equivalent controls should be applied in task definitions where supported.
- If running containers directly outside Kubernetes, consider adding explicit non-root users to Dockerfiles where they are missing.

## Build Context Validation

Each Dockerfile should only rely on files inside its build context.

| Component                    | Build context                    | Important copied files                                                      |
| ---------------------------- | -------------------------------- | --------------------------------------------------------------------------- |
| `frontend`                 | `src/frontend`                 | `go.mod`, `go.sum`, Go source, `templates/`, `static/`              |
| `productcatalogservice`    | `src/productcatalogservice`    | `go.mod`, `go.sum`, Go source, `products.json`                        |
| `checkoutservice`          | `src/checkoutservice`          | `go.mod`, `go.sum`, Go source                                           |
| `shippingservice`          | `src/shippingservice`          | `go.mod`, `go.sum`, Go source                                           |
| `currencyservice`          | `src/currencyservice`          | `package.json`, `package-lock.json`, JS source, proto files, data files |
| `paymentservice`           | `src/paymentservice`           | `package.json`, `package-lock.json`, JS source, proto files             |
| `emailservice`             | `src/emailservice`             | `requirements.txt`, Python source, templates                              |
| `recommendationservice`    | `src/recommendationservice`    | `requirements.txt`, Python source, generated proto files                  |
| `cartservice`              | `src/cartservice/src`          | `.csproj`, C# source, proto files, config                                 |
| `adservice`                | `src/adservice`                | Gradle files, Java source, proto files                                      |
| `loadgenerator`            | `src/loadgenerator`            | `requirements.txt`, `locustfile.py`                                     |
| `shoppingassistantservice` | `src/shoppingassistantservice` | `requirements.txt`, Python source                                         |

## Image Size Review

Image size should be validated after build:

```sh
docker images
docker image inspect <image>:<tag> --format='{{.Size}}'
```

Expected image size guidance:

| Runtime                          | Expected direction                                 |
| -------------------------------- | -------------------------------------------------- |
| Go distroless                    | Smallest images                                    |
| Node.js Alpine                   | Moderate                                           |
| Python Alpine/slim               | Moderate to large depending on native dependencies |
| Java JRE                         | Larger                                             |
| .NET runtime-deps self-contained | Moderate to large                                  |

Platform note:

- Do not judge image size by one global number.
- Compare each image to normal expectations for that runtime.
- Large images are acceptable if justified by runtime dependencies, but unnecessary build tools should not be present in runtime layers.

## Build Validation Commands

Use these during handover to prove containerization works.

Build all known app images:

```sh
docker build -t frontend:local ./src/frontend
docker build -t productcatalogservice:local ./src/productcatalogservice
docker build -t checkoutservice:local ./src/checkoutservice
docker build -t shippingservice:local ./src/shippingservice
docker build -t currencyservice:local ./src/currencyservice
docker build -t paymentservice:local ./src/paymentservice
docker build -t emailservice:local ./src/emailservice
docker build -t recommendationservice:local ./src/recommendationservice
docker build -t cartservice:local ./src/cartservice/src
docker build -t adservice:local ./src/adservice
docker build -t loadgenerator:local ./src/loadgenerator
docker build -t shoppingassistantservice:local ./src/shoppingassistantservice
```

Build using Skaffold:

```sh
skaffold build
```

Build for a target registry:

```sh
skaffold build --default-repo=<registry>/<repo>
```

## Container Runtime Validation Commands

After building images, validate startup:

```sh
docker run --rm <image>:<tag>
```

Validate a port-listening service:

```sh
docker run --rm -p <host-port>:<container-port> <image>:<tag>
```

Validate logs:

```sh
docker logs <container-id>
```

Validate processes:

```sh
docker ps
```

Validate image size:

```sh
docker images
```

## Containerization Risks And Follow-Ups

| Area                       | Risk                                             | Follow-up                                                                 |
| -------------------------- | ------------------------------------------------ | ------------------------------------------------------------------------- |
| Build validation           | Dockerfiles reviewed but not built in this note  | Run image builds in CI or locally                                         |
| Node.js dependency install | Dockerfiles use`npm install --only=production` | Consider`npm ci --omit=dev` for more reproducible builds                |
| Python dependency install  | Dependencies installed from`requirements.txt`  | Confirm lock/pinning policy is acceptable                                 |
| Runtime users              | Some Dockerfiles do not explicitly set`USER`   | Kubernetes security context currently handles this; verify ECS equivalent |
| Optional assistant         | Requires cloud dependencies                      | Exclude or configure explicitly                                           |
| Image size                 | Not measured in this note                        | Measure after build and compare to runtime expectations                   |
| Registry                   | Images not yet pushed by this stage              | Covered in checklist 3: Image Registry and Tagging                        |

## Containerization Output

At the end of this stage, platform engineering should have:

- [X] A Dockerfile inventory.
- [X] A build context map.
- [X] Runtime packaging notes per service.
- [X] Entrypoint and exposed port confirmation.
- [X] A list of standalone and dependency-bound services.
- [X] Build commands for individual services.
- [X] Known containerization risks and follow-ups.
- [ ] Successful local or CI image build results.
- [ ] Successful container startup validation results.
- [ ] Image size measurements.

## Ready For Next Checklist Item

The next stage is:

```text
3. Image Registry And Tagging
```

In that stage, define image naming, tagging, registry location, cluster pull access, image scanning, and promotion strategy.
