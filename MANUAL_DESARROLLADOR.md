# Manual del Desarrollador — Sistema de Créditos Corporativos OxoHotel

> Documento interno. **No publicar en repositorios públicos** (contiene IDs de proyecto, esquema de base de datos y hallazgos de seguridad).
> Versión 1.0 — última actualización 2026-07-04.
> Audiencia: cualquier desarrollador que deba mantener, extender o depurar el sistema.

---

## 1. Qué es esto

Automatiza de punta a punta la gestión de crédito corporativo de la cadena hotelera OxoHotel: desde que un hotel/empresa solicita un cupo de crédito, hasta que el equipo de Crédito y Cartera lo aprueba, activa y el equipo de Reservas lo consulta al momento de facturar una reserva.

Es **4 proyectos independientes de Google Apps Script** (cada uno con su propio `Script ID`, desplegado como Web App), que comparten:
- Un **Google Sheet de configuración** (catálogos: hoteles, tipos de solicitud, tipos de documento, ejecutivos comerciales, usuarios).
- Una **base de datos MySQL** (hosting Hostinger), accedida vía JDBC directo desde Apps Script — es la fuente de verdad real del negocio (solicitudes, empresas, créditos, documentos, auditoría).
- Google Drive (expedientes/documentos) y Gmail (todas las notificaciones).

```
┌─────────────┐      ┌───────────────────┐      ┌─────────────────────────┐      ┌────────────────────────┐
│   Índice    │─────▶│    Formulario      │─────▶│    Panel de Control     │      │  Equipo de Reservas    │
│  (portal)   │      │ (público, wizard)  │      │ (interno, ADMIN only)   │      │ (interno, solo lectura) │
└─────────────┘      └────────┬───────────┘      └───────────┬─────────────┘      └───────────┬─────────────┘
                               │  INSERT staging                │ CRUD completo                  │ SELECT (JDBC)
                               ▼                                ▼                                ▼
                     ┌──────────────────────────────────────────────────────────────────────────────┐
                     │                     MySQL (Hostinger) — 21 tablas                              │
                     │  solicitudes · empresas · representantes · documentos · usuarios · auditoria…  │
                     └──────────────────────────────────────────────────────────────────────────────┘
```

Cada proyecto se despliega y versiona por separado con **`clasp`** (hay un `.clasp.json` por carpeta). No hay build step, bundler ni transpilador: es JavaScript ES5/ES6 plano ejecutado por el runtime V8 de Apps Script, y HTML/CSS/JS de cliente sin framework (sin React/Vue/Angular).

| Módulo | Carpeta | Script ID | Audiencia | Acceso webapp |
|---|---|---|---|---|
| Índice | `Indice/` | `1B8S5pYeIU8JyZSf3i7MHxE-b0UKNfFZ6HCjlAt0_oINXbQw2Evycwasu` | Todos | `ANYONE_ANONYMOUS` |
| Formulario | `Formulario/` | `1M-G5ZapyvNd8roGfzA7MSR7XZdDKQX9eXihLcpWqHe2lFhXN_lF513FE` | Público (hoteles/empresas) | `ANYONE_ANONYMOUS` |
| Panel de control | `Panel de control/` | `1YuP1YSqdhMWUFeYVoUv9BjxJe4lHqQFbfO8mx0tdVhUkgRCHwVo6H4QF` | Crédito y Cartera (rol ADMIN) | `DOMAIN` (Google Workspace OxoHotel) |
| Equipo de reservas | `Equipo de reservas/` | `130KZ4WCtqWnwklAoCs4xbcSwsUymWi6ukCtRhu-fNs3VuERjny0wa-v1` | Reservas (ADMIN/CONSULTAS) | `ANYONE_ANONYMOUS` |

Todos usan `webapp.executeAs: "USER_DEPLOYING"`: el código corre siempre con la identidad de quien desplegó el proyecto, nunca con la del usuario que abre el link — por eso cada módulo implementa su **propio sistema de login** (no delega en la identidad de Google), salvo Índice, que no tiene control de acceso propio porque no expone datos.

---

## 2. Convenciones que aplican a los 4 módulos

