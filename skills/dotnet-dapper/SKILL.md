---
name: dotnet-dapper
description: Implement high-performance data access with Dapper repositories, stored procedure execution, and dynamic parameters. Use when writing SQL queries or repository methods in .NET 8.
---

# SKILL: .NET 8 Dapper & Repository Patterns

## Purpose

Defines Dapper data access patterns for .NET 8 projects: repository interface/implementation, stored procedure execution, dynamic parameter building, pagination queries, and transaction handling.

## When to Use

- Creating repository implementations
- Executing stored procedures via Dapper
- Building dynamic query parameters
- Implementing paginated data access
- Managing transactions across multiple SP calls

---

## 1. Repository Interface

```csharp
public interface ICatalogRepository
{
    Task<PagedResult<CatalogListItem>> SearchPaged(
        int pageIndex, int pageSize, string? keyword, bool? status,
        string sortColumn, string sortDirection);

    Task<CatalogDetail?> GetById(int id);
    Task<int> Create(CatalogEntity entity);
    Task<bool> Update(CatalogEntity entity);
    Task<bool> Delete(int id);
    Task<bool> CheckDuplicateCode(string code, int? excludeId);
    Task<IEnumerable<CatalogLookup>> GetLookup();
}
```

---

## 2. Repository Implementation

```csharp
public class CatalogRepository : ICatalogRepository
{
    private readonly ISqlConnectionFactory _connectionFactory;

    public CatalogRepository(ISqlConnectionFactory connectionFactory)
    {
        _connectionFactory = connectionFactory;
    }

    public async Task<PagedResult<CatalogListItem>> SearchPaged(
        int pageIndex, int pageSize, string? keyword, bool? status,
        string sortColumn, string sortDirection)
    {
        using var conn = _connectionFactory.CreateConnection();
        var parameters = new DynamicParameters();
        parameters.Add("@PageIndex", pageIndex);
        parameters.Add("@PageSize", pageSize);
        parameters.Add("@Keyword", keyword);
        parameters.Add("@Status", status);
        parameters.Add("@SortColumn", sortColumn);
        parameters.Add("@SortDirection", sortDirection);

        var result = await conn.QueryAsync<CatalogListItem>(
            "SP_V2_EDU_CATALOG_SEARCH",
            parameters,
            commandType: CommandType.StoredProcedure
        );

        return new PagedResult<CatalogListItem>
        {
            Data = result.ToList(),
            Total = result.FirstOrDefault()?.TotalCount ?? 0,
        };
    }
}
```

---

## 3. Single Record Query

```csharp
public async Task<CatalogDetail?> GetById(int id)
{
    using var conn = _connectionFactory.CreateConnection();
    var parameters = new DynamicParameters();
    parameters.Add("@Id", id);

    return await conn.QueryFirstOrDefaultAsync<CatalogDetail>(
        "SP_V2_EDU_CATALOG_GETBYID",
        parameters,
        commandType: CommandType.StoredProcedure
    );
}
```

---

## 4. Create with Output

```csharp
public async Task<int> Create(CatalogEntity entity)
{
    using var conn = _connectionFactory.CreateConnection();
    var parameters = new DynamicParameters();
    parameters.Add("@Code", entity.Code);
    parameters.Add("@Name", entity.Name);
    parameters.Add("@Description", entity.Description);
    parameters.Add("@CreatedBy", entity.CreatedBy);
    parameters.Add("@Status", dbType: DbType.Int32, direction: ParameterDirection.Output);
    parameters.Add("@Message", dbType: DbType.String, size: 500, direction: ParameterDirection.Output);
    parameters.Add("@Data", dbType: DbType.Int32, direction: ParameterDirection.Output);

    await conn.ExecuteAsync("SP_V2_EDU_CATALOG_CREATE", parameters,
        commandType: CommandType.StoredProcedure);

    var status = parameters.Get<int>("@Status");
    if (status == 0)
        throw new BusinessException(parameters.Get<string>("@Message"));

    return parameters.Get<int>("@Data");
}
```

---

## 5. Dynamic Parameters Helper

```csharp
public static class DynamicParametersExtensions
{
    public static void AddIfNotNull<T>(this DynamicParameters parameters, string name, T? value)
    {
        if (value is not null)
            parameters.Add(name, value);
    }

    public static void AddNullable<T>(this DynamicParameters parameters, string name, T? value)
    {
        parameters.Add(name, value ?? (object)DBNull.Value);
    }
}
```

---

## 6. Transaction Pattern

```csharp
public async Task<MethodResult> CreateWithItems(CreateCatalogCommand command)
{
    using var conn = _connectionFactory.CreateConnection();
    await conn.OpenAsync();
    using var tx = await conn.BeginTransactionAsync();

    try
    {
        // Create catalog
        var catalogId = await conn.ExecuteAsync("SP_V2_EDU_CATALOG_CREATE", ...,
            transaction: tx);

        // Create items
        foreach (var item in command.Items)
        {
            await conn.ExecuteAsync("SP_V2_EDU_ITEM_CREATE", ...,
                transaction: tx);
        }

        await tx.CommitAsync();
        return MethodResult.Ok();
    }
    catch
    {
        await tx.RollbackAsync();
        throw;
    }
}
```
