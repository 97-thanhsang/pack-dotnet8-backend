---
name: dotnet-controller
description: Implement .NET 8 API controllers with route conventions, MediatR dispatch, and MethodResult envelopes. Use when creating or updating backend API endpoints.
---

# SKILL: .NET 8 API Controller Patterns

## Purpose

Defines API Controller conventions for .NET 8 projects: route naming, MediatR integration, action method templates, MethodResult response pattern, and consistent status codes.

## When to Use

- Creating new API endpoints
- Implementing Controller action methods
- Defining route templates and constants
- Ensuring consistent API response format

---

## 1. Controller Pattern

```csharp
[Route(PatientRouteConstants.Prefix)]
[ApiController]
[Authorize]
public class CatalogController : ControllerBase
{
    private readonly IMediator _mediator;

    public CatalogController(IMediator mediator)
    {
        _mediator = mediator;
    }

    [HttpPost("search")]
    public async Task<ActionResult<MethodResult<PagedResult<CatalogListItem>>>> Search(
        [FromBody] SearchCatalogQuery query)
    {
        var result = await _mediator.Send(query);
        return Ok(result);
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<MethodResult<CatalogDetail>>> GetById(int id)
    {
        var result = await _mediator.Send(new GetCatalogQuery { Id = id });
        return Ok(result);
    }

    [HttpPost]
    public async Task<ActionResult<MethodResult<int>>> Create(
        [FromBody] CreateCatalogCommand command)
    {
        var result = await _mediator.Send(command);
        if (!result.Status) return BadRequest(result);
        return StatusCode(201, result);
    }

    [HttpPut("{id}")]
    public async Task<ActionResult<MethodResult>> Update(int id,
        [FromBody] UpdateCatalogCommand command)
    {
        if (id != command.Id) return BadRequest("Id mismatch");
        var result = await _mediator.Send(command);
        if (!result.Status) return BadRequest(result);
        return Ok(result);
    }

    [HttpDelete("{id}")]
    public async Task<ActionResult<MethodResult>> Delete(int id)
    {
        var result = await _mediator.Send(new DeleteCatalogCommand { Id = id });
        if (!result.Status) return NotFound(result);
        return Ok(result);
    }
}
```

---

## 2. Route Constants Pattern

```csharp
public static class CatalogRouteConstants
{
    public const string Prefix = "api/v1/catalog";
    public const string Search = "search";
    public const string GetById = "{id}";
    public const string Create = "";
    public const string Update = "{id}";
    public const string Delete = "{id}";
    public const string GetLookup = "lookup";
}
```

---

## 3. Action Method Templates

| HTTP Method | Route | Action | Status Code |
|---|---|---|---|
| `POST` | `search` | Query (body) | 200 |
| `GET` | `{id}` | Query (route) | 200 |
| `POST` | `` | Command | 201 |
| `PUT` | `{id}` | Command | 200 |
| `DELETE` | `{id}` | Command | 200 |
| `GET` | `lookup` | Query | 200 |

---

## 4. MethodResult Response Pattern

```csharp
// Success
public ActionResult<MethodResult<int>> Create(CreateCatalogCommand command)
{
    var result = await _mediator.Send(command);
    if (!result.Status) return BadRequest(result);
    return StatusCode(201, result);
}

// Not found
public ActionResult<MethodResult> GetById(int id)
{
    var result = await _mediator.Send(new GetCatalogQuery { Id = id });
    if (!result.Status) return NotFound(result);
    return Ok(result);
}

// Validation error (handled by pipeline behavior)
// Returns 400 with validation details automatically
```

---

## 5. Controller Organization Rules

- One controller per domain aggregate
- Inject `IMediator` only (no direct service injection)
- Action methods are thin — delegate to MediatR
- Use `nameof` for route constants
- Always return `ActionResult<MethodResult<T>>`
- Use consistent status codes (201 for create, 200 for others)
