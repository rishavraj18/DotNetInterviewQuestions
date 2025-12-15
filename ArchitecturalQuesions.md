## How would you design a high-availability, scalable .NET Core Web API for millions of users?

Designing a high-availability, scalable .NET Core Web API for millions of users requires thinking across architecture, infrastructure, data, performance, and operations

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

### Clients (Web / Mobile):

e.g. Web apps (React, Angular, Blazor), Mobile apps (Android / iOS), 3rd-party integrations

* These are the consumers of API.
* Send HTTP/HTTPS requests (REST / GraphQL)
* Authenticate using tokens (JWT / OAuth)
* Consume responses

### CDN – Cloudflare

e.g. Cloudflare, Akamai, Azure Front Door

* A Content Delivery Network sits between clients and your backend.
* Cache static content (images, JS, CSS)
* Reduce latency by serving content from nearest edge
* Protect against attacks

**Key features**

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
| controls        | Speed            | Count            |

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