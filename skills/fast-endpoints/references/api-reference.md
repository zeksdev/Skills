# FastEndpoints API Reference

## Contents

- [Endpoint Configuration Methods](#endpoint-configuration-methods)
- [Response Methods](#response-methods)
- [Binding Attributes](#binding-attributes)
- [Security Methods](#security-methods)
- [Validation Methods](#validation-methods)
- [Swagger/Documentation](#swaggerdocumentation)
- [Global Configuration](#global-configuration)

## Endpoint Configuration Methods

Call in `Configure()` override:

| Method | Description |
|--------|-------------|
| `Get(route)` | HTTP GET |
| `Post(route)` | HTTP POST |
| `Put(route)` | HTTP PUT |
| `Patch(route)` | HTTP PATCH |
| `Delete(route)` | HTTP DELETE |
| `Verbs(Http.GET, Http.POST)` | Multiple HTTP methods |
| `Routes("/path1", "/path2")` | Multiple routes |
| `AllowAnonymous()` | No auth required |
| `AllowFormData()` | Enable multipart/form-data |
| `AllowFileUploads()` | Enable file uploads |
| `DontThrowIfValidationFails()` | Disable auto-validation response |
| `DontAutoTag()` | Disable auto Swagger tagging |
| `Version(1)` | API versioning |
| `Version(1, deprecateAt: 2)` | Version with deprecation |
| `RoutePrefixOverride(prefix)` | Override global route prefix |
| `MaxRequestBodySize(bytes)` | Set max request size |
| `SerializerContext<TContext>()` | STJ source generation |

## Response Methods

### Send Methods (in HandleAsync)

| Method | Description |
|--------|-------------|
| `await SendAsync(response)` | 200 OK with body |
| `await SendAsync(response, statusCode)` | Custom status with body |
| `await SendOkAsync(response)` | Explicit 200 OK |
| `await SendOkAsync()` | 200 OK no body |
| `await SendNoContentAsync()` | 204 No Content |
| `await SendCreatedAtAsync<TEndpoint>(routeValues, response)` | 201 Created |
| `await SendNotFoundAsync()` | 404 Not Found |
| `await SendNotFoundAsync(response)` | 404 with body |
| `await SendUnauthorizedAsync()` | 401 Unauthorized |
| `await SendForbiddenAsync()` | 403 Forbidden |
| `await SendErrorsAsync()` | 400 with ValidationFailures |
| `await SendErrorsAsync(statusCode)` | Custom status with errors |
| `await SendRedirectAsync(url)` | Redirect response |
| `await SendStreamAsync(stream, fileName, contentType)` | Stream file |
| `await SendFileAsync(fileInfo)` | Send FileInfo |
| `await SendBytesAsync(bytes, fileName, contentType)` | Send byte array |
| `await SendResultAsync(typedResult)` | Send TypedResult in HandleAsync |

### Response Property

Assign to Response property for automatic 200 OK:

```csharp
// Assign properties directly
Response.FullName = req.FirstName + " " + req.LastName;
Response.Age = req.Age;

// Or assign new instance
Response = new MyResponse { Id = 1 };
// Auto-sent at end of HandleAsync
```

### Conditional Responses with Task<Void>

Change return type to `Task<Void>` to stop execution after sending:

```csharp
public override async Task<Void> HandleAsync(MyRequest req, CancellationToken ct)
{
    if (req.Id == 0)
        return await SendNotFoundAsync();  // Execution stops here

    return await SendOkAsync(new MyResponse { Id = req.Id });
}
```

### ExecuteAsync vs HandleAsync

| Method | Return Type | Use Case |
|--------|-------------|----------|
| `HandleAsync` | `Task` | Use Send methods or set Response property |
| `HandleAsync` | `Task<Void>` | Conditional responses, early exit |
| `ExecuteAsync` | `Task<TResponse>` | Return response directly |
| `ExecuteAsync` | `Task<Results<...>>` | Union types with TypedResults |

### ExecuteAsync with Direct Return

```csharp
public override Task<UserResponse> ExecuteAsync(GetUserRequest req, CancellationToken ct)
{
    return Task.FromResult(new UserResponse { Id = req.Id, Name = "John" });
}
```

### ExecuteAsync with Union Types

```csharp
public override async Task<Results<Ok<UserResponse>, NotFound, ProblemDetails>> ExecuteAsync(
    GetUserRequest req, CancellationToken ct)
{
    if (req.Id == 0)
        return TypedResults.NotFound();

    var user = await _db.GetUserAsync(req.Id);
    return TypedResults.Ok(new UserResponse { Id = user.Id });
}
```

**Common TypedResults:**
- `TypedResults.Ok<T>(value)` - 200 OK
- `TypedResults.NotFound()` - 404
- `TypedResults.BadRequest()` - 400
- `TypedResults.NoContent()` - 204
- `TypedResults.Created<T>(uri, value)` - 201
- `TypedResults.Problem()` - ProblemDetails

## Binding Attributes

Apply to request DTO properties:

| Attribute | Source |
|-----------|--------|
| `[FromBody]` | JSON body (specific property) |
| `[FromHeader]` | HTTP header |
| `[FromHeader("X-Custom")]` | Named header |
| `[FromClaim]` | JWT/Cookie claim |
| `[FromClaim("sub")]` | Named claim |
| `[FromClaim(IsRequired = false)]` | Optional claim |
| `[QueryParam]` | Query string |
| `[RouteParam]` | Route parameter (explicit) |
| `[FormField]` | Form field (explicit) |
| `[BindFrom("name")]` | Bind from different name |
| `[HasPermission("perm")]` | Permission check (bool) |
| `[DontBind(Source.QueryParam)]` | Exclude binding source |
| `[JsonPropertyName("name")]` | JSON property name |

### Manual Binding

```csharp
var id = Route<int>("id");
var id = Route<int>("id", isRequired: false);
var page = Query<int>("page");
var page = Query<int>("page", isRequired: false);
```

## Security Methods

Call in `Configure()`:

| Method | Description |
|--------|-------------|
| `Roles("Admin", "User")` | Require ANY role |
| `RolesAll("Admin", "User")` | Require ALL roles |
| `Claims("ClaimType")` | Require ANY claim |
| `ClaimsAll("Claim1", "Claim2")` | Require ALL claims |
| `Permissions("Perm1")` | Require ANY permission |
| `PermissionsAll("Perm1", "Perm2")` | Require ALL permissions |
| `Scopes("scope1")` | Require ANY scope |
| `ScopesAll("scope1", "scope2")` | Require ALL scopes |
| `Policies("PolicyName")` | Require policy |
| `AuthSchemes("Bearer", "Cookie")` | Specify auth schemes |
| `AllowAnonymous()` | No auth required |
| `AccessControl("PermCode", Apply.ToThisEndpoint)` | Source-generated permissions |

### User Properties

Access in `HandleAsync`:

```csharp
User.Identity.IsAuthenticated
User.IsInRole("Admin")
User.HasPermission("Users.Create")
User.ClaimValue("sub")
User.Claims
```

### JWT Token Generation

```csharp
var token = JwtBearer.CreateToken(o =>
{
    o.SigningKey = "secret";
    o.ExpireAt = DateTime.UtcNow.AddHours(1);
    o.User.Roles.Add("Admin");
    o.User.Claims.Add(("UserId", "123"));
    o.User.Permissions.Add("Users.Create");
    o.User["CustomClaim"] = "value";
});
```

### Cookie Auth

```csharp
await CookieAuth.SignInAsync(u =>
{
    u.Roles.Add("User");
    u.Claims.Add(new Claim("Email", email));
    u.Permissions.Add("View");
});

await CookieAuth.SignOutAsync();
```

## Validation Methods

### In Validator Constructor

```csharp
RuleFor(x => x.Name).NotEmpty();
RuleFor(x => x.Age).InclusiveBetween(18, 120);
RuleFor(x => x.Email).EmailAddress();
RuleFor(x => x.Items).NotEmpty().ForEach(item => item.NotNull());
```

### In HandleAsync

| Method | Description |
|--------|-------------|
| `AddError(r => r.Prop, "message")` | Add validation error |
| `AddError("field", "message")` | Add error by field name |
| `ThrowIfAnyErrors()` | Send 400 if errors exist |
| `ThrowError("message")` | Immediate 400 response |
| `ThrowError(r => r.Prop, "message")` | Immediate 400 for property |
| `ValidationFailed` | Check if validation failed |
| `ValidationFailures` | Access error collection |

### Global Validation Context

```csharp
// From anywhere (e.g., domain layer)
var ctx = ValidationContext<Request>.Instance;
ctx.AddError(r => r.Id, "Invalid ID");
```

## Swagger/Documentation

### Summary Configuration

```csharp
Summary(s =>
{
    s.Summary = "Short description";
    s.Description = "Detailed description";
    s.ExampleRequest = new MyRequest { };
    s.ResponseExamples[200] = new MyResponse { };
    s.Responses[200] = "Success description";
    s.Responses[400] = "Validation error";
    s.Params["id"] = "The user ID";
    s.RequestParam(r => r.Name, "User name");
});
```

### Description Configuration

```csharp
Description(b => b
    .Accepts<MyRequest>("application/json")
    .Produces<MyResponse>(200)
    .Produces<MyResponse>(200, "application/json")
    .ProducesProblemDetails(400)
    .ProducesProblemFE<ErrorResponse>(500)
    .WithTags("Users")
    .WithName("GetUser"));
```

### Separate Summary Class

```csharp
public class MyEndpointSummary : Summary<MyEndpoint>
{
    public MyEndpointSummary()
    {
        Summary = "Get user";
        ExampleRequest = new MyRequest { Id = 1 };
        Response<MyResponse>(200, "Success", example: new() { Name = "John" });
        Response<ErrorResponse>(400, "Validation failed");
    }
}
```

## Global Configuration

```csharp
app.UseFastEndpoints(c =>
{
    // Routing
    c.Endpoints.RoutePrefix = "api";
    c.Endpoints.ShortNames = true;

    // Serialization
    c.Serializer.Options.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
    c.Serializer.Options.DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull;

    // Binding
    c.Binding.UsePropertyNamingPolicy = true;
    c.Binding.ValueParserFor<Guid>(MyGuidParser);

    // Errors
    c.Errors.UseProblemDetails();
    c.Errors.ResponseBuilder = (failures, ctx, code) => new MyErrorResponse(failures);

    // Validation
    c.Validation.EnableDataAnnotationsSupport = true;

    // Security
    c.Security.RoleClaimType = "role";

    // Global endpoint config
    c.Endpoints.Configurator = ep =>
    {
        ep.PreProcessor<GlobalLogger>(Order.Before);
        ep.Description(d => d.WithTags("API"));
    };
});
```

### Service Registration

```csharp
bld.Services.AddFastEndpoints();
bld.Services.AddFastEndpoints(o => o.IncludeAbstractValidators = true);

// JWT
bld.Services.AddAuthenticationJwtBearer(s => s.SigningKey = "key");
bld.Services.AddAuthorization();

// Cookies
bld.Services.AddAuthenticationCookie(validFor: TimeSpan.FromMinutes(30));

// Swagger
bld.Services.SwaggerDocument(o =>
{
    o.DocumentSettings = s =>
    {
        s.Title = "My API";
        s.Version = "v1";
    };
    o.EnableJWTBearerAuth = true;
    o.ShortSchemaNames = true;
});
```

### Middleware Pipeline Order

```csharp
app.UseAuthentication();
app.UseAuthorization();
app.UseFastEndpoints();
app.UseSwaggerGen();  // Must be after UseFastEndpoints
```

## Pre/Post Processors

### Interfaces

```csharp
// Pre-processor
public class MyPreProcessor : IPreProcessor<TRequest>
{
    public Task PreProcessAsync(IPreProcessorContext<TRequest> ctx, CancellationToken ct);
}

// Post-processor
public class MyPostProcessor : IPostProcessor<TRequest, TResponse>
{
    public Task PostProcessAsync(IPostProcessorContext<TRequest, TResponse> ctx, CancellationToken ct);
}

// Global processors
public class GlobalPre : IGlobalPreProcessor
{
    public Task PreProcessAsync(IPreProcessorContext ctx, CancellationToken ct);
}

public class GlobalPost : IGlobalPostProcessor
{
    public Task PostProcessAsync(IPostProcessorContext ctx, CancellationToken ct);
}
```

### Context Properties

**IPreProcessorContext<TRequest>**:
- `Request` - The request DTO
- `HttpContext` - ASP.NET HttpContext
- `ValidationFailures` - Add errors here

**IPostProcessorContext<TRequest, TResponse>**:
- `Request` - The request DTO
- `Response` - The response DTO
- `HttpContext` - ASP.NET HttpContext
- `ExceptionDispatchInfo` - Exception if thrown
- `HasExceptionOccurred` - Check for exceptions
- `MarkExceptionAsHandled()` - Suppress exception

### Registration

```csharp
// Per-endpoint
public override void Configure()
{
    PreProcessor<MyPreProcessor>();
    PostProcessor<MyPostProcessor>();
}

// Global
c.Endpoints.Configurator = ep =>
{
    ep.PreProcessor<GlobalLogger>(Order.Before);
    ep.PreProcessors(Order.Before, typeof(OpenGenericProcessor<>));
};
```

## Mappers

```csharp
// Full mapper
public class MyMapper : Mapper<TRequest, TResponse, TEntity>
{
    public override TEntity ToEntity(TRequest r) => new();
    public override TResponse FromEntity(TEntity e) => new();
}

// Request-only
public class MyMapper : RequestMapper<TRequest, TEntity>
{
    public override TEntity ToEntity(TRequest r) => new();
}

// Response-only
public class MyMapper : ResponseMapper<TResponse, TEntity>
{
    public override TResponse FromEntity(TEntity e) => new();
}

// Usage in endpoint
public class MyEndpoint : Endpoint<TRequest, TResponse, TMapper>
{
    public override Task HandleAsync(TRequest req, CancellationToken ct)
    {
        var entity = Map.ToEntity(req);
        var response = Map.FromEntity(entity);
    }
}
```

## Command/Event Bus

### Commands

```csharp
// Without result
public class DoSomething : ICommand { }
public class Handler : ICommandHandler<DoSomething>
{
    public Task ExecuteAsync(DoSomething cmd, CancellationToken ct);
}

// With result
public class GetData : ICommand<DataDto> { }
public class Handler : ICommandHandler<GetData, DataDto>
{
    public Task<DataDto> ExecuteAsync(GetData cmd, CancellationToken ct);
}

// Execute
await new DoSomething().ExecuteAsync();
var result = await new GetData().ExecuteAsync();
```

### Events

```csharp
public class MyEvent { }
public class Handler1 : IEventHandler<MyEvent>
{
    public Task HandleAsync(MyEvent e, CancellationToken ct);
}

// Publish
await PublishAsync(new MyEvent());
await PublishAsync(evt, Mode.WaitForNone);  // Fire-and-forget
await PublishAsync(evt, Mode.WaitForAny);   // Wait for first
await PublishAsync(evt, Mode.WaitForAll);   // Wait for all (default)
```

## File Handling

### Configuration

```csharp
public override void Configure()
{
    AllowFileUploads();
    AllowFileUploads(dontAutoBindFormData: true);  // For streaming
    MaxRequestBodySize(100 * 1024 * 1024);
}
```

### Access Files

```csharp
// Via Files collection
foreach (var file in Files)
{
    var name = file.FileName;
    var size = file.Length;
    var stream = file.OpenReadStream();
}

// Via DTO binding
public class Request
{
    public IFormFile File { get; set; }
    public List<IFormFile> Files { get; set; }
}

// Streaming (large files)
await foreach (var section in FormFileSectionsAsync(ct))
{
    await section.Section.Body.CopyToAsync(outputStream, ct);
}
```
