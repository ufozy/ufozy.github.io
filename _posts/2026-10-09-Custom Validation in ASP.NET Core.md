---
layout: post
title: "Custom Validation in ASP.NET Core"
subtitle: "Creating Custom Validation Attributes with ValidationAttribute"
date: 2026-10-09 10:50
author: "Jeff"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
- ASP.NET Core
- Data Annotations
- ValidationAttribute
---

# Custom Validation in ASP.NET Core

ASP.NET Core provides many built-in validation attributes through the `System.ComponentModel.DataAnnotations` namespace.

For more information, see the official documentation:

[System.ComponentModel.DataAnnotations Namespace](https://learn.microsoft.com/en-us/dotnet/api/system.componentmodel.dataannotations?view=net-10.0)

## 1. How to Create a Custom Validation Attribute

To define a custom validation attribute:

1. Create a class that inherits from `ValidationAttribute`.
2. Override the `IsValid` method to implement your validation logic.
3. Return a `ValidationResult` containing an error message if validation fails, or `ValidationResult.Success` if validation succeeds.

### Example: Date Expiration Validation

The following example defines a custom validation attribute that checks whether a date is earlier than a specified number of months from the current date.

```csharp
using System.ComponentModel.DataAnnotations;

public class DateExpireValidation : ValidationAttribute
{
    private readonly int _expireMonth;

    public DateExpireValidation(int expireMonth)
    {
        _expireMonth = expireMonth;
    }

    protected override ValidationResult? IsValid(
        object? value,
        ValidationContext validationContext)
    {
        if (value is DateTime dateValue &&
            dateValue < DateTime.Now.AddMonths(_expireMonth))
        {
            return new ValidationResult(ErrorMessage);
        }

        return ValidationResult.Success;
    }
}
```

### How It Works

* `ValidationAttribute` is the base class for custom validation attributes.
* `_expireMonth` specifies the number of months used in the validation rule.
* `IsValid()` contains the custom validation logic.
* `ErrorMessage` provides the validation error message.
* `ValidationResult.Success` indicates that validation has passed.

## 2. Understanding ValidationContext

The `ValidationContext` parameter provides information about the object being validated and the validation environment.

For example:

```csharp
object instance = validationContext.ObjectInstance;
```

The `ObjectInstance` property returns the object that contains the property being validated.

This is useful when a validation rule needs to access other properties of the same object, rather than validating only the current property's value.

## 3. Using the Custom Validation Attribute

You can apply the custom attribute to a model property:

```csharp
public class Project
{
    [DateExpireValidation(
        3,
        ErrorMessage = "The date must be at least three months in the future.")]
    public DateTime DueDate { get; set; }
}
```

In this example, the validation fails if `DueDate` is earlier than three months from the current date.