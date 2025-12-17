## How would you design a high-availability, scalable .NET Core Web API for millions of users?

### Keep in mind below points while designing an application:
* Scalable → handles millions of users
* Highly available → no single point of failure
* Secure → multiple security layers
* Performant → caching + CDN
* Maintainable → clear separation of concerns
* Designing a high-availability, scalable .NET Core Web API for millions of users requires thinking across architecture, infrastructure, data, performance, and operations.
* Below example is a cloud-native microservices architecture using API Gateway, independently deployable ASP.NET Core services, database-per-service pattern, Redis caching, and event-driven communication via message broker.

```csharp
Clients (Web/Mobile)
        |
   CDN (Cloudflare)
        |
API Gateway (YARP / Ocelot)
        |
Load Balancer (L7)
        |
Multiple ASP.NET Core API instances
        |
-----------------------------------------
|           Backend Services             |
| DB | Cache | Message Broker | Search   |
-----------------------------------------
```

 ```csharp
 Clients (Web / Mobile)
        |
   CDN (Cloudflare)
        |
   API Gateway (YARP / Ocelot)
        |
   Load Balancer (L7)
        |
-------------------------------------------------
|           Microservices Layer                 |
|                                               |
|  Auth Service        User Service              |
|  Order Service       Payment Service           |
|  Product Service     Notification Service      |
|                                               |
| (Each service has its own ASP.NET Core API)   |
-------------------------------------------------
        |
-------------------------------------------------
|        Per-Service Infrastructure             |
|                                               |
| Auth DB        Order DB        Product DB      |
| Redis Cache   Redis Cache     Redis Cache     |
|                                               |
| Message Broker (RabbitMQ / Kafka)              |
|                                               |
| Search (Elasticsearch / OpenSearch)            |
-------------------------------------------------
```

### 1) Clients (Web / Mobile):

e.g. Web apps (React, Angular, Blazor), Mobile apps (Android / iOS), 3rd-party integrations

* These are the consumers of API.
* Send HTTP/HTTPS requests (REST / GraphQL)
* Authenticate using tokens (JWT / OAuth)
* Consume responses

### 2) CDN

e.g. Cloudflare, Akamai, Azure Front Door

* A Content Delivery Network sits between clients and your backend.
* Cache static content (images, JS, CSS)
* Reduce latency by serving content from nearest edge
* Protect against attacks

**Key features**

*DDoS protection: Rate limiting, TLS termination, Edge caching, WAF (Web Application Firewall)*

#### DDoS(Distributed Denial of Service attacks) protection:

* Protects your application from Distributed Denial of Service attacks
* Absorbs traffic at global edge locations, Blocks malicious IPs, bots, and traffic patterns before they reach our servers
* Prevents our API Gateway, Load Balancer, and APIs from being overwhelmed
* Ensures high availability


#### Rate limiting:

* ASP.NET Core provides built-in rate limiting using fixed, sliding, and token bucket algorithms starting from .NET 7.
* For distributed systems, rate limiting should be enforced at API Gateway or using Redis.
* 429 Too Many Requests is returned when the limit is exceeded.
* Restricts the number of requests per client/IP/token in a given time window
* It is applied at CDN / API Gateway level (first line of defense), Sometimes also inside ASP.NET Core (AddRateLimiter)
* Prevents abuse, brute-force attacks, and API overuse and Protects backend resources

```csharp
100 requests / minute / IP   // -> If exceeded → returns 429 Too Many Requests
```

#### TLS termination:

```csharp
Client  →  HTTPS  →  CDN (TLS ends here)
CDN     →  HTTP/HTTPS → Backend API
```

* HTTPS (TLS) encryption is ended at the CDN or Load Balancer
* Benefits : Offloads CPU-intensive encryption from ASP.NET Core servers, Centralized certificate management, Faster request handling
* Cloudflare handles HTTPS, Internal traffic may use mTLS(Mutual Transport Layer Security) or private network HTTP

