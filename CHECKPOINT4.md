# Checkpoint 4 — Sincronización con Ecosistemas de Negocio

Workflow de n8n que conecta **Gmail (soporte)**, **HubSpot (CRM)** y **Slack (operaciones)**
con guardrails de seguridad y control de duplicados y errores.

Autor: Franco Tomas Olmedo

---

## Flujo

| # | Nodo | Qué resuelve |
|---|------|--------------|
| 1 | `Trigger Email Soporte` (Gmail Trigger) | Entrada: email no leído en la casilla de soporte |
| 2 | `Guardrail Anti Auto-Reply` (IF) | **Anti bucle infinito** |
| 3 | `AI Agent Clasifica y Redacta` (+ OpenAI + Structured Parser) | Clasifica y redacta el borrador |
| 4 | `Set Limpieza Payload` | Deja solo `From` / `Subject` / `BodyText` + clasificación |
| 5 | `Filtro Email Valido` | **Evita el 400** |
| 6 | `Loop Un Email a la Vez` (batch = 1) | Aísla cada email |
| 7 | `HubSpot Lookup Contacto` (search by email) | **Evita el 409** |
| 8 | `Contacto Existe en CRM` (IF) → `Update` / `Create` | Rama condicional |
| 9 | `Gmail Create Draft` | **Human-in-the-loop** |
| 10 | `Set Payload Liviano Slack` → `Slack Notifica Operaciones` | Payload sin binarios, 1 sola notificación |

## Guardrails

**Anti auto-reply (nodo 2).** Corta el flujo si el asunto contiene `Auto-reply`,
`Automatic reply`, `Out of office`, `Undeliverable` o `Respuesta automatica`, o si el
remitente es `no-reply@`, `noreply@`, `mailer-daemon@` o `postmaster@`. La rama TRUE
va a un NoOp terminal: no se responde ni se escribe en el CRM. Sin esto, responder una
auto-respuesta genera otra auto-respuesta.

**Validación de payload (nodos 4 y 5).** El `Set` tiene *Include Other Input Fields* en
OFF, así que el item se reconstruye desde cero con los campos necesarios. Calcula
`EmailValido` (email parseable + cuerpo no vacío + borrador generado) y el `Filter`
descarta lo que no valida, antes de tocar HubSpot o Gmail.

**Look up antes de escribir (nodos 6, 7 y 8).** Se busca el contacto por email antes de
cualquier alta. El `Loop` con batch = 1 es necesario: si el trigger trae varios emails y
alguno no existe en el CRM, el nodo de búsqueda devuelve menos filas que las de entrada y
ese contacto se perdería sin crearse nunca.

**Human-in-the-loop (nodo 9).** Operación **Create Draft**, nunca *Send*. El borrador
queda en Gmail y una persona decide si se envía.

**Payload liviano antes de Slack (nodo 10).** *Include Other Input Fields* en OFF: en
Set v3.4, cuando ese flag es `false` n8n fuerza `include = "none"` y no copia la clave
`binary` al item de salida, así que los adjuntos no llegan a Slack. `Execute Once` emite
una única notificación agregada en vez de un mensaje por ticket.

---

## Autenticación y scopes

| Conector | Método | Scopes | Verificado end-to-end |
|----------|--------|--------|-----------------------|
| Gmail | OAuth2 | credencial nativa de n8n | Sí |
| HubSpot | Service Key | `crm.objects.contacts.read`, `crm.objects.contacts.write` | Sí |
| Slack | OAuth2 (user token) | `chat:write`, `channels:read` | No — ver *Estado de pruebas* |

### Dos desviaciones respecto de la consigna, y por qué

**1. HubSpot usa Service Key en lugar de OAuth2.**
HubSpot deshabilitó la creación de apps públicas (las que dan OAuth2 con Client ID y
Secret) desde la interfaz. El mensaje literal en el panel de developers es:

> *"La creación de nuevas aplicaciones públicas anteriores está deshabilitada. Ejecuta
> `hs project create` en la CLI de HubSpot para crear aplicaciones de OAuth en varias
> cuentas."*

La única vía a OAuth2 hoy es la plataforma de proyectos vía CLI. Se optó por una **clave
de servicio** con exactamente los dos scopes que el workflow usa. El resultado es
*más* restrictivo que OAuth2, no menos: ver el punto 2.

**2. Los scopes de OAuth2 en n8n no siempre son configurables.**
La credencial `hubspotOAuth2Api` de n8n define el scope como campo `hidden` y pide **13
scopes fijos** (contacts, companies, deals, owners, schemas, lists, forms y tickets),
sin manera de editarlos desde la UI. Es decir, la vía OAuth2 habría requerido conceder
11 permisos que el workflow no usa.

La credencial `slackOAuth2Api` sí expone un toggle **Custom Scopes**, y ahí se redujo la
lista por defecto de 22 user scopes a los 2 necesarios.

---

## Instalación

1. Importar `checkpoint4_francotomas_olmedo.json` en n8n (*Workflows → Import from File*).
2. Crear y asignar las credenciales: Gmail OAuth2, Slack OAuth2, HubSpot Service Key.
3. En `Slack Notifica Operaciones`, ajustar el canal (por defecto `#operaciones`).
4. En `Trigger Email Soporte`, ajustar el filtro `q` a la label o alias real de soporte.




---

## Estado de pruebas

Probado con ejecuciones reales sobre un email entrante de soporte.

| Nodo | Resultado |
|------|-----------|
| `Trigger Email Soporte` | Cuerpo completo del mail (157 chars con `simple: false`) |
| `Guardrail Anti Auto-Reply` | Email humano ruteado por la salida `false` |
| `AI Agent Clasifica y Redacta` | `categoria: envio`, `prioridad: alta`, `requiere_humano: true` |
| `Set Limpieza Payload` | 12 campos, `EmailValido: true` |
| `Filtro Email Valido` | Item aprobado |
| `HubSpot Lookup Contacto` | Item vacío en la primera corrida (contacto inexistente) |
| `Contacto Existe en CRM` → `Create` | Contacto `866745566409` creado, `isNew: true` |
| `Contacto Existe en CRM` → `Update` | Mismo `vid`, `isNew: false` — **sin duplicado, 409 evitado** |
| `Gmail Create Draft` | Borrador en el hilo original, `labelIds: ["DRAFT"]`, nunca enviado |
| `Set Payload Liviano Slack` | 4 campos agregados, sin binarios |
| `Slack Notifica Operaciones` | **No verificado** |

### Sobre el nodo de Slack

El nodo está configurado y la credencial OAuth2 figura como conectada, pero **no se
logró una publicación exitosa en el canal** dentro de las corridas registradas. El error
observado fue `Unable to sign without access token`.

La causa está identificada: el nodo de Slack pide el token en `authed_user.access_token`
(ver `Slack/V2/GenericFunctions.js`), o sea que requiere que la autorización conceda
**User Token Scopes**. Si la instalación de la app concede únicamente Bot Token Scopes,
esa propiedad no existe en la respuesta de Slack y el nodo no puede firmar la request.

El arreglo es reautorizar la credencial con `chat:write` y `channels:read` cargados en la
sección **User Token Scopes** de la app de Slack, y reinstalar la app antes de reconectar.

---

## Nota de implementación: `typeVersion`

Los `typeVersion` de este workflow están fijados a los que soporta n8n 2.21.7. Poner una
versión inexistente **no falla al guardar ni al validar**: falla recién al activar, con
`Cannot read properties of undefined (reading 'execute')`, sin indicar qué nodo. En esta
instancia, Slack llega hasta `2.4` y Set hasta `3.4`.
