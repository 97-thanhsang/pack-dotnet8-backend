---
name: database-schema
description: Design relational database schemas, SP_V2_EDU_* stored procedures, pagination, and JOIN patterns. Use when designing tables or optimizing stored procedures.
---

# SKILL: Database Schema Design

## Purpose

Defines database schema design patterns for .NET 8 projects: stored procedure naming conventions (SP_V2_EDU_*), pagination patterns, JOIN optimization, and legacy SP migration.

## When to Use

- Creating new stored procedures
- Migrating legacy SPs to V2 format
- Designing paginated queries
- Optimizing JOIN performance
- Designing table schemas for new features

---

## 1. Stored Procedure Naming Convention

```
SP_V2_EDU_{MODULE}_{ACTION}[_SUFFIX]

Examples:
SP_V2_EDU_CATALOG_SEARCH      — Paginated search
SP_V2_EDU_CATALOG_GETBYID     — Single record
SP_V2_EDU_CATALOG_CREATE      — INSERT
SP_V2_EDU_CATALOG_UPDATE      — UPDATE
SP_V2_EDU_CATALOG_DELETE      — DELETE (soft)
SP_V2_EDU_CATALOG_CHECKDUPLICATE — Validation
SP_V2_EDU_CATALOG_GETLOOKUP   — Dropdown data source
```

**Rules:**
- `SP_` prefix for all stored procedures
- `V2_` indicates new architecture
- `EDU_` module prefix (adjust per domain)
- `{MODULE}` matches the domain/feature name
- `{ACTION}` is the operation: SEARCH, GETBYID, CREATE, UPDATE, DELETE

---

## 2. Paginated Search SP Pattern

```sql
CREATE PROCEDURE SP_V2_EDU_CATALOG_SEARCH
    @PageIndex INT = 1,
    @PageSize INT = 20,
    @SortColumn NVARCHAR(50) = 'CreatedDate',
    @SortDirection NVARCHAR(4) = 'DESC',
    @Keyword NVARCHAR(200) = NULL,
    @Status BIT = NULL
AS
BEGIN
    SET NOCOUNT ON;

    SELECT 
        c.Id,
        c.Code,
        c.Name,
        c.Status,
        c.CreatedDate,
        c.UpdatedDate,
        TotalCount = COUNT(*) OVER()
    FROM Catalog c
    WHERE (@Keyword IS NULL OR c.Name LIKE N'%' + @Keyword + '%' OR c.Code LIKE @Keyword + '%')
        AND (@Status IS NULL OR c.Status = @Status)
    ORDER BY
        CASE WHEN @SortColumn = 'Code' AND @SortDirection = 'ASC' THEN c.Code END ASC,
        CASE WHEN @SortColumn = 'Code' AND @SortDirection = 'DESC' THEN c.Code END DESC,
        CASE WHEN @SortColumn = 'CreatedDate' AND @SortDirection = 'ASC' THEN c.CreatedDate END ASC,
        CASE WHEN @SortColumn = 'CreatedDate' AND @SortDirection = 'DESC' THEN c.CreatedDate END DESC
    OFFSET (@PageIndex - 1) * @PageSize ROWS
    FETCH NEXT @PageSize ROWS ONLY;
END
```

**Key patterns:**
- `COUNT(*) OVER()` for total count without separate query
- `OFFSET/FETCH NEXT` for pagination (not `ROW_NUMBER()`)
- `CASE WHEN @SortColumn` for dynamic sorting
- `SET NOCOUNT ON` for performance
- Parameterized filters with NULL checks

---

## 3. Search SP with JOINs

```sql
CREATE PROCEDURE SP_V2_EDU_CATALOG_SEARCH
    @PageIndex INT = 1,
    @PageSize INT = 20
AS
BEGIN
    SET NOCOUNT ON;

    SELECT 
        c.Id, c.Code, c.Name, c.Status,
        cat.Name AS CategoryName,
        u.FullName AS CreatedByName,
        TotalCount = COUNT(*) OVER()
    FROM Catalog c
    LEFT JOIN Category cat ON c.CategoryId = cat.Id
    LEFT JOIN Users u ON c.CreatedBy = u.Id
    WHERE c.Status = 1
    ORDER BY c.CreatedDate DESC
    OFFSET (@PageIndex - 1) * @PageSize ROWS
    FETCH NEXT @PageSize ROWS ONLY;
END
```

**JOIN Rules:**
- Use `LEFT JOIN` for optional relationships
- Use `INNER JOIN` for required relationships
- Prefix all selected columns with table alias
- Always include `TotalCount = COUNT(*) OVER()`

