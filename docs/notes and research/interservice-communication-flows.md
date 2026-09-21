# Interservice Communication Flows

Date: 2026-08-04

Context: this note explains how the microservices communicate with each other when the app is running locally or in a cluster.

## Short Answer

There is no central gRPC server, broker, queue, or message distributor sitting between the services.

Each service that exposes an API runs its own server. Other services connect to it directly using the address supplied through environment variables.

Most backend communication uses:

```text
gRPC + Protocol Buffers over HTTP/2
```

The frontend exposes browser-friendly HTTP routes, but internally it calls backend services using gRPC clients.

## What Protocol Buffers Do

Protocol Buffers define the shared service contract.

Main contract file:

```text
protos/demo.proto
```

This file defines:

- gRPC services.
- RPC methods.
- Request message shapes.
- Response message shapes.
- Shared data types like products, money, carts, orders, addresses, and credit cards.

Think of `demo.proto` like an interface contract shared by all languages.

Example conceptual shape:

```protobuf
service ProductCatalogService {
  rpc ListProducts(Empty) returns (ListProductsResponse);
  rpc GetProduct(GetProductRequest) returns (Product);
}

service CheckoutService {
  rpc PlaceOrder(PlaceOrderRequest) returns (PlaceOrderResponse);
}
```

The actual generated client/server code is language-specific:

- Go services generate Go gRPC clients and servers.
- Python services generate Python gRPC clients and servers.
- Node.js services load or generate protobuf definitions.
- Java service uses generated Java gRPC bindings.
- .NET cart service uses generated .NET gRPC bindings.

## What gRPC Does

gRPC provides the direct remote procedure call mechanism.

Flow:

```text
client service
  -> opens gRPC connection to host:port
  -> calls method from protobuf contract
  -> sends protobuf-encoded request
  -> server service handles request
  -> returns protobuf-encoded response
```

Example:

```text
frontend
  -> ProductCatalogService/ListProducts
  -> productcatalogservice
```

There is no message broker involved.

## What Kubernetes Adds

In Kubernetes, services do not usually call `localhost`.

They call Kubernetes Service DNS names:

```text
productcatalogservice:3550
currencyservice:7000
cartservice:7070
checkoutservice:5050
```

Kubernetes provides:

- DNS name resolution.
- Stable virtual service IPs.
- Load balancing across matching pods.
- Network routing through kube-proxy/CNI.

Kubernetes does not understand or distribute protobuf messages. It routes TCP traffic to pods.

Conceptually:

```text
frontend pod
  -> DNS lookup productcatalogservice
  -> Kubernetes Service productcatalogservice
  -> one ready productcatalogservice pod
  -> gRPC request handled by app code
```

## Local Development Addresses

When running locally, you manually provide dependency addresses.

Example frontend local command:

```bash
PORT=8088 \
PRODUCT_CATALOG_SERVICE_ADDR=localhost:3550 \
CURRENCY_SERVICE_ADDR=localhost:7001 \
CART_SERVICE_ADDR=localhost:5000 \
RECOMMENDATION_SERVICE_ADDR=localhost:8082 \
SHIPPING_SERVICE_ADDR=localhost:50051 \
CHECKOUT_SERVICE_ADDR=localhost:5050 \
AD_SERVICE_ADDR=localhost:9555 \
SHOPPING_ASSISTANT_SERVICE_ADDR=localhost:8083 \
./frontend
```

That command tells frontend where each gRPC backend is listening.

## Core Code References

| Area | Code reference | What to look for |
| --- | --- | --- |
| Shared protobuf contract | `protos/demo.proto` | Service definitions, RPC methods, request/response messages. |
| Frontend env wiring | `src/frontend/main.go` | Required backend env vars such as `PRODUCT_CATALOG_SERVICE_ADDR`, `CHECKOUT_SERVICE_ADDR`, `AD_SERVICE_ADDR`. |
| Frontend gRPC clients | `src/frontend/rpc.go` | Client functions that call backend gRPC services. |
| Frontend HTTP handlers | `src/frontend/handlers.go` | Browser-facing routes that trigger backend RPC calls. |
| Product catalog server | `src/productcatalogservice/server.go` | gRPC server setup and listener. |
| Product catalog implementation | `src/productcatalogservice/product_catalog.go` | `ListProducts`, `GetProduct`, and product data loading. |
| Recommendation server | `src/recommendationservice/recommendation_server.py` | gRPC server, product catalog client, recommendation method. |
| Checkout server | `src/checkoutservice/main.go` | gRPC clients for dependencies and `PlaceOrder` orchestration. |
| Shipping server | `src/shippingservice/main.go` | Shipping gRPC server and quote/shipping methods. |
| Payment server | `src/paymentservice/index.js` or `src/paymentservice/server.js` | Payment gRPC server and charge method. |
| Email server | `src/emailservice/email_server.py` | Email gRPC server and order confirmation method. |
| Currency server | `src/currencyservice/server.js` | Currency gRPC server and conversion methods. |
| Ad server | `src/adservice/src/main/java/hipstershop/AdService.java` | Ad gRPC server and `GetAds` implementation. |
| Cart service | `src/cartservice/src` | .NET gRPC cart implementation and Redis integration. |
| Kubernetes services | `kubernetes-manifests/*.yaml` | Service names, ports, selectors, and deployment env vars. |

