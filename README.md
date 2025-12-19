# DotNetInterviewQuestions

## C# → IL → Machine Code

 ```csharp
C# Code (.cs)
     │
     ▼
Roslyn Compiler (csc.exe)
     │
     ▼
IL + Metadata
(.dll / .exe)
     │
     ▼
CLR (Common Language Runtime)
     │
     ▼
JIT Compilation
     │
     ▼
Native Machine Code
(x64 / ARM / Windows / Linux)
 ```
 -------------------------------------------------------------
 ## Program.cs

```csharp
var builder = WebApplication.CreateBuilder(args);
 ```

✔ Creates the application builder<br/>

This is the entry point for configuring:<br/>

* Services (Dependency Injection)
* Logging
* Configuration (appsettings.json, environment variables)

✔ Internally it sets up:<br/>

* DI Container → builder.Services
* Configuration → builder.Configuration
* Logging → builder.Logging
* Hosting environment → builder.Environment

In simple words:<br/>
This line prepares everything needed to build the ASP.NET Core application.<br/>

```csharp
var app = builder.Build();
```

✔ Builds the WebApplication object<br/>

After configuring services, this line builds the HTTP pipeline.<br/>

Internally:<br/>

* Creates middleware pipeline
* Wires services into the DI container
* Finalizes hosting model

In simple words:<br/>
Creates the actual web application that will handle incoming HTTP requests.<br/>

```csharp
app.UseAuthorization();
```

✔ Adds Authorization Middleware to the pipeline<br/>

This middleware checks:

* If the user is authenticated
* If the user is allowed (based on [Authorize] attributes, roles, policies)

Important:<br/>

Authorization ≠ Authentication<br/>
Authentication: Who are you? (Identity)<br/>
Authorization: What can you do? (Permissions)<br/>

In simple words:

Enforces access rules before controllers run.<br/>

```csharp
app.MapControllers();
```

✔ Maps controller endpoints to the routing system<br/>

Enables attribute routing:<br/>

```csharp
[HttpGet("api/products")]
public IActionResult GetProducts() { ... }
```

It tells ASP.NET Core:<br/>

* Activate all controllers with [ApiController] and route their endpoints.
* Without this line → no controller endpoints work.

```csharp
app.Run();
```

✔ Starts the web application and begins listening for HTTP requests<br/>

This is the terminal middleware in the pipeline.<br/>

The app will:<br/>

* Start Kestrel web server
* Begin processing incoming requests
* Block the thread until app shuts down

In simple words:<br/>
This line starts your API server.<br/>

 -------------------------------------------------------------
## How to secure Asp.Net core application: (****Add more***)

### 1)  Man-in-the-middle attacks (MITM) and downgrade attacks -> UseHsts();

 Enable HTTP Strict Transport Security (HSTS).<br/>
 All future attempts to access HTTP are automatically converted to HTTPS inside the browser, without hitting your server.
 
### 2) Cross-Site Request Forgery (CSRF) 

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

### 3)  SQL Injection

* Attacker injects malicious SQL into input fields to manipulate the database e.g. ' OR 1=1 --
* Impact: Data leakage, Authentication bypass, Data deletion
* Prevention: Parameterized queries, ORM (EF Core), WAF SQL injection rules, Input validation, Never use string concatenation for SQL

```csharp
context.Users
    .Where(u => u.Email == email); // Safe (parameterized)
```

*Stored procedure is dicussed below*

### 4)  XSS (Cross-Site Scripting)

* Injecting malicious JavaScript into web pages
* Types : Stored XSS, Reflected XSS, DOM-based XSS
* Impact : Session hijacking, Cookie theft, Account takeover
* Prevention : Output encoding, Content Security Policy (CSP), WAF XSS filters

e.g. Razor automatically HTML-encodes output


### 5) Path Traversal

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

-------------------------------------------------------------

## Difference Between ROW_NUMBER, RANK, and DENSE_RANK

| Feature            | ROW_NUMBER | RANK        | DENSE_RANK  |
| ------------------ | ---------- | ----------- | ----------- |
| Duplicate handling | ❌ Ignored  | ✅ Same rank | ✅ Same rank |
| Rank gaps          | ❌ No       | ✅ Yes       | ❌ No        |
| Unique numbers     | ✅ Always   | ❌ No        | ❌ No        |
| Best for           | Pagination | Competition | Nth highest |

-------------------------------------------------------------

 ## Parameterized Stored Procedure

* Step 1: SQL Server: Parameterized Stored Procedure*
```sql
CREATE PROCEDURE usp_GetUsersByStatus
(
    @IsActive BIT,
    @MinAge INT
)
AS
BEGIN
    SET NOCOUNT ON;

    SELECT Id, Name, Email, Age, IsActive
    FROM Users
    WHERE IsActive = @IsActive
      AND Age >= @MinAge;
END
```

* Step 2: ASP.NET Core Model (DTO):*

```csharp
public class UserDto
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
    public int Age { get; set; }
    public bool IsActive { get; set; }
}
```

* Step 3: Repository Method (ADO.NET)*

```csharp
using System.Data;
using System.Data.SqlClient;

public class UserRepository
{
    private readonly IConfiguration _configuration;

    public UserRepository(IConfiguration configuration)
    {
        _configuration = configuration;
    }

    public async Task<List<UserDto>> GetUsersAsync(bool isActive, int minAge)
    {
        var users = new List<UserDto>();

        using var connection = new SqlConnection(
            _configuration.GetConnectionString("DefaultConnection"));

        using var command = new SqlCommand("usp_GetUsersByStatus", connection);
        command.CommandType = CommandType.StoredProcedure;

        // Parameters
        command.Parameters.AddWithValue("@IsActive", isActive);
        command.Parameters.AddWithValue("@MinAge", minAge);

        await connection.OpenAsync();

        using var reader = await command.ExecuteReaderAsync();

        while (await reader.ReadAsync())
        {
            users.Add(new UserDto
            {
                Id = reader.GetInt32(0),
                Name = reader.GetString(1),
                Email = reader.GetString(2),
                Age = reader.GetInt32(3),
                IsActive = reader.GetBoolean(4)
            });
        }

        return users;
    }
}
```

*Step 4: API Controller*

```csharp
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    private readonly UserRepository _repository;

    public UsersController(UserRepository repository)
    {
        _repository = repository;
    }

    [HttpGet]
    public async Task<IActionResult> GetUsers(
        bool isActive,
        int minAge)
    {
        var result = await _repository.GetUsersAsync(isActive, minAge);
        return Ok(result);
    }
}
```

*Step 5: ADependency Injection (Program.cs)*

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

builder.Services.AddScoped<UserRepository>();
```

#### API Call Example:
GET /api/users?isActive=true&minAge=18

-------------------------------------------------------------

## Stored Procedure with EF Core:

### 1. SQL Server Stored Procedures

#### SELECT Stored Procedure

 ```sql
 CREATE PROCEDURE usp_GetUsersByStatus
(
    @IsActive BIT,
    @MinAge INT
)
AS
BEGIN
    SET NOCOUNT ON;

    SELECT Id, Name, Email, Age, IsActive
    FROM Users
    WHERE IsActive = @IsActive
      AND Age >= @MinAge;
END
```

#### INSERT Stored Procedure (with OUTPUT)

 ```sql
 CREATE PROCEDURE usp_CreateUser
(
    @Name NVARCHAR(100),
    @Email NVARCHAR(100),
    @Age INT,
    @IsActive BIT,
    @UserId INT OUTPUT
)
AS
BEGIN
    SET NOCOUNT ON;

    INSERT INTO Users (Name, Email, Age, IsActive)
    VALUES (@Name, @Email, @Age, @IsActive);

    SET @UserId = SCOPE_IDENTITY();
