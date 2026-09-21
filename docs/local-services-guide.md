# Local Services Guide

This guide explains how the services in this repo relate technically, which service to learn first, how to run each service directly on your local PC, and how to test the service APIs.

The important mental model: this application is one HTTP website backed by many gRPC services. The `frontend` is the browser-facing app. Almost everything else is an internal gRPC API.

## Where to start

Start with `productcatalogservice`.

It is the easiest service to understand because it has no runtime dependency on other services. It reads `products.json`, exposes product lookup/search/list RPCs, and other services depend on it.

Recommended learning order:

1. `productcatalogservice`: static catalog, no dependencies.
2. `currencyservice`, `shippingservice`, `paymentservice`, `emailservice`, `adservice`: independent leaf services.
3. `cartservice`: simple API, but needs a cart store. It can use Redis or local memory.
4. `recommendationservice`: calls `productcatalogservice`.
5. `checkoutservice`: orchestrates cart, catalog, shipping, currency, payment, and email.
6. `frontend`: HTTP website that calls all backend gRPC services.
7. `loadgenerator`: traffic generator, not part of the user-facing app.
8. `shoppingassistantservice`: optional Gemini/AlloyDB service that needs Google Cloud configuration.

## Service relationship map

```text
Browser
  |
  v
frontend :8080  HTTP
  |
  +--> productcatalogservice :3550  gRPC
  +--> currencyservice       :7000  gRPC
  +--> cartservice           :7070  gRPC
  +--> recommendationservice :8082  gRPC
  |      |
  |      +--> productcatalogservice :3550
  |
  +--> shippingservice       :50051 gRPC
  +--> checkoutservice       :5050  gRPC
  |      |
  |      +--> productcatalogservice :3550
  |      +--> cartservice           :7070
  |      +--> currencyservice       :7000
  |      +--> shippingservice       :50051
  |      +--> paymentservice        :50052
  |      +--> emailservice          :5000
  |
  +--> adservice             :9555  gRPC
  +--> shoppingassistant     :8081  HTTP, optional
```

## Protocols

| Service                      | Language      | Local port in this guide | Protocol    | Depends on                                                      |
| ---------------------------- | ------------- | -----------------------: | ----------- | --------------------------------------------------------------- |
| `frontend`                 | Go            |                 `8080` | HTTP        | product, currency, cart, recommendation, shipping, checkout, ad |
| `productcatalogservice`    | Go            |                 `3550` | gRPC        | none                                                            |
| `currencyservice`          | Node.js       |                 `7000` | gRPC        | none                                                            |
| `cartservice`              | C#/.NET       |                 `7070` | gRPC        | Redis optional                                                  |
| `shippingservice`          | Go            |                `50051` | gRPC        | none                                                            |
| `paymentservice`           | Node.js       |                `50052` | gRPC        | none                                                            |
| `emailservice`             | Python        |                 `5000` | gRPC        | none in dummy mode                                              |
| `recommendationservice`    | Python        |                 `8082` | gRPC        | product catalog                                                 |
| `checkoutservice`          | Go            |                 `5050` | gRPC        | product, cart, currency, shipping, payment, email               |
| `adservice`                | Java          |                 `9555` | gRPC        | none                                                            |
| `loadgenerator`            | Python/Locust |                      n/a | HTTP client | frontend                                                        |
| `shoppingassistantservice` | Python/Flask  |       `8081` suggested | HTTP        | Google Cloud/Gemini/AlloyDB                                     |

## Tools to install

For direct local execution, install the language runtimes used by the services:

- Go
- Node.js and npm
- Python 3
- .NET SDK compatible with `net10.0`
- Java 21
- `grpcurl` for testing gRPC APIs

On macOS, `grpcurl` can usually be installed with:

```sh
brew install grpcurl
```

Run all commands from the repo root unless a command explicitly says to `cd` into a service directory.

## Build and package view

You do not need to know every programming language deeply to understand how this system is built. Treat each service folder as a separate application with four things:

- source code
- dependency manifesto
- local build/run command
- `Dockerfile` that packages the service into a container image

