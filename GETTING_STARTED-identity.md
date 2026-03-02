# Getting Started – Identity Service
## Overview
The Identity Service implements authentication flows and user management, integrating with Azure AD for external identity and providing local pages for sign-in and consent.

### نظرة عامة
توفّر خدمة الهوية مسارات المصادقة وإدارة المستخدمين، وتتكامل مع Azure AD للهوية الخارجية مع توفير صفحات محلية لتسجيل الدخول والموافقة.

## Prerequisites
- [.NET SDK 9.0](https://dotnet.microsoft.com/en-us/download)
- Local SQL Server instance (default connection targets LocalDB; adjust for Docker or Azure SQL).
- Azure AD application registration for federated authentication.
- Seq endpoint for structured logging.

### المتطلبات الأساسية
- [حزمة تطوير .NET 9.0](https://dotnet.microsoft.com/en-us/download)
- مثيل SQL Server محلي (الإعداد الافتراضي يستخدم LocalDB؛ عدّل الإعداد لـ Docker أو Azure SQL عند الحاجة).
- تسجيل تطبيق في Azure AD للمصادقة الاتحادية.
- نقطة Seq للسجلات المهيكلة.

## Run the service locally with the .NET CLI
1. Ensure SQL Server is available and update `ConnectionStrings:IdentityServiceContextConnection` if you are not using LocalDB.
2. Export environment variables (add your secrets as needed):
   ```bash
   export ASPNETCORE_ENVIRONMENT=Development
   export ASPNETCORE_URLS=https://localhost:5001
   export Serilog__WriteTo__0__Name=Seq
   export Serilog__WriteTo__0__Args__serverUrl=http://localhost:5341
   export IdentityServiceUrl=https://login.microsoftonline.com/<your-tenant-id>/v2.0
   export ApiIdentifier="api://<your-api-identifier>"
   ```
3. Trust the developer certificate if you have not already:
   ```bash
   dotnet dev-certs https --trust
   ```
4. Run the service:
   ```bash
   dotnet run --project src/IdentityService/IdentityService.csproj
   ```
5. Open the UI at `https://localhost:5001`.

### تشغيل الخدمة محليًا عبر سطر أوامر .NET
1. تأكد من توفر SQL Server وقم بتحديث قيمة `ConnectionStrings:IdentityServiceContextConnection` إذا لم تستخدم LocalDB.
2. صدّر متغيرات البيئة (أضف القيم السرية حسب حاجتك):
   ```bash
   export ASPNETCORE_ENVIRONMENT=Development
   export ASPNETCORE_URLS=https://localhost:5001
   export Serilog__WriteTo__0__Name=Seq
   export Serilog__WriteTo__0__Args__serverUrl=http://localhost:5341
   export IdentityServiceUrl=https://login.microsoftonline.com/<your-tenant-id>/v2.0
   export ApiIdentifier="api://<your-api-identifier>"
   ```
3. ثق بشهادة المطور إذا لم تفعل ذلك مسبقًا:
   ```bash
   dotnet dev-certs https --trust
   ```
4. شغّل الخدمة:
   ```bash
   dotnet run --project src/IdentityService/IdentityService.csproj
   ```
5. افتح الواجهة على `https://localhost:5001`.

## Run the service with Docker
1. Build and run the container (create a compose service or use `docker build` + `docker run`). Example with Docker Compose override:
   ```bash
   docker compose up -d identity-svc
   ```
2. Provide the same environment variables through compose or a `.env` file, and mount certificates if HTTPS termination is required.

### تشغيل الخدمة باستخدام Docker
1. قم ببناء الحاوية وتشغيلها (أنشئ خدمة Compose أو استخدم أوامر `docker build` و`docker run`). مثال باستخدام Docker Compose:
   ```bash
   docker compose up -d identity-svc
   ```
2. مرّر نفس متغيرات البيئة عبر ملف Compose أو ملف `.env`، وقم بربط الشهادات إذا كان إنهاء HTTPS مطلوبًا.

## Integrating with other services
### Configure ASP.NET Core authentication against the Identity Service
Point your web or API application at the Identity Service authority using `AddAuthentication`/`AddOpenIdConnect`:

```csharp
builder.Services.AddAuthentication(options =>
    {
        options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
        options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
    })
    .AddCookie()
    .AddOpenIdConnect(options =>
    {
        options.Authority = configuration["IdentityService:Authority"] ?? "https://localhost:5001";
        options.ClientId = configuration["IdentityService:ClientId"]!;
        options.ClientSecret = configuration["IdentityService:ClientSecret"];
        options.ResponseType = "code";
        options.SaveTokens = true;
        options.Scope.Add("offline_access");
        options.Scope.Add(configuration["IdentityService:ApiScope"] ?? "api://approval");
    });
```

For API-to-API calls, request tokens via MSAL or `TokenClient` and attach them as bearer tokens to HTTP requests forwarded to other services.

### التكامل مع الخدمات الأخرى
#### إعداد مصادقة ASP.NET Core بالاعتماد على خدمة الهوية
استخدم الكود السابق لضبط `AddOpenIdConnect` بحيث يشير إلى عنوان خدمة الهوية (`IdentityService:Authority`) مع معرف العميل والسر. يمكن للخدمات الخلفية طلب رموز الوصول بالطريقة نفسها وإرسالها في ترويسة `Authorization` عند استدعاء الخدمات الأخرى.

## Configuration reference
| Setting | Description |
| --- | --- |
| `IdentityServiceUrl` | Upstream authority used for external authentication. |
| `ApiIdentifier` | API audience for issued tokens. |
| `ConnectionStrings__IdentityServiceContextConnection` | Database used for ASP.NET Identity storage. |
| `Serilog__WriteTo__0__Args__serverUrl` | Seq ingestion endpoint. |

### مرجع الإعدادات
| الإعداد | الوصف |
| --- | --- |
| `IdentityServiceUrl` | عنوان السلطة المستخدم للمصادقة الخارجية. |
| `ApiIdentifier` | قيمة الجمهور للرموز الصادرة. |
| `ConnectionStrings__IdentityServiceContextConnection` | قاعدة البيانات المستخدمة لتخزين ASP.NET Identity. |
| `Serilog__WriteTo__0__Args__serverUrl` | عنوان إدخال سجلات Seq.

## Useful commands
- Build:
  ```bash
  dotnet build src/IdentityService/IdentityService.csproj
  ```
- Update the database schema:
  ```bash
  dotnet ef database update --project src/IdentityService/IdentityService.csproj
  ```
- Format code:
  ```bash
  dotnet format src/IdentityService/IdentityService.csproj
  ```

### أوامر مفيدة
- بناء المشروع:
  ```bash
  dotnet build src/IdentityService/IdentityService.csproj
  ```
- تحديث مخطط قاعدة البيانات:
  ```bash
  dotnet ef database update --project src/IdentityService/IdentityService.csproj
  ```
- تنسيق الشفرة:
  ```bash
  dotnet format src/IdentityService/IdentityService.csproj
  ```