```csharp
Clients (Browser/Mobile)
   |
   | HTTPS
   |
CDN (Cloudflare)
   |  ← TLS termination here
   |
API Gateway / Load Balancer
   |  ← mTLS starts here
   |
ASP.NET Core APIs
   |
Backend Services (DB, Cache, MQ)
```

#### Edge caching:

* Caches responses close to users at edge locations
* Static assets (JS, CSS, images), Public GET API responses, Frequently requested data
* Benefits: Lower latency, Reduced load on backend APIs, Faster response times globally

#### WAF (Web Application Firewall):

* Security layer that protects against application-level attacks
* Blocks: SQL Injection, XSS, CSRF, Path traversal, Malicious payloads
* Inspects HTTP requests and responses, Uses predefined and custom security rules

### 3) API Gateway (YARP / Ocelot)

* The single entry point to your backend APIs.

#### What it does:

* Routes requests to correct services
* Aggregates multiple APIs into one call
* Handles cross-cutting concerns

#### Common responsibilities:

* Authentication & Authorization
* Rate limiting
* Request/response transformation
* API versioning
* Logging & monitoring

Examples

* YARP (Microsoft, high performance)
* Ocelot (popular in microservices)

```csharp
/api/auth      → Auth Service
/api/orders    → Order Service
/api/products  → Product Service
```

### 4) Load Balancer (Layer 7)

* Distributes traffic across multiple API instances.

#### Why Layer 7 (Application layer)

* Understands HTTP/HTTPS
* Can route based on : URL path, Headers, Cookies
* Examples : NGINX, Azure Application Gateway, AWS ALB
* Benefits : High availability, Fault tolerance, Zero-downtime deployments
* Result : No single server becomes a bottleneck

### 5) Multiple ASP.NET Core API Instances

* Business logic layer, scaled horizontally.
* Characteristics : Stateless APIs, Can be scaled up/down automatically, Run in: Docker containers, Kubernetes pods, App Services
* Responsibilities : Handle requests, Apply business rules, Call backend services
* If one instance fails, others continue serving traffic

### Backend Services: 

Shared infrastructure components used by API instances.<br/>

| DB | Cache | Message Broker | Search |<br/>

#### Database per Service : 

* Each microservice owns its data.
* Stores persistent data 
* Examples: SQL Server, PostgreSQL, MongoDB

```csharp
Order Service → Order DB
User Service  → User DB
Product       → Product DB
```

#### Cache : 

* Fast in-memory storage, 
* Examples: Redis, Memcached, 
* Used for: Frequently accessed data, Session data, Reducing DB load

#### Message Broker : 

* Microservices do NOT call each other synchronously all the time. Publish events and Other services subscribe
* Enables asynchronous communication
* Tools: RabbitMQ, Kafka, Azure Service Bus
* Used for: Background processing, Event-driven architecture, Decoupling services, Improves scalability & resilience

```csharp
OrderPlaced Event
    ↓
Payment Service
Notification Service
Inventory Service
```

#### Search:

* Optimized for full-text search
* Examples: Elasticsearch, OpenSearch
* Used for: Product search, Log analytics, Complex filtering


```csharp
Client → CDN → API Gateway → Load Balancer → ASP.NET Core Instance → Backend Services
```

-------------------------------------------------------------

## Throttling vs Rate Limiting:

| Feature         | Throttling       | Rate Limiting    |
| --------------- | ---------------- | ---------------- |
| Behavior        | Slows requests   | Rejects requests |
| Excess requests | Queued / delayed | Blocked          |
| Response        | Delayed success  | 429 error        |
| User experience | Slower           | Hard failure     |
| Resource usage  | Higher (queues)  | Lower            |
| Best for        | Internal systems | Public APIs      |
| Controls        | Speed            | Count            |

-------------------------------------------------------------

