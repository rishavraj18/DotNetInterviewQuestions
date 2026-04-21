# Practical and Scenario based questions:

## 1) If we are not getting error logs for the issue in production, how to troubleshoot in dotnet core ?

If no error logs are available, I troubleshoot using other signals: verify process/container health, restart events, readiness probes, metrics, network connectivity, reverse proxy logs, and .NET diagnostics tools like dotnet-trace or dotnet-dump. Missing logs often indicate either a logging pipeline issue or failures occurring before the logger initializes, so I also validate startup configuration and reproduce the issue in a similar environment.

* Signals – metrics, health probes, status codes
* Environment – process/container/pod state
* Network – connectivity, DNS, ports
* Runtime dumps – traces, memory, thread state
* Reproduction – local/staging with same config

### 1) Verify the App is Actually Running: 

 #### Container running : 
  docker ps --filter "name=my_container_name" or docker ps -a . "Up" indicates it is running, while "Exited" means it has stopped.

 #### Restart count :  
 
 * In a .NET application refers to the total number of times the application’s worker process (e.g., w3wp.exe in IIS) or container (e.g., in Kubernetes/Docker) has stopped and restarted within a specific timeframe. High restart counts usually indicate instability, such as crashes, memory leaks leading to recycling, or configuration issues.
 * Kubernetes - Use kubectl get pods or kubectl describe pod <pod-name>
 * Azure Services -  Activity Log in the Azure portal for active restart actions,  Check the logs via "Diagnostic Tools" -> "Application Events". Application Insights

 #### Exit code:
 
 * An exit code (also known as an exit status or return code) is an integer number that a computer program sends back to the operating system or parent process upon completion. It serves as a standardized signal to indicate whether the program finished successfully or encountered an error
 * Run $LASTEXITCODE

 #### CPU/memory spikes:

 * On VM:
	* dotnet-dump collect -p 1234
 * inside the container
	* dotnet-dump collect -p 1 (first find process identifier PID, these IDs are isolated, allowing multiple containers to each have their own "PID 1" without conflicting with each other or the host system.)
  * Azure: Diagnose and Solve Problems > Availability and Performance >  CPU Usage or Memory Usage.
	
 * CPU high now → dotnet-counters, dotnet-trace
 * Memory leak / OutOfMemory → dotnet-dump
 * App frozen / deadlock → dotnet-dump
 * Local: VS Diagnostic tool - Snapshot and comapare the sizes from the list

### 2) Check Logging Pipeline Itself

 The app may log correctly, but logs are not collected.

 #### Common Issues:
 * Wrong log level (Warning hides Information)
 * Sink misconfigured (Seq, ELK, App Insights, Splunk)
 * File path permissions
 * Stdout not captured in container
 * Structured logger not initialized early enough

 #### Verify:
 * appsettings.json
 * Environment-specific config
 * Serilog/NLog startup config
 * Container stdout/stderr collector
	

### 3) Use Health Endpoints

 Even without logs, health endpoints reveal state. If readiness fails, app may be alive but blocked by dependencies.</br>

 Add / Check:
 * /health
 * /ready
 * /live

### 4) Use Metrics Instead of Logs:

  Check:
	* Request count
	* Error rate
	* Latency
	* CPU
	* Memory
	* GC pauses
	* Thread pool starvation

Tools:

  * Prometheus / Grafana
  * Azure Monitor
  * Application Insights
  * OpenTelemetry

* High latency + low CPU = blocking I/O
* High memory then restart = memory leak / OOM
* Requests drop to zero = ingress/routing issue

### 5) Capture Live Diagnostics:

Use built-in diagnostics tools in lower environment or local.

* dotnet-counters
* dotnet-trace
* dotnet-dump
* dotnet-monitor

### 6) Check Startup Failures Before Logger Initialization: 

Some failures happen before logging starts.Examples:

* Invalid configuration
* Missing secret
* Bad certificate
* Port already in use
* DI registration failure

### 7) Inspect Reverse Proxy / Infra Logs:

If app logs are empty, request may never reach app.

Check:

* NGINX
* IIS
* Azure App Gateway
* Load Balancer
* Ingress Controller
* WAF

These often show:

* 502
* 503
* Timeout
* TLS failure
* Upstream connection refused

### 8) Reproduce error with Same code in different environment and then verify with Same Configuration:

Add Fallback Global Error Handling:

* Global exception middleware
* Startup try/catch logging
* Request correlation IDs
* Health checks
* OpenTelemetry tracing
* Structured logs

#### 9) Common Real Causes of “No Logs”

| Symptom                  | Likely Cause                    |
| ------------------------ | ------------------------------- |
| App restarts instantly   | Crash / bad config              |
| Pod alive but no traffic | Readiness failed                |
| 502 from gateway         | Port mismatch / app unreachable |
| Slow requests no errors  | Deadlock / thread starvation    |
| Random restarts          | OOMKilled (Kubernetes)          |
| Only local logs work     | Sink/export issue               |


--------------------------------------------------------------------------------------------------------------------------

## 2) DotNet older version and latest version difference ?

* Latest version: .NET 10 (LTS) & C#14
* Latest .NET 10 and C# 14 focus on performance, cloud-native development, and reducing boilerplate.
* C# 14 introduces extension members, lambda improvements, and better type handling.
* .NET 10 improves runtime performance, adds AI capabilities, and strengthens cloud-native support.

### 1) .NET runtime

* The .NET 10 runtime introduces improvements in JIT inlining, method devirtualization, and stack allocations. 
* Improved code generation for struct arguments, and enhanced loop inversion for better optimization.

### 2) .NET libraries

* The .NET 10 libraries introduce new APIs in cryptography, globalization, numerics, serialization, collections, and diagnostics.
* New JSON serialization options include disallowing duplicate properties, strict serialization settings, and PipeReader support for improved efficiency. 

### 3) C#14

Null-conditional assignment

```csharp
if (customer is not null)
{
    customer.Order = GetCurrentOrder();
}

customer?.Order = GetCurrentOrder(); // Same as above code

```

| Version | Feature                    | Example              | Purpose                    |
| ------- | -------------------------- | -------------------- | -------------------------- |
| C# 14   | Null Conditional Write     | `obj?.Prop = value;` | Safe property assignment   |

### 4) Authentication and authorization metrics:

Metrics have been added for certain authentication and authorization events in ASP.NET Core. With this change, you can now obtain metrics for the following events:

Authentication:

* Authenticated request duration
* Challenge count
* Forbid count
* Sign in count
* Sign out count

Authorization:

* Count of requests requiring authorization

### 5) Middleware:

* A new configuration option has been added to the ASP.NET Core exception handler middleware to control diagnostic output: ExceptionHandlerOptions.SuppressDiagnosticsCallback. This callback is passed context about the request and exception, allowing you to add logic that determines whether the middleware should write exception logs and other telemetry.

* The middleware's default behavior has also changed: it no longer writes exception diagnostics for exceptions handled by IExceptionHandler. Based on user feedback, logging handled exceptions at the error level was often undesirable when IExceptionHandler.TryHandleAsync returned true.

We can revert default behaviour:

```csharp
app.UseExceptionHandler(new ExceptionHandlerOptions
{
    SuppressDiagnosticsCallback = context => false;
});
```

--------------------------------------------------------------------------------------------------------------------------

## 3) Core microservices in Home Loan App ?

| Microservice         | Responsibility                      |
| -------------------- | ----------------------------------- |
| Customer Service     | Customer profile, KYC, address      |
| Loan Service         | Loan application, status, sanction  |
| EMI Service          | EMI schedule, interest calculations |
| Document Service     | Upload documents, income details    |
| Underwriting Service | Eligibility / risk checks           |
| Payment Service      | Loan disbursement / repayment       |
| Notification Service | SMS / Email / alerts                |
| Audit Service        | Compliance logs                     |
| Reporting Service    | MIS / dashboards                    |



