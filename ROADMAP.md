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

## Nueva línea de negocio: reutilizar como base para futuros clientes (2026-07-12)

Decisión de Pako: considerar PMPS OS v2 como plantilla base para clientes con requerimientos similares (CRM + automatizaciones + reportes), no solo como sistema exclusivo de Menlun.

- **Reutilizable tal cual (core)**: autenticación (Supabase Auth), sistema de notificaciones + correo real, motor de Workflow visual configurable, auditoría de cambios (`activity_log`), exportación PDF/Excel, arquitectura base (single-file + Supabase + Vercel).
- **A adaptar por cliente**: módulos de negocio específicos (Presupuesto/Forecast/Ventas Históricas están hechos a la medida de cómo Menlun organiza sus datos), rebrand visual (logo/colores/nombre), nombres de campos y etapas del embudo si el giro del cliente es distinto.
- **Pendiente antes de vender esto como plantilla**:
  - Separar código en capa "core" (reutilizable) vs. capa "custom Menlun", para que el siguiente desarrollo no arrastre nada específico de este cliente.
  - Cada cliente futuro necesita su propio proyecto de Supabase (no compartir base de datos entre clientes).
  - Cada cliente futuro necesita su propia key de Resend/dominio de correo (o operarlo Pako en su nombre, a definir).
- **Estimado de costo de mercado si se mandara hacer de cero** (referencia, no cotización formal): freelancer senior en México ~$350,000-700,000 MXN (4-6 meses); agencia de software ~$25,000-60,000 USD; freelancer remoto económico ~$10,000-18,000 USD. La ventaja real de este desarrollo fue la velocidad (días, no meses) gracias al trabajo asistido por IA.

## 🚨 Migración de emergencia a nuevo proyecto Supabase (2026-07-13)

**Qué pasó**: el proyecto original de Supabase (`invwksntpqxsmkqycybf`) se volvió inaccesible — mi conector perdió todo permiso sobre él (hasta un `select 1` fallaba) y el proyecto ya no aparece en la lista de proyectos disponibles. No hay evidencia de que un tercero (mencionaste "Codex") haya entrado y borrado algo; lo único confirmado es que el acceso se cortó del lado del conector. Pako decidió no investigar más y migrar de cero al proyecto nuevo `diqbmyqvuyollvlvjniz` ("PAKO").

**Hallazgo crítico antes de migrar**: el proyecto nuevo NO estaba vacío — ya tenía un sistema completo en producción de **SM Soluciones** (otro negocio de Pako): tablas `usuarios`, `clientes`, `proyectos`, `tareas` (57 filas), `comentarios` (511 filas), `inventory_devices`/`inventory_movements` (200 filas cada una), etc. Para no chocar ni un byte con ese sistema, **todas las tablas de PMPS se crearon con el prefijo `pmps_`** dentro del mismo proyecto (`pmps_usuarios`, `pmps_clientes`, `pmps_leads_menlun`, `pmps_notificaciones`, `pmps_activity_log`, `pmps_workflow_reglas`, `pmps_forecast_2026`, `pmps_presupuesto_2026`, `pmps_presupuesto_asesores`, `pmps_productos`, `pmps_equipos_comodato`, `pmps_ventas_historicas`, `pmps_reportes`). SM Soluciones no fue tocado en absoluto.

**Qué se reconstruyó**:
- Las 13 tablas de PMPS, con columnas reconstruidas a partir del código real de `index.html` (no adivinadas al azar) — se revisó cada módulo para que las columnas coincidan exactamente con lo que el frontend espera.
- RLS activado en las 13 tablas (lectura abierta, escritura solo vía `secure_write()`).
- Funciones `secure_write()` y `crear_notificacion()` recreadas con la misma lógica de antes (validación de PIN del lado del servidor, dedup de notificaciones).
- Los 6 usuarios reconstruidos en `pmps_usuarios` (Administrador, Arturo, Jesús, José Carlos, Ulises, Ma. Teresa) y sus 6 cuentas reales de Supabase Auth recreadas con contraseñas temporales nuevas (ver abajo).
- Edge Functions `send-email` y `run-automatizaciones` redesplegadas en el proyecto nuevo, con la misma lógica de CC a José Carlos **pausada** (igual que antes de la migración) y el cron **sin programar todavía** (no se reactivó automáticamente).
- `index.html` actualizado con la URL y key del proyecto nuevo, y todas las referencias de tabla renombradas a `pmps_*`. Subido a GitHub → Vercel lo redespliega solo.

