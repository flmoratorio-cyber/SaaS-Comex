# SaaS Comex — Fix & Go
## Enterprise Architecture, Case Studies, Incident Post-Mortems & Resilience Engineering

> **Public Architectural Case Study & Technical Documentation / Documentación de Arquitectura y Casos de Estudio**  
> **Status / Estado:** Production Ready / Producción Estable  
> **Note / Nota:** Sensitive credentials, private keys, and internal domains have been sanitized or replaced with placeholders (`<AQUI_VA_TU_CLAVE>`, `<AQUI_VA_TU_DOMINIO>`). / Toda información sensible ha sido enmascarada para su publicación en GitHub.

---

## 🌐 LANGUAGE / IDIOMA
- [Parte 1: Español (#parte-1---español)](#parte-1---español)
- [Part 2: English (#part-2---english)](#part-2---english)

---

# Parte 1 - Español

## 1. Resumen Ejecutivo del Sistema
**SaaS Comex** es una plataforma distribuida de alta disponibilidad diseñada para la gestión operativa, logística y de tesorería en Comercio Exterior (Comex). El sistema procesa legajos aduaneros, permisos de embarque (PE), declaraciones juradas (DJVE), seguimiento de camiones y monitoreo de saldos financieros en tiempo real.

### Stack Tecnológico
* **Frontend:** React 18, Vite, TailwindCSS, Zustand (State Management), React Router v7.
* **Backend:** Node.js (ES Modules), Express.js, Multer, ExcelJS, unpdf, qpdf CLI.
* **Base de Datos & Autenticación:** PostgreSQL en Supabase, PgBouncer/Supavisor Pooler, pgvector, Row-Level Security (RLS).
* **Infraestructura & DevOps:** VPS en Oracle Cloud Infrastructure (OCI) / Hetzner, Coolify v4 (Docker Orchestrator), Cloudflare WAF, GitHub Actions / Webhooks CI/CD.

---

## 2. Casos de Estudio: Incidencias Reales, Análisis de Causa Raíz (RCA) y Soluciones

### Caso 1: Colapso de Conectividad por Incompatibilidad IPv6 vs IPv4 y Burst IOPS
* **Fecha:** 24 de Septiembre, 2026
* **El Problema:** La API de backend comenzó a arrojar timeouts masivos (`upstream request timeout`) al intentar autenticar usuarios o escribir logs de auditoría. Simultáneamente, el motor de PostgreSQL en Supabase reportó agotamiento del *Disk IO Budget*, amenazando con degradar la velocidad de transferencia al piso de 5 MB/s.
* **Causa Raíz:** 
  1. La cadena de conexión apuntaba directamente al host IPv6 de la base de datos por el puerto 5432. La red Virtual Cloud Network (VCN) del VPS en Oracle Cloud operaba exclusivamente bajo IPv4, produciendo un bloqueo silencioso de red (`Network is unreachable`).
  2. Los bucles de reintento congelaron conexiones e impulsaron el uso de memoria Swap en disco. Múltiples consultas sobre la tabla de archivos carecían de índices B-Tree en la columna `created_at`, ejecutando escaneos secuenciales masivos.
* **Solución Aplicada:**
  1. Se migró la URI de conexión hacia el **Connection Pooler (Supavisor)** en el puerto 6543 con soporte dual IPv4 (`<AQUI_VA_TU_HOST_POOLER>:6543`) y `sslmode=require`.
  2. Se activó el Index Advisor de PostgreSQL y se crearon índices B-Tree explícitos en las columnas de ordenamiento y filtrado de alto tráfico, recuperando la bolsa de créditos de IOPS al 100%.

```sql
-- Creación de índices B-Tree para optimización de I/O en PostgreSQL
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_archivos_operacion_created_at 
ON public.archivos_operacion (created_at DESC);
```

---

### Caso 2: Error HTTP 415 por "Split Brain" en Validación Middleware de Archivos Comprimidos
* **Fecha:** 18 de Marzo, 2026
* **El Problema:** Los usuarios recibían una falla HTTP 415 (Unsupported Media Type) al intentar subir paquetes legajo en formato `.zip` o `.rar`.
* **Causa Raíz:** Desincronización entre la capa de rutas (Multer middleware) y la capa de controladores. La lista blanca `MIME_ACEPTADOS` en las rutas no contenía la firma de compresión, mientras que el controlador sí la aceptaba. Multer abortaba la petición en el filtro previo.
* **Solución Aplicada:** Unificación de constantes de formato en un único diccionario centralizado `uploads.config.js` consumido tanto por los middlewares de subida como por los controladores de negocio.

---

### Caso 3: Despliegue en Producción y Desacoplamiento de Monorepo (502 Bad Gateway)
* **Fecha:** 25 de Marzo, 2026
* **El Problema:** Al desplegar el repositorio unificado mediante `concurrently`, la instancia generaba errores 502 Bad Gateway y respuestas de error `Unexpected token '<', "<html>..."`.
* **Causa Raíz:** Conflicto de puertos internos entre el servidor de desarrollo de Vite (puerto 5173) y Express (puerto 4000), sumado al comportamiento del servidor Nginx del frontend que capturaba peticiones con rutas relativas `/api` y devolvía la página 404 estática en lugar de redirigir al backend.
* **Solución Aplicada:**
  1. Desacoplamiento total del Monorepo en dos servicios independientes dentro del orquestador:
     - **Backend API:** `https://<AQUI_VA_TU_DOMINIO_API>` (Contenedor Node.js, puerto 4000).
     - **Frontend Static:** `https://<AQUI_VA_TU_DOMINIO_APP>` (Contenedor Nginx estático, puerto 80).
  2. Configuración de cliente HTTP centralizado en React (`lib/api.js`) con soporte de URL base absoluta inyectada durante la compilación.

---

### Caso 4: Desfase de Relojes de Sesión y Bucle de Expulsión de Usuarios
* **Fecha:** 6 de Abril, 2026
* **El Problema:** El usuario era redirigido a `/login` sin aviso previo justo después de interactuar con la aplicación.
* **Causa Raíz:** Falta de sincronización entre el temporizador de inactividad de la interfaz (15 min) y la expiración del JWT de Supabase (1 hora). El frontend enviaba una petición de refresco a `POST /api/auth/refresh`, pero la ruta no existía en el backend Express, provocando un error 404 que forzaba el cierre de sesión en Zustand.
* **Solución Aplicada:**
  1. Implementación de interceptores HTTP en Axios/Fetch para detectar tokens obsoletos y renovar la sesión mediante el cliente nativo de Supabase Auth.
  2. Adición de directivas `Cache-Control: no-store` en las respuestas de autenticación del servidor.

---

## 3. Patrones de Diseño y Buenas Prácticas Aplicadas
- **Single Source of Truth (SSOT):** Centralización de lógica de negocio en servicios (`services/`), evitando llamadas HTTP directas desorganizadas en componentes React.
- **Fail Fast & Graceful Degradation:** Resiliencia en llamadas a APIs de Inteligencia Artificial (Gemini) utilizando un pipeline en cascada que alterna automáticamente de modelo en caso de respuesta 503 o 429.
- **Row-Level Security (RLS) & RBAC:** Control de acceso basado en roles con aislamiento estricto por usuario y cliente en la capa de PostgreSQL.

---

---

# Part 2 - English

## 1. Executive Summary
**SaaS Comex** is a high-availability distributed platform engineered for international trade operations, logistics tracking, and treasury management. The platform manages customs documentation, shipping permits, sworn export declarations (DJVE), truck fleet tracking, and real-time financial balance monitoring.

### Technical Stack
* **Frontend:** React 18, Vite, TailwindCSS, Zustand (State Management), React Router v7.
* **Backend:** Node.js (ES Modules), Express.js, Multer, ExcelJS, unpdf, qpdf CLI.
* **Database & Auth:** PostgreSQL on Supabase, PgBouncer/Supavisor Pooler, pgvector, Row-Level Security (RLS).
* **Infrastructure & DevOps:** VPS on Oracle Cloud Infrastructure (OCI) / Hetzner, Coolify v4 (Docker Orchestrator), Cloudflare WAF, GitHub Actions / Webhooks CI/CD.

---

## 2. Case Studies: Real-World Incidents, Root Cause Analysis (RCA) & Solutions

### Case 1: Connectivity Breakdown via IPv6 Routing Issue & Burst IOPS Depletion
* **Date:** September 24, 2026
* **Issue:** The backend API started throwing persistent `upstream request timeout` errors during user authentication and audit logging. Simultaneously, Supabase reported *Disk IO Budget* depletion, threatening to throttle throughput to a 5 MB/s baseline.
* **Root Cause:** 
  1. The database connection string pointed directly to the database's IPv6 host on port 5432. The Oracle Cloud VPS Virtual Cloud Network (VCN) only supported IPv4 egress, resulting in `Network is unreachable` errors.
  2. Retry loops saturated memory, triggering Swap disk usage. Unindexed queries on audit and operational file tables forced massive sequential scans (*Seq Scans*).
* **Mitigation:**
  1. Connection URI was migrated to the **Connection Pooler (Supavisor)** on port 6543 featuring dual IPv4 support (`<YOUR_SUPABASE_POOLER_HOST>:6543`) with `sslmode=require`.
  2. PostgreSQL Index Advisor was enabled, and explicit B-Tree indices were added to high-traffic ordering columns, fully restoring the IOPS credit pool.

```sql
-- B-Tree Index creation for PostgreSQL I/O optimization
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_archivos_operacion_created_at 
ON public.archivos_operacion (created_at DESC);
```

---

### Case 2: HTTP 415 Error Caused by Validation "Split Brain" on Compressed File Uploads
* **Date:** March 18, 2026
* **Issue:** Users encountered an HTTP 415 (Unsupported Media Type) error when attempting to upload `.zip` or `.rar` document packages.
* **Root Cause:** Discrepancy between the routing middleware (Multer filter) and controller logic. The `MIME_ACEPTADOS` whitelist in the route definition lacked archive MIME signatures, while the controller supported them. Multer rejected files prior to controller execution.
* **Mitigation:** Unified validation constants into a single source configuration file (`uploads.config.js`) shared across middlewares and business controllers.

---

### Case 3: Production Deployment & Monorepo Decoupling (502 Bad Gateway)
* **Date:** March 25, 2026
* **Issue:** Deploying the monorepo via `concurrently` caused 502 Bad Gateway errors and client-side `Unexpected token '<', "<html>..."` parsing exceptions.
* **Root Cause:** Port binding conflicts between Vite dev server (port 5173) and Express (port 4000). Additionally, the frontend Nginx static server captured relative `/api` calls, serving a static 404 HTML page instead of proxying requests to the backend.
* **Mitigation:**
  1. Decoupled the monorepo into two standalone services within the container orchestrator:
     - **Backend API:** `https://<YOUR_API_DOMAIN>` (Node.js Container, port 4000).
     - **Frontend Static:** `https://<YOUR_APP_DOMAIN>` (Nginx Container, port 80).
  2. Hardened client-side HTTP helper (`lib/api.js`) with absolute build-time base URL injection.

---

### Case 4: Session Clock Skew & Unexpected User Logout Loop
* **Date:** April 6, 2026
* **Issue:** Active users were abruptly logged out and redirected to `/login` without warning.
* **Root Cause:** Mismatch between frontend inactivity timers (15 min) and Supabase JWT expiration (1 hour). When users clicked "Continue", the frontend attempted to invoke `POST /api/auth/refresh`, an unmapped route in Express, resulting in 404 errors that triggered an forced state purge in Zustand.
* **Mitigation:**
  1. Implemented HTTP response interceptors in Axios/Fetch to handle token renewal seamlessly via the native Supabase Auth client.
  2. Injected `Cache-Control: no-store` headers into authentication endpoints.

---

## 3. Core Architectural Patterns & Best Practices
- **Single Source of Truth (SSOT):** Business logic encapsulated inside dedicated services (`services/`), keeping React components UI-focused.
- **Fail Fast & Graceful Degradation:** Cascading fallback pipeline across multiple LLM models (Gemini Flash series) to handle upstream 503/429 service interruptions transparently.
- **Row-Level Security (RLS) & RBAC:** Granular role-based access control coupled with tenant isolation at the PostgreSQL engine level.