## Flow 1: User Opens Homepage

User action:

```text
GET http://localhost:8088/
```

High-level flow:

```text
browser
  -> frontend HTTP server
  -> productcatalogservice gRPC
  -> currencyservice gRPC
  -> recommendationservice gRPC
  -> adservice gRPC
  -> frontend renders HTML
  -> browser receives page
```

Code references:

| Step | Code reference | What happens |
| --- | --- | --- |
| Browser route handled | `src/frontend/handlers.go` | Homepage handler receives HTTP request. |
| Frontend dependency config loaded | `src/frontend/main.go` | Frontend reads service addresses from env vars. |
| Product list fetched | `src/frontend/rpc.go` | Frontend calls product catalog gRPC client. |
| Products served | `src/productcatalogservice/product_catalog.go` | Product catalog service returns product data. |
| Currency data used | `src/frontend/rpc.go`, `src/currencyservice/server.js` | Frontend calls currency service to format/convert prices. |
| Recommendations fetched | `src/frontend/rpc.go`, `src/recommendationservice/recommendation_server.py` | Frontend asks recommendation service for suggested products. |
| Ads fetched | `src/frontend/rpc.go`, `src/adservice/src/main/java/hipstershop/AdService.java` | Frontend asks adservice for ads. |

Important point:

```text
The browser does not directly call productcatalogservice.
The browser calls frontend, and frontend calls productcatalogservice over gRPC.
```

## Flow 2: User Opens Product Detail Page

User action:

```text
GET http://localhost:8088/product/<product-id>
```

Example:

```text
GET http://localhost:8088/product/OLJCESPC7Z
```

High-level flow:

```text
browser
  -> frontend HTTP product route
  -> productcatalogservice/GetProduct
  -> recommendationservice/ListRecommendations
  -> adservice/GetAds
  -> currencyservice conversion/formatting
  -> frontend renders product detail page
```

Code references:

| Step | Code reference | What happens |
| --- | --- | --- |
| Product page handler | `src/frontend/handlers.go` | Handles `/product/{id}` style route. |
| Product RPC client | `src/frontend/rpc.go` | Calls `ProductCatalogService/GetProduct`. |
| Product lookup | `src/productcatalogservice/product_catalog.go` | Finds product by ID from catalog data. |
| Recommendations | `src/recommendationservice/recommendation_server.py` | Uses product IDs to generate recommendations. |
| Ads | `src/adservice/src/main/java/hipstershop/AdService.java` | Returns contextual ads. |

Direct gRPC test:

```bash
grpcurl -plaintext \
  -proto protos/demo.proto \
  -d '{"id":"OLJCESPC7Z"}' \
  localhost:3550 \
  hipstershop.ProductCatalogService/GetProduct
```

## Flow 3: User Adds Item To Cart

User action:

```text
Add item to cart from frontend
```

High-level flow:

```text
browser
  -> frontend HTTP cart handler
  -> cartservice/AddItem
  -> cartservice stores cart data in Redis
  -> frontend redirects/renders cart page
```

Code references:

| Step | Code reference | What happens |
| --- | --- | --- |
| Cart HTTP route | `src/frontend/handlers.go` | Receives add-to-cart form/request. |
| Cart RPC client | `src/frontend/rpc.go` | Calls cart service gRPC method. |
| Cart implementation | `src/cartservice/src` | Handles cart operations. |
| Redis dependency | `src/cartservice/src` | Cart data is stored/retrieved from Redis. |

Important dependency:

```text
cartservice -> Redis
```

Local address used:

```text
CART_SERVICE_ADDR=localhost:5000
REDIS_ADDR=localhost:6379
```

## Flow 4: User Views Cart

User action:

```text
GET http://localhost:8088/cart
```

High-level flow:

```text
browser
  -> frontend HTTP cart page
  -> cartservice/GetCart
  -> productcatalogservice/GetProduct for cart item details
  -> currencyservice for display currency
  -> frontend renders cart page
```

Code references:

| Step | Code reference | What happens |
| --- | --- | --- |
| Cart page handler | `src/frontend/handlers.go` | Handles cart page request. |
| Cart RPC client | `src/frontend/rpc.go` | Fetches user cart. |
| Product RPC client | `src/frontend/rpc.go` | Gets full product details for cart items. |
| Cart backend | `src/cartservice/src` | Reads cart items from Redis. |
| Product backend | `src/productcatalogservice/product_catalog.go` | Returns product details. |

## Flow 5: User Places Order

User action:

```text
POST/GET through frontend checkout flow
```

Observed local URL:

```text
http://localhost:8088/cart/checkout
```

High-level flow:

```text
browser
  -> frontend checkout handler
  -> checkoutservice/PlaceOrder
     -> cartservice/GetCart
     -> productcatalogservice/GetProduct
     -> currencyservice/Convert
     -> shippingservice/GetQuote
     -> paymentservice/Charge
     -> emailservice/SendOrderConfirmation
     -> cartservice/EmptyCart
  -> checkoutservice returns order result
  -> frontend renders confirmation/error page
```

This is the most important interservice flow in the app.

Code references:

| Step | Code reference | What happens |
| --- | --- | --- |
| Frontend checkout route | `src/frontend/handlers.go` | Receives checkout action from browser. |
| Checkout RPC client | `src/frontend/rpc.go` | Calls checkout service over gRPC. |
| Checkout service config | `src/checkoutservice/main.go` | Reads dependency addresses from env vars. |
| Place order orchestration | `src/checkoutservice/main.go` | Coordinates cart, product, currency, shipping, payment, email, and cart cleanup. |
| Cart calls | `src/cartservice/src` | Gets and empties cart. |
| Product calls | `src/productcatalogservice/product_catalog.go` | Retrieves product details/prices. |
| Currency calls | `src/currencyservice/server.js` | Converts money values. |
| Shipping calls | `src/shippingservice/main.go` | Gets quote and ships order. |
| Payment calls | `src/paymentservice/index.js` or `src/paymentservice/server.js` | Charges test credit card. |
| Email calls | `src/emailservice/email_server.py` | Sends dummy order confirmation. |

Local checkout command that made this flow work:

```bash
PRODUCT_CATALOG_SERVICE_ADDR=localhost:3550 \
SHIPPING_SERVICE_ADDR=localhost:50051 \
PAYMENT_SERVICE_ADDR=localhost:50052 \
EMAIL_SERVICE_ADDR=localhost:5005 \
CURRENCY_SERVICE_ADDR=localhost:7001 \
CART_SERVICE_ADDR=localhost:5000 \
./checkoutservice
```

Important lesson:

```text
Checkout must be restarted whenever cart, currency, email, payment, shipping,
or product catalog addresses change.
```

The checkout logs showed stale addresses before the fix:

```text
cartSvcAddr:localhost:7070
currencySvcAddr:localhost:7000
emailSvcAddr:localhost:5000
```

Those were wrong for the final local port map.

## Flow 6: Recommendation Service Calls Product Catalog

User action:

```text
Homepage or product page loads recommendations.
```

High-level flow:

```text
frontend
  -> recommendationservice/ListRecommendations
  -> recommendationservice calls productcatalogservice/ListProducts or GetProduct
  -> recommendationservice returns recommended products
  -> frontend renders recommendations
```

Code references:

| Step | Code reference | What happens |
| --- | --- | --- |
| Frontend recommendation call | `src/frontend/rpc.go` | Calls recommendation gRPC service. |
| Recommendation server startup | `src/recommendationservice/recommendation_server.py` | Requires `PRODUCT_CATALOG_SERVICE_ADDR`. |
| Product catalog client in recommendation | `src/recommendationservice/recommendation_server.py` | Recommendation service calls product catalog. |
| Product data | `src/productcatalogservice/product_catalog.go` | Supplies products used for recommendations. |

Direct gRPC test:

```bash
grpcurl -plaintext \
  -proto protos/demo.proto \
  -d '{"user_id":"test-user","product_ids":["OLJCESPC7Z"]}' \
  localhost:8082 \
  hipstershop.RecommendationService/ListRecommendations
```

## Flow 7: Load Generator Simulates Users

The load generator is not part of the customer-facing app. It is a traffic client.

High-level flow:

```text
loadgenerator
  -> frontend HTTP routes
  -> frontend calls backend gRPC services
  -> backend services respond
```

Code reference:

```text
src/loadgenerator/locustfile.py
```

Run locally:

```bash
python -m locust -f locustfile.py --host=http://localhost:8088 --headless -u 10 -r 1
```

The load generator does not call product catalog, checkout, cart, or ad service directly. It behaves like a user hitting the frontend.

## Service Communication Matrix