END
 ```

### 2. EF Core Entity

```csharp
public class User
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
    public int Age { get; set; }
    public bool IsActive { get; set; }
}
```

### 3. DbContext Configuration

```csharp
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options) { }

    public DbSet<User> Users { get; set; }
}
```


### 4. SELECT Stored Procedure using EF Core

**FromSqlRaw (Read-Only)**

* FromSqlRaw() works only for SELECT
* AsNoTracking() improves performance

```csharp
public async Task<List<User>> GetUsersAsync(bool isActive, int minAge)
{
    return await _context.Users
        .FromSqlRaw(
            "EXEC usp_GetUsersByStatus @IsActive, @MinAge",
            new SqlParameter("@IsActive", isActive),
            new SqlParameter("@MinAge", minAge))
        .AsNoTracking()
        .ToListAsync();
}
```

### 5. INSERT Stored Procedure using EF Core:

* EF Core does NOT support inserts via FromSql, so use ExecuteSqlRawAsync.
* Missing parameters → Runtime error

```csharp
public async Task<int> CreateUserAsync(User user)
{
    var outputParam = new SqlParameter
    {
        ParameterName = "@UserId",
        SqlDbType = SqlDbType.Int,
        Direction = ParameterDirection.Output
    };

    await _context.Database.ExecuteSqlRawAsync(
        "EXEC usp_CreateUser @Name, @Email, @Age, @IsActive, @UserId OUT",
        new SqlParameter("@Name", user.Name),
        new SqlParameter("@Email", user.Email),
        new SqlParameter("@Age", user.Age),
        new SqlParameter("@IsActive", user.IsActive),
        outputParam
    );

    return (int)outputParam.Value;
}
```

### 6. Repository Example:

```csharp
public class UserRepository
{
    private readonly AppDbContext _context;

    public UserRepository(AppDbContext context)
    {
        _context = context;
    }

    public Task<List<User>> GetUsersAsync(bool isActive, int minAge)
        => _context.Users
            .FromSqlRaw(
                "EXEC usp_GetUsersByStatus @IsActive, @MinAge",
                new SqlParameter("@IsActive", isActive),
                new SqlParameter("@MinAge", minAge))
            .AsNoTracking()
            .ToListAsync();

    public Task<int> CreateUserAsync(User user)
        => CreateUserInternalAsync(user);

    private async Task<int> CreateUserInternalAsync(User user)
    {
        var idParam = new SqlParameter("@UserId", SqlDbType.Int)
        {
            Direction = ParameterDirection.Output
        };

        await _context.Database.ExecuteSqlRawAsync(
            "EXEC usp_CreateUser @Name, @Email, @Age, @IsActive, @UserId OUT",
            new SqlParameter("@Name", user.Name),
            new SqlParameter("@Email", user.Email),
            new SqlParameter("@Age", user.Age),
            new SqlParameter("@IsActive", user.IsActive),
            idParam
        );

        return (int)idParam.Value;
    }
}
```

### 7. API Controller:

```csharp
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    private readonly UserRepository _repository;

    public UsersController(UserRepository repository)
    {
        _repository = repository;
    }

    [HttpGet]
    public async Task<IActionResult> Get(bool isActive, int minAge)
    {
        var users = await _repository.GetUsersAsync(isActive, minAge);
        return Ok(users);
    }

    [HttpPost]
    public async Task<IActionResult> Create(User user)
    {
        var userId = await _repository.CreateUserAsync(user);
        return Ok(new { UserId = userId });
    }
}
```

### 8. Program.cs (EF Core Setup)

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

builder.Services.AddScoped<UserRepository>();
```

-------------------------------------------------------------

## SQL Server Stored Procedure with OUTPUT Parameters:

### Example: Insert User + Return ID and Status

```sql
CREATE PROCEDURE usp_CreateUser
(
    @Name NVARCHAR(100),
    @Email NVARCHAR(100),
    @Age INT,
    @UserId INT OUTPUT,
    @StatusMessage NVARCHAR(100) OUTPUT
)
AS
BEGIN
    SET NOCOUNT ON;

    IF EXISTS (SELECT 1 FROM Users WHERE Email = @Email)
    BEGIN
        SET @UserId = 0;
        SET @StatusMessage = 'Email already exists';
        RETURN;
    END

    INSERT INTO Users (Name, Email, Age)
    VALUES (@Name, @Email, @Age);

    SET @UserId = SCOPE_IDENTITY();
    SET @StatusMessage = 'User created successfully';
END
```

### EF Core: Calling Stored Procedure with OUTPUT Parameters:

* EF Core uses ExecuteSqlRawAsync for non-query SPs.

* Repository Method:

```csharp
public async Task<(int userId, string message)> CreateUserAsync(User user)
{
    var userIdParam = new SqlParameter
    {
        ParameterName = "@UserId",
        SqlDbType = SqlDbType.Int,
        Direction = ParameterDirection.Output
    };

    var messageParam = new SqlParameter
    {
        ParameterName = "@StatusMessage",
        SqlDbType = SqlDbType.NVarChar,
        Size = 100,
        Direction = ParameterDirection.Output
    };

    await _context.Database.ExecuteSqlRawAsync(
        "EXEC usp_CreateUser @Name, @Email, @Age, @UserId OUT, @StatusMessage OUT",
        new SqlParameter("@Name", user.Name),
        new SqlParameter("@Email", user.Email),
        new SqlParameter("@Age", user.Age),
        userIdParam,
        messageParam
    );

    return (
        (int)userIdParam.Value,
        messageParam.Value?.ToString()
    );
}
```

### API Controller Usage

```csharp
[HttpPost]
public async Task<IActionResult> Create(User user)
{
    var result = await _repository.CreateUserAsync(user);

    if (result.userId == 0)
        return BadRequest(result.message);

    return Ok(new
    {
        UserId = result.userId,
        Message = result.message
    });
}
```
-------------------------------------------------------------

## Why OUTPUT instead of SELECT?

| OUTPUT                  | SELECT          |
| ----------------------- | --------------- |
| Structured return       | Result set      |
| Multiple values         | Harder to parse |
| Better for status codes | Mainly for data |

-------------------------------------------------------------

## Why it is recommended to set Size for NVARCHAR?

* Without size: SQL Server throws runtime error or truncates value.

-------------------------------------------------------------

## What happens if OUTPUT param is NULL?

* If an OUTPUT parameter is NULL, EF Core returns DBNull.Value, not null, and it must be explicitly checked to avoid runtime exceptions.
* Explicit DBNull check (Recommended)

```csharp
string? msg = Convert.IsDBNull(messageParam.Value)
    ? null
    : messageParam.Value.ToString();
```

-------------------------------------------------------------

## OUTPUT Parameter vs SCOPE_IDENTITY()   

* SCOPE_IDENTITY() is used INSIDE the stored procedure
* OUTPUT parameters are used to return values OUTSIDE the stored procedure
* They are not competitors — they are used together.

| Feature                | OUTPUT Parameter       | SCOPE_IDENTITY()     |
| ---------------------- | ---------------------- | -------------------- |
| Purpose                | Return value to caller | Fetch identity value |
| Where used             | SP ↔ Application       | Inside SP only       |
| Return multiple values | ✅ Yes                 | ❌ No                 |
| Strong typing          | ✅ Yes                 | ❌ No                 |
| Accessible in C#       | ✅ Yes                 | ❌ No                 |
| Safe with triggers     | ✅ Yes (via SP)        | ✅ Yes                |
| Recommended for apps   | ✅ Yes                 | ❌ No                 |


Use SCOPE_IDENTITY() inside the stored procedure to get the identity value, and return it to the application using an OUTPUT parameter. 
Never call SCOPE_IDENTITY() directly from application code.


-------------------------------------------------------------

## When to Use EF Core + Stored Procedures?

