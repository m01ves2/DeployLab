# Publishing BikeShop

## Purpose

This document explains how BikeShop is built and published before deployment to the Ubuntu server.

BikeShop has two deployable ASP.NET Core applications:

* `BikeShop.Blazor` - the Blazor Server user interface.
* `BikeShop.API` - the Web API used by the UI.

The remaining projects are class libraries. They are included as dependencies of the deployable applications and are not published separately.

## Target Framework

All BikeShop projects target `.NET 8`:

```xml
<TargetFramework>net8.0</TargetFramework>
```

The server must therefore have a compatible ASP.NET Core Runtime 8 installed.

## Build and Publish

`dotnet build` compiles the source code.

```bash
dotnet build BikeShop.sln -c Release
```

The build output is created in each project's `bin/Release/net8.0/` directory.

`dotnet publish` creates a deployable folder. It builds the project and copies its dependencies, configuration files, static assets, and runtime metadata into one location.

```bash
dotnet publish BikeShop.Blazor/BikeShop.Blazor.csproj -c Release
dotnet publish BikeShop.API/BikeShop.API.csproj -c Release
```

The resulting folders are:

```text
BikeShop.Blazor/bin/Release/net8.0/publish/
BikeShop.API/bin/Release/net8.0/publish/
```

Only these publish folders will be copied to the server.

## Debug and Release

`Debug` is the default configuration for local development and debugging.

`Release` is the configuration used for deployment. It enables compiler optimizations but does not automatically select the Production environment.

The environment is selected separately. When an application is started outside Visual Studio without an environment variable, ASP.NET Core uses `Production`.

## Publish Output

The publish folders contain:

* application DLL files and their dependencies;
* `.runtimeconfig.json`, which specifies the required .NET Runtime;
* `.deps.json`, which describes dependencies;
* `.pdb` files for readable stack traces;
* `appsettings.json`;
* static files such as Blazor `wwwroot`, images, CSS, and JavaScript.

The Windows `.exe` files in the publish folders are Windows launchers. They are not used on Ubuntu.

The Linux server will start the applications through their DLL files:

```bash
dotnet BikeShop.Blazor.dll
dotnet BikeShop.API.dll
```

## Local Smoke Test

A smoke test verifies that the published files can run without Visual Studio or source code.

### API

From the API publish directory:

```bash
dotnet BikeShop.API.dll --urls http://127.0.0.1:5110
```

Expected result:

```text
Now listening on: http://127.0.0.1:5110
Hosting environment: Production
```

Check that Kestrel responds:

```bash
curl -i http://127.0.0.1:5110
```

`404 Not Found` is expected because the API has no endpoint at `/`. The response still confirms that Kestrel and the published API are running.

During this HTTP-only test, `UseHttpsRedirection()` logs a warning because no HTTPS port is configured. HTTPS will be handled later by Nginx.

### Blazor

From the Blazor publish directory:

```bash
dotnet BikeShop.Blazor.dll --urls http://127.0.0.1:5120
```

Check that it returns HTML:

```bash
curl -i http://127.0.0.1:5120
```

Expected result: `200 OK` with `Server: Kestrel`.

## Verified Result

BikeShop was successfully built and published in Release mode on the Desktop development machine.

* Blazor publish folder size: approximately 6.2 MB.
* API publish folder size: approximately 21 MB.
* Both applications started successfully from their publish folders in the Production environment.

## Notes

Publish folders are generated artifacts. They are not committed to the BikeShop or DeployLab repositories.

The server does not need BikeShop source code, Visual Studio, or the .NET SDK. It needs only the published files and the ASP.NET Core Runtime.