- **`doGet(e)`** es el único punto de entrada HTTP en cada proyecto; sirve `Index.html` vía `HtmlService.createTemplateFromFile`.
- **`include(filename)`** es el patrón estándar para insertar parciales HTML (`<?!= include('archivo'); ?>`, con `!` para no escapar el HTML/CSS/JS inyectado).
- **Sin módulos JS**: todos los `.html` incluidos terminan concatenados en un único `<script>` — todo vive en el mismo scope global. El **orden de `include()` en `Scripts.html`/`Index.html` importa** porque refleja dependencias de definición (variables/objetos de estado deben declararse antes de que otros archivos los usen).
- **Comunicación cliente-servidor**: siempre `google.script.run.withSuccessHandler(cb).withFailureHandler(cb).apiNombreFuncion(args)`. Convención de nombres: funciones públicas invocables desde el cliente llevan prefijo `api` (`apiLogin`, `apiGetSolicitudes`...); funciones privadas/helpers llevan sufijo `_` (`dbGetConnection_`, `getCfg_`...) — es solo convención, Apps Script no tiene encapsulamiento real, cualquier función en el scope global es técnicamente invocable si se conoce su nombre.
- **Sesión y roles**: cada módulo con control de acceso (Formulario es público y no aplica) implementa login propio contra una hoja `Usuarios`, contraseñas con **SHA-256 sin salt** (`Utilities.computeDigest`), y tokens de sesión guardados en `CacheService` (no en MySQL, no hay tabla `sesiones` real en uso pese a existir en el esquema — ver §7). La verificación de rol (`ADMIN` vs `CONSULTAS`/legado `RESERVAS`) se repite manualmente función por función; no hay middleware/decorador central.
- **Google Sheets vs MySQL**: los 4 módulos conservan código legado que lee/escribe Google Sheets para solicitudes (`SPREADSHEET_ID`), pero **la fuente de verdad real y activa es MySQL** (`CONFIG_SPREADSHEET_ID` sí sigue vigente, es donde viven los catálogos). No asumas que una hoja de cálculo refleja el estado real de una solicitud — verifica siempre contra la tabla `solicitudes` en MySQL.

---

## 3. Módulo Índice (`Indice/`)

El más simple: una landing page con 3 tarjetas que enlazan a Formulario, Panel de Control y Equipo de Reservas. No tiene lógica de negocio ni autenticación.

- `getUrls()` lee `FORMULARIO_URL` / `PANEL_URL` / `RESERVAS_URL` de Script Properties y los inyecta en los `href` de las tarjetas. **El vínculo entre los 3 proyectos es 100% manual**: si se crea una nueva versión de despliegue de cualquiera de ellos, hay que actualizar la Script Property aquí a mano.
- `crearHojaFondoHoteles()` / `getBackgroundImages()`: gestionan una hoja "FondoHoteles" con URLs de imágenes — **código muerto**, `Script.html` actual no las invoca (el fondo real es un canvas animado). Queda como remanente de una versión anterior; confirmar con el equipo antes de eliminarlo o si se planea reactivar un carrusel de fondos.
- `inicializarPropiedadesDeScript()`: utilidad de primer setup, siembra las 4 Script Properties con placeholders. Ejecutar manualmente una sola vez al clonar el proyecto a un entorno nuevo.

**Script Properties**: `FORMULARIO_URL`, `PANEL_URL`, `RESERVAS_URL` (todas opcionales — si faltan, el link queda en `#` sin aviso al usuario), `SPREADSHEET_ID` (opcional, con fallback hardcodeado).

---

## 4. Módulo Formulario (`Formulario/`)

Wizard público de 5 pasos donde un hotel/empresa solicita crédito. Público y anónimo (`ANYONE_ANONYMOUS`), sin login — el único control de identidad es un código OTP enviado al correo del representante legal.

### 4.1 Inventario de archivos

| Archivo | Rol |
|---|---|
| `Code.js` | `doGet`, constantes `APP`, `getCfg_`, catálogos, todas las APIs del ciclo de vida de la solicitud |
| `Email.js` | Plantillas de correo (confirmación de radicado, código OTP) |
| `Trigger.js` | Trigger instalable `syncQueueToMySQL` (cada 1 min), staging→producción |
| `docs/PdfGenerator.js` | Generador de PDF **de respaldo**, server-side (HTML→Google Docs→PDF) |
| `docs/Migration.js` | Migración manual, una sola ejecución, de catálogos hardcodeados a Sheet |
| `Index.html` | Esqueleto del wizard, modal de bienvenida/políticas, panel de consulta de estado |
| `Init.html` / `State.html` / `Controls.html` / `UI.html` / `Docs.html` / `Submit.html` | Cliente JS: bootstrap, estado del wizard, widgets, navegación/borrador, documentos, resumen+OTP+PDF+envío |
| `FAQ.html` | Chatbot flotante (Gemini) |
| `TestPdf.html` | Página aislada de prueba de generación de PDF client-side (`?view=testpdf`) |

### 4.2 Flujo end-to-end