### MIS - Management Information System

MIS (Management Information System) reporting services and dashboards aggregate raw data from departments like finance, sales, and operations to provide real-time visibility into business performance. 

#### Key Components of MIS Dashboards:
* Data Aggregation: Consolidates information from ERP, CRM, and accounting systems.
* Visualization: Uses interactive graphs, charts, and tables for easy interpretation.
* Filtering: Allows drilling down into data by time, department, or project.
* KPI Tracking: Monitors sales, inventory, revenue, and employee performance against targets. 

#### Types of MIS Reports:
* Sales Reports: Tracks customer retention and revenue numbers.
* Financial Reports: Includes Profit & Loss, Balance Sheets, and cash-flow statements.
* Operational Reports: Focuses on day-to-day metrics, such as inventory details or project progress.
* Executive Dashboards: High-level summary for leadership, providing a bird's-eye view of operations.

--------------------------------------------------------------------------------------------------------------------------

## 3) Azure Services Used in microservices ?

| Need           | Azure Service                             |
| -------------- | ----------------------------------------- |
| APIs Hosting   | Azure App Service or Azure Container Apps |
| Database       | Azure SQL Database                        |
| File Storage   | Azure Blob Storage                        |
| Authentication | Microsoft Entra ID                        |
| Secrets        | Azure Key Vault                           |
| Messaging      | Azure Service Bus                         |
| Monitoring     | Application Insights                      |
| CI/CD          | Azure DevOps                              |

--------------------------------------------------------------------------------------------------------------------------

## 4) End-to-End Flow:

![HomeLoanArchitecture](/img/HomeLoanArchitecture.JPG "HomeLoanArchitecture")

### Step 1: Customer Login

```flow
User → Authentication(MFA) → JWT Token → Frontend → APIs (with token)
```

### Step 2: Apply for Home Loan

Customer fills:

* Income
* Property value
* Employment details
* Loan amount

Backend Flow:

* Loan Service validates input
* Stores application in SQL database
* Publishes event to queue

Azure Service Used:

* Azure SQL Database
* Azure Service Bus

### Step 3: Upload Documents

Customer uploads: Payslip, Bank statement, Address Proof

Flow:
* Document Service receives files
* Saves in Azure Blob Storage
* Metadata in SQL
* Sends DocumentsUploaded event

```flow
File → API → Blob Storage → Queue Event
```

### Step 4: Eligibility Check

Underwriting Service consumes queue messages.

Checks:
* Credit score
* Income ratio
* Existing liabilities
* Employment risk
* Documents Verified event

### Step 5: Loan Decision taken

If approved:

* Loan amount sanctioned
* Interest rate assigned
* Tenure decided
* Publishes LoanApproved

Queue message goes to:

* Notification Service
* EMI Service
* Audit Service

### Step 6: EMI Schedule Generation

EMI Service calculates monthly installments.
Stores repayment schedule in database.

### Step 7: Customer Notification

Notification Service listens to queue:

* Approved email
* SMS sanction letter
* Push notification
* Uses - Azure Service Bus

### Step 8: Loan Agreement PDF

* Document/Reporting service generates sanction PDF.
* Stores in: Azure Blob Storage
* Trigger email with Loan sanction details

### Step 9: Loan Disbursement

Payment Service triggers disbursement.

Flow:

```flow
Validate approval -> Transfer to seller/builder -> Update status -> Publish event
```

### Step 10: Report Generation

* Report generation and Audit quarterly, yearly.

### Imp Notes:

#### 1) Technical Flow Between Microservices

Synchronous Calls (Immediate Response):

* Frontend → Loan API
* Loan API → Customer API
* Use REST

Asynchronous Calls (Background):

```flow
LoanApproved → Service Bus
              ↓
 Notification / EMI / Audit
 ```

#### 2) Where Each Azure Service Fits

| Azure Service                            | Real Usage in Home Loan App         |
| ---------------------------------------- | ----------------------------------- |
| Azure App Service / Azure Container Apps | Host microservices                  |
| Azure SQL Database                       | Loan data, users, EMI tables        |
| Azure Blob Storage                       | Uploaded docs, PDFs                 |
| Microsoft Entra ID                       | Login, token auth                   |
| Azure Key Vault                          | DB passwords, API keys, certs       |
| Azure Service Bus                        | Events, async workflows             |
| Application Insights                     | Errors, logs, tracing               |
| Azure DevOps                             | Build, test, deploy                 |

#### 3) Security Flow

```flow
App Startup → Managed Identity → Key Vault → Read Secrets
```

#### 4) Monitoring Flow

Every request tracked in: Application Insights

```flow
User Apply Loan Request
   ↓
Loan Service
   ↓
SQL Call
   ↓
Service Bus Event
   ↓
Notification Sent
```

#### 5) CI/CD Flow with Azure DevOps

```flow
Code Commit
   ↓
Build Pipeline
   ↓
Run Unit Tests
   ↓
Security Scan
   ↓
Build Docker Image
   ↓
Deploy to Azure
   ↓
Smoke Test
```

#### 6) Why Microservices Here?

Because banking modules change independently:

* EMI logic changes
* Notification changes
* KYC changes
* Document service scales separately
* Each can deploy independently.

#### 7) Recommended .NET Tech Stack

* ASP.NET Core Web API
* EF Core
* MediatR / CQRS
* Polly
* Serilog
* OpenTelemetry
* BackgroundService workers
* xUnit / NUnit
* Azure

#### 8) Real Production Concerns:

* Idempotent payments
* Retry queue messages
* Audit trails
* RBAC roles
* Encryption at rest
* PII masking
* Disaster recovery
* Blue/Green deployment

--------------------------------------------------------------------------------------------------------------------------

## What is Dependency Injection in .NET Core? How does DI work internally?

ASP.NET Core has a built-in DI container with three lifetimes: Transient, Scoped, Singleton.

* Register services (what types exist and their lifetimes).
* Build a dependency graph (resolve constructor parameters).
* Create instances (construct objects in the correct order).

When Registering:
* Store metadata (type → implementation → lifetime)

When Resolving:
* Lookup descriptor
* Find constructor
* Resolve dependencies recursively
* Construct object (using cached delegates)
* Cache it if singleton/scoped
* Track if disposable

On Scope End:
* Dispose scoped and transient disposables


## Service lifetimes – Transient, Scoped, Singleton

### A) Transient
* Creates a new instance every time
* Light-weight, stateless services
* Helper classes, utilities, Formatter, validators

### B)Scoped
* one instance per HTTP request
* Same instance reused within the same request, New request = new instance
* DbContext, Unit of Work pattern, Business logic that must stay consistent during one request

#### Inside Request #1:
* Controller: instance A
* Service A: instance A
* Repository: instance A

#### Inside Request #2:
* New instance B

### C) Singleton
* Only one instance in the entire application lifetime
* Created once, reused forever
* Same instance for all requests & all users
* Cache services, Logging frameworks, Configuration providers

* Only singleton/transient services are injected into middleware.
* Scoped services are not allowed


| Lifetime      | Where Stored         | Created When                  | Disposed When                |
| ------------- | -------------------- | ----------------------------- | ---------------------------- |
| **Transient** | not stored           | every request for the service | end of scope (if disposable) |
| **Scoped**    | request scope cache  | first request within scope    | end of request               |
| **Singleton** | root container cache | once at app startup           | app shutdown                 |


✔ If it holds global shared state → Singleton

(e.g., memory cache, configuration, system-wide services)

✔ If it works on a specific request → Scoped

(e.g., DbContext, request trackers, user services)

✔ If it’s stateless and lightweight → Transient

(e.g., helpers, utilities, mappers)

