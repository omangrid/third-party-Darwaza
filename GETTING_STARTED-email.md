# Getting Started – Email Service
## Overview
The Email Service listens to workflow events and delivers transactional email notifications. It relies on RabbitMQ for inbound messages and forwards structured logs to Seq for observability.

### نظرة عامة
تستقبل خدمة البريد الإلكتروني أحداث سير العمل وتقوم بإرسال رسائل بريدية تنبيهية. تعتمد على RabbitMQ لاستقبال الرسائل، وتوجّه السجلات المهيكلة إلى Seq لتتبع الأداء.

## Prerequisites
- [.NET SDK 9.0](https://dotnet.microsoft.com/en-us/download)
- Access to a RabbitMQ broker and the Seq log collector.
- SMTP credentials configured in `appsettings.json` or user secrets (if required by your deployment).

### المتطلبات الأساسية
- [حزمة تطوير .NET 9.0](https://dotnet.microsoft.com/en-us/download)
- توفر وسيط RabbitMQ ومجمع السجلات Seq.
- بيانات اعتماد SMTP مضبوطة داخل ملف `appsettings.json` أو في أسرار المستخدم (إذا تطلب النشر ذلك).

## Run the service locally with the .NET CLI
1. Start RabbitMQ and Seq locally. With Docker Compose you can reuse the repo configuration:
   ```bash
   docker compose up -d rabbitmq seq
   ```
2. Export environment variables (adjust hostnames as needed):
   ```bash
   export ASPNETCORE_ENVIRONMENT=Development
   export ASPNETCORE_URLS=http://localhost:7003
   export RabbitMq__Host=localhost
   export RabbitMq__Username=mtuser
   export RabbitMq__Password=StrongP@ss123
   export Serilog__WriteTo__0__Name=Seq
   export Serilog__WriteTo__0__Args__serverUrl=http://localhost:5341
   ```
3. Run the worker:
   ```bash
   dotnet run --project src/EmailService/EmailService.csproj
   ```
4. The HTTP health check endpoint is available at `http://localhost:7003/health` (if enabled).

### تشغيل الخدمة محليًا عبر سطر أوامر .NET
1. شغّل RabbitMQ وSeq محليًا. يمكنك استخدام إعدادات المستودع عبر Docker Compose:
   ```bash
   docker compose up -d rabbitmq seq
   ```
2. صدّر متغيرات البيئة (عدّل أسماء المضيفين حسب حاجتك):
   ```bash
   export ASPNETCORE_ENVIRONMENT=Development
   export ASPNETCORE_URLS=http://localhost:7003
   export RabbitMq__Host=localhost
   export RabbitMq__Username=mtuser
   export RabbitMq__Password=StrongP@ss123
   export Serilog__WriteTo__0__Name=Seq
   export Serilog__WriteTo__0__Args__serverUrl=http://localhost:5341
   ```
3. شغّل الخدمة:
   ```bash
   dotnet run --project src/EmailService/EmailService.csproj
   ```
4. يتوفر مسار التحقق الصحي (إن تم تمكينه) عبر `http://localhost:7003/health`.

## Run the service with Docker
- Start the container via compose:
  ```bash
  docker compose up -d email-svc
  ```
- Port `7003` is published to the host, and the container inherits RabbitMQ/Seq configuration from the compose file.

### تشغيل الخدمة باستخدام Docker
- شغّل الحاوية عبر ملف Compose:
  ```bash
  docker compose up -d email-svc
  ```
- يتم نشر المنفذ `7003` على جهازك، كما يتم تمرير إعدادات RabbitMQ وSeq تلقائيًا من ملف Compose.

## Integrating with other services
### Publish email jobs with MassTransit
The Email Service is message-driven. Publish the `SendEmail` record via MassTransit whenever you want to queue an email:

```csharp
public sealed class EmailPublisher
{
    private readonly IPublishEndpoint _publishEndpoint;

    public EmailPublisher(IPublishEndpoint publishEndpoint) => _publishEndpoint = publishEndpoint;

    public Task SendApprovalAsync(string to, string subject, string bodyHtml, string linkUrl, CancellationToken cancellationToken = default)
        => _publishEndpoint.Publish(new SendEmail
        {
            To = to,
            Subject = subject,
            Id = Guid.NewGuid().ToString(),
            BodyHtml = bodyHtml,
            LinkText = "Open",
            LinkUrl = linkUrl,
            BackgroundColor = "#4cd6c0"
        }, cancellationToken);
}
```

The service uses `KebabCaseEndpointNameFormatter("nt", false)`, so the consumer queue is `nt-send-email`. Ensure your publisher resolves the RabbitMQ host, username, and password the same way the service does (`RabbitMq:*`).

### التكامل مع الخدمات الأخرى
#### نشر مهام البريد الإلكتروني عبر MassTransit
هذه الخدمة قائمة على الرسائل فقط، لذا يكفي نشر سجل `SendEmail` باستخدام `IPublishEndpoint` كما في المثال السابق. سيتم التقاط الرسالة من طابور `nt-send-email` وإرسال البريد بالقالب القياسي. تأكد من مطابقة إعدادات RabbitMQ (`RabbitMq:Host` وبيانات الاعتماد) بين خدمتك وخدمة البريد.

## Configuration reference
| Setting | Description |
| --- | --- |
| `RabbitMq__*` | Connection settings used by MassTransit to subscribe to email events. |
| `Serilog__WriteTo__0__Args__serverUrl` | Seq ingestion endpoint for diagnostics. |
| SMTP configuration | Provide SMTP host, port, and credentials through configuration providers. |

### مرجع الإعدادات
| الإعداد | الوصف |
| --- | --- |
| `RabbitMq__*` | إعدادات الاتصال التي يستخدمها MassTransit للاشتراك في أحداث البريد. |
| `Serilog__WriteTo__0__Args__serverUrl` | عنوان إدخال سجلات Seq لمتابعة الأداء. |
| إعدادات SMTP | وفّر عنوان الخادم والمنفذ وبيانات الاعتماد عبر موفري الإعدادات.

## Useful commands
- Run the project with detailed logging:
  ```bash
  dotnet run --project src/EmailService/EmailService.csproj -- --verbosity detailed
  ```
- Format the codebase:
  ```bash
  dotnet format src/EmailService/EmailService.csproj
  ```

### أوامر مفيدة
- تشغيل المشروع مع تمكين السجلات التفصيلية:
  ```bash
  dotnet run --project src/EmailService/EmailService.csproj -- --verbosity detailed
  ```
- تنسيق الشفرة:
  ```bash
  dotnet format src/EmailService/EmailService.csproj
  ```
