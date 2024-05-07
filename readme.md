# Storyboard

## Prerequsites

Make sure SQL server runs:

```bash
docker run -e "ACCEPT_EULA=Y" -e "MSSQL_SA_PASSWORD=yourStrong(!)Password" -e "MSSQL_PID=Express" -p 1433:1433 -d mcr.microsoft.com/mssql/server:2022-latest 
```

## The Problem

### Create a simple web app `Playground`

```bash
dotnet new web
# Change the port to 5002: app.Run("http://localhost:5002");
dotnet run
```

Do a sample request:

```txt
@host = http://localhost:5002

###
GET {{host}}/
```

```bash
dotnet publish -r win-x64 -o out Playground.csproj
.\Playground\Playground.exe
```

Note that since .NET 8, `dotnet publish` uses _Release_ config automatically.

### Add a non-trival DTO

Add _Model.cs_:

```cs
public record Address(
    string Street,
    string City,
    string State,
    string PostalCode,
    string Country);

public record ContactInfo(
    string Email,
    string PhoneNumber);

public record CompanyInfo(
    string CompanyName,
    string Industry,
    Address CompanyAddress);

public record OrderSummary(
    int OrderId,
    DateTime OrderDate,
    decimal OrderAmount);

public record Customer(
    int CustomerId,
    string FirstName,
    string LastName,
    DateTime DateOfBirth,
    Address HomeAddress,
    ContactInfo ContactDetails,
    CompanyInfo WorkDetails,
    List<OrderSummary> OrderHistory);
```

Add _POST_ endpoint and an endpoint to shutdown the app:

```cs
app.MapPost("/customers", (Customer c, ILogger<Program> logger) =>
{
    logger.LogInformation("Received customer: {Customer}", c);
    return Results.Created("/customers/1", new CreateCustomerResponse(1));
});

app.MapGet("/shutdown", (IHostApplicationLifetime appLifetime) => appLifetime.StopApplication());

// ...

public record CreateCustomerResponse(int Id);
```

Add request to HTTP file:

```txt
###
GET {{host}}/shutdown

###
POST {{host}}/customers
Content-Type: application/json

{
  "CustomerId": 101,
  "FirstName": "Jane",
  "LastName": "Doe",
  "DateOfBirth": "1984-07-23",
  "HomeAddress": {
    "Street": "123 Elm St",
    "City": "Springfield",
    "State": "IL",
    "PostalCode": "62701",
    "Country": "USA"
  },
  "ContactDetails": {
    "Email": "jane.doe@example.com",
    "PhoneNumber": "555-1234"
  },
  "WorkDetails": {
    "CompanyName": "Doe Industries",
    "Industry": "Manufacturing",
    "CompanyAddress": {
      "Street": "456 Oak Ave",
      "City": "Springfield",
      "State": "IL",
      "PostalCode": "62702",
      "Country": "USA"
    }
  },
  "OrderHistory": [
    {
      "OrderId": 2001,
      "OrderDate": "2023-04-15",
      "OrderAmount": 150.75
    },
    {
      "OrderId": 2002,
      "OrderDate": "2023-05-01",
      "OrderAmount": 300.00
    }
  ]
}
```

### Add a database

Add SQL client and Dapper:

```xml
<ItemGroup>
    <PackageReference Include="Microsoft.Data.SqlClient" Version="5.2.0" />
    <PackageReference Include="Dapper" Version="2.1.35" />
</ItemGroup>
```

Add connection string:

```json
{
    ...,
    "ConnectionStrings": {
        "LocalSql": "Server=localhost;Database=master;User=sa;Password=mySecretPassw0rd;TrustServerCertificate=false;Encrypt=false"
    }
}
```

Add DB access:

```cs
app.MapPost("/customers", async (Customer c, ILogger<Program> logger, IConfiguration config) =>
{
    logger.LogInformation("Received customer: {Customer}", c);

    using var conn = new SqlConnection(config.GetConnectionString("LocalSql"));
    await conn.OpenAsync();
    var result = await conn.QueryAsync<int>("SELECT 1 AS [Id]");

    return Results.Ok(new CreateCustomerResponse(result.First()));
});
```

### The problem

```bash
dotnet run
```

Run HTTP requests and show that first calls are slow.

```bash
dotnet publish -r win-x64 -o out Playground.csproj
```

Copy path from exe into clipboard.

Start _PerfView_ and start collection.

Show _Advanced/JITStats_ and talk about the time JITing takes even in such a simple app.

