# Getting Started – File Service
## Overview
The File Service handles document uploads and storage orchestration for the platform. It listens to RabbitMQ events to process files asynchronously and can expose HTTP endpoints for direct uploads if enabled.

### نظرة عامة
تتولى خدمة الملفات تحميل المستندات وإدارة التخزين لمنصة النظام. تستمع لأحداث RabbitMQ لمعالجة الملفات بشكل غير متزامن، ويمكنها توفير واجهات HTTP للتحميل المباشر إذا تم تمكينها.

## Prerequisites
- [.NET SDK 9.0](https://dotnet.microsoft.com/en-us/download)
- Running instances of RabbitMQ and Seq.
- Access to the backing storage account or shared file system configured in `appsettings.json`.

### المتطلبات الأساسية
- [حزمة تطوير .NET 9.0](https://dotnet.microsoft.com/en-us/download)
- تشغيل RabbitMQ وSeq.
- الوصول إلى حساب التخزين أو نظام الملفات المشترك المحدد في ملف `appsettings.json`.

## Run the service locally with the .NET CLI
1. Start RabbitMQ and Seq:
   ```bash
   docker compose up -d rabbitmq seq
   ```
2. Export environment variables:
   ```bash
   export ASPNETCORE_ENVIRONMENT=Development
   export ASPNETCORE_URLS=http://localhost:7004
   export RabbitMq__Host=localhost
   export RabbitMq__Username=mtuser
   export RabbitMq__Password=StrongP@ss123
   export Serilog__WriteTo__0__Name=Seq
   export Serilog__WriteTo__0__Args__serverUrl=http://localhost:5341
   ```
3. Provide storage configuration through user secrets or environment variables (e.g., blob connection string).
4. Run the service:
   ```bash
   dotnet run --project src/FileService/FileService.csproj
   ```

### تشغيل الخدمة محليًا عبر سطر أوامر .NET
1. شغّل RabbitMQ وSeq:
   ```bash
   docker compose up -d rabbitmq seq
   ```
2. صدّر متغيرات البيئة:
   ```bash
   export ASPNETCORE_ENVIRONMENT=Development
   export ASPNETCORE_URLS=http://localhost:7004
   export RabbitMq__Host=localhost
   export RabbitMq__Username=mtuser
   export RabbitMq__Password=StrongP@ss123
   export Serilog__WriteTo__0__Name=Seq
   export Serilog__WriteTo__0__Args__serverUrl=http://localhost:5341
   ```
3. وفّر إعدادات التخزين عبر أسرار المستخدم أو متغيرات البيئة (مثل سلسلة اتصال Blob).
4. شغّل الخدمة:
   ```bash
   dotnet run --project src/FileService/FileService.csproj
   ```

## Run the service with Docker
- Start the container:
  ```bash
  docker compose up -d file-svc
  ```
- Port `7004` is exposed for HTTP access if configured.

### تشغيل الخدمة باستخدام Docker
- شغّل الحاوية:
  ```bash
  docker compose up -d file-svc
  ```
- يتم كشف المنفذ `7004` للوصول عبر HTTP إذا تم تفعيل ذلك.

## Integrating with other services
### Upload and download files over HTTP with retries
Install `Polly.Extensions.Http` and configure a typed client to interact with the minimal API endpoints:

```csharp
builder.Services
    .AddHttpClient<IFileClient, FileClient>(client =>
    {
        client.BaseAddress = new Uri(configuration["Services:Files:BaseUrl"] ?? "http://file-svc");
    })
    .AddPolicyHandler(HttpPolicyExtensions
        .HandleTransientHttpError()
        .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt))));

public interface IFileClient
{
    Task<SignedUploadResponse> CreateSignedUploadAsync(string fileName, string contentType, CancellationToken cancellationToken = default);
    Task UploadAsync(string fileId, Stream content, CancellationToken cancellationToken = default);
    Task<Stream> DownloadAsync(string fileId, CancellationToken cancellationToken = default);
}

public sealed class FileClient : IFileClient
{
    private readonly HttpClient _http;

    public FileClient(HttpClient http) => _http = http;

    public async Task<SignedUploadResponse> CreateSignedUploadAsync(string fileName, string contentType, CancellationToken cancellationToken = default)
    {
        var request = new SignedUploadRequest(fileName, contentType);
        using var response = await _http.PostAsJsonAsync("/uploads/signed", request, cancellationToken);
        response.EnsureSuccessStatusCode();
        return (await response.Content.ReadFromJsonAsync<SignedUploadResponse>(cancellationToken: cancellationToken))!;
    }

    public async Task UploadAsync(string fileId, Stream content, CancellationToken cancellationToken = default)
    {
        using var streamContent = new StreamContent(content);
        using var response = await _http.PutAsync($"/uploads/{fileId}", streamContent, cancellationToken);
        response.EnsureSuccessStatusCode();
    }

    public Task<Stream> DownloadAsync(string fileId, CancellationToken cancellationToken = default)
        => _http.GetStreamAsync($"/uploads/{fileId}", cancellationToken);
}
```

Model the `SignedUploadRequest`/`SignedUploadResponse` types according to the service contract (name, content type, upload URL, generated identifier).

### التكامل مع الخدمات الأخرى
#### تحميل الملفات وتنزيلها عبر HTTP مع تحمل الأعطال
سجّل عميل `HttpClient` كما في المثال السابق. استدعِ `CreateSignedUploadAsync` لإنشاء معرف ملف ومسار التحميل، ثم `UploadAsync` لدفع البايتات و`DownloadAsync` لاسترجاع الملف عند الحاجة. تضمن سياسة إعادة المحاولة التعامل مع أخطاء الشبكة المؤقتة أثناء عمليات التحميل أو التنزيل.

## Configuration reference
| Setting | Description |
| --- | --- |
| `RabbitMq__*` | MassTransit configuration for file processing events. |
| Storage configuration | Configure blob or filesystem storage provider details. |
| `Serilog__WriteTo__0__Args__serverUrl` | Seq ingestion endpoint. |

### مرجع الإعدادات
| الإعداد | الوصف |
| --- | --- |
| `RabbitMq__*` | إعدادات MassTransit الخاصة بأحداث معالجة الملفات. |
| إعدادات التخزين | بيانات موفر التخزين (Blob أو نظام ملفات). |
| `Serilog__WriteTo__0__Args__serverUrl` | عنوان إدخال سجلات Seq.

## Useful commands
- Build:
  ```bash
  dotnet build src/FileService/FileService.csproj
  ```
- Format:
  ```bash
  dotnet format src/FileService/FileService.csproj
  ```

### أوامر مفيدة
- بناء المشروع:
  ```bash
  dotnet build src/FileService/FileService.csproj
  ```
- تنسيق الشفرة:
  ```bash
  dotnet format src/FileService/FileService.csproj
  ```
