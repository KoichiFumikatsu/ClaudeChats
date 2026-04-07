# Kelsie_App (Household OS) - Coding Contexto

## Qué es Kelsie_App

Kelsie_App es un **"Household OS"** — una plataforma modular de gestión del hogar diseñada para exactamente 2 usuarios (pareja) compartiendo un solo hogar. El nombre del paquete npm es `household-os`.

**Características clave:**
- Diseñado para una pareja en Colombia
- **UX neurodivergente**: Un usuario con ADHD (input rápido, feedback visual, gamificación) y otro con rasgos del espectro autista (consistencia, colores semánticos, previsibilidad, datos estructurados)
- Localizado para Colombia: UI en español, COP (pesos colombianos), timezone `America/Bogota`, locale `es-CO`
- Estética inspirada en gacha games: Arknights, Honkai Star Rail, Wuthering Waves (paneles glass, glow effects)

---

## Tech Stack

| Componente | Tecnología |
|-----------|-----------|
| Framework | Next.js 16.2.1 (App Router) |
| Frontend | React 19.2.4 + React Compiler + TypeScript 5+ strict |
| Backend | Supabase (PostgreSQL + RLS + Realtime + Auth) |
| Styling | Tailwind CSS 4 + CSS variables custom (sin component library) |
| Icons | lucide-react |
| Fonts | DM Sans (primary), JetBrains Mono (números/code) |
| Deploy | Vercel |
| Notificaciones | Discord webhooks |
| Data pattern | Server Actions (`'use server'`) + `ActionResult<T>` discriminated union |

**No hay REST API layer** — solo Server Actions y 2 cron API routes.

---

## Estructura del Proyecto

```
Kelsie_App/
├── app/
│   ├── (app)/                    # Shell autenticado (layout con nav)
│   │   ├── layout.tsx            # SideNav + Header + BottomNav
│   │   ├── dashboard/page.tsx    # Dashboard home
│   │   ├── finance/page.tsx      # → FinanceClient
│   │   ├── chores/page.tsx       # → ChoresClient
│   │   ├── tasks/page.tsx        # → TasksClient
│   │   ├── medical/page.tsx      # → MedicalClient
│   │   ├── studies/page.tsx      # → StudiesClient
│   │   └── settings/
│   │       ├── page.tsx          # → SettingsClient
│   │       └── permissions/      # → PermissionsClient
│   ├── (auth)/                   # Páginas sin auth
│   │   ├── login/
│   │   ├── register/
│   │   └── join/[code]/
│   ├── api/cron/                 # Cron jobs (Edge runtime)
│   │   ├── daily-digest/
│   │   └── med-reminders/
│   └── auth/callback/            # OAuth callback
├── actions/                      # Server Actions (16 archivos, ~40+ funciones)
│   ├── core/                     # auth, dashboard, households, permissions, profile
│   ├── finance/                  # quincenas, categorias, transacciones, dashboard
│   ├── chores/                   # templates, instances, rewards
│   ├── tasks/                    # tasks
│   ├── medical/                  # records, medicamentos, reminders
│   └── studies/                  # goals, sessions
├── components/
│   ├── layout/                   # Header, BottomNav, SideNav
│   ├── modules/                  # FinanceClient, ChoresClient, TasksClient, MedicalClient, StudiesClient, Settings
│   └── ui/                       # Button, Modal, BottomSheet, Badge, Progress, Card, Input, Charts, QuickAdd, UserAvatar
├── hooks/                        # useHousehold, usePermissions, useRealtime
├── lib/
│   ├── supabase/                 # client.ts, server.ts, proxy.ts
│   ├── types/                    # modules.types.ts, database.types.ts (placeholder)
│   └── utils/                    # format.ts (COP + fechas), discord.ts
├── supabase/migrations/          # 19 archivos SQL (001-019)
├── proxy.ts                      # Next.js middleware
├── HOUSEHOLD_OS_SPEC.md          # Spec master (386 líneas)
└── .env.local                    # Supabase credentials
```

---

## Flujo de la Aplicación

### Autenticación
1. Middleware (`proxy.ts`) redirige no-autenticados a `/login`
2. **User A (owner)**: `/register` → `auth.signUp` → RPC `setup_owner` (crea household, perfil, 5 permisos full access) → `/dashboard`
3. **User A** comparte invite code (visible en dashboard vía `InviteLinkCard`)
4. **User B (member)**: `/join/[code]` → `get_household_by_invite_code` → `auth.signUp` → RPC `setup_member` (link a household, permisos view+edit) → `/dashboard`