### Dive deeper into the problem.

Add event tracing:

```cs
[EventSource(Name="Techorama2024")]
public sealed class MyAppEventSource : EventSource
{
   public readonly static MyAppEventSource Log = new();

  [Event(1)]
  public void ReadyToReceive() { WriteEvent(1, ""); }

  [Event(2)]
  public void StartRequest(string path) { WriteEvent(2, path); }

  [Event(3)]
  public void EndRequest(string path) { WriteEvent(3, path); }
}
```

Add logging:

```cs
app.Use((context, next) =>
{
    MyAppEventSource.Log.StartRequest(context.Request.Path);
    var n = next();
    MyAppEventSource.Log.EndRequest(context.Request.Path);
    return n;
});

// ...

MyAppEventSource.Log.ReadyToReceive();
```

PerfView:

* Add event tracing to _PerfView_: `*Techorama2024`
* Process filter: `Playground`
* Event type filter: `Process/start|Techorama2024`
* Analyze events in trace and write down durations

**Cold start performance is pretty bad!**
**What if we could get rid of JIT completely?**
**Enter: NativeAOT for ASP.NET Core**

## Activate NativeAOT

```csproj
<PublishAot>true</PublishAot>
<EventSourceSupport>true</EventSourceSupport>
```

```bash
dotnet publish -r win-x64 -o out Playground.csproj
```

Speak about warnings because of assembly trimming.

Start the app and show first request to `/`. It crashes. Speak about the crash because of a lack of reflection. Describe that there is no .NET at runtime anymore, there is no JITer, there is not reflection.

**We need to do the things we used to do at runtime at compile time.**

```cs
builder.Services.ConfigureHttpJsonOptions(options => options.SerializerOptions.TypeInfoResolverChain.Add(AppJsonSerializerContext.Default));


[JsonSerializable(typeof(string))]
[JsonSerializable(typeof(Customer))]
[JsonSerializable(typeof(CreateCustomerResponse))]
internal partial class AppJsonSerializerContext : JsonSerializerContext { }
```

Emit compiler-generated files:

```xml
<EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>
```

```bash
dotnet build
```

Show in generated code how `JsonSerializable` gets type information at **compile time**, not at runtime.

## Add Dapper AOT

```xml
<PackageReference Include="Dapper.AOT" Version="1.0.31" />
```

Add _Dapper.cs_:

```cs
using Dapper;

[module:DapperAot]
```

```bash
dotnet publish -r win-x64 -o out Playground.csproj
```

Fails with some strange _interceptors_ error?!?

```xml
<InterceptorsPreviewNamespaces>$(InterceptorsPreviewNamespaces);Dapper.AOT</InterceptorsPreviewNamespaces>
```

Publish again, now it works.

## What are _interceptors_?

Source generators can **add** code (e.g. `JsonSerializable`). Interceptors can "**modify**" code.

* Create new, empty console app: `dotnet new console`
* Let's intercept `Console.WriteLine`:

```cs
// See https://aka.ms/new-console-template for more information
Console.WriteLine("Hello, World!");

namespace System.Runtime.CompilerServices
{
    public sealed class InterceptsLocationAttribute : Attribute
    {
        public InterceptsLocationAttribute(string path, int line, int column)
        {
        }
    }
}

namespace MyApp
{
    file static class MyInterceptor
    {
        [System.Runtime.CompilerServices.InterceptsLocation(
            @"C:\Code\GitHub\Techorama24NativeAOT\Playground2\Program.cs", 2, 9)]
        public static void MyConsoleWriteLine(string? message)
        {
            Console.WriteLine("Intercepted: " + message);
        }
    }
}
```

We **change** code by **adding** code. **Source generates just got a lot more powerful!**

## ASP.NET Core Request Delegate Generator (RDG)

How does ASP.NET Core Minimal API normally work? It uses reflection to analyze the method signature to provide the corresponding input values.

```cs
using System.Reflection;

Delegate func = DemoClass.DoSomething;

var mi = func.GetMethodInfo();
foreach(var parameter in mi.GetParameters())
{
    Console.WriteLine($"Parameter: {parameter.Name}, Type: {parameter.ParameterType}");
}
Console.WriteLine(mi.ReturnType);

static class DemoClass
{
    public static int DoSomething(string stringInput, double doubleInput) => 42;
}
```

But with NativeAOT, we don't have reflection anymore. We need to provide the information at compile time. **Enter:  ASP.NET Core Request Delegate Generator (RDG)**

Show generated route in generated code.
