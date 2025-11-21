# B. Spring Boot Core  
_Deep Internals of Dependency Injection, Bean Lifecycle, Auto-Configuration, DispatcherServlet, Embedded Containers, and Bootstrap Sequence_

---

# Table of Contents
1. Spring Boot Startup Flow (Deep)
2. Bean Lifecycle (with Diagram)
3. Dependency Injection (DI & IoC Internals)
4. Auto-Configuration & Conditional Config
5. Application Context Internals
6. ConfigurationProperties & External Config
7. Event System (Application Events)
8. Embedded Server (Tomcat/Jetty/Netty internals)
9. DispatcherServlet Request Lifecycle
10. HTTP Message Converters
11. Validation (JSR-303 internals)
12. Profiles & Environment
13. Common Pitfalls & Best Practices

---

# 1. SPRING BOOT STARTUP FLOW (DEEP)

Spring Boot startup is more complex than most engineers realize.

```
SpringApplication.run()
    ↓
Load Bootstrap Context
    ↓
Prepare Environment
    ↓
Process ApplicationContextInitializers
    ↓
Create ApplicationContext
    ↓
Refresh Context
    │
    ├─ Load Bean Definitions
    ├─ Create BeanFactory
    ├─ Instantiate Beans
    ├─ Apply Bean Post-Processors
    ├─ Init Embedded Web Server
    ├─ Register Web Endpoints
    └─ Publish ApplicationStartedEvent
```

---

# 2. BEAN LIFECYCLE (SUPER DEEP)

## Diagram: Bean Lifecycle

```
                Bean Definition
                       │
                       ▼
        ┌───────────────────────────────────┐
        │ Instantiation (Constructor)       │
        └───────────────────────────────────┘
                       │
                       ▼
        ┌───────────────────────────────────┐
        │ Dependency Injection (Setter/Field) │
        └───────────────────────────────────┘
                       │
                       ▼
        ┌───────────────────────────────────┐
        │ BeanPostProcessor (Before Init)   │
        └───────────────────────────────────┘
                       │
                       ▼
        ┌───────────────────────────────────┐
        │ @PostConstruct / init-method      │
        └───────────────────────────────────┘
                       │
                       ▼
        ┌───────────────────────────────────┐
        │ BeanPostProcessor (After Init)    │
        └───────────────────────────────────┘
                       │
                       ▼
                  Bean Ready
                       │
                       ▼
           On shutdown: @PreDestroy / destroy-method
```

---

## 2.1 BeanPostProcessor (Key Concept)

`BeanPostProcessor` allows deep customization:

- Wrap beans in proxies  
- Modify bean properties  
- Add AOP functionality (Spring AOP uses this)  
- Implement `@Transactional`  

Examples:
- `AnnotationAwareAspectJAutoProxyCreator`  
- `AutowiredAnnotationBeanPostProcessor`  
- `ConfigurationPropertiesBindingPostProcessor`  

---

# 3. DEPENDENCY INJECTION (DEEP)

## Spring supports:

- **Constructor injection** (best practice)
- Setter injection
- Field injection (worst practice)
- Lookup method injection
- JavaConfig injection

---

## 3.1 How DI Actually Works

Spring uses:

- `DefaultListableBeanFactory`
- DependencyDescriptor
- AutowireCandidateResolver

Algorithm:

1. Identify injection points  
2. Resolve bean names/types  
3. Handle qualifiers  
4. Detect circular dependencies  
5. Inject dependencies  

---

## 3.2 Circular Dependency Internals

Allowed only in **singleton scope**.

Mechanism:

```
BeanFactory creates Hollow Bean (Object)
      ↓
Adds to earlySingletonObjects
      ↓
Uses setter/field injection
      ↓
Completes full bean creation
```

If constructor injection is used:

❌ Circular dependencies cannot be resolved  
→ throws `BeanCurrentlyInCreationException`

---

# 4. AUTO-CONFIGURATION (The Most Important Feature)

Spring Boot enables functionality using:

```
spring.factories (Spring Boot 2)
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports (Boot 3)
```

---

## 4.1 Auto-configuration Internals

Classes annotated with:

```
@Configuration
@AutoConfiguration
@ConditionalOnClass
@ConditionalOnMissingBean
```

Logic:

1. Check if required class on classpath  
2. Check if user has defined override bean  
3. Register auto-config bean definitions  
4. Apply conditions in order  

