# Aspire Patterns & Practices: AskVantage

This document focuses specifically on **.NET Aspire patterns and practices** used in the AskVantage application. Understanding these patterns will help you build future Aspire applications.

**Reference**: [Aspire Documentation](https://aspire.dev/docs/)

---

## 1. AppHost Architecture Pattern

**Location**: `src/AskVantage/Aspire/AskVantage.AppHost/Program.cs`

The AppHost is the **orchestration layer** that defines your entire application topology in code. This is a core Aspire concept - no YAML, no configuration drift, just C# that describes everything.

```csharp
var builder = DistributedApplication.CreateBuilder(args);
// ... define resources ...
builder.Build().Run();
```

**Key Pattern**: The AppHost uses a fluent builder API to:

1. Define resources (containers, projects, databases, etc.)
2. Establish dependencies between resources
3. Configure service discovery
4. Set up environment variables and connection strings

---

## 2. Resource Creation Patterns

### 2.1 Container Resources (Redis)

```csharp
var redisPassword = builder.AddParameter("Redis-Password", secret: true);
var redis = builder
    .AddRedis("redis", password: redisPassword)
    .WithHostPort(6380)      // Custom port mapping
    .WithLifetime(ContainerLifetime.Persistent)  // Data persists across restarts
    .WithRedisInsight();     // Adds Redis management UI
```

**Learnings**:

- **`AddParameter()`**: Creates externalized configuration (secrets, connection strings)
- **`WithHostPort()`**: Maps container port to host port
- **`WithLifetime()`**: `Persistent` keeps data, `Ephemeral` clears on restart
- **`.WithRedisInsight()`**: Adds optional tooling containers

### 2.2 Project Resources (Your .NET Projects)

```csharp
var imageApi = builder.AddProject<ImageApi>("imageapi")
    .WithReference(openai)
    .WithEnvironment("COMPUTERVISION__ENDPOINT", ocrEndpoint)
    .WithDaprSidecar(opt => { /* ... */ })
    .WaitFor(imageApi);
```

**Learnings**:

- **`AddProject<T>()`**: References a project by type (uses generated `Projects` class)
- **`"imageapi"`**: Service name used for service discovery
- **`.WithReference()`**: Injects connection strings/service discovery info
- **`.WithEnvironment()`**: Sets environment variables (uses double underscore `__` for nested config)
- **`.WaitFor()`**: Ensures dependency starts first

**Service Discovery**: When `imageApi` is referenced, Aspire automatically injects:

- Connection strings in format: `{ResourceName}__{EndpointName}__{Property}`
- Service discovery info accessible via `http://imageapi` (the service name)

### 2.3 Custom Resources (Ollama)

**Location**: `src/Ollama.Hosting/`

This app demonstrates creating a **custom Aspire resource** for Ollama (local LLM):

```csharp
public static IResourceBuilder<OllamaResource> AddOllama(
    this IDistributedApplicationBuilder builder,
    string name = "Ollama", 
    int? port = null, 
    string modelName = DefaultModelName, 
    bool useNvidiaGpu = false)
{
    var ollama = new OllamaResource(name, modelName);
    IResourceBuilder<OllamaResource> resourceBuilder = builder.AddResource(ollama)
        .WithAnnotation(new ContainerImageAnnotation { Image = "ollama/ollama", Tag = "0.3.9" })
        .WithVolume(ModelVolumeName, ModelVolumePath)  // Persistent storage for models
        .WithHttpEndpoint(port, 11434, OllamaResource.OllamaEndpointName)
        .WithLifetime(ContainerLifetime.Persistent)
        .ExcludeFromManifest()  // Don't include in deployment manifests
        .PublishAsContainer();
    
    return resourceBuilder;
}
```

**Key Custom Resource Concepts**:

1. **Inherit from `ContainerResource`**: Base class for container-based resources
2. **Implement `IResourceWithEndpoints`**: Exposes HTTP/gRPC endpoints
3. **Create `EndpointReference`**: Allows other resources to reference this endpoint
4. **Use annotations**: `ContainerImageAnnotation` specifies Docker image
5. **Volumes**: `WithVolume()` for persistent data (model storage)

**Usage in AppHost**:

```csharp
var localQuestionGenerator = runLocalOllama
    ? builder.AddOllama("questiongenerator", modelName: ModelNames.Llama3_1b, port: 11434, useNvidiaGpu: false)
    : null;

var imageApi = builder.AddProject<ImageApi>("imageapi")
    .WithOptionalReference(localQuestionGenerator?.Resource.Endpoint);  // Conditional reference
```

---

## 3. Resource Dependencies & Ordering

Aspire uses **dependency chains** to determine startup order:

```csharp
var imageApiStateStore = builder.AddDaprStateStore("imageapistatestorecomponent", ...)
    .WaitFor(redis);  // State store waits for Redis

var imageApi = builder.AddProject<ImageApi>("imageapi")
    .WithDaprSidecar(opt => {
        opt.WithReference(imageApiStateStore);
        opt.WaitFor(redis);  // Dapr sidecar waits for Redis
    })
    .WaitFor(imageApi);  // Frontend waits for ImageApi
```

**Pattern**: Chain dependencies using `.WaitFor()` to ensure:

- Redis starts before Dapr state store
- Dapr state store is ready before ImageApi
- ImageApi is ready before Frontend

---

## 4. Dapr Integration Pattern

**Location**: `src/AskVantage/Aspire/AskVantage.AppHost/Program.cs` (lines 33-37, 58-69)

Aspire has built-in Dapr support via `CommunityToolkit.Aspire.Hosting.Dapr`:

```csharp
// 1. Define Dapr component (references YAML file)
string daprComponentsPath = Path.GetFullPath(Path.Combine(AppContext.BaseDirectory, "..", "..", "..", "DaprComponents"));
var imageApiStateStore = builder.AddDaprStateStore("imageapistatestorecomponent", new DaprComponentOptions
{
    LocalPath = $"{daprComponentsPath}/statestore.yaml"  // Points to Dapr component YAML
}).WaitFor(redis);

// 2. Attach Dapr sidecar to a project
var imageApi = builder.AddProject<ImageApi>("imageapi")
    .WithDaprSidecar(opt =>
    {
        opt.WithOptions(new DaprSidecarOptions
        {
            AppId = "imageapi",  // Dapr application ID
            SchedulerHostAddress = "",  // Disable Dapr scheduler
            PlacementHostAddress = "",  // Disable Dapr placement
            ResourcesPaths = ImmutableHashSet.Create(daprComponentsPath)  // Where to find component YAMLs
        });
        opt.WithReference(imageApiStateStore);  // Link state store to sidecar
        opt.WaitFor(redis);
    });
```

**Dapr Component YAML** (`DaprComponents/statestore.yaml`):

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: imageapistatestorecomponent
spec:
  type: state.redis
  metadata:
  - name: redisHost
    value: localhost:6380
  - name: redisPassword
    value: "S3cr3tPassw0rd!"
```

**Learnings**:

- Dapr sidecar is automatically injected into the container/process
- Component YAML files define Dapr building blocks (state stores, pub/sub, etc.)
- `AppId` must match what your app uses to identify itself to Dapr
- Resources paths tell Dapr where to find component definitions

---

## 5. Service Discovery Pattern

Aspire automatically handles **service discovery** - services find each other by name, not hardcoded URLs.

**In AppHost**:

```csharp
var imageApi = builder.AddProject<ImageApi>("imageapi");  // Service name: "imageapi"
var frontend = builder.AddProject<AskVantage_Frontend>("AskVantageFrontend")
    .WithReference(imageApi);  // Injects service discovery info
```

**In Service Code** (`src/AskVantage/AskVantage.Frontend/Program.cs`):

```csharp
// Forward calls to image API using service name
app.MapForwarder("/api/image/{**catch-all}", "http://imageapi", "/api/image/{**catch-all}");
app.MapForwarder("/api/question/{**catch-all}", "http://imageapi", "/api/question/{**catch-all}");
```

**How It Works**:

1. Aspire injects service discovery configuration into each service
2. Services reference each other by name (`"http://imageapi"`)
3. At runtime, Aspire resolves the name to actual endpoint (localhost in dev, service mesh in production)
4. No hardcoded URLs needed!

**Service Defaults** (`src/AskVantage/Aspire/AskVantage.ServiceDefaults/Extensions.cs`):

```csharp
builder.Services.ConfigureHttpClientDefaults(http =>
{
    http.AddStandardResilienceHandler(/* retry, circuit breaker, etc. */);
    http.AddServiceDiscovery();  // Enables automatic service discovery
});
```

---

## 6. Configuration & Secrets Pattern

### 6.1 External Parameters (Secrets)

```csharp
var redisPassword = builder.AddParameter("Redis-Password", secret: true);
var openAiApiKey = builder.AddParameter("OpenAiApiKey", secret: true,
    valueGetter: () => runLocalOllama 
        ? builder.Configuration["OpenAILocal:ApiKey"]! 
        : builder.Configuration["OpenAIApiKey"]!);
```

**Learnings**:

- **`AddParameter()`**: Creates externalized configuration
- **`secret: true`**: Marks as sensitive (hidden in logs, prompts for value)
- **`valueGetter`**: Lambda to compute value (supports conditional logic)
- Values come from: user secrets, appsettings, environment variables, or user input

### 6.2 Connection Strings

```csharp
var openai = builder.AddConnectionString("openAiConnection");
// Later...
var imageApi = builder.AddProject<ImageApi>("imageapi")
    .WithReference(openai);  // Injects as connection string
```

**In Service Code**: Access via `IConfiguration["ConnectionStrings:openAiConnection"]`

### 6.3 Environment Variables

```csharp
var imageApi = builder.AddProject<ImageApi>("imageapi")
    .WithEnvironment("COMPUTERVISION__ENDPOINT", ocrEndpoint)
    .WithEnvironment("COMPUTERVISION__APIKEY", ocrApiKey)
    .WithEnvironment("OPENAI__ENDPOINT", openAiEndpoint)
    .WithEnvironment("OPENAI__APIKEY", openAiApiKey);
```

**Learnings**:

- Double underscore `__` represents nested configuration: `COMPUTERVISION__ENDPOINT` → `ComputerVision:Endpoint`
- Use `IResourceBuilder<T>.WithEnvironment()` for project resources
- Use `IResourceBuilder<T>.WithEnvironment(Action<EnvironmentCallbackContext>)` for complex scenarios

---

## 7. Service Defaults Pattern

**Location**: `src/AskVantage/Aspire/AskVantage.ServiceDefaults/Extensions.cs`

**Pattern**: Create a shared project that all services reference to get common Aspire features.

```csharp
public static IHostApplicationBuilder AddServiceDefaults(this IHostApplicationBuilder builder)
{
    builder.ConfigureOpenTelemetry();      // Observability
    builder.AddDefaultHealthChecks();      // Health endpoints
    builder.Services.AddServiceDiscovery(); // Service discovery
    
    builder.Services.ConfigureHttpClientDefaults(http =>
    {
        http.AddStandardResilienceHandler(opt => { /* retry, circuit breaker */ });
        http.AddServiceDiscovery();
    });
    
    return builder;
}
```

**Usage in Services**:

```csharp
// In ImageApi/Program.cs and Frontend/Program.cs
builder.AddServiceDefaults();
```

**Benefits**:

- **DRY**: Define once, use everywhere
- **Consistency**: All services get same telemetry, health checks, resilience
- **Easy updates**: Change in one place, affects all services

---

## 8. Eventing & Lifecycle Hooks Pattern

**Location**: `src/Ollama.Hosting/OllamaResourceEventSubscriber.cs`

Aspire supports **eventing** for custom resource initialization:

```csharp
public class OllamaResourceEventSubscriber : IDistributedApplicationEventingSubscriber
{
    public Task SubscribeAsync(IDistributedApplicationEventing eventing, ...)
    {
        eventing.Subscribe<AfterResourcesCreatedEvent>(OnAfterResourcesCreatedAsync);
        return Task.CompletedTask;
    }
    
    private Task OnAfterResourcesCreatedAsync(AfterResourcesCreatedEvent @event, ...)
    {
        foreach (var resource in @event.Model.Resources.OfType<OllamaResource>())
        {
            DownloadModelInBackground(resource, cancellationToken);
        }
        return Task.CompletedTask;
    }
}
```

**Registration**:

```csharp
builder.Services.TryAddEventingSubscriber<OllamaResourceEventSubscriber>();
```

**Use Cases**:

- Download models after container starts
- Initialize databases with seed data
- Run migrations
- Update resource state in dashboard

**Available Events**:

- `AfterResourcesCreatedEvent`: After all resources are created
- `BeforeStartAsyncEvent`: Before resources start
- `AfterStartAsyncEvent`: After resources start

---

## 9. Conditional Resources Pattern

```csharp
bool runLocalOllama = builder.Configuration.GetValue<bool>("OpenAILocal:RunLocal");

var localQuestionGenerator = runLocalOllama
    ? builder.AddOllama("questiongenerator", modelName: ModelNames.Llama3_1b, port: 11434, useNvidiaGpu: false)
    : null;

var imageApi = builder.AddProject<ImageApi>("imageapi")
    .WithOptionalReference(localQuestionGenerator?.Resource.Endpoint);  // Only if not null
```

**Pattern**: Use conditional logic to:

- Enable/disable features based on configuration
- Switch between local and cloud resources
- Support different deployment scenarios

**Custom Extension** (`WithOptionalReference`):

```csharp
public static IResourceBuilder<TDestination> WithOptionalReference<TDestination>(
    this IResourceBuilder<TDestination> builder, 
    EndpointReference? endpointReference)
    where TDestination : IResourceWithEnvironment
{
    if (endpointReference != null)
    {
        builder.WithReference(endpointReference);
    }
    return builder;
}
```

---

## 10. Dashboard Configuration Pattern

```csharp
builder.Configuration.AddInMemoryCollection(new Dictionary<string, string?>
{
    ["ASPNETCORE_URLS"] = "http://localhost:18888",
    ["ASPIRE_ALLOW_UNSECURED_TRANSPORT"] = "True",
    ["DOTNET_DASHBOARD_OTLP_ENDPOINT_URL"] = "http://localhost:18889",
    ["ASPIRE_DASHBOARD_UNSECURED_ALLOW_ANONYMOUS"] = "true"
});
```

**Learnings**:

- Configure Aspire dashboard (observability UI) via configuration
- Useful for Codespaces/remote development
- `ASPIRE_ALLOW_UNSECURED_TRANSPORT`: Allows HTTP (not HTTPS) for dashboard
- `ASPIRE_DASHBOARD_UNSECURED_ALLOW_ANONYMOUS`: Allows anonymous access

---

## 11. Project Reference Pattern

Aspire uses **source generation** to create a `Projects` class that contains project references:

```csharp
using Projects;  // Generated namespace

var imageApi = builder.AddProject<ImageApi>("imageapi");
var frontend = builder.AddProject<AskVantage_Frontend>("AskVantageFrontend");
```

**How It Works**:

1. Aspire scans your solution for `.csproj` files
2. Generates a `Projects` class with strongly-typed project references
3. `AddProject<T>()` uses these types to find the project
4. The string parameter (`"imageapi"`) is the service name for discovery

**Benefits**:

- Compile-time safety (can't reference non-existent projects)
- IntelliSense support
- Refactoring-friendly

---

## 12. Health Checks Pattern

**In Service Defaults**:

```csharp
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy(), ["live"]);

