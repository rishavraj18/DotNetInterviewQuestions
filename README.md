# DotNetInterviewQuestions

## Difference between .NET Framework & .NET Core

| .NET Framework     | .NET Core / .NET                              |
| ------------------ | --------------------------------------------- |
| Windows-only       | Cross-platform (Windows, Linux, Mac)          |
| Not open-source    | Fully open-source                             |
| Monolithic         | Modular                         |
| Slower performance | High performance (Kestrel, JIT optimizations) |
| Legacy             | Modern / future platform                      |


-------------------------------------------------------------

## ASP.NET Core runtime & architecture

ASP.NET Core handles requests through a pipeline of middleware components. Each middleware either handles the request or calls the next middleware. The pipeline is built in Program.cs using app.Use..., app.Run, app.Map etc.<br/>

### Key points:<br/>

* app.Use(...) registers middleware that can call await next() to continue pipeline.
* app.Run(...) registers terminal middleware — it does not call next.
* app.Map(path, branch) branches the pipeline for a URL prefix.
* Middleware order matters — authorization must appear after authentication, exception handling early, static files before MVC if you want static to serve first.

### Kestrel & reverse proxies<br/>

* Kestrel is the cross-platform web server shipped with ASP.NET Core. It can be exposed directly to the internet but in production many use a reverse proxy (Nginx/IIS/Apache) for features like TLS termination, process management, static file caching, or advanced routing.
* Reverse proxy + Kestrel is recommended for Windows/IIS scenarios or when you need web server features not provided by Kestrel.

### Hosting vs background services

* IHostedService is the primitive for background services. BackgroundService is a helper abstract class implementing IHostedService that provides a long-running task hook (ExecuteAsync).
* Use IHostedService for lifecycle control; BackgroundService for polling jobs, timed tasks, etc. Be careful with shutting down gracefully (honor CancellationToken).

-------------------------------------------------------------

## Explain Hosting vs background services in ASP.NET Core ?

### Hosting is the infrastructure layer that initializes and manages the lifetime of an ASP.NET Core application. Environment where your entire ASP.NET Core app runs.

#### Hosting is responsible for:

* Starting the web server (Kestrel)
* Reading configuration (appsettings.json, environment vars)
* Registering services (DI container)
* Middleware pipeline creation
* Logging infrastructure setup
* Lifetime management (start → run → stop)
* Running hosted background services (IHostedService, BackgroundService)


#### Hosting is configured using:

 ```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();
app.Run();
 ```

### Background services are long-running tasks that run in the background alongside the web app. Long-running worker running inside the host lifecycle.

#### Implemented using:

* IHostedService
* BackgroundService (recommended)
* PeriodicTimer
* Worker Service Template (.NET Generic Host)


#### Examples:

* Sending emails
* Processing a queue (e.g., RabbitMQ, Kafka)
* Running scheduled jobs
* Cleaning expired files/tokens
* Caching refresh tasks


#### Example BackgroundService:

```csharp
public class EmailProcessor : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            Console.WriteLine("Processing emails...");
            await Task.Delay(5000, stoppingToken);
        }
    }
}
```

Register:<br/>

```csharp
builder.Services.AddHostedService<EmailProcessor>();
```


| Feature             | Hosting                               | Background Services                           |
| ------------------- | ------------------------------------- | --------------------------------------------- |
| **Purpose**         | Manages entire app startup & lifetime | Runs long-running or periodic tasks           |
| **Scope**           | Whole application                     | A single worker/task                          |
| **Runs**            | Once at application start             | Entire application lifetime but in background |
| **Created using**   | WebApplication / HostBuilder          | IHostedService / BackgroundService            |
| **Responsible for** | Kestrel, DI, Logging, Config          | Queue processing, jobs, recurring tasks       |
| **Stops when**      | Application stops                     | Application stops or service cancelled        |
| **Example**         | `app.Run()`                           | Email sender, RabbitMQ consumer               |



-------------------------------------------------------------

## What is Collection ?

A collection in .NET is an object/ data type that stores and manages groups of related items (values or objects). 
Collections make it easy to add, remove, search, sort, and iterate over items.

### Types:

### A) Non-Generic Collections

* ArrayList
* Hashtable
* Queue
* Stack


### B) Generic Collections

| Collection                 | Description    |
| -------------------------- | -------------- |
| `List<T>`                  | Dynamic array  |
| `Dictionary<TKey, TValue>` | Key-value pair |
| `HashSet<T>`               | Unique items   |
| `Queue<T>`                 | FIFO           |
| `Stack<T>`                 | LIFO           |
| `SortedList<TKey, TValue>` | Sorted by key  |

------------------------------------------------------------

## Why to prefer Generic Collections?

Because they are:

* Type-safe
* Faster (no boxing/unboxing)
* More flexible
* Easier to use with LINQ

------------------------------------------------------------

## What is a string in C#? Why is it immutable?

* Memory and performance optimization through interning.
* If strings were mutable:
* Changing one would accidentally change others referencing it.
* Thread safety

------------------------------------------------------------

## Array vs List

### Array

* Fixed-size collection of items.
* Fastest index-based data structure.
* You need high performance
* Best for fixed-size data.
* No resize overhead.

### List<T>

* A dynamic, resizable collection built on top of an array.
* Automatically grows as items are added.
* Prefer flexibility over raw speed
* Internally uses an array that auto-grows.
* When List grows, it:
     Allocates a new larger array
     Copies old data → costly operation

| Feature          | Array               | List<T>                            |
| ---------------- | ------------------- | ---------------------------------- |
| Size             | Fixed               | Dynamic (auto-resizes)             |
| Type Safety      | Yes                 | Yes                                |
| Performance      | Faster for indexing | Slight overhead                    |
| Memory           | Less memory         | Extra memory allocated             |
| Flexibility      | Less flexible       | More flexible                      |
| Built-in Methods | Few                 | Many (Add, Remove, Contains, etc.) |
| Usage            | Simple data storage | Complex dynamic collections        |
| LINQ Support     | Works but limited   | Works naturally                    |

------------------------------------------------------------

## List<T> vs ICollection vs IEnumerable

* IEnumerable<T> → Read-only iteration (You can loop through items)

* ICollection<T> → Add, remove, count items (Basic collection operations)

* List<T> → A concrete, resizable array with full features, AddRange, RemoveAt(), Sort(), Find()

### Hierarchy Diagram
IEnumerable<T><br/>
      ↑<br/>
 ICollection<T><br/>
      ↑<br/>
    List<T>   (Concrete class)