1. `doGet` sirve `Index.html`; `init()` trae catálogos (`apiGetCatalogos`) y muestra el modal de bienvenida/políticas.
2. El usuario avanza los 5 pasos (`goTo`/`validateStep`); puede guardar un **borrador en `localStorage`** en cualquier momento (`draftSave`, clave `oxo_credito_draft_v1`) — no sincroniza con servidor, se pierde si cambia de navegador.
3. En el paso 5, tras aceptar términos: `apiSendVerificationCode` envía OTP de 6 caracteres al correo del representante legal (`CacheService`, TTL 10 min); `apiVerifyCode` lo valida — **el código no se invalida tras un uso exitoso** (la línea que lo borraría está comentada) y **el servidor no vuelve a comprobar `emailVerified` en `apiFinalizarSolicitud`**, es una validación solo de UX.
4. Al enviar (`submitForm()`): `apiStartSubmission` crea la carpeta en Drive e inserta un registro "borrador" en `stg_solicitudes` (sentinela `estado_sync='ERROR' / ultimo_error='BORRADOR'`, no una columna de estado dedicada — las constantes `APP.ESTADO_BORRADOR/ESTADO_PENDIENTE` existen pero no se usan en ningún lado). Cada documento se sube en **chunks de 256 KB directo a Drive vía upload resumible** (`apiInitUploadDocumento` + N×`apiUploadChunk`, sesión de 10 min en `CacheService`), evitando el límite de payload de `google.script.run`.
5. El PDF del formato de solicitud se genera **dos veces por diseño** (no simultáneo, es primario+respaldo):
   - **Primario, client-side**: `html2canvas` + `jsPDF` sobre un iframe oculto (`Submit.html`).
   - **Respaldo, server-side**: si el cliente falló, `docs/PdfGenerator.js` genera el mismo documento con una plantilla HTML **distinta** vía el truco "HTML→Google Doc temporal→exportar PDF→borrar Doc". **Ambas plantillas ya divergen visualmente hoy** — cualquier cambio de formato/branding debe replicarse en las dos.
6. `apiFinalizarSolicitud` valida documentos obligatorios, marca `estado_sync='PENDIENTE'`, envía los correos (`enviarCorreoSolicitud_`, al representante + al equipo de créditos, con el PDF adjunto) y sincroniza inmediatamente staging→producción.
7. El trigger `syncQueueToMySQL` (instalado manualmente ejecutando `setupTriggerMinuto()` una vez, cada 1 min) es la red de seguridad que reintenta cualquier solicitud que quedó `PENDIENTE` sin sincronizar.
8. Consulta de estado sin login (`apiGetSolicitudStatusByCodigo`): **solo requiere el código de radicado**, sin segundo factor ni rate-limiting — quien lo conozca puede consultarlo.

### 4.3 Puntos de atención para el mantenedor

- **Límite de 30 MB por archivo (`APP.MAX_FILE_BYTES`) solo se valida en el cliente.** El servidor nunca lo comprueba — un cliente que se salte la UI podría subir archivos más grandes.
- **Documentos**: 4 obligatorios (`CERT_EXISTENCIA`, `CEDULA_REP`, `RUT`, `ESTADOS_FIN`), con `ESTADOS_FIN` intercambiable por `CARTA_SIN_EEFF` (carta de imposibilidad). Vigencia validada client-side: `CERT_EXISTENCIA` ≤200 días, `CARTA_SIN_EEFF` ≤180 días.
- **Catálogos**: `getCatalogoFromSheet_()` tiene un comentario que dice "caché de 6 horas" pero **no hay ningún `CacheService` implementado ahí** — cada llamada lee el Sheet en vivo. La infraestructura de invalidación (`adminClearCatalogCache`) sí existe, sugiriendo que el cacheo se quitó o nunca se terminó.
- **Chatbot** (`FAQ.html` + `apiChatWithGemini`): modelo `gemini-2.5-flash` vía REST directo (no SDK), API key en Script Property `GEMINI_API_KEY`, nunca expuesta al cliente. Prompt de sistema fijo con reglas de negocio embebidas (anticipo 30%, plazo 30 días, vigencia de cupo 1 año, SLA 15 días hábiles). Sin reintentos/backoff ante 429/503.
- **No existe limpieza automática de borradores huérfanos** en `stg_solicitudes` — se sobrescriben solo si la misma empresa (mismo NIT) reintenta.

**Script Properties**: obligatorias `ROOT_FOLDER_ID`, `SPREADSHEET_ID`, `CONFIG_SPREADSHEET_ID`; opcionales con default `SHEET_SOLICITUDES`, `SHEET_DOCUMENTOS`, `CREDITOS_EMAIL`, `HOTEL_EMAIL`, `RECEPCION_EMAIL`; adicionales `DB_HOST/PORT/NAME/USER/PASS`, `SYNC_BATCH_SIZE` (default 25), `GEMINI_API_KEY`, `EJECUTIVOS_SS_ID`.

---

## 5. Módulo Panel de Control (`Panel de control/`)

El módulo más grande y crítico — SPA administrativa exclusiva para el equipo de Crédito y Cartera (rol `ADMIN`; `webapp.access: DOMAIN` es la primera barrera, el login propio es la segunda).

### 5.1 Backend — archivo por archivo

