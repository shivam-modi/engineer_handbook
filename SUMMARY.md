# 📘 Principal Engineer Backend Handbook  
### *Complete Summary + All Mermaid Diagrams (A → K)*  
This `SUMMARY.md` is the **top-level table of contents** + **quick diagram reference** for all chapters.

---

# 🧭 Table of Contents

## **A — Java, Kotlin, JVM Internals**
- JVM structure  
- JIT, GC, Memory model  
- Kotlin coroutines vs threads  

## **B — Spring Boot Core Internals**
- Application startup  
- DispatcherServlet lifecycle  
- Auto-configuration  

## **C — Advanced Spring Boot Deep Dive**
- @Transactional internals  
- AOP proxying  
- JPA entity manager  
- Self-invocation issue  

## **D — Distributed Systems & Microservices**
- Sync/Async boundaries  
- Sagas & Orchestration  
- Event-driven communication  

## **E — Databases & Query Optimization**
- PostgreSQL MVCC  
- Query planner internals  
- Indexing strategies  
- Redis + LSM tree DBs  

## **F — DevOps, Cloud, Networking, K8s**
- Docker runtime stack  
- Kubernetes control plane  
- Networking fundamentals  

## **G — Observability & SRE**
- Logs, metrics, traces  
- SLOs & Error Budgets  

## **H — Security Engineering**
- OAuth2  
- JWT  
- Defense in Depth  

## **I — Performance Engineering**
- GC tuning  
- Backpressure  
- Threading vs async  
- Kernel + network tuning  

## **J — Architecture Design**
- Monolith vs microservices  
- CQRS + Event Sourcing  
- Data mesh  
- Scalability patterns  

## **K — Code Quality + Design Patterns**
- SOLID, DRY, YAGNI, KISS  
- DDD, Hexagonal, Clean Architecture  
- Anti-patterns + Refactoring  

---

# 🧩 Chapter-by-Chapter Mermaid Diagrams

---

# 🅰 Chapter A — JVM, Java, Kotlin

## JVM Architecture
```mermaid
flowchart TD
    A[Java/Kotlin Source] --> B[javac / kotlinc<br/>Compile to .class]
    B --> C[Class Loader Subsystem]
    C --> D[Method Area / Metaspace<br/>Class Metadata]
    C --> E[Heap<br/>Objects]
    C --> F[Stacks<br/>Frames, Locals]
    C --> G[PC Registers]
    C --> H[Native Method Area]

    D & E & F & G & H --> I[Execution Engine]
    I --> J[Interpreter]
    I --> K[JIT Compiler]
    I --> L[GC Subsystem]
```

## Threads vs Coroutines
```mermaid
flowchart LR
    subgraph Java Threads
      T1[Thread 1]:::java
      T2[Thread 2]:::java
      T3[Thread 3]:::java
    end

    subgraph Kotlin Coroutines
      PT[Dispatcher Pool]:::kotlin
      C1[Coroutine 1]:::cor
      C2[Coroutine 2]:::cor
      C3[Coroutine 3]:::cor
    end

    PT <-- schedules/resumes --> C1
    PT <-- schedules/resumes --> C2
    PT <-- schedules/resumes --> C3

    classDef java fill:#f6d5ff,stroke:#b56bdc;
    classDef kotlin fill:#d5f6ff,stroke:#4d9fdc;
    classDef cor fill:#ffe6cc,stroke:#e6a23c;
```

---

# 🅱 Chapter B — Spring Boot Core

## Spring Boot Startup
```mermaid
flowchart TD
    A[SpringApplication.run()] --> B[Prepare Environment]
    B --> C[Create ApplicationContext]
    C --> D[Load Bean Definitions]
    D --> E[Refresh Context]
    E --> F[Start Embedded Server]
```

## DispatcherServlet Lifecycle
```mermaid
sequenceDiagram
    participant Client
    participant DS as DispatcherServlet
    participant HM as HandlerMapping
    participant HA as HandlerAdapter
    participant Ctrl as Controller
    participant MSG as MessageConverter

    Client->>DS: HTTP Request
    DS->>HM: find handler
    HM-->>DS: HandlerMethod
    DS->>HA: invoke handler
    HA->>Ctrl: controller logic
    Ctrl-->>HA: Response DTO
    HA->>MSG: serialize
    MSG-->>Client: HTTP Response
```

---

# 🅲 Chapter C — Advanced Spring (TX, AOP)

## Transaction Proxy Flow
```mermaid
sequenceDiagram
    participant Client
    participant Proxy as @Transactional Proxy
    participant TM as TransactionManager
    participant Srv as Service
    participant DB as DB

    Client->>Proxy: placeOrder()
    Proxy->>TM: begin()
    TM-->>Proxy: Tx started
    Proxy->>Srv: placeOrder()
    Srv->>DB: SQL Ops
    DB-->>Srv: OK
    Proxy->>TM: commit()
    TM-->>Proxy: done
    Proxy-->>Client: result
```

## Self Invocation Issue
```mermaid
flowchart LR
    A[Client] --> B[Transactional Proxy]
    B --> C[outer()]
    C --> D[inner()]:::nopxy

    classDef nopxy fill:#ffe6e6,stroke:#ff4d4f;
```

---

# 🅳 Chapter D — Distributed Systems