| Feature            | IEnumerable<T> | ICollection<T> | List<T>      |
| ------------------ | -------------- | -------------- | ------------ |
| Supports iteration | ✔              | ✔              | ✔            |
| Add/Remove items   | ❌              | ✔              | ✔            |
| Count              | ❌              | ✔              | ✔            |
| Index access       | ❌              | ❌              | ✔            |
| Lazy evaluation    | ✔              | ✔              | ✔            |
| Interface or class | Interface      | Interface      | Class        |
| Modify collection  | ❌              | ✔              | ✔            |
| When to use        | Read-only      | Modify items   | Full control |



* Return IEnumerable<T> when:

You want to give read-only access
LINQ deferred execution is needed

* Return ICollection<T> when:

You want to allow collection modification
But want to hide implementation (good abstraction)

* Return List<T> when:

Caller needs indexing
You need List-specific features

------------------------------------------------------------

## How concurrent request is handled in asp.net core app ? Which Tool is popular ?

JMeter & K6 is the tool used for testing mostly.

In ASP.NET Core, concurrency is handled through a combination of threading, async I/O, Kestrel’s event loop, and request isolation.

ASP.NET Core handles concurrent requests using the Kestrel web server’s event-driven non-blocking I/O model. 
Each request is executed on a CLR ThreadPool thread and when asynchronous operations occur, the thread is released to serve other requests. 
ASP.NET Core ensures request isolation with Scoped DI, while concurrency in shared state must be managed using thread-safe constructs like 
ConcurrentDictionary or locks.


### Kestrel
ASP.NET Core apps run on Kestrel, a high-performance, asynchronous server.
Kestrel uses an I/O Completion Ports (IOCP) model so asynchronous operations don’t block threads.

ASP.NET Core / Kestrel:<br/>

1 request does not require a dedicated thread<br/>
Threads exist only during CPU execution

### Thread Pool – Request Execution

When the request reaches your controller/middleware:<br/>

* It is executed on a CLR ThreadPool thread.
* ASP.NET Core reuses these threads to avoid creating new ones.
* If your code uses async/await, the thread is released during I/O (DB query, HTTP call, file read).

📌 Result:<br/>
Other requests can use the same thread → high concurrency.

### Async/Await Improves Concurrency

await _dbContext.Users.ToListAsync();


During the await:<br/>

✔ The thread is returned to the thread pool<br/>
✔ Kestrel serves other requests<br/>
✔ When the I/O completes, the continuation runs on a thread pool thread<br/>

📌 Important:<br/>
Async provides concurrency even with few threads.<br/>

### Request Isolation (No Shared State)

ASP.NET Core ensures one request cannot affect another because:<br/>

* Each request has its own HttpContext
* Scoped services are created per request
* No shared global state unless explicitly added (Singletons)


### Middleware Pipeline Executes Concurrently

Request → Middleware 1 → Middleware 2 → Controller → Response<br/>

Each request gets its own pipeline execution, so concurrency is not blocked unless:<br/>

* You use lock
* You do blocking calls (.Result or .Wait())
* You call a Singleton service with shared state

### You scale horizontally:

* Kubernetes pods
* Docker containers
* Load balancers (NGINX, Azure LB, AWS ELB)
* Autoscaling based on CPU/RPS


### Handling Shared State (Thread Safety)

| Scenario           | Safe Mechanism                            |
| ------------------ | ----------------------------------------- |
| Shared collections | `ConcurrentDictionary`, `ConcurrentQueue` |
| Shared counters    | `Interlocked.Increment`                   |
| Shared operations  | `lock` keyword                            |
| EF DbContext       | **Not thread-safe**, use Scoped           |

<br/>

ASP.NET Core achieves massive scalability through a combination of:

| Operation         | Requests/sec  |
| ----------------- | ------------- |
| Static text       | 5M–7M RPS     |
| Minimal API       | 1.5M–3M RPS   |
| Simple controller | 400k–800k RPS |
| DB-backed API     | 20k–80k RPS   |


------------------------------------------------------------

## What are different type of testing and tools to perform that ?

![Types of testing in SDLC](/img/TestingTypes.JPG "Testing Types")


------------------------------------------------------------

## What is Dependency Injection in .NET Core? How does DI work internally?

ASP.NET Core has a built-in DI container with three lifetimes: Transient, Scoped, Singleton.

* Register services (what types exist and their lifetimes).
* Build a dependency graph (resolve constructor parameters).
* Create instances (construct objects in the correct order).

When Registering:
Store metadata (type → implementation → lifetime)

When Resolving:
Lookup descriptor
Find constructor
Resolve dependencies recursively
Construct object (using cached delegates)
Cache it if singleton/scoped
Track if disposable

On Scope End:
Dispose scoped and transient disposables

-------------------------------------------------------------

## Service lifetimes – Transient, Scoped, Singleton

A) Transient
Creates a new instance every time
Light-weight, stateless services
Helper classes, utilities, Formatter, validators

B)Scoped
one instance per HTTP request
Same instance reused within the same request, New request = new instance
DbContext, Unit of Work pattern, Business logic that must stay consistent during one request

Inside Request #1:
Controller: instance A
Service A: instance A
Repository: instance A

Inside Request #2:
New instance B

C) Singleton
Only one instance in the entire application lifetime
Created once, reused forever
Same instance for all requests & all users
Cache services, Logging frameworks, Configuration providers

Only singleton/transient services are injected into middleware.
Scoped services are not allowed


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

-------------------------------------------------------------

## What are the key advantages of Dependency Injection (DI)

* Loose Coupling, modularity, testability, maintainability, and flexibility
* Easy Unit Testing (Mocking becomes simple)
* Clean Code
* Better Extensibility
* Centralized Dependency Management
* Promotes SOLID Principles

-------------------------------------------------------------

## What is Captive dependency problem ?

* Happens when a singleton depends on a scoped service. The scoped dependency is "captured" and effectively promoted to singleton lifetime, causing incorrect behavior and potential memory leaks or thread-safety issues.
* Avoid by not injecting scoped services into singletons. Use factories (IServiceProvider.CreateScope) or IOptions/configuration approaches.

-------------------------------------------------------------

## What is Constructor vs property/method injection ?

* Constructor injection is preferred for required dependencies — makes dependencies explicit and immutable.
* Property/method injection used for optional or circular dependencies, but should be avoided where possible.

-------------------------------------------------------------

## When it is required to Replace DI container ?

* Built-in container is fine for most tasks. Use Autofac when you need advanced features (module scanning, property injection patterns, lifetime scopes with advanced control).
* Example of avoiding captive dependency:

```csharp
public class MySingleton
{
    private readonly IServiceProvider _sp;
    public MySingleton(IServiceProvider sp) { _sp = sp; }

    public void DoWork()
    {
        using var scope = _sp.CreateScope();
        var scopedService = scope.ServiceProvider.GetRequiredService<IMyScoped>();
        scopedService.Do();
    }
}
```

