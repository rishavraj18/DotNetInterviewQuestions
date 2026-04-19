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

