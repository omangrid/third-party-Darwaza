# Getting Started – Notification Service
## Overview
The Notification Service centralizes delivery of in-app and push notifications triggered by workflow events. It consumes messages from RabbitMQ, persists notification state to SQL Server, and publishes telemetry to Seq.

### نظرة عامة
تُدير خدمة الإشعارات إرسال التنبيهات داخل التطبيق والتنبيهات الفورية الناتجة عن أحداث سير العمل. تستهلك الرسائل من RabbitMQ، وتخزن حالة الإشعارات في SQL Server، وترسل القياسات إلى Seq.

## Prerequisites
- [.NET SDK 9.0](https://dotnet.microsoft.com/en-us/download)
- Running instances of RabbitMQ, SQL Server, and Seq.
- Azure AD configuration for validating bearer tokens.

### المتطلبات الأساسية
- [حزمة تطوير .NET 9.0](https://dotnet.microsoft.com/en-us/download)
- تشغيل خدمات RabbitMQ وSQL Server وSeq.
- تكوين Azure AD للتحقق من رموز الوصول.

## Run the service locally with the .NET CLI
1. Start the required infrastructure:
   ```bash
   docker compose up -d rabbitmq sqlserver seq
   ```
2. Apply database migrations if needed:
   ```bash
   dotnet ef database update --project src/NotificationService/NotificationService.csproj
   ```
3. Export configuration values:
   ```bash
   export ASPNETCORE_ENVIRONMENT=Development
   export ASPNETCORE_URLS=http://localhost:7005
   export RabbitMq__Host=localhost
   export RabbitMq__Username=mtuser
   export RabbitMq__Password=StrongP@ss123
   export ConnectionStrings__DefaultConnection="Data Source=localhost,1433;Initial Catalog=notify;User ID=sa;Password=Edl@1234;TrustServerCertificate=True;"
   export AzureAd__Instance=https://login.microsoftonline.com
   export AzureAd__TenantId=<your-tenant-id>
   export AzureAd__Authority="https://login.microsoftonline.com/<your-tenant-id>/v2.0"
   export AzureAd__Issuer="https://sts.windows.net/<your-tenant-id>/"
   export AzureAd__RequireHttpsMetadata=true
   export Serilog__WriteTo__0__Name=Seq
   export Serilog__WriteTo__0__Args__serverUrl=http://localhost:5341
   ```
4. Run the service:
   ```bash
   dotnet run --project src/NotificationService/NotificationService.csproj
   ```
5. Health endpoints and Swagger (if enabled) will be served from `http://localhost:7005`.

### تشغيل الخدمة محليًا عبر سطر أوامر .NET
1. شغّل البنية التحتية اللازمة:
   ```bash
   docker compose up -d rabbitmq sqlserver seq
   ```
2. طبّق ترحيلات قاعدة البيانات عند الحاجة:
   ```bash
   dotnet ef database update --project src/NotificationService/NotificationService.csproj
   ```
3. صدّر قيم الإعدادات:
   ```bash
   export ASPNETCORE_ENVIRONMENT=Development
   export ASPNETCORE_URLS=http://localhost:7005
   export RabbitMq__Host=localhost
   export RabbitMq__Username=mtuser
   export RabbitMq__Password=StrongP@ss123
   export ConnectionStrings__DefaultConnection="Data Source=localhost,1433;Initial Catalog=notify;User ID=sa;Password=Edl@1234;TrustServerCertificate=True;"
   export AzureAd__Instance=https://login.microsoftonline.com
   export AzureAd__TenantId=<your-tenant-id>
   export AzureAd__Authority="https://login.microsoftonline.com/<your-tenant-id>/v2.0"
   export AzureAd__Issuer="https://sts.windows.net/<your-tenant-id>/"
   export AzureAd__RequireHttpsMetadata=true
   export Serilog__WriteTo__0__Name=Seq
   export Serilog__WriteTo__0__Args__serverUrl=http://localhost:5341
   ```
4. شغّل الخدمة:
   ```bash
   dotnet run --project src/NotificationService/NotificationService.csproj
   ```
5. ستُخدم نقاط الصحة وSwagger (إن وُجدت) من `http://localhost:7005`.

## Run the service with Docker
- Start the container:
  ```bash
  docker compose up -d notify-svc
  ```
- Dependencies are injected automatically and port `7005` is exposed to the host.

### تشغيل الخدمة باستخدام Docker
- شغّل الحاوية:
  ```bash
  docker compose up -d notify-svc
  ```
- يتم تمرير الاعتمادات تلقائيًا ويتم كشف المنفذ `7005` على الجهاز المضيف.

## Integrating with other services
### Query notifications over HTTP with retries
Install `Polly.Extensions.Http` and configure a typed client to retrieve the authenticated user's notifications:

```csharp
builder.Services
    .AddHttpClient<INotificationClient, NotificationClient>(client =>
    {
        client.BaseAddress = new Uri(configuration["Services:Notifications:BaseUrl"] ?? "http://notification-svc");
        client.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));
    })
    .AddPolicyHandler(HttpPolicyExtensions
        .HandleTransientHttpError()
        .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt))));

public interface INotificationClient
{
    Task<IReadOnlyList<NotificationDto>> GetUnreadAsync(CancellationToken cancellationToken = default);
}

public sealed class NotificationClient : INotificationClient
{
    private readonly HttpClient _http;

    public NotificationClient(HttpClient http) => _http = http;

    public async Task<IReadOnlyList<NotificationDto>> GetUnreadAsync(CancellationToken cancellationToken = default)
    {
        using var response = await _http.GetAsync("/api/notifications", cancellationToken);
        response.EnsureSuccessStatusCode();
        var items = await response.Content.ReadFromJsonAsync<List<NotificationDto>>(cancellationToken: cancellationToken);
        return items ?? new List<NotificationDto>();
    }
}
```

Define `NotificationDto` with the fields exposed by the controller (`Id`, `Title`, `Content`, `Link`, etc.) and inject `INotificationClient` wherever you need to render unread alerts.

### التكامل مع الخدمات الأخرى
#### استرجاع الإشعارات عبر HTTP مع تحمل الأعطال
سجّل عميل `HttpClient` كما في المثال السابق ثم استدعِ `GetUnreadAsync` للحصول على الإشعارات غير المقروءة مع الاستفادة من محاولات إعادة المحاولة التلقائية ضد الأعطال المؤقتة.

### Publish notification messages with MassTransit
Send a `NotificationMessage` over RabbitMQ to push alerts to the service. The endpoint formatter prefixes queues with `nt-`, so the consumer queue is `nt-notification`.

```csharp
public sealed class NotificationPublisher
{
    private readonly IPublishEndpoint _publishEndpoint;

    public NotificationPublisher(IPublishEndpoint publishEndpoint) => _publishEndpoint = publishEndpoint;

    public Task SendAsync(string userId, string title, string content, CancellationToken cancellationToken = default) =>
        _publishEndpoint.Publish(new NotificationMessage
        {
            Id = Guid.NewGuid(),
            UserId = userId,
            Title = title,
            Content = content,
            Link = "https://app.approval.local/notifications"
        }, cancellationToken);
}
```

### نشر رسائل الإشعارات عبر MassTransit
لنشر إشعار جديد استخدم `IPublishEndpoint` كما في المثال أعلاه لإرسال كائن `NotificationMessage`. ستتم معالجته من قِبل `NotificationConsumer` الذي يخزن الرسالة ويقوم ببثها عبر SignalR للمستخدم المستهدف.

## Configuration reference
| Setting | Description |
| --- | --- |
| `ConnectionStrings__DefaultConnection` | SQL Server connection string for notification persistence. |
| `RabbitMq__*` | MassTransit bus configuration. |
| `AzureAd__*` | Azure AD authority/tenant settings for bearer authentication. |
| `Serilog__WriteTo__0__Args__serverUrl` | Seq ingestion endpoint. |

### مرجع الإعدادات
| الإعداد | الوصف |
| --- | --- |
| `ConnectionStrings__DefaultConnection` | سلسلة اتصال SQL Server لتخزين بيانات الإشعارات. |
| `RabbitMq__*` | إعدادات حافلة MassTransit. |
| `AzureAd__*` | إعدادات Azure AD الخاصة بالسلطة ومعرّف المستأجر للمصادقة بالرموز. |
| `Serilog__WriteTo__0__Args__serverUrl` | عنوان إدخال سجلات Seq.

## Useful commands
- Restore and build:
  ```bash
  dotnet build src/NotificationService/NotificationService.csproj
  ```
- Format code:
  ```bash
  dotnet format src/NotificationService/NotificationService.csproj
  ```

### أوامر مفيدة
- استعادة الحزم وبناء المشروع:
  ```bash
  dotnet build src/NotificationService/NotificationService.csproj
  ```
- تنسيق الشفرة:
  ```bash
  dotnet format src/NotificationService/NotificationService.csproj
  ```
