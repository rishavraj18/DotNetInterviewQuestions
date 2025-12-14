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
* Prevents your API Gateway, Load Balancer, and APIs from being overwhelmed
* Ensures high availability


#### Rate limiting:

* Restricts the number of requests per client/IP/token in a given time window
* It is applied at CDN / API Gateway level (first line of defense), Sometimes also inside ASP.NET Core (AddRateLimiter)
* Prevents abuse, brute-force attacks, and API overuse and Protects backend resources

```csharp
100 requests / minute / IP   // -> If exceeded → returns 429 Too Many Requests
```




TLS termination


Edge caching


WAF (Web Application Firewall)

  -------------------------------------------------------------