---

## 4. Create SP Pattern (with OUTPUT)

```sql
CREATE PROCEDURE SP_V2_EDU_CATALOG_CREATE
    @Code NVARCHAR(50),
    @Name NVARCHAR(200),
    @Description NVARCHAR(500) = NULL,
    @CreatedBy INT
AS
BEGIN
    SET NOCOUNT ON;

    -- Check duplicate
    IF EXISTS (SELECT 1 FROM Catalog WHERE Code = @Code)
    BEGIN
        SELECT Status = 0, Message = N'Mã đã tồn tại', Data = NULL;
        RETURN;
    END

    INSERT INTO Catalog (Code, Name, Description, Status, CreatedBy, CreatedDate)
    VALUES (@Code, @Name, @Description, 1, @CreatedBy, GETDATE());

    SELECT Status = 1, Message = N'Thành công', Data = SCOPE_IDENTITY();
END
```

**Key patterns:**
- Check duplicate before insert
- Return MethodResult format: `Status`, `Message`, `Data`
- Use `SCOPE_IDENTITY()` for new ID
- Parameterize all inputs (no string concatenation)

---

## 5. Update SP Pattern

```sql
CREATE PROCEDURE SP_V2_EDU_CATALOG_UPDATE
    @Id INT,
    @Code NVARCHAR(50),
    @Name NVARCHAR(200),
    @Status BIT,
    @UpdatedBy INT
AS
BEGIN
    SET NOCOUNT ON;

    -- Check duplicate (exclude self)
    IF EXISTS (SELECT 1 FROM Catalog WHERE Code = @Code AND Id <> @Id)
    BEGIN
        SELECT Status = 0, Message = N'Mã đã tồn tại', Data = NULL;
        RETURN;
    END

    UPDATE Catalog
    SET Code = @Code,
        Name = @Name,
        Status = @Status,
        UpdatedBy = @UpdatedBy,
        UpdatedDate = GETDATE()
    WHERE Id = @Id;

    SELECT Status = 1, Message = N'Cập nhật thành công', Data = @Id;
END
```

---

## 6. Legacy SP Migration Checklist

- [ ] Read existing SP_XXX_XXX from legacy code
- [ ] Identify business logic vs infrastructure code
- [ ] Add pagination (OFFSET/FETCH NEXT)
- [ ] Add dynamic sorting (CASE WHEN)
- [ ] Add parameterized filters (NULL checks)
- [ ] Rename to SP_V2_EDU_{MODULE}_{ACTION}
- [ ] Add MethodResult return format
- [ ] Add duplicate check for CREATE/UPDATE
- [ ] Add TotalCount for paginated queries
- [ ] Add JOINs for display fields (category, creator names)

---

## 7. Normalization (1NF → 2NF → 3NF)

### 7.1 First Normal Form (1NF)

| Rule | Violation Example |
|:-----|:------------------|
| Atomic values, no repeating groups | `product_ids = '1,2,3'` |

```sql
-- BAD: Multiple values in one column
CREATE TABLE Orders (Id INT, ProductIds VARCHAR(255));  -- '101,102,103'

-- GOOD: Separate child table
CREATE TABLE Orders (Id INT IDENTITY PRIMARY KEY);
CREATE TABLE OrderItems (Id INT IDENTITY PRIMARY KEY, OrderId INT NOT NULL REFERENCES Orders(Id));
```

### 7.2 Second Normal Form (2NF)

| Rule | Violation Example |
|:-----|:------------------|
| No partial dependencies | `customer_name` in `order_items` depends only on `customer_id` |

```sql
-- BAD: customer_name depends on customer_id (partial key dependency)
CREATE TABLE OrderItems (OrderId INT, ProductId INT, CustomerName NVARCHAR(100), PRIMARY KEY (OrderId, ProductId));

-- GOOD: Separate Customers table
CREATE TABLE Customers (Id INT IDENTITY PRIMARY KEY, Name NVARCHAR(100));
```

### 7.3 Third Normal Form (3NF)

| Rule | Violation Example |
|:-----|:------------------|
| No transitive dependencies | `country` derived from `postal_code` |

```sql
-- BAD: country depends on postal_code (transitive dependency)
CREATE TABLE Customers (Id INT IDENTITY PRIMARY KEY, PostalCode NVARCHAR(10), Country NVARCHAR(50));

-- GOOD: Lookup table
CREATE TABLE PostalCodes (Code NVARCHAR(10) PRIMARY KEY, Country NVARCHAR(50));
```