### App Shell
- **Desktop**: SideNav fijo izquierda (w-56) + Header sticky + contenido
- **Mobile**: Header sticky + contenido + BottomNav fijo abajo
- Todas las páginas de módulo son server components delgados que renderizan un `*Client` component

### Data Flow
- Carga datos vía Server Actions desde `useEffect`/`useCallback` en client components
- Datos compartidos (finance, chores) tienen **Supabase Realtime** subscriptions que triggean re-fetches
- Datos privados (work tasks) filtrados por `user_id` en server actions
- Mutaciones vía Server Actions → retornan `ActionResult<T>`

### Notificaciones
- **Discord webhooks**: `lib/utils/discord.ts` envía rich embeds al webhook configurado del household
- **Cron daily-digest** (noon UTC / 6AM Colombia): agregado de chores, studies, tasks, finance → un mensaje Discord por household
- **Cron med-reminders**: notificaciones de medicamentos, auto-desactiva expirados

---

## Los 5 Módulos

### 1. Finance (shared, accent `#1A7A5A` verde)
- Presupuesto basado en **quincenas** (1-15 y 16-fin de mes)
- Auto-creación de quincenas con rollover de saldo
- 5 tipos de transacción: gasto, ingreso, ahorro, bolsillo, credito
- Categorías con presupuesto por quincena, ícono, orden, `assigned_to`, `quincena_half` (1 o 2)
- Dashboard KPI: saldo inicial, totales por tipo, saldo actual, presupuesto vs real por categoría
- Formato COP en toda la app
- Realtime subscription en `transacciones`

### 2. Chores (shared, accent `#7C5CBF` púrpura)
- Sistema de plantillas con 5 frecuencias: diaria, semanal, quincenal, mensual, única
- Auto-generación de instancias diarias desde plantillas
- Acciones complete/skip con sistema de puntos
- **Gamificación**: scoreboard entre miembros, registro de recompensas
- Selector "¿Quién lo hizo?" para múltiples miembros
- 4 vistas: Hoy, Pendientes, Historial, Calendario
- Realtime subscription en `chore_instances`

### 3. Work Tasks (privado por usuario, accent `#2563A8` azul)
- **Kanban** con 4 columnas: backlog, in_progress, done, cancelled
- Prioridades: low/mid/high/urgent (con colores)
- Subtareas (add/toggle/remove), tags, recurring (daily/weekly/monthly)
- Vista calendario con grid mensual
- Quick-add para creación rápida
- Detección de overdue con indicadores visuales

### 4. Medical (shared, accent `#C0472A` rojo)
- 2 secciones: Timeline (registros) + Medicamentos
- Tipos de registro: consulta, examen, vacuna, control
- Medicamentos con scheduling: hora_inicio, frecuencia_horas, duracion_dias, proxima_toma, fecha_fin auto-calculada
- Medicamentos vinculables a registros médicos vía `record_id`
- Panel próximos recordatorios (2 semanas)
- Auto-desactivación de medicamentos expirados vía cron

### 5. Studies (shared, accent `#B87428` ámbar)
- Metas de estudio con categorías: curso, libro, certificacion, idioma, habilidad
- Progreso: total_unidades vs unidades_completadas con `CircularProgress` rings
- Horario: hora + días de clase (array de weekdays)
- Sesiones de estudio (minutos + unidades + notas)
- Auto-completado al alcanzar target; auto-inicio al registrar primera sesión
- **Study streak** (días consecutivos con sesiones)

---

## Base de Datos (19 migraciones SQL)

### Core
| Tabla | Propósito |
|---|---|
| `households` | id, nombre, invite_code, discord_webhook_url, owner_id |
| `profiles` | id (FK auth.users), household_id, display_name, color_hex, role (owner/member) |
| `module_permissions` | user_id, household_id, module_name, can_view/edit/delete/manage |
| `notification_log` | household_id, modulo, titulo, cuerpo, enviado_at |

### Finance
| Tabla | Propósito |
|---|---|
| `quincenas` | household_id, nombre, fecha_inicio/fin, saldo_inicial, is_active |
| `categorias` | household_id, nombre, tipo, presupuesto_default, icono, orden, assigned_to, quincena_half |
| `presupuestos_quincena` | quincena_id, categoria_id, monto_previsto |
| `transacciones` | quincena_id, categoria_id, user_id, household_id, tipo, fecha, importe, descripcion |

