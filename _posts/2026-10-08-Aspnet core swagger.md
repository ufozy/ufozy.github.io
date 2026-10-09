---
layout:     post
title:      "How to Use Swagger in ASP.NET Core 9 and Later"
subtitle:   "Understanding OpenAPI and Swagger UI in ASP.NET Core 9+"
date:       2026-10-09 10:50
author:     "Jeff"
header-img: "img/post-bg-2015.jpg"
catalog:    true
tags:
- asp.net core
- OpenAPI
- Swagger
---

# How to Use Swagger in ASP.NET Core 9 and Later

Starting with .NET 9, ASP.NET Core provides built-in support for generating OpenAPI documents. However, Swagger UI is not included by default in the standard Web API template.

There are two common ways to use Swagger UI in ASP.NET Core 9 and later.

## 1. Method One: Use the Built-in OpenAPI Support

In this approach, ASP.NET Core generates the OpenAPI document, and Swagger UI provides a web interface for exploring and testing API endpoints.

### Step 1: Install the NuGet Package

Install the `Swashbuckle.AspNetCore.SwaggerUI` package.

```powershell
Install-Package Swashbuckle.AspNetCore.SwaggerUI
```

### Step 2: Configure `Program.cs`

Make sure OpenAPI generation is configured in your application:

```csharp
builder.Services.AddOpenApi();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();

    app.UseSwaggerUI(options =>
    {
        options.SwaggerEndpoint(
            "/openapi/v1.json",
            "My API v1");
    });
}

app.Run();
```

The key methods are:

* `AddOpenApi()` registers the services required to generate OpenAPI documents.
* `MapOpenApi()` exposes the generated OpenAPI document at `/openapi/v1.json` by default.
* `UseSwaggerUI()` serves Swagger UI and configures it to load the OpenAPI document.

**Note:** If your project already calls `AddOpenApi()` and `MapOpenApi()`, you only need to install the Swagger UI package and configure the UI middleware.

## 2. Method Two: Use Swashbuckle to Generate OpenAPI Documents

You can also use Swashbuckle as in earlier ASP.NET Core projects. Swashbuckle generates the OpenAPI document and provides Swagger UI.

### Step 1: Install the NuGet Package

```powershell
Install-Package Swashbuckle.AspNetCore
```

### Step 2: Configure `Program.cs`

```csharp
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.Run();
```

The key methods are:

* `AddEndpointsApiExplorer()` registers API endpoint metadata services used by API description and OpenAPI tooling, particularly for Minimal APIs.
* `AddSwaggerGen()` registers Swashbuckle's OpenAPI document generator.
* `UseSwagger()` exposes the generated OpenAPI document.
* `UseSwaggerUI()` serves the Swagger UI interface.

## 3. History and Key Differences

In earlier ASP.NET Core project templates, including the .NET 8 Web API template, Swashbuckle was commonly included by default.

Starting with .NET 9, the standard Web API template uses built-in OpenAPI support instead of including Swashbuckle by default.

OpenAPI and Swagger UI serve different purposes:

* **OpenAPI** is a specification for describing HTTP APIs in a machine-readable format.
* **OpenAPI document generation** produces a document describing API endpoints, parameters, request bodies, and responses.
* **Swagger UI** is a web interface that displays an OpenAPI document and allows users to explore and test API endpoints.

Therefore, you do not have to use Swashbuckle to generate OpenAPI documents. You can use ASP.NET Core's built-in OpenAPI support and add Swagger UI separately.

Both approaches are valid. Choose the built-in OpenAPI support if you want to follow the default approach in newer ASP.NET Core projects, or use Swashbuckle if you prefer its document-generation features and configuration options.