## Throttling:

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddConcurrencyLimiter("concurrent", opt =>
    {
        opt.PermitLimit = 5;   // max concurrent requests
        opt.QueueLimit = 10;   // queued requests
    });
});
```

*API Gateway*

* Allows only 5 concurrent requests
* Others wait in queue

*E-commerce Checkout*

* Throttles payment requests during flash sales
* Prevents DB overload

*Background Workers*

* Processes jobs slowly to avoid DB saturation

#### Throttling at Infrastructure Level (Best)

* NGINX (limit_req, limit_conn)
* API Gateway (Azure APIM, AWS API Gateway)
* Cloudflare

-------------------------------------------------------------

### Use Throttling when:

* Internal APIs
* Background processing
* Controlled client systems
* You prefer slow responses over failures

### Use Rate Limiting when:

* Public APIs
* Security-sensitive endpoints
* Prevent abuse
* Cost control

-------------------------------------------------------------

## mTLS:

### TLS vs mTLS:

| Feature              | TLS (HTTPS)     | mTLS                                 |
| -------------------- | --------------- | ------------------------------------ |
| Client authenticated | ❌ No            | ✅ Yes                              |
| Server authenticated | ✅ Yes           | ✅ Yes                              |
| Security level       | High            | **Very High**                        |
| Typical use          | Public websites | **Service-to-service communication** |


### How mTLS Works:

* Client sends request + its certificate
* Server verifies client certificate against a trusted CA
* Server sends its certificate
* Client verifies server certificate
* Encrypted, trusted communication begins 🔒
* mTLS is NOT used for browsers
* mTLS is used internally between services

Both sides say:<br/>
“I trust you, and you trust me.”<br/>

### mTLS usages:

* Microservices communication
* API Gateway → Backend APIs
* Payment systems
* Banking / Healthcare systems
* Internal admin APIs


| Method   | Security      | Best for                    |
| -------- | ------------- | --------------------------- |
| API Key  | Low           | Internal quick setup        |
| OAuth2   | High          | User authentication         |
| **mTLS** | **Very High** | **Service-to-service auth** |


-------------------------------------------------------------

## How do you decide between monolith vs microservices for a new project ?

* To migrate from a monolith to microservices, I would start by modularizing the monolith, identify bounded contexts, and incrementally extract services using the Strangler Fig pattern.
* Each microservice would own its database, communicate via APIs or events, and be deployed independently behind an API Gateway.
* This minimizes risk while gradually achieving scalability and autonomy.

| Factor            | Monolith            | Microservices    |
| ----------------- | ------------------- | ---------------- |
| Team size         | Small               | Large            |
| Domain complexity | Low                 | High             |
| Deployment        | Simple              | Complex          |
| Scalability       | Vertical            | Horizontal       |
| Performance       | Faster (in-process) | Network overhead |
| Failure isolation | Poor                | Excellent        |
| DevOps maturity   | Low                 | High             |

-------------------------------------------------------------

## Explain how to migrate monolith → microservices?

Migrating from a monolith to microservices is a gradual, evolutionary process, not a rewrite.

### 1) Start With a Modular Monolith:

Each module should:<br/>

* Own its business logic
* Communicate via interfaces (not shared DB calls)

```csharp
/Orders
/Payments
/Users
/Inventory
```

### 2) Identify Bounded Contexts:

Use Domain-Driven Design: Identify

* Which parts change together?
* Which parts scale independently?
* Which teams own which domains?

```csharp
Authentication
Notifications
Reporting
Search
Payments
```

### 3) Apply the Strangler Fig Pattern:

* New features go to microservices
* Old features remain in monolith
* Gradually route traffic from monolith → services

```csharp
Client
   |
API Gateway
   |
