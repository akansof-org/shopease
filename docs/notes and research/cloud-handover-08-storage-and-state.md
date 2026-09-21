# Cloud Handover Checklist: 8. Storage And State

Scenario: the development team has handed this application to the platform engineering team. We are preparing it for cloud infrastructure, instrumentation, and deployment to Kubernetes or ECS.

This note covers the eighth checklist item only: **Storage And State**.

## Objective

Stateful components need special attention.

This stage answers:

- What data does the app store?
- Which components are databases, caches, queues, or persistent volumes?
- Is the state temporary or source-of-truth?
- Can the data be lost?
- Is a managed service preferred over in-cluster storage?
- Are migrations required before deployment?
- What happens during pod/task restart?
- What is the backup and restore plan?

## Checklist

- [x] Identify databases, caches, queues, and persistent volumes.
- [ ] Decide managed service vs in-cluster service.
- [ ] Confirm persistence requirements.
- [ ] Confirm backup and restore plan.
- [x] Confirm migration strategy.
- [ ] Confirm data retention requirements.
- [ ] Confirm storage class.
- [ ] Confirm volume size.
- [ ] Confirm access mode.
- [x] Confirm what happens during pod/task restart.

Note:

- The default app uses in-cluster Redis for cart state.
- The default Redis storage is ephemeral.
- Optional Kustomize components support managed Redis/Memorystore, Spanner, and AlloyDB.
- The default deployment does not define PersistentVolumeClaims.

## Questions To Answer

### Can this data be lost?

For the default demo deployment:

```text
Yes, cart data can be lost.
```

Reason:

- `redis-cart` stores data on an `emptyDir` volume.
- `emptyDir` is deleted when the Redis pod is deleted or rescheduled.
- Cart data is not treated as durable source-of-truth in the default demo.

For production:

```text
This must be a product/platform decision.
```

If cart state matters for user experience or business continuity, move it to managed Redis, Spanner, AlloyDB, or another durable store.

### Does the app need migrations before deploy?

Default deployment:

```text
No migrations required.
```

Optional storage integrations:

```text
Yes, setup/migrations are required.
```

Examples:

- Spanner requires `CartItems` table and index.
- AlloyDB requires database, cart table, index, secret, and network connectivity.
- Shopping assistant requires AlloyDB tables populated with product/vector data.

### Is this cache or source-of-truth storage?

Default deployment:

```text
redis-cart behaves like temporary session/cart cache.
```

Production decision:

- If cart contents are allowed to disappear, Redis cache may be fine.
- If cart contents must survive restarts, choose a durable or managed storage design.

### What is the recovery process?

Default deployment:

```text
Restart/recreate Redis and accept cart data loss.
```

Production:

- Restore from managed Redis backup/snapshot if available.
- Restore Spanner/AlloyDB from backups if using database-backed cart state.
- Rebuild optional product catalog/vector tables from source data if documented.

## Storage Inventory

| Component | Storage/state | Required by default? | Storage type | Persistence | Notes |
| --- | --- | --- | --- | --- | --- |
| `redis-cart` | Cart data | Yes | Redis | Ephemeral by default | Uses `emptyDir`; data lost when pod is recreated. |
| `cartservice` | Reads/writes cart data | Yes | Client of Redis/Spanner/AlloyDB/in-memory | Depends on selected backend | Default uses Redis via `REDIS_ADDR`. |
| `productcatalogservice` | Product catalog | Yes | Local JSON file by default | Baked into image | `products.json` is packaged in the image. |
| `productcatalogservice` | Product catalog in AlloyDB | No | AlloyDB | Durable managed DB | Enabled if AlloyDB env vars are set. |
| `cartservice` | Cart data in Spanner | No | Spanner | Durable managed DB | Optional Kustomize component. |
| `cartservice` | Cart data in AlloyDB | No | AlloyDB | Durable managed DB | Optional Kustomize component. |
| `shoppingassistantservice` | Product embeddings/vector data | No | AlloyDB vector store | Durable managed DB | Optional AI assistant feature. |
| `loadgenerator` | None | No | n/a | n/a | Stateless traffic generator. |
| Other app services | None | Yes | n/a | n/a | Stateless services. |

