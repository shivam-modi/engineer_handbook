# G. Observability, Reliability & SRE Fundamentals (Principal Engineer Depth)

This chapter contains:

- Observability pillars (Metrics, Logs, Traces)
- RED & USE methods
- Distributed tracing internals (OpenTelemetry)
- Metrics design (cardinality issues)
- SLO / SLI / SLA (deep)
- Error budgets
- Alerting best practices
- Reliability patterns
- Incident response + on-call operations
- Chaos engineering
- High availability design
- Real-world production failure patterns

---

# 1. WHAT IS OBSERVABILITY (REAL DEFINITION)

Observability is not “monitoring.”  
Monitoring answers known questions.  
Observability answers **unknown** questions.

Three pillars:

```
Metrics  → What is happening?
Logs     → Why is it happening?
Tracing  → Where is it happening?
```

Modern systems → must support all three.

---

# 2. METRICS (EXTREMELY DEEP)

Metrics must be:

- low cardinality  
- cheap to collect  
- cheap to query  
- aggregated over time  

### High-cardinality example (BAD):

```
user_id="123123" 
session_id="ab1234" 
order_id="5478"
```

This WILL break Prometheus.

---

## 2.1 RED metrics (for services)

For HTTP/gRPC APIs:

```
Rate      - requests per second
Errors    - error ratio
Duration  - latency (p50/p90/p99)
```

---

## 2.2 USE metrics (for infrastructure)

For resources:

```
Utilization     → percentage of time resource is used
Saturation      → queue length/pressure
Errors          → failures
```

Examples:
- CPU utilization  
- Disk queue length  
- TCP retransmissions  
- Kafka consumer lag  

---

## 2.3 Metric Types

- Counter (only increases)
- Gauge (up/down)
- Histogram (latency buckets)
- Summary (not aggregatable across replicas)

Histograms are the standard for latency.

---

# 3. LOGGING (DEEP PRACTICAL)

Logs must be:

- structured (JSON)
- parseable
- context-rich
- correlated with trace IDs

Example:

```json
{
  "ts": "2025-11-25T12:33:22Z",
  "level": "INFO",
  "trace_id": "abc-123",
  "user_id": 42,
  "endpoint": "/checkout"
}
```

### Avoid logs in hot paths → CPU + I/O overhead.

---

# 4. DISTRIBUTED TRACING (OpenTelemetry / Zipkin / Jaeger)

### Why tracing?
Microservices = request fan-out.

Need to see:

- where latency accumulates  
- where errors originate  
- how systems interact  

---

## 4.1 Trace Structure

A **trace** consists of **spans**.

```
Trace: Checkout request
 ├── Span: API Gateway
 ├── Span: Order Service
 │      ├── Span: call Payment
 │      └── Span: call Inventory
 └── Span: Audit publisher
```

---

## 4.2 Context Propagation (VERY IMPORTANT)

Context travels across services through headers:

- `traceparent`
- `tracestate`
- `x-request-id`

Propagation pitfalls:

- lost context in async tasks  
- thread pool boundaries  
- mixing Kotlin coroutines & MDC (use ThreadContextElement)  

---

# 5. SLO, SLI, SLA (DEEP)

### SLI: Service Level Indicator  
*Measured metric*

Examples:
- request success ratio  
- p99 latency  
- consumer lag  

### SLO: Service Level Objective  
*Target for SLIs*

Examples:
```
99.9% requests must succeed over 30 days
p99 < 200ms
99% Kafka lag < 1 minute
```

### SLA: Business contract  
*Legal or financial obligations.*

---

## 5.1 Error Budgets

If SLO = 99.9% uptime:

```
0.1% downtime allowed
= 0.1% * 30 days = 43 minutes / month
```

Error budget is used to:

- throttle releases  
- stop feature deployment  
- allocate time for reliability work  

---

# 6. ALERTING (DEEP)

Alerts must be:

- actionable  
- urgent  
- paging only for user-impacting issues  

### 3 alert categories:

1. **Page Alerts (P1/P2)**  
   - action needed NOW  
   - high severity  
2. **Ticket Alerts**  
   - needs work later  
3. **Dashboard Alerts**  
   - informational  

### Golden rule:

**DON’T PAGE ON SYMPTOMS, PAGE ON USER IMPACT.**

Bad:
```
CPU > 80%
Memory > 70%
```

Good:
```
p99 latency > 300ms for 5 minutes
error rate > 1% for 10 minutes
```

---

# 7. RELIABILITY PATTERNS

## 7.1 Circuit Breakers

```
closed → open → half-open
```

Used to prevent cascading failures.

---

## 7.2 Bulkheads

Resource partitioning:

- separate thread pools  
- separate queues  
- separate memory pools  

---

## 7.3 Rate Limiting

- token bucket  
- leaky bucket  
- sliding window  

---

## 7.4 Load Shedding

Drop low-priority requests if system is overloaded.

---

## 7.5 Retry Budgets

Avoid retry storms.

---

# 8. INCIDENT RESPONSE & ONCALL PRACTICES

## 8.1 Incident lifecycle

```
detect → page → mitigate → root cause analysis → fix → postmortem
```

---

## 8.2 Good Oncall Behavior

- prioritize user impact  
- apply temporary patches  
- do not debug deeply in crisis  
- rollback > fix  
- high-quality postmortems  

---

## 8.3 Postmortem Template

```
What happened?
Timeline (with evidence)
Root cause
Why it wasn’t caught earlier
User impact
Action items
Preventive measures
```

Postmortems must be **blameless**.

---

# 9. CHAOS ENGINEERING (DEEP)

Simulate failures:

- kill pods  
- inject latency  
- drop packets  
- shut down nodes  
- expire certificates  

Use tools:
- Chaos Monkey  
- LitmusChaos  
- AWS Fault Injection Simulator  

---

# 10. HIGH AVAILABILITY DESIGNS

### Multi-AZ (minimum requirement)  
All production databases must run multi-AZ.

### Multi-region (advanced)

Patterns:
- active/passive  
- active/active  
- write-leader replication  
- global load balancers  

---

# 11. REAL PRODUCTION FAILURE PATTERNS ( MUST KNOW )

- Thread pool exhaustion  
- Kafka consumer lag → data loss  
- Redis eviction → lost session data  
- DB connection pool exhaustion  
- GC pauses → node death  
- Memory leak → OOM kill  
- Index bloat → slow queries  
- Incorrect retry logic → cascading outage  
- Clock drift → authentication failures  
- DNS outage → cluster-wide failure  
- Kafka rebalance storm → message delays  
- TLS certificate expiration outages  
- Feature flag misconfiguration  
- Canary failures not caught  
- Network partition between AZs  

---

# 12. OBSERVABILITY IN PRACTICE (STANDARD STACKS)

### 1. Prometheus + Grafana (metrics)
### 2. Loki / ELK (logs)
### 3. Jaeger / Zipkin / Tempo (tracing)
### 4. OpenTelemetry SDKs everywhere

Best practice:

```
Add trace_id to all logs
Add metrics to all critical paths
Add alerts on RED metrics
```

---

# END OF CHAPTER G