| Archivo | Responsabilidad | Notas clave |
|---|---|---|
| `Code.js` | `doGet`, utilidades de diagnóstico/hotfix | Ver §5.4 — ejecuta DDL en cada carga de página |
| `Config.js` | `getCfg_()` centralizado, constantes `PANEL` | Único punto que lee Script Properties de negocio |
| `Helpers.js` | Fechas, mapeo Sheet→objeto, montos en letras, catálogos de Sheets | Contiene una de las 3 copias de la regla "vencimiento a 365 días" |
| `Auth.js` | Login propio, tokens de sesión, CRUD de usuarios | SHA-256 sin salt; ver §5.5 |
| `Auditoria.js` | Log de auditoría en hoja `AUDITORIA` (no MySQL) | `apiGetLogAuditoria` sin control de rol propio |
| `ApiSolicitudes.js` | CRUD + aprobar/rechazar solicitudes, CRUD genérico de catálogos, notificaciones leídas | El archivo de negocio más denso (1435 líneas) |
| `ApiCreditos.js` | "Créditos" **virtualizados** desde `solicitudes` (no hay tabla de créditos independiente) | Importación masiva vía hoja `PLANTILLA_IMPORTAR_CREDITOS` |
| `ApiDocumentos.js` | Expediente documental; auto-sincroniza documentos subidos manualmente a Drive | Heurística por nombre de archivo para adivinar el tipo |
| `GeminiIA.js` | Auditoría documental con Gemini 2.5 Flash (multimodal, PDFs en base64) | Prompt de cientos de líneas con lista blanca de ~29 hoteles embebida |
| `Alertas.js` | Cron de vencimientos (documentos y cupo a 1 año) | Trigger **no** configurado en código (ver §5.4) |
| `Backup.js` | Backup completo (Sheets+MySQL a CSV) y restauración con código OTP | Backup de pre-restauración automático antes de restaurar |
| `BaseDatos.js` | Toda la capa JDBC: conexión, queries, migraciones | El archivo más grande (1986 líneas) |
| `Exportador.js` | Excel/CSV/Slides/PDF | Excel y PDF se generan delegando en las URLs de exportación nativas de Sheets/Slides, no con librerías propias |

**Conexión a MySQL** (`dbGetConnection_`, en `BaseDatos.js`): credenciales 100% desde Script Properties (`DB_HOST/PORT/NAME/USER/PASS`), 5 reintentos con 300ms de espera, `SELECT 1` de verificación, y un truco documentado en el propio código: **aleatoriza el casing del hostname en cada intento** para "evadir el pool de conexiones envenenado de Apps Script" (workaround de un problema conocido con JDBC + Hostinger). Si faltan credenciales retorna `null` silenciosamente; si la conexión falla tras reintentar, lanza excepción.

**Identificadores**: las solicitudes usan `uuid BINARY(16)` en MySQL; el código convierte constantemente entre el formato con guiones (cliente/URLs) y binario/hex (SQL) con el patrón `UNHEX(REPLACE(?, '-', ''))` / `HEX(s.uuid)`.

**Optimización de latencia notable**: `dbGetSolicitudesDesdeMySQL_` usa `JSON_ARRAYAGG(JSON_OBJECT(...))` de MySQL para traer una página completa de resultados en una sola celda JSON y minimizar round-trips JDBC — hay incluso una función `runDbDiagnostics()` en `Code.js` que compara 5 estrategias de query distintas por latencia.

### 5.2 Frontend — arquitectura SPA

Sin framework. `Index.html` es el shell único; `Scripts.html` concatena, en este orden (importa por dependencias de definición, no por ejecución):

```
JS_Init.html → JS_Router.html → JS_Helpers.html → JS_Dashboard.html → JS_Notificaciones.html
→ JS_Solicitudes.html → JS_HotelesCreditos.html → JS_Modal.html → JS_Documentos.html → JS_IA.html
→ JS_Acciones.html → JS_Auditoria.html → JS_Configuracion.html → JS_Toast.html → JS_LoadingProcess.html
→ JS_Usuarios.html → JS_Crud.html → JS_Backups.html
```

- **Estado global**: objeto `STATE` (bolsa de propiedades mutada libremente por todos los módulos, sin setters/getters). Varios módulos mantienen además su propio "islote" de estado (`CRUD_STATE`, `CONFIG_STATE`, `HOTELES_CREDITOS_STATE`...) en vez de centralizarlo.
- **Router** (`JS_Router.html`): `showView(viewId)` sin manejo de URL/history — recargar (F5) siempre vuelve al dashboard. Agregar una vista nueva exige tocar `VIEW_META`, la sección HTML, **y** los `if` de `showView()` y `recargarVista()` por separado (duplicación intencional pero frágil).
- **RPC**: siempre `google.script.run.withSuccessHandler/withFailureHandler`. No hay wrapper central — cada llamada repite manualmente `showLoading()`/`hideLoading()` y el `showToast` de error.
- **Modal de confirmación genérico** (`mostrarModalConfirmacion`, en `JS_Helpers.html`): componente reutilizable central para toda acción destructiva/sensible (aprobar, rechazar, eliminar, cerrar sesión...). Solo puede haber uno activo a la vez.
- **Gráficas**: Chart.js por CDN sin versión fijada (riesgo si el CDN sirve un major nuevo). Los sparklines de las tarjetas KPI son **decorativos y aleatorios** (`_generateTrend`, `Math.random()`), no reflejan tendencia real.
- **Tema oscuro — bug confirmado**: `toggleTheme()`/`applyTheme()` persisten bien en `localStorage`, pero `initTheme()` (`JS_Init.html`) **fuerza `'light'` siempre al arrancar**, sin leer el valor guardado. El tema oscuro nunca sobrevive a un F5.
- **Duplicaciones a tener presentes al extender**: badges de estado calculados en 3 lugares distintos (`getBadgeHtml`, `_getAuditBadge`, inline en `JS_Crud.html`); exportación Excel/CSV repetida casi textual en 3 módulos; `PROCESS_LOADER` reimplementado aparte en `JS_Backups.html`.