## What are the key advantages of Dependency Injection (DI)

* Loose Coupling, modularity, testability, maintainability, and flexibility
* Easy Unit Testing (Mocking becomes simple)
* Clean Code
* Better Extensibility
* Centralized Dependency Management
* Promotes SOLID Principles


## Ways to Inject Dependencies:

### 1) Constructor Injection ✅ (Recommended)

```csharp
public class UserController
{
    private readonly IUserService _service;

    public UserController(IUserService service)
    {
        _service = service;
    }
}
```

#### Why best:
* Explicit dependencies
* Easy to test
* Immutable dependencies


### 2) Method Injection

Dependency passed into a method.

```csharp
public void Process(IEmailService emailService)
{
}
```

#### Use for:
* Rare/optional dependencies
* One-time operations

### 3) Property Injection

* Dependency set through property.

```csharp
public IEmailService EmailService { get; set; }
```

#### Use cautiously:

* Less preferred because dependency may be missing.

### 4) E.g.

```csharp
services.AddDbContext<AppDbContext>();
services.AddHttpClient<IMyApi, MyApi>();
services.AddSingleton<IMemoryCache, MemoryCache>();
```

✅ Do
* Prefer constructor injection
* Use scoped for request-based services
* Use singleton for shared stateless services
* Keep services focused

❌ Don’t
* Inject scoped service into singleton directly
* Put state in singleton unless thread-safe
* Overuse service locator pattern
* Use property injection as default


## .Net 8 onwards

* When multiple Implementations of Same Interface
* Earlier: Used factory pattern for multiple implementations.
* Now: Use Keyed Services directly.

```csharp
builder.Services.AddScoped<ILoanService, LoanService>();
builder.Services.AddScoped<IAccountService, AccountService>();

builder.Services.AddKeyedScoped<INotificationService, SmsNotification>("sms");
builder.Services.AddKeyedScoped<INotificationService, EmailNotification>("email");
```

```csharp
public class LoanController(
    ILoanService loanService,
    [FromKeyedServices("sms")] INotificationService notifier)
{
}
```

### Older dotnet version (Factory way):

* Manual logic
* Harder to test
* Violates Open/Closed Principle

```csharp
public IPaymentService Get(string type)
{
   return type switch
   {
      "upi" => new UpiPayment(),
      "card" => new CardPayment()
   };
}
```
--------------------------------------------------------------------------------------------------------------------------

## What are the design pattern automatically getting implemented by modern .net core package ?

| Feature           | Pattern                    |
| ----------------- | -------------------------- |
| DI Container      | Dependency Injection / IoC |
| Middleware        | Chain of Responsibility    |
| Logging           | Factory                    |
| HttpClientFactory | Factory + Policy           |
| EF Core           | Repository + Unit of Work  |
| Filters           | Decorator                  |
| Events            | Observer                   |
| Hosted Services   | Template Method            |
| Minimal APIs      | Builder                    |


### 1. Dependency Injection (DI) → Inversion of Control Pattern

<br/>Built into ASP.NET Core by default.

#### Where it happens:

* builder.Services.AddScoped<>();
* Constructor injection in controllers/services

#### Benefits:

* Loose coupling
* Easy unit testing

### 2. Middleware Pipeline → Chain of Responsibility

Request flows through multiple middleware components.

#### e.g. 

```csharp
app.UseAuthentication();
app.UseAuthorization();
app.UseExceptionHandler();
```

#### Benefits:

* Flexible request pipeline
* Plug-and-play behavior

### 3. Logging → Factory Pattern

The logging system creates logger instances dynamically. Internally uses ILoggerFactory.

```csharp
ILogger<MyService> logger;
```
#### Benefits:

* Abstracts logging providers (Serilog, NLog, etc.)

### 4. Configuration → Options Pattern

Strongly-typed configuration binding.

```csharp
services.Configure<MySettings>(config.GetSection("MySettings"));
```

#### Pattern:

* Options Pattern (specialized pattern)
* Uses Singleton + Factory

#### Benefits:
Centralized configuration management

### 5. HttpClient → Factory + Resilience (Policy Pattern)

Using IHttpClientFactory

```csharp
services.AddHttpClient();
```

#### Pattern:
* Factory Pattern
* Policy Pattern (via Polly)

#### Benefits:

Retry, circuit breaker, timeout handling

### 6. Entity Framework Core → Multiple Patterns

#### Patterns used:

* Repository Pattern
* Unit of Work
* Change Tracking (Observer-like)

```csharp
DbContext.SaveChanges();
```

#### Benefits:

* Manages transactions automatically
* Tracks entity changes

### 7.  Filters → Decorator Pattern

Filters wrap around request execution.

```csharp
[Authorize]
[ActionFilter]
```

#### Benefits:

Add behavior without modifying core logic

### 8.  Minimal APIs → Builder Pattern

Fluent configuration style

```csharp
var app = builder.Build();
```

--------------------------------------------------------------------------------------------------------------------------

## SOLID Principle

### 1) Single Responsibility Principle (SRP)

A class should have only one responsibility.

#### Violation Smell:

One class does business logic + DB + logging + email

#### Bad Eg.

```csharp
public class OrderService
{
    public void CreateOrder() { }
    public void SaveToDb() { }
    public void SendEmail() { }
}
```

#### Good Fix

```csharp
public class OrderService { }
public class OrderRepository { SaveToDb() }
public class EmailService { }
```

### 2) Open / Closed Principle (OCP)

A Class should be open for extension and closed for modification. Adding a new feature should not require touching tested code.

#### Violation Smell:

* if / else
* switch
* string-based type checks

#### Bad Eg.

```csharp
if(type == "VIP") { }
else if(type == "Regular") { }
```

#### Good Fix (Strategy Pattern)

```csharp
public interface IDiscountStrategy
{
    double Apply(double amount);
}

public class VipDiscount : IDiscountStrategy
{
    public double Apply(double amount) => amount * 0.8;
}
```

### 3) Liskov Substitution Principle (LSP)

* Subtypes must be replaceable for their base types without breaking behavior. 
* LSP is violated when a subclass removes or weakens behavior promised by the base class.
* In banking systems, this often happens when fixed deposits or savings accounts are forced into a common withdrawable hierarchy.

#### Violation Smell:

* Overriding behavior to throw exceptions
* Breaking assumptions of base class
* Base class contract is broken
 
#### Bad Eg.

```csharp
public class BankAccount
{
    public virtual void Withdraw(decimal amount)
    {
        Console.WriteLine($"Withdrawing {amount}");
    }
}

public class FixedDepositAccount : BankAccount
{
    public override void Withdraw(decimal amount)
    {
        throw new NotSupportedException(
            "Withdrawals not allowed before maturity");
    }
}
```

```csharp
public void ProcessWithdrawal(BankAccount account)
{
    account.Withdraw(1000); // Expectation: withdrawal always works
}

// Subclass changed behavior in a way client code didn’t expect

BankAccount account = new FixedDepositAccount();
ProcessWithdrawal(account); // Runtime exception
```

#### Correct Design (LSP-Compliant)

##### Approach : Split hierarchy by behavior

```csharp
public abstract class Account
{
    public decimal Balance { get; protected set; }
}

public interface IWithdrawable
{
    void Withdraw(decimal amount);
}

public class SavingsAccount : Account, IWithdrawable
{
    public void Withdraw(decimal amount)
    {
        Balance -= amount;
    }
}

public class FixedDepositAccount : Account
{
    // No withdrawal capability
}

public void ProcessWithdrawal(IWithdrawable account)
{
    account.Withdraw(1000);
}
```

### 4) Interface Segregation Principle (ISP)

Clients should not depend on methods they don’t use. Smaller, role-based interfaces improve flexibility and testability.

#### Violation Smell:

* Fat interfaces
* NotImplementedException

