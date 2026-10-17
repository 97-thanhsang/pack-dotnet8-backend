---
name: dotnet-cqrs
description: Implement CQRS architecture with MediatR commands, queries, handlers, and pipeline behaviors in .NET 8. Use when building application-layer business logic.
---

# SKILL: .NET 8 CQRS with MediatR Patterns

## Purpose

Defines the CQRS architecture with MediatR for .NET 8 projects: Command/Query/Handler pattern, pipeline behaviors for validation/logging, AutoMapper profiles.

## When to Use

- Creating new Command/Query/Handler
- Adding MediatR pipeline behaviors
- Implementing request validation via FluentValidation
- Structuring application layer in Clean Architecture

---

## 1. Command Pattern

```csharp
public record CreateCatalogCommand : IRequest<MethodResult<int>>
{
    public string Code { get; init; } = string.Empty;
    public string Name { get; init; } = string.Empty;
    public string? Description { get; init; }
    public int CreatedBy { get; init; }
}
```

```csharp
public class CreateCatalogCommandHandler : IRequestHandler<CreateCatalogCommand, MethodResult<int>>
{
    private readonly ICatalogRepository _repository;
    private readonly IMapper _mapper;

    public CreateCatalogCommandHandler(ICatalogRepository repository, IMapper mapper)
    {
        _repository = repository;
        _mapper = mapper;
    }

    public async Task<MethodResult<int>> Handle(CreateCatalogCommand request, CancellationToken cancellationToken)
    {
        // Duplicate check
        var exists = await _repository.CheckDuplicateCode(request.Code, null);
        if (exists)
            return MethodResult<int>.Fail("Mã đã tồn tại");

        var entity = _mapper.Map<CatalogEntity>(request);
        var id = await _repository.Create(entity);
        return MethodResult<int>.Ok(id, "Thêm mới thành công");
    }
}
```

---

## 2. Query Pattern

```csharp
public record SearchCatalogQuery : IRequest<MethodResult<PagedResult<CatalogListItem>>>
{
    public int PageIndex { get; init; } = 1;
    public int PageSize { get; init; } = 20;
    public string? Keyword { get; init; }
    public bool? Status { get; init; }
    public string SortColumn { get; init; } = "CreatedDate";
    public string SortDirection { get; init; } = "DESC";
}
```

```csharp
public class SearchCatalogQueryHandler : IRequestHandler<SearchCatalogQuery, MethodResult<PagedResult<CatalogListItem>>>
{
    private readonly ICatalogRepository _repository;

    public SearchCatalogQueryHandler(ICatalogRepository repository)
    {
        _repository = repository;
    }

    public async Task<MethodResult<PagedResult<CatalogListItem>>> Handle(
        SearchCatalogQuery request, CancellationToken cancellationToken)
    {
        var result = await _repository.SearchPaged(
            request.PageIndex, request.PageSize,
            request.Keyword, request.Status,
            request.SortColumn, request.SortDirection
        );
        return MethodResult<PagedResult<CatalogListItem>>.Ok(result);
    }
}
```

---

## 3. Update Command Pattern

```csharp
public record UpdateCatalogCommand : IRequest<MethodResult>
{
    public int Id { get; init; }
    public string Code { get; init; } = string.Empty;
    public string Name { get; init; } = string.Empty;
    public bool Status { get; init; }
    public int UpdatedBy { get; init; }
}
```

Handler checks existence, checks duplicate (exclude self), updates.

---

## 4. Delete Command Pattern

```csharp
public record DeleteCatalogCommand : IRequest<MethodResult>
{
    public int Id { get; init; }
}

// Handler: check exists → soft delete (Status = false) or hard delete
// Return MethodResult.Ok() or MethodResult.Fail("Không tìm thấy")
```

---

## 5. Pipeline Behavior (Validation)

```csharp
public class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
    {
        _validators = validators;
    }

    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        if (!_validators.Any()) return await next();

        var context = new ValidationContext<TRequest>(request);
        var failures = _validators
            .Select(v => v.Validate(context))
            .SelectMany(r => r.Errors)
            .Where(f => f != null)
            .ToList();

        if (failures.Any())
            throw new ValidationException(failures);

        return await next();
    }
}
```

---

## 6. Naming Conventions

| Element | Example |
|---|---|
| Command record | `{Action}{Entity}Command` |
| Query record | `{Action}{Entity}Query` |
| Handler class | `{Command/Query}Handler` |
| Validator class | `{Command/Query}Validator` |
| Repository interface | `I{Entity}Repository` |
