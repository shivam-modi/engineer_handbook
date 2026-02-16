# C. Spring Boot Advanced Internals

This chapter explains:

- @Transactional internals  
- AOP proxying (JDK, CGLIB)  
- Transaction propagation (with diagrams)  
- Isolation levels  
- PersistenceContext & Entity lifecycle  
- Lazy loading internals  
- Hibernate dirty checking  
- N+1 detection  
- Spring threading model  
- Self-invocation trap  
- @Async internals  
- Filter/Interceptor/ArgumentResolver internals  
- ExceptionResolver flow  
- Advanced caching internals  
- Hibernate flush modes  

---

# 1. TRANSACTION MANAGEMENT INTERNALS (VERY DEEP)

Spring uses **AOP** around methods annotated with:

```
@Transactional
```

## 1.1 What @Transactional Actually Does

When you annotate:

```java
@Transactional
public void placeOrder() {
    ...
}
```

Spring:

1. Wraps the bean inside a **proxy**
2. At method entry:  
   - Opens a transaction (via PlatformTransactionManager)
3. Executes target logic
4. At method exit:  
   - Commit OR rollback depending on exceptions

### How does Spring decide commit or rollback?

```
RuntimeException, Error → rollback
Checked Exception → commit (unless configured otherwise)
```

Can override:

```java
@Transactional(rollbackFor = Exception.class)
```

---

# 2. SPRING AOP PROXY INTERNALS

Spring uses proxies to intercept method calls.

There are **TWO types of proxies**:

| Proxy Type | Used When | Notes |
|------------|-----------|-------|
| JDK Dynamic Proxy | Interface present | Only intercepts interface methods |
| CGLIB Proxy | No interface OR forced | Creates subclass of your class |

---

## 2.1 Proxy Diagram

```
Client
  │
  ▼
[ Spring Proxy ]  ← TransactionInterceptor, SecurityInterceptor, Metrics, Logs
  │
  ▼
[ Target Bean Method ]
```

---

## 2.2 Self-Invocation Problem (Critical)

```java
class OrderService {

    @Transactional
    public void place() { ... }

    public void outer() {
        place(); // ❌ NO TRANSACTION !!! (self-invocation)
    }
}
```

Why?

Because call goes directly to the method, bypassing proxy.

### Fix:

- Move transactional method to another bean  
- Inject bean and call via proxy  

---

# 3. TRANSACTION PROPAGATION (DEEP + DIAGRAMS)

Spring supports **7 propagation types**:

```
REQUIRED
REQUIRES_NEW
NESTED
MANDATORY
NEVER
NOT_SUPPORTED
SUPPORTS
```

---

## 3.1 REQUIRED (default)

```
Outer (Tx1)
  └── inner (joins Tx1)
```

Diagram:

```
Tx1: ┌───── outer() ─────┐
     │   inner()         │
     └───────────────────┘
```

- If outer fails → whole transaction fails  
- If inner fails → whole transaction fails  

---

## 3.2 REQUIRES_NEW

Suspends outer transaction, starts a new one.

```
Tx1 (outer)
   ↓ suspend
Tx2 (inner)
   ↓ commit/rollback
resume Tx1
```

Used for:

- Logging  
- Audits  
- Sending emails in failure cases  

---

## 3.3 NESTED

Creates **savepoint** within existing transaction.

Requires a JDBC driver that supports savepoints.

```
Tx1: outer()
    ├── Savepoint A
    └── inner() rollback to A only
```

---

## 3.4 MANDATORY

Requires existing transaction or throws exception.

---

## 3.5 NOT_SUPPORTED

Suspend current transaction, run without Tx.

---

## 3.6 NEVER

Throw exception if transaction exists.

---

## 3.7 SUPPORTS

Run within transaction if exists, otherwise non-transactional.

---

# 4. ISOLATION LEVELS (DATABASE-HEAVY)

```
READ_UNCOMMITTED  → dirty reads allowed
READ_COMMITTED    → no dirty reads (Postgres default)
REPEATABLE_READ   → no non-repeatable reads (Postgres default)
SERIALIZABLE      → full isolation, highest cost
```

| Issue | RC | RR | S |
|-------|----|----|---|
| Dirty read | ❌ | ✔ | ✔ |
| Non-repeatable read | ❌ | ✔ | ✔ |
| Phantom read | ❌ | ❌ | ✔ |

---

# 5. HIBERNATE / JPA PERSISTENCE CONTEXT (VERY DEEP)

### Persistence Context = **1st level cache**

Stores:

- Managed entities  
- Dirty tracking snapshots  
- Proxy references for lazy fields  

---

## 5.1 Entity Lifecycle Diagram

