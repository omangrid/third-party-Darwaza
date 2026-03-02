# Getting Started – Request Service
## Overview
The Request Service orchestrates incoming approval requests, coordinating with the USB and Tar services to assemble complete payloads. It persists request data to SQL Server, emits events through RabbitMQ, and logs to Seq.

### نظرة عامة
تتولى خدمة الطلبات تنظيم طلبات الموافقات الواردة، وتنسق مع خدمتي USB وTar لتجميع البيانات الكاملة. تخزّن بيانات الطلب في SQL Server، وترسل الأحداث عبر RabbitMQ، وتدوّن السجلات في Seq.

## Prerequisites
- [.NET SDK 9.0](https://dotnet.microsoft.com/en-us/download)
- Running instances of SQL Server, RabbitMQ, Seq, and the dependent USB/Tar services.

### المتطلبات الأساسية
- [حزمة تطوير .NET 9.0](https://dotnet.microsoft.com/en-us/download)
- تشغيل SQL Server وRabbitMQ وSeq وخدمات USB/Tar التابعة.

## Run the service locally with the .NET CLI
1. Start dependencies:
   ```bash
   docker compose up -d sqlserver rabbitmq seq
   ```
   Ensure the USB and Tar services are also running if you need live integrations.
2. Apply migrations:
   ```bash
   dotnet ef database update --project src/RequestService/RequestService.csproj
   ```
3. Export environment variables:
   ```bash
   export ASPNETCORE_ENVIRONMENT=Development
   export ASPNETCORE_URLS=http://localhost:7010
   export RabbitMq__Host=localhost
   export RabbitMq__Username=mtuser
   export RabbitMq__Password=StrongP@ss123
   export ConnectionStrings__DefaultConnection="Data Source=localhost,1433;Initial Catalog=requests;User ID=sa;Password=Edl@1234;TrustServerCertificate=True;"
   export Services__Usb__BaseAddress=http://localhost:7002
   export Services__Tar__BaseAddress=http://localhost:7007
   export Serilog__WriteTo__0__Name=Seq
   export Serilog__WriteTo__0__Args__serverUrl=http://localhost:5341
   ```
4. Run the service:
   ```bash
   dotnet run --project src/RequestService/RequestService.csproj
   ```
5. The API is available at `http://localhost:7010`.

### تشغيل الخدمة محليًا عبر سطر أوامر .NET
1. شغّل الخدمات التابعة:
   ```bash
   docker compose up -d sqlserver rabbitmq seq
   ```
   تأكد من تشغيل خدمتي USB وTar إذا كنت بحاجة للتكامل المباشر.
2. طبّق الترحيلات:
   ```bash
   dotnet ef database update --project src/RequestService/RequestService.csproj
   ```
3. صدّر متغيرات البيئة:
   ```bash
   export ASPNETCORE_ENVIRONMENT=Development
   export ASPNETCORE_URLS=http://localhost:7010
   export RabbitMq__Host=localhost
   export RabbitMq__Username=mtuser
   export RabbitMq__Password=StrongP@ss123
   export ConnectionStrings__DefaultConnection="Data Source=localhost,1433;Initial Catalog=requests;User ID=sa;Password=Edl@1234;TrustServerCertificate=True;"
   export Services__Usb__BaseAddress=http://localhost:7002
   export Services__Tar__BaseAddress=http://localhost:7007
   export Serilog__WriteTo__0__Name=Seq
   export Serilog__WriteTo__0__Args__serverUrl=http://localhost:5341
   ```
4. شغّل الخدمة:
   ```bash
   dotnet run --project src/RequestService/RequestService.csproj
   ```
5. ستكون الواجهة متاحة على `http://localhost:7010`.

## Run the service with Docker
- Start the container:
  ```bash
  docker compose up -d request-svc
  ```
- Compose injects the correct base URLs for USB and Tar services and exposes port `7010`.

### تشغيل الخدمة باستخدام Docker
- شغّل الحاوية:
  ```bash
  docker compose up -d request-svc
  ```
- يمرّر ملف Compose عناوين خدمات USB وTar ويكشف المنفذ `7010`.

## Integrating with other services
### Submit requests over HTTP with resilience
Expose the Request Service to other ASP.NET Core components by registering a typed client with Polly retries. Install the `Polly.Extensions.Http` package if it is not already referenced:

```csharp
builder.Services
    .AddHttpClient<IRequestGateway, RequestGateway>(client =>
    {
        client.BaseAddress = new Uri(configuration["Services:Requests:BaseUrl"] ?? "http://request-svc");
        client.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));
    })
    .AddPolicyHandler(HttpPolicyExtensions
        .HandleTransientHttpError()
        .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt))));

public interface IRequestGateway
{
    Task SubmitUsbAsync(RequestSubmissionDto payload, CancellationToken cancellationToken = default);
}

public sealed class RequestGateway : IRequestGateway
{
    private readonly HttpClient _http;

    public RequestGateway(HttpClient http) => _http = http;

    public async Task SubmitUsbAsync(RequestSubmissionDto payload, CancellationToken cancellationToken = default)
    {
        using var response = await _http.PostAsJsonAsync("/api/requests/usb", payload, cancellationToken);
        response.EnsureSuccessStatusCode();
    }
}
```

Inject `IRequestGateway` and call `SubmitUsbAsync` (or use another route such as `tar`) to create approval requests with automatic retries for transient errors.

### التكامل مع الخدمات الأخرى
#### إرسال الطلبات عبر HTTP مع تحمل الأعطال
سجّل عميلًا مخصصًا يوجّه الطلبات إلى واجهة `/api/requests/{app}` مع سياسة إعادة المحاولة من Polly للتعامل مع الأعطال المؤقتة:

```csharp
builder.Services
    .AddHttpClient<IRequestGateway, RequestGateway>(client =>
    {
        client.BaseAddress = new Uri(configuration["Services:Requests:BaseUrl"] ?? "http://request-svc");
        client.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));
    })
    .AddPolicyHandler(HttpPolicyExtensions
        .HandleTransientHttpError()
        .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt))));
```

بعد تسجيل العميل يمكنك استدعاء `SubmitUsbAsync` أو أي مسار آخر (`tar` مثلًا) لإنشاء طلبات جديدة بأمان.

### Forward workflow messages with MassTransit
When you already have a complete `WorkflowBuilding` payload you can skip the HTTP hop and send it straight to the Approval Service queue:

```csharp
public sealed class WorkflowForwarder
{
    private readonly ISendEndpointProvider _sendEndpointProvider;

    public WorkflowForwarder(ISendEndpointProvider sendEndpointProvider) => _sendEndpointProvider = sendEndpointProvider;

    public async Task SendAsync(WorkflowBuilding workflow, CancellationToken cancellationToken = default)
    {
        var endpoint = await _sendEndpointProvider.GetSendEndpoint(new Uri("queue:approval-workflow-building"));
        await endpoint.Send(workflow, cancellationToken);
    }
}
```

The Approval Service expects the `WorkflowBuilding` contract defined in `src/Contracts`. Populate fields such as `App`, `RelatedId`, `UserId`, `Request`, and optional `Attachments` before sending.

### تمرير رسائل سير العمل عبر MassTransit
إذا كان لديك حمولة جاهزة من نوع `WorkflowBuilding` فيمكنك إرسالها مباشرة إلى طابور `approval-workflow-building` باستخدام `ISendEndpointProvider` كما في المثال أعلاه. تأكد من تعبئة الحقول (`App`، `RelatedId`، `UserId`، `Request`، والمرفقات عند الحاجة) قبل الإرسال.

## Configuration reference
| Setting | Description |
| --- | --- |
| `ConnectionStrings__DefaultConnection` | SQL Server connection used to persist requests. |
| `RabbitMq__*` | MassTransit configuration for publishing domain events. |
| `Services__Usb__BaseAddress` | Base URL for the USB service when enriching requests. |
| `Services__Tar__BaseAddress` | Base URL for the Tar service. |
| `Serilog__WriteTo__0__Args__serverUrl` | Seq ingestion endpoint. |

### مرجع الإعدادات
| الإعداد | الوصف |
| --- | --- |
| `ConnectionStrings__DefaultConnection` | اتصال SQL Server المستخدم لتخزين الطلبات. |
| `RabbitMq__*` | إعدادات MassTransit لنشر أحداث المجال. |
| `Services__Usb__BaseAddress` | العنوان الأساسي لخدمة USB عند استكمال البيانات. |
| `Services__Tar__BaseAddress` | العنوان الأساسي لخدمة Tar. |
| `Serilog__WriteTo__0__Args__serverUrl` | عنوان إدخال سجلات Seq.

## Useful commands
- Build:
  ```bash
  dotnet build src/RequestService/RequestService.csproj
  ```
- Format:
  ```bash
  dotnet format src/RequestService/RequestService.csproj
  ```

### أوامر مفيدة
- بناء المشروع:
  ```bash
  dotnet build src/RequestService/RequestService.csproj
  ```
- تنسيق الشفرة:
  ```bash
  dotnet format src/RequestService/RequestService.csproj
  ```
