# Best Practices for Microservices with .NET

## Overview

Microservices architecture involves decomposing an application into small, independently deployable services that communicate over well-defined APIs. .NET provides a rich ecosystem for building, deploying, and managing microservices. This document outlines best practices to follow when building microservices with .NET.

---

## 1. Design Principles

### Single Responsibility
Each microservice should own a single bounded context and expose only the functionality relevant to that context. Avoid the temptation to grow a service beyond its intended scope.

### Loose Coupling
Services should be independent of each other's internal implementation. Communicate through stable, versioned APIs (REST, gRPC, or message queues) rather than sharing databases or internal libraries.

### High Cohesion
Keep related functionality together within a service. A well-cohesive service is easier to understand, test, and maintain.

### Design for Failure
Assume that any downstream service or infrastructure component can fail. Use patterns like Circuit Breaker, Retry, and Bulkhead to build resilient services.

---

## 2. Project Structure

Organise each microservice as an independent .NET solution (or project) with its own:
- Source code
- Unit and integration tests
- Dockerfile
- CI/CD pipeline definition

A recommended folder layout for a single service:

```
src/
  MyService/
    Controllers/
    Models/
    Services/
    Data/
    Program.cs
    appsettings.json
tests/
  MyService.UnitTests/
  MyService.IntegrationTests/
Dockerfile
```

---

## 3. API Design

- **Use RESTful conventions** (or gRPC for performance-critical internal communication).
- **Version your APIs** (e.g., `/api/v1/orders`) to allow backward-compatible evolution.
- **Return standard HTTP status codes** that accurately represent the outcome of each operation.
- **Use OpenAPI / Swagger** to document APIs. In .NET, add Swashbuckle or NSwag:

```csharp
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
```

- **Validate input** using Data Annotations or FluentValidation to reject malformed requests early.

---

## 4. Configuration Management

- Store configuration in environment variables or a secrets manager (e.g., Azure Key Vault, AWS Secrets Manager) rather than hard-coding values.
- Use the .NET `IConfiguration` abstraction so the source of configuration is transparent to the application.
- Apply the **Options pattern** (`IOptions<T>`) for strongly typed settings:

```csharp
builder.Services.Configure<DatabaseOptions>(
    builder.Configuration.GetSection("Database"));
```

- Never commit secrets to source control. Use **User Secrets** (`dotnet user-secrets`) for local development and `.gitignore` to exclude files such as `appsettings.Local.json` or `secrets.json` that may contain sensitive values. `appsettings.Development.json` is typically committed as it holds non-sensitive defaults for local development.

---

## 5. Data Management

- **Database per service**: each microservice should own its own data store. Sharing a database between services creates tight coupling.
- Prefer lightweight databases (e.g., PostgreSQL, SQL Server, Cosmos DB) that match the service's data access patterns.
- Use **EF Core migrations** to manage schema changes in a repeatable, version-controlled way.
- Implement the **Repository pattern** to abstract data access logic and improve testability.

---

## 6. Communication Patterns

### Synchronous (HTTP / gRPC)
Use for request/response interactions where an immediate answer is required. Prefer gRPC for internal service-to-service calls that demand low latency and strong contracts.

### Asynchronous (Message Queues / Event Bus)
Use for decoupled, event-driven workflows. Suitable libraries and platforms include:
- **Azure Service Bus** or **RabbitMQ** with MassTransit
- **Apache Kafka** with Confluent.Kafka or MassTransit Kafka rider

Example of publishing an event with MassTransit:

```csharp
await publishEndpoint.Publish(new OrderPlaced { OrderId = order.Id });
```

### Saga Pattern
Manage long-running distributed transactions using sagas (orchestration or choreography) to maintain data consistency without distributed locks.

---

## 7. Resilience

Use **Microsoft.Extensions.Http.Resilience** (or Polly) to add retry, timeout, circuit-breaker, and fallback policies to outbound HTTP calls:

```csharp
builder.Services.AddHttpClient<IOrderServiceClient, OrderServiceClient>()
    .AddStandardResilienceHandler();
```

Key policies to consider:
| Policy | Purpose |
|---|---|
| Retry | Automatically retry transient failures |
| Circuit Breaker | Stop calling a failing service temporarily |
| Timeout | Prevent indefinitely hanging calls |
| Bulkhead Isolation | Limit concurrent calls to protect resources |

---

## 8. Observability

### Structured Logging
Use `Microsoft.Extensions.Logging` with a provider such as Serilog or OpenTelemetry to emit structured, searchable logs:

```csharp
Log.Logger = new LoggerConfiguration()
    .WriteTo.Console(new JsonFormatter())
    .CreateLogger();
```

Always include a correlation/trace ID in log entries to trace a request across multiple services.

### Distributed Tracing
Use **OpenTelemetry** to capture traces across service boundaries:

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddOtlpExporter());
```

### Health Checks
Expose `/health` and `/ready` endpoints using `Microsoft.AspNetCore.Diagnostics.HealthChecks`:

```csharp
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>()
    .AddUrlGroup(new Uri("https://dependency/health"), "dependency");

app.MapHealthChecks("/health");
```

### Metrics
Emit application metrics (request rate, error rate, latency) using OpenTelemetry Metrics or Prometheus exporters for dashboards and alerting.

---

## 9. Security

- **Authenticate** inter-service calls using OAuth 2.0 / OpenID Connect (e.g., Duende IdentityServer, Azure AD).
- **Authorise** using role or policy-based authorization via `IAuthorizationService`.
- **Validate tokens** in every service; do not rely solely on a gateway.
- **Encrypt data in transit** — enforce HTTPS and use TLS for all communications.
- **Scan container images** for known vulnerabilities as part of CI/CD.
- Apply the **principle of least privilege** to service identities and database accounts.

---

## 10. Containerisation and Deployment

- Package each service as a **Docker container**. Keep images small by using multi-stage builds:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /app

FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /app .
ENTRYPOINT ["dotnet", "MyService.dll"]
```

- Use **Kubernetes** (AKS, EKS, GKE) or **Azure Container Apps** to orchestrate and scale services.
- Define resource requests and limits for every container.
- Use **Helm charts** or **Kustomize** to manage Kubernetes manifests across environments.

---

## 11. CI/CD

- Maintain a separate pipeline per service so changes can be built, tested, and deployed independently.
- Gate deployments on passing unit tests, integration tests, and static analysis (e.g., SonarCloud, Roslyn analyzers).
- Use **blue/green** or **canary deployments** to reduce deployment risk.
- Automate rollback on failed health checks or elevated error rates.

---

## 12. Testing Strategy

| Level | Scope | Tools |
|---|---|---|
| Unit | Individual classes/methods | xUnit, NUnit, Moq, FluentAssertions |
| Integration | Service + database/external APIs | `WebApplicationFactory`, Testcontainers |
| Contract | API compatibility between producer & consumer | Pact .NET |
| End-to-End | Full user journey across services | Playwright, Selenium |

Keep unit tests fast and isolated. Use integration tests to verify infrastructure boundaries. Run contract tests in CI to catch API-breaking changes early.

---

## Further Reading

- [.NET Microservices Architecture Guide (Microsoft)](https://learn.microsoft.com/dotnet/architecture/microservices/)
- [Dapr for .NET Developers](https://learn.microsoft.com/dotnet/architecture/dapr-for-net-developers/)
- [OpenTelemetry .NET](https://opentelemetry.io/docs/languages/net/)
- [MassTransit Documentation](https://masstransit.io/)
- [Polly Resilience Library](https://www.pollydocs.org/)
