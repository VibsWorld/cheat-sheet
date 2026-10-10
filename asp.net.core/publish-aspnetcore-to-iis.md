# Publishing an ASP.NET Core (8+) Application to Windows IIS

A generic, step-by-step guide for deploying any ASP.NET Core 8 or later application (MVC, Razor Pages, Web API, Blazor Server) to a Windows IIS server, including static assets under `wwwroot`.

> **Placeholders used in this guide** – replace them with your own values:
>
> | Placeholder | Example |
> |---|---|
> | `<AppName>` | `MyWebApp` |
> | `<ProjectFile>` | `MyWebApp.csproj` |
> | `<PublishPath>` | `C:\inetpub\mywebapp` |
> | `<AppPoolName>` | `MyWebApp` |
> | `<TargetFramework>` | `net8.0`, `net9.0`, `net10.0` |

---

## 1. Prerequisites

### 1.1 On the IIS server

| Component | Why it is needed |
|---|---|
| **.NET Hosting Bundle** (same major version as your app, or newer) | Installs the .NET runtime, ASP.NET Core runtime, and the **ASP.NET Core Module (ANCM)** that lets IIS host and proxy to the app. Download from [dotnet.microsoft.com/download/dotnet](https://dotnet.microsoft.com/download/dotnet). |
| **IIS role** | Enable via *Server Manager → Add Roles and Features → Web Server (IIS)* or PowerShell (below). |
| **Database / external services** | Any database or API your app depends on must be reachable from the server (firewall, DNS, credentials). |

Enable IIS and the commonly required features with PowerShell (run as Administrator):

```powershell
Install-WindowsFeature -Name Web-Server, Web-Common-Http, Web-Static-Content, `
  Web-Default-Doc, Web-Http-Errors, Web-Http-Logging, Web-Request-Monitor, `
  Web-Filtering, Web-Stat-Compression, Web-Mgmt-Console, Web-WebSockets `
  -IncludeManagementTools
```

> **WebSocket Protocol** is required for SignalR and Blazor Server. It is harmless to enable for other apps.

After installing the Hosting Bundle, restart IIS so ANCM is loaded:

```powershell
net stop was /y
net start w3svc
# or simply:
iisreset /restart
```

> **Order matters.** Install IIS *first*, then the Hosting Bundle. If you installed the bundle before IIS, repair/re-run the bundle installer.

Verify the install:

```powershell
dotnet --list-runtimes
# Expect Microsoft.AspNetCore.App 8.x (or higher) and Microsoft.NETCore.App 8.x
```

### 1.2 On the build machine

- The **.NET SDK** matching your target framework (`dotnet --version`).
- Optionally **Visual Studio 2022** (17.8+ for .NET 8) with the *ASP.NET and web development* workload.

---

## 2. Understand the deployment options

| Choice | Options | Guidance |
|---|---|---|
| **Deployment mode** | *Framework-dependent* vs *Self-contained* | Framework-dependent is smaller and uses the server's shared runtime (patched via Windows/Hosting Bundle updates). Self-contained bundles the runtime and does not require the Hosting Bundle's runtime, but **ANCM is still required**. |
| **Hosting model** | *In-process* (default) vs *Out-of-process* | In-process runs the app inside the IIS worker process (`w3wp.exe`) using IIS HTTP Server – faster. Out-of-process runs Kestrel as a separate process behind IIS as a reverse proxy. |
| **Runtime identifier** | `win-x64`, `win-x86`, `win-arm64`, or portable | Match the server's architecture. In-process hosting requires the app pool bitness to match the app. |

---

## 3. Verify static files will be published

For projects using `Microsoft.NET.Sdk.Web`, **everything under `wwwroot/` is copied to the publish folder automatically** – no manual configuration needed.

Typical layout:

```text
wwwroot/
├── css/
├── js/
├── lib/            # Bootstrap, jQuery, etc. (npm / LibMan / static)
├── images/
├── fonts/
└── <AppName>.styles.css   # Generated scoped-CSS bundle (MVC / Razor Pages)
```

> The `<AppName>.styles.css` file is generated at build/publish time from `*.cshtml.css` / `*.razor.css` files. You do not create it by hand.

Static files outside `wwwroot` (for example, a `Templates` or `Uploads` folder) are **not** published unless declared in the `.csproj`:

```xml
<ItemGroup>
  <Content Include="MyStaticFiles\**" CopyToPublishDirectory="PreserveNewest" />
</ItemGroup>
```

Also make sure `app.UseStaticFiles();` is present in `Program.cs` (it is by default in the standard templates).

### 3.1 Do a local test publish

```powershell
dotnet publish <ProjectFile> -c Release -o .\publish
Get-ChildItem .\publish\wwwroot -Recurse -File | Select-Object -First 20
```

> **If static files are missing after publish**, check for `<Content Remove="..." />` rules in the `.csproj`, `<None Update="..." CopyToPublishDirectory="Never" />` entries, or exclusions in a publish profile.

---

## 4. Publish the application

### 4.1 Option A – `dotnet publish` (recommended, scriptable)

```powershell
dotnet publish <ProjectFile> `
  -c Release `
  -o <PublishPath> `
  --self-contained false `
  --runtime win-x64
```

| Parameter | Meaning |
|---|---|
| `-c Release` | Optimized build. |
| `-o <PublishPath>` | Output folder (on the IIS server, or a staging folder you copy over). |
| `--self-contained false` | Use the runtime installed on the server. Use `true` to bundle the runtime. |
| `--runtime win-x64` | Target runtime identifier. Omit for a portable build. |

> If the solution contains multiple projects, publish the **web project's `.csproj`**, not the `.sln`.

Typical output:

```text
<PublishPath>
├── <AppName>.exe
├── <AppName>.dll
├── web.config                    # Generated by the SDK; required by IIS
├── appsettings.json
├── appsettings.Development.json  # Remove for production
├── wwwroot\
└── (dependency DLLs, *.deps.json, *.runtimeconfig.json)
```

### 4.2 Option B – Visual Studio Folder profile

1. Right-click the web project → **Publish**.
2. Choose **Folder** as the target and set the path (e.g. `<PublishPath>`).
3. Click **Show all settings** and verify:
   - **Configuration**: Release
   - **Target framework**: `<TargetFramework>`
   - **Deployment mode**: Framework-dependent (or Self-contained)
   - **Target runtime**: `win-x64` (or the server's architecture)
   - **Delete all existing files prior to publish**: usually **Off** for incremental deploys (preserves server-only files such as `appsettings.Production.json` and logs).
4. **Save**, then **Publish**.

### 4.3 Option C – Web Deploy / CI-CD

If you use Azure DevOps, GitHub Actions, or Jenkins, publish to a build artifact with `dotnet publish`, copy it to the server (Web Deploy, `robocopy`, SSH/WinRM, or a self-hosted agent), and follow the redeployment steps in [section 10](#10-redeploying-updates-with-minimal-downtime).

---

## 5. Configure the IIS site

### 5.1 Create the site

1. Open **IIS Manager** (`inetmgr`).
2. Right-click **Sites → Add Website…**.
3. Fill in:
   - **Site name**: `<AppName>`
   - **Physical path**: `<PublishPath>` (the folder containing `web.config`, **not** `wwwroot`)
   - **Binding**: `http`, IP `All Unassigned`, port `80`, and a host name if you use one.
4. Click **OK**.

Or via PowerShell:

```powershell
Import-Module WebAdministration

New-WebAppPool -Name "<AppPoolName>"
Set-ItemProperty IIS:\AppPools\<AppPoolName> -Name managedRuntimeVersion -Value ""   # No managed code
Set-ItemProperty IIS:\AppPools\<AppPoolName> -Name managedPipelineMode -Value "Integrated"

New-Website -Name "<AppName>" -PhysicalPath "<PublishPath>" `
  -ApplicationPool "<AppPoolName>" -Port 80
```

### 5.2 Application pool settings

1. Open **Application Pools** and select the pool for the site.
2. **Basic Settings…**
   - **.NET CLR version**: **No Managed Code**
   - **Managed pipeline mode**: **Integrated**
3. **Advanced Settings…**
   - **Identity**: `ApplicationPoolIdentity` (default), or a domain/service account if the app needs Windows authentication to a database or network share.
   - **Enable 32-bit Applications**: `False` for x64 apps, `True` only for x86 builds.
   - **Start Mode**: `AlwaysRunning` and **Idle Time-out**: `0` (optional) to avoid cold starts on low-traffic apps.
   - **Load User Profile**: `True` if the app needs a user profile (e.g. for certificates or the Data Protection key ring).

> **Why "No managed code"?** ASP.NET Core does not use the .NET Framework CLR hosted by IIS. IIS only forwards requests through ANCM.

### 5.3 Folder permissions

The app pool identity needs **Read & Execute** on the publish folder:

```powershell
$path = "<PublishPath>"
$user = "IIS AppPool\<AppPoolName>"

icacls $path /grant "${user}:(OI)(CI)(RX)" /T
```

Grant **Modify** *only* on folders the app writes to (logs, uploads, temp files, SQLite databases, etc.):

```powershell
New-Item -ItemType Directory -Force -Path "<PublishPath>\logs" | Out-Null
icacls "<PublishPath>\logs" /grant "${user}:(OI)(CI)(M)" /T
```

> Prefer storing uploads and persistent data **outside** the publish folder (e.g. `D:\AppData\<AppName>`) so redeployments never overwrite them.

---

## 6. Configure the application

### 6.1 Environment

By default, ASP.NET Core runs as **Production** when `ASPNETCORE_ENVIRONMENT` is not set. To set it explicitly, use either method:

**In `web.config`** (inside the `<aspNetCore>` element):

```xml
<environmentVariables>
  <environmentVariable name="ASPNETCORE_ENVIRONMENT" value="Production" />
</environmentVariables>
```

**Or via IIS Configuration Editor** → section `system.webServer/aspNetCore` → `environmentVariables`.

> A republish regenerates `web.config`, which can overwrite manual edits. Prefer the **Configuration Editor** (stored in `applicationHost.config`) or machine-level environment variables for settings that must survive redeploys. Alternatively, add a `web.config` to your project with the settings you need; the SDK will merge its required `aspNetCore` attributes into it.

### 6.2 Connection strings and secrets

Never commit production credentials to source control. Choose one:

**Option 1 – `appsettings.Production.json`** (created only on the server):

```json
{
  "ConnectionStrings": {
    "Default": "Server=YOUR_SERVER;Database=YOUR_DB;User Id=YOUR_USER;Password=YOUR_PASSWORD;TrustServerCertificate=True;"
  }
}
```

**Option 2 – Environment variables** (override JSON config). Use double underscores for nested keys:

| Name | Value |
|---|---|
| `ConnectionStrings__Default` | `Server=...;Database=...;` |
| `Smtp__Password` | `...` |

Set these in IIS **Configuration Editor → `system.webServer/aspNetCore` → `environmentVariables`**.

**Option 3 – Windows authentication to SQL Server.** Use `Integrated Security=True;` and run the app pool under an identity that has a SQL login, e.g. `IIS APPPOOL\<AppPoolName>` (add it as a login on the SQL Server if the DB is on the same machine) or a domain service account.

**Option 4 – A secrets store** such as Azure Key Vault, HashiCorp Vault, or Windows DPAPI-protected configuration.

### 6.3 Database migrations

How you apply schema changes is app-specific. Common approaches:

| Approach | Notes |
|---|---|
| **Auto-migrate on startup** (EF Core `Database.Migrate()`, FluentMigrator, DbUp) | Convenient, but the DB login needs DDL rights, and a failed migration will cause HTTP 500.30. |
| **Generate SQL scripts** (`dotnet ef migrations script --idempotent`) | Review and run scripts with a DBA-controlled account. Recommended for production. |
| **EF migration bundle** (`dotnet ef migrations bundle`) | Self-contained executable you run during deployment. |

### 6.4 Behind a proxy / load balancer / HTTPS offloading

If TLS terminates at IIS (or an upstream device), make sure the app sees the correct scheme and client IP:

```csharp
builder.Services.Configure<ForwardedHeadersOptions>(o =>
{
    o.ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto;
});

var app = builder.Build();
app.UseForwardedHeaders();
```

> When using IIS **in-process**, the IIS Integration middleware already handles `X-Forwarded-*` for IIS itself. Configure forwarded headers explicitly only if there is another proxy in front of IIS.

### 6.5 Data Protection keys (multi-server / app pool recycling)

If you use cookie authentication, anti-forgery tokens, or `IDataProtector`, keys must persist across restarts and be shared on web farms:

```csharp
builder.Services.AddDataProtection()
    .PersistKeysToFileSystem(new DirectoryInfo(@"D:\AppData\<AppName>\keys"))
    .SetApplicationName("<AppName>");
```

Grant the app pool identity **Modify** on that folder. For a web farm, use a shared network path, database, or Redis.

---

## 7. Start the site and verify

1. In IIS Manager, select the site and click **Start**.
2. Browse to `http://localhost` (or your binding / host name).
3. Check:
   - The home page loads without errors.
   - **Static files**: press `F12` → **Network**; CSS, JS, images, and fonts return `200`.
   - **Application logs** are being written (if configured).
   - **Health endpoint** (if you expose one, e.g. `app.MapHealthChecks("/health")`) returns `Healthy`.
   - **Database features** (login, data pages) work end to end.

---

## 8. Production hardening

1. **Remove development-only files**

   ```powershell
   Remove-Item "<PublishPath>\appsettings.Development.json" -ErrorAction SilentlyContinue
   ```

2. **Use HTTPS**
   - Add an `https` binding on port `443` with a valid certificate (IIS Manager → *Bindings…* → *Add…*).
   - Optionally, redirect HTTP to HTTPS in the app (`app.UseHttpsRedirection();`) or with the **IIS URL Rewrite** module.
   - Enable HSTS in production: `app.UseHsts();`

3. **Security headers** – add via middleware or `web.config` `<customHeaders>`: `X-Content-Type-Options`, `Content-Security-Policy`, `Referrer-Policy`, etc. Remove the `X-Powered-By` header.

4. **Request limits** – default max request body is ~28.6 MB under IIS. For larger uploads, raise both limits:

   ```xml
   <system.webServer>
     <security>
       <requestFiltering>
         <requestLimits maxAllowedContentLength="104857600" /> <!-- 100 MB -->
       </requestFiltering>
     </security>
   </system.webServer>
   ```

   and in code: `builder.WebHost.ConfigureKestrel(o => o.Limits.MaxRequestBodySize = 104857600);` (or `[RequestSizeLimit]` / `FormOptions`).

5. **Compression and caching** – enable IIS static/dynamic compression, or use `app.UseResponseCompression()`. Use `asp-append-version="true"` on `<link>` / `<script>` tags for cache-busting.

6. **Least privilege** – no `db_owner` for runtime DB users once the schema exists; give the app pool identity only the folders it needs.

7. **Windows Firewall** – open only required ports (80/443).

8. **Stdout logging (temporary only)** for diagnosing startup failures:

   ```xml
   <aspNetCore processPath="dotnet"
               arguments=".\<AppName>.dll"
               stdoutLogEnabled="true"
               stdoutLogFile=".\logs\stdout"
               hostingModel="inprocess" />
   ```

   Create the `logs` folder, grant **Modify** to the app pool identity, and **turn logging off** after debugging to avoid filling the disk.

---

## 9. Common issues and fixes

| Error / symptom | Likely cause | Fix |
|---|---|---|
| **HTTP 404.0 / 404.15 on all routes** | Wrong physical path (pointing to `wwwroot` or an empty folder), or `web.config` missing. | Point the site to the publish root containing `web.config`. |
| **HTTP 500.0 – ANCM In-Process Handler Load Failure** | Hosting Bundle missing/mismatched, or the app targets a runtime not installed. | Install the matching Hosting Bundle and run `iisreset`. Check `dotnet --list-runtimes`. |
| **HTTP 500.19** | Invalid `web.config`, or ANCM module not installed. | Validate `web.config` XML; install/repair the Hosting Bundle (after IIS). |
| **HTTP 500.30 – In-Process Start Failure** | App throws on startup (config, DB, migration, missing permissions). | Check **Event Viewer → Windows Logs → Application**, the stdout log, and app logs. Run `dotnet <AppName>.dll` from a console under the same environment to see the exception. |
| **HTTP 500.31 – Failed to find native dependencies** | Required .NET runtime not installed. | Install the correct runtime / Hosting Bundle. |
| **HTTP 500.32 – Unable to load .NET runtime / bitness mismatch** | App is x86 but pool is x64 (or vice versa). | Match **Enable 32-Bit Applications** to the build's RID. |
| **HTTP 500.35 – Multiple in-process apps in same pool** | Two in-process apps share one app pool. | Use a **separate app pool per in-process app**. |
| **HTTP 502.5 – Process Failure (out-of-process)** | Kestrel process failed to start. | Run the DLL manually and check logs; verify `processPath` and `arguments`. |
| **HTTP 403.14** | No default document and directory browsing disabled; site points to the wrong folder. | Point to the publish root; ensure ANCM handler is mapped (`*` → `AspNetCoreModuleV2`). |
| **HTTP 413 / upload fails** | Request-size limit exceeded. | Raise `maxAllowedContentLength` and Kestrel/form limits (section 8). |
| **Static files (CSS/JS) return 404** | `wwwroot` not published, `UseStaticFiles()` missing, or wrong path. | Re-publish; confirm `wwwroot` exists in the output and middleware is configured. |
| **Old CSS/JS after deploy** | Browser or proxy cache. | Hard refresh (`Ctrl+F5`) and use `asp-append-version="true"`. |
| **Login / antiforgery errors after restart** | Data Protection keys are not persisted. | Persist keys to a stable path (section 6.5). |
| **"Access denied" writing files** | App pool identity lacks write permission. | Grant **Modify** on the specific folder only. |
| **Database login failed / not reachable** | Wrong connection string, firewall, or SQL permissions. | Override the connection string; test with `sqlcmd` from the IIS server. |
| **Site works locally but not remotely** | Binding, firewall, or DNS. | Verify bindings, open port 80/443, and check DNS. |
| **WebSockets / SignalR fail** | WebSocket Protocol feature not installed. | Install `Web-WebSockets` and restart IIS. |

---

## 10. Redeploying updates with minimal downtime

ASP.NET Core in-process apps lock their DLLs while running. To avoid "file in use" errors, take the app offline during file copy.

**Using `app_offline.htm`** (supported by ANCM):

```powershell
$site = "<PublishPath>"

# 1. Take the app offline (ANCM shuts the app down and serves this page)
Set-Content "$site\app_offline.htm" "<html><body><h1>Updating… back shortly.</h1></body></html>"
Start-Sleep -Seconds 3

# 2. Copy new files (exclude server-only files and data)
robocopy .\publish $site /MIR /XD logs uploads keys /XF appsettings.Production.json

# 3. Bring it back online
Remove-Item "$site\app_offline.htm"
```

> **Careful with `/MIR`** – it deletes anything in the destination that is not in the source. Always exclude server-only folders/files (`logs`, `uploads`, `appsettings.Production.json`, etc.), or use `/E` instead (copy without deleting).

For zero-downtime deployments, use two sites/slots behind a load balancer (blue/green) or Web Deploy with `-enableRule:AppOffline`.

**Rollback:** keep the previous publish output (e.g. `<PublishPath>_prev` or a timestamped folder) and swap the site's physical path, or restore from the previous artifact.

---

## 11. Quick checklist

**Server**
- [ ] IIS installed (with WebSocket Protocol if needed).
- [ ] .NET Hosting Bundle (matching major version) installed **after** IIS; `iisreset` run.
- [ ] Database/external services reachable from the server.

**Build & publish**
- [ ] `dotnet publish <ProjectFile> -c Release` succeeds.
- [ ] `wwwroot` and `web.config` exist in the publish output.
- [ ] `appsettings.Development.json` removed from the server.

**IIS**
- [ ] Site created, pointing to the publish root (not `wwwroot`).
- [ ] App pool: **No Managed Code**, **Integrated**, correct bitness, one pool per in-process app.
- [ ] App pool identity: **Read & Execute** on the app folder, **Modify** only on logs/uploads/keys.

**Configuration**
- [ ] `ASPNETCORE_ENVIRONMENT=Production` set.
- [ ] Connection strings and secrets supplied via server-only config / environment variables / a secret store.
- [ ] Database schema applied (auto-migration or reviewed scripts).
- [ ] Data Protection keys persisted (if applicable).
- [ ] HTTPS binding with a valid certificate; HTTP→HTTPS redirect configured.

**Verification**
- [ ] Home page loads; static files return `200`.
- [ ] Health endpoint (if any) returns healthy.
- [ ] Logs are written; stdout logging turned off.
- [ ] Rollback plan / previous build retained.
0 words  •  0 characters
