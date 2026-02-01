---
name: fast-endpoints
description: |
  Build performant REST APIs with FastEndpoints in ASP.NET Core. Use when: (1) Creating new API endpoints with request/response DTOs, (2) Implementing REPR pattern (Request-Endpoint-Response), (3) Adding validation with FluentValidation, (4) Configuring authentication/authorization (JWT, cookies, policies), (5) Setting up Swagger/OpenAPI documentation, (6) Implementing pre/post processors, (7) Working with command/event bus patterns, (8) Handling file uploads, (9) Creating domain entity mappers. FastEndpoints provides minimal-API performance with controller-like organization.
---

# FastEndpoints

High-performance REST API framework for ASP.NET Core using the REPR (Request-Endpoint-Response) pattern.

## Quick Start

```csharp
// Program.cs
var bld = WebApplication.CreateBuilder();
bld.Services.AddFastEndpoints();
var app = bld.Build();
app.UseFastEndpoints();
app.Run();
```

## Endpoint Structure

### Basic Endpoint

```csharp
public class CreateUserRequest
{
    public string Name { get; set; }
    public int Age { get; set; }
}

public class CreateUserResponse
{
    public int Id { get; set; }
    public string FullName { get; set; }
}

public class CreateUserEndpoint : Endpoint<CreateUserRequest, CreateUserResponse>
{
    public override void Configure()
    {
        Post("/api/users");
        AllowAnonymous();
    }

    public override async Task HandleAsync(CreateUserRequest req, CancellationToken ct)
    {
        await SendAsync(new CreateUserResponse
        {
            Id = 1,
            FullName = req.Name
        });
    }
}
```

### Endpoint Base Classes

| Base Class | Use Case |
|------------|----------|
| `Endpoint<TRequest>` | Request only |
| `Endpoint<TRequest, TResponse>` | Request + Response |
| `EndpointWithoutRequest` | No DTOs |
| `EndpointWithoutRequest<TResponse>` | Response only |

Fluent alternative: `Ep.Req<TRequest>.Res<TResponse>`

### Attribute-Based Configuration

```csharp
[HttpPost("/api/users")]
[Authorize(Roles = "Admin")]
[AllowAnonymous]
public class MyEndpoint : Endpoint<MyRequest, MyResponse>
{
    public override Task HandleAsync(MyRequest req, CancellationToken ct) { }
}
```

## Model Binding

Binding priority (highest to lowest): JSON Body → Form → Route → Query → Claims → Headers

### Binding Attributes

```csharp
public class MyRequest
{
    // Route parameter
    public int Id { get; set; }  // Matches route: /api/items/{Id}

    // Explicit binding sources
    [FromHeader] public string TenantId { get; set; }
    [FromClaim] public string UserId { get; set; }
    [QueryParam] public int Page { get; set; }
    [FromBody] public Address Address { get; set; }

    // Name mapping
    [BindFrom("customer_id")] public string CustomerId { get; set; }

    // Permission check
    [HasPermission("Admin")] public bool IsAdmin { get; set; }
}
```

### Manual Parameter Access

```csharp
public override Task HandleAsync(CancellationToken ct)
{
    var id = Route<int>("id");
    var page = Query<int>("page", isRequired: false);
}
```

## Validation

Uses FluentValidation (included automatically).

```csharp
public class CreateUserValidator : Validator<CreateUserRequest>
{
    public CreateUserValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Name is required")
            .MinimumLength(3).WithMessage("Name too short");

        RuleFor(x => x.Age)
            .InclusiveBetween(18, 120);
    }
}
```

### In-Handler Validation

```csharp
public override async Task HandleAsync(MyRequest req, CancellationToken ct)
{
    if (await UserExists(req.Email))
        AddError(r => r.Email, "Email already in use");

    ThrowIfAnyErrors();  // Sends 400 with errors

    // Or immediate abort:
    ThrowError("Something went wrong!");
}
```