| Service                      | Dependency/build files                                      | Local build command                                                     | Container build command                                                     | Runtime artifact                                       |
| ---------------------------- | ----------------------------------------------------------- | ----------------------------------------------------------------------- | --------------------------------------------------------------------------- | ------------------------------------------------------ |
| `frontend`                 | `go.mod`, `go.sum`, `Dockerfile`                      | `cd src/frontend && go build -o frontend .`                           | `docker build -t frontend ./src/frontend`                                 | Go binary plus`templates/` and `static/`           |
| `productcatalogservice`    | `go.mod`, `go.sum`, `Dockerfile`                      | `cd src/productcatalogservice && go build -o productcatalogservice .` | `docker build -t productcatalogservice ./src/productcatalogservice`       | Go binary plus`products.json`                        |
| `checkoutservice`          | `go.mod`, `go.sum`, `Dockerfile`                      | `cd src/checkoutservice && go build -o checkoutservice .`             | `docker build -t checkoutservice ./src/checkoutservice`                   | Go binary                                              |
| `shippingservice`          | `go.mod`, `go.sum`, `Dockerfile`                      | `cd src/shippingservice && go build -o shippingservice .`             | `docker build -t shippingservice ./src/shippingservice`                   | Go binary                                              |
| `currencyservice`          | `package.json`, `package-lock.json`, `Dockerfile`     | `cd src/currencyservice && npm install`                               | `docker build -t currencyservice ./src/currencyservice`                   | Node.js app run with`node server.js`                 |
| `paymentservice`           | `package.json`, `package-lock.json`, `Dockerfile`     | `cd src/paymentservice && npm install`                                | `docker build -t paymentservice ./src/paymentservice`                     | Node.js app run with`node index.js`                  |
| `emailservice`             | `requirements.txt`, `Dockerfile`                        | `cd src/emailservice && python3 -m pip install -r requirements.txt`   | `docker build -t emailservice ./src/emailservice`                         | Python app run with`python email_server.py`          |
| `recommendationservice`    | `requirements.txt`, `Dockerfile`                        | `cd src/recommendationservice && pip install -r requirements.txt`     | `docker build -t recommendationservice ./src/recommendationservice`       | Python app run with`python recommendation_server.py` |
| `cartservice`              | `cartservice.csproj`, `cartservice.sln`, `Dockerfile` | `cd src/cartservice/src && dotnet publish -c release -o ./publish`    | `docker build -t cartservice ./src/cartservice/src`                       | Published .NET app then`(cd src/ && dotnet run`)     |
| `adservice`                | `build.gradle`, `gradlew`, `Dockerfile`               | `cd src/adservice && ./gradlew installDist`                           | `docker build -t adservice ./src/adservice`                               | Java distribution under`build/install/hipstershop`   |
| `loadgenerator`            | `requirements.txt`, `Dockerfile`                        | `cd src/loadgenerator && pip install -r requirements.txt`             | `docker build -t loadgenerator ./src/loadgenerator`                       | Python/Locust app                                      |
| `shoppingassistantservice` | `requirements.txt`, `Dockerfile`                        | `cd src/shoppingassistantservice && pip install -r requirements.txt`  | `docker build -t shoppingassistantservice ./src/shoppingassistantservice` | Python/Flask app                                       |

The broad packaging pattern is:

1. Compile or install dependencies inside the service folder.
2. Produce a runnable process for that service.
3. Set environment variables for ports and dependency addresses.
4. Start the process.

The container packaging pattern is:

```sh
docker build -t <service-name> ./src/<service-folder>
```

Then run the image with the same environment variables used in Docker Compose or in this guide:

```sh
docker run --rm -p <host-port>:<container-port> \
  -e PORT=<container-port> \
  <service-name>
```

For example:

```sh
docker build -t productcatalogservice ./src/productcatalogservice
docker run --rm -p 3550:3550 \
  -e PORT=3550 \
  -e DISABLE_PROFILER=1 \
  productcatalogservice
```

For multi-service container packaging, `docker-compose.yml` and `skaffold.yaml` are the orchestration files:

- `docker-compose.yml` describes how to build and connect services locally with Docker.
- `skaffold.yaml` describes how to build all service images and deploy them to Kubernetes.
- `kubernetes-manifests/` and `release/kubernetes-manifests.yaml` describe the Kubernetes Deployments and Services.

So, from a DevOps/build perspective, you can understand this repo by reading the manifests first, then each service's dependency file and `Dockerfile`. You only need to open the application code when you want to understand behavior inside a specific service.

## gRPC testing pattern

Most services do not expose browser-friendly HTTP endpoints. Use `grpcurl` with the repo protobuf:

```sh
grpcurl -plaintext -proto protos/demo.proto localhost:<PORT> list
```

Health check:

```sh
grpcurl -plaintext \
  -import-path protos \
  -import-path src/currencyservice/proto \
  -proto grpc/health/v1/health.proto \
  localhost:<PORT> grpc.health.v1.Health/Check
```

For most service methods, this simpler shape is enough:

```sh
grpcurl -plaintext -proto protos/demo.proto -d '<JSON>' \
  localhost:<PORT> hipstershop.ServiceName/MethodName
```

## Run and test each service

### productcatalogservice

What it does: lists products, returns one product by ID, and searches products. It reads from `src/productcatalogservice/products.json`.

Run:

```sh
cd src/productcatalogservice
PORT=3550 DISABLE_PROFILER=1 go run .
```

Test:

```sh
grpcurl -plaintext -proto protos/demo.proto -d '{}' \
  localhost:3550 hipstershop.ProductCatalogService/ListProducts

grpcurl -plaintext -proto protos/demo.proto \
  -d '{"id":"OLJCESPC7Z"}' \
  localhost:3550 hipstershop.ProductCatalogService/GetProduct

grpcurl -plaintext -proto protos/demo.proto \
  -d '{"query":"sunglasses"}' \
  localhost:3550 hipstershop.ProductCatalogService/SearchProducts
```

### currencyservice

What it does: lists supported currencies and converts money values.

First install dependencies once:

```sh
cd src/currencyservice
npm install
```

Run:

```sh
cd src/currencyservice
PORT=7000 DISABLE_PROFILER=1 node server.js
```

Test:

```sh
grpcurl -plaintext -proto protos/demo.proto -d '{}' \
  localhost:7000 hipstershop.CurrencyService/GetSupportedCurrencies

grpcurl -plaintext -proto protos/demo.proto \
  -d '{"from":{"currency_code":"USD","units":10,"nanos":0},"to_code":"EUR"}' \
  localhost:7000 hipstershop.CurrencyService/Convert
```

### shippingservice

What it does: calculates mock shipping quotes and returns mock tracking IDs.

Run:

```sh
cd src/shippingservice
PORT=50051 DISABLE_TRACING=1 DISABLE_PROFILER=1 go run .
```

Test:

```sh
grpcurl -plaintext -proto protos/demo.proto \
  -d '{"address":{"street_address":"1600 Amphitheatre Pkwy","city":"Mountain View","state":"CA","country":"US","zip_code":94043},"items":[{"product_id":"OLJCESPC7Z","quantity":2}]}' \
  localhost:50051 hipstershop.ShippingService/GetQuote

grpcurl -plaintext -proto protos/demo.proto \
  -d '{"address":{"street_address":"1600 Amphitheatre Pkwy","city":"Mountain View","state":"CA","country":"US","zip_code":94043},"items":[{"product_id":"OLJCESPC7Z","quantity":2}]}' \
  localhost:50051 hipstershop.ShippingService/ShipOrder
```

### paymentservice

What it does: validates a mock credit card and returns a mock transaction ID.

Use `50052` locally because `shippingservice` already uses `50051`.

First install dependencies once:

```sh
cd src/paymentservice
npm install
```

Run:

```sh
cd src/paymentservice
PORT=50052 DISABLE_PROFILER=1 node index.js
```

Test:

```sh
grpcurl -plaintext -proto protos/demo.proto \
  -d '{"amount":{"currency_code":"USD","units":20,"nanos":0},"credit_card":{"credit_card_number":"4111111111111111","credit_card_cvv":123,"credit_card_expiration_year":2030,"credit_card_expiration_month":12}}' \
  localhost:50052 hipstershop.PaymentService/Charge
```

### emailservice

What it does: accepts an order confirmation request. In this repo it runs in dummy mode and logs the email instead of sending it.

First create/install Python dependencies once:

```sh
cd src/emailservice
python3 -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt
```

Run:

```sh
cd src/emailservice
. .venv/bin/activate
PORT=5000 DISABLE_PROFILER=1 python email_server.py
```

Test:

```sh
grpcurl -plaintext -proto protos/demo.proto \
  -d '{"email":"test@example.com","order":{"order_id":"order-1","shipping_tracking_id":"track-1","shipping_cost":{"currency_code":"USD","units":5,"nanos":0},"shipping_address":{"street_address":"1600 Amphitheatre Pkwy","city":"Mountain View","state":"CA","country":"US","zip_code":94043},"items":[{"item":{"product_id":"OLJCESPC7Z","quantity":1},"cost":{"currency_code":"USD","units":19,"nanos":990000000}}]}}' \
  localhost:5000 hipstershop.EmailService/SendOrderConfirmation
```

