# SaaS Comex — Fix & Go
## Case Studies, Incident Post-Mortems & Resilience Engineering

> **Public Architectural Case Study & Technical Documentation / Documentación de Arquitectura y Casos de Estudio**  
> **Status / Estado:** Production Ready / Producción Estable  
> **Note / Nota:** Sensitive credentials, private keys, and internal domains have been sanitized (`<AQUI_VA_TU_CLAVE>`, `<AQUI_VA_TU_DOMINIO>`). / Toda información sensible ha sido enmascarada para su publicación en GitHub.

---

## 🌐 LANGUAGE / IDIOMA
- [Parte 1: Español (#parte-1---español)](#parte-1---español)
- [Part 2: English (#part-2---english)](#part-2---english)

---

# Parte 1 - Español

## 1. Qué es
SaaS Comex es una plataforma web orientada a la gestión operativa, logística y de tesorería para comercio exterior.
Permite el seguimiento en tiempo real de legajos aduaneros, permisos de embarque (PE), declaraciones juradas (DJVE), flota de transporte y saldos financieros.

## 2. Arquitectura
```text
[ Cliente Web / React 18 (Vite) ]
               │
               │ HTTPS (TLS 1.3) via Cloudflare WAF
               ▼
   ┌──────────────────────┐
   │  Nginx Static Server │  app.<AQUI_VA_TU_DOMINIO> (Port 80)
   └──────────────────────┘
               │
               │ API REST / JSON
               ▼
   ┌──────────────────────┐
   │   Node.js / Express  │  api.<AQUI_VA_TU_DOMINIO> (Port 4000)
   └──────────────────────┘
         │            │
         │            └─────────────────────────┐
         │ PgBouncer / Supavisor (Port 6543)    │ Storage / Auth
         ▼                                      ▼
┌──────────────────┐                  ┌──────────────────┐
│ PostgreSQL Engine│                  │ Supabase / Drive │
│  (Supabase IPv4) │                  │ (Documentos/Auth)│
└──────────────────┘                  └──────────────────┘
```

## 3. Stack
* **Frontend:** React 18, Vite, TailwindCSS, Zustand, React Router v7.
* **Backend:** Node.js (ES Modules), Express.js, Multer, ExcelJS, unpdf, qpdf CLI.
* **Base de Datos & Auth:** PostgreSQL en Supabase, Supavisor Connection Pooler (IPv4 / Puerto 6543), pgvector, Row-Level Security (RLS).
* **Infraestructura & DevOps:** VPS (Hetzner / Oracle Cloud), Coolify v4 (Docker Orchestrator), Nginx, Cloudflare WAF, Webhooks ( Coolify ).

## 4. Incidentes / Decisiones

### Incidente 1: Colapso de Conectividad por Incompatibilidad IPv6 vs IPv4 y Burst IOPS
* **Síntoma:** Timeouts masivos (`upstream request timeout`) en la API y reporte de agotamiento de *Disk IO Budget* en Supabase.
* **Causa:** La URI de conexión apuntaba al host IPv6 directo por el puerto 5432, inalcanzable desde la VCN IPv4 del VPS (`Network is unreachable`), generando reintentos que agotaron la memoria y el Swap por falta de índices en `archivos_operacion`.
* **Solución:** Migración de la conexión al Connection Pooler (Supavisor) por IPv4 en puerto 6543 (`<AQUI_VA_TU_HOST_POOLER>:6543?sslmode=require`) y creación de índices B-Tree en `created_at`.

### Incidente 2: Error HTTP 415 en Carga de Archivos Comprimidos (.zip / .rar)
* **Síntoma:** Rechazo HTTP 415 (*Unsupported Media Type*) al intentar subir legajos comprimidos.
* **Causa:** Desincronización "Split Brain" entre la lista blanca de MIMEs del middleware Multer (que no incluía zip) y la del controlador (que sí la aceptaba).
* **Solución:** Unificación de la validación de extensiones y MIMEs en un diccionario de configuración único (`uploads.config.js`).

### Incidente 3: Error 502 Bad Gateway por Desacoplamiento de Monorepo
* **Síntoma:** Fallas 502 Bad Gateway y retornos de HTML estático en lugar de JSON al consumir la API en producción.
* **Causa:** Conflicto de puertos internos con `concurrently` y mal comportamiento del proxy Nginx al capturar rutas relativas `/api`.
* **Solución:** División del monorepo en dos servicios Docker independientes: Frontend Nginx (puerto 80) y Backend Express (puerto 4000) con clientes HTTP usando URL base absoluta.

### Incidente 4: Expulsión Repentina de Sesión (Clock Skew)
* **Síntoma:** Usuarios redirigidos a la pantalla de `/login` intempestivamente mientras operaban.
* **Causa:** Desfase entre el timer de inactividad de la UI y la expiración del JWT de Supabase, disparando llamadas a un endpoint `/api/auth/refresh` inexistente que devolvía 404.
* **Solución:** Interceptores de refresco automático vía SDK de Supabase e inyección de cabeceras `Cache-Control: no-store` en los endpoints de autenticación.

## 5. Estado real

### En Producción
* Core operativo completo: Módulo de Operaciones, Cumplidos, Tesorería (Cashbook/Vencimientos) y Catálogo NCM.
* Infraestructura desacoplada en VPS con Coolify + Cloudflare SSL/WAF.
* Conexión estabilizada a Supabase vía Supavisor Pooler IPv4 y aislamiento de datos RLS por cliente.

### En Plan (Backlog)
* Módulo RAG / ComexBot con embeddings vectoriales (`pgvector`) para consulta inteligente de normativas aduaneras.
* Migración completa a arquitectura Multi-Tenant aislada por esquemas dinámicos.
* Automatización de reportes ejecutivos semanales exportables a PDF/Excel.

---

---

# Part 2 - English

## 1. What it is
SaaS Comex is a web platformdesigned for operational, logistics, and treasury management in foreign trade.
It enables real-time tracking of customs folders, shipping permits (PE), sworn export declarations (DJVE), transport fleets, and financial balances.

## 2. Architecture
```text
[ Web Client / React 18 (Vite) ]
               │
               │ HTTPS (TLS 1.3) via Cloudflare WAF
               ▼
   ┌──────────────────────┐
   │  Nginx Static Server │  app.<YOUR_DOMAIN> (Port 80)
   └──────────────────────┘
               │
               │ REST API / JSON
               ▼
   ┌──────────────────────┐
   │   Node.js / Express  │  api.<YOUR_DOMAIN> (Port 4000)
   └──────────────────────┘
         │            │
         │            └─────────────────────────┐
         │ PgBouncer / Supavisor (Port 6543)    │ Storage / Auth
         ▼                                      ▼
┌──────────────────┐                  ┌──────────────────┐
│ PostgreSQL Engine│                  │ Supabase / Drive │
│  (Supabase IPv4) │                  │ (Docs / Auth)    │
└──────────────────┘                  └──────────────────┘
```

## 3. Stack
* **Frontend:** React 18, Vite, TailwindCSS, Zustand, React Router v7.
* **Backend:** Node.js (ES Modules), Express.js, Multer, ExcelJS, unpdf, qpdf CLI.
* **Database & Auth:** PostgreSQL on Supabase, Supavisor Connection Pooler (IPv4 / Port 6543), pgvector, Row-Level Security (RLS).
* **Infrastructure & DevOps:** VPS (Hetzner / Oracle Cloud), Coolify v4 (Docker Orchestrator), Nginx, Cloudflare WAF, Webhooks ( Coolify ).

## 4. Incidents / Decisions

### Incident 1: Connectivity Breakdown via IPv6 vs IPv4 Incompatibility & IOPS Depletion
* **Symptom:** API `upstream request timeout` errors and Supabase *Disk IO Budget* depletion warnings.
* **Cause:** Connection string targeted the direct IPv6 host on port 5432, unreachable from the VPS IPv4 VCN (`Network is unreachable`), triggering retry loops that exhausted memory and Swap due to unindexed queries on `archivos_operacion`.
* **Solution:** Migrated connection URI to Supavisor Connection Pooler over IPv4 on port 6543 (`<YOUR_SUPABASE_POOLER_HOST>:6543?sslmode=require`) and added B-Tree indices on `created_at`.

### Incident 2: HTTP 415 Error on Compressed File Uploads (.zip / .rar)
* **Symptom:** HTTP 415 (*Unsupported Media Type*) error when uploading compressed packages.
* **Cause:** "Split Brain" validation mismatch between Multer route middleware (missing zip MIME) and controller logic.
* **Solution:** Unified extension and MIME validations into a single config dictionary (`uploads.config.js`).

### Incident 3: HTTP 502 Bad Gateway via Monorepo Decoupling
* **Symptom:** HTTP 502 Bad Gateway and HTML error responses instead of JSON when invoking API routes in production.
* **Cause:** Internal port conflicts with `concurrently` and proxy routing issues when serving relative `/api` paths through Nginx.
* **Solution:** Decoupled monorepo into two standalone Docker containers: Frontend Nginx (port 80) and Backend Express (port 4000) with absolute base URLs in HTTP clients.

### Incident 4: Unexpected Session Eviction Loop
* **Symptom:** Users suddenly evicted to `/login` during active sessions.
* **Cause:** Skew between UI inactivity timers and Supabase JWT expiration, triggering 404 calls on an unmapped `/api/auth/refresh` route.
* **Solution:** Implemented automated refresh interceptors via Supabase SDK and injected `Cache-Control: no-store` headers into authentication routes.

## 5. Current State

### In Production
* Full operational core: Customs Operations, Compliance, Treasury (Cashbook/Maturities), and NCM Tariff Catalog.
* Decoupled VPS infrastructure managed with Coolify + Cloudflare SSL/WAF.
* Stabilized database connection via Supavisor Pooler IPv4 with tenant RLS isolation.

### Planned (Backlog)
* RAG / ComexBot module using vector embeddings (`pgvector`) for automated customs regulation queries.
* Migration to full Multi-Tenant architecture isolated via dynamic database schemas.
* Automated weekly executive report exports in PDF/Excel.
