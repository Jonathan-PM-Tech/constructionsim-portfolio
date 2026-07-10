# ConstructionSIM — From Browser Prototype to Production Pilot

*A case study: taking an AI-powered training simulator from an insecure, browser-only prototype to a secure, multi-tenant production deployment in one focused build session.*

> **Note:** Client and company names in this write-up have been fictionalized for confidentiality. The technical architecture, decisions, and outcomes described are accurate to the real engagement.

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

### Key Technical Decisions

- **Raw SQL over an ORM**: with only 5 tables and business-critical rate-limit checks that must be transactional, hand-written SQL queries were simpler to reason about under time pressure than ORM session/model overhead.
- **JWKS-based JWT verification over trusting client-supplied tokens naively**: the backend verifies every request's identity locally and cheaply, with a cached key set, rather than calling out to the auth provider on every request.
- **Server-side recomputation of derived game data** (cost/schedule performance metrics) rather than trusting client-submitted values, closing off an obvious tampering vector into the AI prompts.
- **Config-at-container-start, not config-at-build-time**: the static frontend regenerates its runtime config from environment variables on every container boot, so the same Docker image can be promoted from a test deploy to production without a rebuild.

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

### Decisiones Técnicas Clave

- **SQL directo en vez de un ORM**: con solo 5 tablas y validaciones de límite de uso críticas para el negocio que deben ser transaccionales, las consultas SQL escritas a mano fueron más simples de razonar bajo presión de tiempo que la sobrecarga de sesiones/modelos de un ORM.
- **Verificación de JWT vía JWKS en vez de confiar ingenuamente en tokens enviados por el cliente**: el backend verifica la identidad de cada request localmente y de forma económica, con un conjunto de llaves cacheado, en vez de llamar al proveedor de auth en cada request.
- **Recálculo del lado del servidor de datos derivados del juego** (métricas de desempeño de costo/cronograma) en vez de confiar en valores enviados por el cliente, cerrando un vector obvio de manipulación hacia los prompts de IA.
- **Configuración al arrancar el contenedor, no al construir la imagen**: el frontend estático regenera su configuración de tiempo de ejecución desde variables de entorno en cada arranque del contenedor, para que la misma imagen Docker pueda promoverse de un despliegue de prueba a producción sin reconstruirla.

### Stack Tecnológico

`Python` · `FastAPI` · `asyncpg` · `Supabase (Postgres + Auth)` · `API de Anthropic (Claude)` · `React (vanilla, sin build step)` · `Docker` · `Railway` · `GitHub`

### Resultado

Llevé el producto de "demo que solo corre en una laptop" a un piloto multi-cliente en vivo — desplegado, autenticado de punta a punta con un flujo de login real, y verificado funcionando con un escenario real generado por IA — en una sola sesión de ingeniería enfocada.
