---
name: dotnet-middleware
description: Configure .NET 8 middleware pipeline for exception handling, request logging, correlation IDs, and CORS. Use when adding cross-cutting HTTP request processing.
---

# SKILL: .NET 8 Middleware Patterns

## Purpose

Defines ASP.NET Core middleware pipeline patterns: global exception handling, request/response logging, correlation ID tracking, response formatting, and CORS configuration.

## When to Use

- Adding global exception handling
- Implementing request logging
- Tracking correlation IDs across requests
- Standardizing API response format
- Configuring CORS

---

## 1. Exception Handling Middleware

```csharp
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;

    public ExceptionHandlingMiddleware(RequestDelegate next, ILogger<ExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (ValidationException ex)
        {
            _logger.LogWarning(ex, "Validation failed");
            context.Response.StatusCode = 400;
            await context.Response.WriteAsJsonAsync(MethodResult.Fail(ex.Message));
        }
        catch (BusinessException ex)
        {
            _logger.LogWarning(ex, "Business rule violation");
            context.Response.StatusCode = 400;
            await context.Response.WriteAsJsonAsync(MethodResult.Fail(ex.Message));
        }
        catch (NotFoundException ex)
        {
            context.Response.StatusCode = 404;
            await context.Response.WriteAsJsonAsync(MethodResult.Fail(ex.Message));
        }
        catch (UnauthorizedAccessException ex)
        {
            context.Response.StatusCode = 403;
            await context.Response.WriteAsJsonAsync(MethodResult.Fail(ex.Message));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception");
            context.Response.StatusCode = 500;
            await context.Response.WriteAsJsonAsync(
                MethodResult.Fail("Có lỗi xảy ra, vui lòng thử lại sau"));
        }
    }
}

// Registration in Program.cs:
// app.UseMiddleware<ExceptionHandlingMiddleware>();
```

---

## 2. Request Logging Middleware

```csharp
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestLoggingMiddleware> _logger;

    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();
        var correlationId = context.TraceIdentifier;

        _logger.LogInformation("[{CorrelationId}] {Method} {Path}",
            correlationId, context.Request.Method, context.Request.Path);

        await _next(context);

        stopwatch.Stop();
        _logger.LogInformation("[{CorrelationId}] {StatusCode} in {Elapsed}ms",
            correlationId, context.Response.StatusCode, stopwatch.ElapsedMilliseconds);
    }
}
```

---

## 3. Correlation ID Middleware

```csharp
public class CorrelationIdMiddleware
{
    private readonly RequestDelegate _next;

    public async Task InvokeAsync(HttpContext context)
    {
        var correlationId = context.Request.Headers["X-Correlation-ID"]
            .FirstOrDefault() ?? Guid.NewGuid().ToString();

        context.TraceIdentifier = correlationId;
        context.Response.Headers["X-Correlation-ID"] = correlationId;

        await _next(context);
    }
}
```

---

## 4. Middleware Pipeline Order

```csharp
// Program.cs — CORRECT ORDER MATTERS
var app = builder.Build();

app.UseMiddleware<CorrelationIdMiddleware>();
app.UseMiddleware<RequestLoggingMiddleware>();
app.UseMiddleware<ExceptionHandlingMiddleware>();

app.UseCors();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
```

**Order rules:**
1. Correlation ID (earliest — needed by all downstream)
2. Request Logging (capture request before processing)
3. Exception Handling (catch all exceptions from downstream)
4. CORS (before auth)
5. Authentication
6. Authorization
7. Controllers / Endpoints