-------------------------------------------------------------

## What is “Safe” in REST?

A Safe HTTP method means:<br/>

➡ It does NOT change the server state<br/>
➡ It is read-only<br/>
➡ Repeated calls do not have side effects on data<br/>
eg. GET, HEAD<br/>

Calling below once or 100 times → no change to the resource.

```csharp
GET /api/products/10
```

-------------------------------------------------------------

## What is “Idempotent” in REST?

A method is Idempotent if:<br/>

➡ Running it once or multiple times has the same effect on the resource<br/>
➡ The final state on the server is identical<br/>
eg. PUT, DELETE, GET<br/>

1st call → Updates user → final state = John, 30<br/>
10th call → Same update → final state still = John, 30<br/>

```csharp
PUT /api/users/10
Body: { "name": "John", "age": 30 }
```

-------------------------------------------------------------

## Is POST Safe or Idempotent?

❌ POST is not safe<br/>
POST modifies server state (e.g., creates new resource)<br/>

❌ POST is not idempotent<br/>
Each call creates a new item.<br/>

Calling 3 times may create 3 orders → final state is different.<br/>

```csharp
POST /orders
```

-------------------------------------------------------------

## Idempotent vs Safe:

| Principle                  | Meaning                                            | Typical Methods  |
| -------------------------- | -------------------------------------------------- | ---------------- |
| **Safe**                   | Operation **does not change** server state         | GET, HEAD        |
| **Idempotent**             | Multiple identical requests → **same final state** | GET, PUT, DELETE |
| **All Safe = Idempotent?** | Yes                                                | GET              |
| **All Idempotent = Safe?** | No                                                 | DELETE, PUT      |


-------------------------------------------------------------

## HEAD vs OPTIONS vs TRACE

| Method      | Safe | Idempotent | Purpose                             | Typical Use                           |
| ----------- | ---- | ---------- | ----------------------------------- | ------------------------------------- |
| **HEAD**    | Yes  | Yes        | Same as GET but without body        | Check resource metadata, existence    |
| **OPTIONS** | Yes  | Yes        | Returns allowed methods + CORS info | Preflight, API capability discovery   |
| **TRACE**   | Yes  | Yes        | Echo request for debugging          | Debug proxies; disabled in production |


-------------------------------------------------------------

## What is mocking in Unit Testing ?

Mocking in NUnit means creating fake versions of dependencies 
so you can test your class without using the real implementations 
(like real database, email service, API calls, etc.).

-------------------------------------------------------------

## Why Does the Test Pass Even When No Real Email Was Sent?

using NUnit.Framework;
using Moq;

[TestFixture]
public class UserServiceTests
{
    [Test]
    public void Register_Should_Call_EmailService_Send()
    {
        // Arrange
        var mockEmailService = new Mock<IEmailService>();

        var userService = new UserService(mockEmailService.Object);

        // Act
        userService.Register("test@example.com");

        // Assert
        mockEmailService.Verify(
            x => x.Send("test@example.com", "Welcome!"), 
            Times.Once
        );
    }
}

Because unit testing is NOT meant to test external systems(like email server, SMTP, database, API).
Unit test should test only the logic inside your class, not the dependencies.

In unit testing:
We do NOT test whether an email is actually delivered
We test whether our class attempted to send the email correctly
The email sending itself is tested in integration tests, not unit tests

-------------------------------------------------------------

## Explian Program.cs files component ?

WebApplication.CreateBuilder(args) sets up everything your application needs before it starts 
running — configuration, DI, logging, and hosting environment.

Step 1 — Create Builder
var builder = WebApplication.CreateBuilder(args);

Step 2 — Register Services in DI Container
builder.Services.AddControllers();

Step 3 — Build the App
var app = builder.Build();

Step 4 — Configure Middleware
app.UseAuthorization();
app.MapControllers();

Property				Purpose
builder.Services		Register DI services
builder.Configuration	Read config files
builder.Environment		Check environment (Dev/Prod)
builder.Logging			Configure logging
builder.WebHost			Configure Kestrel, URLs, etc.

-------------------------------------------------------------

## Explain ASP.NET Core Lifecycle

        +-----------------+
        | Program.cs/Main |
        +-----------------+
                 |
                 v
        +-----------------+
        | Host Creation   |
        | (Kestrel + DI)  |
        +-----------------+
                 |
                 v
        +-----------------+
        | Startup Class   |
        | ConfigureServices |
        | Configure        |
        +-----------------+
                 |
                 v
        +------------------------+
        | Middleware Pipeline    |
        | (UseRouting, UseAuth,  |
        |  Custom Middleware...) |
        +------------------------+
                 |
                 v
        +-----------------+
        | Routing/Endpoint|
        +-----------------+
                 |
                 v
        +-----------------+
        | Controller /    |
        | Razor / API     |
        +-----------------+
                 |
                 v
        +-----------------+
        | Response Back   |
        | through Pipeline|
        +-----------------+
                 |
                 v
        +-----------------+
        | Client Receives |
        +-----------------+

-------------------------------------------------------------

## Explain the Request Pipeline in .NET Core

The request pipeline in ASP.NET Core is a sequence of middleware components that an HTTP request 
passes through before reaching an endpoint (controller, minimal API, Razor page), 
and then back out to produce the response.

ASP.NET Core creates a linked list of delegates representing each middleware in order
Incoming Request → Middleware #1 → Middleware #2 → ... → Endpoint → Response

Each middleware can:
Run logic before passing the request to the next middleware
Run logic after the next middleware has completed
Short-circuit the pipeline (stop it early)
Modify the request or response

-------------------------------------------------------------

## What is Middleware and how it executes?

A middleware is usually a class with:
A constructor that receives the next middleware delegate
An Invoke or InvokeAsync method that executes the logic

ASP.NET Core uses DI to construct middleware once at startup.
_next is the next middleware delegate in the pipeline.

public class LoggerMiddleware
{
    private readonly RequestDelegate _next;

    public LoggerMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        Console.WriteLine("Before next middleware");

        await _next(context); // Call next component

        Console.WriteLine("After next middleware");
    }
}

app.UseMiddleware<LoggerMiddleware>();

Pipeline behavior is completely determined by the order you add middleware.

Middleware components are singletons

Scoped services used inside middleware must be resolved from:
context.RequestServices

Only app.Run() creates a terminal middleware
Meaning the pipeline ends there.


### Flow of a Request

Step 1 → Kestrel receives HTTP request

Kestrel is the built-in web server.

Step 2 → A request scope is created

DI Scoped services live inside this scope.

Step 3 → Middleware pipeline begins

Each middleware is executed in order they were added.

Step 4 → Endpoint is selected (Routing)

