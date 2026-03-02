# Getting Started – Gateway Service
## Overview
The Gateway Service fronts the internal APIs, handling authentication via Azure AD and proxying requests to downstream microservices. It also enriches HTTP logging with Serilog and forwards events to Seq.

### نظرة عامة
تعمل خدمة البوابة كواجهة أمامية للواجهات البرمجية الداخلية، حيث تتولى المصادقة عبر Azure AD وتمرير الطلبات إلى الخدمات المصغرة الخلفية. كما تدعم سجلات HTTP باستخدام Serilog وترسل الأحداث إلى Seq.

## Prerequisites
- [.NET SDK 9.0](https://dotnet.microsoft.com/en-us/download)
- Azure AD application (client ID, tenant ID, issuer) configured for API access.
- Access to the Seq log collector.

### المتطلبات الأساسية
- [حزمة تطوير .NET 9.0](https://dotnet.microsoft.com/en-us/download)
- تكوين تطبيق Azure AD (معرّف العميل، معرّف المستأجر، المُصدِر) للوصول إلى الواجهات.
- إمكانية الوصول إلى نظام سجلات Seq.

## Run the service locally with the .NET CLI
1. Export required environment variables (replace placeholders with your Azure AD values):
   ```bash
   export ASPNETCORE_ENVIRONMENT=Docker
   export ASPNETCORE_URLS=http://localhost:6001
   export Serilog__WriteTo__0__Name=Seq
   export Serilog__WriteTo__0__Args__serverUrl=http://localhost:5341
   export ApiIdentifier="api://<your-api-identifier>"
   export AzureAd__Instance=https://login.microsoftonline.com
   export AzureAd__TenantId=<your-tenant-id>
   export AzureAd__Authority="https://login.microsoftonline.com/<your-tenant-id>/v2.0"
   export AzureAd__Issuer="https://sts.windows.net/<your-tenant-id>/"
   export AzureAd__RequireHttpsMetadata=false
   export ClientApp=http://localhost:3000
   ```
2. Run the project:
   ```bash
   dotnet run --project src/GateWayService/GateWayService.csproj
   ```
3. Requests routed through the gateway will be available at `http://localhost:6001` by default.

### تشغيل الخدمة محليًا عبر سطر أوامر .NET
1. صدّر متغيرات البيئة المطلوبة (استبدل القيم بقيم Azure AD لديك):
   ```bash
   export ASPNETCORE_ENVIRONMENT=Docker
   export ASPNETCORE_URLS=http://localhost:6001
   export Serilog__WriteTo__0__Name=Seq
   export Serilog__WriteTo__0__Args__serverUrl=http://localhost:5341
   export ApiIdentifier="api://<your-api-identifier>"
   export AzureAd__Instance=https://login.microsoftonline.com
   export AzureAd__TenantId=<your-tenant-id>
   export AzureAd__Authority="https://login.microsoftonline.com/<your-tenant-id>/v2.0"
   export AzureAd__Issuer="https://sts.windows.net/<your-tenant-id>/"
   export AzureAd__RequireHttpsMetadata=false
   export ClientApp=http://localhost:3000
   ```
2. شغّل المشروع:
   ```bash
   dotnet run --project src/GateWayService/GateWayService.csproj
   ```
3. ستكون الطلبات المتجهة عبر البوابة متاحة على `http://localhost:6001` بشكل افتراضي.

## Run the service with Docker
- Launch the container:
  ```bash
  docker compose up -d gateway-svc
  ```
- Port `6001` is exposed, and the compose file injects Azure AD settings along with Seq integration.

### تشغيل الخدمة باستخدام Docker
- شغّل الحاوية:
  ```bash
  docker compose up -d gateway-svc
  ```
- يتم كشف المنفذ `6001`، ويقوم ملف Compose بتمرير إعدادات Azure AD والتكامل مع Seq تلقائيًا.

## Integrating with other services
### Call downstream APIs through the gateway
Register a typed client that targets the gateway and injects Azure AD tokens retrieved by `ITokenAcquisition` or MSAL. The example below uses an `IAccessTokenProvider` abstraction:

```csharp
builder.Services
    .AddHttpClient<IGatewayClient, GatewayClient>(client =>
    {
        client.BaseAddress = new Uri(configuration["Services:Gateway:BaseUrl"] ?? "http://gateway-svc");
    })
    .AddPolicyHandler(HttpPolicyExtensions
        .HandleTransientHttpError()
        .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt))));

public interface IGatewayClient
{
    Task<HttpResponseMessage> ForwardAsync(HttpRequestMessage request, CancellationToken cancellationToken = default);
}

public sealed class GatewayClient : IGatewayClient
{
    private readonly HttpClient _httpClient;
    private readonly IAccessTokenProvider _tokenProvider;

    public GatewayClient(HttpClient httpClient, IAccessTokenProvider tokenProvider)
    {
        _httpClient = httpClient;
        _tokenProvider = tokenProvider;
    }

    public async Task<HttpResponseMessage> ForwardAsync(HttpRequestMessage request, CancellationToken cancellationToken = default)
    {
        var token = await _tokenProvider.GetAccessTokenAsync(cancellationToken);
        request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token);
        return await _httpClient.SendAsync(request, cancellationToken);
    }
}
```

This ensures every proxied call carries the correct bearer token and benefits from the gateway's resilience policies.

### التكامل مع الخدمات الأخرى
#### استدعاء الواجهات عبر خدمة البوابة
سجّل عميل `HttpClient` كما في المثال السابق وحدّد العنوان الأساسي للبوابة (`Services:Gateway:BaseUrl`). استخدم موفّر رموز للوصول (`IAccessTokenProvider`) لإرفاق رمز الوصول في ترويسة `Authorization` قبل إرسال الطلب. تضمن سياسة إعادة المحاولة التعامل مع انقطاعات الشبكة المؤقتة أثناء تمرير الطلبات إلى الخدمات الخلفية.

## Configuration reference
| Setting | Description |
| --- | --- |
| `ApiIdentifier` | Audience that the gateway expects on incoming JWT tokens. |
| `AzureAd__*` | Azure AD authority, tenant, and issuer details for validating tokens. |
| `ClientApp` | Origin of the frontend application for CORS policies. |
| `Serilog__WriteTo__0__Args__serverUrl` | Seq ingestion endpoint. |

### مرجع الإعدادات
| الإعداد | الوصف |
| --- | --- |
| `ApiIdentifier` | قيمة الجمهور المتوقعة في رموز JWT الواردة. |
| `AzureAd__*` | تفاصيل Azure AD الخاصة بالسلطة ومعرّف المستأجر والمُصدِر للتحقق من الرموز. |
| `ClientApp` | أصل تطبيق الواجهة الأمامية المستخدم في سياسات CORS. |
| `Serilog__WriteTo__0__Args__serverUrl` | عنوان إدخال سجلات Seq.

## Useful commands
- Restore and build:
  ```bash
  dotnet build src/GateWayService/GateWayService.csproj
  ```
- Format code:
  ```bash
  dotnet format src/GateWayService/GateWayService.csproj
  ```

### أوامر مفيدة
- استعادة الحزم وبناء المشروع:
  ```bash
  dotnet build src/GateWayService/GateWayService.csproj
  ```
- تنسيق الشفرة:
  ```bash
  dotnet format src/GateWayService/GateWayService.csproj
  ```