Old Monolith  ←→  New Microservice
```

### 4) Extract One Microservice at a Time:

* Pick low-risk, high-value module
* Create a new service with: Own API, Own DB
* Redirect traffic via API Gateway
* Disable that functionality in monolith
* Never extract multiple services at once

### 5) Database Decomposition:

* Don’t share databases
* Each microservice must:Own its database, Have its own schema
* Data sync via: Events (RabbitMQ / Kafka), API calls (if required)

### 6) Use Event-Driven Communication:

Replace in-process calls with:<br/>

* Domain events
* Asynchronous messaging
* Benefits: Loose coupling, Resilience, Scalability

```csharp
OrderCreated → Event Bus → Inventory / Payment / Notification
```

### 7) Centralize Cross-Cutting Concerns:

Move these out of monolith:<br/>

* Authentication → Identity service
* Authorization → Policy service
* Logging → ELK
* Caching → Redis
* Configuration → Config service

### 8) Introduce API Gateway:

* Use: YARP, Ocelot, Azure API Management
* Gateway handles: Routing, Rate limiting, Auth, Versioning

### 9) Containerization & Orchestration:

* Dockerize services
* Deploy via Kubernetes
* Enable : Auto-scaling, Health checks, Rolling updates

### 10) Observability & Resilience:

* Must-have before scaling: Distributed tracing (OpenTelemetry), Centralized logging, Metrics (Prometheus)
* Add resilience: Retries, Circuit breakers (Polly), Timeouts, Fallbacks

-------------------------------------------------------------

## Explain data consistency patterns (Saga, Outbox)?

Saga manages consistency across multiple microservices by executing local transactions and compensating on failure, avoiding distributed transactions.
Outbox ensures reliable event publishing by storing events in the same database transaction as business data.
Together, they provide eventual consistency in microservice architectures.

### A Saga is a sequence of local transactions across multiple services, where:
* Each service performs its own transaction
* If something fails, compensating actions are executed

### The Outbox Pattern ensures reliable event publishing when a service updates its database.

* Service updates its DB
* Service writes an event record to an Outbox table in the same transaction
* Background worker reads Outbox table
* Publishes events to message broker
* Marks Outbox record as processed


| Feature           | Saga                           | Outbox                          |
| ----------------- | ------------------------------ | ------------------------------- |
| Purpose           | Manage multi-service workflows | Reliable event publishing       |
| Solves            | Cross-service consistency      | DB ↔ Message broker consistency |
| Scope             | Business process               | Infrastructure reliability      |
| Uses compensation | ✔ Yes                          | ❌ No                          |
| Works with events | ✔ Yes                          | ✔ Yes                          |


-------------------------------------------------------------

## How do you design an application to handle peak traffic (10x sudden load)?

To handle sudden 10× traffic, I design the system with CDN and rate limiting at the edge, stateless horizontally scalable services, multi-level caching, 
asynchronous processing via queues, database protection with read replicas and circuit breakers, and graceful degradation. 
Auto-scaling and observability ensure the system absorbs spikes without crashing.

-------------------------------------------------------------

## Explain how you would implement caching at different layers (DB, API, Client).

Caching should be layered, not single-point. Each layer reduces load on the next one down. Cache as close to the user as possible and invalidate as close to the data as necessary.

### (A) Database-Level Caching (Closest to Data):

* Reduce DB CPU, I/O, and query execution time.

#### 1) Query Result Cache (DB Engine)

*SQL Server buffer pool caches:*

* Data pages
* Execution plans

*Automatic, but:*

* Cleared under memory pressure
* Not enough for high traffic


#### 2) Read Replicas

* Offload reads from primary DB
* Used with heavy read workloads

When DB Cache is Useful<br/>

* Complex queries
* Reporting
* Aggregations

### (B) API-Level Caching (Most Important Layer):

Avoid hitting DB entirely for repeated requests.

#### 1) In-Memory Cache:

* IMemoryCache (Single Instance)
* Limitations: Not shared across instances, Lost on restart

```csharp
services.AddMemoryCache();

var data = await _cache.GetOrCreateAsync("products", async entry =>
{
    entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);
    return await _repo.GetProductsAsync();
});
```

#### 2) Distributed Cache (Recommended):

* Example : Redis
* Benefits : Shared across instances, Survives restarts, Scales horizontally

```csharp
services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379";
});
```

```csharp
var cached = await _cache.GetStringAsync(key);
if (cached == null)
{
    var data = await GetFromDb();
    await _cache.SetStringAsync(key, Json, new DistributedCacheEntryOptions
    {
        AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
    });
}
```

#### 3) Output Caching (ASP.NET Core 7+)

* Caches full HTTP response 
* Very effective for GET APIs

```csharp
services.AddOutputCache();