UseRouting() inspects the request:

* Path
* HTTP method
* Route templates

It selects the matching endpoint.

Step 5 → Endpoint executes (Controller/Minimal API)

EndpointMiddleware triggers the controller or handler.

Step 6 → Response travels back up the pipeline

After the endpoint responds, control unwinds:
Endpoint → Authorization → Authentication → Routing → Exception handler

Step 7 → Scope is disposed

All scoped/transient disposables are cleaned up.

-------------------------------------------------------------
## What is Short-Circuiting the Pipeline ?

A middleware can decide NOT to call the next middleware:

app.Use(async (context, next) =>
{
    if (!context.Request.Headers.ContainsKey("Auth"))
    {
        context.Response.StatusCode = 401;
        return; // Pipeline stops here
    }

    await next();
});


Used for:
Authentication failures
Authorization failures
Blocking invalid requests
Returning static files early
CORS failures

-------------------------------------------------------------

## What are constructors?

A constructor is a special method inside a class that runs automatically 
when you create (instantiate) an object.

Its job is to:

* Initialize the object
* Set default values
* Inject dependencies

Features:

* It has same name as the class
* No return type

-------------------------------------------------------------
## Parameterized vs Non-parameterized constructors

| Feature               | Non-parameterized       | Parameterized             |
| --------------------- | ----------------------- | ------------------------- |
| **Takes arguments**   | ❌ No                    | ✔ Yes                     |
| **Provides defaults** | ✔ Yes                   | ❌ Caller must pass values |
| **Used for DI**       | Rarely                  | Very commonly             |
| **Enforces rules**    | ❌ No                    | ✔ Yes                     |
| **Flexibility**       | High                    | High but requires inputs  |
| **Auto-generated?**   | ✔ Yes (if none defined) | ❌ No                      |


-------------------------------------------------------------

## Method Overloading vs Method Overriding (examples)

| Feature              | Method Overloading                                                 | Method Overriding                                                  |
| -------------------- | ------------------------------------------------------------------ | ------------------------------------------------------------------ |
| **Concept**          | Same method name, different signatures (compile-time polymorphism) | Same method name & signature in child class (runtime polymorphism) |
| **Occurs in**        | Same class (or derived class)                                      | Only in inheritance (base → derived class)                         |
| **Parameters**       | Must be different (type, number, order)                            | Must be same                                                       |
| **Return type**      | Can be different                                                   | Must be same (or covariant)                                        |
| **Modifier needed?** | No special keyword                                                 | Needs `virtual` in base + `override` in derived                    |
| **Binding time**     | Compile-time                                                       | Runtime                                                            |
| **Purpose**          | Increase flexibility                                               | Change/extend parent behavior                                      |


-------------------------------------------------------------
## What are Delegates?

A delegate is a type-safe function pointer —
meaning it can hold a reference to a method, and you can call that method through the delegate.

A delegate is a type-safe object used to reference methods. It allows you to pass methods as parameters, store them in variables, and execute them indirectly. 
Delegates enable callbacks, events, LINQ, and functional programming in C#.

✔ Type-safe → method signature must match the delegate
✔ Can point to static or instance methods
✔ Can point to multiple methods (multicast)
✔ Foundation for events, LINQ, async, Func<>, Action<>

-------------------------------------------------------------
## Where Delegates Are Used ?

Delegates form the foundation for:
Events
LINQ expressions
Callbacks
Async/await
Middleware registration
Timer callbacks
Threading (Task.Run)

Example:

app.Use(async (context, next) => { ... });


This works because middleware takes RequestDelegate, which is a delegate type.

-------------------------------------------------------------

## What Is Asynchronous Programming? Why do we need it?

Asynchronous programming allows your application to perform tasks without blocking the main thread.
Instead of waiting for long-running operations (like database queries, file I/O, API calls), the program continues doing other work.

* Improve performance
* Increase responsiveness
* Avoid UI freezing (in desktop/mobile apps)
* Handle high-concurrency (in web apps)


### Asynchronous Return Types

| Return Type                  | Meaning                                             |
| ---------------------------- | --------------------------------------------------- |
| `Task`                       | async method that returns nothing                   |
| `Task<T>`                    | async method that returns a value                   |
| `ValueTask` / `ValueTask<T>` | optimization for high-performance                   |
| `void`                       | only for event handlers (not recommended otherwise) |


### Async in ASP.NET Core

public async Task<IActionResult> GetUsers()
{
    var users = await _service.GetUsersAsync();
    return Ok(users);
}


-------------------------------------------------------------

## Explain async-await & asynchronous programming in C#

1. The method starts running.

2. When it hits await SomeTask:
   * It returns control to the caller.
   * It lets the thread do other work.

3. When the awaited task completes:
   * The method resumes from where it stopped.

This creates the illusion of "pausing," but really nothing is blocked.

async keyword
Marks a method as asynchronous and allows await to be used inside it.

await keyword
Pauses the method until the awaited task finishes, without blocking the thread.

Return type - Task or void

-------------------------------------------------------------

## Synchronous vs Asynchronous

### Synchronous:

var data = GetData();   // program waits here
Console.WriteLine("Done"); 

### Asynchronous:

var data = await GetDataAsync(); // does NOT block
Console.WriteLine("Done");       // runs after task finishes


-------------------------------------------------------------
## Why to avoid Blocking Calls in Async Code

Never use:

* .Result
* .Wait()
* .GetAwaiter().GetResult()

These cause deadlocks and block threads

-------------------------------------------------------------

## Code First vs Database First

* Code First: 
You create C# model classes first, and EF generates the database using migrations. Best for new projects and domain-driven development.

* Database First: 
You start with an existing database, and EF generates classes from it (scaffolding). Best for legacy databases or DB-first environments.

When to use Code First:

New project (no existing database)
Domain-driven design (DDD)
Want full control over models & migrations
Agile / fast development cycles

✔ Pros:
Full control over the domain model
Easy migrations (add, update, rollback)
Cleaner, modern approach (.NET Core default)

✔ Cons:
Requires EF Core knowledge
Not ideal if database already exists
Harder when DB structure is complex or controlled by DBAs

When to use Database First:

You already have a production database
Database is large/complex
DB schema is managed by DBAs
You want models to match exactly what's in DB

✔ Pros:
Perfect for legacy databases
No risk of EF creating wrong structure
Faster when DB already exists

✔ Cons:

Models are tightly coupled to database schema
Changing the DB requires re-scaffolding
Migrations become harder or disabled
Not ideal for domain-driven design

-------------------------------------------------------------

## How migrations work in Code First

Migrations in EF Core are a way to:

Track changes to your model (classes)
Generate SQL files automatically
Apply database schema updates safely

