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
- [x] Notificaciones — centro de notificaciones en la app (campana con contador, por usuario, tabla `notificaciones` en Supabase), probado en vivo. **Pendiente real**: el envío por correo en sí requiere conectar un proveedor (Gmail autorizado o una API key de Resend/SendGrid) — sin eso, las notificaciones se quedan dentro de la app, no salen a la bandeja de entrada.
- [x] Integraciones externas fase 1 (sin costo/credenciales) — botón 💬 WhatsApp (abre chat directo) en Leads/Clientes/Directorio, y botón 📅 para descargar recordatorio .ics (Google/Outlook/Apple Calendar) en pagos por vencer. Probado en vivo. Correo real sigue bloqueado (ver bloqueadores).
- [x] Workflow visual configurable — nuevo tab "⚙️ Workflow" en Administrador: crea reglas propias (módulo, campo, condición, valor, mensaje) sin tocar código, además de las 3 fijas de Automatizaciones. Cada regla activa genera notificaciones reales. Probado en vivo con una regla real (13 coincidencias correctas).

## Bloqueadores externos (requieren algo de tu parte, no lo puedo resolver yo solo)

- Envío real de correo: necesito que autorices el conector de Gmail (Configuración de Claude → Conectores) o me des una API key de Resend/SendGrid.
- WhatsApp Business API real (mensajes automáticos, no solo el botón wa.me): requiere una cuenta de WhatsApp Business API (Meta) — costo y alta aparte.
- Integración de Calendario real (Google Calendar): requiere autorizar el conector de Google Calendar.

(Marca cada uno aquí conforme se vaya cerrando, o pídeme que lo actualice.)