### cartservice

What it does: stores cart items by user ID.

It can run without Redis. If `REDIS_ADDR` is unset, it uses local in-memory storage. That is easiest for learning. Data disappears when the service stops.

Run with in-memory storage:

```sh
cd src/cartservice/src
ASPNETCORE_HTTP_PORTS=7070 dotnet run
```

Optional Redis mode:

```sh
redis-server --port 6379
cd src/cartservice/src
ASPNETCORE_HTTP_PORTS=7070 REDIS_ADDR=localhost:6379 dotnet run
```

Test:

```sh
grpcurl -plaintext -proto protos/demo.proto \
  -d '{"user_id":"local-user","item":{"product_id":"OLJCESPC7Z","quantity":2}}' \
  localhost:7070 hipstershop.CartService/AddItem

grpcurl -plaintext -proto protos/demo.proto \
  -d '{"user_id":"local-user"}' \
  localhost:7070 hipstershop.CartService/GetCart

grpcurl -plaintext -proto protos/demo.proto \
  -d '{"user_id":"local-user"}' \
  localhost:7070 hipstershop.CartService/EmptyCart
```

### recommendationservice

What it does: asks `productcatalogservice` for all products, removes products already in the current cart/context, and returns random recommendations.

Start `productcatalogservice` first.

First install Python dependencies once:

```sh
cd src/recommendationservice
python3 -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt
```

Run:

```sh
cd src/recommendationservice
. .venv/bin/activate
PORT=8082 PRODUCT_CATALOG_SERVICE_ADDR=localhost:3550 DISABLE_PROFILER=1 python recommendation_server.py
```

Test:

```sh
grpcurl -plaintext -proto protos/demo.proto \
  -d '{"user_id":"local-user","product_ids":["OLJCESPC7Z"]}' \
  localhost:8082 hipstershop.RecommendationService/ListRecommendations
```

### adservice

What it does: returns ads based on context keys. If no keys match, it returns random ads.

Build once:

```sh
cd src/adservice
./gradlew installDist
```

Run:

```sh
cd src/adservice
PORT=9555 ./build/install/hipstershop/bin/AdService
```

Test:

```sh
grpcurl -plaintext -proto protos/demo.proto \
  -d '{"context_keys":["clothing"]}' \
  localhost:9555 hipstershop.AdService/GetAds

grpcurl -plaintext -proto protos/demo.proto \
  -d '{}' \
  localhost:9555 hipstershop.AdService/GetAds
```

### checkoutservice

What it does: orchestrates the order flow. It reads the user's cart, gets product prices, converts currency, calculates shipping, charges payment, sends email, and empties the cart.

Start these first:

- `productcatalogservice` on `3550`
- `currencyservice` on `7000`
- `cartservice` on `7070`
- `shippingservice` on `50051`
- `paymentservice` on `50052`
- `emailservice` on `5000`

Add an item to the cart before placing an order:

```sh
grpcurl -plaintext -proto protos/demo.proto \
  -d '{"user_id":"local-user","item":{"product_id":"OLJCESPC7Z","quantity":1}}' \
  localhost:7070 hipstershop.CartService/AddItem
```

Run:

```sh
cd src/checkoutservice
PORT=5050 \
PRODUCT_CATALOG_SERVICE_ADDR=localhost:3550 \
CART_SERVICE_ADDR=localhost:7070 \
CURRENCY_SERVICE_ADDR=localhost:7000 \
SHIPPING_SERVICE_ADDR=localhost:50051 \
PAYMENT_SERVICE_ADDR=localhost:50052 \
EMAIL_SERVICE_ADDR=localhost:5000 \
go run .
```

Test:

```sh
grpcurl -plaintext -proto protos/demo.proto \
  -d '{"user_id":"local-user","user_currency":"USD","address":{"street_address":"1600 Amphitheatre Pkwy","city":"Mountain View","state":"CA","country":"US","zip_code":94043},"email":"test@example.com","credit_card":{"credit_card_number":"4111111111111111","credit_card_cvv":123,"credit_card_expiration_year":2030,"credit_card_expiration_month":12}}' \
  localhost:5050 hipstershop.CheckoutService/PlaceOrder
```