#### Bad Eg.

```csharp
public interface IBankAccount
{
    void Deposit(decimal amount);
    void Withdraw(decimal amount);
    void ApplyLoan(decimal amount);
    void IssueChequeBook();
    void EarnInterest();
}

public class FixedDepositAccount : IBankAccount
{
    public void Deposit(decimal amount)
    {
        Console.WriteLine("Deposited");
    }

    public void Withdraw(decimal amount)
    {
        throw new NotImplementedException();
    }

    public void EarnInterest()
    {
        Console.WriteLine("Interest added");
    }
}
```

#### Correct Design (ISP)

```csharp
public interface IDepositable
{
    void Deposit(decimal amount);
}

public interface IWithdrawable
{
    void Withdraw(decimal amount);
}

public interface IInterestBearing
{
    void EarnInterest();
}

public class FixedDepositAccount :
    IDepositable,
    IInterestBearing
{
    public void Deposit(decimal amount) { }
    public void EarnInterest() { }
}
```

### 5) Dependency Inversion Principle (DIP)

* High-level modules should depend on abstractions, not concrete implementations. 
* DIP enables loose coupling and makes unit testing possible.

#### Violation Smell:

* new keyword inside business logic
* Hard-coded dependencies

#### Bad Example:
private FileLogger logger = new FileLogger();

#### Correct Design (DIP)

```csharp
public interface ILogger
{
    void Log(string message);
}

public class UserService
{
    private readonly ILogger _logger;

    public UserService(ILogger logger)
    {
        _logger = logger;
    }
}
```


| Question                                               | Expected Answer  |
| ------------------------------------------------------ | ---------------- |
| Which SOLID principle is most violated in legacy code? | SRP & DIP        |
| Which principle reduces `if/else`?                     | OCP              |
| Which principle prevents `NotImplementedException`?    | ISP              |
| Which principle improves testability most?             | DIP              |
| Can SOLID increase complexity?                         | Yes, if overused |

--------------------------------------------------------------------------------------------------------------------------

## How to secure Asp.Net core application: 

### 1) Authentication:

* JWT Bearer Tokens
* OpenID Connect
* OAuth2
* Microsoft Entra ID

### 2) Authorization

* Prefer policy-based auth for enterprise apps e.g. [Authorize(Policy = "CanApproveLoan")]
* Principle of Least Privilege

### 3) Always Use HTTPS

Encrypt traffic in transit.

```csharp
app.UseHttpsRedirection();
```

### 4)  Man-in-the-middle attacks (MITM) and downgrade attacks -> UseHsts();

 Enable HTTP Strict Transport Security (HSTS).<br/>
 All future attempts to access HTTP are automatically converted to HTTPS inside the browser, without hitting your server.

 ### 5) Secure Secrets Management:

 Never store secrets in code or Git.

 #### Use:
 * Environment variables
 * Azure Key Vault
 
### 6) Cross-Site Request Forgery (CSRF) 

Tricking a logged-in user to perform actions unknowingly<br/>

#### When a form is rendered:

<br/>ASP.NET Core generates:

* A cookie token → stored in the browser
* A request token → stored in a hidden field inside the form
* Example : User logged into bank, Visits malicious site, Hidden form submits transfer request
* Impact : Unauthorized actions
* Prevention : Anti-forgery tokens, SameSite cookies, Authorization checks, APIs usually use JWT instead of cookies, so CSRF risk is lower

```csharp
<form method="post">
    @Html.AntiForgeryToken()
    <button type="submit">Submit</button>
</form>
```

```csharp
[ValidateAntiForgeryToken]
public IActionResult Submit(MyModel model)
{
    // Your logic
}
```

### 7) Input Validation:

* Data annotations
* FluentValidation

### 8) Prevent SQL Injection

* Attacker injects malicious SQL into input fields to manipulate the database e.g. ' OR 1=1 --
* Impact: Data leakage, Authentication bypass, Data deletion
* Prevention: Parameterized queries, ORM (EF Core), WAF SQL injection rules, Input validation, Never use string concatenation for SQL, Stored Procedure

```csharp
context.Users
    .Where(u => u.Email == email); // Safe (parameterized)
```

### 9)  XSS (Cross-Site Scripting)

* Injecting malicious JavaScript into web pages
* Types : Stored XSS, Reflected XSS, DOM-based XSS
* Impact : Session hijacking, Cookie theft, Account takeover
* Prevention : Output encoding (HTML encode output), Content Security Policy (CSP) headers, WAF XSS filters

e.g. Razor automatically HTML-encodes output

 ### 10) CORS

 ```csharp
 builder.Services.AddCors(options =>
 {
    options.AddPolicy("AllowUI", policy =>
        policy.WithOrigins("https://myapp.com")
              .AllowAnyHeader()
              .AllowAnyMethod());
 });
```

### 11) Path Traversal

* Accessing files outside intended directories
* Impact : Sensitive file access, Config leakage
* Prevention : Never accept raw file paths e.g NAS only with required service account
* WAF path traversal rules

```csharp

CDN / WAF
 ├─ Blocks SQLi, XSS, Path Traversal
 ├─ Rate limits malicious payloads
 ├─ Signature + behavioral rules
 |
ASP.NET Core
 ├─ Validation
 ├─ Authentication & Authorization
 ├─ Secure coding practices
 |
Database
 ├─ Least privilege
 ├─ Read/write separation

 ```

 ### 12) Rate Limiting

 ### 13) Use Secure Headers

Add headers like:

* X-Content-Type-Options
* X-Frame-Options
* Content-Security-Policy
* Referrer-Policy

### 14) Secure Coding Practices

* Nullable reference types
* Avoid hardcoded credentials
* Dispose resources
* Validate file uploads
* Sanitize filenames
* Avoid deserialization risks

### 15) Secret rotation

CyberArk password rotation

### 16) 


--------------------------------------------------------------------------------------------------------------------------

## Explain Checkmarx, SonarQube, Snyk ?

| Tool      | Main Focus                   | Best At                                      |
| --------- | ---------------------------- | -------------------------------------------- |
| Checkmarx | App security testing         | Finding vulnerabilities in source code       |
| SonarQube | Code quality + some security | Bugs, code smells, maintainability, coverage |
| Snyk      | Open source & cloud security | Vulnerable dependencies, containers, IaC     |

```compare
Checkmarx = Is my own code vulnerable?
SonarQube = Is my code clean and maintainable?
Snyk = Are my packages / containers vulnerable?
```

### 1) Checkmarx:

* Checkmarx is mainly a security scanning tool for your application source code.
* Static Application Security Testing (SAST)

#### Common Findings
* SQL Injection
* XSS
* Command Injection
* Hardcoded secrets
* Insecure cryptography
* Path traversal

### 2) SonarQube:

SonarQube focuses on code quality, maintainability, and also some security rules.

#### Common Findings:

* Null reference risks
* Unused variables
* Duplicate methods
* High cyclomatic complexity
* Missing unit tests
* Bad naming conventions


### 3) Snyk:

Snyk is strongest in dependency and supply-chain security.

#### What It Does
* Scans NuGet / npm / Maven packages
* Finds known CVEs
* Container image scanning
* Infrastructure as Code scanning
* Secrets scanning

--------------------------------------------------------------------------------------------------------------------------

## What is the benefit of code first approach in EF core?

The main benefit of Code First in EF Core is that developers define entities in C# and manage database changes through migrations. It improves productivity, type safety, maintainability, version control, and works well for modern agile development.

```flow
Create C# Model
   ↓
Add DbContext
   ↓
Create Migration
   ↓
Update Database
   ↓
App Uses Database
```

### Easier to align with Domain-Driven Design (DDD)

### Migrations = Controlled Schema Changes

```bash
dotnet ef migrations add AddEmailToCustomer
dotnet ef database update
```