### 5.3 El modelo de "créditos" — importante para no confundirse

**No existe una tabla de créditos independiente activa.** `ApiCreditos.js` lo declara explícitamente: los créditos son una **vista virtual** calculada sobre `solicitudes` con estado aprobado, expandida por cada hotel presente en la columna JSON `cupos_distribuidos_json`. (La tabla `creditos_vigentes` sí existe en el esquema SQL — ver §7 — pero según lo revisado, el camino activo de "créditos" del Panel pasa por `solicitudes`, no por esa tabla; confirmar con el equipo si `creditos_vigentes` sigue alimentándose desde algún proceso no cubierto en esta revisión antes de asumir que está en desuso.)

### 5.4 Deuda técnica y decisiones a revisar

1. **`doGet()` ejecuta DDL en cada carga de página, sin flag de protección**: además de la migración condicional (protegida por `MIGRACION_PERFIL_CREADO_OK`), hay un segundo bloque que ejecuta `ALTER TABLE solicitudes MODIFY COLUMN perfil_creado TEXT NULL` **en cada `doGet`**, sin ninguna bandera — abre una conexión JDBC extra por cada carga del panel solo para reemitir un DDL que ya debería ser un no-op. Candidato claro a eliminar o proteger igual que el bloque 1.
2. **Regla de negocio "vencimiento a 365 días" triplicada**: en `mapRowToSolicitud_()` (JS sobre Sheets), `dbComputeAccion_()` (JS sobre MySQL) y `SQL_ACCION_CALC_` (expresión SQL). Cambiar la ventana de vigencia exige editar los tres lugares.
3. **Triggers de cron no versionados**: ni `cronEjecutarBackupDiario` (`Backup.js`) ni `cronAlertaVencimientos` (`Alertas.js`) tienen un `ScriptApp.newTrigger(...).create()` en el código — dependen de que alguien los haya configurado manualmente en el editor de Apps Script (Triggers → Add Trigger). Si ese trigger se pierde (por ejemplo al copiar el proyecto a otra cuenta), **no hay backups automáticos ni alertas de vencimiento** y nadie lo notaría hasta necesitarlos.
4. **Funciones de diagnóstico/hotfix residentes en producción**: `runDbDiagnostics`, `apiDiagnosticoSolicitud`, `diagnosticoExpediente`, `corregirFondoGarantias` (hotfix hardcodeado para un caso real puntual), `exportarCredencialesTemp`, `diagnosticoModelosGemini`, `testAnalisisIA`. Ninguna está pensada para el usuario final; algunas no verifican sesión/rol.
5. **Emails hardcodeados de fallback en lógica de negocio**: si el correo de un ejecutivo comercial resuelto contiene `amgonzalez@oxohotel.com`, `ApiSolicitudes.js` lo sustituye silenciosamente por una cuenta personal de Gmail — parece un resto de pruebas, revisar antes de que afecte una aprobación real.
6. **Etiqueta con mojibake** en el modal "Editar solicitud" del frontend: `Â¿¿Es Filial OxoHotel...` (error de codificación de caracteres en el propio HTML fuente).
7. **Catálogo "Monedas"** existe y es editable, pero el Formulario público tiene el campo de moneda fijo en `COP` (input oculto) — agregar una moneda ahí no habilita nada en el flujo real todavía.

### 5.5 Seguridad — hallazgos a revisar con prioridad

