# Getting Started – Approval Service
## Overview
The Approval Service exposes the core workflow APIs used by the platform to create, review, and approve requests. It persists approval data in SQL Server, publishes workflow events over RabbitMQ, and emits telemetry to Seq.

### نظرة عامة
توفر خدمة الموافقات واجهات برمجة التطبيقات الأساسية لمنصة الطلبات من أجل إنشاء الطلبات ومراجعتها واعتمادها. تقوم بتخزين بيانات الموافقات في قاعدة بيانات SQL Server، وترسل أحداث سير العمل عبر RabbitMQ، كما ترسل القياسات إلى نظام Seq.

## Prerequisites
- [.NET SDK 9.0](https://dotnet.microsoft.com/en-us/download)
- Local or containerized dependencies: SQL Server, RabbitMQ, and Seq.
- Access to Azure AD application settings when running authentication flows locally.

### المتطلبات الأساسية
- [حزمة تطوير .NET 9.0](https://dotnet.microsoft.com/en-us/download)
- توفر خدمات SQL Server وRabbitMQ وSeq محليًا أو داخل حاويات.
- بيانات إعداد تطبيق Azure AD عند تشغيل تدفقات المصادقة محليًا.

## Run the service locally with the .NET CLI
1. Start the dependent infrastructure (SQL Server, RabbitMQ, Seq). You can spin them up with Docker Compose:
   ```bash
   docker compose up -d sqlserver rabbitmq seq
   ```
2. Apply the latest database migrations if required.
   ```bash
   dotnet ef database update --project src/ApprovalService/ApprovalService.csproj
   ```
3. Export the environment variables that match your setup. The values below mirror the compose file:
   ```bash
   export ASPNETCORE_ENVIRONMENT=Development
   export ASPNETCORE_URLS=http://localhost:7001
   export RabbitMq__Host=localhost
   export RabbitMq__Username=mtuser
   export RabbitMq__Password=StrongP@ss123
   export ConnectionStrings__DefaultConnection="Data Source=localhost,1433;Initial Catalog=approvals;User ID=sa;Password=Edl@1234;TrustServerCertificate=True;"
   export ProfileService__BaseUrl=http://localhost:7006
   export Serilog__WriteTo__0__Name=Seq
   export Serilog__WriteTo__0__Args__serverUrl=http://localhost:5341
   ```
   Configure the Azure AD settings (`ApiIdentifier`, `AzureAd__TenantId`, etc.) via user secrets or exported variables if you need to validate JWTs locally.
4. Run the service.
   ```bash
   dotnet run --project src/ApprovalService/ApprovalService.csproj
   ```
5. The API will be reachable at `http://localhost:7001` by default.

### تشغيل الخدمة محليًا عبر سطر أوامر .NET
1. شغّل الخدمات الداعمة (SQL Server وRabbitMQ وSeq). يمكنك تشغيلها بواسطة Docker Compose:
   ```bash
   docker compose up -d sqlserver rabbitmq seq
   ```
2. طبّق آخر ترحيلات قاعدة البيانات عند الحاجة.
   ```bash
   dotnet ef database update --project src/ApprovalService/ApprovalService.csproj
   ```
3. صدّر متغيرات البيئة بما يتوافق مع بيئتك. القيم التالية تطابق ملف `docker-compose.yml`:
   ```bash
   export ASPNETCORE_ENVIRONMENT=Development
   export ASPNETCORE_URLS=http://localhost:7001
   export RabbitMq__Host=localhost
   export RabbitMq__Username=mtuser
   export RabbitMq__Password=StrongP@ss123
   export ConnectionStrings__DefaultConnection="Data Source=localhost,1433;Initial Catalog=approvals;User ID=sa;Password=Edl@1234;TrustServerCertificate=True;"
   export ProfileService__BaseUrl=http://localhost:7006
   export Serilog__WriteTo__0__Name=Seq
   export Serilog__WriteTo__0__Args__serverUrl=http://localhost:5341
   ```
   قم بضبط إعدادات Azure AD مثل `ApiIdentifier` و`AzureAd__TenantId` عبر أسرار المستخدم أو متغيرات البيئة إذا كنت تريد اختبار التحقق من الرموز المميزة محليًا.
4. شغّل الخدمة:
   ```bash
   dotnet run --project src/ApprovalService/ApprovalService.csproj
   ```
5. ستجد واجهات الخدمة متاحة على `http://localhost:7001` افتراضيًا.

## Run the service with Docker
- Build and start the container in isolation:
  ```bash
  docker compose up -d approval-svc
  ```
- The compose file automatically links the container to SQL Server, RabbitMQ, and Seq and exposes port `7001` on the host.

### تشغيل الخدمة باستخدام Docker
- لبناء الحاوية وتشغيلها بشكل منفصل:
  ```bash
  docker compose up -d approval-svc
  ```
- يقوم ملف Compose بربط الحاوية تلقائيًا مع SQL Server وRabbitMQ وSeq ويكشف المنفذ `7001` على المضيف.

## Integrating with other services
### Call the HTTP API from ASP.NET Core
Register a typed `HttpClient` so other services can call `api/approvals` with automatic retries. Install the `Polly.Extensions.Http` package and configure the client during startup:

```csharp
using Polly;
using Polly.Extensions.Http;

builder.Services
    .AddHttpClient<IApprovalsClient, ApprovalsClient>(client =>
    {
        client.BaseAddress = new Uri(configuration["Services:Approvals:BaseUrl"] ?? "http://approval-svc");
        client.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));
    })
    .AddPolicyHandler(HttpPolicyExtensions
        .HandleTransientHttpError()
        .WaitAndRetryAsync(
            configuration.GetValue("HttpClient:RetryCount", 3),
            retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt))));

public interface IApprovalsClient
{
    Task<WorkflowSummaryDto?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
}

public sealed class ApprovalsClient : IApprovalsClient
{
    private readonly HttpClient _http;

    public ApprovalsClient(HttpClient http) => _http = http;

    public async Task<WorkflowSummaryDto?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default)
    {
        using var response = await _http.GetAsync($"/api/approvals/{id}", cancellationToken);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<WorkflowSummaryDto>(cancellationToken: cancellationToken);
    }
}
```

Inject `IApprovalsClient` where you need it and call `await approvalsClient.GetByIdAsync(workflowId);`. The retry policy shields callers from transient HTTP or network faults.

### التكامل مع الخدمات الأخرى
#### استدعاء واجهات HTTP من تطبيق ASP.NET Core
سجّل عميلًا مخصصًا باستخدام `HttpClientFactory` وPolly لتضمين محاولات إعادة المحاولة عند استدعاء واجهة `api/approvals`:

```csharp
builder.Services
    .AddHttpClient<IApprovalsClient, ApprovalsClient>(client =>
    {
        client.BaseAddress = new Uri(configuration["Services:Approvals:BaseUrl"] ?? "http://approval-svc");
        client.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));
    })
    .AddPolicyHandler(HttpPolicyExtensions
        .HandleTransientHttpError()
        .WaitAndRetryAsync(
            configuration.GetValue("HttpClient:RetryCount", 3),
            retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt))));
```

بعد تسجيل العميل يمكنك حقنه واستدعاء `GetByIdAsync` للحصول على تفاصيل الموافقة مع معالجة أعطال الشبكة المؤقتة تلقائيًا.

### Publish workflow notifications over RabbitMQ
Approval Service publishes `SendEmail` and `NotificationMessage` events after each state transition. Subscribe with MassTransit to react to approvals in your own service:

```csharp
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<WorkflowNotificationConsumer>();

    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host(configuration["RabbitMq:Host"], "/", h =>
        {
            h.Username(configuration.GetValue("RabbitMq:Username", "guest")!);
            h.Password(configuration.GetValue("RabbitMq:Password", "guest")!);
        });

        cfg.ReceiveEndpoint("approval-notification", endpoint =>
        {
            endpoint.ConfigureConsumer<WorkflowNotificationConsumer>(context);
        });
    });
});

public sealed class WorkflowNotificationConsumer : IConsumer<NotificationMessage>
{
    public Task Consume(ConsumeContext<NotificationMessage> context)
    {
        // Update dashboards, push to SignalR, etc.
        return Task.CompletedTask;
    }
}
```

You can also trigger the Email Service manually by publishing a `SendEmail` record using `IPublishEndpoint`:

```csharp
await publishEndpoint.Publish(new SendEmail
{
    To = approverEmail,
    Subject = "Approval reminder",
    Id = workflowId.ToString(),
    BodyHtml = "Please review the pending approval.",
    LinkText = "Open workflow",
    LinkUrl = approvalLink,
    BackgroundColor = "#FFA73B"
});
```

### نشر إشعارات سير العمل عبر RabbitMQ
ترسل خدمة الموافقات رسائل `SendEmail` و`NotificationMessage` عند تغير حالة سير العمل. يمكنك الاشتراك بها باستخدام MassTransit كما في المثال السابق ثم التعامل معها لتحديث تطبيقك أو إرسال تنبيهات إضافية. كما يمكن نشر رسالة `SendEmail` عبر `IPublishEndpoint` لتحفيز خدمة البريد الإلكتروني عند الحاجة.

## Configuration reference
| Setting | Description |
| --- | --- |
| `RabbitMq__Host`, `RabbitMq__Username`, `RabbitMq__Password` | Broker configuration for MassTransit message publishing. |
| `ConnectionStrings__DefaultConnection` | SQL Server connection used by Entity Framework Core. |
| `ProfileService__BaseUrl` | Base URL for retrieving profile data. |
| `Serilog__WriteTo__0__Args__serverUrl` | Seq ingestion endpoint for structured logs. |
| `ApiIdentifier`, `AzureAd__*` | Azure AD configuration for validating JWT bearer tokens. |

### مرجع الإعدادات
| الإعداد | الوصف |
| --- | --- |
| `RabbitMq__Host`، `RabbitMq__Username`، `RabbitMq__Password` | إعدادات الوسيط المستخدمة في نشر الرسائل عبر MassTransit. |
| `ConnectionStrings__DefaultConnection` | اتصال SQL Server المستخدم بواسطة Entity Framework Core. |
| `ProfileService__BaseUrl` | عنوان الخدمة المسؤول عن جلب بيانات الملفات الشخصية. |
| `Serilog__WriteTo__0__Args__serverUrl` | عنوان إدخال سجلات Seq. |
| `ApiIdentifier` و`AzureAd__*` | إعدادات Azure AD للتحقق من الرموز المميزة لنظام المصادقة.

## Useful commands
- Run unit tests:
  ```bash
  dotnet test tests/ApprovalService.Unit.Tests/ApprovalService.Unit.Tests.csproj
  ```
- Run integration tests (requires dependencies):
  ```bash
  dotnet test tests/ApprovalService.IntegrationTests/ApprovalService.IntegrationTests.csproj
  ```
- Format the source:
  ```bash
  dotnet format src/ApprovalService/ApprovalService.csproj
  ```

### أوامر مفيدة
- تشغيل اختبارات الوحدة:
  ```bash
  dotnet test tests/ApprovalService.Unit.Tests/ApprovalService.Unit.Tests.csproj
  ```
- تشغيل اختبارات التكامل (تتطلب تشغيل الخدمات الداعمة):
  ```bash
  dotnet test tests/ApprovalService.IntegrationTests/ApprovalService.IntegrationTests.csproj
  ```
- تنسيق الشفرة:
  ```bash
  dotnet format src/ApprovalService/ApprovalService.csproj
  ```