### Faster Development

* Developers can create models and database quickly without waiting for DBA scripts in early stages.

### Easy Refactoring

* Need to rename a property or split an entity? Update code + migration.

### Great for CI/CD

* Migrations can run in pipelines during deployment.
* Automated database updates
* Consistent environments
* Dev/Test/Prod alignment

### Use Code First when:

* Building a new application
* Developers own schema changes
* Agile iterations happen often
* You want version-controlled DB evolution

### Database First may be better when:

* Large existing legacy database already exists
* DBA team fully controls schema
* Complex stored-procedure-heavy system
* Shared database used by many systems

 --------------------------------------------------------------------------------------------------------------------------

## Microservices benefits over monoliths

### 1) Independent Deployment:

Each service can be released separately. e.g. Deploy only Payment service without redeploying Orders.

#### Benefits:
* Faster releases
* Lower deployment risk
* Smaller change scope

### 2) Independent Scaling

Scale only the busy service. e.g. Product Search gets heavy traffic, scale Search service heavily.

#### Benefits:
* Better cost efficiency
* Better performance under load

### 3) Better Fault Isolation

Failure in one service may not crash entire system. e.g. Notification service fails, checkout still works.

#### Benefits:
* Higher resilience
* Lower Downtime

### 4) Team Autonomy

Different teams can own different services.

#### Example:
* Team A → Orders
* Team B → Identity
* Team C → Billing


#### Benefits:
* Parallel development
* Less coordination bottleneck

### 5) Technology Flexibility

Different services can use different stacks when justified.

#### Example:
* Orders → .NET
* Analytics → Python
* Realtime chat → Node.js

#### Benefits:
* Use best tool for each problem.

### 6) Easier Large-System Maintenance

Smaller codebases are easier to understand than one giant application.

#### Benefits:
* Cleaner boundaries
* Easier onboarding
* Lower cognitive load

### 7) Strong Domain Boundaries

Microservices align well with business capabilities.

#### Example:
* Customer Service
* Loan Service
* Payment Service
* Notification Service

#### Benefits:
* Encourages better design using bounded contexts.

--------------------------------------------------------------------------------------------------------------------------

## Microservices Challenges:

* Distributed transactions
* Network failures
* Service discovery
* API versioning
* Observability
* DevOps maturity
* Security between services
* Data consistency
* More infrastructure cost

--------------------------------------------------------------------------------------------------------------------------

## Custom Middleware:

```flow
Request → Middleware 1 → Middleware 2 → Controller → Response
```
* In ASP.NET Core, middleware is a component in the HTTP pipeline that can inspect, modify, allow, block, or pass requests/responses to the next component.
* To create custom middleware in ASP.NET Core, create a class with a constructor accepting RequestDelegate, implement InvokeAsync(HttpContext context), add custom logic, call _next(context), and register it using app.UseMiddleware<T>().

### Step 1: Create Custom Middleware Class

```C#
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestLoggingMiddleware> _logger;

    public RequestLoggingMiddleware(
        RequestDelegate next,
        ILogger<RequestLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        _logger.LogInformation("Incoming Request: {Method} {Path}",
            context.Request.Method,
            context.Request.Path);

        await _next(context); // call next middleware

        _logger.LogInformation("Outgoing Response: {StatusCode}",
            context.Response.StatusCode);
    }
}
```

### Step 2: Cleaner Extension Method (Recommended)

```C#
public static class MiddlewareExtensions
{
    public static IApplicationBuilder UseRequestLogging(
        this IApplicationBuilder app)
    {
        return app.UseMiddleware<RequestLoggingMiddleware>();
    }
}
```

### Step 3: Register Middleware in Program.cs

```C#
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.UseRequestLogging(); // With extension method
// app.UseMiddleware<RequestLoggingMiddleware>();  // Without extension method

app.MapControllers();

app.Run();
```

### Real Production Middleware Example Ideas

#### 1) Correlation ID

Add request trace id.

#### 2) Global Exception Handler

Catch all unhandled exceptions.

#### 3) Security Headers

Add: X-Frame-Options, Content-Security-Policy

--------------------------------------------------------------------------------------------------------------------------

## Middleware vs Filters:

| Feature                   | Middleware        | Filters                    |
| ------------------------- | ----------------- | -------------------------- |
| Runs For                  | All HTTP requests | MVC/API actions only       |
| Layer                     | HTTP Pipeline     | MVC Pipeline               |
| Access Controller Context | No                | Yes                        |
| Access Action Arguments   | No                | Yes                        |
| Can Short-Circuit         | Yes               | Yes                        |
| Use Per Action            | No                | Yes                        |
| Best For                  | Global concerns   | Controller/action concerns |

--------------------------------------------------------------------------------------------------------------------------

## Explain Request Flow in dotnet core application?

```Request flow

Client                  (Client Sends HTTP Request)
  ↓                     (Sent over network to our server)
Kestrel                 (Accepts incoming connections & Converts raw network data into HttpContext)
  ↓
HttpContext Created     (For each request, .NET creates an HttpContext)
  ↓
Middleware Pipeline     (The request enters middleware one by one in order configured in Program.cs & Dependency Injection Resolves Services)
  ↓
Routing                 
  ↓
Controller              (When route matches, controller action runs)
  ↓
Service Layer           (Controller usually calls service layer)
  ↓
Repository              (_context.Customers.Find(id); => SQL generated and sent to database)
  ↓
Database                (SELECT * FROM Customers WHERE Id = 10 => Database returns result)
  ↓
Data Returned
  ↓
JSON Serialization      (Object converted to JSON using System.Text.Json)
  ↓
Middleware Response Flow (Response Travels Back Through Middleware)
  ↓
Kestrel                  (Final response sent over network to browser/Postman/mobile)
  ↓
Client Receives Response

{
  "id":10,
  "name":"Rishav"
}

```

--------------------------------------------------------------------------------------------------------------------------
## Data Annotation vs Fluent API:

| Feature              | Data Annotation | Fluent API |
| -------------------- | --------------- | ---------- |
| Simple validation    | ✅               | ✅          |
| MaxLength            | ✅               | ✅          |
| Relationships        | Limited         | ✅ Strong   |
| Composite Key        | ❌               | ✅          |
| Indexes              | Limited         | ✅          |
| Advanced Mapping     | ❌               | ✅          |
| Cleaner Domain Model | ❌               | ✅          |

### Eg. Data Annotation

```c#
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; }
    public List<Order> Orders { get; set; }
}

public class Order
{
    public int Id { get; set; }
    public int CustomerId { get; set; }
    public Customer Customer { get; set; }
}


protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Customer>(entity =>
    {
        entity.ToTable("Customers");

        entity.HasKey(x => x.Id);

        entity.Property(x => x.Name)
              .IsRequired()
              .HasMaxLength(100);
    });

    modelBuilder.Entity<Order>(entity =>
    {
        entity.HasKey(x => x.Id);

        entity.HasOne(x => x.Customer)
              .WithMany(x => x.Orders)
              .HasForeignKey(x => x.CustomerId);
    });
}

// Seed data

entity.HasData(
    new Customer { Id = 1, Name = "Rishav" }
);

```

### Why Fluent API over attributes?

More powerful, centralized, cleaner.

### Can both be used together?

Yes. Fluent API overrides annotations if conflict occurs.

### Where does it execute?

When EF builds the model at startup / first use.

--------------------------------------------------------------------------------------------------------------------------

## Async vs Parallelism

| Topic                      | Async                        | Parallelism                             |
| -------------------------- | ---------------------------- | --------------------------------------- |
| Goal                       | Don’t block while waiting    | Finish work faster using multiple cores |
| Best For                   | I/O operations               | CPU-intensive operations                |
| Uses Threads Continuously? | Usually no while waiting     | Yes, active workers                     |
| Scales Web APIs?           | ✅ Excellent                  | ⚠️ Limited if overused                  |
| Example                    | DB call, HTTP call, file I/O | Image processing, hashing, calculations |


