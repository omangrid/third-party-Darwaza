# Getting Started – Profile Service
## Overview
The Profile Service manages employee and requester metadata that other services query when processing approvals. It relies on Azure AD to obtain access tokens for downstream APIs and logs to Seq.

### نظرة عامة
تدير خدمة الملفات الشخصية بيانات الموظفين ومقدمي الطلبات التي تستعلم عنها الخدمات الأخرى أثناء معالجة الموافقات. تعتمد على Azure AD للحصول على رموز الوصول للخدمات الخلفية، وترسل السجلات إلى Seq.

## Prerequisites
- [.NET SDK 9.0](https://dotnet.microsoft.com/en-us/download)
- Azure AD confidential client configuration (client ID, secret, tenant ID).
- Seq endpoint for logging.

### المتطلبات الأساسية
- [حزمة تطوير .NET 9.0](https://dotnet.microsoft.com/en-us/download)
- تهيئة عميل سري في Azure AD (معرّف العميل، السر، معرّف المستأجر).
- نقطة Seq للسجلات.

## Run the service locally with the .NET CLI
1. Export configuration values:
   ```bash
   export ASPNETCORE_ENVIRONMENT=Development
   export ASPNETCORE_URLS=http://localhost:7006
   export AzureAd__TenantId=<your-tenant-id>
   export AzureAd__ClientId=<your-client-id>
   export AzureAd__ClientSecret=<your-client-secret>
   export Serilog__WriteTo__0__Name=Seq
   export Serilog__WriteTo__0__Args__serverUrl=http://localhost:5341
   ```
2. Run the service:
   ```bash
   dotnet run --project src/ProfileService/ProfileService.csproj
   ```
3. Access the API at `http://localhost:7006`.

### تشغيل الخدمة محليًا عبر سطر أوامر .NET
1. صدّر قيم الإعدادات:
   ```bash
   export ASPNETCORE_ENVIRONMENT=Development
   export ASPNETCORE_URLS=http://localhost:7006
   export AzureAd__TenantId=<your-tenant-id>
   export AzureAd__ClientId=<your-client-id>
   export AzureAd__ClientSecret=<your-client-secret>
   export Serilog__WriteTo__0__Name=Seq
   export Serilog__WriteTo__0__Args__serverUrl=http://localhost:5341
   ```
2. شغّل الخدمة:
   ```bash
   dotnet run --project src/ProfileService/ProfileService.csproj
   ```
3. يمكنك الوصول إلى الواجهة عبر `http://localhost:7006`.

## Run the service with Docker
- Start the container:
  ```bash
  docker compose up -d profile-svc
  ```
- Port `7006` is exposed; provide Azure AD secrets through compose overrides.

### تشغيل الخدمة باستخدام Docker
- شغّل الحاوية:
  ```bash
  docker compose up -d profile-svc
  ```
- يتم كشف المنفذ `7006`؛ مرّر أسرار Azure AD عبر ملفات Compose الإضافية.

## Integrating with other services
### Query profiles over HTTP with retries
Install `Polly.Extensions.Http` and configure a typed client to reach the `/users` endpoints:

```csharp
builder.Services
    .AddHttpClient<IProfileClient, ProfileClient>(client =>
    {
        client.BaseAddress = new Uri(configuration["Services:Profiles:BaseUrl"] ?? "http://profile-svc");
        client.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));
    })
    .AddPolicyHandler(HttpPolicyExtensions
        .HandleTransientHttpError()
        .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt))));

public record DirectoryUser(Guid Id, string? DisplayName, string? UserPrincipalName, string? Mail);

public interface IProfileClient
{
    Task<User?> GetUserAsync(Guid id, CancellationToken cancellationToken = default);
    Task<IReadOnlyList<DirectoryUser>> SearchAsync(string term, CancellationToken cancellationToken = default);
}

public sealed class ProfileClient : IProfileClient
{
    private readonly HttpClient _http;

    public ProfileClient(HttpClient http) => _http = http;

    public async Task<User?> GetUserAsync(Guid id, CancellationToken cancellationToken = default)
        => await _http.GetFromJsonAsync<User>($"/users/{id}", cancellationToken);

    public async Task<IReadOnlyList<DirectoryUser>> SearchAsync(string term, CancellationToken cancellationToken = default)
    {
        using var response = await _http.GetAsync($"/users/search?term={Uri.EscapeDataString(term)}", cancellationToken);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<IReadOnlyList<DirectoryUser>>(cancellationToken: cancellationToken)
               ?? Array.Empty<DirectoryUser>();
    }
}

// Microsoft.Graph.Models.User is returned by the service and can be referenced from the Microsoft Graph SDK.
```

### التكامل مع الخدمات الأخرى
#### الاستعلام عن الملفات الشخصية عبر HTTP مع تحمل الأعطال
سجّل عميل `HttpClient` كما في المثال السابق للوصول إلى مسارات `/users` و`/users/search`. تمكّنك سياسة إعادة المحاولة من التعامل مع انقطاعات الشبكة أثناء جلب بيانات المستخدمين أو نتائج البحث.

## Configuration reference
| Setting | Description |
| --- | --- |
| `AzureAd__TenantId` | Tenant that issues tokens for calling downstream APIs. |
| `AzureAd__ClientId` / `AzureAd__ClientSecret` | Confidential client credentials used when acquiring tokens. |
| `Serilog__WriteTo__0__Args__serverUrl` | Seq ingestion endpoint. |

### مرجع الإعدادات
| الإعداد | الوصف |
| --- | --- |
| `AzureAd__TenantId` | المستأجر الذي يصدر الرموز للوصول إلى الخدمات الخلفية. |
| `AzureAd__ClientId` / `AzureAd__ClientSecret` | بيانات اعتماد العميل السري عند طلب الرموز. |
| `Serilog__WriteTo__0__Args__serverUrl` | عنوان إدخال سجلات Seq.

## Useful commands
- Build:
  ```bash
  dotnet build src/ProfileService/ProfileService.csproj
  ```
- Format:
  ```bash
  dotnet format src/ProfileService/ProfileService.csproj
  ```

### أوامر مفيدة
- بناء المشروع:
  ```bash
  dotnet build src/ProfileService/ProfileService.csproj
  ```
- تنسيق الشفرة:
  ```bash
  dotnet format src/ProfileService/ProfileService.csproj
  ```
