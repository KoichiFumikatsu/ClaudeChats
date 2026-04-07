# Coding - Contexto: AZCKeeper

## Qué es AZCKeeper

AZCKeeper (también llamado Keeper AZC) es un **sistema de monitoreo de actividad y productividad de empleados** construido para AZC Legal. Consiste en:

1. **Cliente de escritorio stealth** (C# .NET 8 WinForms) — corre invisible en PCs de empleados
2. **Backend API** (PHP 8 + MySQL) — recibe datos, maneja políticas y autenticación
3. **Panel de administración web** (PHP + Tailwind CSS + Alpine.js) — en `keep.azclegal.com`
4. **Updater** (C# .NET 8) — ejecutable auto-contenido para actualizaciones del cliente

**Versión actual:** 3.0.1.8
**Creador:** Andres Felipe Zafra P. (Kelsie)

---

## Tech Stack

| Componente | Tecnología |
|-----------|-----------|
| Cliente desktop | C# .NET 8, WinForms, win-x64, SQLite (offline queue), DPAPI |
| Backend | PHP 8.x, PDO (raw queries), Apache (XAMPP) |
| Base de datos | MySQL 8.0 (remoto: `mysql.server1872.mylogin.co`) |
| Admin panel | PHP + Tailwind CSS (CDN) + Alpine.js (CDN) + Inter font |
| Auth | Sesiones httpOnly, SHA-256 token hashing, bcrypt passwords |
| Logging | Archivos locales + Discord webhooks |
| Updater | C# .NET 8, self-contained, single-file, trimmed |
| Infraestructura | Xeon E3-1260L V5, 32GB RAM, 2x SSD RAID, Hepsia panel |
| URL producción | `https://keep.azclegal.com/public/index.php/api/` |

---

## Estructura del Proyecto

```
AZCKeeper/
├── AZCKeeper.sln                    # Solución Visual Studio (2 proyectos)
├── AZCKeeper_Client/                # Cliente desktop C# .NET 8
│   ├── Auth/                        # Login, DPAPI token/credential storage
│   ├── Blocking/                    # Bloqueo remoto de dispositivos (pantalla completa)
│   ├── Config/                      # ConfigManager (client_config.json)
│   ├── Core/                        # CoreService orchestrator, TimeSync, MasterTimer
│   ├── Logging/                     # LocalLogger (archivo + Discord webhook)
│   ├── Network/                     # ApiClient (HTTP), OfflineQueue (SQLite)
│   ├── Startup/                     # Auto-inicio vía Registry
│   ├── Tracking/                    # ActivityTracker, WindowTracker, WorkSchedule
│   ├── Update/                      # Auto-update manager
│   └── Web/                         # Backend PHP + Admin panel
│       ├── src/                     # PHP source (namespace Keeper\)
│       │   ├── Endpoints/           # Handshake, Login, ActivityDay, etc.
│       │   ├── Repos/               # Repositorios PDO
│       │   └── Services/            # ProductivityCalculator, DualJobDetector
│       ├── public/                  # Web root
│       │   ├── index.php            # API router
│       │   └── admin/               # Panel admin (15+ páginas)
│       └── migrations/              # 17 archivos SQL de migración
├── AZCKeeperUpdater/                # Updater self-contained
├── FallbackTest/                    # Proyecto test para DB fallback
├── build/                           # Output de compilación
├── build-release.bat                # Script de build (produce ZIP)
├── install.bat                      # Instalador del cliente
├── reparar-cliente.bat              # Script de reparación
└── firma-email/                     # Firma de email corporativa
```

---

## Flujo de la Aplicación

### 1. Instalación Inicial
1. `install.bat` despliega en `%LOCALAPPDATA%\AZCKeeper\app\`
2. Primera ejecución: genera DeviceId (GUID) → guarda en `client_config.json`
3. Muestra LoginForm (empleado ingresa cédula CC + contraseña)
4. `POST /api/client/login` autentica contra `keeper_users.password_hash` (bcrypt)
5. Servidor registra dispositivo en `keeper_devices`, retorna Bearer token
6. Cliente almacena token encriptado con DPAPI → `auth_token.bin`
7. También guarda credenciales (DPAPI) → `auth_credentials.bin` para re-login silencioso

### 2. Operación Normal (ciclo continuo)

```
┌─────────────────────────────────────────────────────────┐
│  HANDSHAKE (cada 5 min + jitter)                        │
│  POST /api/client/handshake                             │
│  Envía: deviceId, version, device_name                  │
│  Recibe: config efectiva, horario, hora UTC, displayName│
│  → Aplica políticas: módulos on/off, bloqueo, updates   │
├─────────────────────────────────────────────────────────┤
│  ACTIVITY TRACKING (cada 1s)                            │
│  GetLastInputInfo Win32 API → mide idle time            │
│  Categoriza cada segundo: active/idle                   │
│  Sub-categoriza: WorkHours/LunchTime/AfterHours         │
├─────────────────────────────────────────────────────────┤
│  ACTIVITY FLUSH (cada 10-30s)                           │
│  POST /api/client/activity-day                          │
│  Upsert resumen diario en keeper_activity_day           │
├─────────────────────────────────────────────────────────┤
│  WINDOW TRACKING (event-driven + timer fallback)        │
│  SetWinEventHook(EVENT_SYSTEM_FOREGROUND)               │
│  Registra episodios: proceso, título, duración          │
│  Detecta apps de llamadas (Zoom, Teams, etc.)           │
│  POST /api/client/window-episode                        │
├─────────────────────────────────────────────────────────┤
│  OFFLINE QUEUE (retry cada 30s)                         │
│  Requests fallidos → SQLite offline_queue.db            │
│  Reintentos automáticos; dead letters después de max    │
└─────────────────────────────────────────────────────────┘
```

### 3. Re-autenticación Silenciosa
Si token se pierde/expira:
1. Intenta re-enroll por `device_guid`
2. Si falla, usa credenciales DPAPI guardadas
3. Si ambos fallan, muestra LoginForm

### 4. Sistema de Políticas
- Almacenadas en `keeper_policy_assignments` (blobs JSON)
- 3 scopes: global, per-user, per-device (mergeadas por prioridad)
- Controlan: módulos habilitados, intervalos de timer, logging, bloqueo, updates, keywords de llamadas

---

## Features/Módulos

### A. Monitoreo de Actividad
- Tracking active/idle por segundo (Win32 `GetLastInputInfo`)
- Categorización temporal: horas trabajo, almuerzo, horas extra
- Resúmenes diarios en `keeper_activity_day`
- Day resume: al reiniciar, recupera datos existentes del servidor

### B. Tracking de Ventanas/Aplicaciones
- Cada cambio de ventana activa → episodio (proceso, título, duración)
- **Detección de llamadas**: Zoom, Teams, Skype, 3CX, Zoiper, WebEx
- **Detección de ocio**: listas configurables de apps (proceso exacto) y ventanas (keyword en título)
- Tiempo en llamadas cuenta como activo aunque no haya input de teclado/mouse

### C. Bloqueo Remoto de Dispositivos
- Admin activa bloqueo remotamente vía política
- Crea formularios fullscreen en TODOS los monitores (multi-monitor aware)
- Keyboard hook bloquea: Win key, Alt+Tab, Ctrl+Shift+Esc
- Desbloqueo vía PIN (validado localmente, notificado al servidor)

### D. Focus Score (Productividad 0-100)
- **5 componentes**: Context switches (20%), Deep work (25%), Distraction (20%), Punctuality (15%), Constancy (20%)
- **Productividad %**: `(work_active - distraction) / (work_active + work_idle) * 100`
- Cálculo server-side (cron nocturno o botón manual)
- Almacenado en `keeper_focus_daily`

### E. Detección de Doble Empleo
- Alertas automáticas por patrones sospechosos:
  - Actividad after-hours (>5 días con >1h después de horario)
  - Apps foráneas (TeamViewer, AnyDesk, VirtualBox, VMware)
  - Remote desktop en horas laborales
  - Patrones sospechosos de idle + actividad after-hours
- Alertas en `keeper_dual_job_alerts` con niveles de severidad
- Workflow de revisión con notas

### F. Auto-Update
- Cliente consulta `/api/client/version` periódicamente
- Compara current vs latest vs minimum version
- Descarga ZIP, extrae, lanza `AZCKeeperUpdater.exe` que reemplaza archivos
- Force-update para violaciones de versión mínima/crítica

### G. Logging
- Multi-destino: archivo local (`%APPDATA%\AZCKeeper\Logs\`) + Discord webhooks
- Niveles configurables: None, Error, Warn, Info
- Sanitización de tokens/credenciales en output
- Servidor puede sobrescribir niveles vía handshake

### H. Sincronización de Tiempo
- `TimeSync` calcula offset entre reloj del cliente y servidor
- Todos los timestamps usan hora ajustada al servidor

---

## Base de Datos (24+ tablas keeper_*)

### Core
| Tabla | Propósito |
|---|---|
| `keeper_users` | Empleados (id, cc, display_name, email, password_hash, status) |
| `keeper_devices` | Dispositivos (device_guid, device_name, client_version, last_seen_at) |
| `keeper_sessions` | Sesiones auth (token_hash, expires_at, revoked_at) |

### Tracking
| Tabla | Propósito |
|---|---|
| `keeper_activity_day` | Resúmenes diarios por usuario/dispositivo |
| `keeper_window_episode` | Episodios de ventanas (proceso, título, duración, is_in_call) |
| `keeper_events` | Store genérico de eventos (schema flexible + JSON payload) |
| `keeper_daily_metrics` | Métricas diarias key-value por usuario/dispositivo |

### Organización
| Tabla | Propósito |
|---|---|
| `keeper_sociedades` | Organizaciones/empresas top-level |
| `keeper_firmas` | Firmas/clientes (ej. firmas de abogados US) |
| `keeper_areas` | Áreas organizacionales (jerárquicas parent/child) |
| `keeper_cargos` | Cargos con niveles de jerarquía |
| `keeper_sedes` | Sedes/sucursales físicas |
| `keeper_user_assignments` | Mapeo users → sociedad/firma/area/cargo/sede |

### Admin Panel
| Tabla | Propósito |
|---|---|
| `keeper_admin_accounts` | Admin users (vinculados a keeper_users, con rol + scope) |
| `keeper_admin_sessions` | Sesiones del panel admin |
| `keeper_panel_roles` | Roles dinámicos (superadmin, admin, manager, viewer) con permisos JSON |
| `keeper_panel_settings` | Settings key-value (menu_visibility, productivity thresholds) |

### Políticas & Control
| Tabla | Propósito |
|---|---|
| `keeper_policy_assignments` | Políticas JSON (scope global/user/device con prioridad) |
| `keeper_device_locks` | Registros de lock/unlock remoto |
| `keeper_work_schedules` | Horarios por usuario (inicio/fin, almuerzo, días) |
| `keeper_client_releases` | Versiones del cliente con URLs, notas, force-update flags |

### Productividad
| Tabla | Propósito |
|---|---|
| `keeper_focus_daily` | Focus scores diarios (0-100) con desglose |
| `keeper_dual_job_alerts` | Alertas de doble empleo con evidencia JSON |
| `keeper_suspicious_apps` | Lista configurable de apps sospechosas |

### Multi-Tenant
| Tabla | Propósito |
|---|---|
| `keeper_data_sources` | Conexiones DB por firma (credenciales AES-256-CBC) |

---

## API Endpoints

### GET
| Ruta | Handler | Propósito |
|---|---|---|
| `/api/health` | `Health::handle` | Health check del servidor |
| `/api/client/activity-day` | `ActivityDay::handleGet` | Resumir contadores del día |
| `/api/client/version` | `ClientVersion::handle` | Info de última versión |

### POST
| Ruta | Handler | Propósito |
|---|---|---|
| `/api/client/login` | `ClientLogin::handle` | Autenticación (CC + password) |
| `/api/client/handshake` | `ClientHandshake::handle` | Sync config, políticas, horario |
| `/api/client/activity-day` | `ActivityDay::handle` | Upsert resumen diario |
| `/api/client/window-episode` | `WindowEpisode::handle` | Registrar episodio de ventana |
| `/api/client/event` | `EventIngest::handle` | Ingestión genérica de eventos |
| `/api/client/force-handshake` | `ForceHandshake::handle` | Forzar refresh de config |
| `/api/client/re-enroll` | `ClientReEnroll::handle` | Re-autenticación por device_guid |
| `/api/cron/productivity` | `ProductivityCron::handle` | Cálculo nocturno de productividad |

---

## Panel de Administración (15+ páginas)

| Página | Propósito |
|---|---|
| `login.php` | Login admin (email + password, cookie httpOnly 8h) |
| `index.php` | Dashboard principal (KPIs, Focus Score gauge, donut productividad, top users) |
| `users.php` | Lista usuarios con paginación, búsqueda, CRUD, status |
| `user-dashboard.php` | Dashboard individual (actividad, ventanas, focus score breakdown) |
| `devices.php` | Gestión dispositivos (revocar, activar, eliminar; chart versiones) |
| `organization.php` | CRUD para Firmas, Áreas, Cargos, Sedes (tabs, modals) |
| `policies.php` | Config políticas de bloqueo (horario, apps/windows ocio) |
| `assignments.php` | Mapeo usuario → organización |
| `releases.php` | Gestión versiones del cliente |
| `admin-users.php` | Gestión cuentas admin |
| `roles.php` | Roles y permisos granulares |
| `panel-settings.php` | Visibilidad de módulos por rol |
| `productivity.php` | Dashboard Focus Score (gauge, ranking, tendencias) |
| `dual-job-alerts.php` | Gestión alertas doble empleo (review, notas, filtros) |
| `sedes-dashboard.php` | KPIs agrupados por sede física |
| `server-health.php` | Monitoreo salud del servidor |

### Sistema de Permisos
- **Roles jerárquicos**: superadmin > admin > manager > viewer
- **Permisos granulares**: por módulo (can_view, can_create, can_edit, can_delete, can_export)
- **Filtrado por scope**: admins ven solo datos de su firma/area/sede asignada
- **Visibilidad modules**: configurable por rol vía `keeper_panel_settings.menu_visibility`

---

## Módulos C# del Cliente

| Módulo | Archivo | Propósito |
|---|---|---|
| Program | `Program.cs` | Entry point; mutex single-instance; exception handlers; ProcessExit flush |
| CoreService | `Core\CoreService.cs` | Orchestrador central; init, start, stop; handshake; políticas; timers |
| TimeSync | `Core\TimeSync.cs` | Sync de reloj con servidor (offset) |
| MasterTimer | `Core\MasterTimer.cs` | Timer base 1s con eventos multiplexados |
| ConfigManager | `Config\ConfigManager.cs` | Carga/guarda `client_config.json` desde `%APPDATA%\AZCKeeper\Config\` |
| AuthManager | `Auth\AuthManager.cs` | Token management (DPAPI encrypted); credential storage para re-login |
| LoginForm | `Auth\LoginForm.cs` | UI de login WinForms |
| ApiClient | `Network\ApiClient.cs` | Comunicación HTTP (HTTPS con fallback HTTP); todos los API calls |
| OfflineQueue | `Network\OfflineQueue.cs` | Cola SQLite persistente para requests fallidos; retry automático |
| ActivityTracker | `Tracking\ActivityTracker.cs` | Medición active/idle vía `GetLastInputInfo`; boundaries de día |
| WindowTracker | `Tracking\WindowsTracker.cs` | Tracking ventana activa vía `SetWinEventHook`; episodios; llamadas |
| WorkSchedule | `Tracking\WorkSchedule.cs` | Clasificación temporal (work/lunch/after-hours) |
| KeyBlocker | `Blocking\KeyBlocker.cs` | Bloqueo remoto; fullscreen; keyboard hook |
| UpdateManager | `Update\UpdateManager.cs` | Auto-update (check, download ZIP, launch updater) |
| StartupManager | `Startup\StartupManager.cs` | Auto-inicio vía `HKCU\...\Run` registry key |
| LocalLogger | `Logging\LocalLogger.cs` | Logging multi-destino; sanitización de secretos |

---

## Decisiones Arquitectónicas Clave

1. **Operación stealth**: Corre sin UI visible (ApplicationContext sin form principal), sin ícono en taskbar
2. **Prevención thundering herd**: Jitter en handshake y flush timers (delay random 0-interval antes del primer tick)
3. **Resiliencia offline**: Cola SQLite persistente para API calls fallidos con retry automático
4. **Fallback HTTPS→HTTP**: Si HTTPS falla, intenta HTTP automáticamente
5. **Dual auth headers**: Token en `Authorization: Bearer` y `X-Auth-Token` (compatibilidad Apache)
6. **Productividad server-side**: Todos los cálculos de Focus Score en servidor (cron/manual), sin cambios en cliente
7. **Multi-tenant ready**: `keeper_data_sources` con credenciales encriptadas por firma, `Db::sourceFor($firmaId)`
8. **Coexistencia legacy**: Dual PDO (keeper DB + legacy `pipezafra_soporte_db`); `LegacySyncService` conecta tabla antigua

---

## Historial de Cambios Relevantes

1. **Leisure split**: Separación en Apps (match exacto proceso) vs Windows (match parcial título)
2. **Fix timezone**: Eliminado offset -5h incorrecto en window episodes
3. **Detección primer login**: Cambiado a usar primer window episode después de 5 AM
4. **devices.php**: Nueva página gestión dispositivos con KPIs, tabla, búsqueda, chart versiones
5. **organization.php**: Nuevo CRUD para Firmas, Áreas, Cargos, Sedes
6. **Sidebar + Roles**: Módulo organización agregado a navegación y permisos
7. **Fix búsqueda devices**: Alpine.js `applyFilters()` reemplazando `get filtered()` roto
8. **Permisos users.php**: `can_create` para botón creación
9. **Sistema Productividad/Focus Score**: Implementación completa (BD, backend, frontend, permisos)
10. **Botón Calculate manual**: Reemplazo temporal del cron en productividad
11. **Separación multi-database**: Dual PDO (keeper DB + legacy DB) con data sources encriptados
12. **Fix Discord webhook**: Contexto de usuario en mensajes, removido filtro Info
13. **DisplayName en handshake**: Refresh nombre desde servidor en cada handshake
14. **Auto-re-login**: Recuperación silenciosa de credenciales vía DPAPI

---

## Build y Despliegue

### Build
- `build-release.bat`: Compila Updater (self-contained single-file) + Client (self-contained ReadyToRun)
- Output: `build\AZCKeeper_v3.0.1.8.zip`

### Flujo de Deploy
1. Upload ZIP a GitHub Releases
2. Registrar versión en `/admin/releases.php`
3. Habilitar `autoDownload=true` en política del servidor
4. Clientes auto-detectan vía `/api/client/version`
5. Cliente descarga ZIP → extrae → lanza `AZCKeeperUpdater.exe`

### Instalación
- `install.bat`: Instala en `%LOCALAPPDATA%\AZCKeeper\app\`
- Mata procesos existentes, limpia, copia nuevos archivos, lanza cliente
- Soporta instalación desde DWService (detecta perfil de usuario logueado)

### Reparación
- `reparar-cliente.bat`: Script de reparación del cliente

---

## Roadmap Pendiente

- **Screenshots**: No existe módulo, endpoint, ni tabla DB — por construir desde cero
- **Alertas outbound**: EventIngest recibe pero no despacha notificaciones
- **Reglas de alerta configurables**: No hay tabla ni UI
- **Setup Wizard**: Planeado (PLAN_SETUP_WIZARD.md) pero no implementado — onboarding para nuevas empresas (5 pasos: superadmin, org, users, assignments, policies)

---

## Paleta Corporativa

- Navy: `#003A5D`
- Red: `#BE1622`
- Estilo inspirado en LawyerDesk