- **API key de Gemini con valor de respaldo hardcodeado en el código fuente** (`Config.js::getCfg_`): si `GEMINI_API_KEY` no está configurada como Script Property, el código escribe automáticamente una key embebida en el propio repositorio. Debe rotarse y eliminarse del código.
- **Contraseñas con SHA-256 sin sal** (`hashSha256_`, usado en Panel, Reservas). Vulnerable a rainbow tables / fuerza bruta si la hoja `Usuarios` o un backup se filtra. Recomendado migrar a un esquema con sal + KDF lenta.
- **Ventana de exposición de credenciales de BD**: `exportarCredencialesTemp()` (Panel) escribe host/usuario/password de MySQL en texto plano en una hoja `TEMP_DB_CONFIG` del Spreadsheet de configuración compartido, para que el proyecto "Equipo de reservas" las importe y luego borre la hoja. Mientras la hoja existe, cualquiera con acceso de lectura a ese Sheet puede leer las credenciales de la base de datos.
- **Funciones sensibles sin verificación de sesión/rol homogénea**: `apiGetLogAuditoria`, `runDbDiagnostics`, `apiDiagnosticoSolicitud` no validan `apiVerificarSesion`/rol dentro de la función — dependen de que el frontend no las exponga en la UI a quien no debería verlas. `access: DOMAIN` mitiga parte del riesgo (solo cuentas del Workspace de OxoHotel pueden ni siquiera cargar el panel), pero no reemplaza una validación explícita por función.
- **Código OTP reutilizable** (Formulario): tras validarse correctamente, el código no se invalida (la línea `cache.remove(...)` está comentada) — sigue siendo válido hasta que expire su TTL de 10 min.

**Script Properties del Panel** (lista completa): obligatoria `SPREADSHEET_ID`; opcionales con default `CONFIG_SPREADSHEET_ID` (fallback a `SPREADSHEET_ID`), `SHEET_SOLICITUDES`, `SHEET_DOCUMENTOS`, `CREDITOS_EMAIL`, `BCC_TECH_EMAIL` (fallback a `CREDITOS_EMAIL`), `RECEPCION_EMAIL`, `GEMINI_API_KEY` (con fallback inseguro, ver arriba); MySQL `DB_HOST/PORT/NAME/USER/PASS`; internas/autogestionadas `MIGRACION_PERFIL_CREADO_OK`, `BACKUP_FOLDER_ID`.

---

## 6. Módulo Equipo de Reservas (`Equipo de reservas/`)

Portal de **solo consulta** de créditos aprobados/activos, leído en tiempo real desde MySQL vía JDBC (no desde Sheets).

- **Login propio**, independiente del Panel: hoja `Usuarios` en `CONFIG_SPREADSHEET_ID`, mismo esquema de hash SHA-256 sin salt, sesión en `CacheService` con prefijo `SES_RESERVAS_` y TTL de 4h deslizante (`apiVerificarSesion` renueva el TTL en cada llamada).
- **Roles**: acepta `ADMIN`, `CONSULTAS` (nuevo) y `RESERVAS` (legado, aceptado por compatibilidad — dos nombres para el mismo concepto conviviendo en la whitelist de login). **Dentro de este portal específico, ADMIN y CONSULTAS tienen exactamente los mismos permisos** — ningún endpoint ni el frontend distinguen el rol más allá de mostrarlo como texto informativo. La única escritura permitida (marcar/desmarcar "Perfil creado (PMS/Hotel)") está disponible para cualquier sesión válida.
- **Consulta principal** (`SQL_CREDITOS_BASE_`): `solicitudes JOIN empresas JOIN estados_solicitud LEFT JOIN solicitudes_facturacion`, filtrando por estados relevantes (`APROBADA, ACTIVA, VENCIDO, VENCIDA, POR_VENCER, BLOQUEADO_CORPORATIVO, BLOQUEADO_GERENCIA`). El catálogo de hoteles sigue viniendo de Google Sheets, no de MySQL. Resultado cacheado 15 min (`CacheService`, fragmentado en bloques de 80 KB por el límite de 100 KB por valor).
- **Registro público deshabilitado**: alta de usuarios solo manual en la hoja `Usuarios`.
- **Recuperación de contraseña**: mismo patrón de código de 6 dígitos por correo (`CacheService`, TTL 10 min).

**Script Properties**: `CONFIG_SPREADSHEET_ID` (obligatoria), `SPREADSHEET_ID`/`PANEL_URL`/`FORMULARIO_URL` (opcionales), `DB_HOST/PORT/NAME/USER/PASS` (MySQL — `dbGetConnection_` no valida que `DB_PASS` esté presente, solo host/name/user).

---

## 7. Base de datos MySQL (Hostinger)

21 tablas, InnoDB, `utf8mb4_unicode_ci`. Fuente: dump `Docs/u327033516_Inversionistas.sql`. No hay un archivo de esquema DDL versionado en el repo aparte de este dump — trátalo como la referencia autoritativa del esquema.

### 7.1 Tablas de negocio (núcleo)