| Scenario                | Recommendation   |
| ----------------------- | ---------------- |
| Complex joins/reporting | Stored Procedure |
| CRUD heavy apps         | EF Core LINQ     |
| Legacy DB               | Stored Procedure |
| High performance        | SP + Dapper      |
| Simple apps             | EF Core          |

-------------------------------------------------------------

 ## Cloudflare

1. Cloudflare = CDN + Security + Performance Layer

Cloudflare provides:

✅ 1. Content Delivery Network (CDN)

Caches your static resources (images, CSS, JS, videos) on servers worldwide

Makes websites load faster globally

Reduces load on your web server

✅ 2. DDoS Protection

Protects your website/API from attacks that try to overload it.

Cloudflare can block:

DDoS attacks

Bot traffic

Malicious IPs

Automated scraping

✅ 3. WAF (Web Application Firewall)

Stops:

SQL Injection

XSS

CSRF

OWASP Top 10 vulnerabilities

Useful for ASP.NET Core APIs.

✅ 4. DNS Management

Cloudflare offers fast, globally distributed DNS.

Very quick domain resolution

Highly reliable (less downtime)

✅ 5. SSL/TLS

Gives free HTTPS certificates, with automatic renewal.

Modes:

Flexible SSL

Full SSL

Full (Strict)

✅ 6. Reverse Proxy / Edge Network

Cloudflare processes requests before they reach your server.

Example:

User → Cloudflare → Your Server


This reduces:

Server load

Bandwidth usage

Attack risk

✅ 7. Caching for APIs (Edge Caching)

Cloudflare Workers & Edge Cache can cache even API responses.


 -------------------------------------------------------------

 ## Interface vs Abstract class

| Feature                              | **Interface**                                       | **Abstract Class**                                        |
| ------------------------------------ | --------------------------------------------------- | --------------------------------------------------------- |
| Contains implementation?             | ❌ Before C# 8<br>✔ Default methods allowed in C# 8+ | ✔ Can contain full implementation                         |
| Fields allowed?                      | ❌ No instance fields                                | ✔ Yes, fields allowed                                     |
| Constructor?                         | ❌ Not allowed                                       | ✔ Allowed                                                 |
| Access modifiers?                    | ❌ All members are public                            | ✔ Members can have any access modifier                    |
| Multiple inheritance?                | ✔ Supports multiple interfaces                      | ❌ A class can inherit only one abstract class             |
| Must implement all members?          | ✔ Yes (unless default interface methods)            | ❌ No, because abstract class can have concrete methods    |
| When to use?                         | Define **contract/behavior**                        | Create **base class + shared logic**                      |
| Supports constants?                  | ✔ Yes                                               | ✔ Yes                                                     |
| Supports properties/events/indexers? | ✔ Yes                                               | ✔ Yes                                                     |
| Can be instantiated?                 | ❌ No                                                | ❌ No                                                      |
| Versioning                           | ❌ Harder                                            | ✔ Easier (because default implementations work naturally) |

 -------------------------------------------------------------

 ## Abstract vs Virtual Method

 | Feature                | **Abstract Method**                          | **Virtual Method**                             |
| ---------------------- | -------------------------------------------- | ---------------------------------------------- |
| Has implementation?    | ❌ No                                         | ✅ Yes (default implementation)                 |
| Must be overridden?    | ✅ Yes                                        | ❌ No (optional)                                |
| Declared inside?       | Only in **abstract class**                   | In **regular** or **abstract class**           |
| Purpose                | Force subclass to provide implementation     | Allow subclass to optionally override behavior |
| Can instantiate class? | ❌ No (abstract class cannot be instantiated) | ✔ Yes (if class is not abstract)               |
| Polymorphism?          | ✔ Yes                                        | ✔ Yes                                          |


### Abstract Method

 ```csharp
public abstract class Shape
{
    public abstract double Area();  // No body → must override
}


public class Circle : Shape
{
    public double Radius { get; set; }
    public override double Area() => Math.PI * Radius * Radius;
}
 ```

 ### Virtual Method

 ```csharp
public class Animal
{
    public virtual void Speak()
    {
        Console.WriteLine("Animal makes a sound");
    }
}

public class Dog : Animal
{
    public override void Speak()
    {
        Console.WriteLine("Dog barks");
    }
}
 ```

 -------------------------------------------------------------

 ## Generic collections

* Generic collections in C# (List<T>, Dictionary<TKey,TValue>, HashSet<T>, Queue<T>, Stack<T>) provide type-safe methods for adding, removing, searching, and iterating over elements.
* They also support LINQ extension methods for querying data.
* Interfaces like IEnumerable<T> and ICollection<T> define common operations for all generic collections.

### List:

 | Method                      | Purpose                  |
| --------------------------- | ------------------------ |
| `Add(T item)`               | Adds an item             |
| `AddRange(IEnumerable<T>)`  | Adds multiple items      |
| `Insert(int index, T item)` | Inserts at position      |
| `Remove(T item)`            | Removes first occurrence |
| `RemoveAt(int index)`       | Removes by index         |
| `Clear()`                   | Removes all items        |
| `Contains(T item)`          | Checks if item exists    |
| `IndexOf(T item)`           | Gets index               |
| `Sort()`                    | Sorts list               |
| `Reverse()`                 | Reverse order            |
| `Find(Func<T, bool>)`       | Returns first match      |
| `FindAll(Func<T, bool>)`    | Returns all matches      |
| `Exists(Func<T, bool>)`     | Returns true/false       |
| `ForEach(Action<T>)`        | Loop with action         |


### Dictionary<TKey, TValue>:

| Method                        | Purpose                |
| ----------------------------- | ---------------------- |
| `Add(key, value)`             | Adds key-value pair    |
| `Remove(key)`                 | Removes by key         |
| `ContainsKey(key)`            | Checks if key exists   |
| `ContainsValue(value)`        | Checks if value exists |
| `TryGetValue(key, out value)` | Safe get               |
| `Clear()`                     | Remove all             |


### Generic Collection Methods (HashSet<T>)

 | Method               | Purpose                  |
| -------------------- | ------------------------ |
| `Add(item)`          | Adds unique item         |
| `Remove(item)`       | Remove item              |
| `Contains(item)`     | Check existence          |
| `UnionWith(set)`     | Merge sets               |
| `IntersectWith(set)` | Common elements          |
| `ExceptWith(set)`    | Remove overlapping items |


### Generic Collection Methods (Stack<T>)

| Method       | Purpose         |
| ------------ | --------------- |
| `Push(item)` | Add to stack    |
| `Pop()`      | Remove from top |
| `Peek()`     | Look at top     |
| `Clear()`    | Remove all      |


### Generic Collection Methods (Queue<T>)


| Method          | Purpose           |
| --------------- | ----------------- |
| `Enqueue(item)` | Add to queue      |
| `Dequeue()`     | Remove from front |
| `Peek()`        | See first element |
| `Clear()`       | Remove all        |



### IEnumerable<T>

GetEnumerator() → supports foreach

### ICollection<T>

| Method                         | Purpose       |
| ------------------------------ | ------------- |
| `Add`                          | Add item      |
| `Remove`                       | Remove item   |
| `Contains`                     | Exists check  |
| `CopyTo(T[] array, int index)` | Copy elements |
| `Count`                        | Total items   |


### LINQ Extension Methods (Work on all Generic Collections)

| Category       | Methods                                                    |
| -------------- | ---------------------------------------------------------- |
| Filtering      | `Where`, `First`, `FirstOrDefault`, `Single`, `Any`, `All` |
| Projection     | `Select`, `SelectMany`                                     |
| Sorting        | `OrderBy`, `OrderByDescending`                             |
| Aggregation    | `Count`, `Sum`, `Min`, `Max`, `Average`                    |
| Grouping       | `GroupBy`                                                  |
| Joining        | `Join`, `GroupJoin`                                        |
| Set Operations | `Distinct`, `Union`, `Intersect`, `Except`                 |
| Conversion     | `ToList`, `ToArray`, `ToDictionary`                        |