## Default Storage Architecture

Default path:

```text
frontend
  -> cartservice
  -> redis-cart
  -> emptyDir volume mounted at /data
```

Kubernetes default:

```yaml
volumes:
- name: redis-data
  emptyDir: {}
```

Impact:

- Redis data exists only for the lifetime of the pod.
- Pod restart may preserve data only if the same pod sandbox survives, but pod deletion/rescheduling loses data.
- Node drain, rollout, eviction, or manual delete can lose cart data.

Platform interpretation:

```text
This is demo-grade storage, not production-grade persistent storage.
```

## Cart Storage Selection Logic

`cartservice` chooses storage based on runtime configuration.

Priority order:

1. If `REDIS_ADDR` is set, use Redis.
2. Else if `SPANNER_PROJECT` or `SPANNER_CONNECTION_STRING` is set, use Spanner.
3. Else if `ALLOYDB_PRIMARY_IP` is set, use AlloyDB.
4. Else use in-memory storage.

Platform note:

```text
In-memory cart storage should only be used for local learning/testing.
It is not suitable for clustered deployment because data is per pod and disappears on restart.
```

## Managed Service vs In-Cluster Service

| Option | Pros | Cons | Recommended use |
| --- | --- | --- | --- |
| In-cluster Redis with `emptyDir` | Simple, cheap, fast demo setup | Data loss on pod recreation; no backup; no auth by default | Local/dev/demo only |
| In-cluster Redis with PVC | More durable than `emptyDir` | You operate Redis; failover/backup complexity | Small non-critical environments |
| Managed Redis | Operationally simpler; better availability; backup/security options | Cloud cost; network/IAM setup | Production-like cart cache |
| Spanner | Durable, scalable, managed database | Schema setup; cloud-specific; more cost/complexity | Durable cart storage on GCP |
| AlloyDB | Durable managed PostgreSQL-compatible storage | Network/secret/schema setup; cloud-specific | Durable cart storage or product/vector data |

Recommended production direction:

```text
Use a managed service for state whenever possible.
For this app, managed Redis is the closest replacement for the default Redis path.
```

## Kubernetes Persistent Volume Status

Default manifests:

| Object | Present? | Notes |
| --- | --- | --- |
| PersistentVolumeClaim | No | No PVC is defined by default. |
| StorageClass | No | No storage class is referenced by default. |
| StatefulSet | No | Redis runs as Deployment. |
| `emptyDir` | Yes | Used by `redis-cart`. |

If choosing in-cluster persistent Redis, define:

- StatefulSet or Redis operator/Helm chart.
- PersistentVolumeClaim.
- StorageClass.
- Volume size.
- Access mode, usually `ReadWriteOnce`.
- Backup/restore process.

Example PVC skeleton:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
  storageClassName: standard
```

## Optional Storage Components In This Repo

The repo includes Kustomize components for storage alternatives.

| Component | Purpose | Effect |
| --- | --- | --- |
| `kustomize/components/memorystore` | Use Google Cloud Memorystore Redis | Removes in-cluster `redis-cart`, points `cartservice` to managed Redis. |
| `kustomize/components/spanner` | Use Cloud Spanner for cart storage | Removes in-cluster Redis, configures `cartservice` Spanner env vars. |
| `kustomize/components/alloydb` | Use AlloyDB for cart storage | Removes in-cluster Redis, configures `cartservice` AlloyDB env vars. |
| `kustomize/components/shopping-assistant` | Add AI assistant backed by AlloyDB vector data | Adds assistant service and requires AlloyDB/product vector setup. |

Enable a component from `kustomize/`:

```sh
kustomize edit add component components/memorystore
```

Then render/apply:

```sh
kubectl kustomize .
kubectl apply -k .
```

## Migration And Setup Requirements

### Default Redis

Required setup:

- None beyond deploying manifests.

Migration:

- None.

Backup:

- None by default.

Recovery:

- Recreate Redis and accept cart loss.

### Managed Redis / Memorystore

Required setup:

- Provision managed Redis.
- Ensure cluster/network can reach Redis.
- Update `REDIS_ADDR`.
- Remove in-cluster Redis if not needed.

Migration:

- Usually none for cart cache.
- If preserving carts is required, design migration separately.

Backup:

- Use provider-supported backup/snapshot if available and required.

Recovery:

- Restore managed Redis or recreate and accept cart loss based on business requirement.

### Spanner

Required setup:

- Enable Spanner API.
- Create Spanner instance.
- Create database.
- Create `CartItems` table.
- Create `CartItemsByUserId` index.
- Configure workload identity/IAM for `cartservice`.

Schema from repo docs:

```sql
CREATE TABLE CartItems (
  userId STRING(1024),
  productId STRING(1024),
  quantity INT64
) PRIMARY KEY (userId, productId);

