# PMPS CRM V2 — Arquitectura técnica

Documento de referencia rápida: qué pieza hace qué y cómo se conectan. Para historial de decisiones/bugs, ver `ROADMAP.md`.

## 1. Vista general

```
┌─────────────┐      ┌──────────────────┐      ┌─────────────────────────┐
│   GitHub    │─────▶│      Vercel      │─────▶│   Usuario (navegador)   │
│ (index.html)│ push │ (hosting + build)│ https│ pmps-os.vercel.app      │
└─────────────┘      └──────────────────┘      └───────────┬─────────────┘
                                                             │
                        ┌────────────────────────────────────┼───────────────────────┐
                        ▼                                    ▼                       ▼
              ┌───────────────────┐              ┌────────────────────┐   ┌──────────────────┐
              │ Supabase (PAKO)   │              │ Google Apps Script │   │  Resend (email)  │
              │ diqbmyqvuyollv... │              │  Web App (SCRIPT_  │   │  vía Edge Func   │
              │ Postgres+Auth+    │              │  URL) → Google     │   └──────────────────┘
              │ Storage+Functions │              │  Sheets            │
              └───────────────────┘              └────────────────────┘
```

Todo el frontend vive en **un solo archivo**: `index.html`. No hay build step, no hay framework — HTML/CSS/JS plano. GitHub aloja el código; cada push a `main` hace que Vercel redespliegue automáticamente.

## 2. Dónde vive cada dato (importante — hay DOS fuentes distintas)

| Dato | Vive en | Por qué |
|---|---|---|
| **Leads y Clientes** (Kanban, pipeline, Workflow) | **Google Sheets**, vía Google Apps Script Web App (`SCRIPT_URL`) | Es el sistema original (V1) de Menlun; nunca se migró a Supabase. La app lee/escribe ahí con `apiGet()`/`apiPost()`. |
| Forecast, Presupuesto, Ventas Históricas, Catálogo, Equipos, Reportes, Usuarios, Notificaciones, Workflow (reglas), Config, Activity log | **Supabase Postgres**, tablas con prefijo `pmps_` | Se construyeron después, ya directo en Supabase. |
| Directorio (NEXUS — prospectos DENUE) | Supabase, tabla `pmps_leads_menlun` (nombre parecido pero es OTRA cosa, no confundir con los leads del Kanban) | Módulo distinto, prospección fría. |

**Ojo**: el proyecto de Supabase (`diqbmyqvuyollvlvjniz`, alias "PAKO") es **compartido** — en la misma base de datos conviven, con sus propios prefijos, otros sistemas de Pako (SM Soluciones, ProKicks Arena, Menlun ERP interno). Ninguno se toca entre sí porque cada quien usa su propio prefijo de tabla (`pmps_*`, `sm_*`, `prokicks_*`, `menlun_*`), pero viven en el mismo servidor físico — si algo le pasa a ese proyecto de Supabase, afecta a los 4 sistemas a la vez, no solo al CRM.

## 3. Backend (Supabase)

- **Postgres**: todas las tablas `pmps_*` (14 tablas). RLS activo en todas — lectura abierta (`select` público), escritura únicamente vía la función `secure_write()`.
- **`secure_write(nombre, pin, tabla, accion, match, data)`**: única puerta de entrada para insert/update/delete. Valida el PIN del usuario del lado del servidor (nunca confía en el navegador) contra `pmps_usuarios`. Solo puede tocar las tablas listadas en su whitelist interna.
- **Auth**: Supabase Auth real (email + contraseña) para los usuarios que tienen correo asignado en `pmps_usuarios.email`. Los que no, usan clave local (PIN) validada por `usuario_verificar_clave_local()`.
- **Storage**: bucket `equipos-fotos` (fotos del módulo Equipos).
- **Edge Functions** (Deno, corren en el servidor de Supabase, nunca exponen sus claves al navegador):
  - `send-email` — envía correo vía Resend cuando algo se dispara desde la app abierta (con José Carlos siempre en copia).
  - `run-automatizaciones` — el motor automático. Corre solo, sin que nadie tenga la app abierta.
  - `admin-usuarios` — crea/edita cuentas reales de Auth (usa la Service Role Key, solo accesible desde el servidor).
- **pg_cron + pg_net**: agenda `run-automatizaciones` cada 20 minutos, llamándola por HTTP con un secreto (`x-cron-secret`) para que nadie más pueda dispararla.

## 4. Automatizaciones / correo (flujo completo)

```
pg_cron (cada 20 min)
   │
   ▼
run-automatizaciones (Edge Function)
   │  1. Lee leads/clientes en vivo desde Google Sheets (misma fuente que la app)
   │  2. Lee reglas + umbrales desde pmps_config y pmps_workflow_reglas
   │  3. Evalúa las 3 reglas fijas + reglas custom de Workflow
   │  4. Por cada match nuevo (no duplicado): crear_notificacion() (RPC)
   ▼
pmps_notificaciones (tabla)          →  campanita en la app (tiempo real al recargar)
   │
   ▼ (si el usuario tiene correo real)
send-email / Resend  →  correo al usuario + CC a José Carlos
```

## 5. Monitoreo

- **Sentry** — captura errores de JavaScript en producción.
- **UptimeRobot** — verifica que `pmps-os.vercel.app` esté arriba.
- No hay monitoreo de la salud de Supabase en sí (cron fallido, cuota de API, etc.) — dependemos del dashboard de Supabase para eso.

## 6. Respaldo de datos (backup)

**Estado actual: sin backup automático administrado.** Supabase plan gratuito no incluye respaldos diarios (eso es plan Pro). El único precedente de recuperación fue manual, desde los Excel originales, tras la migración de emergencia del 13 de julio de 2026 (ver ROADMAP.md).

Opciones para resolverlo, de más a menos robusta:
1. **Supabase Pro** (~$25 USD/mes) — respaldos diarios gestionados, sin que dependa de nada nuestro.
2. **Tarea programada diaria** (via Cowork/Claude) que exporta las tablas `pmps_*` a JSON y las guarda localmente — más barato, pero depende de que la tarea se ejecute correctamente cada noche sin supervisión activa.
3. Combinar ambas (recomendado si el presupuesto lo permite): Pro como respaldo "de fondo" garantizado, más la exportación local como copia adicional fácil de revisar.

## 7. Despliegue

1. Se edita `index.html` (u otro archivo).
2. Se sube a GitHub (`pakostudio/pmps-os`, rama `main`) vía la interfaz web de GitHub.
3. Vercel detecta el push y redespliega automáticamente — sin pasos manuales adicionales.
4. El usuario ve los cambios reflejados; si no, el botón "↻ Recargar app" fuerza una recarga sin caché.

## 8. Resumen de piezas externas usadas

| Pieza | Para qué |
|---|---|
| GitHub | Control de versiones del código |
| Vercel | Hosting del frontend |
| Supabase | Base de datos, autenticación, storage, funciones servidor, cron |
| Google Sheets + Apps Script | Fuente de datos de Leads/Clientes (Kanban) |
| Resend | Envío de correos transaccionales/automáticos |
| Sentry | Monitoreo de errores en el navegador |
| UptimeRobot | Monitoreo de disponibilidad del sitio |

---
*Última actualización: 2026-07-25.*
