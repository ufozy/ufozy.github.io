---
layout: post
title: "Designing Background Services with Hangfire"
subtitle: "A Practical Guide to Background Jobs in ASP.NET Core"
date: 2026-10-28 15:53
author: "Jeff"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - dotnet
  - Hangfire
  - Background Services
---

# 1. Create a Web API Project

Create an ASP.NET Core Web API project:

```bash
dotnet new webapi --use-controllers --name HangfireDemo --framework net10.0
```

Navigate to the project directory:

```bash
cd HangfireDemo
```

# 2. Add the Required NuGet Packages

Add the following package references to the `ItemGroup` element in your project file (`HangfireDemo.csproj`). If the `ItemGroup` element does not exist, create one inside the `Project` element.

```xml
<ItemGroup>
  <PackageReference Include="Hangfire" Version="1.8.25" />
  <PackageReference Include="Hangfire.Dashboard.Basic.Authentication" Version="8.0.0" />
  <PackageReference Include="Microsoft.AspNetCore.OpenApi" Version="10.0.14" />
  <PackageReference Include="Newtonsoft.Json" Version="13.0.4" />
  <PackageReference Include="Swashbuckle.AspNetCore.SwaggerUI" Version="10.3.1" />
  <PackageReference Include="System.Data.SqlClient" Version="4.9.1" />
</ItemGroup>
```