How it works:
You write POCO classes
Configure them using attributes or Fluent API
Run migrations → EF generates tables, relations, constraints

Add-Migration AddEmailToStudent
Update-Database

EF reads all migration files in order
Finds which migrations have not yet run
Executes the SQL in the Up() methods
Updates a table in the database:

-------------------------------------------------------------

## What are Indexes?

It helps the database find rows faster without scanning the entire table.

Why use indexes?
Speed up SELECT queries
Improve performance of search, sorting, JOINs, WHERE, ORDER BY
Reduce I/O operations

Downside:
Slows down INSERT / UPDATE / DELETE
Takes additional memory space

-------------------------------------------------------------

## Clustered vs Non-Clustered Indexes

| Feature                        | Clustered Index             | Non-Clustered Index                 |
| ------------------------------ | --------------------------- | ----------------------------------- |
| Defines physical order of data | ✔ Yes                       | ❌ No                                |
| Number allowed per table       | **1**                       | **Many**                            |
| Storage                        | Actual table                | Separate from table                 |
| Speed                          | Faster                      | Slightly slower (extra lookup)      |
| Use case                       | Primary keys, unique values | Searching frequently used columns   |
| Pointer                        | Not needed                  | Points to clustered index key / row |
| Best for                       | Range queries               | High-selectivity columns            |


-------------------------------------------------------------

## Stored Procedures vs Views

| Feature                       | Stored Procedure                | View                                  |
| ----------------------------- | ------------------------------- | ------------------------------------- |
| Type                          | Program (SQL code block)        | Virtual table                         |
| Contains                      | Multiple SQL statements + logic | Single SELECT query                   |
| Parameters                    | ✔ Yes                           | ❌ No                                  |
| Return type                   | Result sets, output params      | Result set only                       |
| Supports INSERT/UPDATE/DELETE | ✔ Yes                           | ❌ Usually No (except updatable views) |
| Logic (IF, loops)             | ✔ Yes                           | ❌ No                                  |
| Executes                      | Using CALL/EXEC                 | Using SELECT                          |
| Used for                      | Business logic                  | Data representation                   |
| Stores data?                  | ❌ No                            | ❌ No (only definition)                |


-------------------------------------------------------------


## Docker vs Container:
			
| Feature    | Docker                                       | Container                                  |
| ---------- | -------------------------------------------- | ------------------------------------------ |
| Definition | A platform/tool to build & manage containers | A lightweight runtime environment for apps |
| Type       | Technology / Software                        | The actual runtime instance                |
| Purpose    | Create, run, and orchestrate containers      | Run the application in isolation           |
| Includes   | Docker Engine, Docker CLI, Docker Hub        | App + dependencies                         |
| Build?     | Yes (Dockerfile → Image → Container)         | No (container is the result)               |
| Run?       | Runs containers                              | The thing being run                        |


-------------------------------------------------------------
## Difference between Filter and Middleware

| Feature                                      | Middleware                                | Filter                                                            |
| -------------------------------------------- | ----------------------------------------- | ----------------------------------------------------------------- |
| **Where it works**                           | Entire request pipeline (before routing)  | Only inside MVC pipeline (Controllers/Actions)                    |
| **Purpose**                                  | Cross-cutting system-level concerns       | Action-level concerns in MVC                                      |
| **Executes**                                 | From Program.cs pipeline                  | Around controller/action execution                                |
| **Knows about Controller/Action?**           | ❌ No                                      | ✔ Yes                                                             |
| **Access to Model Binding, Action Context?** | ❌ No                                      | ✔ Yes                                                             |
| **Order**                                    | Order added in Program.cs                 | Executes based on filter type & order                             |
| **Types**                                    | Only one: middleware                      | Many types: Auth, Resource, Action, Exception, Result             |
| **Used for**                                 | Logging, routing, auth, compression, CORS | Validation, caching, authorization per action, exception handling |
| **Runs for static files?**                   | ✔ Yes                                     | ❌ No                                                              |


-------------------------------------------------------------

## What is an Extension Method in C#

public static class StringExtensions
{
    public static int WordCount(this string str)
    {
        return str.Split(' ', StringSplitOptions.RemoveEmptyEntries).Length;
    }
}

string message = "Hello World from C#";
int count = message.WordCount();  // Looks like a built-in method

Has this keyword before the first parameter
→ This tells C# which type you are extending.

Even though you never modified the string class, it behaves like the method exists on it.

-------------------------------------------------------------

## Event call in Microservices

In microservices, an event call means one service publishes an event, 
and one or more services react to it asynchronously.

➡️ It is asynchronous communication
➡️ Services do not call each other directly
➡️ They communicate through events via a message broker 
(Kafka, RabbitMQ, Azure Service Bus, etc.)

Example: User places an order

1. Order Service publishes:
OrderCreatedEvent { OrderId = 1, UserId = 10, Amount = 200 }

2. Subscribers react:
Payment Service listens → charges card
Inventory Service listens → reduces stock
Notification Service listens → sends email

Order Service doesn't know who listens.
Subscribers don’t know who published.

This is decoupled asynchronous communication.

eg.

1. User places order
Order Service publishes event:
OrderCreatedEvent { OrderId = 101 }

2. RabbitMQ Exchange receives it
Exchange routes message to queues:
 → billing.queue
 → email.queue
 → inventory.queue

3. Billing Service consumes event
Processes payment

4. Email Service consumes event
Sends email: “Order Confirmed”

5. Inventory Service consumes event
Reduces stock

All services remain completely independent.

-------------------------------------------------------------


## const vs readonly vs static?

| Feature               | const                     | readonly                     | static               |
| --------------------- | ------------------------- | ---------------------------- | -------------------- |
| Value assigned at     | Compile time              | Runtime                      | Runtime (any time)   |
| Value can change?     | ❌ No                      | ❌ No (after ctor)            | ✅ Yes                |
| Per instance?         | ❌ No                      | ✅ Yes                        | ❌ No                 |
| Implicitly static     | ✅ Yes                     | ❌ No                         | —                    |
| Requires constructor? | ❌ No                      | Optional                     | ❌ No                 |
| Best for              | Never-changing primitives | Values known only at runtime | Shared mutable state |

-------------------------------------------------------------

## CLR & CTS?

| Feature    | CLR                           | CTS                        |
| ---------- | ----------------------------- | -------------------------- |
| Purpose    | Executes code                 | Defines common types       |
| Role       | Runtime engine                | Type specification         |
| Handles    | GC, JIT, exceptions, security | Data types & rules         |
| Works with | IL Code                       | All .NET languages         |
| Ensures    | Application execution         | Cross-language type safety |

-------------------------------------------------------------

## IIS vs Kestrel?

