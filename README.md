# ConstructionSIM — From Browser Prototype to Production Pilot

*A case study: taking an AI-powered training simulator from an insecure, browser-only prototype to a secure, multi-tenant production deployment in one focused build session.*

> **Note:** Client and company names in this write-up have been fictionalized for confidentiality. The technical architecture, decisions, and outcomes described are accurate to the real engagement.

**🎥 Demo:** [Try the interactive demo](https://jonathan-pm-tech.github.io/constructionsim-portfolio/demo/) — a scripted sample playthrough (no live AI, no backend, no login) built from the real product's UI. *Español:* [Probar la demo interactiva](https://jonathan-pm-tech.github.io/constructionsim-portfolio/demo/) — un recorrido de muestra guionado (sin IA en vivo, sin backend, sin login) construido con la UI real del producto.

🇺🇸 [English](#english) · 🇪🇸 [Español](#español)

---

## English

### Overview

**ConstructionSIM** is an AI-powered decision-training simulator for construction professionals (Owners, GCs, Architects, Engineers, Trade Partners). Players load a real or example project, face AI-generated realistic field events (RFIs, change orders, safety incidents), and get every decision graded against industry best practice and their own company standards.

I was brought in by **NorthPeak Construction Technologies** (name changed) to take an existing working prototype — built as a single self-contained HTML file that called the Anthropic API directly from the browser — and turn it into something safe to actually sell.

### The Problem

The prototype worked, but the architecture blocked selling it to a single customer:

- **IP exposure** — the four prompt templates that make up the product's core value were shipped in plain text in the browser bundle, readable by anyone via "View Source."
- **No usage metering** — every customer would need their own Anthropic API key pasted into the app; the business had no way to see or bill for usage.
- **No access control** — anyone with the file could play; there was no way to restrict access to paying customers or track who belonged to which organization.

### What I Built

Acting as backend architect and sole engineer, I designed and shipped a complete production backend and got it live:

- **FastAPI proxy backend** — 4 endpoints (`/api/scenario`, `/api/evaluate`, `/api/recovery`, `/api/final`) that own the Anthropic API key and the prompt templates server-side. The client sends only structured game state; it never sees prompt text again.
- **Supabase-backed auth & data layer** — magic-link authentication with local JWT verification via JWKS (no per-request round-trip to the auth provider), and a 5-table Postgres schema (organizations, users, sessions, decisions, reports) with row-level security as defense-in-depth.
- **Multi-tenant organization model** — each customer organization gets its own invite link and seat limit, enforced transactionally at the database level — not just in the UI.
- **Cost & abuse guardrails** — a global kill switch, and per-session, per-hour, and per-month call ceilings, all checked before every AI call.
- **Anthropic prompt caching** — restructured the prompt payloads into a cached system block (persona, project context, company standards — identical across a session) and a dynamic per-call user message, to cut input token cost on repeated calls within a session.
- **Production deployment** — two containerized services (API + static frontend) deployed independently, with runtime environment-variable injection for per-deploy configuration (API base URL, auth provider keys) without rebuilding images.
- **Frontend integration** — replaced the direct-to-Anthropic browser calls with authenticated calls to the new backend, and added a full login/invite-acceptance flow to the existing single-file frontend without a framework migration.

### Architecture

```
Browser (React, vanilla — single static file, no build step)
        │  Supabase magic-link session (JWT)
        ▼
FastAPI backend — 4 endpoints (/api/scenario, /api/evaluate, /api/recovery, /api/final)
        │
        ├── Local JWT verification (Supabase JWKS, cached 10 min — no per-request auth round-trip)
        ├── Rate-limit guard — kill switch → session cap → hourly cap → org monthly cap
        │       (cheapest, in-memory check first; DB checks only run if that passes)
        ├── asyncpg pool → Postgres (orgs, users, sessions, decisions, reports, ai_call_log)
        └── Anthropic client — cached system-prompt block + dynamic per-turn user message

Deployment: two independent containerized services (API + static frontend) on Railway,
each regenerating its runtime config from environment variables on container boot.
```

### Key Technical Decisions

- **Raw SQL over an ORM**: with only 5 tables and business-critical rate-limit checks that must be transactional, hand-written SQL queries were simpler to reason about under time pressure than ORM session/model overhead.
- **JWKS-based JWT verification over trusting client-supplied tokens naively**: the backend verifies every request's identity locally and cheaply, with a cached key set, rather than calling out to the auth provider on every request.
- **Server-side recomputation of derived game data** (cost/schedule performance metrics) rather than trusting client-submitted values, closing off an obvious tampering vector into the AI prompts.
- **Config-at-container-start, not config-at-build-time**: the static frontend regenerates its runtime config from environment variables on every container boot, so the same Docker image can be promoted from a test deploy to production without a rebuild.
- **Rate-limit checks called explicitly, not wired as a framework dependency**: the guard function is invoked directly at the top of each AI route instead of relying on the framework's dependency-injection system, because the session ID it needs lives inside the request body, not somewhere the framework can see before the handler runs.

### Real Problems Solved During Deployment

- **Silent connection failures under Supabase's pooled Postgres connection**: the pooler runs in transaction mode, which is incompatible with a database driver's default behavior of caching prepared statements across connections — reused connections would intermittently fail. Fixed by disabling the client-side statement cache for pooled connections, before it ever reached real traffic.
- **Static frontend didn't load at the bare domain root**: the lightweight file server used to host the static frontend only auto-serves a file literally named `index.html`, but the app shipped as a single file named after the product. Fixed by generating an `index.html` copy at container boot, alongside the existing runtime-config injection.
- **Auth-provider key rotation could have locked out every logged-in user at once**: instead of treating an unrecognized signing key as a hard failure, the JWT verifier forces one cache refresh and retries before rejecting a token — so a routine signing-key rotation on the auth provider's side degrades gracefully instead of causing a mass logout.

### Tech Stack

`Python` · `FastAPI` · `asyncpg` · `Supabase (Postgres + Auth)` · `Anthropic API (Claude)` · `React (vanilla, no build step)` · `Docker` · `Railway` · `GitHub`

### Outcome

Took the product from "demo that only runs on one laptop" to a live, multi-tenant pilot — deployed, authenticated end-to-end with a real login flow, and verified working with a real AI-generated scenario — in a single focused engineering session.

---

## Español

### Resumen

**ConstructionSIM** es un simulador de entrenamiento de decisiones con IA para profesionales de la construcción (Dueños, Contratistas Generales, Arquitectos, Ingenieros, Subcontratistas). El jugador carga un proyecto real o de ejemplo, enfrenta eventos de campo generados por IA (RFIs, órdenes de cambio, incidentes de seguridad), y cada decisión se califica contra las mejores prácticas de la industria y los estándares propios de su empresa.

Me contrató **NorthPeak Construction Technologies** (nombre cambiado) para tomar un prototipo funcional existente — un solo archivo HTML autocontenido que llamaba directo a la API de Anthropic desde el navegador — y convertirlo en algo seguro para vender de verdad.

### El Problema

El prototipo funcionaba, pero su arquitectura bloqueaba venderlo a un solo cliente:

- **Exposición de propiedad intelectual** — las cuatro plantillas de prompts que constituyen el valor central del producto se enviaban en texto plano dentro del paquete del navegador, legibles por cualquiera con "Ver código fuente."
- **Sin medición de uso** — cada cliente necesitaría pegar su propia API key de Anthropic en la app; el negocio no tenía forma de ver ni cobrar por el uso real.
- **Sin control de acceso** — cualquiera con el archivo podía jugar; no había forma de restringir el acceso a clientes que pagaran, ni de rastrear a qué organización pertenecía cada usuario.

### Lo Que Construí

Como arquitecto de backend e ingeniero único del proyecto, diseñé y entregué un backend de producción completo, y lo dejé funcionando en vivo:

- **Backend proxy en FastAPI** — 4 endpoints (`/api/scenario`, `/api/evaluate`, `/api/recovery`, `/api/final`) que resguardan la API key de Anthropic y las plantillas de prompts del lado del servidor. El cliente solo envía el estado estructurado del juego; nunca vuelve a ver texto de prompts.
- **Capa de autenticación y datos con Supabase** — autenticación por magic-link con verificación local de JWT vía JWKS (sin ida y vuelta al proveedor de auth en cada request), y un esquema Postgres de 5 tablas (organizaciones, usuarios, sesiones, decisiones, reportes) con seguridad a nivel de fila como defensa adicional.
- **Modelo de organizaciones multi-cliente** — cada organización cliente tiene su propio link de invitación y límite de asientos, impuesto de forma transaccional a nivel de base de datos — no solo en la interfaz.
- **Controles de costo y abuso** — un interruptor de apagado global, y topes de llamadas por sesión, por hora y por mes, verificados antes de cada llamada a la IA.
- **Prompt caching de Anthropic** — reestructuré los prompts en un bloque de sistema cacheable (persona, contexto del proyecto, estándares de la empresa — idéntico durante toda una sesión) y un mensaje de usuario dinámico por llamada, para reducir el costo de tokens de entrada en llamadas repetidas dentro de una misma sesión.
- **Despliegue de producción** — dos servicios containerizados (API + frontend estático) desplegados de forma independiente, con inyección de variables de entorno en tiempo de ejecución para configuración por despliegue (URL base de la API, llaves del proveedor de auth) sin reconstruir las imágenes.
- **Integración de frontend** — reemplacé las llamadas directas del navegador a Anthropic por llamadas autenticadas al nuevo backend, y agregué un flujo completo de login/aceptación de invitación al frontend existente de un solo archivo, sin migrar a ningún framework.

### Arquitectura

```
Navegador (React, vanilla — un solo archivo estático, sin build step)
        │  Sesión magic-link de Supabase (JWT)
        ▼
Backend FastAPI — 4 endpoints (/api/scenario, /api/evaluate, /api/recovery, /api/final)
        │
        ├── Verificación local de JWT (JWKS de Supabase, cacheado 10 min — sin ida y vuelta por request)
        ├── Guardia de rate-limit — kill switch → tope de sesión → tope por hora → tope mensual por org
        │       (primero la validación en memoria, más barata; las de base de datos solo si esa pasa)
        ├── Pool de asyncpg → Postgres (orgs, usuarios, sesiones, decisiones, reportes, ai_call_log)
        └── Cliente de Anthropic — bloque de sistema cacheado + mensaje de usuario dinámico por turno

Despliegue: dos servicios containerizados independientes (API + frontend estático) en Railway,
cada uno regenerando su configuración de tiempo de ejecución desde variables de entorno al arrancar.
```

### Decisiones Técnicas Clave

- **SQL directo en vez de un ORM**: con solo 5 tablas y validaciones de límite de uso críticas para el negocio que deben ser transaccionales, las consultas SQL escritas a mano fueron más simples de razonar bajo presión de tiempo que la sobrecarga de sesiones/modelos de un ORM.
- **Verificación de JWT vía JWKS en vez de confiar ingenuamente en tokens enviados por el cliente**: el backend verifica la identidad de cada request localmente y de forma económica, con un conjunto de llaves cacheado, en vez de llamar al proveedor de auth en cada request.
- **Recálculo del lado del servidor de datos derivados del juego** (métricas de desempeño de costo/cronograma) en vez de confiar en valores enviados por el cliente, cerrando un vector obvio de manipulación hacia los prompts de IA.
- **Configuración al arrancar el contenedor, no al construir la imagen**: el frontend estático regenera su configuración de tiempo de ejecución desde variables de entorno en cada arranque del contenedor, para que la misma imagen Docker pueda promoverse de un despliegue de prueba a producción sin reconstruirla.
- **Las validaciones de rate-limit se llaman de forma explícita, no como dependencia del framework**: la función de guardia se invoca directamente al inicio de cada ruta de IA en vez de depender del sistema de inyección de dependencias del framework, porque el ID de sesión que necesita vive dentro del cuerpo del request, no en un lugar que el framework pueda ver antes de que corra el handler.

### Problemas Reales Resueltos Durante el Despliegue

- **Fallas silenciosas de conexión con el Postgres pooled de Supabase**: el pooler corre en modo transacción, lo cual es incompatible con el comportamiento por defecto de un driver de base de datos que cachea prepared statements entre conexiones — las conexiones reutilizadas fallaban de forma intermitente. Se resolvió desactivando el cache de statements del lado del cliente para conexiones pooled, antes de que llegara a tráfico real.
- **El frontend estático no cargaba en la raíz del dominio**: el servidor de archivos liviano usado para alojar el frontend estático solo sirve automáticamente un archivo llamado literalmente `index.html`, pero la app se entregaba como un solo archivo con el nombre del producto. Se resolvió generando una copia como `index.html` al arrancar el contenedor, junto a la inyección de configuración ya existente.
- **La rotación de llaves del proveedor de auth podía dejar afuera a todos los usuarios logueados a la vez**: en vez de tratar una llave de firma no reconocida como una falla dura, el verificador de JWT fuerza un refresh del cache y reintenta antes de rechazar un token — así una rotación de rutina en el proveedor de auth se degrada de forma controlada en vez de causar un logout masivo.

### Stack Tecnológico

`Python` · `FastAPI` · `asyncpg` · `Supabase (Postgres + Auth)` · `API de Anthropic (Claude)` · `React (vanilla, sin build step)` · `Docker` · `Railway` · `GitHub`

### Resultado

Llevé el producto de "demo que solo corre en una laptop" a un piloto multi-cliente en vivo — desplegado, autenticado de punta a punta con un flujo de login real, y verificado funcionando con un escenario real generado por IA — en una sola sesión de ingeniería enfocada.
