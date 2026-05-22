# PrivateCloud — End-to-End Private IaaS Platform

A fully functional private cloud platform implementing Infrastructure as a Service (IaaS) on a single machine. Built with FastAPI, Angular 17, and VirtualBox — demonstrating multi-tenancy, self-service VM provisioning, resource quota enforcement, real-time monitoring, audit logging, and a browser-based SSH terminal.

---

## Table of Contents

1. [Tech Stack](#tech-stack)
2. [Features](#features)
3. [Architecture](#architecture)
4. [Database Schema](#database-schema)
5. [Project Structure](#project-structure)
6. [Prerequisites](#prerequisites)
7. [Setup & Installation](#setup--installation)
8. [Environment Variables](#environment-variables)
9. [Default Credentials](#default-credentials)
10. [Running the Application](#running-the-application)
11. [Access URLs](#access-urls)
12. [API Reference](#api-reference)
13. [Background Jobs](#background-jobs)
14. [Security Model](#security-model)
15. [Running Tests](#running-tests)
16. [Database Migrations](#database-migrations)
17. [Demo Mode](#demo-mode)
18. [IaaS Concepts Demonstrated](#iaas-concepts-demonstrated)
19. [Troubleshooting](#troubleshooting)
20. [Academic Reference](#academic-reference)

---

## Tech Stack

| Layer | Technology | Version |
|---|---|---|
| Frontend | Angular + Angular Material + Chart.js (ng2-charts) | 17 |
| Backend | FastAPI (Python) + Uvicorn ASGI | 0.111.0 / 0.29.0 |
| Hypervisor | Oracle VirtualBox (Type-2) via VBoxManage CLI | 7.x |
| Database | SQLite in WAL mode via SQLAlchemy ORM | 2.0.30 |
| Migrations | Alembic | 1.13.1 |
| Auth | JWT (python-jose) + bcrypt (passlib) | — |
| Scheduling | APScheduler (AsyncIOScheduler) | 3.10.4 |
| Real-time | WebSocket — VM status stream + SSH terminal | — |
| SSH Terminal | paramiko (server-side) + xterm.js (browser) | 4.0.0 |
| Monitoring | psutil | 5.9.8 |
| Validation | Pydantic v2 | 2.7.1 |
| Tests | pytest + unittest.mock | 8.2.0 |

---

## Features

### Virtual Machine Management
- Create VMs with configurable vCPUs, RAM, disk size, OS type, and boot ISO
- Full VM lifecycle: **start, stop, pause, resume, reset, delete**
- **Network modes (Issue 15):** NAT (SSH port-forwarding), Host-Only (direct IP on `vboxnet0`), Bridged (VM on physical LAN)
- Each NAT VM gets a unique SSH port mapped from the host to guest port 22
- Disk images stored as VDI files under `~/VirtualBox VMs/<name>/`
- Boot order: DVD → Disk

### VM Templates (Issue 14)
- Save any VM's configuration as a reusable template (`POST /api/v1/vms/templates/`)
- Create templates manually (supply `vcpus`, `ram_mb`, `disk_gb`) or by copying an existing VM (`source_vm_id`)
- Provision a new VM from a template with a single call (`POST /api/v1/vms/templates/{id}/provision?name=...`)
- Templates are owned per-user; admins can see all templates

### Browser SSH Terminal
- `xterm.js` terminal embedded in the VM detail page
- WebSocket proxy (`/api/v1/vms/{id}/ws/terminal`) uses `paramiko` to connect browser → VM
- Terminal appears automatically when the VM reaches running state
- Copy-to-clipboard SSH command button (`ssh user@localhost -p <port>`)
- **Idle timeout (Issue 16):** sessions with no keystrokes for 30 minutes are automatically closed (configurable via `TERMINAL_IDLE_TIMEOUT`)

### Real-time VM Status
- WebSocket endpoint `/api/vms/ws/status` pushes all VM states every 3 seconds
- JWT passed as a query parameter (`?token=...`) since browsers cannot set WebSocket headers
- Status badge on the VM detail and list pages updates without any page refresh

### Resource Quota System
- Per-user limits: `max_vcpus`, `max_ram_mb`, `max_disk_gb`, `max_vms`
- Default limits: 4 vCPUs, 8 192 MB RAM, 100 GB disk, 5 VMs per user
- Quota is validated against both the user's remaining allocation and actual host availability
- `used_*` counters incremented/decremented atomically within the same DB transaction as the VM record
- Quota usage bars visible in the Create VM form and user profile page

### Real-time & Historical Monitoring
- Live host CPU, RAM, and disk metrics polled every 5 seconds from `GET /api/monitoring/host`
- Historical metrics stored every 30 seconds in a `host_metrics_history` time-series table
- Records older than 24 hours are automatically pruned
- Dashboard charts support **1h / 6h / 24h** time window toggle

### Role-Based Access Control (RBAC)
- Two roles: `ADMIN` and `USER` (stored as a SQLAlchemy Enum)
- `AdminGuard` on Angular routes blocks non-admin navigation client-side
- `require_admin` FastAPI dependency enforces server-side; returns HTTP 403 for non-admins
- Deactivated users (`is_active=False`) are rejected even with a valid JWT

### Admin Panel
- **User table** — sort, filter, role chips, activate/deactivate toggle
- **Quota editor** — dialog with sliders for all four resource limits per user
- **Role editor** — promote/demote users between ADMIN and USER
- **Audit log viewer** — paginated, filterable by user, action type, and date

### Audit Logging
- Every mutating operation is logged with: user ID, action, resource type, resource ID, client IP, and UTC timestamp
- Logged actions: `USER_REGISTER`, `USER_LOGIN`, `VM_CREATE`, `VM_ACTION`, `VM_DELETE`, `QUOTA_UPDATE`, `ROLE_UPDATE`, `USER_STATUS_UPDATE`
- Implemented in `audit_service.py` via a reusable `log_action()` helper

### Background Jobs (APScheduler)
- **VM state sync** every 30 s — queries `VBoxManage showvminfo` for each non-transient VM; corrects DB drift
- **Metrics snapshot** every 30 s — records a `HostMetricsHistory` row via `psutil`; prunes rows older than 24 h
- **Quota reconciliation** every 5 min — recomputes `used_*` counters from live VMs to correct any drift
- **Token blocklist cleanup** every 1 h — deletes expired JWT entries from `token_blocklist`
- All jobs run on the `AsyncIOScheduler` started in the FastAPI `lifespan` context manager

### Demo Mode
- Set `DEMO_MODE=true` in `.env` to run without VirtualBox installed
- A `DemoVBoxManager` class replaces all VBoxManage calls with realistic in-memory responses
- All VM operations return fake UUIDs, ports, and states — suitable for demos on any machine

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                    User Browser                      │
│                                                     │
│  Angular 17 SPA                                     │
│  ├── AuthService      (JWT storage + decode)        │
│  ├── VmService        (CRUD + actions HTTP)         │
│  ├── WebSocketService (status stream)               │
│  ├── MonitoringService (live + history HTTP)        │
│  └── AdminService     (users, quotas, audit logs)   │
└────────────────┬────────────────────────────────────┘
                 │  HTTP REST + WebSocket
                 ▼
┌─────────────────────────────────────────────────────┐
│           FastAPI (uvicorn, port 8000)               │
│                                                     │
│  Routers                                            │
│  ├── /api/auth        JWT issue / verify            │
│  ├── /api/vms         VM CRUD + lifecycle actions   │
│  │       └── /ws/status      WebSocket status push  │
│  │       └── /{id}/ws/terminal  SSH proxy           │
│  ├── /api/monitoring  Live metrics + history        │
│  └── /api/admin       Users, quotas, audit (admin)  │
│                                                     │
│  Services                                           │
│  ├── VMService        quota validation + VBox calls │
│  ├── AuthService      bcrypt + JWT                  │
│  ├── AuditService     append-only log_action()      │
│  └── MonitorService   psutil wrapper                │
│                                                     │
│  APScheduler (every 30 s)                           │
│  ├── sync_vm_states()     → VBoxManage → DB correct │
│  └── record_metrics_snapshot() → psutil → DB        │
└──────────────┬──────────────────┬───────────────────┘
               │                  │
               ▼                  ▼
┌──────────────────┐   ┌──────────────────────────────┐
│  SQLite (WAL)    │   │   Oracle VirtualBox 7.x       │
│  cloud.db        │   │                              │
│  ├── users       │   │  VBoxManage CLI              │
│  ├── resource_   │   │  ├── createvm / modifyvm     │
│  │   quotas      │   │  ├── startvm / controlvm     │
│  ├── virtual_    │   │  ├── showvminfo              │
│  │   machines    │   │  └── unregistervm / closemedium│
│  ├── host_       │   │                              │
│  │   metrics_    │   │  Guest VMs (NAT networking)  │
│  │   history     │   │  SSH port-forwarded to host  │
│  └── audit_logs  │   └──────────────────────────────┘
└──────────────────┘

Alembic manages schema migrations (2 revisions)
```

---

## Database Schema

### `users`
| Column | Type | Notes |
|---|---|---|
| id | Integer PK | Auto-increment |
| email | String(255) | Unique, indexed |
| username | String(100) | Unique, indexed |
| hashed_password | String(255) | bcrypt hash |
| full_name | String(255) | Optional |
| role | Enum | `admin` \| `user` |
| is_active | Boolean | Default `true`; deactivated users rejected at login |
| created_at / updated_at | DateTime | UTC, server-managed |

### `virtual_machines`
| Column | Type | Notes |
|---|---|---|
| id | Integer PK | |
| name | String(100) | Indexed |
| vbox_uuid | String(36) | UUID assigned by VBoxManage |
| vcpus / ram_mb / disk_gb | Integer | Provisioned specs |
| os_type | String(50) | VBoxManage OS type string (e.g. `Ubuntu_64`) |
| iso_file | String(255) | Boot ISO filename |
| network_mode | Enum | `nat` \| `bridged` \| `hostonly` (default `nat`) |
| mac_address | String(17) | Assigned by VBoxManage |
| ip_address | String(15) | Populated by guest IP lookup |
| ssh_port | Integer | Host-side NAT port mapped to guest :22 (NAT mode only) |
| status | Enum | `creating` \| `running` \| `stopped` \| `paused` \| `deleting` \| `error` |
| owner_id | FK → users | Ownership enforced at query level |
| created_at / updated_at / started_at | DateTime | UTC |

### `vm_templates`
| Column | Type | Notes |
|---|---|---|
| id | Integer PK | |
| name | String(100) | Template display name |
| description | Text | Optional |
| vcpus / ram_mb / disk_gb | Integer | Saved resource specs |
| os_type | String(50) | VBoxManage OS type |
| iso_file | String(255) | Optional boot ISO |
| network_mode | Enum | `nat` \| `bridged` \| `hostonly` |
| owner_id | FK → users | Template owner |
| created_at | DateTime | UTC |

### `resource_quotas`
| Column | Type | Notes |
|---|---|---|
| user_id | FK → users (unique) | One quota row per user |
| max_vcpus / max_ram_mb / max_disk_gb / max_vms | Integer | Admin-configurable caps |
| used_vcpus / used_ram_mb / used_disk_gb / used_vms | Integer | Maintained transactionally with VM operations |

### `host_metrics_history`
| Column | Type | Notes |
|---|---|---|
| cpu_percent / ram_percent / disk_percent | Float | Host-level percentages |
| ram_used_mb | Integer | Absolute RAM usage |
| vm_count_running | Integer | Running VM count at snapshot time |
| recorded_at | DateTime | Indexed; rows older than 24 h are pruned automatically |

### `audit_logs`
| Column | Type | Notes |
|---|---|---|
| user_id | FK → users | Who performed the action |
| action | String(100) | e.g. `VM_CREATE`, `USER_LOGIN` |
| resource_type / resource_id | String | What was acted on |
| details | Text | JSON or human-readable extra context |
| ip_address | String(45) | Client IP (supports IPv6) |
| timestamp | DateTime | UTC, server-managed |

---

## Project Structure

```
project/
├── backend/
│   ├── app/
│   │   ├── main.py              # FastAPI app, CORS, GZip, lifespan, APScheduler
│   │   ├── config.py            # Settings — reads from .env via pydantic-settings
│   │   ├── database.py          # SQLAlchemy engine + SessionLocal + Base
│   │   ├── dependencies.py      # get_db, get_current_user, require_admin
│   │   ├── models.py            # ORM models (User, VirtualMachine, ResourceQuota, ...)
│   │   ├── schemas.py           # Pydantic request/response schemas
│   │   ├── routers/
│   │   │   ├── auth.py          # Register, login, /me, /quota
│   │   │   ├── vms.py           # VM CRUD, lifecycle actions, WS status, SSH terminal
│   │   │   ├── monitoring.py    # Live metrics, history endpoint, ISO list
│   │   │   └── admin.py         # User mgmt, quota editor, audit logs (admin only)
│   │   └── services/
│   │       ├── vm_service.py    # VBoxManager, DemoVBoxManager, VMService, validate_resources()
│   │       ├── auth_service.py  # create_user(), verify_password(), create_access_token()
│   │       ├── audit_service.py # log_action()
│   │       └── monitor_service.py # get_host_metrics() via psutil
│   ├── alembic/
│   │   └── versions/
│   │       ├── 0001_initial_schema.py      # users, resource_quotas, virtual_machines, audit_logs
│   │       └── 0002_add_metrics_history.py # host_metrics_history
│   ├── tests/
│   │   ├── conftest.py          # In-memory SQLite DB, VBox mocks, host metrics mocks
│   │   ├── test_auth.py         # 5 tests — register, login, duplicate, bad password, /me
│   │   ├── test_vms.py          # 11 tests — CRUD, lifecycle, quota enforcement
│   │   ├── test_admin.py        # 11 tests — RBAC, user mgmt, quota update, audit logs
│   │   └── test_monitoring.py   # 11 tests — host metrics, history, ISO list endpoints
│   ├── .env                     # Local environment config (not committed)
│   ├── Dockerfile               # Container image (DEMO_MODE only — no VirtualBox)
│   ├── .dockerignore
│   ├── alembic.ini
│   └── requirements.txt
├── frontend/
│   └── private-cloud-ui/        # Angular 17 workspace
│       └── src/app/
│           ├── app.module.ts
│           ├── app-routing.module.ts
│           ├── core/
│           │   ├── guards/
│           │   │   └── auth.guard.ts        # Redirects unauthenticated users
│           │   ├── interceptors/
│           │   │   └── auth.interceptor.ts  # Attaches Bearer token to every request
│           │   ├── models/
│           │   │   └── vm.model.ts          # TypeScript interfaces
│           │   └── services/
│           │       ├── auth.service.ts      # Login, register, JWT decode, isAdmin
│           │       ├── vm.service.ts        # VM CRUD + action HTTP calls
│           │       ├── admin.service.ts     # Users, quotas, audit log HTTP calls
│           │       ├── monitoring.service.ts # Live + history polling
│           │       └── websocket.service.ts  # WS connection + reconnection logic
│           ├── features/
│           │   ├── auth/
│           │   │   ├── login/               # Login form component
│           │   │   └── register/            # Registration form component
│           │   ├── dashboard/               # Live metrics cards + historical charts
│           │   ├── vms/
│           │   │   ├── vm-list/             # Paginated VM table with status badges
│           │   │   ├── vm-create/           # Create VM form with quota preview
│           │   │   ├── vm-detail/           # VM info, action buttons, SSH button
│           │   │   └── vm-terminal/         # xterm.js WebSocket SSH terminal
│           │   └── admin/
│           │       ├── user-table/          # Sortable/filterable user management table
│           │       ├── quota-editor/        # Slider dialog for resource limits
│           │       └── audit-log/           # Paginated audit log with action chips
│           └── layout/                      # Shell — Material sidenav + toolbar
├── docker-compose.yml           # Compose file — runs backend in DEMO_MODE
├── scripts/
│   └── seed_data.py             # Optional: seed demo users and VMs
├── architecture-diagram.html    # Visual architecture diagram (open in browser)
├── abbreviations.html           # Glossary of abbreviations used in the project
├── VIVA_GUIDE.md                # 10-minute demo script + expected Q&A
└── README.md                    # This file
```

---

## Prerequisites

| Requirement | Minimum Version | Notes |
|---|---|---|
| Python | 3.11+ | Required for FastAPI lifespan context manager |
| Node.js | 18+ | Required by Angular CLI |
| npm | 9+ | Comes with Node.js 18 |
| Oracle VirtualBox | 7.x | Only needed if `DEMO_MODE=false` |
| ISO image | Any | Place Linux ISOs under `~/ISOs/` (Alpine recommended for low RAM usage) |

> If you do not have VirtualBox installed, set `DEMO_MODE=true` in `.env` and skip the VirtualBox installation step.

---

## Setup & Installation

### 1. Clone / extract the project

```bash
cd project
```

### 2. Backend

```bash
cd backend

# Create and activate virtual environment
python -m venv venv
venv\Scripts\activate          # Windows
source venv/bin/activate       # Linux / macOS

# Install dependencies
pip install -r requirements.txt
```

Create a `.env` file in the `backend/` directory (see [Environment Variables](#environment-variables) below).

Apply database migrations:

```bash
alembic upgrade head
```

### 3. Frontend

```bash
cd frontend/private-cloud-ui

npm install --legacy-peer-deps
```

> The `--legacy-peer-deps` flag is required due to Angular Material peer dependency constraints.

---

## Environment Variables

Create `backend/.env` with the following keys:

```env
# --- Security ---
SECRET_KEY=change-this-to-a-random-256-bit-string

# --- Database ---
DATABASE_URL=sqlite:///./cloud.db

# --- VirtualBox ---
# Full path to VBoxManage executable
VBOXMANAGE_PATH=C:\Program Files\Oracle\VirtualBox\VBoxManage.exe
# Directory where VM disk images are stored (default: ~/VirtualBox VMs)
VM_STORAGE_PATH=C:\Users\<username>\VirtualBox VMs
# Directory scanned for ISO files (default: ~/ISOs)
BASE_ISO_PATH=C:\Users\<username>\ISOs

# --- Resource Limits ---
# Default quota applied to new users
MAX_VCPUS_PER_USER=4
MAX_RAM_MB_PER_USER=8192
MAX_DISK_GB_PER_USER=100
# MB of host RAM to keep reserved (protects the host OS)
RESERVED_HOST_RAM_MB=2048
# Host CPU % threshold beyond which VM creation is refused
MAX_HOST_CPU_PERCENT=85.0

# --- Demo Mode ---
# Set to true to simulate VirtualBox without it installed
DEMO_MODE=false

# --- Misc ---
DEBUG=false
ACCESS_TOKEN_EXPIRE_MINUTES=480
# SSH terminal idle timeout in seconds (default 1800 = 30 min)
TERMINAL_IDLE_TIMEOUT=1800
```

---

## Default Credentials

The server auto-seeds an admin account on first start if no `admin` user exists:

| Field | Value |
|---|---|
| Username | `admin` |
| Password | `Admin@123456` |
| Role | ADMIN |

To promote any other registered user to admin via the CLI:

```bash
cd backend
venv\Scripts\python -c "
from app.database import SessionLocal
from app import models
db = SessionLocal()
user = db.query(models.User).filter(models.User.username == 'your-username').first()
user.role = models.UserRole.ADMIN
db.commit()
db.close()
print('Done.')
"
```

---

## Running the Application

### Backend

```bash
cd backend
venv\Scripts\activate          # Windows
source venv/bin/activate       # Linux / macOS

python -m uvicorn app.main:app --port 8000 --reload
```

> Remove `--reload` in production. The `--reload` flag restarts on code changes.

### Frontend

```bash
cd frontend/private-cloud-ui
ng serve --port 4201
```

Both processes must be running simultaneously.

---

## Access URLs

| URL | Description |
|---|---|
| `http://localhost:4201` | Angular web UI |
| `http://localhost:8000/api/docs` | Swagger / OpenAPI interactive documentation |
| `http://localhost:8000/api/redoc` | ReDoc alternative API documentation |
| `http://localhost:8000/api/health` | Health check — returns `{"status": "healthy", "version": "2.0.0"}` |

---

## API Reference

All endpoints (except `/api/v1/auth/login` and `/api/v1/auth/register`) require:

```
Authorization: Bearer <JWT token>
```

### Auth

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/api/v1/auth/register` | None | Register a new user |
| POST | `/api/v1/auth/login` | None | Login; returns a signed JWT |
| POST | `/api/v1/auth/logout` | User | Revoke the current token server-side |
| GET | `/api/v1/auth/me` | User | Current user profile |
| GET | `/api/v1/auth/quota` | User | Current user quota usage |

### Virtual Machines

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/api/v1/vms/` | User | List caller's VMs (admin sees all); supports `?skip=&limit=` pagination |
| POST | `/api/v1/vms/` | User | Create a VM — returns **202 Accepted**; provisioning runs in background |
| GET | `/api/v1/vms/{id}` | User | Get a single VM's details |
| POST | `/api/v1/vms/{id}/action` | User | Send action: `start`, `stop`, `pause`, `resume`, `reset` |
| DELETE | `/api/v1/vms/{id}` | User | Delete a VM and release quota |
| WS | `/api/v1/vms/ws/status?token=<jwt>` | User | Real-time VM status stream (push every 3 s) |
| WS | `/api/v1/vms/{id}/ws/terminal?token=<jwt>` | User | Browser SSH terminal proxy (auto-closes after 30 min idle) |

### VM Templates

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/api/v1/vms/templates/` | User | List saved VM templates |
| POST | `/api/v1/vms/templates/` | User | Create a template (manually or from `source_vm_id`) |
| DELETE | `/api/v1/vms/templates/{id}` | User | Delete a template |
| POST | `/api/v1/vms/templates/{id}/provision?name=<vm-name>` | User | Create a VM from a template |

### Monitoring

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/api/v1/monitoring/host` | User | Live CPU, RAM, disk metrics |
| GET | `/api/v1/monitoring/history?hours=1` | User | Historical metrics (1–24 h window) |
| GET | `/api/v1/monitoring/isos` | User | List available ISO files from `BASE_ISO_PATH` |

### Admin (ADMIN role required)

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/api/v1/admin/users` | Admin | List all users |
| PATCH | `/api/v1/admin/users/{id}/quota` | Admin | Update a user's resource quota |
| PATCH | `/api/v1/admin/users/{id}/role` | Admin | Change a user's role |
| PATCH | `/api/v1/admin/users/{id}/status` | Admin | Activate or deactivate a user |
| GET | `/api/v1/admin/audit-logs` | Admin | Paginated audit log with filters |

---

## Background Jobs

Two jobs run on an `AsyncIOScheduler` (started in the FastAPI `lifespan`):

### `sync_vm_states` — every 30 seconds
Reconciles the database against VirtualBox reality (eventual consistency):
1. Queries all VMs that have a `vbox_uuid` and are not in a transient state (`CREATING`, `DELETING`)
2. Runs `VBoxManage showvminfo <uuid>` for each
3. If the actual state differs from the DB state, logs a warning and corrects the DB
4. For running VMs without an IP, attempts a guest IP lookup and persists it
5. Skipped entirely when `DEMO_MODE=true`

### `record_metrics_snapshot` — every 30 seconds
Maintains the historical metrics time series:
1. Calls `get_host_metrics()` (psutil) to get current CPU/RAM/disk percentages
2. Counts running VMs from the DB
3. Inserts a new `HostMetricsHistory` row
4. Deletes all rows with `recorded_at < now - 24h` (sliding window cleanup)

---

## Security Model

### Authentication
- Passwords hashed with **bcrypt** (passlib); plain-text passwords are never stored
- On login, a signed **HS256 JWT** is issued containing `sub` (user ID), `role`, and `exp` (8-hour expiry)
- The frontend stores the token in memory (not `localStorage`) to mitigate XSS token theft
- Every request attaches the token as `Authorization: Bearer <token>` via an Angular `HttpInterceptor`
- WebSocket endpoints accept the token as a `?token=` query parameter (browser WebSocket API limitation)

### Authorization
- `get_current_user` FastAPI dependency decodes and verifies the JWT on every request
- `require_admin` dependency additionally rejects non-admin users with HTTP 403
- Deactivated users (`is_active=False`) are rejected by `get_current_user` with HTTP 403 even with a valid token
- `AdminGuard` on Angular routes mirrors the server-side check to prevent unauthorized navigation

### VM Ownership
- All VM queries in user-facing endpoints filter by `owner_id = current_user.id`
- Admins bypass this filter to see and manage all VMs

### Quota Enforcement
`validate_resources()` in `vm_service.py` performs these checks before creating a VM:
1. `used_vcpus + req_vcpus <= max_vcpus`
2. `used_ram_mb + req_ram_mb <= max_ram_mb`
3. `used_disk_gb + req_disk_gb <= max_disk_gb`
4. `used_vms + 1 <= max_vms`
5. `host_free_ram_mb - req_ram_mb >= RESERVED_HOST_RAM_MB` (protects host OS)
6. `host_cpu_percent < MAX_HOST_CPU_PERCENT`

On success, `used_*` counters are incremented in the same DB transaction as the VM record — preventing double-spend race conditions.

---

## Running Tests

```bash
cd backend
venv\Scripts\activate          # Windows

python -m pytest tests/ -v
```

Expected output:

```
tests/test_auth.py        5 passed
tests/test_vms.py        11 passed
tests/test_admin.py      11 passed
tests/test_monitoring.py 11 passed   ← expanded from 1 to 11

========================= 38 passed in ~37s =========================
```

### Angular unit tests

```bash
cd frontend/private-cloud-ui
ng test --no-watch
```

Spec files cover `AuthService` (9 tests) and `VmService` (11 tests) using `HttpClientTestingModule`. No browser or backend required.

### How tests work without VirtualBox

- `conftest.py` creates a fresh in-memory SQLite database before each test module
- All calls to `VBoxManage` are patched with `unittest.mock.patch("app.services.vm_service.vbox")`
- The mock is configured to return fake UUIDs, port numbers, and `None` for void calls
- `get_host_metrics` is patched to return an object with abundant free resources so host-capacity checks always pass

---

## Database Migrations

```bash
# Apply all pending migrations
alembic upgrade head

# View migration history
alembic history --verbose

# Roll back one revision
alembic downgrade -1

# Roll back to a specific revision
alembic downgrade 0001
```

### Migration history

| Revision | ID | Description |
|---|---|---|
| 1 | `0001_initial_schema` | `users`, `resource_quotas`, `virtual_machines`, `audit_logs` |
| 2 | `0002_add_metrics_history` | `host_metrics_history` (time-series monitoring data) |
| 3 | `0003_add_token_blocklist` | `token_blocklist` (server-side JWT revocation) |
| 4 | `0004_network_mode_and_templates` | `network_mode` column on `virtual_machines`; `vm_templates` table |

---

## Demo Mode

Demo mode lets you run the full application without VirtualBox installed. It is useful for demonstrations on any machine.

Enable it in `.env`:
```env
DEMO_MODE=true
```

When `DEMO_MODE=true`:
- A `DemoVBoxManager` replaces the real `VBoxManager`
- VM create/start/stop/pause/resume/reset/delete all return realistic responses (fake UUIDs, random ports)
- The `sync_vm_states` background job is skipped (no real hypervisor to query)
- Quota enforcement, audit logging, RBAC, and all other features remain fully functional

---

## Docker (Demo Mode)

VirtualBox is a Type-2 hypervisor that requires bare metal. The container image therefore always runs in **Demo Mode** (no real VMs are created — all operations are simulated).

### Quick start

```bash
# From the project root
docker compose up --build
```

The backend will be available at `http://localhost:8000`.

### Manual build

```bash
cd backend
docker build -t privatecloud-backend .
docker run -p 8000:8000 \
  -e SECRET_KEY=change-this-secret \
  -e DEMO_MODE=true \
  privatecloud-backend
```

### Environment variables via docker-compose

Edit `docker-compose.yml` to set `SECRET_KEY` (required) and other settings. Persistent data is stored in `./data/private_cloud.db` on the host.

---

## IaaS Concepts Demonstrated

| IaaS Concept | Implementation in this Project |
|---|---|
| **Self-service provisioning** | Users create and manage VMs via REST API / UI — no admin involvement required |
| **Resource pooling** | Host CPU, RAM, and disk are shared across all tenants; quota system enforces fair allocation |
| **Metered service** | `used_vcpus`, `used_ram_mb`, `used_disk_gb`, `used_vms` tracked per user in `resource_quotas` |
| **Multi-tenancy** | VM ownership enforced at the DB query level; users are isolated by JWT identity |
| **Elasticity** | Admin can raise or lower any user's quota at runtime via the Admin Panel |
| **Eventual consistency** | `sync_vm_states()` background job reconciles DB state with VirtualBox every 30 s |
| **Audit compliance** | Append-only `audit_logs` table records every mutating operation (analogous to AWS CloudTrail) |
| **Type-2 hypervisor** | VirtualBox runs on top of the host OS; managed programmatically via `VBoxManage` CLI |
| **Real-time monitoring** | Live host metrics (psutil) + historical time-series (SQLite WAL) with dashboard charts |
| **NAT networking** | Each VM gets a unique SSH port forwarded from the host via VirtualBox NAT |

---

## Troubleshooting

### Backend won't start
- Ensure the virtual environment is activated before running uvicorn
- Check that `alembic upgrade head` has been run at least once
- Verify `.env` exists in the `backend/` directory

### `VBoxManage not found` error
- Set the full path in `.env`: `VBOXMANAGE_PATH=C:\Program Files\Oracle\VirtualBox\VBoxManage.exe`
- Or set `DEMO_MODE=true` to bypass VirtualBox entirely

### Frontend can't connect to backend
- Confirm the backend is running on port 8000
- The Angular dev server proxies requests to `http://localhost:8000` — check `CORS` settings in `main.py` if using a different port
- Allowed origins: `localhost:4200` and `localhost:4201`

### Tests fail with `ModuleNotFoundError`
- Ensure the venv is activated and `pip install -r requirements.txt` was run inside it
- Run pytest from the `backend/` directory: `python -m pytest tests/ -v`

### SQLite locked errors
- SQLite WAL mode is enabled to support concurrent reads during WebSocket connections
- If you see lock errors, ensure no other process has the DB open exclusively
- Do not open `cloud.db` in DB Browser for SQLite while the server is running

### ISO list is empty in Create VM form
- Place at least one `.iso` file in `BASE_ISO_PATH` (default: `~/ISOs/`)
- Alpine Linux ISO is recommended as it boots in under 512 MB RAM

---

## Academic Reference

**Project Title:** End-to-End Private Cloud Implementation using Open-Source Tools  
**Course:** Cloud Computing — BITS Pilani (Semester 4)  
**Deployment Platform:** Single-node laptop  
**Hypervisor Type:** Type-2 (Oracle VirtualBox)  
**API Version:** 2.0.0  