| Feature               | IIS                                           | Kestrel                                   |
| --------------------- | --------------------------------------------- | ----------------------------------------- |
| Type                  | Full-featured web server                      | Lightweight web server                    |
| Platform              | Windows only                                  | Cross-platform                            |
| Performance           | Good                                          | Excellent, very fast                      |
| Reverse proxy needed? | No                                            | Not always, but recommended in production |
| Capabilities          | Rich (URL rewrite, authentication, filtering) | Minimal                                   |
| Use in ASP.NET Core   | Legacy + reverse proxy                        | Primary runtime server                    |
| Suitable for          | Windows on-prem                               | Cloud, Docker, cross-platform             |

-------------------------------------------------------------
## Use() vs Add() vs Run() vs Map()?

Use()

Adds middleware that can run before and after the next component.

● Run()

Adds terminal middleware that ends the pipeline and doesn’t call next.

● Map()

Branches the pipeline based on route/path.

● AddXXX()

Registers services to DI and configures the application — not part of the HTTP pipeline.

-------------------------------------------------------------

## ViewBag vs ViewData vs TempData?

| Feature                | ViewData           | ViewBag           | TempData                |
| ---------------------- | -----------------  | ----------------- | ----------------------- |
| Request lifespan       | 1 request          | 1 request         | Multiple (via redirect) |
| Type                   | Dictionary         | Dynamic           | Dictionary              |
| Use case               | Pass data to View  | Pass data to View | Redirect, notifications |
| Works across Redirect? | ❌ No              | ❌ No              | ✔ Yes                  |
| Uses Session/Cookies?  | ❌ No              | ❌ No              | ✔ Yes                  |


-------------------------------------------------------------

## How Routing works in Asp.Net MVC vs Asp.Net Core MVC vs Asp.Net Core Web Api?

| Feature            | ASP.NET MVC (Framework)      | ASP.NET Core MVC         | ASP.NET Core Web API |
| ------------------ | ---------------------------- | ------------------------ | -------------------- |
| Routing Engine     | System.Web.Routing           | Endpoint Routing         | Endpoint Routing     |
| Pipeline           | System.Web (IIS)             | Middleware Pipeline      | Middleware Pipeline  |
| Default Routing    | Conventional                 | Conventional + Attribute | Attribute-first      |
| REST Support       | Limited                      | Good                     | Native               |
| Verb-based Routing | No                           | Yes                      | Yes (Primary)        |
| Cross-platform     | ❌ No                         | ✔ Yes                    | ✔ Yes                |
| Performance        | Slow                         | Faster                   | Fastest (API focus)  |
| Typical Route      | `{controller}/{action}/{id}` | Same or attribute        | `/api/products/{id}` |
| Attribute Routing  | Optional (added in MVC5)     | Fully enabled            | Primary method       |

-------------------------------------------------------------
## What is CORS?

CORS allows a server to specify which external origins can access its resources, 
preventing unauthorized cross-domain attacks. 
ASP.NET Core implements CORS using middleware and policies.

Origin = protocol + domain + port

builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowClient", policy =>
        policy.WithOrigins("https://myapp.com")
              .AllowAnyHeader()
              .AllowAnyMethod());
});

app.UseCors("AllowClient");

When Should You Enable CORS?

✔ SPA frameworks (React, Angular, Vue)
✔ Different frontend + backend domains
✔ Microservices
✔ API Gateway to downstream services

-------------------------------------------------------------
## What is a Reverse Proxy?

A Reverse Proxy is a server that sits between the client and backend services:

✔ You want to hide microservices behind a gateway
✔ Forwards requests to downstream APIs
✔ Hides backend URLs
✔ Handles routing, load balancing, caching, SSL termination, etc.

Nginx
Apache
IIS
YARP (.NET reverse proxy)

-------------------------------------------------------------

## SOAP vs REST?

| Feature      | SOAP                       | REST                          |
| ------------ | -------------------------- | ----------------------------- |
| Type         | Protocol                   | Architectural style           |
| Data format  | XML only                   | JSON, XML, HTML, text         |
| Ease of use  | Complex                    | Simple                        |
| Speed        | Slower (heavy XML)         | Fast (lightweight)            |
| Transport    | Any (HTTP, SMTP, TCP)      | HTTP only                     |
| Security     | Very strong (WS-Security)  | Relies on HTTP (HTTPS, OAuth) |
| Caching      | ❌ No                       | ✔ Yes                         |
| Statefulness | Can be stateful            | Always stateless              |
| Best for     | Enterprise banking/finance | Modern web/mobile apps        |

-------------------------------------------------------------

## Global exception handling?

Global exception handling in ASP.NET Core allows you to catch all unhandled exceptions in one central place, 
instead of writing try-catch blocks everywhere in controllers or services.


Why Do We Need Global Exception Handling?

Without global handling, every unhandled exception:
Crashes the request
Produces random HTTP 500 pages
Returns inconsistent responses
Pollutes logs

With global handling:
One middleware handles ALL exceptions
Standard error format
Consistent HTTP status codes

| Approach                        | When to Use                                   |
| ------------------------------- | --------------------------------------------- |
| **UseExceptionHandler()**       | Most production-ready apps; simple + reliable |
| **Custom Exception Middleware** | Pure APIs, microservices, need custom JSON    |
| **Filters**                     | Only inside MVC, not global pipeline          |


Using UseExceptionHandler() (Built-in Middleware)

Best for APIs and MVC apps.

Program.cs:

app.UseExceptionHandler("/error");


ErrorController.cs

[ApiController]
public class ErrorController : ControllerBase
{
    [Route("error")]
    public IActionResult HandleError() =>
        Problem();  // Returns RFC 7807 standard error response
}


✔ Clean
✔ Built-in
✔ Recommended for production

🔷 2. Custom Exception Handling Middleware

Used when you want full control.

Middleware:

public class GlobalExceptionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger _logger;

    public GlobalExceptionMiddleware(RequestDelegate next, ILogger<GlobalExceptionMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task Invoke(HttpContext context)
    {
        try
        {
            await _next(context); // call next middleware
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception");
            context.Response.StatusCode = 500;
            context.Response.ContentType = "application/json";

            var response = new { message = "Something went wrong." };
            await context.Response.WriteAsJsonAsync(response);
        }
    }
}


Register it:

app.UseMiddleware<GlobalExceptionMiddleware>();



Client
 ↓
Middleware Pipeline
 ↓
Custom Exception Middleware (catches all)
 ↓
Routing
 ↓
MVC (Action Filters → Exception Filters)
 ↓
Controller Action
 ↓
Exception Occurs
 ↑
Handled Globally


-------------------------------------------------------------

##  Unit of Work (UoW) pattern:

Manages a group of repository transactions as a single atomic operation.

