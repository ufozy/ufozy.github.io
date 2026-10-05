---
layout:     post
title:      "Razor Pages Authentication and Authorization"
subtitle:   "Understanding Cookies, Claims, Policies, and Custom Authorization"
date:       2026-10-05 22:52
author:     "Jeff"
header-img: "img/post-bg-2015.jpg"
catalog:    true
tags:
- Razor Pages
- Authentication
- Authorization
- Claims
---

## `Program.cs`

### Configure the Authentication Scheme

```csharp
builder.Services.AddAuthentication("MyCookieAuth")
    .AddCookie("MyCookieAuth", options =>
    {
        options.Cookie.Name = "MyCookieAuth";
        options.LoginPath = "/Account/Login";
        options.AccessDeniedPath = "/Account/AccessDenied";
        options.LogoutPath = "/Account/Logout";
        options.ExpireTimeSpan = TimeSpan.FromSeconds(10);
    });
```

| Property           | Effect                                                                                               |
| ------------------ | ---------------------------------------------------------------------------------------------------- |
| `Cookie.Name`      | Specifies the name of the authentication cookie.                                                     |
| `LoginPath`        | Specifies the path of the login page.                                                                |
| `AccessDeniedPath` | Specifies the path to redirect to when an authenticated user is not authorized to access a resource. |
| `ExpireTimeSpan`   | Specifies how long the authentication ticket remains valid.                                          |

### Configure Authorization Policies

Policies define authorization rules. A policy can contain built-in requirements or custom authorization requirements.

```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy(
        "AdminOnly",
        policy => policy.RequireClaim("Admin"));

    options.AddPolicy(
        "MustBelongToHRDepartment",
        policy => policy.RequireClaim("Department", "HR"));

    options.AddPolicy(
        "HRManagerOnly",
        policy =>
        {
            policy
                .RequireClaim("Department", "HR")
                .RequireClaim("Manager")
                .AddRequirements(
                    new HRManagerProbationRequirement(3));
        });
});
```

Multiple requirements in the same policy must all be satisfied.

For example, `HRManagerOnly` requires:

```text
Department = HR
        AND
Manager claim exists
        AND
HRManagerProbationRequirement is satisfied
```

### Add the Authentication and Authorization Middleware

```csharp
app.UseAuthentication();
app.UseAuthorization();
```

`UseAuthentication()` identifies the current user.

`UseAuthorization()` checks whether the current user is authorized to access the requested resource.

Authentication should be added before authorization.

---

## Custom Authorization Requirement

### `HRManagerProbationRequirement.cs`

A custom authorization requirement consists of two main parts:

1. A class that implements `IAuthorizationRequirement` and describes the requirement.
2. A handler that inherits from `AuthorizationHandler<T>` and implements the validation logic for that requirement.

```csharp
public class HRManagerProbationRequirement : IAuthorizationRequirement
{
    public HRManagerProbationRequirement(int probationPeriod)
    {
        ProbationPeriod = probationPeriod;
    }

    public int ProbationPeriod { get; }
}
```

The requirement above represents:

> The user must satisfy a probation-period requirement.

For example:

```csharp
new HRManagerProbationRequirement(3)
```

means that the required probation period is 3 months.

The requirement itself does not perform the validation.

The validation logic is implemented by the handler:

```csharp
public class HRManagerProbationRequirementHandler
    : AuthorizationHandler<HRManagerProbationRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        HRManagerProbationRequirement requirement)
    {
        if (!context.User.HasClaim(x => x.Type == "EmploymentDate"))
            return Task.CompletedTask;

        if (DateTime.TryParse(
            context.User.FindFirst(x => x.Type == "EmploymentDate")?.Value,
            out DateTime employmentDate))
        {
            var period = DateTime.Now - employmentDate;

            if (period.Days > 30 * requirement.ProbationPeriod)
            {
                context.Succeed(requirement);
            }
        }

        if (context.User.HasClaim(
                c => c.Type == "Department" && c.Value == "HR") &&
            context.User.HasClaim(
                c => c.Type == "ProbationPeriod" &&
                int.TryParse(c.Value, out int probationPeriod) &&
                probationPeriod >= requirement.ProbationPeriod))
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}
```

`context.Succeed(requirement)` indicates that the current requirement has been satisfied.

`return Task.CompletedTask` only indicates that the handler has finished executing. It does **not** mean that authorization has failed.

For example:

```csharp
return Task.CompletedTask;
```

means:

> This handler did not mark the requirement as successful.

It does not explicitly reject the authorization request.

If a handler calls:

```csharp
context.Fail();
```

the authorization context is explicitly marked as failed.

---

## Create a Cookie Authentication Ticket

### `Login.cshtml.cs`

```csharp
public async Task<IActionResult> OnPostAsync()
{
    if (!ModelState.IsValid)
        return Page();

    if (Credential.UserName == "admin" &&
        Credential.Password == "admin")
    {
        var claims = new List<Claim>
        {
            new Claim(ClaimTypes.Name, Credential.UserName),
            new Claim(ClaimTypes.Email, "admin@my.com"),
            new Claim("Department", "HR"),
            new Claim("Admin", "true"),
            new Claim("Manager", "true"),
            new Claim("EmploymentDate", "2026-01-01"),
        };

        var identity = new ClaimsIdentity(
            claims,
            "MyCookieAuth");

        var claimsPrincipal = new ClaimsPrincipal(identity);

        await HttpContext.SignInAsync(
            "MyCookieAuth",
            claimsPrincipal,
            new AuthenticationProperties
            {
                IsPersistent = Credential.RememberMe
            });

        return RedirectToPage("/Index");
    }

    return Page();
}
```

The main objects have different responsibilities:

```text
Claim
  ↓
Individual piece of information

ClaimsIdentity
  ↓
A collection of claims representing one identity

ClaimsPrincipal
  ↓
A container that can contain one or more identities

Authentication Cookie
  ↓
Stores the authentication ticket
```

For example:

```text
ClaimsPrincipal
    │
    └── ClaimsIdentity
          ├── Name = admin
          ├── Email = admin@my.com
          ├── Department = HR
          ├── Admin = true
          ├── Manager = true
          └── EmploymentDate = 2026-01-01
```

`ClaimsPrincipal` can contain multiple `ClaimsIdentity` objects.

`HttpContext.User.Identity` represents the identity selected by the principal for the current user. In the common case where there is one authenticated identity, it refers to that identity.

---

## Logout

### `Logout.cshtml.cs`

```csharp
public async Task<IActionResult> OnPostAsync()
{
    await HttpContext.SignOutAsync("MyCookieAuth");

    return RedirectToPage("/Index");
}
```

`SignOutAsync()` removes or invalidates the authentication ticket associated with the specified authentication scheme.

Using the scheme explicitly is clearer when multiple authentication schemes are configured:

```csharp
await HttpContext.SignOutAsync("MyCoo
```

## Source Code

The complete source code for this article is available on GitHub:

<https://github.com/ufozy/WebApp_Security>