| Caller | Callee | Protocol | Address source |
| --- | --- | --- | --- |
| Browser | `frontend` | HTTP | Public/local frontend URL |
| `frontend` | `productcatalogservice` | gRPC | `PRODUCT_CATALOG_SERVICE_ADDR` |
| `frontend` | `currencyservice` | gRPC | `CURRENCY_SERVICE_ADDR` |
| `frontend` | `cartservice` | gRPC | `CART_SERVICE_ADDR` |
| `frontend` | `recommendationservice` | gRPC | `RECOMMENDATION_SERVICE_ADDR` |
| `frontend` | `shippingservice` | gRPC | `SHIPPING_SERVICE_ADDR` |
| `frontend` | `checkoutservice` | gRPC | `CHECKOUT_SERVICE_ADDR` |
| `frontend` | `adservice` | gRPC | `AD_SERVICE_ADDR` |
| `frontend` | `shoppingassistantservice` | HTTP/gRPC depending on implementation | `SHOPPING_ASSISTANT_SERVICE_ADDR` |
| `checkoutservice` | `productcatalogservice` | gRPC | `PRODUCT_CATALOG_SERVICE_ADDR` |
| `checkoutservice` | `shippingservice` | gRPC | `SHIPPING_SERVICE_ADDR` |
| `checkoutservice` | `paymentservice` | gRPC | `PAYMENT_SERVICE_ADDR` |
| `checkoutservice` | `emailservice` | gRPC | `EMAIL_SERVICE_ADDR` |
| `checkoutservice` | `currencyservice` | gRPC | `CURRENCY_SERVICE_ADDR` |
| `checkoutservice` | `cartservice` | gRPC | `CART_SERVICE_ADDR` |
| `recommendationservice` | `productcatalogservice` | gRPC | `PRODUCT_CATALOG_SERVICE_ADDR` |
| `cartservice` | Redis | TCP | `REDIS_ADDR` |
| `loadgenerator` | `frontend` | HTTP | `FRONTEND_ADDR` or Locust `--host` |

## How To Read The Code Like A Platform Engineer

Use this method for each service:

1. Start with `protos/demo.proto`.
2. Find the service definition.
3. Find the service implementation file.
4. Find where the server listens on a port.
5. Find where env vars are read.
6. Find where clients are created for downstream services.
7. Map the required env vars into local or Kubernetes service addresses.

Example for frontend:

```text
protos/demo.proto
  -> defines backend services frontend will call
src/frontend/main.go
  -> reads backend addresses from env vars
src/frontend/rpc.go
  -> creates gRPC clients and calls backends
src/frontend/handlers.go
  -> receives browser requests and triggers RPC calls
```

Example for checkout:

```text
protos/demo.proto
  -> defines CheckoutService and dependency service contracts
src/checkoutservice/main.go
  -> reads dependency env vars
src/checkoutservice/main.go
  -> creates clients to cart/product/currency/shipping/payment/email
src/checkoutservice/main.go
  -> PlaceOrder coordinates the full order flow
```

## Local vs Kubernetes Addressing

Local:

```text
PRODUCT_CATALOG_SERVICE_ADDR=localhost:3550
CURRENCY_SERVICE_ADDR=localhost:7001
CART_SERVICE_ADDR=localhost:5000
```

Kubernetes:

```text
PRODUCT_CATALOG_SERVICE_ADDR=productcatalogservice:3550
CURRENCY_SERVICE_ADDR=currencyservice:7000
CART_SERVICE_ADDR=cartservice:7070
```

Important:

```text
localhost inside a container or pod means that same container/pod,
not another service.
```

That is why cluster manifests use service DNS names instead of `localhost`.

## Mental Model

Use this mental model:

```text
HTTP routes are mostly at the frontend.
Business capabilities are split into backend gRPC services.
The protobuf file defines the language-neutral contract.
Each service runs its own server.
Callers create clients and call services directly.
Kubernetes/ECS service discovery gives those clients stable names.
```

So the app is not:

```text
frontend -> central gRPC broker -> all services
```

It is:

```text
frontend -> direct gRPC call -> target service
checkoutservice -> direct gRPC calls -> dependency services
recommendationservice -> direct gRPC call -> productcatalogservice
```

## Practical Debug Rule

When a user flow fails, trace the request hop by hop.

Example checkout failure:

```text
browser
  -> frontend
  -> checkoutservice
  -> cartservice
  -> productcatalogservice
  -> currencyservice
  -> shippingservice
  -> paymentservice
  -> emailservice
```

At each hop, verify:

- Is the target service running?
- Is the caller using the correct env var?
- Is the target address correct?
- Is the target port listening?
- Does gRPC health pass?
- Are logs showing connection refused, unavailable, timeout, or bad request?

That is how we found the checkout issue during local testing: checkout had stale addresses for cart, currency, and email.
