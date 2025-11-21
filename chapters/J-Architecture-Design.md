# J. Architecture Design (Principal Engineer Depth)

This chapter covers:

- Monolith vs Microservices vs Modular Monoliths  
- Event-driven architecture  
- Sagas vs Orchestrators  
- CQRS (Command Query Responsibility Segregation)  
- Event sourcing  
- API gateway patterns  
- Data mesh  
- Scalability patterns  
- Resiliency patterns  
- Multi-region & global-scale architecture  
- Anti-patterns

---

# 1. ARCHITECTURE PHILOSOPHY (REAL REALITY)

Architecture is **not**:

- drawing boxes
- trendy microservices
- Kubernetes buzzwords

Architecture **is**:

- choosing trade-offs  
- minimizing complexity  
- aligning systems with business domains  
- providing safety + speed  

---

# 2. MONOLITH VS MICROSERVICES

## 2.1 Monolith

### Pros:
- simple debugging  
- simple deployment  
- strong consistency  
- no network overhead  
- easy local dev  

### Cons:
- scaling EVERYTHING together  
- large codebase → build time slow  
- harder to enforce boundaries  

---

## 2.2 Modular Monolith (BEST DEFAULT)

It’s like:

**“Microservices inside a monolith.”**

Each module = independent:

- bounded context  
- domain-owned data  
- no cross-module DB queries  
- separate packages  
- strict contracts  

Often the **best architecture for most companies under 200 engineers**.

---

## 2.3 Microservices

### Pros:
- independent deployability  
- scale only hot components  
- polyglot freedom  
- failure isolation  
- team autonomy  

### Cons:
- distributed transactions  
- operational overhead  
- requires observability everywhere  
- higher latency  
- requires strong infra maturity  

---

# 3. MICROSERVICE BOUNDARIES (DDD)

Use **Bounded Contexts**:

- Payments  
- Orders  
- Shipping  
- Catalog  
- Inventory  

Not:

- UserService  
- NotificationService (just a utility)  

Domain modeling drives architecture.

---

# 4. INTERSERVICE COMMUNICATION PATTERNS

### Sync (HTTP/gRPC)
- request-response  
- strong consistency  
- immediate feedback  

### Async (Kafka/SQS)
- eventual consistency  
- decoupling  
- high throughput  

RULE OF THUMB:

```
Sync = commands
Async = events
```

---

# 5. API GATEWAY PATTERNS

API Gateway provides:

- routing  
- rate limiting  
- authentication  
- request transformation  
- versioning  

Patterns:

- BFF (Backend For Frontend)  
- Edge Services  
- Aggregators  

---

# 6. EVENT-DRIVEN ARCHITECTURE

Events should represent **facts**:

```
OrderPlaced
PaymentAuthorized
StockReserved
OrderShipped
```

Avoid events like:
```
OrderUpdated
ItemChanged
```

Too vague → hard to reason about.

---

# 7. SAGA PATTERNS (DISTRIBUTED WORKFLOWS)

## 7.1 Choreography Saga

```
Order Service → OrderCreated
Inventory Service → StockReserved
Payment Service → PaymentCaptured
Shipping Service → ShipmentCreated
```

### Pros:
- scalable  
- no central coordinator  
- simple at small scale  

### Cons:
- spaghetti events  
- hard debugging  
- hard compensation  
- unclear workflow  

---

## 7.2 Orchestration Saga

Central orchestrator:

```
Orchestrator → ReserveStock()
              → CapturePayment()
              → CreateShipment()
```

### Pros:
- clear workflow  
- easier compensation  
- central visibility  

### Cons:
- orchestrator becomes critical component  

---

# 8. CQRS (Command Query Responsibility Segregation)

Separate:

- write model (commands)  
- read model (queries)  

Used when:

- read scalability needed  
- write paths are complex  
- event sourcing is used  

---

### Example:

Write DB:
```
INSERT order_events
UPDATE order_state
```

Read DB:
```
orders_view (denormalized read model)
```

---

# 9. EVENT SOURCING (DEEP)

Instead of storing state:

Store **events**:

```
OrderCreated
OrderPaid
OrderShipped
OrderDelivered
```

Rebuild state by replay.

## Pros:
- audit log  
- time travel  
- immutability  

## Cons:
- event schema evolution  
- rebuilding read models  
- debugging more complex  

---

# 10. DATA MESH (ADVANCED)

Data Mesh = decentralizing analytics ownership.

Principles:

- domain-owned data  
- data products  
- discoverability via catalogs  
- self-serve platform  
- federated governance  

Useful only when company > 300–500 engineers.

---

# 11. MULTI-REGION ARCHITECTURE (DEEP)

Patterns:

### 1. Active/Passive
Primary region handles all traffic.
Secondary region only DR.

### 2. Active/Active
Both regions handle real traffic.

Challenges:
- global cache invalidation  
- consistency  
- clock drift  
- replication lag  
- conflict resolution  

---

# 12. SCALABILITY PATTERNS

### 12.1 Vertical Scaling
Add more CPU/RAM.

### 12.2 Horizontal Scaling
Add more instances.

---

## 12.3 Patterns for massive scaling:

- Sharding  
- Partitioning  
- Load balancing  
- Caching (L1 + L2 + CDN)  
- Batching  
- Event-driven pipelines  
- Idempotent processing  
- Stateless compute (12-factor)  

---

# 13. RESILIENCY PATTERNS (MUST KNOW)

- Circuit breakers  
- Bulkheads  
- Load shedding  
- Rate limiting  
- Retry + backoff  
- Hedged requests  
- Chaos testing  

---

# 14. API VERSIONING STRATEGIES

### URI versioning:
```
/v1/orders
/v2/orders
```

### Header versioning:
```
Accept: application/vnd.company.v2+json
```

### Backward compatibility is KEY.

---

# 15. DATABASE ARCHITECTURE IN MICROSERVICES

Rule:
**each microservice owns its database.**

Never share the same DB schema.

Options:
- PostgreSQL per service  
- DynamoDB for large-scale key-value workloads  
- Kafka + Debezium for CDC  
- Outbox for events  

---

# 16. CACHING ARCHITECTURE

Multi-layer caching:

```
L1 (in-memory / Caffeine)
L2 (Redis cluster)
CDN (Cloudflare / Akamai)
```

Patterns:
- cache-aside  
- write-through  
- write-behind  
- stale-while-revalidate  

---

# 17. REAL-WORLD ARCHITECTURE DIAGRAM

```
Client → API Gateway → Services
                      ↙        ↘
               Order Service   Payment Service
                   │                │
                   ▼                ▼
         PostgreSQL (Order DB)   PostgreSQL (Payment DB)
                   │                │
                   ▼                ▼
                 Kafka ←────────────┘
```

---

# 18. COMMON ARCHITECTURE ANTI-PATTERNS

❌ distributed monolith  
❌ overusing microservices  
❌ multiple services sharing the same DB  
❌ event storming with no boundaries  
❌ too many sync calls between services  
❌ no idempotency  
❌ too many queues everywhere → chaos  
❌ no observability → blind distributed system  

---

# 19. BEST PRACTICES (PRINCIPAL ENGINEER)

✔ start with monolith or modular monolith  
✔ evolve gradually into microservices  
✔ use DDD for boundaries  
✔ enforce idempotency  
✔ implement outbox pattern  
✔ async events for cross-domain workflows  
✔ reliable event schemas  
✔ use SLOs for design  
✔ align architecture with org structure (Conway’s law)  

---

# END OF CHAPTER J