**Security note:** Use Newtonsoft.Json version 13.0.1 or later to address the vulnerability discussed in [GHSA-5crp-9r3c-p9vr](https://github.com/advisories/GHSA-5crp-9r3c-p9vr). The example uses version 13.0.4.

After adding the packages, restore the dependencies:

```bash
dotnet restore
```

# 3. Configure Hangfire

## 3.1 Configure Services in `Program.cs`

Register Hangfire and configure SQL Server as its persistent storage:

```csharp
builder.Services.AddHangfire((sp, config) =>
{
    // Retrieve the connection string from application configuration.
    var connectionString = sp
        .GetRequiredService<IConfiguration>()
        .GetConnectionString("HangfireConnection");
    // Hangfire automatically creates the necessary data tables.
    config.UseSqlServerStorage(connectionString);
});

// Register the Hangfire background job server.
builder.Services.AddHangfireServer();
```

`AddHangfire()` configures Hangfire and its storage, while `AddHangfireServer()` registers a background server that retrieves queued jobs and executes them.

## 3.2 Configure the Hangfire Dashboard

Add the following configuration to `Program.cs`:

```csharp
app.UseHangfireDashboard("/test/job-dashboard", new DashboardOptions
{
    DashboardTitle = "Hangfire Job Dashboard",
    DarkModeEnabled = false,
    DisplayStorageConnectionString = false,

    // Configure dashboard authentication.
    Authorization = new[]
    {
        new HangfireCustomBasicAuthenticationFilter
        {
            User = builder.Configuration["HangfireSettings:User"],
            Pass = builder.Configuration["HangfireSettings:Password"]
        }
    }
});
```

You can access the dashboard at:

```text
https://localhost:<port>/test/job-dashboard
```

Replace `<port>` with the HTTPS port displayed when you run the application.

The dashboard allows you to inspect job states, monitor execution, and troubleshoot failed jobs.


## 3.3 Configure `appsettings.json`

Add the following settings:

```json
{
  "ConnectionStrings": {
    "HangfireConnection": "Server=(localdb)\\MSSQLLocalDB;Database=HangfireDemo;Integrated Security=True;TrustServerCertificate=True;"
  },
  "HangfireSettings": {
    "User": "admin",
    "Password": "ReplaceWithAStrongPassword"
  }
}
```

This example uses SQL Server LocalDB for local development. For production, use an appropriately configured SQL Server instance and store sensitive credentials in a secure configuration provider.

# 4. Demonstrate the Four Types of Background Jobs

Add the following namespace to your controller:

```csharp
using Hangfire;
```

Hangfire provides several ways to schedule and execute background jobs. The following examples demonstrate four commonly used approaches.

## 4.1 `BackgroundJob.Enqueue` — Fire-and-Forget Jobs

### Purpose

Enqueues a job for execution as soon as a background worker can process it. The job does not execute directly on the HTTP request thread.

### Common Use Cases

- Sending emails or notifications
- Generating reports
- Processing uploaded files
- Performing other tasks that do not need to block an HTTP response

### Example

```csharp
BackgroundJob.Enqueue(
    () => Console.WriteLine("Background Job Triggered"));
```

The method returns a job ID without waiting for the job to finish. The background worker executes the job when resources are available.

## 4.2 `BackgroundJob.Schedule` — Delayed Jobs

### Purpose

Schedules a job to become eligible for execution after a specified delay or at a specified time.

### Common Use Cases

- Checking unpaid orders after a certain period
- Sending reminder emails
- Scheduling delayed notifications

### Example

```csharp
BackgroundJob.Schedule(
    () => Console.WriteLine("Scheduled Job Triggered"),
    TimeSpan.FromSeconds(5));
```

This example schedules the job to run after approximately five seconds.

**Note:** A scheduled job is not guaranteed to start at the exact scheduled time. Actual execution depends on the Hangfire scheduler and worker availability.

For business-critical operations, such as canceling unpaid orders, always check the current business state before performing the operation.

## 4.3 `BackgroundJob.ContinueJobWith` — Job Chaining

### Purpose

Creates a continuation job that runs after its parent job reaches a qualifying state. By default, a continuation runs after its parent succeeds.

### Common Use Cases

- Generate an invoice after payment succeeds
- Upload the invoice after it has been generated
- Email the invoice after the upload completes

### Example

```csharp
var parentJobId = BackgroundJob.Enqueue(
    () => Console.WriteLine("Parent Job Triggered"));

var continuationJobId = BackgroundJob.ContinueJobWith(
    parentJobId,
    () => Console.WriteLine("Continuation Job 1 Triggered"));

BackgroundJob.ContinueJobWith(
    continuationJobId,
    () => Console.WriteLine("Continuation Job 2 Triggered"));
```

The intended execution order is:

1. Parent job
2. Continuation job 1
3. Continuation job 2

Each continuation depends on the preceding job satisfying its continuation condition. This approach is useful when tasks must run in sequence and need independent execution states or retry handling.

## 4.4 `RecurringJob.AddOrUpdate` — Recurring Jobs

### Purpose

Schedules a job to run repeatedly according to a Cron expression.

### Common Use Cases

- Generating daily reports
- Cleaning up expired data
- Synchronizing data with third-party systems
- Periodically checking order statuses

### Example

```csharp
RecurringJob.AddOrUpdate(
    "recurring-job",
    () => Console.WriteLine("Recurring Job Triggered"),
    Cron.Minutely());
```

This example schedules the job to run every minute.

The recurring job ID (`"recurring-job"`) identifies the schedule. Calling `AddOrUpdate` with the same ID updates the existing recurring job rather than creating a separate schedule.

Other common Cron helpers include:

| Expression | Schedule |
|---|---|
| `Cron.Hourly()` | Every hour |
| `Cron.Daily()` | Every day |
| `Cron.Weekly()` | Every week |
| `Cron.Monthly()` | Every month |

**Note:** Recurring jobs require a running Hangfire Server to be scheduled and processed. Configure the appropriate time zone when the schedule must follow a specific business or local time.

# 5. Practice code
```C#
using Hangfire;
using Microsoft.AspNetCore.Mvc;

namespace HangfireDemo.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class JobController : ControllerBase
    {
        [HttpPost]
        [Route("CreateBackgroundJob")]
        public ActionResult CreateBackgroundJob()
        {
            BackgroundJob.Enqueue(() => Console.WriteLine("Background Job Triggered"));
            return Ok();
        }

        [HttpPost]
        [Route("CreateScheduledJob")]
        public ActionResult CreateScheduledJob()
        {
            var scheduleDateTime = DateTime.UtcNow.AddSeconds(5);
            var dateTimeOffset = new DateTimeOffset(scheduleDateTime);
            BackgroundJob.Schedule(() => Console.WriteLine($"Scheduled Job Triggered"), dateTimeOffset);
            return Ok();
        }


        [HttpPost]
        [Route("CreateContinuationJob")]
        public ActionResult CreateContinuationJob()
        {
            var parentJobId = BackgroundJob.Enqueue(() => Console.WriteLine("Parent Job Triggered"));
            var continuationJobId = BackgroundJob.ContinueJobWith(parentJobId, () => Console.WriteLine("Continuation Job 1 Triggered"));
            BackgroundJob.ContinueJobWith(continuationJobId, () => Console.WriteLine("Continuation Job 2 Triggered"));
            return Ok();
        }

        [HttpPost]
        [Route("CreateRecurringJob")]
        public ActionResult CreateRecurringJob()
        {
            RecurringJob.AddOrUpdate("Recurring Job", () => Console.WriteLine("Recurring Job Triggered"), Cron.Minutely());
            return Ok();
        }
    }
}

```

# 6. Summary

Hangfire supports several background job patterns for different requirements:

| Job Type | Method | Typical Purpose |
|---|---|---|
| Fire-and-forget | `BackgroundJob.Enqueue` | Execute a task as soon as possible |
| Delayed | `BackgroundJob.Schedule` | Execute a task after a delay |
| Continuation | `BackgroundJob.ContinueJobWith` | Execute dependent tasks in sequence |
| Recurring | `RecurringJob.AddOrUpdate` | Execute tasks periodically |

Choose the job type based on when a task should run, whether it must repeat, and whether it depends on another task.

For production applications, also consider retry behavior, idempotency, secure configuration, and monitoring through the Hangfire Dashboard.