### Disable Auto-Validation

```csharp
public override void Configure()
{
    DontThrowIfValidationFails();
}

public override Task HandleAsync(MyRequest req, CancellationToken ct)
{
    if (ValidationFailed)
    {
        // Handle manually using ValidationFailures
    }
}
```

## Sending Responses

### Option 1: Send Methods

```csharp
// Success responses
await SendAsync(response);                    // 200 OK with body
await SendAsync(response, statusCode);        // Custom status with body
await SendOkAsync(response);                  // Explicit 200 OK
await SendOkAsync();                          // 200 OK without body
await SendNoContentAsync();                   // 204 No Content
await SendCreatedAtAsync<GetEndpoint>(routeValues, response);  // 201 Created

// Error responses
await SendNotFoundAsync();                    // 404 Not Found
await SendNotFoundAsync(response);            // 404 with body
await SendUnauthorizedAsync();                // 401 Unauthorized
await SendForbiddenAsync();                   // 403 Forbidden
await SendErrorsAsync();                      // 400 with ValidationFailures
await SendErrorsAsync(statusCode);            // Custom status with errors

// Redirect
await SendRedirectAsync(url);                 // Redirect response

// File/Stream responses
await SendStreamAsync(stream, fileName, contentType);
await SendFileAsync(fileInfo);
await SendBytesAsync(bytes, fileName, contentType);
```

### Option 2: Response Property

Assign to `Response` property for automatic 200 OK:

```csharp
public override async Task HandleAsync(MyRequest req, CancellationToken ct)
{
    // Assign properties
    Response.FullName = req.FirstName + " " + req.LastName;
    Response.Age = req.Age;

    // Or assign new instance
    Response = new MyResponse
    {
        FullName = "john doe",
        Age = 30
    };
    // Response sent automatically at end of HandleAsync
}
```

### Option 3: Conditional Responses with Task<Void>

Change return type to `Task<Void>` to stop execution after sending:

```csharp
public override async Task<Void> HandleAsync(MyRequest req, CancellationToken ct)
{
    if (req.Id == 0)
        return await SendNotFoundAsync();  // Stops here

    if (!await UserExistsAsync(req.Id))
        return await SendNotFoundAsync("User not found");

    return await SendOkAsync(new MyResponse { Id = req.Id });
}
```

### Option 4: ExecuteAsync with Union Types

Override `ExecuteAsync` instead of `HandleAsync` for typed results:

```csharp
public class GetUserEndpoint : Endpoint<GetUserRequest, Results<Ok<UserResponse>, NotFound, ProblemDetails>>
{
    public override async Task<Results<Ok<UserResponse>, NotFound, ProblemDetails>> ExecuteAsync(
        GetUserRequest req, CancellationToken ct)
    {
        if (req.Id == 0)
            return TypedResults.NotFound();

        var user = await _db.GetUserAsync(req.Id);
        if (user == null)
            return TypedResults.NotFound();

        return TypedResults.Ok(new UserResponse { Id = user.Id, Name = user.Name });
    }
}
```

Common TypedResults: `Ok<T>`, `NotFound`, `BadRequest`, `NoContent`, `Created<T>`, `ProblemDetails`

### ExecuteAsync vs HandleAsync

| Method | Return Type | Use Case |
|--------|-------------|----------|
| `HandleAsync` | `Task` or `Task<Void>` | Use Send methods, set Response property |
| `ExecuteAsync` | `Task<TResponse>` or `Task<Results<...>>` | Return response directly, union types |

```csharp
// ExecuteAsync returning response directly
public override Task<UserResponse> ExecuteAsync(GetUserRequest req, CancellationToken ct)
{
    return Task.FromResult(new UserResponse { Id = req.Id });
}
```

## Dependency Injection

### Property Injection

```csharp
public class MyEndpoint : Endpoint<MyRequest>
{
    public IUserService UserService { get; set; }  // Auto-injected
}
```