---

## 4.2 Key Conditional Annotations

| Annotation | Purpose |
|-----------|----------|
| `@ConditionalOnClass` | Only load if class exists |
| `@ConditionalOnMissingBean` | Do not override user beans |
| `@ConditionalOnProperty` | Enable/disable features |
| `@ConditionalOnWebApplication` | Only load for web apps |

---

# 5. APPLICATION CONTEXT INTERNALS

### Types:

- `AnnotationConfigApplicationContext` (non-web)
- `ServletWebServerApplicationContext`
- `ReactiveWebServerApplicationContext`

Internals:

- BeanFactory  
- Environment  
- ResourceLoader  
- EventMulticaster  
- ConversionService  

---

# 6. CONFIGURATION PROPERTIES

## How Spring binds properties

```
application.properties / yml
    ↓
Environment abstraction
    ↓
Binder API
    ↓
@ConfigurationProperties class
```

Supports:

- nested properties  
- lists  
- maps  
- relaxed binding  

---

# 7. EVENT SYSTEM

### Event Flow:

```
ContextRefreshedEvent
ApplicationStartedEvent
ApplicationReadyEvent
ApplicationFailedEvent
```

Custom events:

```java
@Component
public class MyListener implements ApplicationListener<MyEvent> {}
```

Or:

```java
@EventListener
public void handle(MyEvent e) {}
```

---

# 8. EMBEDDED SERVERS

Spring Boot supports:

- Tomcat (default)
- Jetty
- Netty (for WebFlux)
- Undertow (optional)

---

## 8.1 Embedded Tomcat Architecture

```
┌─────────────────────┐
│      Tomcat         │
├─────────────────────┤
│ NIO Connector       │  ← handles IO, threads, polling
├─────────────────────┤
│ ProtocolHandler     │
├─────────────────────┤
│ Request Parsing     │
├─────────────────────┤
│ Application Filters │
├─────────────────────┤
│ DispatcherServlet   │
└─────────────────────┘
```

---

# 9. DISPATCHER SERVLET LIFECYCLE (CORE OF SPRING MVC)

### Request Flow Diagram

```
Client
  ↓
Servlet Container (Tomcat)
  ↓
DispatcherServlet
  ↓
HandlerMapping
  ↓
HandlerAdapter
  ↓
Controller
  ↓
Return value
  ↓
HttpMessageConverter
  ↓
Response
```

---

## 9.1 HandlerMapping

Determines controller method:

- `RequestMappingHandlerMapping`
- `SimpleUrlHandlerMapping`

---

## 9.2 HandlerAdapter

Adapts controller invocation:

- `RequestMappingHandlerAdapter`
- `HttpRequestHandlerAdapter`

---

## 9.3 ArgumentResolvers

Inject:

- @PathVariable  
- @RequestParam  
- @RequestBody  
- @Valid  
- @AuthenticationPrincipal  

---

# 10. HTTP MESSAGE CONVERTERS

Convert between:

```
JSON  ↔  Java Object
XML   ↔  Java Object
String ↔  Java Object
```

Default converters:

- Jackson  
- Gson  
- JAXB  
- Resource converters  

---

# 11. VALIDATION (JSR-303)

Uses:

- Hibernate Validator  
- Bean Validation API  

Examples:

```java
class User {
    @NotNull
    String name;

    @Email
    String email;
}
```

Validation triggered by:

- @Valid in controller  
- Method-level validation  

---

# 12. PROFILES & ENVIRONMENT

Profiles:

```
spring.profiles.active=dev
```

Conditional beans:

```
@Profile("dev")
```

Environment access:

```
@Value("${prop}")
Environment env
@ConfigurationProperties
```

---

# 13. COMMON PITFALLS & BEST PRACTICES

## ❌ BAD PRACTICES:
- Field injection  
- Allowing circular dependencies  
- Too many @Autowired  
- Having 3000+ line configuration classes  
- Using @Transactional on private methods  
- Using @Async on methods in the same class  

---

## ✔ GOOD PRACTICES:
- Prefer constructor injection  
- Use records (Java 17) or Kotlin data classes  
- Separate configuration modules  
- Use `@ConfigurationProperties`  
- Avoid logic in configuration classes  
- Keep controllers thin  
- Keep services cohesive  

---

# END OF CHAPTER B