### 7.4 When to Denormalize

| Scenario | Strategy |
|:---------|:---------|
| Read-heavy reporting | Pre-calculated aggregates |
| Expensive JOINs | Cached derived columns |
| Analytics dashboards | Indexed views |

---

## 8. Indexing Strategy (SQL Server)

### 8.1 Index Decision Table

| Query Pattern | Index Type | SQL Server Syntax |
|:--------------|:-----------|:------------------|
| FK column JOINs | NONCLUSTERED | `CREATE NONCLUSTERED INDEX IX_Table_FkId ON Table(FkId)` |
| WHERE + ORDER BY | Composite | `CREATE NONCLUSTERED INDEX IX_Table_Col1_Col2 ON Table(Col1, Col2)` |
| Unique constraint | UNIQUE | `CREATE UNIQUE NONCLUSTERED INDEX IX_Table_Code ON Table(Code)` |
| Full-text search | FULLTEXT | `CREATE FULLTEXT CATALOG FT_Catalog AS DEFAULT; CREATE FULLTEXT INDEX ON Table(Col) KEY INDEX PK` |
| High-write table | COLUMNSTORE | `CREATE CLUSTERED COLUMNSTORE INDEX IX_Table ON Table` |

### 8.2 Composite Index Rule

```sql
-- Most selective column FIRST
CREATE NONCLUSTERED INDEX IX_Orders_CustomerId_Status ON Orders(CustomerId, Status);

-- Uses index: WHERE CustomerId = 123
-- Uses index: WHERE CustomerId = 123 AND Status = 'pending'
-- Does NOT use index: WHERE Status = 'pending'  (leading column missing)
```

### 8.3 Index Maintenance

```sql
-- Rebuild fragmented index
ALTER INDEX ALL ON Table REBUILD;

-- Update statistics
UPDATE STATISTICS Table;
EXEC sp_updatestats;
```

---

## 9. Relationship Patterns (SQL Server)

| Type | Pattern |
|:-----|:--------|
| **One-to-Many** | FK on child table + NONCLUSTERED INDEX on FK |
| **Many-to-Many** | Junction table + composite PK + FK indexes |
| **Self-Referencing** | FK referencing same table PK |

```sql
-- One-to-Many
CREATE TABLE Orders (Id INT IDENTITY PRIMARY KEY, CustomerId INT NOT NULL);
CREATE NONCLUSTERED INDEX IX_Orders_CustomerId ON Orders(CustomerId);

-- Many-to-Many
CREATE TABLE Enrollments (StudentId INT, CourseId INT, PRIMARY KEY (StudentId, CourseId));
CREATE NONCLUSTERED INDEX IX_Enrollments_CourseId ON Enrollments(CourseId);

-- Self-Referencing
CREATE TABLE Employees (Id INT IDENTITY PRIMARY KEY, ManagerId INT REFERENCES Employees(Id));
```

---

## 10. Migration Patterns (Zero-Downtime)

### 10.1 Adding a Column

```sql
-- Step 1: Add nullable
ALTER TABLE Users ADD Phone NVARCHAR(20) NULL;
-- Step 2: Deploy code that writes to new column
-- Step 3: Backfill
UPDATE Users SET Phone = '' WHERE Phone IS NULL;
-- Step 4: Make required
ALTER TABLE Users ALTER COLUMN Phone NVARCHAR(20) NOT NULL;
```

### 10.2 Reversible Migration Template

```sql
-- UP
BEGIN TRAN;
ALTER TABLE Users ADD Phone NVARCHAR(20) NULL;
CREATE NONCLUSTERED INDEX IX_Users_Phone ON Users(Phone);
COMMIT;

-- DOWN
BEGIN TRAN;
DROP INDEX IX_Users_Phone ON Users;
ALTER TABLE Users DROP COLUMN Phone;
COMMIT;
```

---

## 11. Schema Verification Checklist

- [ ] Every table has a primary key (IDENTITY)
- [ ] All FKs have NONCLUSTERED indexes
- [ ] ON DELETE strategy defined (CASCADE / RESTRICT / SET NULL)
- [ ] Indexes match query patterns (WHERE + ORDER BY)
- [ ] DECIMAL(18,2) for money, not FLOAT
- [ ] NOT NULL on required fields
- [ ] UNIQUE constraints where needed
- [ ] created_at + updated_at with DEFAULT GETDATE()
- [ ] Migration scripts reversible (UP + DOWN)
- [ ] SP naming follows SP_V2_EDU_* convention
- [ ] No SELECT * in SPs
