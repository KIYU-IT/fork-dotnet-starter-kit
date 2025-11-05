# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

FullStackHero .NET 9 Starter Kit - A Clean Architecture solution with ASP.NET Core Web API, Blazor Client, and .NET Aspire orchestration. Features multi-tenancy, modular architecture, CQRS with MediatR, and comprehensive identity management.

**Tech Stack**: .NET 9, Entity Framework Core 9, Blazor, PostgreSQL, Redis, MediatR, FluentValidation, Hangfire, Carter (Minimal APIs), Finbuckle.MultiTenant

## Development Commands

### Running the Application

**Using .NET Aspire (Recommended)**:
```bash
# From repository root, open the solution
cd src
# Set Aspire Host as startup project and run from Visual Studio
# OR use CLI:
cd aspire/Host
dotnet run
```

Access points:
- Aspire Dashboard: `https://localhost:7200/`
- API (Swagger): `https://localhost:7000/swagger/index.html`
- Blazor: `https://localhost:7100/`

**Individual Projects**:
```bash
# API only
cd src/api/server
dotnet run

# Blazor only
cd src/apps/blazor/client
dotnet run
```

### Database Configuration

**Connection String**: Configure in `src/api/server/appsettings.Development.json`:
```json
{
  "DatabaseOptions": {
    "Provider": "postgresql",
    "ConnectionString": "Server=localhost;Port=5432;Database=fullstackhero;User Id=postgres;Password=yourpassword"
  }
}
```

Supported providers: `postgresql`, `mssql`

### Entity Framework Migrations

**All migration commands must be run from `src/api/server` directory**:

```bash
cd src/api/server

# Add new migrations (select appropriate context and migration project)
dotnet ef migrations add "Migration Name" --project ../../migrations/PostgreSQL/ --context IdentityDbContext -o Identity
dotnet ef migrations add "Migration Name" --project ../../migrations/PostgreSQL/ --context TenantDbContext -o Tenant
dotnet ef migrations add "Migration Name" --project ../../migrations/PostgreSQL/ --context TodoDbContext -o Todo
dotnet ef migrations add "Migration Name" --project ../../migrations/PostgreSQL/ --context CatalogDbContext -o Catalog

# For SQL Server migrations
dotnet ef migrations add "Migration Name" --project ../../migrations/MSSQL/ --context [Context] -o [OutputFolder]

# Update database
dotnet ef database update --context [ContextName]

# Remove last migration
dotnet ef migrations remove --project ../../migrations/PostgreSQL/ --context [Context]
```

**Available DbContexts**:
- `IdentityDbContext` - User authentication and authorization
- `TenantDbContext` - Multi-tenancy management
- `TodoDbContext` - Todo module
- `CatalogDbContext` - Catalog module (Products and Brands)

### Building

```bash
# Build entire solution
dotnet build src/FSH.Starter.sln

# Build specific project
dotnet build src/api/server/Server.csproj
```

### Testing

```bash
# Run all tests (when available)
dotnet test src/FSH.Starter.sln
```

## Architecture Overview

### Modular Monolith Structure

```
src/
├── api/
│   ├── framework/          # Shared framework components
│   │   ├── Core/          # Domain abstractions, interfaces, DTOs
│   │   └── Infrastructure/ # Framework implementations (Auth, Caching, Persistence, etc.)
│   ├── modules/            # Feature modules (self-contained)
│   │   ├── Catalog/       # Product catalog (Domain, Application, Infrastructure)
│   │   └── Todo/          # Todo feature (simplified vertical slice)
│   ├── migrations/         # EF Core migrations per database provider
│   │   ├── PostgreSQL/
│   │   └── MSSQL/
│   └── server/            # API host/startup project
├── apps/
│   └── blazor/
│       ├── client/        # Blazor WebAssembly UI
│       ├── infrastructure/ # Blazor services, API client, auth
│       └── shared/        # Shared Blazor components/models
├── aspire/
│   ├── Host/              # .NET Aspire orchestration
│   └── service-defaults/   # OpenTelemetry, health checks
└── Shared/                 # Cross-cutting shared code (Authorization constants)
```