In EF Core, the DbContext itself is a unit of work (because SaveChanges is transactional).

But UoW pattern:
Groups multiple repositories
Provides a single Save() method
Ensures all DB operations succeed together or rollback together

public interface IUnitOfWork : IDisposable
{
    IGenericRepository<Product> Products { get; }
    IGenericRepository<Category> Categories { get; }
    Task<int> SaveAsync();
}

public class UnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _context;

    public IGenericRepository<Product> Products { get; }
    public IGenericRepository<Category> Categories { get; }

    public UnitOfWork(AppDbContext context)
    {
        _context = context;
        Products = new GenericRepository<Product>(_context);
        Categories = new GenericRepository<Category>(_context);
    }

    public async Task<int> SaveAsync()
    {
        return await _context.SaveChangesAsync();
    }

    public void Dispose()
    {
        _context.Dispose();
    }
}

public class ProductService
{
    private readonly IUnitOfWork _unitOfWork;

    public ProductService(IUnitOfWork unitOfWork)
    {
        _unitOfWork = unitOfWork;
    }

    public async Task AddProduct(Product p)
    {
        await _unitOfWork.Products.AddAsync(p);
        await _unitOfWork.SaveAsync();
    }
}

-------------------------------------------------------------

## Lazy vs Eager vs Explicit loading?

| Feature         | Eager Loading               | Lazy Loading                   | Explicit Loading         |
| --------------- | --------------------------- | ------------------------------ | ------------------------ |
| When data loads | Immediately with main query | When accessed                  | When manually triggered  |
| Queries         | 1 big query                 | Many small queries             | Controlled extra queries |
| Performance     | Good for predictable data   | Can be bad (N+1 problem)       | Good control / optimized |
| Implementation  | `Include()`                 | Virtual navigation + proxies   | `.Load()`                |
| Use case        | Need related data upfront   | Don’t always need related data | Precise control required |


-------------------------------------------------------------

## Task vs Thread vs async/await?

| Feature    | Thread                  | Task                   | async/await                      |
| ---------- | ----------------------- | ---------------------- | -------------------------------- |
| Level      | Low-level               | High-level             | Language-level                   |
| Type       | Physical OS thread      | Logical unit of work   | Syntax for async operations      |
| Created by | Manually by developer   | CLR (.NET thread pool) | Compiler-managed                 |
| Best for   | Long-running operations | CPU-bound work         | I/O-bound async                  |
| Cost       | Expensive               | Cheaper                | Cheapest (no thread blocked)     |
| Blocking   | Yes                     | Depends                | Never blocks thread during await |

-------------------------------------------------------------

## App performance tuning in .NET?

### 1) General Performance Tuning Principles

* Use AsNoTracking for read queries
* Avoid Lazy Loading → N+1 problem
* Use caching aggressively
* Reduce middleware pipeline
* Prefer async/await everywhere
* Optimize EF queries (Include, projections, filtering)
* Use Response Compression and Caching
* Reduce garbage collection pressure (Span/Pool/ValueTask)
* Profile your application regularly
* Use Redis for distributed cache & performance


### 2) Code-Level Optimizations

✔ Avoid unnecessary object allocations<br/>
// Bad: allocates every loop iteration<br/>
var list = new List<int>();<br/>

// Good: specify capacity<br/>
var list = new List<int>(1000);<br/>

✔ Use Span<T> / Memory<T> for high-performance memory operations (C# 7.2+).<br/>
✔ Avoid boxing/unboxing<br/>
* Use generics instead of object.<br/>
✔ Use StringBuilder for string concatenation in loops.<br/>
✔ Prefer asynchronous APIs<br/>
* Async releases threads → more scalable APIs.

### 3) ASP.NET Core Performance

✔ Use Response Caching<br/>
* app.UseResponseCaching();<br/>
[ResponseCache(Duration = 60)]<br/>
public IActionResult GetData() => Ok();<br/>

✔ Enable Compression (Brotli/Gzip)<br/>
* services.AddResponseCompression();<br/>

✔ Minimize Middleware<br/>

* Each middleware adds overhead. Remove unused ones.<br/>

✔ Use IHttpContextAccessor sparingly.<br/>

It’s expensive; inject HttpContext only where needed.<br/>

### 4) Database Performance (EF Core)

✔ Use AsNoTracking for read-only queries<br/>
* var data = context.Users.AsNoTracking().ToList();<br/>

✔ Avoid Lazy Loading → Causes N+1 issues<br/>
* Prefer Eager or Explicit loading.<br/>

✔ Use Compiled Queries for frequently-used queries<br/>

✔ Optimize LINQ Queries<br/>
* Don’t evaluate on client side.<br/>

✔ Index the database properly (Clustered, Non-Clustered)<br/>

✔ Batch SaveChanges<br/>
* await context.SaveChangesAsync();<br/>
* Instead of saving inside loops.

### 5) Multithreading & Async Improvements

✔ Avoid blocking calls<br/>

❌ Task.Result<br/>
❌ Task.Wait()<br/>

Always use:<br/>
✔ await<br/>

✔ Use ValueTask for performance-critical async calls<br/>

Reduces allocations for short-lived operations.<br/>

### 6) Caching Strategies

🔹 In-Memory Caching<br/>
For small, frequently accessed data.<br/>

🔹 Distributed Cache (Redis)<br/>
For multi-server deployments / microservices.<br/>

🔹 Output Caching (.NET 8 feature)<br/>

🔹 Don’t cache large objects unnecessarily<br/>

Avoid LOH (Large Object Heap) pressure.<br/>

### 7) Memory Optimizations

✔ Remove unused DI services<br/>
Every service consumes memory.<br/>

✔ Clean up IDisposable<br/>
Use:<br/>
using var stream = ...<br/>

✔ Avoid large in-memory collections if not needed.<br/>

✔ Use ArrayPool<T> / ObjectPool<T><br/>
Reduces GC pressure.<br/>

### 8) Logging & Diagnostics

✔ Reduce logging level in Production<br/>

Use only:<br/>
* Information
* Error
* Critical

✔ Enable Application Insights / OpenTelemetry for tracing.<br/>
✔ Use dotnet-trace, dotnet-dump, and dotnet-counters for performance profiling.<br/>

### 9)  API Response Optimization
✔ Return IAsyncEnumerable<T> for streaming large lists<br/>
✔ Use Minimal APIs (lightweight compared to MVC)<br/>
✔ Compress JSON (System.Text.Json faster than Newtonsoft)<br/>
services.AddControllers()
    .AddJsonOptions(o => o.JsonSerializerOptions.PropertyNamingPolicy = null);

### 10) Deployment Optimizations
✔ Enable ReadyToRun (AOT compilation)<br/>

Improves startup time.<br/>

✔ Publish using --self-contained for runtime optimization.<br/>
✔ Use Kestrel + Reverse Proxy (Nginx/IIS)<br/>
✔ Enable HTTP/2 or HTTP/3 for better network performance.<br/>

-------------------------------------------------------------

## Caching strategies?

What Is Caching?

Caching is the technique of storing frequently accessed data in fast storage (memory, Redis, file) so future requests can access it quickly without recomputing or hitting the database.

🧩 Types of Caching Strategies
1️⃣ In-Memory Cache (IMemoryCache)

Stored in application memory, fastest cache.

✔ Best for:

Single server deployments

Small data (< few MBs)

Frequently accessed, rarely changed data

Example:
services.AddMemoryCache();

public class ProductService
{
    private readonly IMemoryCache _cache;

    public ProductService(IMemoryCache cache)
    {
        _cache = cache;
    }

    public Product GetProduct(int id)
    {
        return _cache.GetOrCreate($"product_{id}", entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);
            return db.Products.Find(id);
        });
    }
}