### frontend

What it does: serves the HTTP storefront and calls the backend gRPC services.

Start these first:

- `productcatalogservice`
- `currencyservice`
- `cartservice`
- `recommendationservice`
- `shippingservice`
- `paymentservice`
- `emailservice`
- `checkoutservice`
- `adservice`

Run:

```sh
cd src/frontend
PORT=8080 \
ENV_PLATFORM=local \
ENABLE_PROFILER=0 \
PRODUCT_CATALOG_SERVICE_ADDR=localhost:3550 \
CURRENCY_SERVICE_ADDR=localhost:7000 \
CART_SERVICE_ADDR=localhost:7070 \
RECOMMENDATION_SERVICE_ADDR=localhost:8082 \
SHIPPING_SERVICE_ADDR=localhost:50051 \
CHECKOUT_SERVICE_ADDR=localhost:5050 \
AD_SERVICE_ADDR=localhost:9555 \
SHOPPING_ASSISTANT_SERVICE_ADDR=localhost:8081 \
go run .
```

Test in a browser:

```text
http://localhost:8080/
http://localhost:8080/product/OLJCESPC7Z
http://localhost:8080/cart
http://localhost:8080/_healthz
```

Test with curl:

```sh
curl http://localhost:8080/_healthz
curl http://localhost:8080/product-meta/OLJCESPC7Z
```

Add to cart through the frontend:

```sh
curl -i -X POST http://localhost:8080/cart \
  -d 'product_id=OLJCESPC7Z' \
  -d 'quantity=1'
```

### loadgenerator

What it does: generates traffic against the `frontend`. It is useful after the full application is running.

First install Python dependencies once:

```sh
cd src/loadgenerator
python3 -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt
```

Run:

```sh
cd src/loadgenerator
. .venv/bin/activate
FRONTEND_ADDR=localhost:8080 locust --host=http://localhost:8080
```

Open the Locust UI if it prints a local web URL, or run headless:

```sh
cd src/loadgenerator
. .venv/bin/activate
locust --host=http://localhost:8080 --headless -u 10 -r 1
```

### shoppingassistantservice

What it does: optional AI assistant endpoint used by `frontend` route `/bot`.

This service is not a simple local-only service. It expects Google Cloud and AlloyDB/Gemini environment variables at import time:

- `PROJECT_ID`
- `REGION`
- `ALLOYDB_DATABASE_NAME`
- `ALLOYDB_TABLE_NAME`
- `ALLOYDB_CLUSTER_NAME`
- `ALLOYDB_INSTANCE_NAME`
- `ALLOYDB_SECRET_NAME`

Skip it while learning the core app. The frontend still starts as long as `SHOPPING_ASSISTANT_SERVICE_ADDR` is set; only the assistant feature needs this service to be running.

## Minimal local learning sessions

### Session 1: learn one service

Run only `productcatalogservice`, then test `ListProducts`, `GetProduct`, and `SearchProducts`.

### Session 2: learn service-to-service calls

Run:

- `productcatalogservice`
- `recommendationservice`

Then call `RecommendationService/ListRecommendations`. Watch how recommendation needs the catalog address.

### Session 3: learn state

Run `cartservice` in memory mode. Add items, get the cart, empty the cart. Restart the service and notice the data is gone.

### Session 4: learn orchestration

Run the services needed by `checkoutservice`, add an item to cart, then call `CheckoutService/PlaceOrder`.

### Session 5: run the full storefront

Run every backend service and then start `frontend`. Visit `http://localhost:8080`.

## Common local problems

### Port conflicts

Docker Compose can reuse `50051` inside separate containers. Your laptop cannot run two local processes on the same port. This guide uses `paymentservice` on `50052` to avoid conflicting with `shippingservice`.

### gRPC is not normal curl

Most backend services are gRPC, not JSON-over-HTTP. Use `grpcurl`, BloomRPC, Postman gRPC, or another gRPC client.

### Relative files matter

Some services expect to run from their own directory:

- `productcatalogservice` loads `products.json`.
- `emailservice` loads `templates/confirmation.html`.
- `frontend` loads `templates/*.html` and `static/`.

If a service cannot find a file, `cd` into that service directory and run it from there.

### Optional telemetry

Tracing and profiling are cloud-oriented. For local learning, disable them with the env vars shown in the run commands.
