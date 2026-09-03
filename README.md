# DeployLab

DeployLab is a local deployment laboratory for the BikeShop application.

The goal is to learn how an ASP.NET Core application is prepared, deployed, configured, updated, and diagnosed in a production-like Linux environment.

The target server is a ThinkPad T450s running Ubuntu in a home network. It replaces a public VPS for this project, so the deployment can be studied without external hosting, payment, or domain requirements.

## Application Under Deployment

BikeShop is a separate ASP.NET Core project. Its source code is not copied into this repository.

DeployLab contains the deployment documentation, server configuration templates, and scripts required to deploy BikeShop from a development machine to the Ubuntu server.

## Target Architecture

```text
Desktop development machine
    │
    │ publish and deploy
    ▼
ThinkPad Ubuntu server
    │
    ├── Nginx
    │     ├── BikeShop.Blazor running with Kestrel
    │     └── BikeShop.API running with Kestrel
    │
    └── SQL Server
```

Nginx is the only public entry point in the local network. The Blazor application and API run as separate systemd services. Kestrel listens only on the server itself.

## Learning Goals

* Understand `dotnet publish` and the difference between Debug and Release builds.
* Understand SDK, Runtime, framework-dependent, and self-contained deployment.
* Configure production settings, environment variables, and connection strings.
* Apply database migrations on a server.
* Run ASP.NET Core applications as Linux services with systemd.
* Configure Nginx as a reverse proxy for Kestrel.
* Deploy and update an application without Visual Studio.
* Diagnose common deployment problems using logs and Linux tools.
* Understand the role of HTTPS and how local HTTPS differs from public HTTPS.

## Scope

This project focuses on deployment inside a home network.

A public VPS, domain name, Let's Encrypt certificate, and CI/CD are outside the required scope. They may be added in a future project when a suitable hosting option is available.

## Repository Contents

* `docs/` — architecture notes, setup instructions, deployment runbook, and troubleshooting notes.
* `scripts/` — repeatable publish and deployment scripts.
* `nginx/` — Nginx configuration templates.
* `systemd/` — systemd service unit templates.
* `config/` — safe configuration examples without secrets.

No passwords, JWT keys, production connection strings, or published application files are committed to this repository.

## Status

In progress.