### Framework Layer (`api/framework/`)

**Core** (`api/framework/Core/`):
- Domain base classes: `BaseEntity`, `AuditableEntity`, `DomainEvent`
- Abstractions: `IRepository`, `ICurrentUser`, `ITokenService`, `IRoleService`, `ITenantService`
- Cross-cutting concerns: Caching, Paging, Specifications (Ardalis.Specification)
- Identity features: Users, Roles, Permissions, Tokens (JWT)
- Multi-tenancy abstractions

**Infrastructure** (`api/framework/Infrastructure/`):
- Framework implementations for all Core abstractions
- Identity: ASP.NET Core Identity with custom `FshUser`, `FshRole`, `IdentityDbContext`
- Multi-tenancy: Finbuckle.MultiTenant integration with `TenantDbContext`
- Persistence: `FshDbContext` base class with multi-tenancy and audit support
- Authentication: JWT Bearer with custom claims and permissions
- Authorization: Permission-based policy handlers
- OpenAPI/Swagger configuration with versioning
- Distributed caching (Redis)
- Background jobs (Hangfire)
- Email (MailKit)
- Health checks, CORS, rate limiting, security headers

### Module Architecture

Each module is self-contained with its own:

**Option 1: Full Clean Architecture** (Catalog module):
```
Catalog/
├── Catalog.Domain/         # Entities, domain events, exceptions
├── Catalog.Application/    # Use cases (CQRS), DTOs, validators
└── Catalog.Infrastructure/ # Persistence, endpoints (Carter), module registration
```

**Option 2: Vertical Slice** (Todo module):
```
Todo/
├── Domain/                 # Entities and events
├── Features/               # Organized by feature (Create, Update, Delete, Get)
│   └── [Feature]/v1/      # Versioned commands, handlers, validators, endpoints
├── Persistence/            # DbContext, repository
└── TodoModule.cs           # Module registration
```

### Module Registration Pattern

All modules follow this pattern:

1. **Define endpoints** using Carter's `CarterModule`:
```csharp
public class Endpoints : CarterModule
{
    public override void AddRoutes(IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("resource").WithTags("tag");
        group.MapEndpoint();
    }
}
```

2. **Register services** in `RegisterXServices(WebApplicationBuilder)`:
```csharp
builder.Services.BindDbContext<ModuleDbContext>();
builder.Services.AddScoped<IDbInitializer, ModuleDbInitializer>();
builder.Services.AddKeyedScoped<IRepository<Entity>, Repository<Entity>>("key");
```

3. **Register in host** (`api/server/Extensions.cs`):
```csharp
public static WebApplicationBuilder RegisterModules(this WebApplicationBuilder builder)
{
    var assemblies = new Assembly[] { typeof(Module).Assembly };
    builder.Services.AddValidatorsFromAssemblies(assemblies);
    builder.Services.AddMediatR(cfg => cfg.RegisterServicesFromAssemblies(assemblies));
    builder.RegisterModuleServices();
    builder.Services.AddCarter(config => config.WithModule<Module.Endpoints>());
    return builder;
}
```

### CQRS Pattern with MediatR

All features use CQRS:
- **Commands**: Create, Update, Delete operations return response DTOs
- **Queries**: Get, Search operations with pagination support
- **Validators**: FluentValidation for command/query validation
- **Handlers**: `IRequestHandler<TRequest, TResponse>` implementations

Example flow:
```
Endpoint → MediatR Command/Query → Validator → Handler → Repository → Response
```

### Multi-Tenancy

Built-in support via Finbuckle.MultiTenant:
- Tenant resolution from headers, claims, or host
- Per-tenant database isolation available
- `TenantDbContext` manages tenant metadata
- Automatic tenant filtering in queries via `FshDbContext`

### Authentication & Authorization