CREATE INDEX CartItemsByUserId ON CartItems(userId);
```

Migration:

- Required if moving existing cart data from Redis to Spanner.
- For new environment, schema creation is enough.

Backup:

- Use Spanner backup/restore and retention policy.

### AlloyDB For Cart

Required setup:

- Enable AlloyDB, Service Networking, and Secret Manager APIs.
- Create AlloyDB cluster and instance.
- Create database.
- Create cart table and index.
- Store DB password in Secret Manager.
- Configure workload identity/IAM for `cartservice`.
- Configure private network connectivity.

Schema from repo docs:

```sql
CREATE DATABASE carts;

CREATE TABLE cart_items (
  userId text,
  productId text,
  quantity int,
  PRIMARY KEY(userId, productId)
);

CREATE INDEX cartItemsByUserId ON cart_items(userId);
```

Migration:

- Required if moving existing cart data from Redis.
- For new environment, schema creation is enough.

Backup:

- Use AlloyDB backup policy.
- Note that the repo setup example disables automated backup; production should revisit that.

### Product Catalog Local JSON

Default:

- Product catalog is loaded from `products.json`.
- File is copied into the product catalog image.

Persistence:

- Static image artifact, not runtime storage.

Migration:

- None.

Operational note:

```text
Updating products.json requires rebuilding and redeploying productcatalogservice.
```

### Product Catalog AlloyDB

Optional:

- If product catalog AlloyDB env vars are set, `productcatalogservice` loads products from AlloyDB instead of local JSON.

Required setup:

- AlloyDB database and table.
- Secret Manager secret for DB password.
- IAM/workload identity.
- Network connectivity.
- Data load process.

Migration:

- Required to load product data into AlloyDB.

### Shopping Assistant Vector Store

Optional:

- Requires AlloyDB-backed product/vector table.
- Requires setup scripts to create/populate tables.
- Requires Google Generative AI/Gemini access.

Migration/setup:

- Required before enabling assistant.

Platform note:

```text
Do not enable shopping assistant until its database, embeddings/vector data, secret access, and API credentials are configured.
```

## Data Classification

| Data | Stored in | Classification | Can lose? | Notes |
| --- | --- | --- | --- | --- |
| Shopping cart | Redis by default | User session/cart state | Demo: yes. Prod: decide. | Main stateful data in default app. |
| Product catalog | `products.json` image file | Reference/catalog data | Not runtime-generated | Rebuild image to change default catalog. |
| Product catalog DB copy | AlloyDB optional | Reference/catalog data | No, if DB source-of-truth | Optional mode. |
| Product embeddings | AlloyDB optional | Derived AI/RAG data | Usually regenerable | Requires generation/population process. |
| Orders | Not durably stored | Transaction response only | n/a | Checkout returns order but no order DB in default app. |
| Payment transactions | Mock response | Not durably stored | n/a | Payment service is mock. |
| Emails | Mock/dummy send | Not durably stored | n/a | Email service logs dummy confirmation. |

Important:

```text
This app does not include a durable order database.
Checkout creates an order response but does not persist order history.
```

## Backup And Restore Decisions

Fill this table for each environment.

| Storage | Backup required? | Backup method | Restore method | RPO | RTO | Owner |
| --- | --- | --- | --- | --- | --- | --- |
| `redis-cart` default | No for demo | None | Recreate pod | n/a | n/a | Platform |
| Managed Redis | Decide | Provider snapshot/backup | Provider restore |  |  |  |
| Spanner | Yes if used | Spanner backup | Spanner restore |  |  |  |
| AlloyDB | Yes if used | AlloyDB backup | AlloyDB restore |  |  |  |
| Assistant vector data | Decide | DB backup or regenerate | Restore/regenerate |  |  |  |

## Retention Decisions

Questions:

- How long should cart data live?
- Should carts expire automatically?
- Should abandoned carts be retained?
- Is cart data personal data?
- Does any data fall under compliance requirements?
- Should Redis keys have TTL?

Default app behavior:

- Cart storage does not clearly define business retention in manifests.
- Redis default persistence/TTL policy is not configured by Kubernetes manifests.

Platform/product follow-up:

```text
Get product decision on cart retention before production.
```

## What Happens During Restart

| Component | Restart behavior |
| --- | --- |
| Stateless app services | Safe to restart; no local persistent state expected. |
| `frontend` | Session ID is cookie-based; restart should not lose server-side app data. |
| `cartservice` | Safe if backing store survives; loses carts if using in-memory fallback. |
| `redis-cart` default | Cart data can be lost if pod is recreated/rescheduled because storage is `emptyDir`. |
| `productcatalogservice` | Reloads product catalog from image file or optional AlloyDB. |
| `loadgenerator` | Restarts traffic generation. |
| `shoppingassistantservice` | Depends on external AlloyDB/API availability. |

## Kubernetes Validation Commands

Check volumes:

```sh
kubectl describe deployment redis-cart
```

Look for:

```text
Volumes:
  redis-data:
    Type: EmptyDir
