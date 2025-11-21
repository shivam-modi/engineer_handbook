# D. Distributed Systems & Microservices (Deep Architecture & Theory)

This chapter includes:

- Distributed systems realities  
- CAP, PACELC (real-world understanding)  
- Consistency models (linearizable, sequential, causal, eventual)  
- Microservices boundaries (DDD-driven)  
- Inter-service communication patterns  
- Retries, backpressure, hedging, load shedding  
- Idempotency + exactly-once semantics (real truth)  
- Distributed locks  
- Kafka internals (leader election, ISR, offsets)  
- Event-driven architecture  
- Sagas (choreography & orchestration)  
- Outbox pattern  
- Event sourcing  
- Time & clock drift  
- Fault tolerance patterns  
- Circuit breakers  
- Async/reactive flows  
- Real production failure patterns  

---

# 1. DISTRIBUTED SYSTEM REALITIES (FALLACIES OF DISTRIBUTED COMPUTING)

These are the **8 fallacies** every engineer must internalize:

1. The network is reliable  
2. Latency is zero  
3. Bandwidth is infinite  
4. The network is secure  
5. Topology doesn’t change  
6. There is one administrator  
7. Transport cost is zero  
8. The network is homogeneous  

## What this means:

- Always expect failure  
- Always assume latency variability  
- Always isolate blast radius  
- Always implement graceful degradation  

Distributed systems are systems where **things fail all the time**.

---

# 2. CAP THEOREM (REAL-WORLD UNDERSTANDING)

```
Consistency  (C)
Availability (A)
Partition Tolerance (P)
```

You can choose **only two** during a partition:

| System Type | Example |
|-------------|----------|
| CP | ZooKeeper, etcd |
| AP | DynamoDB, Cassandra |
| CA | (Only possible without partitions — unrealistic) |

### CAP Diagram

```
              Consistency
                 /\
                /  \
               /    \
              /------\
     Partition|        |Availability
```

---

# 3. PACELC (REAL WORLD RULE)

CAP only describes partitions.  
PACELC extends the model:

```
If Partition:
    Choose Availability or Consistency
Else (normal condition):
    Choose Latency or Consistency
```

Examples:

```
DynamoDB → PA/EL system
Cassandra → PA/EL
Spanner → PC/EC
Etcd → PC/EC
```

---

# 4. CONSISTENCY MODELS (MUST KNOW)

## 1. Linearizable (strongest)
Real-time constraint: reads reflect the most recent write.

Example:  
- etcd  
- Zookeeper  

## 2. Sequential Consistency  
All nodes see operations in the same order, but not tied to real-time.

## 3. Causal Consistency  
Preserves cause → effect relationships.

Example:  
- Dynamo-style systems  

## 4. Eventual Consistency  
All nodes converge eventually.

Used by:
- Cassandra  
- DynamoDB  

---

# 5. MICROSERVICE BOUNDARIES — THE REAL RULE

Microservices should be aligned with **bounded contexts**.

### BAD boundaries (anti-pattern):
- UserService  
- OrderService  
- PaymentService  
- ReportService  

This is CRUD-layer splitting.

### GOOD boundaries:
- Inventory (stock, reservations, holds)  
- Payments (money movement)  
- Orders (order workflow, state machine)  
- Catalog (products, prices, attributes)  
- Shipping (fulfillment, tracking)  

---

# 6. INTERSERVICE COMMUNICATION

## 6.1 Synchronous (blocking)
### gRPC / HTTP

Used when:
- You need immediate result  
- Low latency on LAN  
- Workflow is short  

**Risks:**
- Cascading failure  
- Latency spikes  

## 6.2 Asynchronous (non-blocking)
### Kafka / SQS / PubSub / Kinesis

Used when:
- Loose coupling required  
- Retry-friendly  
- Large fan-out  

---

# 6.3 Communication Diagram

```
Client
  ↓
API Gateway
  ↓ Sync (HTTP)
Service A
  ↓ Event Publish (Kafka)
Service B
  ↓
Service C
```

---

# 7. TIMEOUTS, RETRIES, HEDGING, BACKOFF

## 7.1 Timeouts  
**Non-negotiable rule:** all network calls MUST have timeouts.

```
Default HTTP client timeout = DEAD
```

## 7.2 Retries

### Dangers:
- retry storms  
- thundering herd  
- duplicated writes  

### Always use:
- exponential backoff  
- jitter  
- retry budgets  

## 7.3 Hedged Requests
Send a duplicate request if first is too slow.

Used by Google, AWS DynamoDB.

## 7.4 Backpressure
Slow down senders when consumers are overloaded.

Reactive systems use:
- bounded queues  
- dropping  
- delaying  

---

# 8. IDEMPOTENCY (MUST KNOW)

Idempotent operation = multiple identical requests → same effect.

Examples:
- PUT /resource  
- DELETE /resource  

Not idempotent:
- POST /resource  
- Charge payment  
- Send email  

### How to enforce idempotency:

```
Idempotency-Key: <uuid>
```

Store the key in a database or Redis.

---

# 9. EXACTLY-ONCE VS AT-LEAST-ONCE DELIVERY

Truth:

```
Exactly-once delivery does not exist in distributed systems.
Only exactly-once processing is achievable through:
    - idempotency
    - deduplication
    - deterministic workflows
```

Kafka provides:
- at-least-once by default  
- exactly-once processing = idempotent producers + transactions  

---

# 10. DISTRIBUTED LOCKS

