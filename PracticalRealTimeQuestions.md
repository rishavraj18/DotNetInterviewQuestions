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

* Report generation and Audit monthly, quarterly, yearly.

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

#### Violation Smell:

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