app.UseOutputCache();

[OutputCache(Duration = 60)]
public IActionResult GetProducts() => Ok(data);
```

| Strategy      | Use Case           |
| ------------- | ------------------ |
| TTL           | Simple data        |
| Key eviction  | Known changes      |
| Event-based   | Microservices      |
| Write-through | Strong consistency |


### (C) Client-Side Caching (Closest to User):

* Eliminate network calls completely.

#### 1) Browser Cache (HTTP Headers)

```http
Cache-Control: public, max-age=3600
ETag: "v1"
```

```csharp
[ResponseCache(Duration = 60, Location = ResponseCacheLocation.Client)]
```

#### 2) CDN / Edge Cache (Cloudflare):

* Cache static files
* Cache public GET APIs

Use cache keys:
* URL
* Headers

Query params:
* Often reduces 70–80% traffic.


#### Cache Invalidation Across Layers:

* If data changes, invalidate from the top layer down

```csharp
DB Update
  ↓
Invalidate Redis
  ↓
Invalidate Output Cache
  ↓
Purge CDN
```

-------------------------------------------------------------

## How do you ensure fault tolerance and resiliency in distributed systems?

I ensure fault tolerance by using redundancy, stateless services, load balancing, retries with backoff, circuit breakers, asynchronous messaging, and graceful degradation. Observability, idempotency, and chaos testing ensure the system recovers quickly without cascading failures.

* Ensuring fault tolerance and resiliency in distributed systems means designing the system to expect failures, contain them, and recover automatically without cascading outages.
* Failures are normal. Design for failure, not for success.

### 1) Redundancy & High Availability:

* Multiple Instances
* Multi-AZ / Multi-Region

### 2) Load Balancing:

Load Balancers - Route only to healthy instances, Automatic failover

### 3) Health Checks:

* A health check endpoint reports the status of - The API process, Its dependencies (DB, Redis, RabbitMQ, etc.)

#### Used by:

* Load Balancers (L7)
* Kubernetes (liveness / readiness)
* Monitoring tools (Prometheus, App Insights)

### 4) Stateless Services:

* Stateless services scale and recover easily.
* JWT
* Distributed cache (Redis)


### 5) Asynchronous & Event-Driven Design:

* Message Queues : RabbitMQ / Kafka / Service Bus
* Absorbs spikes
* Prevents cascading failures

### 6) Data Resilience

* Database Replication: Primary + read replicas, Automatic failover
* Eventual Consistency : Saga pattern, Compensating transactions
* Outbox Pattern : Prevent lost messages

### 7) Graceful Degradation

When parts fail:

* Serve cached or stale data
* Disable non-critical features
* Return partial responses
* Partial availability is better than downtime.

### Observability & Fast Recovery

#### Monitoring

* Latency (p95/p99)
* Error rate
* Saturation

#### Logging & Tracing

* Centralized logs (ELK)
* Distributed tracing (OpenTelemetry)

#### Alerting

* Actionable alerts only

### 7) Chaos & Failure Testing

#### Test:

* Instance crashes
* Network latency
* DB slowdowns
* Message loss
* Tools: Chaos Monkey, k6, Fault injection

| Failure              | Strategy             |
| -------------------- | -------------------- |
| Payment API slow     | Circuit breaker      |
| Notification failure | Queue + retry        |
| DB overload          | Cache + read replica |
| Service crash        | Auto-restart         |
| Region down          | Multi-AZ failover    |


-------------------------------------------------------------

## Healthcheck Types:

**1) Liveness Probe**

```csharp
GET /health/live
```

* Check if the app running?
* Detects deadlocks, crashes
* If fails → container restarted

**2) Readiness Probe**

```csharp
GET /health/ready
```

* Is the app ready to receive traffic?
* Checks DB, cache, external services
* If fails → traffic is stopped (but app not restarted)

**3) Startup Probe (K8s)**

* Has the app fully started?
* Used when startup is slow (migrations, warmup)


-------------------------------------------------------------

## Implementing Healthchecks in 

### 1) ASP.Net core:

Health checks are lightweight API endpoints that report application and dependency health. Load balancers and Kubernetes use them to route traffic, restart unhealthy instances, and maintain high availability.

#### Step 1: Add Health Checks

```csharp
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy())
    .AddSqlServer(
        connectionString: builder.Configuration.GetConnectionString("Default"))
    .AddRedis("localhost:6379");