## Microservice Topology
```mermaid
flowchart LR
    Client --> APIGW[API Gateway]
    APIGW --> S1[Orders Service]
    S1 -->|HTTP sync| S2[Payments]
    S1 -->|Publish Event| K[Kafka]
    K --> S3[Billing]
    K --> S4[Notifications]
```

## Saga Orchestration
```mermaid
flowchart TD
    O[Saga Orchestrator] --> A[Reserve Stock]
    A --> B[Capture Payment]
    B --> C[Create Shipment]
    C --> D[Complete Saga]

    A -->|Fail| RA[Compensate: Release Stock]
    B -->|Fail| RB[Refund Payment]
    C -->|Fail| RC[Cancel Shipment]
```

---

# 🅴 Chapter E — Databases

## PostgreSQL MVCC
```mermaid
flowchart LR
    A[INSERT xmin=T1] --> B[UPDATE<br/>old xmax=T2<br/>new xmin=T2]
    B --> C[DELETE xmax=T3]
    C --> D[VACUUM removes dead tuples]
```

## Query Lifecycle
```mermaid
flowchart TD
    Q[SQL Query] --> P[Parse]
    P --> R[Rewrite]
    R --> O[Planner]
    O --> E[Executor]
    E --> RES[Result]
```

---

# 🅵 Chapter F — DevOps & K8s

## K8s Control Plane
```mermaid
flowchart TD
    subgraph Control Plane
        API[API Server]
        SCH[Scheduler]
        CM[Controller Manager]
        ETCD[etcd]
    end

    subgraph Node
        KL[Kubelet]
        KP[Kube-Proxy]
        CR[containerd]
        PODS[Pods]
    end

    API <--> KL
    KL --> CR
    KP --> PODS
```

## Docker Stack
```mermaid
flowchart TD
    CLI[Docker CLI] --> DE[Docker Engine]
    DE --> CTRD[containerd]
    CTRD --> RUNC[runc]
    RUNC --> KERNEL[Linux Kernel]
```

---

# 🅶 Chapter G — Observability & SRE

## Observability Pillars
```mermaid
flowchart TD
    S[Service] --> M[Metrics]
    S --> L[Logs]
    S --> T[Traces]

    M --> DASH[Dashboards]
    L --> ANA[Log Analysis]
    T --> TRC[Trace Explorer]
```

## Error Budget
```mermaid
flowchart LR
    SLO[SLO: 99.9%] --> EB[Error Budget 0.1%]
    EB --> POL[Release Policy]
```

---

# 🅷 Chapter H — Security

## Defense in Depth
```mermaid
flowchart TD
    NET[Network Security] --> APP[App/Auth]
    APP --> DATA[Data Security]
    DATA --> OPS[Ops/Audit/Secrets]
```

## OAuth2 / JWT
```mermaid
sequenceDiagram
    participant User
    participant Client
    participant Auth
    participant API

    User->>Client: Authenticate
    Client->>Auth: Auth Code Request
    Auth-->>Client: Code
    Client->>Auth: Exchange Code
    Auth-->>Client: Access Token + Refresh Token
    Client->>API: Bearer Token
    API-->>Client: Response
```

---

# 🅸 Chapter I — Performance

## Request Path
```mermaid
flowchart LR
    Client --> NW[Network Stack]
    NW --> LB[Load Balancer]
    LB --> APP[Application]
    APP --> DB[Database / Cache]
    DB --> APP --> Client
```

## Backpressure
```mermaid
flowchart TD
    P[Producers] --> Q[Queue]
    Q --> C[Consumers]
    Q -->|High| BP[Backpressure]
    BP --> P
```

---

# 🅹 Chapter J — Architecture

## Monolith → Modular → Microservices
```mermaid
flowchart LR
    M[Monolith] --> MM[Modular Monolith] --> MS[Microservices]
```

## CQRS Architecture
```mermaid
flowchart TD
    C[Commands] --> WM[Write Model]
    WM --> EVT[Events]
    EVT --> BUS[Kafka/Event Bus]
    BUS --> RM[Read Model Builders]
    RM --> Q[Query API]
```

---

# 🅺 Chapter K — Design Patterns & Principles

## Hexagonal Architecture
```mermaid
flowchart TD
    subgraph Core[Domain & Use Cases]
      UC[Use Cases]
      DM[Domain Model]
    end

    subgraph Inbound[Inbound Adapters]
      REST[REST Controller]
      GRPC[gRPC Handler]
      KIN[Kafka Consumer]
    end

    subgraph Outbound[Outbound Adapters]
      DB[DB Repo]
      KOUT[Kafka Producer]
      EXT[External API]
    end

    REST --> UC
    GRPC --> UC
    KIN --> UC

    UC --> DB
    UC --> KOUT
    UC --> EXT
```

## Principle Map
```mermaid
flowchart TD
    SRP[SRP] --> Cohesion[High Cohesion]
    DIP[DIP] --> LowCoup[Low Coupling]
    DRY[DRY] --> ReduceDup
    KISS[KISS] --> Simplicity
    YAGNI[YAGNI] --> AvoidOverEng

    Cohesion --> Maintainable
    LowCoup --> Maintainable
    Simplicity --> Maintainable
```

---

# 📘 End of SUMMARY.md  
This file acts as the **top-level book index** + **universal diagram reference**.