```

Check PVCs:

```sh
kubectl get pvc
```

Default expectation:

```text
No resources found
```

Check Redis pod restart history:

```sh
kubectl get pods -l app=redis-cart
kubectl describe pod -l app=redis-cart
```

Check cartservice storage config:

```sh
kubectl set env deployment/cartservice --list
```

Expected default:

```text
REDIS_ADDR=redis-cart:6379
```

## ECS Storage Translation

If deploying to ECS:

| Storage need | ECS/AWS option |
| --- | --- |
| Cart cache | ElastiCache Redis |
| Durable cart DB | DynamoDB, Aurora/PostgreSQL, RDS, or another DB |
| Shared persistent file storage | EFS |
| Secrets | AWS Secrets Manager or SSM Parameter Store |
| Product catalog static file | Baked into image or externalized to S3/DB |

ECS recommendation:

```text
Do not run Redis as an ordinary ECS task for production unless you are intentionally operating Redis yourself.
Use ElastiCache or another managed store.
```

## Storage And State Risks

| Area | Risk | Follow-up |
| --- | --- | --- |
| Redis `emptyDir` | Cart data lost on pod recreation | Decide if acceptable; use managed Redis or PVC if not. |
| No order DB | Checkout/order history is not persisted | Confirm product requirements. |
| In-memory cart fallback | Data is per pod and temporary | Ensure clustered deployments set real storage env vars. |
| No Redis auth | Default Redis is unauthenticated | Harden or use managed Redis for production. |
| No PVC | No durable in-cluster state | Add PVC/StatefulSet only if operating in-cluster storage. |
| Optional DB schemas | Spanner/AlloyDB need schema setup | Add migrations/setup jobs or IaC steps. |
| AlloyDB backup example | Repo setup example disables automated backup | Enable production backup policy. |
| Shopping assistant data | Requires populated vector/product tables | Run setup scripts before enabling feature. |

## Storage And State Output

At the end of this stage, platform engineering should have:

- [x] Stateful components identified.
- [x] Default Redis storage behavior documented.
- [x] Optional managed storage paths documented.
- [x] Persistence risks documented.
- [x] Migration/setup needs documented.
- [x] Restart behavior documented.
- [ ] Managed vs in-cluster storage decision finalized.
- [ ] Backup/restore plan finalized.
- [ ] Data retention requirements confirmed.
- [ ] StorageClass/PVC strategy finalized if needed.
- [ ] Production Redis/database security model defined.
- [ ] Optional storage setup automated through IaC or migration jobs.

## Ready For Next Checklist Item

The next stage is:

```text
9. Networking And Exposure
```

In that stage, define public entrypoints, private service boundaries, ingress/load balancer design, TLS/DNS, network policies, and cloud network requirements for managed storage.