```


#### Step 2: Map Endpoints

```csharp
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => check.Name == "self"
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = _ => true
});
```

#### Sample Health Check Response

```json
{
  "status": "Healthy",
  "checks": [
    {
      "name": "sqlserver",
      "status": "Healthy",
      "duration": "00:00:00.021"
    }
  ]
}
```

### 2) YARP:

* Active Health Checks (Polling) : YARP periodically calls backend /health/ready.
* Passive Health Checks (Failure-based) : YARP marks a service unhealthy when requests fail (5xx, timeout)

### 3) Ocelot Health Checks:

Ocelot relies more on downstream health endpoints and load balancer awareness.

### 4) Gateway + Kubernetes 

```json
# Gateway
livenessProbe:
  httpGet:
    path: /health
    port: 80

# Backend services
readinessProbe:
  httpGet:
    path: /health/ready
    port: 80
```
-------------------------------------------------------------

## Delivery Semantics

* Message delivery semantics define how many times a message is delivered to a consumer.
* At-least-once delivery guarantees no message loss but allows duplicates, so consumers must be idempotent. Exactly-once delivery ensures messages are processed only once but is complex and expensive to achieve in distributed systems. In practice, most systems prefer at-least-once delivery combined with idempotent processing for scalability and reliability.


| Feature        | At-Least-Once       | Exactly-Once |
| -------------- | ------------------- | ------------ |
| Message loss   | ❌ No                | ❌ No         |
| Duplicates     | ✔ Possible          | ❌ No         |
| Complexity     | Low                 | Very high    |
| Performance    | High                | Lower        |
| Scalability    | Excellent           | Limited      |
| Common usage   | Very common         | Rare         |
| Consumer logic | Idempotent required | Simpler      |
| Cloud friendly | ✔ Yes               | ⚠️ Limited   |


* At-Least-Once : RabbitMQ, Azure Service Bus, AWS SQS, Kafka (default behavior)
* Exactly-Once : Kafka : Idempotent producers


-------------------------------------------------------------

## How would you design a secure authentication system using JWT/OAuth ?

JWT is not authentication — OAuth/OIDC is. JWT is just the token format.

I design authentication using OAuth 2.0 with OpenID Connect, issuing short-lived JWT access tokens and securely stored refresh tokens. APIs validate JWTs using asymmetric keys, while authorization is enforced via scopes and policies. Token rotation, TLS, rate limiting, and secure storage ensure the system is resilient against common attacks.

* Designing a secure authentication system using JWT and OAuth 2.0 / OpenID Connect (OIDC) requires clear separation of authentication, authorization, and token handling.
* Below is a production-grade design, aligned with how real systems (Auth0, Azure AD, Keycloak) work.


| Term           | Meaning                       |
| -------------- | ----------------------------- |
| Authentication | Who you are                   |
| Authorization  | What you can access           |
| OAuth 2.0      | Authorization framework       |
| OpenID Connect | Authentication layer on OAuth |
| JWT            | Token format                  |


### High-Level Architecture:

```json
Client (Web / Mobile)
   |
Authorization Server (OAuth / OIDC)
   |
