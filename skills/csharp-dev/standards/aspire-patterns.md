# Standard: Aspire 13 Patterns

> **Audience:** AI agents writing, reviewing, or refactoring .NET Aspire code.
> **Prerequisite:** Read `skills/csharp-dev/standards/idiomatic-csharp.md` for general C# rules.
> **Aspire version:** 13 — **no dotnet workloads**. All capabilities via NuGet packages only.

## Core Principle

Aspire is an **opinionated orchestrator** for distributed .NET applications. It provides service discovery, observability, and containerised hosting out of the box. Let Aspire do the wiring — application code should be infrastructure-agnostic.

---

## 1. No Workloads — NuGet Only

**Rule:** Never install Aspire as a dotnet workload. All functionality comes from NuGet packages. The AppHost references `Aspire.Hosting` and `Aspire.Hosting.*` packages. Service projects reference `Aspire.ServiceDefaults`.

```xml
<!-- AppHost.csproj -->
<ItemGroup>
  <PackageReference Include="Aspire.Hosting" Version="13.*" />
  <PackageReference Include="Aspire.Hosting.Orleans" Version="13.*" />
  <PackageReference Include="Aspire.Hosting.NodeJs" Version="13.*" />
</ItemGroup>
```

```xml
<!-- ServiceDefaults.csproj -->
<ItemGroup>
  <PackageReference Include="Microsoft.Extensions.ServiceDiscovery" Version="13.*" />
  <PackageReference Include="Microsoft.Extensions.Http.Resilience" Version="13.*" />
  <PackageReference Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" Version="*" />
  <PackageReference Include="OpenTelemetry.Extensions.Hosting" Version="*" />
  <PackageReference Include="OpenTelemetry.Instrumentation.AspNetCore" Version="*" />
  <PackageReference Include="OpenTelemetry.Instrumentation.Http" Version="*" />
  <PackageReference Include="OpenTelemetry.Instrumentation.Runtime" Version="*" />
</ItemGroup>
```

### Guidelines

- **`dotnet workload list` should show nothing Aspire-related.**
- Pin major version (`13.*`) in the AppHost. Let patch versions float.
- If a colleague says "run `dotnet workload install aspire`" — that's the old way. Redirect them.

---

## 2. AppHost Wiring

**Rule:** The AppHost is a pure orchestration project. It declares resources, their relationships, and environment configuration. No business logic.

```csharp
var builder = DistributedApplication.CreateBuilder(args);

// Infrastructure
var storage = builder.AddAzureStorage("storage")
    .RunAsEmulator();
var clusterTable = storage.AddTables("clustering");
var grainStorage = storage.AddBlobs("grain-state");

// Orleans silo
var orleans = builder.AddOrleans("default")
    .WithClustering(clusterTable)
    .WithGrainStorage("store", grainStorage);

// Backend API (hosts the Orleans silo)
var api = builder.AddProject<Projects.MyApp_Api>("api")
    .WithReference(orleans)
    .WithReference(orleans.AsClient())
    .WithExternalHttpEndpoints();

// React frontend
var web = builder.AddNpmApp("web", "../MyApp.Web", "dev")
    .WithReference(api)
    .WithHttpEndpoint(env: "PORT")
    .WithExternalHttpEndpoints()
    .PublishAsDockerFile();

builder.Build().Run();
```

### Guidelines

- **One resource per `Add*` call** — don't over-compose.
- **Infrastructure resources** (storage, databases) at the top, then services that depend on them.
- **`.WithReference()`** wires service discovery automatically — no hardcoded URLs in app code.
- **`.WithExternalHttpEndpoints()`** on anything that needs to be reachable outside the Aspire network.
- **`.PublishAsDockerFile()`** for Node.js apps — tells Aspire to containerise for deployment.

---

## 3. Service Defaults — Observability by Default

**Rule:** Every service project calls `builder.AddServiceDefaults()` which configures OpenTelemetry, health checks, service discovery, and resilience. This is the single cross-cutting setup point.

