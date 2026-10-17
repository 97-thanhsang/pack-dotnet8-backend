---
name: api-contract
description: Design REST API contracts with MethodResult envelopes, request/response models, and endpoint conventions. Use when defining new API endpoints or aligning FE/BE contracts.
---

# SKILL: REST API Contract Design

## Purpose

Defines REST API Contract standards: MethodResult response pattern, Request/Response field-level tables, endpoint naming conventions, HTTP status codes, and contract-first design for BE → FE integration.

## When to Use

- Designing new API endpoints
- Documenting Request/Response contracts for FE consumption
- Ensuring BE-FE contract alignment
- Reviewing API consistency

---

## 1. MethodResult Pattern (Universal Response)

```csharp
// Success with data
public class MethodResult<T>
{
    public bool Status { get; set; }
    public string Message { get; set; } = string.Empty;
    public T? Data { get; set; }
}

// Success without data (void operations)
public class MethodResult
{
    public bool Status { get; set; }
    public string Message { get; set; } = string.Empty;
}

// Factory methods
public static MethodResult<T> Ok(T data, string message = "Thành công")
    => new() { Status = true, Message = message, Data = data };

public static MethodResult Fail(string message)
    => new() { Status = false, Message = message };
```

---

## 2. PagedResult Pattern

```csharp
public class PagedResult<T>
{
    public List<T> Data { get; set; } = new();
    public int Total { get; set; }
}

// Wrapped in MethodResult:
// MethodResult<PagedResult<CatalogListItem>>
```

---

## 3. Endpoint Contract Template

Each endpoint is documented with:

```markdown
### POST /api/v1/catalog/search

**Description:** Tìm kiếm danh mục (phân trang)

**Request Body:**
| Field | Type | Required | Description |
|---|---|---|---|
| pageIndex | int | No (default: 1) | Trang hiện tại |
| pageSize | int | No (default: 20) | Số lượng/trang |
| keyword | string | No | Từ khóa tìm kiếm |
| status | boolean | No | Lọc theo trạng thái |
| sortColumn | string | No | Tên cột sắp xếp |
| sortDirection | string | No | ASC/DESC |

**Response (200):**
| Field | Type | Description |
|---|---|---|
| status | boolean | true = thành công |
| message | string | Thông báo |
| data | object | |
| data[].id | int | Mã định danh |
| data[].code | string | Mã danh mục |
| data[].name | string | Tên danh mục |
| data[].status | boolean | Trạng thái |
| total | int | Tổng số bản ghi |
```

---

## 4. Endpoint Contract Table

| Method | Route | Request | Response | Status |
|---|---|---|---|---|
| POST | `api/v1/catalog/search` | `SearchCatalogQuery` | `PagedResult<CatalogListItem>` | 200 |
| GET | `api/v1/catalog/{id}` | Route param | `CatalogDetail` | 200 / 404 |
| POST | `api/v1/catalog` | `CreateCatalogCommand` | `MethodResult<int>` | 201 / 400 |
| PUT | `api/v1/catalog/{id}` | `UpdateCatalogCommand` | `MethodResult` | 200 / 400 |
| DELETE | `api/v1/catalog/{id}` | Route param | `MethodResult` | 200 / 404 |
| GET | `api/v1/catalog/lookup` | — | `List<CatalogLookup>` | 200 |

---

## 5. BE → FE Contract Alignment Rules

1. **BE thiết kế endpoint trước** → xác định Request/Response
2. **FE dùng chính xác contract đó** → không tự suy luận field
3. **Mọi response đều wrap trong MethodResult** → FE check `res.status`
4. **Response field phải đủ cho FE hiển thị** (ví dụ: bao gồm `categoryName`, `createdByName` — không chỉ id)
5. **CamelCase JSON** (`createdDate` not `CreatedDate`)
6. **DateTime format**: ISO 8601 (`yyyy-MM-ddTHH:mm:ss`)

---

## 6. Status Code Convention

| Code | When | Response |
|---|---|---|
| 200 | Success | `MethodResult<T>` |
| 201 | Created | `MethodResult<T>` with Location header |
| 400 | Validation/Business error | `MethodResult` with error message |
| 401 | Unauthenticated | — |
| 403 | Unauthorized | — |
| 404 | Not found | `MethodResult` with message |
| 409 | Duplicate | `MethodResult` with message |
| 500 | Server error | `MethodResult` (generic message) |