Resource Server (API)
```

* Authorization Server → Issues tokens
* Resource Server (API) → Validates tokens
* Client → Uses tokens

### JWT Design (Security Best Practices)

#### 1) Claims:

```json
{
  "sub": "user-id",
  "iss": "auth-server",
  "aud": "api",
  "exp": 1710000000,
  "scope": "orders.read"
}
```

#### 2) Signing:

* Use RS256 (asymmetric)
* Private key → Auth server
* Public key → APIs
* Enables key rotation
* Avoid : Long-lived JWTs, Storing JWT in localStorage


#### 3) Token Validation in API (ASP.NET Core)

```csharp
builder.Services.AddAuthentication("Bearer")
.AddJwtBearer("Bearer", options =>
{
    options.Authority = "https://auth.example.com";
    options.Audience = "api";
});

```

* Signature validation
* Issuer & audience check
* Expiry enforcement

#### 4) Authorization

*Scope-Based:*

```csharp
scope: orders.read orders.write

[Authorize("orders.read")]
```

*Role-Based:*

```csharp
[Authorize(Roles = "Admin")]
```

*Policy-Based:*

```csharp
services.AddAuthorization(options =>
{
    options.AddPolicy("CanEditOrder",
        policy => policy.RequireClaim("scope", "orders.write"));
});
```

#### 5) Token Refresh & Revocation:

*Refresh Token Rotation:*

* New refresh token issued every time
* Old token invalidated

*Revocation Strategy:*

* Short access-token lifetime
* Maintain refresh-token store
* Blacklist for critical revocations

#### 6) Protect Against Common Attacks

| Threat            | Protection                      |
| ----------------- | ------------------------------- |
| Token theft       | Short TTL + HTTPS               |
| Replay            | jti + nonce                     |
| XSS               | HttpOnly cookies                |
| CSRF              | SameSite cookies + anti-forgery |
| Man-in-the-middle | TLS + HSTS                      |
| Brute force       | Rate limiting                   |


#### 7) Microservices Integration:

* Central Auth Server : Keycloak / Auth0 / Azure AD
* API Gateway : Token validation, Rate limiting, Header propagation



-------------------------------------------------------------

## What trade-offs exist between event-driven architecture vs request-response?

### High-Level Difference

| Aspect        | Request–Response | Event-Driven                       |
| ------------- | ---------------- | ---------------------------------- |
| Communication | Synchronous      | Asynchronous                       |
| Coupling      | Tight            | Loose                              |
| Flow          | Caller waits     | Fire-and-forget                    |
| Typical tech  | HTTP, gRPC       | Kafka, RabbitMQ, Azure Service Bus |

### Consistency

| Model            | Consistency |
| ---------------- | ----------- |
| Request-Response | Strong      |
| Event-Driven     | Eventual    |

### Performance & Latency Trade-off

| Concern      | Request-Response | Event-Driven              |
| ------------ | ---------------- | ------------------------- |
| Latency      | Low (single hop) | Higher (async processing) |
| Throughput   | Limited          | High                      |
| Peak traffic | Hard             | Easy                      |



### Request-Response (Sync) :

#### Advantages:

* Simple to understand & debug
* Client knows success/failure instantly
* Good for: Login, Checkout validation, Data fetch APIs
* Strong consistency : Transaction completes before response

#### Disadvantages:

* Tight coupling
* Scalability bottleneck
* Poor resilience

### Event-Driven Architecture (Async)

#### Advantages

* Loose coupling: Producer doesn’t know consumers, Services evolve independently
* High scalability: Services process events at their own pace, Easy to scale consumers horizontally
* Better resilience

#### Disadvantages:

* Eventual consistency
* Increased complexity : Message brokers, Idempotency, Schema versioning, Distributed tracing, Debugging is harder


#### Most production systems use both. Hybrid approach. Example: E-commerce

```csharp
Client → Order API (Request-Response)
              |
              └─ OrderCreated Event
                   ├─ Email Service
                   ├─ Inventory Service
                   └─ Analytics Service

```

Choose Request-Response when:

✅ Immediate response needed
✅ Strong consistency required
✅ Simple workflows

Choose Event-Driven when:

✅ High scalability needed
✅ Loose coupling required
✅ Many downstream consumers
-------------------------------------------------------------