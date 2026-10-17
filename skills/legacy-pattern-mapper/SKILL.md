---
name: legacy-pattern-mapper
description: Chuyển đổi logic nghiệp vụ từ WinForms/BLL/DAL legacy sang .NET 8 Clean Architecture (C# 12, Minimal APIs, Primary Constructors) và Angular standalone/TypeScript 5+.
---

# SKILL: Legacy Pattern Mapper

Trích xuất business rules từ WinForms (.cs, BLL, DAL, Stored Procedures). Chuyển đổi sang .NET 8 (C# 12), Minimal APIs, FluentValidation và TypeScript 5+. Không tái sử dụng code rác cũ.

## When to Use

Kích hoạt khi gặp các tình huống sau:
- Tái cấu trúc (refactor/modernize) code WinForms, BLL, DAL sang Web API (.NET 8) và Frontend (Angular/TypeScript 5+).
- Đọc source file `.cs` legacy, phát hiện logic nằm trong event handler UI (`btnSave_Click`, `Grid_Load`), BLL hoặc Stored Procedures.
- Cần bóc tách validation rules, checks trùng lặp, tính toán tài chính/đơn hàng khỏi SQL/BLL.
- Thiết kế endpoint Minimal API mới thay thế các controller hoặc WebForms/WCF endpoints cũ.

---

## 1. Bảng đối chiếu Patterns Legacy sang Modern

| Thành phần Legacy | .NET 8 / C# 12 / TS 5+ Target | Chi tiết xử lý |
|---|---|---|
| `BLL.ValidateXxx()` | `AbstractValidator<TCommand>` | Tách sang FluentValidation độc lập. |
| `BLL.CheckDuplicate()` | Domain Service hoặc Repository query | Check `EXISTS` trong DB. Bắn `ConflictHttpResult` (409). |
| `BLL.CalculateXxx()` | Pure static method / Domain logic | Tách thành hàm thuần (pure function), immutable record. |
| `DAL.Repository.GetAll()` | Minimal API + Dapper / EF Core | Endpoint `app.MapGet()` với paging (`limit`, `offset`). |
| `WinForms Grid_Load` | Angular Signal-based Component | Gọi API bằng `HttpClient`, bind qua Signal / `toSignal()`. |
| `WinForms btnSave_Click` | `app.MapPost()` + Reactive Form | Client gửi JSON payload; server bắt qua Primary Constructor DTO. |
| `DataSet` / `DataTable` | Typed Record / ReadOnlyCollection | Bỏ untyped data, dùng `public record ItemDto(...)`. |
| `SP_XXX_GetAll` | SQL pagination query | Chuyển sang `OFFSET @Skip ROWS FETCH NEXT @Take ROWS ONLY`. |

---

## 2. Quy trình bóc tách logic (Extraction Process)

1. **Đọc mã nguồn legacy**: Xác định input, output, side effects.
2. **Lọc nghiệp vụ (Business Rules)**:
   - Rule xác thực dữ liệu vào -> Đưa vào FluentValidation.
   - Rule biến đổi dữ liệu / tính toán -> Đưa vào Domain logic (Pure functions, C# 12 records).
   - Rule phân quyền -> Đưa vào Endpoint Filter / Authorization Policy.
   - Truy vấn DB -> Viết lại query rõ ràng, có phân trang.
3. **Hiện đại hóa cú pháp**: Áp dụng C# 12 Primary Constructors, Minimal APIs, TS 5 Typed DTOs.
4. **Viết test tự động**: Tạo unit test tối thiểu cho các edge cases của rule đã bóc tách.

---

## 3. Code Example: Legacy vs Modern

### Legacy BLL (Trước)
```csharp
// CatalogBLL.cs (Legacy)
public class CatalogBLL
{
    private CatalogDAL _dal = new CatalogDAL();
    
    public int SaveCatalog(string code, string name, decimal price)
    {
        if (string.IsNullOrEmpty(code)) throw new Exception("Code is required");
        if (price <= 0) throw new Exception("Price must be positive");
        if (_dal.CheckExist(code) > 0) throw new Exception("Code already exists");
        
        return _dal.Insert(code, name, price);
    }
}
```

### Modern .NET 8 / C# 12 (Sau)

```csharp
// 1. DTO & Validation
using FluentValidation;

public readonly record struct CreateCatalogCommand(string Code, string Name, decimal Price);

public sealed class CreateCatalogValidator : AbstractValidator<CreateCatalogCommand>
{
    public CreateCatalogValidator()
    {
        RuleFor(x => x.Code).NotEmpty().MaximumLength(50);
        RuleFor(x => x.Name).NotEmpty().MaximumLength(200);
        RuleFor(x => x.Price).GreaterThan(0);
    }
}

// 2. Repository Contract & Minimal API Endpoint
public interface ICatalogRepository
{
    Task<bool> ExistsByCodeAsync(string code, CancellationToken ct);
    Task<int> CreateAsync(CreateCatalogCommand command, CancellationToken ct);
}

// Map endpoints using Minimal APIs & C# 12 Primary Constructor pattern
public static class CatalogEndpoints
{
    public static RouteGroupBuilder MapCatalogEndpoints(this RouteGroupBuilder group)
    {
        group.MapPost("/", async (
            CreateCatalogCommand cmd,
            IValidator<CreateCatalogCommand> validator,
            ICatalogRepository repo,
            CancellationToken ct) =>
        {
            var validationResult = await validator.ValidateAsync(cmd, ct);
            if (!validationResult.IsValid)
            {
                return Results.ValidationProblem(validationResult.ToDictionary());
            }

            if (await repo.ExistsByCodeAsync(cmd.Code, ct))
            {
                return Results.Conflict(new { message = $"Mã {cmd.Code} đã tồn tại." });
            }

            var id = await repo.CreateAsync(cmd, ct);
            return Results.Created($"/catalogs/{id}", new { id });
        });

        return group;
    }
}
```

### Client Interface (TypeScript 5+)

```typescript
// catalog.model.ts
export type CreateCatalogPayload = {
  readonly code: string;
  readonly name: string;
  readonly price: number;
};

export type CatalogResponse = {
  readonly id: number;
};
```

---

## 4. Verification & Quality Gates

Mọi đoạn mã chuyển đổi phải đạt các tiêu chí sau trước khi bàn giao:

- [ ] **No Legacy Leaks**: Không còn tham chiếu đến `DataSet`, `DataTable`, `SqlDataAdapter`, `System.Windows.Forms`.
- [ ] **C# 12 Compliance**: Sử dụng Primary Constructors trên class/record khi phù hợp, `record` cho DTOs bất biến.
- [ ] **Minimal APIs**: Không dùng Controller cồng kềnh cho các endpoint CRUD đơn giản.
- [ ] **Typed Results**: Endpoint trả về `TypedResults` hoặc `IResult` rõ ràng (`Results.Ok`, `Results.Conflict`, `Results.ValidationProblem`).
- [ ] **Cancellation Token**: Mọi thao tác async I/O đều truyền `CancellationToken`.
- [ ] **Zero UI Code in API**: Loại bỏ hoàn toàn messagebox, alert string hoặc mapping control của WinForms.