**JWT-based authentication**:
- Token generation endpoint: `POST /api/v1/tokens`
- Refresh token support
- Custom claims: UserId, Email, Tenant, FullName, etc.

**Permission-based authorization**:
- Permissions defined in `FshPermissions` (e.g., `Permissions.Catalog.View`)
- Role-permission mapping stored in `FshRoleClaim`
- Custom authorization handler: `RequiredPermissionAuthorizationHandler`
- Apply via `[RequiredPermission("Resource", "Action")]` attribute or `.RequirePermission()` on endpoints

### API Versioning

Using `Asp.Versioning`:
- Routes: `/api/v{version:apiVersion}/[module]/[resource]`
- Versions: v1, v2 (configured in `Extensions.cs`)
- Group endpoints by version in module registration

### .NET Aspire Integration

**Aspire Host** (`aspire/Host/Program.cs`):
- Orchestrates all services (API, Blazor, PostgreSQL, Grafana, Prometheus)
- Service discovery and health checks
- Observability with OpenTelemetry (traces, metrics, logs)

**ServiceDefaults** project:
- Shared telemetry configuration
- Health check endpoints
- Resilience policies

## Key Patterns and Conventions

### Entity Framework

- **Base classes**: All entities inherit from `BaseEntity` (has `Id`, implements `IEntity`)
- **Audit trails**: Use `AuditableEntity` for `CreatedBy`, `CreatedOn`, `LastModifiedBy`, `LastModifiedOn`
- **Soft delete**: Implement `ISoftDeletable` for `DeletedOn`, `DeletedBy`
- **Multi-tenancy**: `FshDbContext` automatically applies tenant filtering
- **Repository pattern**: Use `IRepository<T>` and `IReadRepository<T>` (Ardalis.Specification)

### Domain Events

- Inherit from `DomainEvent` (implements `IDomainEvent`)
- Raised via `entity.RegisterDomainEvent(event)`
- Handled by `INotificationHandler<TEvent>` implementations
- Published automatically by `FshDbContext` on `SaveChangesAsync`

### Specification Pattern

Use Ardalis.Specification for complex queries:
```csharp
public class GetProductByIdSpec : Specification<Product>
{
    public GetProductByIdSpec(Guid id) => Query.Where(p => p.Id == id).Include(p => p.Brand);
}
```

### Validation

- Use FluentValidation for all commands/queries
- Automatic validation via `ValidationBehavior` MediatR pipeline
- Common validators in framework Core layer

### Exception Handling

Custom exceptions in `api/framework/Core/Exceptions/`:
- `NotFoundException` - Returns 404
- `UnauthorizedException` - Returns 401
- `ForbiddenException` - Returns 403
- Global exception handler: `CustomExceptionHandler` in Infrastructure

### Adding New Modules

1. Create module structure in `src/api/modules/[ModuleName]/`
2. Define domain entities, events, and business logic
3. Create features with commands/queries, handlers, and validators
4. Implement `DbContext` inheriting from `FshDbContext`
5. Create Carter endpoints in `[Module]Module.cs`
6. Register services via `Register[Module]Services()`
7. Add migrations from `src/api/server/` directory
8. Register module in `api/server/Extensions.cs`

### Configuration

- `appsettings.json` - Production settings
- `appsettings.Development.json` - Local development overrides
- Configuration sections: `DatabaseOptions`, `JwtOptions`, `CacheOptions`, `HangfireOptions`, `MailOptions`, `CorsOptions`

## Important Notes

- **Prerequisites**: .NET 9 SDK, PostgreSQL (or SQL Server), Docker Desktop (for Aspire), Redis (optional for caching)
- **Status**: Work in progress, NuGet package not yet available
- **Default credentials** (development only): See `appsettings.Development.json` for JWT keys, Hangfire credentials
- **Multi-tenancy**: Enabled by default, tenant resolution via headers or authentication
- **Observability**: OpenTelemetry configured for traces, metrics, and logs (Aspire dashboard)