### Chores
| Tabla | Propósito |
|---|---|
| `chore_templates` | household_id, nombre, frecuencia, puntos, assigned_to, is_active |
| `chore_instances` | template_id, household_id, assigned_to, due_date, status (pending/done/skipped), puntos_earned |
| `reward_logs` | household_id, user_id, puntos, razon |

### Work Tasks
| Tabla | Propósito |
|---|---|
| `work_tasks` | household_id, user_id, titulo, prioridad, status, due_date/time, subtasks (jsonb), tags (text[]), recurring |

### Medical
| Tabla | Propósito |
|---|---|
| `medical_records` | household_id, user_id, tipo, especialidad, fecha, doctor, clinica, notas |
| `medicamentos` | household_id, user_id, nombre, dosis, hora_inicio, frecuencia_horas, duracion_dias, proxima_toma, record_id |

### Studies
| Tabla | Propósito |
|---|---|
| `study_goals` | household_id, user_id, titulo, categoria, total_unidades, unidades_completadas, horario, dias_clase |
| `study_sessions` | goal_id, user_id, fecha, minutos, unidades_avanzadas, nota |

### RLS (Row Level Security)
- Todas las tablas tienen RLS habilitado
- Patrón general: usuarios solo acceden datos de su household (vía `profiles.household_id`)
- Work tasks adicionalmente filtrados por `user_id` (privado)

### Funciones DB (SECURITY DEFINER)
- `setup_owner()` — Crea household, actualiza perfil, crea 5 permisos full access
- `setup_member()` — Vincula perfil a household, crea 5 permisos view+edit
- `get_household_by_invite_code()` — Lookup household por invite code
- `auto_saldo_inicial()` — Trigger auto-set saldo_inicial desde balance del período anterior

---

## Design System

### Colores
- Canvas: `#E8E6E1` (warm gray)
- Surface: `#FFFFFF`, Surface-2: `#F0EEEA`, Glass: `rgba(255,255,255,0.45)`
- Texto: `#141413` (primary), `#575650` (secondary), `#9C9A94` (muted)
- Semánticos: income `#1A7A5A`, expense `#C0472A`, warn `#B87428`, info `#2563A8`, credit `#9333EA`
- Module accents: finance=verde, chores=púrpura, tasks=azul, medical=rojo, studies=ámbar

### Componentes UI (hand-built, sin library)
- `Button` — 4 variantes: primary, secondary, danger, ghost
- `Modal` — Dialog centrado con backdrop frosted + escape to close
- `BottomSheet` — Slide-up desde abajo, drag-to-close con detección de velocidad
- `Badge` / `StatusBadge` — Tags coloreados, mapeo status→color
- `ProgressBar` / `CircularProgress` — Barras y rings SVG
- `Card` / `ModuleCard` — Superficies con accent top-border
- `QuickAdd` — Input rápido de una línea

### Animaciones
- slide-up, fade-in, pulse-expand, slide-out-right, float-up

---

## Estado Actual

**Completamente implementado y funcional:**
- Los 5 módulos con CRUD completo, UI, y server actions
- Auth (register, login, join via invite code)
- Household + member management
- Permisos por módulo
- Dashboard cross-module
- Discord webhooks + 2 cron jobs
- Realtime subscriptions para módulos compartidos
- 19 migraciones DB con RLS completo

**Incompleto / Placeholder:**
- `database.types.ts` es placeholder (tipos manuales en `modules.types.ts`, no auto-generados de Supabase)
- `README.md` sigue siendo el default de create-next-app
- `presupuestos_quincena` (overrides por quincena) tiene tabla pero KPI usa `presupuesto_default`

---

## Historial de Evolución (por migraciones)

1. **001-003**: Schema core, fix RLS recursion, funciones de registro
2. **004-007**: Schema finance (quincenas, categorias, presupuestos, transacciones)
3. **008-010**: Schema chores (templates, instances, rewards)
4. **011**: Schema work tasks
5. **012-013**: Schema medical (records, medicamentos)
6. **014**: Schema studies (goals, sessions)
7. **015-019**: Mejoras incrementales — subtasks/time para tasks, medical compartido, scheduling medicamentos, auto-saldo, assigned_to/quincena_half para categorías, tipo credito
