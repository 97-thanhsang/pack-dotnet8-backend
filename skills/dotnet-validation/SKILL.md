---
name: dotnet-validation
description: Implement input validation with FluentValidation validators, custom business rules, and error envelopes in .NET 8. Use when validating command or query payloads.
---

# SKILL: .NET 8 Validation Patterns (FluentValidation)

## Purpose

Defines FluentValidation patterns for .NET 8 projects: command/query validators, custom validation rules, cross-field validation, Vietnamese error messages.

## When to Use

- Adding validation to Command/Query objects
- Implementing complex cross-field validation
- Customizing validation error messages
- Reusable validation rule sets

---

## 1. Command Validator Pattern

```csharp
public class CreateCatalogCommandValidator : AbstractValidator<CreateCatalogCommand>
{
    public CreateCatalogCommandValidator()
    {
        RuleFor(x => x.Code)
            .NotEmpty().WithMessage("Mã không được để trống")
            .MaximumLength(50).WithMessage("Mã không quá 50 ký tự");

        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Tên không được để trống")
            .MaximumLength(200).WithMessage("Tên không quá 200 ký tự");

        RuleFor(x => x.CreatedBy)
            .GreaterThan(0).WithMessage("Người tạo không hợp lệ");
    }
}
```

---

## 2. Update Validator Pattern

```csharp
public class UpdateCatalogCommandValidator : AbstractValidator<UpdateCatalogCommand>
{
    public UpdateCatalogCommandValidator()
    {
        RuleFor(x => x.Id)
            .GreaterThan(0).WithMessage("Id không hợp lệ");

        RuleFor(x => x.Code)
            .NotEmpty().WithMessage("Mã không được để trống")
            .MaximumLength(50).WithMessage("Mã không quá 50 ký tự");

        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Tên không được để trống");
    }
}
```

---

## 3. Cross-Field Validation

```csharp
public class DateRangeValidator : AbstractValidator<DateRangeQuery>
{
    public DateRangeValidator()
    {
        RuleFor(x => x.FromDate)
            .NotEmpty().WithMessage("Từ ngày không được để trống");

        RuleFor(x => x.ToDate)
            .NotEmpty().WithMessage("Đến ngày không được để trống");

        RuleFor(x => x)
            .Must(x => x.ToDate >= x.FromDate)
            .WithMessage("Đến ngày phải lớn hơn hoặc bằng từ ngày");
    }
}
```

---

## 4. Custom Validation Rules

```csharp
public static class CustomValidators
{
    // Validate code format (alphanumeric + underscore)
    public static IRuleBuilderOptions<T, string> ValidCode<T>(
        this IRuleBuilder<T, string> ruleBuilder)
    {
        return ruleBuilder
            .Matches(@"^[a-zA-Z0-9_]+$")
            .WithMessage("Mã chỉ bao gồm chữ, số và dấu gạch dưới");
    }

    // Validate phone number
    public static IRuleBuilderOptions<T, string> ValidPhone<T>(
        this IRuleBuilder<T, string> ruleBuilder)
    {
        return ruleBuilder
            .Matches(@"^0[0-9]{9,10}$")
            .WithMessage("Số điện thoại không hợp lệ");
    }
}

// Usage:
// RuleFor(x => x.Code).ValidCode();
```

---

## 5. Query Validator Pattern

```csharp
public class SearchCatalogQueryValidator : AbstractValidator<SearchCatalogQuery>
{
    public SearchCatalogQueryValidator()
    {
        RuleFor(x => x.PageIndex)
            .GreaterThanOrEqualTo(1).WithMessage("Trang phải lớn hơn 0");

        RuleFor(x => x.PageSize)
            .InclusiveBetween(1, 200).WithMessage("Số lượng phải từ 1 đến 200");
    }
}
```

---

## 6. Pipeline Registration

```csharp
// Program.cs or ServicesExtensions
services.AddValidatorsFromAssemblyContaining<CreateCatalogCommandValidator>();
services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
```