### Constructor Injection

```csharp
public class MyEndpoint : Endpoint<MyRequest>
{
    private readonly IUserService _userService;

    public MyEndpoint(IUserService userService)
    {
        _userService = userService;
    }
}
```

### Manual Resolution

```csharp
var service = Resolve<IUserService>();           // Throws if not found
var service = TryResolve<IUserService>();        // Returns null if not found
```

### Pre-Resolved Services

- `Config` → `IConfiguration`
- `Env` → `IWebHostEnvironment`
- `Logger` → `ILogger`

## Security

### JWT Authentication

```csharp
// Setup
bld.Services
   .AddAuthenticationJwtBearer(s => s.SigningKey = "your-secret-key")
   .AddAuthorization();

app.UseAuthentication()
   .UseAuthorization()
   .UseFastEndpoints();

// Generate token
var token = JwtBearer.CreateToken(o =>
{
    o.SigningKey = "your-secret-key";
    o.ExpireAt = DateTime.UtcNow.AddHours(1);
    o.User.Roles.Add("Admin");
    o.User.Claims.Add(("UserId", "123"));
    o.User.Permissions.Add("Users.Create");
});
```

### Endpoint Authorization

```csharp
public override void Configure()
{
    Post("/api/admin/users");

    // Any of these
    Roles("Admin", "Manager");
    Claims("AdminId");
    Permissions("Users.Create");
    Scopes("api:write");
    Policies("AdminOnly");

    // All required
    RolesAll("Admin", "Manager");
    ClaimsAll("AdminId", "TenantId");
    PermissionsAll("Users.Create", "Users.Update");

    // Allow unauthenticated
    AllowAnonymous();

    // Specific auth scheme
    AuthSchemes("Bearer", "ApiKey");
}
```

### Cookie Authentication

```csharp
bld.Services.AddAuthenticationCookie(validFor: TimeSpan.FromMinutes(30));

// Sign in
await CookieAuth.SignInAsync(u =>
{
    u.Roles.Add("User");
    u.Claims.Add(new("Email", email));
});

// Sign out
await CookieAuth.SignOutAsync();
```

## Pre/Post Processors

### Pre-Processor

```csharp
public class TenantChecker : IPreProcessor<MyRequest>
{
    public Task PreProcessAsync(IPreProcessorContext<MyRequest> ctx, CancellationToken ct)
    {
        if (!ctx.HttpContext.Request.Headers.ContainsKey("X-Tenant-Id"))
        {
            ctx.ValidationFailures.Add(new("TenantId", "Missing tenant header"));
            return ctx.HttpContext.Response.SendErrorsAsync(ctx.ValidationFailures);
        }
        return Task.CompletedTask;
    }
}

public override void Configure()
{
    PreProcessor<TenantChecker>();
}
```

### Post-Processor

```csharp
public class AuditLogger<TReq, TRes> : IPostProcessor<TReq, TRes>
{
    public Task PostProcessAsync(IPostProcessorContext<TReq, TRes> ctx, CancellationToken ct)
    {
        var logger = ctx.HttpContext.Resolve<ILogger<AuditLogger<TReq, TRes>>>();
        logger.LogInformation("Request completed: {Path}", ctx.HttpContext.Request.Path);
        return Task.CompletedTask;
    }
}
```

### Global Processors

```csharp
app.UseFastEndpoints(c =>
{
    c.Endpoints.Configurator = ep =>
    {
        ep.PreProcessor<GlobalLogger>(Order.Before);
        ep.PostProcessor<GlobalAudit>(Order.After);
    };
});
```

## Domain Mapping

### Separate Mapper Class

