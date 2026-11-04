# Eagle Clean

> Proof-of-work tracking for custodial routes — a .NET MAUI mobile app plus a web portal.
> Built for **USPS** in association with **IPC Mobile**.

- **Mobile app:** `Custodian/` (.NET MAUI, Android)
- **Web portal:** https://eagleclean-fe.azurewebsites.net/
- **Backend API:** https://eagleclean-be.azurewebsites.net/swagger/index.html
- **Demo video:** [[EagleClean Demo](./EagleClean%20Demo.mp4)](https://drive.google.com/file/d/1USpzS4oNzE11u0VZUxsJqe2rTxwGEkvx/view?usp=sharing)


---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
- [Repository Layout](#repository-layout)
- [Prerequisites](#prerequisites)
- [Setup & Run](#setup--run)
- [Configuration](#configuration)
- [Development Notes](#development-notes)
- [Security Notes](#security-notes)
- [Known Limitations](#known-limitations)
- [Contributing](#contributing)
- [License](#license)
- [Acknowledgements](#acknowledgements)

---

## Overview

Eagle Clean is a proof-of-work solution for custodians. A custodian logs in with their USPS badge, follows an assigned cleaning route, completes the route's tasks (with planned vs. actual times), captures evidence (photos, signatures, barcodes), and submits a signed record. Records are stored locally on the device and synchronized to the backend when connectivity is available, so work continues offline.

The mobile app is used in the field on IPC mobiles; the web portal is used to review submitted proof-of-work, routes, and (TODO: describe portal audience — supervisors / admins / auditors).

## Features

### Mobile app (Android)

- **Badge-based login** — 7- or 12-digit USPS badge IDs validated with a Mod 10 check digit; supports manual entry and CODE128 barcode scanning.
- **User agreement & signature capture** — onboarding agreement plus a reusable signature pad (Syncfusion) saved per badge.
- **Home dashboard** — counts of partial vs. completed routes for the day and quick actions.
- **Job scanning** — scan a job/facility barcode to start work.
- **Nearby facilities** — discovers facilities within a configurable GPS radius (default 5,000 m) of the device.
- **Daily schedule** — view assigned routes for the day.
- **Active route execution** — task checklist with planned time, actual time, completion toggles, start/end GPS, and start/end barcodes.
- **Ad-hoc work** — record work not tied to a scheduled route.
- **Travel / map navigation** — turn-by-turn style travel page between stops.
- **Time clock** — clock in/out.
- **My Work** — review and reopen today's partial and completed routes.
- **Evidence capture** — add pictures to a route; signatures.
- **Safety Data Sheets (SDS)** — in-app viewer for SDS sheets.
- **Training** — CTC monthly training, job aids, and training videos.
- **Offline-first sync** — records are written to local storage (`ToUpload/`) and uploaded by a background thread; a sync indicator shows upload status.
- **Location tracking** — GPS sampled every minute and persisted (last known location restored on failure).
- **Diagnostics** — diagnostic menu and global exception/unhandled-error logging.

### Web portal

- Hosted at https://eagleclean-fe.azurewebsites.net/
- Backed by the `EagleClean.API` surface, which suggests portal capabilities for:
  - User/role management and sign-in (`/user/*`, `/users/*`, `/user/roles`)
  - Facility and route management (`/facility/*`, `/facilities/*`, `/routes/*`)
  - Custodian management and supervisor comments (`/custodian/*`, `/custodians/*`)
  - Supervisor dashboards, radar upload tracking, certifications approval (`/supervisor/*`)
  - Completion/reporting dashboards (`/reports/*`, `/userprofile/mydashboard/*`)
  - Helpdesk ticketing (`/helpdesk/ticket/*`)
- _Confirm which of these are exposed in the portal UI — details to be documented by the maintainer._

### Backend API

- **API:** `EagleClean.API` v1 — ASP.NET Core Web API documented with OpenAPI/Swagger
  - Swagger UI: https://eagleclean-be.azurewebsites.net/swagger/index.html
  - OpenAPI JSON: https://eagleclean-be.azurewebsites.net/swagger/v1/swagger.json
- **Auth:** two-layer scheme observed against the live API:
  - **Every endpoint requires an `ECApiKey` header** (shared API key; missing/invalid key returns 401 "Api Key was not provided"/"Api Key is not correct").
  - **User/role endpoints additionally require `Authorization: Bearer <token>`** (token obtained via `/user/signin/{userName}/{password}`), returning `WWW-Authenticate: Bearer` when absent.
  - `/user/signin/{userName}/{password}` itself requires the `ECApiKey` header but no Bearer token.
- **Major endpoint groups** (see Swagger for full contracts):
  - **Auth/Users:** `/user/signin`, `/user/register`, `/user/{userName}`, `/user/save`, `/users`, `/user/roles`, `/users/superadmins`, `/user/changestatus/{userName}/{isActive}`
  - **Facilities:** `/facility/*`, `/facilities/*` (search by lat/long/radius/date, details, schedules, routes, staffing, 4776/4852 notes)
  - **Routes:** `/routes/{facilityId}/...`, `/route/{facilityId}/{routeId}`
  - **Work records:** `/workrecord/merge`, `/workrecords/completed/{facilityId}/{date}`
  - **Custodians:** `/custodians/...`, `/custodian/save`, `/custodian/comments/...`, `/custodian/changestatus/...`
  - **Supervisors:** `/supervisor/*`, `/supervisors`
  - **Locations:** `/location/current/save`, `/location/gps/event/add`
  - **Reports:** `/reports/*` (completion, facility/district/region/custodian, 4776 daily/weekly, work records, TMK summary)
  - **Helpdesk tickets:** `/helpdesk/ticket/*` (CRUD, comments, attachments, history, assignment, status)
  - **Guaranteed upload:** `/guaranteedupload/*` (device info, errors, tmk records)
  - **CW import:** `/cwimport/schedule/{projectId}/{facilityId}/{year}`
- **Mobile app usage:** the MAUI app currently calls only:
  - `GET /facilities/{latitude}/{longitude}/{radius}/{date}` — nearby facilities (app currently omits `{date}`; verify against deployed API)
  - `POST /workrecord/merge` — submit a `WorkRecordSaveDTO` (`{ id, json }`)

## Architecture

### High-level

```
┌────────────────────┐        HTTPS/JSON        ┌────────────────────┐
│  MAUI Android app  │  ─────────────────────►  │  ASP.NET Core API  │
│  (custodian device)│  ◄─────────────────────  │  (Azure App Svc)   │
└────────┬───────────┘                          └────────┬───────────┘
         │ local JSON files                               │
         │ (ToUpload / Uploaded)                          ▼
         │                                          ┌────────────┐
         │                                          │  Database  │
         │                                          └────────────┘
         ▼
┌────────────────────┐
│  Web portal (SPA?) │  ◄── reviews / reports
└────────────────────┘
```


### Mobile app

- **Framework:** .NET MAUI 7, single project, `net7.0-android` target, min Android SDK 21.
- **Pattern:** MVVM via `CommunityToolkit.Mvvm` + `CommunityToolkit.Maui`; Shell navigation (`AppShell`); DI registered in `MauiProgram`.
- **Services:**
  - `IApiClientService` / `ApiClientService` — HttpClient wrapper over the backend base URL.
  - `IFacilityService` / `FacilityService` — facility lookup.
  - `IProofOfWorkService` / `ProofOfWorkSerive` — submit work records (`workrecord/merge`).
  - `ILocationService` / `LocationService` — GPS sampling & persistence.
  - `UploadThread` — background sync of offline records.
- **Offline storage:** PCLStorage rooted at `/storage/emulated/0/Custodian/`:
  - `Custodian/config.json` — `Config` (radius, API base URL)
  - `Custodian/Data/ToUpload/{guid}_{yyyyMMdd}.json` — pending records
  - `Custodian/Data/Uploaded/{guid}_{yyyyMMdd}.json` — uploaded records
  - `Custodian/Location/User_Location.json` — last known GPS
  - `Custodian/Signatures/{badgeId}.png` — signature image
- **Record lifecycle:** a route becomes a `MergeRecord` with `seq` 1 (started) → 2/3 (partial) → 4 (completed); `filename` ties it to disk; `UploadThread` uploads and moves files to `Uploaded/`.
- **Background work:** two long-running threads — location sampling (1 min) and upload polling.

### Data model (mobile)

| Type | Purpose |
|---|---|
| `Route` | Assigned route: facility, route name, steps with estimated minutes. |
| `Task` | A step with planned/actual time and completion flag. |
| `MergeRecord` | Serializable proof-of-work record (start/end time, GPS, barcode, tasks, photos, status, seq). |
| `WorkRecord` | Wrapper pairing a GUID/filename with a JSON record (used for offline queues). |
| `Facility` | Facility metadata (id, type, address, lat/long, distance). |
| `Config` | Local config (search radius, API base URL). |

### Web portal / backend

- **Backend:** ASP.NET Core Web API (`EagleClean.API` v1) hosted on Azure App Service, documented via Swagger/OpenAPI.
- **Auth:** Bearer token + `ECApiKey` header; user accounts/roles managed via `/user/*` and `/users/*` endpoints.
- **Portal:** https://eagleclean-fe.azurewebsites.net/ — Blazor Server app (ASP.NET Core), using Bootstrap, Syncfusion Blazor, and Radzen Blazor components.
- **Data store:** SQL Server.

## Repository Layout

```
Eagle-Clean/
├── README.md
├── Custodian.sln
├── key.keystore                 # Android signing keystore (dev)
└── Custodian/                   # .NET MAUI app
    ├── Custodian.csproj
    ├── MauiProgram.cs            # DI + app bootstrap
    ├── App.xaml(.cs)             # startup, global exception handler
    ├── AppShell.xaml             # Shell navigation
    ├── Pages/                    # XAML pages
    ├── ViewModels/
    ├── Services/                 # API, facility, proof-of-work
    ├── Helpers/                  # filesystem, location, signatures, upload thread, utils
    ├── Models/                   # Route, MergeRecord, Facility, ...
    ├── Controls/  Popups/  Screens/  Messages/  ActivityLog/
    └── Resources/                # images, fonts, raw assets
```


## Prerequisites

- .NET 7 SDK with the .NET MAUI workload (`dotnet workload install maui`)
- Visual Studio 2022 (17.x) with the ".NET MAUI" workload, or the MAUI CLI
- Android SDK (min API 21) and an emulator/device
- Syncfusion MAUI license key (used in `App.OnStart`)

## Setup & Run

### 1. Clone

```bash
git clone https://github.com/mfaizey/Eagle-Clean.git
cd Eagle-Clean
```

### 2. Configure

- Set the backend base URL in `Custodian/Services/Client/ApiClientService.cs` (`_baseUrl`) and/or in the device's `config.json`.
- Register the Syncfusion license in `Custodian/App.xaml.cs` (`OnStart`) — replace with your own key.

### 3. Build & run the mobile app

```bash
cd Custodian
dotnet restore
dotnet build Custodian.csproj -f net7.0-android
```

Then deploy to an emulator/device from Visual Studio, or:

```bash
dotnet build Custodian.csproj -f net7.0-android -t:Run
```



## Configuration

| Setting | Location | Default / Notes |
|---|---|---|
| API base URL | `ApiClientService._baseUrl` / `config.json` | `https://eagleclean-be.azurewebsites.net` (Swagger: `/swagger/index.html`) |
| Search radius (m) | `config.json` (`Radius`) | `5000` |
| Android signing keystore | `Custodian.csproj` | `../key.keystore` (password `123456` in repo — dev only) |
| Syncfusion license | `App.xaml.cs` `OnStart` | hardcoded (replace for production) |
| Local data root | `Utils.ROOT_PATH` | `/storage/emulated/0/` |

## Development Notes

- Logging goes through `ActivityLog/Logger` and `RecordLogger` (records download/upload payloads).
- Global unhandled exceptions are captured in `App.CurrentDomain_UnHandledException`.
- Only **today's** records (`yyyyMMdd` match) are loaded/uploaded; older files remain on disk.
- `UploadThread` sleeps when the queue is empty and wakes on new records.
- `LocationService` samples GPS every minute and falls back to the last persisted location.
- Badge IDs must be 7 or 12 digits and pass the Mod 10 check digit (`Utils.IsBadgeValid`).

## Security Notes

- The repo currently ships a signing keystore and a Syncfusion license key for development. **Do not reuse these in production.**
- Mobile login is badge-based (7/12-digit USPS badge with Mod 10 check digit) plus a captured signature; no password is used on-device.
- Portal/API auth uses Bearer tokens and an `ECApiKey` header; user accounts, roles, and status are managed via `/user/*` and `/users/*`.






## Acknowledgements

- USPS and IPC Mobile for the project and devices.
- Syncfusion for MAUI components (SignaturePad, ProgressBar).
- .NET MAUI and the CommunityToolkit teams.

---

## Suggested Repository Metadata



- **Description:** .NET MAUI (Android) + web portal for proof-of-work tracking of custodial routes; built for USPS with IPC mobiles.
- **Suggested topics:** `blazor`,`dotnet-maui`, `maui`, `android`, `xamarin-alternative`, `proof-of-work`, `usps`, `offline-first`, `azure`, `csharp`, `mvvm`

Maintainer: @mfaizey 