```csharp
// In ServiceDefaults project — Extensions.cs
public static IHostApplicationBuilder AddServiceDefaults(
    this IHostApplicationBuilder builder)
{
    builder.ConfigureOpenTelemetry();
    builder.AddDefaultHealthChecks();
    builder.Services.AddServiceDiscovery();
    builder.Services.ConfigureHttpClientDefaults(http =>
    {
        http.AddStandardResilienceHandler();
        http.AddServiceDiscovery();
    });
    return builder;
}

public static IHostApplicationBuilder ConfigureOpenTelemetry(
    this IHostApplicationBuilder builder)
{
    builder.Logging.AddOpenTelemetry(logging =>
    {
        logging.IncludeFormattedMessage = true;
        logging.IncludeScopes = true;
    });

    builder.Services.AddOpenTelemetry()
        .WithMetrics(metrics => metrics
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddRuntimeInstrumentation())
        .WithTracing(tracing => tracing
            .AddSource(DiagnosticHeaders.DefaultListenerName)
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation());

    builder.AddOpenTelemetryExporters();
    return builder;
}
```

### Guidelines

- **Never skip `AddServiceDefaults()`** — it's the foundation of the observability story.
- **Health checks are mandatory** — Aspire uses them for orchestration. Add domain-specific checks:
  ```csharp
  builder.Services.AddHealthChecks()
      .AddCheck("orleans-cluster", () => /* cluster health */);
  ```
- **Structured logging** — use `ILogger<T>` with message templates. OpenTelemetry exports them automatically.

---

## 4. Service Discovery

**Rule:** Use Aspire service discovery URIs (`http://api`, `http://web`) instead of hardcoded URLs. Service discovery is configured by `AddServiceDefaults()`.

### Bad

```csharp
services.AddHttpClient("api", c => c.BaseAddress = new("http://localhost:5001"));
```

### Good

```csharp
services.AddHttpClient("api", c => c.BaseAddress = new("http://api"));
```

### Guidelines

- **Service names** come from the AppHost (`builder.AddProject<T>("api")` → the name is `"api"`).
- **HTTPS** — use `https://api` when the service exposes HTTPS. Aspire handles cert routing.
- **Environment-based** — in production, service discovery resolves to real DNS. No code changes needed.

---

## 5. Containerised Hosting

**Rule:** Design all services to run in containers. Use Aspire's container orchestration for local dev and publish profiles for production.

### Guidelines

- **Dockerfile per service** — keep them minimal (multi-stage build with SDK + runtime images).
- **React apps** — `AddNpmApp(...).PublishAsDockerFile()` creates a container running the Vite dev server locally, and builds a static asset container for production.
- **Emulators for infrastructure** — `RunAsEmulator()` on Azure resources spins up Azurite/CosmosDB emulator locally. Real services in production via connection strings.
- **No `docker-compose.yml`** — Aspire replaces it. The AppHost IS your compose file.

---

## 6. Configuration and Secrets

**Rule:** Use Aspire's parameter system for configuration that varies by environment. Never hardcode connection strings.

```csharp
// AppHost
var apiKey = builder.AddParameter("api-key", secret: true);

builder.AddProject<Projects.MyApp_Api>("api")
    .WithEnvironment("API_KEY", apiKey);
```

### Guidelines

- **`secret: true`** — Aspire prompts for the value and stores it securely (not in source control).
- **Connection strings** flow automatically via `.WithReference()` — don't manually wire them.
- **`appsettings.json`** for app-level config, Aspire parameters for infrastructure config.

---

## Self-Check Table

| # | Smell | Fix |
|---|-------|-----|
| 1 | Aspire workload installed | Uninstall. Use NuGet packages only |
| 2 | Hardcoded `localhost:PORT` URLs | Service discovery URI (`http://service-name`) |
| 3 | Missing `AddServiceDefaults()` | Add to every service project's startup |
| 4 | No health checks | `AddDefaultHealthChecks()` + domain-specific checks |
| 5 | `docker-compose.yml` alongside Aspire | Remove — AppHost replaces it |
| 6 | Connection strings in appsettings | Use `.WithReference()` or `AddParameter()` |
| 7 | Business logic in AppHost | Move to service project. AppHost is orchestration only |
| 8 | Missing OpenTelemetry configuration | `ConfigureOpenTelemetry()` in service defaults |