**⚠️ Lo que se perdió y no se pudo recuperar**: todos los datos reales que vivían en las tablas del proyecto viejo (leads/clientes cargados, historial de ventas, forecast, presupuesto capturado, catálogo de productos, equipos en comodato, reportes FO-VE-01). Las tablas nuevas tienen la **estructura correcta pero están vacías**. Los módulos de Leads y Clientes del CRM principal (Kanban) siguen funcionando normal porque esos viven en el Google Apps Script externo, no en Supabase — esos datos NO se perdieron.

**Contraseñas temporales nuevas (avisar a cada quien, cambiarlas apenas entren)**:
- Administrador (pako@sportcstudio.com): `Pmps2026!`
- Arturo: `Pmps2026Art!`
- Jesús: `Pmps2026Jes!`
- José Carlos: `Pmps2026JC!`
- Ulises: `Pmps2026Uli!`
- Ma. Teresa: `Pmps2026Ter!`

**Pendiente para revisar mañana con calma**:
1. Entrar y probar cada módulo (Forecast, Presupuesto, Catálogo, Equipos, Ventas Históricas, Reportes, Directorio) — si algo tira error de "columna no existe", avisar de inmediato para corregir el esquema (es rápido).
2. Recargar los datos reales si tienes los Excel originales o algún respaldo — sin eso, las tablas siguen vacías.
3. Decidir cuándo reactivar el cron de `run-automatizaciones` y el CC a José Carlos (ambos quedaron igual de pausados que antes de la migración).
4. Auditoría completa pendiente (#32) — hacerla ya en el proyecto nuevo, no en el viejo (que ya no existe para nosotros).

(Marca cada uno aquí conforme se vaya cerrando, o pídeme que lo actualice.)

## ✅ Recuperación de datos reales post-migración (2026-07-14/15)

Tras la migración de emergencia, se recargaron los datos reales desde los 4 Excel que Pako ya tenía disponibles, más un fix crítico de login. Esto se hizo mientras Pako dormía, con autorización explícita ("dale en loop hasta terminar todo... tienes mi autorización para entrar a todo").

**Fix de login (crítico)**: después del reseteo de contraseñas, el login fallaba con error 500 genérico para los 6 usuarios. Causa raíz: al recrear las cuentas en `auth.users` vía SQL manual, columnas como `email_change` quedaron en `NULL` en vez de cadena vacía `''` — un bug conocido de GoTrue (Supabase Auth) que rompe el login de forma silenciosa. Se corrigió con un `UPDATE` que llena esas columnas con `''`. Contraseña de Pako reseteada de nuevo a `Pmps2026Nueva!` y verificada con una prueba directa contra el endpoint de auth (login exitoso confirmado).

**Nota sobre el susto del "sí veo datos"**: lo que se vio en una captura de pantalla (Presupuesto con columnas históricas C'17-C'25 llenas) nunca estuvo en riesgo — esas columnas son constantes fijas dentro del código de `index.html`, no vienen de la base de datos. Los valores de C'26 que parecían llenos eran memoria vieja de la pestaña del navegador (abierta desde antes de la migración) — se confirmó consultando la base directamente (todo en cero) y se resolvió con un refresh forzado (Cmd+Shift+R).

**Datos recargados desde los Excel originales**:

| Módulo | Tabla | Fuente (archivo / hoja) | Filas cargadas |
|---|---|---|---|
| Presupuesto general 2026 | `pmps_presupuesto_2026` | PRESUPUESTO PMPS® 2026.xlsx / "P 2026 (NUEVO)" | 12 (meses) |
| Presupuesto por asesor | `pmps_presupuesto_asesores` | PRESUPUESTO PMPS® 2026.xlsx / hojas Corpo, Arturo, Teresa, Jesus, Asesor LD, PACIFICO, DISTRIBUIDOR | 84 (7 asesores × 12 meses) |
| Forecast 2026 | `pmps_forecast_2026` | FORECAST_OPERADORA_2026.xlsx / hojas CORP, ASA, JGT, MTOS, ALYD, PACIFICO | 259 (por producto/asesor) |
| Catálogo de productos | `pmps_productos` | FORECAST_OPERADORA_2026.xlsx / hoja "ARTÍCULOS" | 320 |
| Ventas Históricas | `pmps_ventas_historicas` | VENTAS PMPS.xlsx / hojas 2020 a 2026 | 622 (7 años) |
| Directorio (Menlun) | `pmps_leads_menlun` | Matriz_Clientes-Precio-Producto.xlsx / hojas "2025 CLIENTES" y "2026 CLIENTES" | 122 clientes (con vendedor asignado; 82 con sector/estado cruzado desde Ventas Históricas 2026) |

**Actualización 2026-07-15**: Pako compartió `FO-VE-##_Equipos dosificadores y refacciones.xlsx`, con el inventario de equipos/refacciones en covacha (almacén). Se cargaron **29 equipos** en `pmps_equipos_comodato` (nombre del equipo, proveedor, código Microsip/interno y stock en notas, estado "Disponible"/"Sin stock" según existencias). Nota importante: este archivo es inventario de almacén, no un registro de qué cliente tiene qué equipo en comodato — por eso el campo `cliente` quedó vacío en los 29 registros y `ubicacion` se puso como "Covacha (almacén PMPS)". Si existe un registro de qué equipos están instalados en qué cliente (aunque sea informal), se puede migrar después asignando cada equipo a su cliente correspondiente.

**Lo que sigue vacío (sin fuente disponible)**:
- `pmps_reportes` (Reportes FO-VE-01) — son reportes de servicio que se generan hacia adelante desde la app, no hay histórico que recuperar de Excel.
- El Directorio quedó con datos básicos (nombre, asesor, sector/estado cuando fue posible cruzarlo); campos como razón social, giro, tier, municipio exacto, teléfono, email y notas no estaban en la Matriz de Clientes-Precio-Producto (esa hoja es una matriz de precios por cliente/producto, no un directorio completo) — si existe un directorio más detallado en otro archivo, se puede enriquecer después.
- El `index.html` subido como "CRM v1" (archivo aparte que Pako compartió) se revisó y confirmó que es una versión anterior del mismo código de la app, sin datos nuevos que recuperar — usa el mismo Google Apps Script para Leads/Clientes (que nunca se perdió) y su propia conexión a Supabase apunta al proyecto viejo ya inaccesible.

**Resultado**: PMPS OS v2 en el proyecto nuevo (`diqbmyqvuyollvlvjniz`) ya tiene datos reales operativos en Presupuesto, Forecast, Catálogo, Ventas Históricas y Directorio. Pendiente para cuando Pako despierte: revisar visualmente cada módulo, confirmar que los números cuadran con lo que recuerda, y decidir sobre Equipos (única tabla sin fuente de datos).

## ✅ Cierre de los 5 bugs reales de feedback de José Carlos + 1 bug sistémico descubierto (2026-07-16)

José Carlos mandó 19 puntos de feedback sobre el CRM en producción. Diagnóstico separó los 19 en: 5 bugs reales confirmados, varios "no-bugs" (memoria vieja del navegador — se resuelven con refresh), y varias solicitudes de feature nuevas (fuera de este cierre). Instrucción de Pako: **"todo trabaja en un loop hasta dejarlo cerrado y hayas auditado todo, nada de parches"** — arreglos de raíz únicamente, cada uno probado en vivo contra la app real desplegada, no solo revisado en código.

**1. Usuarios/Claves no persistían (bug real, raíz profunda)**: toda la lista de usuarios (`USERS_LIVE`, `CLAVES_LIVE`, `EMAIL_MAP`) estaba escrita como constantes de JavaScript directamente en el código fuente de `index.html` — completamente desconectada de la tabla `pmps_usuarios`, que sí existía y tenía los datos correctos. Cualquier edición de un admin era solo en memoria y se perdía al recargar la página.
- **Arreglo de raíz**: se agregaron columnas `iniciales`, `clave_local`, `activo` a `pmps_usuarios`; se crearon dos funciones RPC nuevas — `usuario_verificar_clave_local()` (compara la contraseña contra el hash guardado del lado del servidor, nunca se manda el hash al navegador) y `usuario_set_clave_local()` (solo un admin puede cambiar la clave de alguien). Para las cuentas con correo real (Administrador, José Carlos), se creó una Edge Function `admin-usuarios` que usa la Service Role Key del lado del servidor para crear/cambiar contraseña/eliminar cuentas reales de Supabase Auth — la key nunca toca el navegador. Todo el módulo de Usuarios/Claves en `index.html` se reescribió para leer y escribir contra la base de datos real en vez de las constantes hardcodeadas.
- **Se agregaron 2 usuarios nuevos que José Carlos pidió**: Daniel y Jorge (reemplazando a "Rocío", que ya no aplica).
- **Probado en vivo**: login dropdown ahora se llena dinámicamente desde la base (8 usuarios reales, con ⚙️ marcando cuentas con correo); panel de Usuarios muestra roles y contadores reales desde la base.

**2. Workflow "no me dejó crear una regla nueva" (bug real)**: el código pedía la lista de reglas ordenada por una columna `creado_en` que no existe en la tabla — la columna real es `created_at`. Esto hacía que la consulta fallara silenciosamente y la lista se quedara vacía, aunque el insert sí hubiera funcionado.
- **Arreglo de raíz**: corregido el nombre de columna en la consulta.
- **Bug más profundo descubierto al probar este fix**: al crear una regla de prueba, saltó un error de "columna id no puede ser NULL" — esto expuso que la función central `secure_write()` (usada por TODA la app para insertar registros) tenía un bug de raíz: su rama de inserción usaba `jsonb_populate_record` contra un registro vacío, lo cual pone en `NULL` cualquier columna que no venga en el JSON enviado — **incluyendo columnas con un DEFAULT real como `id uuid default gen_random_uuid()`**. Esto rompía silenciosamente cualquier inserción en cualquier tabla de PMPS que use ese patrón de id (no solo Workflow).
- **Arreglo de raíz (el más importante de esta sesión)**: se reescribió la rama de inserción de `secure_write()` para construir un `insert into tabla (columnas...) values (valores...)` dinámico usando solo las columnas presentes en el JSON recibido — dejando que cualquier columna omitida use su DEFAULT real, igual que ya hacía la rama de actualización.
- **Probado en vivo**: se creó una regla de prueba real ("TEST-AUDITORIA-BORRAR") que se guardó con id generado correctamente y coincidió en vivo con 57 leads reales — luego se eliminó de la base para no dejar basura de prueba.

**3. Equipos "no guardó la foto" (bug real)**: el bucket de Storage `equipos-fotos` nunca se recreó después de la migración de emergencia — solo existía un bucket sin relación (`content-media`). Cualquier intento de subir foto fallaba en la llamada a Storage.
- **Arreglo de raíz**: se creó el bucket `equipos-fotos` con las políticas correctas de lectura/escritura, probado con una subida y descarga real de imagen de prueba.

**4. Reportes "dropdown de cliente vacío" (bug real)**: el formulario de nuevo reporte leía los clientes desde la tabla `pmps_clientes`, que tiene 0 filas — los clientes reales del CRM viven en el Google Apps Script externo (cargados en el arreglo en memoria `clientes` al iniciar sesión), nunca se sincronizaron a esa tabla.
- **Arreglo de raíz**: se cambió la fuente del dropdown para leer del arreglo `clientes` ya cargado en memoria, igual que el resto de la app.
- **Probado en vivo**: el dropdown ahora muestra 50 clientes reales; se guardó y confirmó un reporte de prueba end-to-end, luego se eliminó de la base.

**5. Manual de Usuario da error 404 (bug real)**: el enlace de descarga apuntaba a un PDF que nunca se generó ni se subió al repositorio.
- **Arreglo de raíz**: se generó un manual de usuario completo y real (13 secciones cubriendo todos los módulos: login, CRM/Kanban, Catálogo, Equipos, Reportes, Forecast, Ventas Históricas, Presupuesto, Directorio, Usuarios/Claves, Workflow, Automatizaciones, FAQ) y se subió al repositorio en la ruta exacta que la app espera.
- **Probado en vivo**: la URL de descarga ya no da 404.

**Desplegado**: `index.html` actualizado (Usuarios/Claves + Workflow + Reportes) y el manual PDF, ambos subidos a GitHub → Vercel los redesplegó automáticamente. Verificado en vivo iniciando sesión como José Carlos en la app real (no solo pruebas de API): login dinámico, panel de Usuarios con datos reales, regla de Workflow creada y visible, dropdown de Reportes poblado, los tres probados end-to-end y limpiados después.

**Nota menor pendiente (no bloqueante)**: quedó una imagen de prueba huérfana en el bucket `equipos-fotos` de una prueba anterior (1x1 px, inofensiva) — se puede borrar manualmente desde el dashboard de Supabase Storage cuando sea conveniente.

**Los 19 puntos originales de feedback que NO requerían arreglo** (memoria vieja de navegador o solicitudes de feature nueva, fuera del alcance de este cierre): quedan documentados en el diagnóstico que se le compartió a José Carlos; si se decide construirlas, entran como nuevas líneas al roadmap (ej. filtros de Kanban por etapa, edición de acciones de Automatizaciones más allá de encender/apagar, botón de carga directa de formato Microsip en Ventas Históricas, límite de gráficas por año).
