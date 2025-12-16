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