--------------------------------------------------------------------------------------------------------------------------

## .NET Core (ASP.NET Core) application performance

### 1) Use Async:

* Use async/await for I/O operations (DB, API calls)
* Prevents thread starvation → improves scalability
* Avoid blocking calls (.Result, .Wait())

```csharp
public async Task<IActionResult> GetOrders()
{
    var orders = await _context.Orders.ToListAsync();
    return Ok(orders);
}
```

### 2) Threading & Concurrency:


### 3) Minimize Allocations:

* Avoid unnecessary object creation
* Use Span<T>, Memory<T> where needed
* Reuse objects (e.g., HttpClient) using IHttpClientFactory

```csharp
// Span<T>, Memory<T>

int[] numbers = { 1, 2, 3, 4, 5 };

Span<int> span = numbers;
span[0] = 100;

Console.WriteLine(numbers[0]); // 100 It modifies original array because span is just a view.

// Memory<T>

Memory<T> is similar to Span<T>, but it can live on the heap and be used in:

async methods
await
fields
properties
long-lived objects

int[] data = { 1, 2, 3, 4, 5 };

Memory<int> memory = data;
Memory<int> part = memory.Slice(1, 2);

Span<int> span = part.Span;

Console.WriteLine(span[0]); // 2

```

| Feature               | Span<T> | Memory<T>       |
| --------------------- | ------- | --------------- |
| Stack only            | ✅       | ❌               |
| Heap storage          | ❌       | ✅               |
| Async safe            | ❌       | ✅               |
| Fastest access        | ✅       | Slightly slower |
| Can be field/property | ❌       | ✅               |
| Use in sync methods   | ✅       | ✅               |


```csharp

//IHttpClientFactory

builder.Services.AddHttpClient();

public class ApiService
{
    private readonly HttpClient _client;

    public ApiService(IHttpClientFactory factory)
    {
        _client = factory.CreateClient();
    }

    public async Task<string> GetData()
    {
        return await _client.GetStringAsync("https://api.example.com");
    }
}
```

### 4) Response Compression:

Reduces payload size → faster network transfer

```csharp
services.AddResponseCompression();
```

### 4) Use Efficient Serialization:

Prefer System.Text.Json over Newtonsoft as: No extra package required, Span<T> and Low allocations, Faster serialization/deserialization

### 5) Caching Strategy:

#### Types:
* Response caching
* Data caching
* Output caching (.NET 7+)

### 5) API & Network Optimization:

* Use Pagination

```csharp
.Skip(page * size).Take(size)
```

* Enable HTTP/2 + Kestrel tuning

```csharp
"Kestrel": {
  "Limits": {
    "MaxConcurrentConnections": 100
  }
}
```

* gRPC:

Faster due to binary protocol


### 6) Azure-Specific Optimization:

* Use Auto Scaling
* Static files → Azure CDN
* Use Azure Front Door / API Management : Caching + load balancing


### 7) Benchmarking & Profiling:

* dotnet-counters
* dotnet-trace
* PerfView
* Application Insights


### 8) Use Minimal APIs (for lightweight services)

### 9) Use Background Processing

Offload heavy tasks:

* Azure Service Bus

### 10) Use Event-Driven Architecture

--------------------------------------------------------------------------------------------------------------------------

## Database performance tuning

### 1) Indexing:

#### Create indexes on:

* WHERE conditions
* JOIN columns
* ORDER BY, GROUP BY

#### Types:

* Clustered Index → primary sorting
* Non-clustered Index → fast lookups
* Composite Index → multi-column queries

### 2) Normalize & Denormalize

* Normalization → avoids redundancy
* Denormalization → improves read performance (for heavy read systems)

### 3) Proper Data Types

Use correct types:

* INT instead of VARCHAR
* DATETIME2 instead of DATETIME

### 4) Query Optimization

* Avoid SELECT *
* Use only required columns
* Avoid N+1 Queries

```csharp
// Bad (multiple DB calls)

foreach (var user in users)
{
    var orders = context.Orders.Where(o => o.UserId == user.Id);
}

// Good (single query)

var data = context.Users
    .Include(u => u.Orders)
    .ToList();
```

### 5) .NET / Entity Framework Optimization

Use AsNoTracking (for read-only)

```csharp
var users = context.Users.AsNoTracking().ToList();
```

### 6) Monitoring & Diagnostics

* SQL Server Profiler
* Azure Application Insights

#### Metrics to Watch
* Slow queries
* Deadlocks
* CPU usage
* IO waits

## Common Mistakes
* Missing indexes
* Using SELECT *
* Not using pagination
* Ignoring execution plans
* Overusing EF without optimization
* Not caching frequently accessed data

### If your app is slow:

* Check slow queries (Profiler / Query Store)
* Add indexes
* Optimize queries
* Add caching
* Optimize EF (AsNoTracking, Includes)
* Scale DB (replicas / partitioning)
--------------------------------------------------------------------------------------------------------------------------

## Lifecycle of middleware in dotnet core:

* The HTTP request travels downward through middleware in the order they are registered.
* The HTTP response travels upward back through the same middleware in reverse order.
* Above happens because each middleware calls await next() and then continues executing after the next middleware finishes.

### Analogy:

Like opening nested boxes:

```flow
Open Box A
 Open Box B
  Open Box C
   Item
  Close Box C
 Close Box B
Close Box A
```

Middleware are small components connected in order. Each middleware can:

* Inspect request
* Modify request
* Call next middleware
* Short-circuit pipeline
* Inspect/modify response
* Handle exceptions
* Log timing/security data

### Step 1 — Request Arrives

* GET /api/loan/10
* Kestrel (web server) receives it.

### Step 2 — Pipeline Starts

ASP.NET Core creates HttpContext containing:

* Request
* Response
* User
* Headers
* Services
* Items
* Cancellation token

### Step 3 — Middleware Executes in Order

#### A) Exception Middleware

Wraps downstream pipeline. If any later middleware throws, it catches and returns friendly error.

```csharp
app.UseExceptionHandler(...)
```

#### B) HTTPS Redirection

Redirects HTTP to HTTPS.

```csharp
app.UseHttpsRedirection();
```

#### C) Static Files

If request is /logo.png, it may serve file directly and stop pipeline. This is called short-circuiting.

```csharp
app.UseStaticFiles();
```

#### D) Routing

Matches URL to endpoint metadata.

#### Example:
/api/loan/10 → LoanController.Get(id)

```csharp
app.UseRouting();
```

#### E) Authentication

```csharp
app.UseAuthentication();
```

Validates cookie/JWT token and sets:

* HttpContext.User

#### F) Authorization

```csharp
app.UseAuthorization();
```

Checks permissions like:

```csharp
[Authorize]
[Authorize(Policy = "CanApproveLoan")]
```

#### G) Endpoint Execution

```csharp
app.MapControllers();
```

* Controller action runs.
* return Ok(data);

--------------------------------------------------------------------------------------------------------------------------

## Example request-down / response-up behavior with two middleware:

```csharp
app.Use(async (ctx, next) =>
{
    Console.WriteLine("A Request");
    await next();
    Console.WriteLine("A Response");
});

app.Use(async (ctx, next) =>
{
    Console.WriteLine("B Request");
    await next();
    Console.WriteLine("B Response");
});

app.Run(async ctx =>
{
    Console.WriteLine("Endpoint");
    await ctx.Response.WriteAsync("Done");
});

```

### Output Order:

* A Request
* B Request
* Endpoint
* B Response
* A Response

--------------------------------------------------------------------------------------------------------------------------

## Design patterns in microservices ?