Use:
- Redis Redlock (with caveats)  
- etcd locks  
- Zookeeper locks  

Best practice:
- Avoid distributed locks if possible  
- Model with idempotency & CAS instead  

---

# 11. CLOCK DRIFT & TIME ISSUES

Distributed systems require **NTP synchronization**.

If not:
- tokens expire early  
- logs reorder  
- inconsistent ordering  
- billing incorrect  

---

# 12. KAFKA INTERNALS (VERY DEEP)

## 12.1 Partitions

Each topic consists of partitions:

```
Partition 0: messages 0,1,2,3
Partition 1: messages 0,1,2
```

Only **within a partition** ordering is guaranteed.

---

## 12.2 Leader/Follower Replication

```
         ┌──────────┐
         │ Leader    │
         └─────┬────┘
               │ replicate
       ┌───────▼────────┐
       │ F1    │   F2    │
       └───────┴────────┘
```

Followers pull from leader.

---

## 12.3 ISR (In-Sync Replicas)

Messages are “committed” when all ISR replicas have written them.

---

## 12.4 Consumer Group Deep Dive

```
Group Coordinator manages:
- partition assignment
- consumer heartbeats
- rebalancing
```

Rebalancing causes:
- no message consumption  
- duplicate processing from last committed offset  

### Use cooperative rebalancing for smooth transitions.

---

# 13. EVENT-DRIVEN ARCHITECTURE (EDA)

## EDA Pipeline Diagram

```
Producer
  → Kafka Topic
     → Consumer Group
        → Processing
           → Downstream Events
```

---

# 14. SAGAS — DISTRIBUTED WORKFLOW COORDINATION

Two patterns:

---

## 14.1 Choreography Saga (event-based)

```
Order Service     → OrderCreated
   Inventory Service → StockReserved
      Payment Service → PaymentCompleted
```

Pros:
- No central orchestrator  
- Naturally scalable  

Cons:
- Hard to visualize  
- Hard to manage failure compensation  

---

## 14.2 Orchestration Saga (central coordinator)

```
     ┌─────────────────────┐
     │  Saga Coordinator   │
     └───────┬────────────┘
             │ instruct
             ▼
        Order Service
             │
        Payment Service
             │
        Inventory Service
```

Pros:
- Clear flow  
- Easier compensations  
- Better failure reporting  

Cons:
- Coordinator is a central point  

---

# 15. OUTBOX PATTERN (MUST KNOW)

Solve:  
**DB write + event publish must be atomic.**

### How it works:

```
Write business data to DB
Write event to OUTBOX table in same transaction
Publisher reads OUTBOX table
Publishes to Kafka
Marks outbox row as processed
```

Guarantees:
- no lost messages  
- no duplicates (use dedup)

---

# 16. EVENT SOURCING (DEEP)

Instead of storing current state:

Store events:

```
UserCreated
EmailUpdated
SubscriptionAdded
```

Then reconstruct state:

```
Fold events → aggregate state
```

Benefits:
- full audit  
- replay  
- time travel  
- perfect append-only log  

Challenges:
- snapshotting  
- schema evolution  
- projections  

---

# 17. SERVICE DISCOVERY

Use:

- Eureka  
- Consul  
- Kubernetes DNS  
- Envoy + xDS  
- AWS CloudMap  

Clients talk using:

```
service-name.namespace.svc.cluster.local
```

---

# 18. LOAD BALANCING PATTERNS

- Client-side LB (Ribbon, Spring Cloud LoadBalancer)  
- Server-side LB (NGINX, Envoy)  
- Global LB (AWS ALB/NLB, Cloudflare, GCP LB)  

---

# 19. FAULT TOLERANCE PATTERNS (ADVANCED)

### 19.1 Circuit Breakers

```
closed → open → half-open
```

### 19.2 Bulkheads

Isolate system resources.

### 19.3 Rate limiting

- token bucket  
- leaky bucket  

### 19.4 Load shedding

Drop low-priority traffic.

---

# 20. CONSISTENCY BETWEEN MICROSERVICES

Use:

- Outbox  
- Idempotent handlers  
- Event versioning  
- Eventual consistency with retries  
- Distributed transaction with saga  

---

# 21. VERSIONING EVENTS & SCHEMAS

Use:

- Schema registry  
- Backward compatibility  
- Forward compatibility  

---

# 22. REAL-WORLD FAILURE MODES

- Message reorder  
- Duplicate deliveries  
- Partial failures  
- Zombie consumers  
- Split-brain  
- Network partitions  
- Leader election storms  
- Long GC pauses → false node death  
- Slow consumers causing lag  
- Head-of-line blocking  

---

# 23. MICROSERVICE ANTI-PATTERNS

❌ Distributed Monolith  
❌ Chatty services  
❌ Sync chains deeper than 2 hops  
❌ Shared database  
❌ Too many tiny microservices  
❌ Entity-based boundaries (UserService everywhere)  
❌ No idempotency  
❌ No retry budgets  

---

# 24. MICROSERVICE BEST PRACTICES

✔ API contracts  
✔ Backward-compatible event evolution  
✔ Idempotent handlers  
✔ Outbox  
✔ Well-defined bounded contexts  
✔ No shared DB  
✔ Limits, timeouts, retries  
✔ Circuit breakers  
✔ Observability (traces, logs, metrics)  
✔ Blue/green, canary deployments  
✔ Chaos testing  

---

# END OF CHAPTER D