```csharp
var result = users
    .Where(u => u.IsActive)
    .OrderBy(u => u.Name)
    .ToList();
```

-------------------------------------------------------------

## How Delegate is delared ?

```csharp
public delegate void MyDelegate(string msg);

public class Demo
{
    public static void Print(string msg)
    {
        Console.WriteLine(msg);
    }

    public static void Main()
    {
        MyDelegate d = Print;   // method reference
        d("Hello World");
    }
}
```

### Usage:

* Asynchronous operation : Fund Transfer, Fetching Transaction History
* Event Notifications : SMS, Email
* 3rd Party Integration : UPI
* Background processing : Interest Calculation, Settlements

-------------------------------------------------------------


### Difference between .NET Framework & .NET Core

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
## What is Task Parallel Library (TPL) ?

* It is part of the System.Threading.Tasks namespace.
* The Task Parallel Library (TPL) is a set of APIs in .NET that simplifies writing asynchronous and parallel code.
* It uses the ThreadPool efficiently, supports tasks, parallel loops (Parallel.For, Parallel.ForEach), and PLINQ.
* TPL is the foundation of the async/await pattern and provides better performance, easier coding, and automatic thread management compared to manual threading.

### Key Features of TPL<br/>
1. Task-based programming model

Instead of manually creating threads:

```csharp
var task = Task.Run(() => DoWork());
```

2. Automatic thread-pool usage<br/>

* You don’t manage threads.
* TPL uses the ThreadPool efficiently.

3. Parallel execution<br/>

* Works across all CPU cores:

```csharp
Parallel.For(0, 10, i =>
{
    Console.WriteLine(i);
});
```

4. Supports async/await<br/>

* TPL is the core of async/await in C#.

```csharp
await Task.Delay(1000);
```

5. Continuations<br/>

* Execute something after a task completes:

```csharp
task.ContinueWith(t => Console.WriteLine("Task finished"));
```

6. Aggregates exceptions cleanly<br/>

* Multiple exceptions → wrapped into AggregateException.

### TPL Components

1. Task<br/>

* A representation of an asynchronous operation.

2. Parallel class<br/>

Used for parallel loops:<br/>

* Parallel.Invoke
* Parallel.For
* Parallel.ForEach

3. PLINQ (Parallel LINQ)<br/>

Speeds up LINQ using multiple processors.<br/>

```csharp
var result = numbers.AsParallel().Where(n => n % 2 == 0).ToList();
```

4. TaskScheduler<br/>

Controls how tasks are scheduled.<br/>

### Example: Simple TPL Task

```csharp
Task task = Task.Run(() =>
{
    Console.WriteLine("Running on a background task");
});

task.Wait();
```

### Example: Parallel.For

```csharp
Parallel.For(0, 5, i =>
{
    Console.WriteLine($"Processing {i}");
});
```

### Uses multiple CPU cores automatically.

### Example: Async/Await Uses TPL Under the Hood

```csharp
public async Task FetchDataAsync()
{
    await Task.Delay(1000); // Uses TPL internally
}
```

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


## Global exception handling ?

Global exception handling centralizes error handling for the entire application. In ASP.NET Core, it is implemented using UseExceptionHandler(), custom exception middleware, or MVC exception filters. The preferred method is middleware because it captures all exceptions across the entire request pipeline and ensures consistent logging and error responses.<br/>

It ensures:<br/>
✔ Unified error responses<br/>
✔ Logging in a single place<br/>
✔ Cleaner, maintainable code<br/>
✔ No leaking of stack traces to clients<br/>


### Why Do We Need Global Exception Handling ?

Without global handling, every unhandled exception:
* Crashes the request
* Produces random HTTP 500 pages
* Returns inconsistent responses
* Pollutes logs

With global handling:
* One middleware handles ALL exceptions
* Standard error format
* Consistent HTTP status codes

<br/>

| Approach                        | When to Use                                   |
| ------------------------------- | --------------------------------------------- |
| **UseExceptionHandler()**       | Most production-ready apps; simple + reliable |
| **Custom Exception Middleware** | Pure APIs, microservices, need custom JSON    |
| **Filters**                     | Only inside MVC, not global pipeline          |


* Using UseExceptionHandler() (Built-in Middleware)

Best for APIs and MVC apps.<br/>

#### Program.cs:

```csharp
app.UseExceptionHandler("/error");
```

#### ErrorController.cs

```csharp
[ApiController]
public class ErrorController : ControllerBase
{
    [Route("error")]
    public IActionResult HandleError() =>
        Problem();  // Returns RFC 7807 standard error response
}
```

Advantages:<br/>
1) Clean<br/>
2) Built-in<br/>
3) Recommended for production<br/>

* Custom Exception Handling Middleware<br/>

Used when you want full control.<br/>

#### Middleware:

```csharp
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
```

#### Register:

```csharp
app.UseMiddleware<GlobalExceptionMiddleware>();
```

Advantages:<br/>
✔ Full control over logging<br/>
✔ Custom JSON format<br/>
✔ Great for APIs<br/>

* Exception Filters (Only MVC, Not Middleware)

#### Runs only for MVC controllers.

```csharp
public class GlobalExceptionFilter : IExceptionFilter
{
    public void OnException(ExceptionContext context)
    {
        context.Result = new ObjectResult(new { message = "Error occurred" })
        {
            StatusCode = 500
        };
        context.ExceptionHandled = true;
    }
}
```

#### Register:

```csharp
builder.Services.AddControllers(options =>
{
    options.Filters.Add<GlobalExceptionFilter>();
});
```

#### Global Exception Handling Flow (Request Pipeline):

```csharp
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
```

-------------------------------------------------------------

## Logging in Asp.Net Core ?

* Most commonly used (default in every ASP.NET Core app). 
* Lightweight, fast, structured, and integrated with DI.
* Works with: Console, Debug, EventLog, Azure, Application Insights.

This is what most enterprise apps use, often with an external provider like Serilog plugged in.

Serilog (Most Popular Third-Party Logger)

### Why Serilog is popular?

* Structured logging (JSON logs)
* Powerful sinks (write to DB, Elasticsearch, File, Seq, Graylog, Splunk, etc.)
* Enrichers (add machine name, request id, user, IP)
* Simple configuration in Program.cs
* Excellent for production logging

### Usages: 
Microservices
Cloud apps (Kubernetes, Docker)
APIs with centralized logging

### Nuget package Serilog.AspNetCore, Serilog.Sinks

### Program.cs

```csharp
using Serilog;

var builder = WebApplication.CreateBuilder(args);

// 1️⃣ Configure Serilog
Log.Logger = new LoggerConfiguration()
    .ReadFrom.Configuration(builder.Configuration)     // read from appsettings.json
    .Enrich.FromLogContext()                           // include RequestId, CorrelationId, etc.
    .WriteTo.Console()
    .WriteTo.File("logs/app.log",
        rollingInterval: RollingInterval.Day,
        retainedFileCountLimit: 7)
    .CreateLogger();

// 2️⃣ Replace default logging with Serilog
builder.Host.UseSerilog();

var app = builder.Build();

app.MapGet("/", () => "Hello World!");

app.Run();
```

### appsettings.json

```csharp
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "System": "Warning"
      }
    },
    "WriteTo": [
      {
        "Name": "Console"
      },
      {
        "Name": "File",
        "Args": {
          "path": "logs/log-.txt",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 7
        }
      }
    ],
    "Enrich": [ "FromLogContext", "WithMachineName", "WithThreadId" ]
  }
}
```

### Logging Inside Controllers / Services:

