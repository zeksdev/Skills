# FastEndpoints Testing

## Contents

- [Integration Testing](#integration-testing)
- [Unit Testing](#unit-testing)
- [Mocking Commands and Events](#mocking-commands-and-events)

## Integration Testing

Uses xUnit, WebApplicationFactory, and Shouldly.

### App Fixture

```csharp
public class MyApp : AppFixture<Program>
{
    protected override ValueTask SetupAsync()
    {
        // One-time setup
        return ValueTask.CompletedTask;
    }

    protected override void ConfigureServices(IServiceCollection s)
    {
        // Replace services for testing
        s.AddSingleton<IEmailService, FakeEmailService>();
    }

    protected override ValueTask TearDownAsync()
    {
        // Cleanup
        return ValueTask.CompletedTask;
    }
}
```

### Test Class

```csharp
public class UserTests(MyApp App) : TestBase<MyApp>
{
    [Fact]
    public async Task Create_User_Returns_Success()
    {
        var (rsp, res) = await App.Client.POSTAsync<CreateUserEndpoint, CreateUserRequest, CreateUserResponse>(
            new CreateUserRequest { Name = "John", Age = 25 });

        rsp.IsSuccessStatusCode.ShouldBeTrue();
        res.FullName.ShouldBe("John");
    }

    [Fact]
    public async Task Get_User_Returns_NotFound()
    {
        var rsp = await App.Client.GETAsync<GetUserEndpoint, GetUserRequest>(
            new GetUserRequest { Id = 999 });

        rsp.StatusCode.ShouldBe(HttpStatusCode.NotFound);
    }
}
```

### Request Annotations for Testing

```csharp
public class MyRequest
{
    [RouteParam]
    public int Id { get; set; }

    [FromHeader]
    public string CorrelationId { get; set; }

    [QueryParam]
    public int PageSize { get; set; }

    // No attribute = JSON body
    public string Name { get; set; }
}
```

### State Fixture (Shared State)

```csharp
public class TestState : StateFixture
{
    public int CreatedUserId { get; set; }

    protected override async ValueTask SetupAsync()
    {
        // Setup shared state
        CreatedUserId = 123;
    }
}

public class UserTests(MyApp App, TestState State) : TestBase<MyApp, TestState>
{
    [Fact]
    public async Task Test_With_State()
    {
        var id = State.CreatedUserId;
        // Use shared state...
    }
}
```

### Test Ordering

```csharp
// In AssemblyInfo.cs
[assembly: EnableAdvancedTesting]

public class OrderedTests(MyApp App) : TestBase<MyApp>
{
    [Fact, Priority(1)]
    public async Task First_Test() { }

    [Fact, Priority(2)]
    public async Task Second_Test() { }
}
```

### Collection Fixtures

```csharp
public class MyTestCollection : TestCollection<MyApp>;

[Collection<MyTestCollection>]
public class TestClassA(MyApp App) : TestBase { }

[Collection<MyTestCollection>]
public class TestClassB(MyApp App) : TestBase { }
```

## Unit Testing

Isolated endpoint testing without middleware, auth, or validation.

### Basic Unit Test

```csharp
[Fact]
public async Task Handler_Returns_Correct_Response()
{
    // Arrange
    var fakeService = A.Fake<IUserService>();
    A.CallTo(() => fakeService.GetUser(1))
        .Returns(Task.FromResult(new User { Id = 1, Name = "John" }));

    var ep = Factory.Create<GetUserEndpoint>(fakeService);
    var req = new GetUserRequest { Id = 1 };

    // Act
    await ep.HandleAsync(req, default);

    // Assert
    ep.Response.Name.ShouldBe("John");
    ep.ValidationFailed.ShouldBeFalse();
}
```

### With Service Registration

```csharp
[Fact]
public async Task Test_With_Services()
{
    var fakeRepo = A.Fake<IUserRepository>();

    var ep = Factory.Create<CreateUserEndpoint>(ctx =>
    {
        ctx.AddTestServices(s =>
        {
            s.AddSingleton(fakeRepo);
        });
    });

    await ep.HandleAsync(new CreateUserRequest { Name = "Test" }, default);
}
```

### ExecuteAsync Pattern

For endpoints returning DTOs directly:

```csharp
public class GetUserEndpoint : Endpoint<GetUserRequest, UserResponse>
{
    public override Task<UserResponse> ExecuteAsync(GetUserRequest req, CancellationToken ct)
    {
        return Task.FromResult(new UserResponse { Id = req.Id });
    }
}

// Test
[Fact]
public async Task Execute_Returns_Response()
{
    var ep = Factory.Create<GetUserEndpoint>();
    var result = await ep.ExecuteAsync(new GetUserRequest { Id = 1 }, default);

    result.Id.ShouldBe(1);
}
```

### Route Parameters in Unit Tests

```csharp
var ep = Factory.Create<GetUserEndpoint>(ctx =>
{
    ctx.Request.RouteValues.Add("id", "123");
});
```

### Testing Validation Failures

```csharp
[Fact]
public async Task Invalid_Request_Fails_Validation()
{
    var ep = Factory.Create<CreateUserEndpoint>();
    var req = new CreateUserRequest { Name = "" };  // Invalid

    await ep.HandleAsync(req, default);

    ep.ValidationFailed.ShouldBeTrue();
    ep.ValidationFailures.ShouldContain(f => f.PropertyName == "Name");
}
```

## Mocking Commands and Events

### Mock Command Handler

```csharp
[Fact]
public async Task Test_With_Mocked_Command()
{
    var fakeHandler = A.Fake<ICommandHandler<GetUserQuery, UserDto>>();
    A.CallTo(() => fakeHandler.ExecuteAsync(A<GetUserQuery>.Ignored, A<CancellationToken>.Ignored))
        .Returns(Task.FromResult(new UserDto { Id = 1, Name = "Mocked" }));

    fakeHandler.RegisterForTesting();

    var result = await new GetUserQuery { UserId = 1 }.ExecuteAsync();

    result.Name.ShouldBe("Mocked");
}
```

### Mock Event Handler

```csharp
[Fact]
public async Task Test_Event_Publishing()
{
    var fakeHandler = A.Fake<IEventHandler<UserCreatedEvent>>();

    Factory.RegisterTestServices(s =>
    {
        s.AddSingleton(fakeHandler);
    });

    var ep = Factory.Create<CreateUserEndpoint>();
    await ep.HandleAsync(new CreateUserRequest { Name = "Test" }, default);

    A.CallTo(() => fakeHandler.HandleAsync(
        A<UserCreatedEvent>.That.Matches(e => e.UserId > 0),
        A<CancellationToken>.Ignored))
        .MustHaveHappened();
}
```

### HTTP Client Methods

| Method | Usage |
|--------|-------|
| `GETAsync<TEndpoint, TRequest>` | GET with request |
| `GETAsync<TEndpoint, TRequest, TResponse>` | GET with request and response |
| `POSTAsync<TEndpoint, TRequest, TResponse>` | POST with request and response |
| `PUTAsync<TEndpoint, TRequest, TResponse>` | PUT with request and response |
| `PATCHAsync<TEndpoint, TRequest, TResponse>` | PATCH with request and response |
| `DELETEAsync<TEndpoint, TRequest>` | DELETE with request |