```
         ┌──────────-┐
         │ transient │
         └─────┬────-┘
               │ persist()
               ▼
         ┌──────────-┐
         │ managed   │ ← persistence context
         └─────┬────-┘
               │ remove()
               ▼
         ┌──────────-┐
         │ removed   │
         └─────┬───-─┘
               │ detach()
               ▼
         ┌──────────-┐
         │ detached  │
         └─────────-─┘
```

---

# 5.2 Dirty Checking Internals

Hibernate tracks:

```
initial snapshot (before)
compare snapshot (after)
```

On transaction commit:

- Identify modified fields  
- Generate SQL `UPDATE`  
- Only dirty fields updated (if dynamic-update enabled)

---

# 5.3 Lazy Loading Internals (Proxy)

Hibernate creates proxies:

```
User user = session.load(User.class, id)
```

Object is:

```
User$HibernateProxy
```

Accessing field triggers:

```
LazyInitializer.initialize()
```

### LAZY Pitfalls:

- LazyInitializationException if outside transaction  
- N+1 SELECT issue

---

# 6. N+1 QUERY PROBLEM (HOW IT HAPPENS)

```
SELECT * FROM orders;   → N rows
for each row:
    SELECT * FROM user WHERE id=?
```

Why?

Because `@ManyToOne(fetch = LAZY)` triggers separate query per row.

Fixes:

- join fetch  
- batch-size  
- entity graph  
- DTO projections  

---

# 7. HIBERNATE FLUSH MODES

| Mode | Behavior |
|------|----------|
| AUTO | flush before query & commit |
| COMMIT | flush only on commit |
| MANUAL | never flush unless triggered |

---

# 8. SPRING MVC INTERNAL PIPELINE (IN DEPTH)

Complete flow:

```
FilterChain
    ↓
Spring Security Filter Chain
    ↓
DispatcherServlet
    ↓
HandlerMapping
    ↓
HandlerAdapter
    ↓
ArgumentResolvers
    ↓
Controller Method
    ↓
Return Value
    ↓
HandlerMethodReturnValueHandler
    ↓
HttpMessageConverter
```

---

# 9. INTERCEPTORS VS FILTERS VS AOP (DEEP DIFFERENCES)

| Feature | Filter | Interceptor | AOP |
|---------|--------|-------------|-----|
| Layer | Servlet | Spring MVC | Spring Core |
| Scope | request | handler | bean method |
| Access request body? | ❌ | ❌ | ❌ |
| Use cases | logging, CORS | auth, pre/post | cross-cutting concerns |

---

# 10. ARGUMENT RESOLVER INTERNALS

Spring resolves method parameters using:

```
HandlerMethodArgumentResolverComposite
```

Examples:

| Annotation | Resolver |
|------------|-----------|
| @RequestBody | RequestResponseBodyMethodProcessor |
| @PathVariable | PathVariableMethodArgumentResolver |
| @RequestParam | RequestParamMethodArgumentResolver |

---

# 11. EXCEPTION HANDLING (DEEP INTERNALS)

Flow:

```
Controller throws exception
    ↓
ExceptionHandlerExceptionResolver
    ↓
@ExceptionHandler or @ControllerAdvice
    ↓
Return ResponseEntity
```

---

# 12. @ASYNC INTERNALS

`@Async` uses:

- AOP proxies  
- Executor from `@EnableAsync`

Flow:

```
Proxy intercepts @Async
   ↓
Wraps in Runnable/Callable
   ↓
Submit to ThreadPoolTaskExecutor
```

⚠ **Self-invocation trap applies to @Async too**

---

# 13. CACHING INTERNALS

`@Cacheable` uses:

- CacheAspectSupport  
- KeyGenerator  
- CacheResolver  
- Underlying cache (Caffeine, Redis, Ehcache)

Flow:

```
CacheAspect intercepts
   ↓ check cache
   ↓ call method if miss
   ↓ store result
```

---

# 14. SPRING'S THREADING MODEL

### MVC Model (Blocking)

Each request:

```
Tomcat thread → Controller → Service → Repo
```

### WebFlux Model (Non-blocking)

```
EventLoop → Mono/Flux → Reactor chain → thread hops
```

---

# 15. TOP PITFALLS & BEST PRACTICES (ADVANCED)

## ❌ Common Mistakes:

- @Transactional on **private methods**
- @Transactional not working due to **self-invocation**
- Lazy loading outside transaction
- Using `FetchType.EAGER` everywhere
- Letting Hibernate issue 200 queries per request
- Using Entity as API response (leaks internals)
- Catching exceptions but not marking rollback

---

## ✔ Best Practices:

- DTOs for API responses  
- Use Transactional on **service layer only**  
- Prefer **constructor injection**  
- Keep repositories small  
- Use Page and Slice for pagination  
- Enable batch inserts & batch updates  
- Always log SQL + parameters in dev  
- Profile Hibernate queries  
- Use entity graphs for complex loads  

---

# END OF CHAPTER C