```csharp
public class TestService
{
    private readonly ILogger<TestService> _logger;

    public TestService(ILogger<TestService> logger)
    {
        _logger = logger;
    }

    public void DoWork()
    {
        _logger.LogInformation("Doing some work at {Time}", DateTime.UtcNow);
    }
}
```

### Enable Request Logging:

* HTTP method
* URL
* Response time
* Status code
* Exceptions

```csharp
app.UseSerilogRequestLogging();
```

-------------------------------------------------------------

## How SeriLog Logging in Asp.Net Core working actually ?

e.g.

```csharp
_logger.LogInformation("Doing some work at {Time}", DateTime.UtcNow);
```


```csharp
_logger.LogInformation(...) // This goes through the standard Microsoft.Extensions.Logging pipeline.
```

But because we configured:

```csharp
builder.Host.UseSerilog();
```

ASP.NET Core automatically forwards all log messages to Serilog’s sinks (console, file).

-------------------------------------------------------------

## Why placeholders matter ({Time}) ?

```csharp
Doing some work at {Time}
```

Serilog cannot extract structured properties → It becomes plain text and cannot be filtered or queried later.

Using {PropertyName} creates structured logs, usable in:<br/>

* Kibana (Elasticsearch)
* Seq
* Loki / Grafana
* Splunk
* Datadog

-------------------------------------------------------------

## What is ELK ?

ELK is a centralized logging stack made of Elasticsearch, Logstash, and Kibana. Elasticsearch stores and searches logs, Logstash collects and transforms logs, and Kibana visualizes logs and helps in monitoring and debugging systems. It is widely used for log aggregation, analytics, and observability in distributed applications.
It is widely used with ASP.NET Core, microservices, Docker, and cloud environments.<br/>


### Elasticsearch

A distributed search and analytics engine.

* Stores logs as JSON documents
* Fast full-text search
* Distributed, scalable cluster
* Used to query millions of logs quickly
* Nuget Serilog.Sinks.Elasticsearch

Think of it as a large, scalable NoSQL database optimized for search.<br/>

### Logstash

A data processing pipeline.<br/>

* Collects logs
* Parses/filters/transforms them
* Sends them to Elasticsearch

It can read from:<br/>

* Files
* TCP/UDP
* Kafka
* Beats
* Syslog
* Application log forwarders<br/>

Logstash = log collector + transformer + shipper.<br/>

### Kibana

A visualization and dashboard tool.

* Shows logs stored in Elasticsearch
* Provides dashboards, graphs, alerts
* Helps developers analyze issues
* Used for monitoring services, failures, performance<br/>

Kibana = UI for searching logs and building dashboards.

```csharp
App → Logstash → Elasticsearch → Kibana
```

### Use Cases:

* Tracking API errors and exceptions
* Monitoring performance issues
* Auditing user activity
* Analyzing request logs
* Debugging distributed microservice issues
* Alerting on failures (via Elastic alerts)

### Why ELK is popular?

* Centralized logging across many services
* Super fast searching & filtering
* JSON-based structured logs work perfectly with Serilog, NLog, etc.
* Great visualization dashboards
* Scales horizontally
* Works perfectly with microservices & Kubernetes

-------------------------------------------------------------

## What is mocking in Unit Testing ?

Mocking in NUnit means creating fake versions of dependencies 
so you can test your class without using the real implementations 
(like real database, email service, API calls, etc.).

```csharp
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
```

-------------------------------------------------------------

## Why Does the Test Pass Even When No Real Email Was Sent?

Because unit testing is NOT meant to test external systems(like email server, SMTP, database, API).
Unit test should test only the logic inside your class, not the dependencies.<br/>

In unit testing:<br/>
* We do NOT test whether an email is actually delivered
* We test whether our class attempted to send the email correctly
* The email sending itself is tested in integration tests, not unit tests

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

* Lazy Loading loads related data only when navigation properties are accessed, causing many small queries.
* Eager Loading loads related data upfront using .Include(), usually in a single optimized query.
* Explicit Loading loads related data on demand using Entry().Load(), giving full control without automatic queries.

### Lazy Loading:

Entities are loaded only when you access navigation properties. EF makes an automatic query behind the scenes when needed.

```csharp
var student = context.Students.First();

var courses = student.Courses;   // **Lazy Loading** triggers a DB query here
```

### Eager Loading:

Related data is loaded upfront, at the same time as the main entity. Use .Include() or .ThenInclude().

```csharp
var student = context.Students
                     .Include(s => s.Courses)
                     .First();
```

### Explicit Loading

Manually load related data only when you want, using Entry(). Not automatic (like Lazy), not upfront (like Eager).

```csharp
var student = context.Students.First();

// Later, load only if needed:
context.Entry(student)
       .Collection(s => s.Courses)
       .Load();

```


| Feature               | Lazy Loading                     | Eager Loading       | Explicit Loading                      |
| --------------------- | -------------------------------- | ------------------- | ------------------------------------- |
| **When data loads?**  | Only when navigation accessed    | With main query     | When manually requested               |
| **How?**              | EF generates query automatically | `.Include()`        | `context.Entry().Collection().Load()` |
| **Number of queries** | Many (risk of N+1)               | Typically 1         | As many as you trigger                |
| **Control**           | Low                              | Medium              | High                                  |
| **Performance**       | Worst (hidden queries)           | Best                | Good                                  |
| **Use Case**          | Very small apps                  | APIs, microservices | Complex scenarios                     |



-------------------------------------------------------------

## When should you use AsNoTracking()?

When EF Core fetches entities, by default it tracks them so it can detect changes and later save them using SaveChanges(). 
AsNoTracking() is an Entity Framework Core query optimization method that tells EF Core NOT to track the returned entities in the DbContext's Change Tracker.

Tracking adds overhead:
* Memory (keeps copies of original values)
* CPU (detects changes)
* Change tracker overhead increases with large result sets

```csharp
var users = await _context.Users
    .AsNoTracking()
    .ToListAsync();
```

* Read-only queries (most common)
* High-performance reporting / dashboards
* When fetching referenced data but not modifying it
* When loading large collections (millions of rows)

-------------------------------------------------------------

## How to use IDbContextTransaction?

```csharp
using var transaction = await _context.Database.BeginTransactionAsync();

try
{
    var product = new Product { Name = "Laptop" };
    _context.Products.Add(product);
    await _context.SaveChangesAsync();

    var order = new Order { ProductId = product.Id };
    _context.Orders.Add(order);
    await _context.SaveChangesAsync();

    await transaction.CommitAsync();  // All good → commit
}
catch (Exception)
{
    await transaction.RollbackAsync(); // Something failed → rollback
    throw;
}
```

What happens?<br/>

* If both SaveChanges succeed → transaction commits
* If any SaveChanges fails → everything is rolled back
* Database stays consistent


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

| Feature                        | **Response Caching**             | **Output Caching**      | **Distributed Caching**                |
| ------------------------------ | -------------------------------- | ----------------------- | -------------------------------------- |
| Where cached?                  | **Client** (Browser, CDN, Proxy) | **Server** (App memory) | **External cache store** (Redis/SQL)   |
| What is cached?                | Full HTTP response               | Full HTTP response      | Raw data / objects (not full response) |
| Who controls it?               | Browser/CDN via headers          | Server middleware       | Application code                       |
| Cache survives server restart? | ✔ Yes                            | ❌ No                    | ✔ Yes                                  |
| Works for authenticated users? | ❌ Generally no                   | ✔ Yes                   | ✔ Yes                                  |
| Main benefit                   | Reduce network calls             | Boost throughput        | Share cache across servers             |
| Introduced                     | Since .NET Core 1                | .NET 7+                 | Since .NET Core 1                      |


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

### What Is Caching?

Caching is the technique of storing frequently accessed data in fast storage (memory, Redis, file) so future requests can access it quickly without recomputing or hitting the database.<br/>