| Tabla | Propósito | PK/UUID |
|---|---|---|
| `empresas` | Clientes que solicitan crédito (razón social, NIT) | `id` + `uuid BINARY(16)` |
| `representantes` | Representante legal de cada empresa | `id` + `uuid BINARY(16)`, FK `empresa_id` |
| `solicitudes` | **Tabla núcleo** — cada fila es una solicitud de crédito completa (monto, distribución por hotel en JSON, estado, firma/biometría) | `id` + `uuid BINARY(16)` + `codigo_solicitud` (ambos UNIQUE) |
| `solicitudes_contactos` | Contactos operativos por área (Tesorería, Cuentas por pagar, Dir. Financiero, Comercial) | FK `solicitud_id`, CASCADE |
| `solicitudes_facturacion` | Datos de facturación, 1:1 con `solicitudes` | FK `solicitud_id` UNIQUE, CASCADE |
| `solicitudes_referencias` | Referencias comerciales/hoteleras adicionales | FK `solicitud_id`, CASCADE |
| `documentos` | Documentos definitivos del expediente (Drive + hash SHA-256) | FK `solicitud_id` CASCADE, `tipo_documento_id` |
| `tipos_documento` | Catálogo de tipos de documento (RUT, Cert. Existencia...) | — |
| `estados_solicitud` | Catálogo del workflow (10 estados: BORRADOR, PENDIENTE_REVISION, EN_ANALISIS, OBSERVADA, APROBADA*, RECHAZADA*, VENCIDO*, BLOQUEADO_CORPORATIVO*, BLOQUEADO_GERENCIA*, POR_VENCER*; *=terminal) | `codigo` UNIQUE |
| `historial_estados` | Auditoría específica de cambios de estado por solicitud | FK `solicitud_id` CASCADE |
| `hoteles` | Catálogo de propiedades OxoHotel (cupo mínimo, correo de reservas) | `codigo` UNIQUE — relación **lógica**, no FK, con `solicitudes.hotel_codigos` |
| `cartas_autorizacion` | Cartas de aprobación generadas (versionado, archivo en Drive) | FK `solicitud_id` CASCADE |
| `creditos_vigentes` | Reporte desnormalizado de créditos vigentes (FKs nulleables SET NULL) | — |
| `analisis_ia` | Resultado de la auditoría documental con Gemini | `uuid VARCHAR` (inconsistente vs. resto en `BINARY`) |
| `usuarios` | Usuarios internos (staff), roles `ADMIN/ANALISTA/GERENTE_CUENTA/SOLO_LECTURA/RESERVAS` | `correo` UNIQUE |
| `sesiones` | Sesiones por portal (`PANEL/RESERVAS/INDICE`) — **existe en el esquema pero el código revisado usa `CacheService`, no esta tabla**, para sesiones activas | FK `usuario_id` CASCADE |
| `auditoria` | Log de auditoría genérico/polimórfico (`entidad`+`entidad_id`), con snapshots antes/después en JSON | FK `usuario_id` SET NULL |
| `notificaciones_leidas` | Estado leído/no-leído por usuario y solicitud | PK compuesta `(usuario_correo, solicitud_uuid)` |

### 7.2 Tablas de staging (importación masiva / integración)

| Tabla | Propósito |
|---|---|
| `stg_solicitudes` | Landing zone del importador masivo: espejo casi 1:1 de `solicitudes`, con `contactos_json`/`referencias_json` en vez de tablas hijas, bloqueo optimista (`bloqueado_por`/`bloqueado_en`) y máquina `estado_sync` (`PENDIENTE/SINCRONIZANDO/SINCRONIZADO/ERROR`) |
| `stg_documentos` | Documentos en tránsito hacia `documentos`, mismo patrón `estado_sync`/`intentos`/`ultimo_error` |
| `stg_ejecuciones` | Bitácora de corridas del job de importación masiva (contadores de éxito/error) |

**Flujo de staging→producción**: un proceso (el trigger `syncQueueToMySQL` del Formulario, y la ruta equivalente de importación masiva del Panel) lee `stg_solicitudes` con `estado_sync='PENDIENTE'`, toma el lock, crea/actualiza `empresas`/`representantes`/`solicitudes`, expande los JSON a las tablas hijas normalizadas, promueve `stg_documentos`→`documentos`, y marca `SINCRONIZADO`.

### 7.3 Notas para quien vaya a tocar el esquema

- **Inconsistencia de tipo en UUIDs**: la mayoría usa `BINARY(16)`, pero `analisis_ia.uuid`, `notificaciones_leidas.solicitud_uuid`, `stg_documentos.solicitud_uuid` y `stg_solicitudes.uuid` son `VARCHAR`. Cualquier JOIN cruzado entre estas tablas requiere convertir formato (`HEX()`/`UNHEX()` vs. comparación de string).
- Varias relaciones son **lógicas, no FKs declaradas** (por diseño, para desacoplar staging/integraciones externas): `hoteles.codigo` ↔ `solicitudes.hotel_codigos` (texto, una solicitud puede tener varios hoteles), `analisis_ia.codigo_solicitud` ↔ `solicitudes.codigo_solicitud`, `auditoria.entidad`+`entidad_id` (polimórfica hacia cualquier tabla).
- Columnas `*_json` (`cupos_distribuidos_json`, `contactos_json`, `referencias_json`) son `LONGTEXT` con `CHECK(json_valid(...))` (sintaxis MariaDB) en vez del tipo nativo `JSON` — tenlo presente si se migra de motor/versión.
- `creditos_vigentes` existe en el esquema con sus propias FKs (`SET NULL`), pero el Panel de Control actual **calcula los créditos como vista virtual sobre `solicitudes`**, no sobre esta tabla — confirmar con el equipo el estado real de esta tabla antes de asumir cuál es la fuente activa.