Microservices commonly use many design patterns to solve problems like service communication, resiliency, data consistency, deployment, observability, and scaling. It has a collection of patterns used together.

### Categories of Microservices Patterns
* Decomposition patterns
* Communication patterns
* Data patterns
* Resiliency patterns
* Observability patterns
* Deployment patterns
* Security patterns

### Decomposition patterns

#### 1) Decompose by Business Capability

* Split services based on business functions/capabilities.
* Focus: organizational/business responsibilities
* Departments in a company : HR, Finance, Sales

* Loan Service
* Customer Service
* Payment Service
* Notification Service

#### 2) Bounded Context (DDD)

* A boundary where a domain model has consistent meaning, rules, and language. 
* Focus: model consistency and domain language
* Same word means different things in each department : Bank account, User account etc.

#### The word Customer may mean different things in:
* CRM Context
* Billing Context
* Account

#### 3) Strangler Fig Pattern

* Gradually replace monolith modules with microservices
* Old Monolith -> Route Loan Module -> New Loan Service
* Best for migrations.

### Communication Patterns

#### 1) API Gateway Pattern

Single entry point for clients.
Example: Microsoft Azure API Management, YARP, Ocelot.

#### Responsibilities:

* Routing
* Authentication
* Rate limiting
* Aggregation
* Logging

#### 2) Synchronous Request/Response

* REST or gRPC between services.
* Use when immediate response is required.

#### 3) Asynchronous Messaging

* Better decoupling.

Use queues/events via:

* Microsoft Azure Service Bus
* RabbitMQ
* Apache Kafka


### Data Patterns

#### 1) Database per Service

* Each microservice owns its own database.
* Prevents tight coupling.

```database
Loan Service -> Loan DB
Customer Service -> Customer DB
```

#### 2) CQRS (Command Query Responsibility Segregation)

Separate:
* Writes (commands)
* Reads (queries)

Useful for high-scale systems.

#### 3) Event Sourcing

Store state changes as events.

* LoanCreated
* LoanApproved
* LoanDisbursed

Can rebuild state later.

#### 4) Saga Pattern:

* Manages distributed transactions across services.
* Instead of one DB transaction.

Example loan flow:

* Create application
* Reserve funds
* Send documents
* If step fails → compensate previous steps

Types:

* Choreography
* Orchestration

### Resiliency Patterns

#### 1) Retry Pattern

* Retry temporary failures.
* Used with Polly / resilience libraries.

#### 2) Circuit Breaker

* Stop calling failing service temporarily.
* Prevents cascading failures.

#### 3) Timeout

Fail fast if dependency is slow.

### Observability Patterns

#### 1) Centralized Logging

* Collect logs from all services.
* Example: ELK, Seq, Application Insights.

#### 2) Distributed Tracing

* Track one request across many services.
* Trace ID flows through services.
* Example: OpenTelemetry.

#### 3) Health Checks

* Expose readiness/liveness endpoints.
* Used by Kubernetes, Aspire

### Deployment Patterns

#### 1) Blue-Green Deployment

Two environments:

* Blue = current
* Green = new
* Switch traffic safely.

#### 2) Canary Deployment

Release to small % of users first.

### Security Patterns

#### 1) Central Authentication

Use identity provider:

* Microsoft Entra ID
* IdentityServer
* OAuth providers

Token Propagation
* JWT passed between services securely.

### Outbox Pattern (Very Important)

#### Problem:
DB save succeeds but event publish fails.

#### Solution:
Save event in DB outbox table, publish later reliably.
Used in enterprise systems.

### Common Patterns in .NET Microservices

For ASP.NET Core:

* API Gateway
* HttpClientFactory
* Retry + Circuit Breaker
* Background Worker
* Outbox Pattern
* CQRS + MediatR
* Saga
* Distributed Cache
* Health Checks

--------------------------------------------------------------------------------------------------------------------------

## SAGA 

* The Saga pattern is used in microservices to manage a business transaction that spans multiple services without using a distributed database transaction.
* With Azure Service Bus, Saga is commonly implemented using messages, queues/topics, events, and compensating actions.
* This is very useful in banking, e-commerce, loan processing, and order workflows.

### Why Saga Is Needed ?

In a monolith with one database:

```flow
BEGIN TRANSACTION
Step1
Step2
Step3
ROLLBACK/COMMIT
```

In microservices:

* Each service has its own database
* No single DB transaction across services
* Need eventual consistency
* Saga solves this

## Real Flow (Orchestration)

### Step 1 — API Receives Request

```flow
POST /loan/apply
```

* Loan service stores application.
* Publishes command: VerifyDocuments

### Step 2 — Document Service

* Consumes message
* If success: Publish DocumentsVerified
* If fail: Publish DocumentsRejected

### Step 3 — Credit Service

* Consumes next command/event.
* Publishes:CreditApproved
* or failure event.

### Step 4 — Funding Service

* Reserves funds.
* If fails: FundsReservationFailed

### Step 5 — Compensation

* Each service performs its own compensating action.
* Need to track progress

#### If funds reservation fails:

* Mark loan rejected
* Release temporary holds
* Notify customer

#### E.g. Instead of rollback SQL transaction:

* Undo Document Verification Status
* Undo Credit Reservation
* Set Loan Status = Failed


### Advantages of SAGA pattern:

#### Works Across Multiple Services

* Each microservice owns its own database.
* Saga lets these services participate in one business workflow without sharing a single DB transaction.

#### Avoids Distributed Transactions

* Traditional 2-phase commit across services is:Complex, Slow
* Saga avoids this by using local transactions + events/messages.

#### Loose Coupling

* Services communicate through messages/events.
* They do not need tight runtime dependency on one shared transaction manager.

#### Fault Tolerance

If one step fails:

* Retry
* Compensate previous steps
* Continue safely

#### Eventual Consistency

* Instead of instant global consistency, the system becomes consistent over time.
* This is practical and common in distributed systems.

#### Better Business Traceability

* ApplicationCreated
* DocumentsVerified
* CreditApproved
* FundsReserved
* LoanApproved

--------------------------------------------------------------------------------------------------------------------------

### Why Azure Service Bus Helps

#### Reliable Delivery
Messages persisted

#### Retry Support
Transient failures recover

#### Dead Letter Queue
Poison messages isolated

#### Duplicate Detection
Avoid repeated processing

#### Sessions
Keep related messages grouped

--------------------------------------------------------------------------------------------------------------------------

## Container App Deployment:

### 1) Deploy from VS

```docker
VS:
Dot net project -> docker file
Publish> Add profile > push image to Azure Container Registry with tag 'latest'
Authenticate VS using Azure account credential -> List Registry from Azure
Publish


Azure:
Azure container Registry created
Access Keys : 
  Registry name and Login server
  Admin User : UserName and Password
Once Publish from VS, Repository is created in ACR
Container should be associated to App Service:
 - App Service > Create Web App > Subscription > Resource Group > Publish > Container
 - Image Source = ACR
 - Select Registry, Image, Tag 'latest', any startup command

```

### 2) Deploy from Azure CLI

```docker
Azure CLI: To push image to ACR

az login
 - popup browser for authentication
 - show user id and details if successfull login

az acr login -n "Username" 

az acr build --image "repostitory image name" --registy "Container registry name(login server)" "Docker file path name/dockerfile ."

```

--------------------------------------------------------------------------------------------------------------------------
## Blocking vs Deadlock vs Timeout

| Feature            | Blocking           | Deadlock                  | Timeout         |
| ------------------ | ------------------ | ------------------------- | --------------- |
| Cause              | Resource busy      | Circular wait             | Wait too long   |
| Needs Lock?        | Usually yes        | Yes / resource contention | Not always      |
| Automatic Recovery | Wait until free    | DB kills one victim       | Operation fails |
| Temporary?         | Often yes          | No, must break cycle      | Ends with error |
| Common In          | Databases, threads | Databases, threads        | DB, HTTP, APIs  |