🧩 Types of Caching Strategies: <br/>

#### 1️) In-Memory Cache (IMemoryCache)<br/>

Stored in application memory, fastest cache.<br/>

✔ Best for:<br/>

* Single server deployments
* Small data (< few MBs)
* Frequently accessed, rarely changed data

Example:<br/>

```csharp
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
```

#### 2️) Distributed Cache (Redis / SQL Server Cache)<br/>

Stored outside the application, in Redis or SQL.<br/>

✔ Best for:<br/>

* Load-balanced / multi-server applications
* Microservices
* Large caching needs

Example:<br/>

```csharp
services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379";
});
```

#### 3️) Response Caching<br/>

Caches entire HTTP responses.<br/>

✔ Best for:

* GET endpoints returning static or semi-static data
* Public APIs with largely identical responses

```csharp
app.UseResponseCaching();

[ResponseCache(Duration = 60)]
public IActionResult Get() => Ok();
```

#### 4️) Output Caching (New in .NET 8)<br/>

* More powerful than response caching:
* Can cache POST responses
* Supports tags, vary-by, invalidation

```csharp
builder.Services.AddOutputCache();

app.MapGet("/products", () => data).CacheOutput();
```

#### 5️) Client-Side Caching<br/>

Use browser cache headers.<br/>

Examples:<br/>

* Cache-Control
* ETag
* Last-Modified

✔ Best for:

* Static files
* Images, CSS, JS

```csharp
app.UseStaticFiles(new StaticFileOptions
{
    OnPrepareResponse = ctx =>
    {
        ctx.Context.Response.Headers["Cache-Control"] = "public,max-age=86400";
    }
});
```

#### 6️) Distributed Memory Caching (Hybrid)<br/>

Combination of in-memory + Redis, where local memory is used for fast reads, Redis for consistency.<br/>

#### 7️) Query Caching (EF Core)<br/>

Cache repeated database calls manually like:<br/>

```csharp
var users = await _cache.GetOrCreateAsync("users_all", async e =>
{
    e.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);
    return await context.Users.AsNoTracking().ToListAsync();
});
```

### Caching Patterns (Strategies)
#### A. Absolute Expiration

Cache expires after fixed duration.<br/>

```csharp
entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);
```

✔ Predictable<br/>
✘ Might serve stale data<br/>

#### B. Sliding Expiration

Resets expiration time each time the cache is accessed.<br/>

```csharp
entry.SlidingExpiration = TimeSpan.FromMinutes(5);
```

✔ Great for frequently accessed but rarely updated data<br/>
✘ Cache may stay forever if constantly used<br/>

#### C. Cache Aside (Lazy Loading)

Most common pattern.<br/>

Check cache<br/>

If not found → load from DB<br/>

Put data into cache<br/>

Return<br/>

* Simple
* Efficient
* Works for most apps

#### D. Write-Through Cache

When data is written to DB, it's also written to cache immediately.<br/>

✔ Cache always consistent<br/>
✘ Slight write performance hit<br/>

#### E. Write-Behind Cache

Write to cache → DB updated later in background.<br/>

✔ Super fast writes<br/>
✘ Risk of data inconsistency<br/>

#### F. Read-Through Cache

Application always reads through the cache provider, which loads from DB automatically.<br/>

✔ Encapsulated, clean<br/>
✘ Provider complexity<br/>

#### G. Cache Invalidation Strategies

The hardest part of caching.<br/>

Techniques:<br/>
* Time-based expiration
* Manual invalidation via key
* Tag-based invalidation (Output Cache / Redis)
* Event-based invalidation (e.g., on DB change)


#### Types
* In-memory cache: very fast, single-instance.
* Distributed cache (Redis, Memcached): shared across instances and durable for scaling.

#### Patterns
* Cache-aside (application handles cache): common — read from cache, fallback to DB then set cache.
* Read-through/write-through: cache handles reading/writing (requires specialized cache).
* Prevent cache stampede: use locks, jitter, randomized TTLs, or request coalescing.

#### Eviction & TTL
* Use sliding vs absolute expiration wisely. Keep caches small and invalidate on writes if necessary.

```csharp
var key = $"product:{id}";
var cached = await _cache.GetStringAsync(key);
if (cached != null) return JsonConvert.DeserializeObject<Product>(cached);
var product = await _repo.Get(id);
await _cache.SetStringAsync(key, JsonConvert.SerializeObject(product), new DistributedCacheEntryOptions{ AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5) });
return product;
```


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

-------------------------------------------------------------
# MicroServices:

## What is bounded context?

A Bounded Context is one of the most important concepts in Domain-Driven Design (DDD).
It defines a clear boundary in your system within which a particular domain model is valid.

-------------------------------------------------------------

## Why Do We Need Bounded Contexts?

Because large applications become messy if you try to force one big model everywhere.
Different parts of your business may use the same word but mean different things.

-------------------------------------------------------------

## Real Example: E-commerce System

You would split into multiple bounded contexts:


| Bounded Context | What it Contains                    |
| --------------- | ----------------------------------- |
| **Ordering**    | Cart, Order, OrderItem, Order Rules |
| **Inventory**   | Stock, Warehouse, Quantity rules    |
| **Payment**     | Payment, Invoice, Payment Rules     |
| **Shipping**    | Shipment, Tracking, Delivery Rules  |
| **Catalog**     | Products, Categories, Attributes    |


Each context has: </br>

* Different models
* Different logic
* Different teams
* Often different databases
* Clear boundaries

-------------------------------------------------------------

## How Contexts Interact — Context Mapping

Bounded contexts communicate via:

* Domain events
* API calls
* Message brokers (RabbitMQ, Kafka)
* Anti-corruption layer

Each context protects its internal models from being polluted by others.

Example in Microservices

Each microservice typically = 1 bounded context.

* Ordering Service
* Inventory Service
* Payment Service
* Shipping Service

Each one owns its data.

-------------------------------------------------------------

## How to design a microservice communication system?

### 1) Start with the goals & constraints</br>

* Always begin by answering:
* Latency requirements (sub-second v/s minutes)
* Consistency model (strong v/s eventual)
* Throughput and scale targets
* Fault tolerance / availability SLAs
* Team boundaries (who owns what)
* Operational constraints (Kubernetes? Cloud? On-prem?)

### 2) Two fundamental communication styles</br>

#### A. Synchronous (request/response)</br>
 
* Protocols: HTTP/REST, gRPC
* Use when: caller needs immediate result (auth, lookup, simple CRUD).
* Pros: simple, easy debugging, straightforward semantics.
* Cons: tight coupling, cascading failures, higher latency under load.

#### B. Asynchronous (message/event-driven)</br>

* Protocols: message brokers (Kafka, RabbitMQ, AWS SNS/SQS), streaming.
* Use when: eventual consistency is acceptable; decoupling, resilience, high throughput.
* Pros: loose coupling, buffering, retries, scale.
* Cons: complexity, eventual consistency, more operational overhead.

### 3) Patterns & architectures</br>


#### A. API Gateway + Edge</br>

* Single entry for clients: authentication, TLS termination, rate limiting, routing, versioning.
* Offloads cross-cutting concerns from services.

#### B. Service Discovery & Load Balancing</br>

* Static endpoints, or dynamic (Consul, Kubernetes DNS, AWS ALB).
* Client-side (gRPC + client LB) or server-side (gateway/load balancer).

### C. Choreography vs Orchestration</br>

* Choreography: services emit events; others react. Simple, loosely coupled.
* Orchestration: a central orchestrator (workflow engine, saga coordinator) drives steps. Easier for complex long-running processes.

### D. Saga Pattern for distributed transactions</br>

* Sequence of local transactions + compensating actions.

Two implementations:</br>

* Choreography: services publish/subscribe events.
* Orchestration: saga orchestrator issues commands and handles failures.