2️⃣ Distributed Cache (Redis / SQL Server Cache)

Stored outside the application, in Redis or SQL.

✔ Best for:

Load-balanced / multi-server applications

Microservices

Large caching needs

Example:
services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379";
});

3️⃣ Response Caching

Caches entire HTTP responses.

✔ Best for:

GET endpoints returning static or semi-static data

Public APIs with largely identical responses

app.UseResponseCaching();

[ResponseCache(Duration = 60)]
public IActionResult Get() => Ok();

4️⃣ Output Caching (New in .NET 8)

More powerful than response caching:

Can cache POST responses

Supports tags, vary-by, invalidation

builder.Services.AddOutputCache();

app.MapGet("/products", () => data).CacheOutput();

5️⃣ Client-Side Caching

Use browser cache headers.

Examples:

Cache-Control

ETag

Last-Modified

✔ Best for:

Static files

Images, CSS, JS

app.UseStaticFiles(new StaticFileOptions
{
    OnPrepareResponse = ctx =>
    {
        ctx.Context.Response.Headers["Cache-Control"] = "public,max-age=86400";
    }
});

6️⃣ Distributed Memory Caching (Hybrid)

Combination of in-memory + Redis, where local memory is used for fast reads, Redis for consistency.

7️⃣ Query Caching (EF Core)

Cache repeated database calls manually like:

var users = await _cache.GetOrCreateAsync("users_all", async e =>
{
    e.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);
    return await context.Users.AsNoTracking().ToListAsync();
});

🧠 Caching Patterns (Strategies)
🔥 A. Absolute Expiration

Cache expires after fixed duration.

entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);


✔ Predictable
✘ Might serve stale data

🔥 B. Sliding Expiration

Resets expiration time each time the cache is accessed.

entry.SlidingExpiration = TimeSpan.FromMinutes(5);


✔ Great for frequently accessed but rarely updated data
✘ Cache may stay forever if constantly used

🔥 C. Cache Aside (Lazy Loading)

Most common pattern.

Check cache

If not found → load from DB

Put data into cache

Return

✔ Simple
✔ Efficient
✔ Works for most apps

🔥 D. Write-Through Cache

When data is written to DB, it's also written to cache immediately.

✔ Cache always consistent
✘ Slight write performance hit

🔥 E. Write-Behind Cache

Write to cache → DB updated later in background.

✔ Super fast writes
✘ Risk of data inconsistency

🔥 F. Read-Through Cache

Application always reads through the cache provider, which loads from DB automatically.

✔ Encapsulated, clean
✘ Provider complexity

🔥 G. Cache Invalidation Strategies

The hardest part of caching.

Techniques:

Time-based expiration

Manual invalidation via key

Tag-based invalidation (Output Cache / Redis)

Event-based invalidation (e.g., on DB change)

-------------------------------------------------------------

## What is ACID?

ACID stands for Atomicity, Consistency, Isolation, Durability.
It ensures that database transactions execute reliably.
Atomicity ensures all-or-nothing, 
consistency ensures rules are followed, 
isolation ensures concurrency safety, 
and durability ensures data is permanent after commit.

-------------------------------------------------------------

## Normalization vs denormalization?

normalization reduces redundancy by splitting data into multiple related tables, improving data integrity but requiring more joins.

Denormalization combines tables to reduce joins and speed up read operations but introduces redundancy and consistency challenges.

Normalization is the process of organizing database tables to reduce data redundancy and improve data integrity.

🎯 Goal:

Avoid duplicate data

Ensure consistent data

Create efficient updates/inserts

📌 Techniques include:

1NF — Remove repeating groups

2NF — Remove partial dependencies

3NF — Remove transitive dependencies

BCNF, 4NF, 5NF (advanced)

✔ Example of Normalization
❌ Denormalized table (repeating data)
OrderId	CustomerName	CustomerCity	Product	Price
1	John	NYC	Laptop	1000
2	John	NYC	Mouse	20

Customer info is repeated.

✔ Normalized tables

Customers table:

CustomerId	Name	City
1	John	NYC

Orders table:

OrderId	CustomerId	Product	Price
1	1	Laptop	1000
2	1	Mouse	20

No repetition → better integrity.

🎯 When to Use Normalization

OLTP systems (transactional apps)

Banking, finance, inventory systems

Write-heavy systems

When consistency and data integrity matter most

👍 Benefits

Eliminates duplicates

Saves storage

Easier updates (data in one place)

Strong integrity & consistency

👎 Drawbacks

More joins → slower reads

More tables → more complex queries

🚀 Denormalization

Denormalization is the process of combining tables to improve read performance by reducing joins — at the cost of more redundant data.

🎯 Goal:

Faster reads

Pre-joined, pre-computed data

Better performance for reporting

✔ Example of Denormalization

Instead of splitting:

Customers table
Orders table

We combine into one table:

OrderId	CustomerName	City	Product	Price
1	John	NYC	Laptop	1000
2	John	NYC	Mouse	20

No joins → faster read.

🎯 When to Use Denormalization

OLAP / analytics / reporting

Read-heavy workloads

Data warehousing

Microservices where joins across services aren't possible

Caching and materialized views

👍 Benefits

Faster read performance (fewer joins)

Simplifies query logic

Good for reporting and dashboards

👎 Drawbacks

Data redundancy

Update anomalies

Harder to maintain consistency

More storage required

🆚 Normalization vs Denormalization (Comparison Table)
Feature	Normalization	Denormalization
Goal	Reduce redundancy	Improve read performance
Data Integrity	High	Lower
Storage	Low	High
Read Performance	Slower (more joins)	Faster
Write Performance	Faster	Slower
Use Cases	OLTP	OLAP / Reporting
Complexity	More tables	Fewer tables