### Blocking:
A holds lock → B waits

### Deadlock:
* A waits for B
* B waits for A

### Timeout:
B waits too long → operation aborts

--------------------------------------------------------------------------------------------------------------------------

## Explicit Transaction in EF core

```Csharp
using var tx = await db.Database.BeginTransactionAsync();

try
{
    await db.SaveChangesAsync();

    // more changes
    await db.SaveChangesAsync();

    await tx.CommitAsync();
}
catch
{
    await tx.RollbackAsync();
}
```

--------------------------------------------------------------------------------------------------------------------------
## How to add Retry Logic for Transient Failures ?

```Csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(
        connectionString,
        sqlOptions =>
        {
            sqlOptions.EnableRetryOnFailure(
                maxRetryCount: 5,
                maxRetryDelay: TimeSpan.FromSeconds(10),
                errorNumbersToAdd: null);
        }));

```


```Csharp
builder.Services.AddHttpClient("payment")
    .AddTransientHttpErrorPolicy(policy =>
        policy.WaitAndRetryAsync(3,
            retryAttempt => TimeSpan.FromSeconds(retryAttempt)));
```
--------------------------------------------------------------------------------------------------------------------------

## OpenTelemetry in dotnet core app ?

* OpenTelemetry in .NET Core is used for observability by collecting traces, metrics, and logs from the application. It helps monitor performance, diagnose issues, and trace requests across distributed systems. In ASP.NET Core, it is configured using AddOpenTelemetry() with instrumentations like ASP.NET Core, HttpClient, and SqlClient.

In ASP.NET Core, OpenTelemetry is an open standard for observability. It helps you collect and export:

* Logs → what happened
* Metrics → how much / how fast
* Traces → where request traveled

It is widely used in modern cloud and microservices systems.



```Pillars
1. Traces

Track a request across services.

Example:

Request started
Payment API call
SQL query
Response returned
2. Metrics

Numerical measurements:

Request count
CPU
Memory
Latency
Error rate
3. Logs

Structured events:

Exceptions
Warnings
Business events
```

### OpenTelemetry is commonly used to collect telemetry from your apps and send it to Azure monitoring tools such as:

* Azure Monitor
* Application Insights (builder.Configuration["APPLICATIONINSIGHTS_CONNECTION_STRING"];)
* Log Analytics

```c#
using OpenTelemetry.Trace;
using OpenTelemetry.Metrics;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddSqlClientInstrumentation()
            .AddConsoleExporter();
    })
    .WithMetrics(metrics =>
    {
        metrics
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddConsoleExporter();
    });

var app = builder.Build();
app.MapGet("/", () => "Hello");
app.Run();

```

```c#
using OpenTelemetry.Trace;
using OpenTelemetry.Metrics;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOpenTelemetry()
    .WithTracing(t =>
    {
        t.AddAspNetCoreInstrumentation()
         .AddHttpClientInstrumentation()
         .AddSqlClientInstrumentation()
         .AddAzureMonitorTraceExporter(options =>
         {
             options.ConnectionString =
                 builder.Configuration["APPLICATIONINSIGHTS_CONNECTION_STRING"];
         });
    })
    .WithMetrics(m =>
    {
        m.AddAspNetCoreInstrumentation()
         .AddHttpClientInstrumentation()
         .AddAzureMonitorMetricExporter(options =>
         {
             options.ConnectionString =
                 builder.Configuration["APPLICATIONINSIGHTS_CONNECTION_STRING"];
         });
    });
```
--------------------------------------------------------------------------------------------------------------------------
## Azure Key Vault ?

* Store secrets in Key Vault → App authenticates using Managed Identity → Read secret securely at runtime.
* This avoids storing passwords in appsettings.json, source code, or pipelines.

```flow
Store Secret in Azure Key Vault
        ↓
Enable Managed Identity on App Service / VM
        ↓
Give App Access to Key Vault using RBAC or Access policy
        ↓
.NET App uses DefaultAzureCredential()
        ↓
Reads password at runtime
```

```C#
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;

var vaultUri = new Uri("https://yourvaultname.vault.azure.net/");

var client = new SecretClient(
    vaultUri,
    new DefaultAzureCredential());

KeyVaultSecret secret = await client.GetSecretAsync("DbPassword");

string password = secret.Value;
```

--------------------------------------------------------------------------------------------------------------------------

## What are the test which can run in CI/CD pipeline ADO ?

```flow
Commit / PR
   ↓
Build
   ↓
Unit Tests
   ↓
Code Quality / Security
   ↓
Integration Tests
   ↓
API / UI Tests
   ↓
Performance Tests
   ↓
Deploy
   ↓
Smoke / Regression Tests
```

### Unit Tests

```yaml
- task: DotNetCoreCLI@2
  inputs:
    command: test
    projects: '**/*Tests.csproj'
```

```C# 
using Xunit;

public class CalculatorTests
{
    [Fact]
    public void Add_ReturnsSum()
    {
        // Arrange
        var calculator = new Calculator();
        int a = 2;
        int b = 3;

        // Act
        var result = calculator.Add(a, b);

        // Assert
        Assert.Equal(5, result);
    }
}
```

```C# 
using NUnit.Framework;

[TestFixture]
public class CalculatorTests
{
    [Test]
    public void Add_ReturnsSum()
    {
        // Arrange
        var calculator = new Calculator();
        int a = 2;
        int b = 3;

        // Act
        var result = calculator.Add(a, b);

        // Assert
        Assert.AreEqual(5, result);
    }
}
```

```C# 

// Mocking

using Xunit;
using Moq;

public class UserServiceTests
{
    [Fact]
    public void GetWelcomeMessage_ReturnsExpectedMessage()
    {
        // Arrange
        var mockRepo = new Mock<IUserRepository>();

        mockRepo.Setup(x => x.GetUserName(1))
                .Returns("Rishav");

        var service = new UserService(mockRepo.Object);

        // Act
        var result = service.GetWelcomeMessage(1);

        // Assert
        Assert.Equal("Welcome Rishav", result);
    }
}
```

| Feature                           | xUnit                                  | NUnit                   |
| --------------------------------- | -------------------------------------- | ----------------------- |
| Popular in modern .NET Core       | ✅ Very high                            | ✅ High                  |
| Microsoft samples/community usage | ✅ Strong                               | ✅ Strong                |
| Test attribute                    | `[Fact]`, `[Theory]`                   | `[Test]`, `[TestCase]`  |
| Setup/Teardown style              | Constructor / `IDisposable` / fixtures | `[SetUp]`, `[TearDown]` |
| Parameterized tests               | `[Theory]` + `InlineData`              | `[TestCase]`            |
| Parallel support                  | ✅ Good                                 | ✅ Good                  |
| Assertions                        | `Assert.Equal()`                       | `Assert.AreEqual()`     |
| Learning curve                    | Simple                                 | Very approachable       |
| Legacy .NET Framework usage       | Good                                   | Very strong             |
| Extensibility                     | Strong                                 | Strong                  |

--------------------------------------------------------------------------------------------------------------------------
## CI/CD

```YAML
trigger:
- main

steps:
- task: UseDotNet@2

- script: dotnet restore
- script: dotnet build --configuration Release
- script: dotnet test --collect:"XPlat Code Coverage"

- script: dotnet publish
```

```flow
CI Stage:
Build solution
Unit tests
SonarQube scan
Snyk dependency scan

CD Stage (QA):
Deploy
Integration tests
API tests
Smoke tests

Pre-Prod:
Performance tests
Regression suite

Prod:
Blue/Green deploy
Smoke tests
```
--------------------------------------------------------------------------------------------------------------------------