### E. Command Query Responsibility Segregation (CQRS)</br>

* Separate read & write models. Combine with event sourcing if needed.

### 4) Reliable messaging & delivery semantics

* At-most-once: simple, may lose messages.
* At-least-once: common (Kafka, RabbitMQ). Requires idempotency & deduplication.
* Exactly-once: hard; some systems provide transactional guarantees (Kafka transactions, some cloud services).

### 5) Failure handling & resilience

* Retries: exponential backoff + jitter.
* Circuit Breaker: avoid cascading failures (Polly for .NET).
* Bulkhead isolation: separate resources per service.
* Timeouts: short, explicit.
* Backpressure: broker-based buffering, rate limits.
* Health checks: readiness vs liveness.

### 6) Observability & tracing

* Correlation/Trace ID: pass a request id through HTTP headers and messages (X-Request-Id, traceparent).
* Distributed tracing: OpenTelemetry, Jaeger, Zipkin.
* Centralized logging: structured logs → ELK/EFK/Seq.
* Metrics & Alerts: Prometheus + Grafana, latency & error SLOs.
* Audit / events: persistent event store for important business events.

### 7) Security

* Authentication & Authorization: OAuth2 / OpenID Connect at gateway or per-service.
* mTLS or network policy inside cluster.
* Encrypt data in transit (TLS).
* Service-to-service auth: JWT or short-lived mTLS certs.
* Secrets management: Vault, Kubernetes Secrets, AWS Secrets Manager.

### 8) Data consistency & ownership

* Database-per-service (recommended) to avoid coupling.
* Use eventual consistency when cross-service data must be synchronized.
* Use CDC (Change Data Capture) + event streaming for syncing data into other services/analytics.
* Use anti-corruption layer when integrating legacy systems.

### 9) Example: Order processing (concrete flow)

Two flavors — Choreography (event-driven) and Orchestration.

#### Choreography (event-driven)

* Client POST /orders → Order service creates Order (state: Created) and emits OrderCreated event to Kafka.
* Payment service consumes OrderCreated, attempts payment → emits PaymentSucceeded or PaymentFailed.
* Inventory service consumes PaymentSucceeded, reserves stock → emits InventoryReserved or InventoryFailed.
* Shipping consumes InventoryReserved → schedules shipment → emits ShipmentScheduled.
* Order service listens and updates order state as events arrive.

Pros: highly decoupled; good scaling.</br>
Cons: reasoning about global state is distributed; need idempotency, retries, and eventual consistency.</br>

#### Orchestration (saga coordinator)

* Order service calls Saga Orchestrator to start saga.
* Orchestrator commands Payment, waits response, then commands Inventory, etc.
* On failure, orchestrator triggers compensating actions (refund, restock).

Pros: easier to follow the flow & implement compensations.</br>

### 10) Concrete tech stack suggestions

* Transport: HTTP/REST + gRPC (internal), Kafka or RabbitMQ for async.
* Gateway: Kong / Nginx / Envoy / Ocelot / AWS API Gateway.
* Service Discovery: Kubernetes DNS / Consul.
* Circuit Breaker & Retry: Polly (.NET) or client libraries.
* Tracing: OpenTelemetry, Jaeger.
* Logging: Serilog → Elasticsearch / Seq.
* Message schema: Protobuf (gRPC) or Avro with Schema Registry (Kafka).
* Orchestrator: Temporal / MassTransit saga / custom coordinator.
* Deployment: Kubernetes (Helm), containers.

### 11) Practical implementation checklist

* API Gateway for clients
* Well-defined service APIs (OpenAPI)
* Use TLS everywhere
* Service discovery & load balancing
* Choose message broker + schema registry
* Implement idempotency keys for command endpoints / message handlers
* Add correlation IDs for tracing
* Retries with exponential backoff + jitter
* Dead-letter queue + visibility/alerting
* Circuit breakers and bulkheads
* Monitoring: distributed traces, metrics, dashboards, alerts
* Contract/versioning policy (both sync & async)
* Security: OAuth2, mTLS, secrets manager
* Test harness: integration tests with test broker, chaos testing

### 11) Short code sketches (pseudo / .NET oriented)

#### Produce an event after creating an order:

```csharp
// create order within EF transaction
await _dbContext.Orders.AddAsync(order);
await _dbContext.SaveChangesAsync();

// publish event (ensure publish is retried or stored in outbox)
await _messageBus.PublishAsync(new OrderCreated { OrderId = order.Id, Total = order.Total });
```

#### Idempotent handler:

```csharp
public async Task Handle(PaymentSucceeded evt)
{
    if (await _processedEventStore.ExistsAsync(evt.EventId)) return; // dedupe
    await _processedEventStore.MarkProcessedAsync(evt.EventId);

    var order = await _orderRepo.Get(evt.OrderId);
    order.MarkPaid();
    await _orderRepo.Update(order);
}
```

#### Using outbox pattern (recommended):

* Write domain changes + outgoing events to DB in same transaction.
* A separate process reads outbox table and publishes to broker, marking them sent.

#### Common pitfalls & anti-patterns

* Synchronous calls across many services in a single user request → high latency & brittle.
* Not designing for idempotency → duplicate side effects.
* Tight coupling on database schemas between services.
* No schema management → breaking consumers on change.
* No observability → hard to debug distributed failures.
* No DLQ or retries → lost/poison messages.

-------------------------------------------------------------

## What is an API Gateway?

In a microservices architecture, an API Gateway is a single entry point for all client requests. Instead of clients calling each microservice directly, all requests go through the gateway.</br>

Why?</br>

Because microservices architecture has:</br>

* Many services
* Many URLs
* Different protocols
* Different authentication mechanisms

Key Responsibilities of an API Gateway</br>

1. Request Routing
* Gateway → forwards request to correct microservice.

2. Aggregation / Composition</br>
* Combines data from multiple microservices into a single response.

3. Authentication & Authorization</br>
* Validates JWT tokens
* Integrates with Identity Server / Azure AD / OAuth

4. Rate Limiting / Throttling</br>
* Prevents abuse or overload.

5. Logging & Monitoring</br>
* Centralizes logs
* Adds correlation IDs

6. Caching</br>
* Stores responses to reduce load on backend services.

7. Load Balancing</br>
* Sends requests to available instances.

-------------------------------------------------------------

## When to Use an API Gateway?

Use when:</br>
* You have many microservices
* You want a single public endpoint
* You need centralized auth/logging/routing

Avoid when:</br>
* You have a small monolithic app
* Internal microservices don't face external users

-------------------------------------------------------------

## Ocelot vs YARP

| Feature       | **Ocelot**      | **YARP**                |
| ------------- | --------------- | ----------------------- |
| Developer     | Community       | Microsoft               |
| Config        | JSON file       | JSON or code            |
| Customization | Medium          | Very high               |
| Performance   | Good            | Excellent               |
| Use Case      | Simple gateways | High-perf reverse proxy |

-------------------------------------------------------------

## Banking Application – Microservices

#### A typical banking architecture will have services like:

* AuthService – login, token issue
* CustomerService – profile, KYC
* AccountService – balance, statements
* TransactionService – transfer, payments
* CardService – debit/credit card operations
* NotificationService – email/SMS
* LoanService – loan details, EMI, eligibility


#### Each one runs on its own container/port:

| Service      | URL                                                          |
| ------------ | ------------------------------------------------------------ |
| Auth         | [http://auth-service:5001](http://auth-service:5001)         |
| Customer     | [http://customer-service:5002](http://customer-service:5002) |
| Accounts     | [http://account-service:5003](http://account-service:5003)   |
| Transactions | [http://txn-service:5004](http://txn-service:5004)           |
| Cards        | [http://card-service:5005](http://card-service:5005)         |
| Loans        | [http://loan-service:5006](http://loan-service:5006)         |

#### Goal of the API Gateway

A gateway will:

#### Provide single public entry point:
👉 https://bank.myapp.com/api

Handle:</br>
✔ JWT Validation</br>
✔ Rate limiting (prevent brute-force)</br>
✔ Logging + correlation ID</br>
✔ Routing to microservices</br>
✔ Response aggregation (e.g., "dashboard" API)</br>
✔ CORS/HTTPS</br>
✔ Request/Response transformation</br>

#### Ocelot Gateway – Route Design

| Gateway Route      | Downstream Microservice |
| ------------------ | ----------------------- |
| `/auth/**`         | AuthService             |
| `/customers/**`    | CustomerService         |
| `/accounts/**`     | AccountService          |
| `/transactions/**` | TransactionService      |
| `/cards/**`        | CardService             |
| `/loans/**`        | LoanService             |


#### ocelot.json – Banking API Gateway

```csharp
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/auth/{everything}",
      "DownstreamHostAndPorts": [
        { "Host": "auth-service", "Port": 5001 }
      ],
      "UpstreamPathTemplate": "/auth/{everything}",
      "UpstreamHttpMethod": [ "POST", "GET" ],
      "AuthenticationOptions": {
        "AuthenticationProviderKey": "JwtBearer",
        "AllowedScopes": []
      }
    },
    {
      "DownstreamPathTemplate": "/api/customers/{everything}",
      "DownstreamHostAndPorts": [
        { "Host": "customer-service", "Port": 5002 }
      ],
      "UpstreamPathTemplate": "/customers/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ],
      "AuthenticationOptions": {
        "AuthenticationProviderKey": "JwtBearer",
        "AllowedScopes": []
      }
    },
    {
      "DownstreamPathTemplate": "/api/accounts/{everything}",
      "DownstreamHostAndPorts": [
        { "Host": "account-service", "Port": 5003 }
      ],
      "UpstreamPathTemplate": "/accounts/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST" ],
      "AuthenticationOptions": {
        "AuthenticationProviderKey": "JwtBearer"
      }
    },
    {
      "DownstreamPathTemplate": "/api/transactions/{everything}",
      "DownstreamHostAndPorts": [
        { "Host": "txn-service", "Port": 5004 }
      ],
      "UpstreamPathTemplate": "/transactions/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST" ],
      "AuthenticationOptions": {
        "AuthenticationProviderKey": "JwtBearer"
      }
    }
  ],

  "GlobalConfiguration": {
    "BaseUrl": "https://bank.myapp.com/api"
  }
}
```

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddAuthentication("JwtBearer")
    .AddJwtBearer("JwtBearer", options =>
    {
        options.Authority = "https://auth.mybank.com";
        options.Audience = "banking-api";
    });

builder.Services.AddOcelot();

builder.Logging.AddSerilog();

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

await app.UseOcelot();
app.Run();
```

#### Security inside the Gateway:

1) JWT Validation

* Authenticates every request from Mobile/Web App.

2️) Rate Limiting

* Prevent brute-force or DDoS.

3️) API Key for internal microservices

* Service-to-service communication uses headers

4) Correlation ID

* Used for tracing in ELK.


#### How Client Calls Gateway

Instead of calling direct services:</br>

❌ https://account-service:5003/api/accounts/123 </br>

Client calls:</br>

✔ https://bank.myapp.com/api/accounts/123 </br>

Gateway → routes → account-service.

-------------------------------------------------------------

## How to handle Distributed transactions in microservice app ?

Distributed transactions are one of the hardest problems in microservices because each service has its own database, transactions, and autonomy.</br>
Traditional DB transactions (ACID) cannot work across multiple services.</br>

So in microservices, we handle distributed transactions using patterns and event-driven architecture, not 2-phase commit.</br>

* Microservices cannot use ACID transactions across services.
* Instead, we use Saga patterns (choreography/orchestration), Outbox pattern, and event-driven architecture to maintain consistency.
* Compensating actions are used instead of rollback.
* This ensures availability, reliability, and eventual consistency.

#### Why we can’t use ACID (2PC) in microservices?</br>

* Microservices use independent databases
* Services run in containers → dynamic, ephemeral
* 2PC (two-phase commit) is slow and blocks resources
* Breaks service autonomy
* Causes cascading failures

### 1) Saga Pattern (Most widely used)

A Saga splits a large transaction into a series of local transactions, each done by a different service, coordinated by events.

#### A. Choreography Saga (Event-based Saga)

No central coordinator.</br>
Services listen to events and publish next events.</br>

Example: Bank Money Transfer (Debit → Credit)</br>

TransactionService publishes:</br>
MoneyTransferRequested</br>

AccountService listens → Debit account</br>
Publishes AccountDebited</br>

NotificationService listens → Send SMS</br>
Publishes MoneyTransferCompleted</br>

When to use:

* Simple workflows
* Very high performance
* No complex branching

#### B. Orchestration Saga (Central Coordinator)

A Saga Orchestrator instructs services what to do.</br>

Example Flow:</br>

Orchestrator → tells AccountService: “debit the amount”</br>
AccountService returns success</br>
Orchestrator → tells TransactionService: “record transaction”</br>
Orchestrator → tells NotificationService: “send confirmation”</br>

When to use:</br>

* Complex workflows
* Compensation logic
* Multiple steps or branching

### 2) Eventual Consistency

Services update their own DB locally.</br>
Other services adjust later when they receive events.</br>

Used by:</br>
* Amazon
* Uber
* Netflix

### 3) TCC (Try Confirm Cancel)

Used for payment gateways, booking systems.</br>

Steps:</br>

* Try → Reserve resources (e.g., freeze payment amount)
* Confirm → Complete transaction
* Cancel → Rollback reservation


#### Tools used in Distributed Transactions

| Purpose             | Tools                                                |
| ------------------- | ---------------------------------------------------- |
| Message Broker      | RabbitMQ, Kafka, Azure Service Bus                   |
| Saga Orchestrator   | MassTransit, NServiceBus, Dapr Workflow, Temporal.io |
| Outbox              | EF Core Outbox, MassTransit Outbox                   |
| Distributed Tracing | OpenTelemetry, Jaeger, Zipkin                        |
| Idempotency         | Redis, DB locks                                      |

-------------------------------------------------------------

## Azure knowledge

App Service vs Function vs Container</br>

App Service: managed web hosting for web apps/APIs.</br>

Function App: serverless, event-driven, good for small tasks and pay-per-execution.</br>

Container Apps/AKS: container orchestrations for microservices with custom dependencies.</br>

Scaling</br>

Horizontal scale: add instances; Azure Load Balancer or front door distributes traffic.</br>

Vertical scaling: increase VM size; typically limited.</br>

Azure SQL vs Cosmos DB</br>

Azure SQL: relational workloads, strong ACID, SQL queries.</br>

Cosmos DB: globally-distributed, multi-model, low-latency, tunable consistency.</br>

Load Balancer</br>

Routes inbound packets to back-end VMs/instances; works at layer 4. For HTTP routing layer 7 use Application Gateway or Azure Front Door.</br>

-------------------------------------------------------------

## Performance tuning

Common levers</br>

Reduce allocations; prefer Span<T> where appropriate.</br>

Use AsNoTracking() on read-only EF queries.</br>

Avoid boxing and excessive LINQ that hides allocations.</br>

Cache expensive computations.</br>

Profiling tools</br>

dotnet-counters, dotnet-trace, PerfView, Visual Studio Profiler, and third-party profilers (JetBrains dotTrace).</br>

GC basics</br>

Generations: Gen0 (short-lived), Gen1, Gen2 (long-lived). Large Object Heap (LOH) stores >85K arrays/objects. Frequent Gen2/LOH collections indicate memory pressure.