// In Program.cs
app.MapDefaultEndpoints();  // Maps /health and /alive
```

**Endpoints**:

- `/health`: All health checks must pass (readiness)
- `/alive`: Only checks tagged with "live" must pass (liveness)

**Aspire Integration**: Aspire dashboard automatically shows health status for all resources.

---

## 13. OpenTelemetry Pattern

**In Service Defaults**:

```csharp
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics => {
        metrics.AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddRuntimeInstrumentation();
    })
    .WithTracing(tracing => {
        tracing.AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation();
    });
```

**Learnings**:

- Aspire automatically collects telemetry from all services
- Metrics: Request rates, durations, errors
- Traces: Distributed request tracing across services
- Logs: Structured logging with correlation IDs
- Dashboard: Visualizes all telemetry in one place

---

## 14. Resilience Pattern

**In Service Defaults**:

```csharp
builder.Services.ConfigureHttpClientDefaults(http =>
{
    http.AddStandardResilienceHandler(opt =>
    {
        opt.RateLimiter.DefaultRateLimiterOptions.PermitLimit = 1;
        opt.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(240);
        opt.AttemptTimeout.Timeout = TimeSpan.FromSeconds(120);
        opt.CircuitBreaker.SamplingDuration = TimeSpan.FromSeconds(360);
    });
});
```

**Features**:

- **Retry**: Automatic retries on transient failures
- **Circuit Breaker**: Stops calling failing services
- **Timeout**: Prevents hanging requests
- **Rate Limiting**: Prevents overwhelming services

**Applied Automatically**: All `HttpClient` instances get resilience by default.

---

## 15. Deployment Environments

```csharp
builder.AddDockerComposeEnvironment("compose");
```

**Pattern**: Aspire supports multiple deployment targets:

- **Local**: Runs containers/projects directly
- **Docker Compose**: Generates `docker-compose.yml`
- **Kubernetes**: Generates manifests (future)
- **Azure Container Apps**: Deploys to Azure (future)

**Learnings**:

- Same AppHost code works for all environments
- Aspire handles environment-specific transformations
- No need to write separate deployment configs

---

## Quick Reference: Common Aspire Patterns

| Pattern | Code Example | Use Case |
| ------- | ------------ | -------- |
| Add Container | `builder.AddRedis("redis")` | Add infrastructure containers |
| Add Project | `builder.AddProject<MyApi>("myapi")` | Add your .NET projects |
| Add Parameter | `builder.AddParameter("Key", secret: true)` | Externalized secrets/config |
| Add Connection String | `builder.AddConnectionString("db")` | Database connection strings |
| Reference Resource | `.WithReference(otherResource)` | Inject service discovery |
| Set Environment | `.WithEnvironment("KEY", value)` | Set env vars |
| Wait For | `.WaitFor(dependency)` | Control startup order |
| Dapr Sidecar | `.WithDaprSidecar(opt => {...})` | Attach Dapr to project |
| Service Defaults | `builder.AddServiceDefaults()` | Add telemetry, health, discovery |
| Custom Resource | `builder.AddResource(new MyResource())` | Create custom resources |
| Eventing | `eventing.Subscribe<Event>(handler)` | Lifecycle hooks |

---

## Additional Resources

### Aspire Documentation

- **Aspire Homepage**: <https://aspire.dev/docs/> - Official Aspire documentation
- **What is Aspire?**: <https://aspire.dev/docs/fundamentals/what-is-aspire> - Core concepts
- **The AppHost**: <https://aspire.dev/docs/fundamentals/what-is-the-apphost> - AppHost pattern
- **Understanding Resources**: <https://aspire.dev/docs/fundamentals/understanding-resources> - Resource model
- **Service Discovery**: <https://aspire.dev/docs/fundamentals/service-discovery> - How services find each other
- **Service Defaults**: <https://aspire.dev/docs/fundamentals/service-defaults> - Shared configuration pattern
- **Health Checks**: <https://aspire.dev/docs/fundamentals/health-checks> - Health monitoring
- **Telemetry**: <https://aspire.dev/docs/fundamentals/telemetry> - Observability
- **Networking Overview**: <https://aspire.dev/docs/fundamentals/networking-overview> - Network configuration
- **Testing**: <https://aspire.dev/docs/testing/overview> - Testing Aspire applications

### Related Patterns

- **Aspire Resource Model**: Declarative infrastructure as code
- **Aspire Service Discovery**: Microservices communication pattern
- **Aspire Service Defaults**: Shared configuration pattern
- **Aspire Eventing**: Lifecycle hooks and custom initialization

---

*Last Updated: [Current Date]*
*Based on: AskVantage codebase, .NET 10.0, Aspire*
