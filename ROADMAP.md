# PMPS OS v2 — Roadmap "Nivel Referencia de Mercado"

Objetivo: llevar PMPS CRM V2 al nivel de Salesforce/HubSpot/Monday en los puntos donde hoy nos quedamos cortos, para poder mostrarlo con orgullo a clientes potenciales y usarlo como diferenciador comercial.

Última actualización: 2026-07-12.

## Estado de fondo ya resuelto (no confundir con pendientes)

- PIN de acceso validado del lado del servidor vía `secure_write()` (RPC en Supabase) — ya no es una validación de adorno en el navegador.
- Políticas RLS de escritura cerradas en las 7 tablas que no las necesitaban.
- Benchmark 1-5 (Kanban, Automatizaciones básicas, Velocidad/conversión, Vistas Tabla/Tarjetas, Forecast ponderado) — completos.
- Manual de usuario + FAQ — completo.

Lo que sigue abierto es autenticación **real** (sesiones por usuario), no el candado de PIN en sí.

## Plan de trabajo priorizado

| Prioridad | # | Pendiente | Por qué va en este orden | Esfuerzo | Se puede hacer en vivo? |
|---|---|---|---|---|---|
| 1 | 1 | Auditoría / historial de cambios (`activity_log`) | Rápido de entregar, cero riesgo (solo agrega, no toca nada existente), y es lo primero que pregunta un cliente que audita procesos: "¿quién cambió esto y cuándo?" | Media | Sí, sin riesgo |
| 2 | 2 | Autenticación real (Supabase Auth, login por usuario) | Es el hueco más serio de cara a un cliente técnico, pero requiere más cuidado: rehacer login, migrar 8 usuarios, reescribir políticas RLS | Grande | Mejor en ambiente de prueba primero |
| 3 | 3 | Notificaciones por correo (fase 1, sin push todavía) | Reutiliza las reglas de Automatizaciones que ya existen — solo agregamos el canal de aviso (Resend o similar) | Media | Sí, bajo riesgo |
| 4 | 4 | Integraciones externas (WhatsApp primero, luego Calendario, luego correo) | Alto impacto comercial, pero se puede ir por partes empezando por lo más barato (botón WhatsApp ya casi listo) | Grande (por fases) | Sí, incremental |
| 5 | 5 | Automatización tipo workflow visual configurable | La pieza más compleja de diseñar bien; tiene más sentido atacarla cuando ya haya tracción real con clientes y sepamos qué reglas piden de verdad | Grande | Sí, sin riesgo pero es la de más diseño |

## Por qué este orden (criterio)

Empezamos por lo que es rápido y de cero riesgo (auditoría), seguido de lo que es más urgente aunque más delicado (auth real, mejor en ambiente de prueba), y dejamos lo más costoso de diseñar (workflow visual) para cuando ya tengamos señal real de qué necesitan los clientes, en vez de adivinar de más.

## Registro de avance

- [x] Auditoría / historial de cambios — tabla `activity_log` + pestaña "Actividad" en Administrador
- [x] Autenticación real (Supabase Auth) — 6 cuentas con email+contraseña verificadas en el servidor; probado en vivo (admin, vendedor, contraseña incorrecta rechazada)
- [x] Notificaciones — centro de notificaciones en la app (campana con contador, por usuario, tabla `notificaciones` en Supabase) **+ envío real por correo vía Resend**, probado en vivo (correo recibido, status 200). Solo envía a usuarios con cuenta real de Supabase Auth (los 6 con correo); Daniel/Invitado siguen viendo el aviso solo dentro de la app.
- [x] **José Carlos copiado siempre (CC)** en todo correo de notificación, tanto el que se dispara al abrir la app como el automático — implementado en las dos funciones de envío (`send-email` y `run-automatizaciones`), verificado en el código desplegado.
- [x] **Automatización 100% independiente de la app** — nueva función `run-automatizaciones` en Supabase, corre sola dos veces al día (8:00 am y 2:00 pm hora de México) vía `pg_cron` + `pg_net`, sin que nadie tenga que abrir PMPS CRM. Revisa leads/clientes/reglas de Workflow y manda notificación + correo (con José Carlos en copia) cuando hay algo nuevo que avisar. Probada en vivo end-to-end (status 200, `ok:true`).
- [x] Integraciones externas fase 1 (sin costo/credenciales) — botón 💬 WhatsApp (abre chat directo) en Leads/Clientes/Directorio, y botón 📅 para descargar recordatorio .ics (Google/Outlook/Apple Calendar) en pagos por vencer. Probado en vivo. Correo real sigue bloqueado (ver bloqueadores).
- [x] Workflow visual configurable — nuevo tab "⚙️ Workflow" en Administrador: crea reglas propias (módulo, campo, condición, valor, mensaje) sin tocar código, además de las 3 fijas de Automatizaciones. Cada regla activa genera notificaciones reales. Probado en vivo con una regla real (13 coincidencias correctas).