```csharp
public class UserMapper : Mapper<CreateUserRequest, UserResponse, User>
{
    public override User ToEntity(CreateUserRequest r) => new()
    {
        Name = r.Name,
        Email = r.Email
    };

    public override UserResponse FromEntity(User e) => new()
    {
        Id = e.Id,
        FullName = e.Name
    };
}

public class CreateUserEndpoint : Endpoint<CreateUserRequest, UserResponse, UserMapper>
{
    public override async Task HandleAsync(CreateUserRequest req, CancellationToken ct)
    {
        var entity = Map.ToEntity(req);
        // Save entity...
        await SendAsync(Map.FromEntity(entity));
    }
}
```

## Command Bus

```csharp
// Command definition
public class GetUserQuery : ICommand<UserDto>
{
    public int UserId { get; set; }
}

// Handler
public class GetUserHandler : ICommandHandler<GetUserQuery, UserDto>
{
    public Task<UserDto> ExecuteAsync(GetUserQuery cmd, CancellationToken ct)
    {
        return Task.FromResult(new UserDto { Id = cmd.UserId });
    }
}

// Execute from endpoint
var user = await new GetUserQuery { UserId = 1 }.ExecuteAsync();
```

## Event Bus

```csharp
// Event
public class UserCreatedEvent
{
    public int UserId { get; set; }
}

// Handler (multiple allowed)
public class SendWelcomeEmail : IEventHandler<UserCreatedEvent>
{
    public Task HandleAsync(UserCreatedEvent e, CancellationToken ct)
    {
        // Send email...
        return Task.CompletedTask;
    }
}

// Publish
await PublishAsync(new UserCreatedEvent { UserId = 1 });
await PublishAsync(evt, Mode.WaitForNone);  // Fire-and-forget
```

## File Handling

```csharp
public override void Configure()
{
    Post("/api/upload");
    AllowFileUploads();
}

public override async Task HandleAsync(MyRequest req, CancellationToken ct)
{
    foreach (var file in Files)
    {
        using var stream = file.OpenReadStream();
        // Process file...
    }
}
```

### DTO Binding

```csharp
public class UploadRequest
{
    public IFormFile Document { get; set; }
    public List<IFormFile> Attachments { get; set; }
}
```

### Large Files (Streaming)

```csharp
public override void Configure()
{
    AllowFileUploads(dontAutoBindFormData: true);
    MaxRequestBodySize(100 * 1024 * 1024);  // 100MB
}

public override async Task HandleAsync(EmptyRequest req, CancellationToken ct)
{
    await foreach (var section in FormFileSectionsAsync(ct))
    {
        using var fs = File.Create(section.FileName);
        await section.Section.Body.CopyToAsync(fs, ct);
    }
}
```

## Swagger/OpenAPI

```csharp
bld.Services.AddFastEndpoints().SwaggerDocument(o =>
{
    o.DocumentSettings = s =>
    {
        s.Title = "My API";
        s.Version = "v1";
    };
});

app.UseFastEndpoints().UseSwaggerGen();
```

### Endpoint Documentation

```csharp
public override void Configure()
{
    Summary(s =>
    {
        s.Summary = "Creates a new user";
        s.Description = "Detailed description...";
        s.ExampleRequest = new CreateUserRequest { Name = "John" };
        s.ResponseExamples[200] = new UserResponse { Id = 1 };
        s.Responses[400] = "Validation failed";
    });

    Description(b => b
        .Produces<UserResponse>(200)
        .ProducesProblemDetails(400));
}
```

## Configuration Options

```csharp
app.UseFastEndpoints(c =>
{
    // Route prefix for all endpoints
    c.Endpoints.RoutePrefix = "api";

    // JSON serialization
    c.Serializer.Options.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;

    // Error response format
    c.Errors.UseProblemDetails();

    // Global endpoint settings
    c.Endpoints.Configurator = ep =>
    {
        ep.Description(d => d.WithTags("API"));
    };
});
```

## Testing

See [testing.md](references/testing.md) for integration and unit testing patterns.

## API Reference

See [api-reference.md](references/api-reference.md) for complete method reference.
