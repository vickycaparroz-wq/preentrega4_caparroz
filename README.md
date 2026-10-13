[README_1.md](https://github.com/user-attachments/files/33408722/README_1.md)
# IT Support Triage - Checkpoint 4: Integraciones Avanzadas

Proyecto integrador del curso de n8n (Coderhouse). Este checkpoint evoluciona el workflow del Módulo 3 (agente Manager con workers especializados, memoria y Google Sheets) y le suma integraciones con herramientas externas: Gmail, HubSpot y Slack.

- Workflow: `IT Support Triage - Checkpoint 4 - Advanced Integrations`
- Archivo: `checkpoint4_maria_victoria_caparroz.json` (exportado desde n8n con Workflow > Download)
- Estado: workflow guardado e **inactivo** (no publicado)

## Captura de credenciales

Sección **Credentials** de n8n con las conexiones usadas por el workflow: HubSpot (`HubSpot account`, OAuth2), Gmail (`Gmail account`, OAuth2) y Slack (`Slack account`, Slack API). Solo se muestran los nombres; no se expone ningún token ni secreto.

![Credentials de n8n con HubSpot, Gmail y Slack](./credentials.png)

## Arquitectura

El workflow conserva todo lo del Módulo 3 (Agente Principal, workers Access y Technical, memoria por `Session_ID`, registro de tickets en Google Sheets) y agrega estas piezas:

1. **Entrada por Gmail**: `Gmail Trigger (solicitudes de soporte)` → `Filtrar y normalizar correo` → `¿Ignorar correo?`.
2. **Contacto en el CRM**: `Preparar contacto HubSpot` → `¿Email válido para HubSpot?` → `Buscar contacto en HubSpot` → `Resolver contacto HubSpot` → `¿Contacto ya existe?` → `Actualizar contacto HubSpot` o `Crear contacto HubSpot` → `Consolidar resultado HubSpot`.
3. **Salida por Gmail**: `Notificación de Observabilidad (Gmail)` y `Borrador de respuesta (Gmail)`, ambos con la operación *Create Draft*.
4. **Slack**: `Preparar mensaje Slack` → `¿Escalar a Slack?` → `Slack: Escalar incidencia` (rama true) y `Slack: Notificar ticket` (rama false).

## Capturas del workflow por secciones

Capturas tomadas del editor de n8n con el workflow guardado e **inactivo**. Los nodos de Slack aparecen como *Deactivated* a propósito.

### 1. Gmail Trigger y filtros

Entrada por Gmail: `Gmail Trigger (solicitudes de soporte)` → `Filtrar y normalizar correo` → `¿Ignorar correo?`. Las autorrespuestas y correos no válidos terminan en `Correo ignorado`, y el resto sigue al flujo principal.

![Gmail Trigger y filtros anti auto-reply](./screenshots/03_gmail_trigger_filtros.png)

### 2. Integración con HubSpot

Cadena del CRM: `Preparar contacto HubSpot` → `¿Email válido para HubSpot?` → `Buscar contacto en HubSpot` → `Resolver contacto HubSpot` → `¿Contacto ya existe?` → `Actualizar` o `Crear contacto HubSpot` → `Consolidar resultado HubSpot`. Siempre se busca antes de crear para evitar duplicados.

![Integración con HubSpot: búsqueda, validación y create/update](./screenshots/04_hubspot_integracion.png)

### 3. Gmail observabilidad

`Notificación de Observabilidad (Gmail)` recibe el resultado desde `Estado del Ticket` y crea un **borrador** (operación *Create Draft*), por lo que nada se envía sin revisión humana.

![Gmail observabilidad: borrador de notificación](./screenshots/05_gmail_observabilidad.png)

### 4. Slack: integración y conexiones

`Estado del Ticket` alimenta en paralelo a `Preparar contacto HubSpot` y a `Preparar mensaje Slack`. Luego `¿Escalar a Slack?` deriva a `Slack: Escalar incidencia` (true) o `Slack: Notificar ticket` (false). Los 4 nodos de Slack están deshabilitados hasta autorizar pruebas en `#it-triage-test`.

![Rama de Slack conectada y deshabilitada](./screenshots/06_slack_integracion.png)

## Controles implementados

| Control | Dónde | Qué evita |
|---|---|---|
| Filtro anti auto-reply | `Filtrar y normalizar correo` + `¿Ignorar correo?` | Bucle infinito de respuestas automáticas (Auto-reply, Out of office, Undeliverable, casillas `no-reply@`). Los correos ignorados terminan en `Correo ignorado`. |
| Validación de email | `Preparar contacto HubSpot` + `¿Email válido para HubSpot?` | Payload mal formado hacia el CRM (error 400). Si el email no es válido, no se llama a HubSpot. |
| Look up antes del Create | `Buscar contacto en HubSpot` + `¿Contacto ya existe?` | Duplicados de contactos (error 409). Si existe se actualiza; si no, se crea. Ante duplicados se usa el más antiguo. |
| Human-in-the-loop | `Borrador de respuesta (Gmail)` y `Notificación de Observabilidad (Gmail)` | Solo operación *Create Draft*: nada se envía sin revisión humana. |
| Limpieza de payload antes de Slack | `Preparar mensaje Slack` | Deja solo el texto necesario para el canal, sin objetos pesados ni binarios. |
| Manejo de errores | Nodos HubSpot con `continueRegularOutput` | Un fallo del CRM no detiene el resto del flujo. |

## Permisos y mínimo privilegio

- HubSpot: se usa una sola credencial, solo con las operaciones de búsqueda y upsert de contactos, contra el portal académico de pruebas. No se toca ningún portal productivo.
- Gmail: lectura de la casilla de soporte y creación de borradores. No se usa la operación de envío.
- Slack: el token se recomienda con el permiso mínimo `chat:write`. Los nodos permanecen **deshabilitados** hasta autorizar pruebas.

## Pruebas realizadas y límites

Lo que se probó y lo que no, de forma transparente:

- **Pruebas emuladas en el navegador (no son ejecuciones del motor de n8n):** 11 casos. Filtro de auto-reply 5/5, validación de email 3/3, deduplicación 3/3.
- **Verificación real contra HubSpot:** consulta de datos de la cuenta que confirmó el portal académico de pruebas (tipo `DEVELOPER_TEST`), con ejecución exitosa en n8n.
- **No completado:** la búsqueda aislada de contacto en HubSpot con *Test step*. La interfaz abrió el panel de chat y quedó esperando al Chat Trigger, así que se detuvo para no ejecutar el workflow completo. No se creó ni modificó ningún contacto.
- **No ejecutado:** pruebas de extremo a extremo y envío real a Slack. La credencial de Slack se configuró recién al final y los nodos siguen deshabilitados.
- **Conexión a Slack:** `Estado del Ticket` alimenta en paralelo a `Preparar contacto HubSpot` y a `Preparar mensaje Slack` (rama HubSpot intacta). Los 4 nodos de Slack siguen deshabilitados.
- **Pendiente:** pruebas con *Test step* una vez autorizado el canal `#it-triage-test`.

## Seguridad

- Ningún token ni contraseña está en el JSON; las credenciales se referencian por nombre.
- El workflow no está publicado: no se dispara solo.
- Los borradores se crean en Gmail y nunca se envían automáticamente.

## Cómo importarlo

1. En n8n: Workflows > Import from file > `checkpoint4_maria_victoria_caparroz.json`.
2. Asignar tus propias credenciales (HubSpot, Gmail, Slack) en los nodos correspondientes.
3. Revisar el destinatario de los borradores de observabilidad (nodo `Notificación de Observabilidad (Gmail)`), que apunta a un correo de pruebas.