## Bloqueadores externos (requieren algo de tu parte, no lo puedo resolver yo solo)

- ~~Envío real de correo~~ — **resuelto (2026-07-12)**: API key de Resend creada (`pmps-crm-notificaciones`, solo permiso de envío, restringida a `sportcstudio.com`), función `send-email` desplegada en Supabase, probada en vivo (correo recibido). Detalle técnico abajo.
- ~~WhatsApp Business API real~~ — descartado por decisión de Pako (2026-07-12): tiene costo (Meta + proveedor intermediario) y no se justifica todavía. El botón directo wa.me (fase 1, sin costo) se queda como está.
- Integración de Calendario real (Google Calendar, sincronización de eventos): el .ics descargable (fase 1) cubre el caso de uso básico; una sincronización en vivo requeriría autorizar el conector de Google Calendar más adelante, si hace falta.

### Cómo quedó armado el envío de correo (para referencia técnica)

- Edge Function `send-email` en el proyecto de Supabase, con la key de Resend embebida en su propio código (no en `index.html` — esa función corre solo en el servidor, nunca se manda al navegador).
- Nota: el dashboard de Supabase de este proyecto (`invwksntpqxsmkqycybf`) no es visible desde la cuenta de Supabase con la que Pako tiene sesión iniciada en Chrome (esa cuenta solo ve el proyecto "ProKicks Arena") — por eso se optó por este método en vez de guardar la key como "secret" del proyecto vía dashboard. Si en algún momento se quiere mover la key a una variable de entorno real, hace falta entrar al dashboard con la cuenta correcta que sí administra este proyecto.
- Solo envía a los 6 usuarios con cuenta real de Supabase Auth (requiere JWT válido); Daniel/Invitado siguen viendo el aviso solo dentro de la app hasta que tengan cuenta.
- Remitente: `notificaciones@sportcstudio.com`.

### Cómo quedó armado "no dejar nada suelto" en notificaciones (2026-07-12)

- **José Carlos en copia siempre**: las dos funciones que mandan correo (`send-email`, disparada al abrir la app, y `run-automatizaciones`, la automática) agregan a José Carlos en CC en cada correo, salvo que él mismo sea el destinatario principal (para no mandarle dos copias).
- **Corre sola, sin depender de que alguien abra la app**: nueva función `run-automatizaciones` en Supabase, programada con `pg_cron` + `pg_net` para ejecutarse dos veces al día — 8:00 am y 2:00 pm hora de Ciudad de México. Revisa leads/clientes contra las reglas fijas y las de Workflow, crea la notificación y manda el correo (con José Carlos en copia) si es algo nuevo.
- Probada en vivo simulando exactamente cómo la llamaría el cron: respuesta `status_code: 200`, `{"ok":true,"notificaciones_nuevas":0}` — corrió sin errores; el "0 nuevas" es porque en ese momento no había nada que no se hubiera avisado ya (el sistema evita duplicados a propósito).
- **⏸️ PAUSADA a propósito (2026-07-12)**: para que ningún usuario reciba correos automáticos antes de que se libere v2 y se confunda, los dos jobs de cron (`run-automatizaciones-manana`, `run-automatizaciones-tarde`) fueron desactivados (`cron.unschedule`). La función sigue desplegada y lista — solo hay que re-agendarla cuando decidan liberar v2. Para reactivarla, correr de nuevo el `cron.schedule(...)` de ambos horarios (8am/2pm hora de México) contra la función `run-automatizaciones`.
- **⏸️ CC a José Carlos pausado también (2026-07-12)**: mientras la automática está detenida, el correo que sí sigue activo es el que se dispara al abrir la app. Para que José Carlos no reciba nada antes de que Pako le avise, se desplegó `send-email` v4 con una bandera `CC_JOSE_CARLOS_ACTIVO = false` — nadie le manda copia por ahora. Cuando Pako le avise (le dijo que mañana entra a revisar y le explica que empezará a recibir correos de prueba), avisar para cambiar la bandera a `true` y redesplegar (toma 1 minuto).

(Marca cada uno aquí conforme se vaya cerrando, o pídeme que lo actualice.)