---

## 8. Guía de despliegue

1. Cada carpeta de módulo tiene su propio `.clasp.json` (Script ID) y `appsscript.json` (manifiesto). Usa `clasp push` desde dentro de cada carpeta para subir cambios al proyecto de Apps Script correspondiente — Apps Script no tiene subcarpetas reales, así que archivos organizados en subcarpetas locales (`docs/PdfGenerator.js`) se suben como archivos planos en la raíz del proyecto online.
2. Tras un `clasp push`, hay que crear manualmente una **nueva implementación (deployment)** desde el editor de Apps Script (o `clasp deploy`) para que los cambios lleguen a la URL pública del webapp — pushear código no actualiza automáticamente una implementación ya publicada salvo que sea la implementación "HEAD"/de prueba.
3. **Triggers que deben configurarse manualmente** (no están en el código, ver §5.4): `syncQueueToMySQL` (Formulario, cada 1 min — instalar ejecutando `setupTriggerMinuto()`), `cronEjecutarBackupDiario` y `cronAlertaVencimientos` (Panel, diarios — configurar desde el editor, Triggers → Add Trigger). Verificar periódicamente que sigan existiendo tras cualquier copia/migración del proyecto.
4. **Script Properties**: no se versionan con `clasp` (viven en la config del proyecto en la nube). Al clonar/migrar un proyecto a otra cuenta, hay que volver a configurarlas todas manualmente — usa las tablas de cada sección de este manual como checklist.
5. Si cambia el Script ID de despliegue de Formulario, Panel o Reservas (por ejemplo al recrear el proyecto desde cero), **actualizar `FORMULARIO_URL`/`PANEL_URL`/`RESERVAS_URL` en el proyecto Índice** — es el único lugar donde se enlazan entre sí.

---

## 9. Resumen de deuda técnica y hallazgos (índice rápido)

| # | Hallazgo | Módulo | Severidad sugerida |
|---|---|---|---|
| 1 | `doGet` ejecuta `ALTER TABLE` en cada carga de página sin flag de protección | Panel | Media (rendimiento) |
| 2 | Regla "vencimiento a 365 días" triplicada (JS×2 + SQL) | Panel | Media (mantenibilidad) |
| 3 | Triggers de cron no versionados en código (backup diario, alertas, sync) | Formulario / Panel | Alta (riesgo silencioso) |
| 4 | API key de Gemini con fallback hardcodeado en el código fuente | Panel | Alta (seguridad) |
| 5 | Contraseñas SHA-256 sin sal | Panel / Reservas | Alta (seguridad) |
| 6 | Credenciales de MySQL en texto plano en hoja compartida temporal | Panel → Reservas | Alta (seguridad) |
| 7 | Funciones de diagnóstico/hotfix sin control de acceso en producción | Panel | Media (seguridad) |
| 8 | Límite de 30MB de archivo solo validado en cliente | Formulario | Media (seguridad/robustez) |
| 9 | Código OTP reutilizable tras validación exitosa | Formulario | Baja-media (seguridad) |
| 10 | Dos generadores de PDF independientes que ya divergen visualmente | Formulario | Media (mantenibilidad) |
| 11 | Modo oscuro no persiste entre recargas (bug de inicialización) | Panel | Baja (UX) |
| 12 | Catálogo "Monedas" no conectado al Formulario (siempre COP) | Formulario / Panel | Baja (funcional) |
| 13 | Correos de prueba/placeholder repetidos en catálogos (Ejecutivos, Reservas de hoteles) | Panel | Baja (calidad de datos) — validar con el equipo |
| 14 | Emails hardcodeados de fallback en lógica de aprobación | Panel | Media (correctitud de negocio) |

---

## 10. Dónde seguir leyendo

- Manuales de usuario (HTML, con capturas de pantalla) para cada perfil: `Manuales/manual-solicitante.html`, `Manuales/manual-credito-cartera.html`, `Manuales/manual-reservas.html` — útiles para entender el comportamiento *esperado* desde la perspectiva del usuario antes de tocar el código correspondiente.
- El dump SQL completo: `Docs/u327033516_Inversionistas.sql` (estructura + posibles datos de ejemplo).
- `Docs/Configuración_formulario_panel.xlsx` y `Docs/Solicitud de créditos.xlsx` — no se inspeccionaron en profundidad en esta revisión (requieren un lector de `.xlsx` no disponible en este entorno); revisarlos manualmente si necesitas el detalle exacto de columnas de los catálogos de configuración.
