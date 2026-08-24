# Checkpoint de Integraciones Avanzadas — Sistema de Triage Médico

## Adaptación del caso de la clase en vivo

La consigna plantea el ecosistema e-commerce (tienda → CRM → Gmail → Slack). Este proyecto aplica los mismos conceptos a un dominio distinto: un **sistema de triage médico** que recibe consultas de pacientes por chat, las clasifica por nivel de gravedad con un LLM, y las deriva a un flujo de aprobación humana según corresponda.

El mapeo de roles es directo:

| Rol en la consigna | Herramienta usada | Rol en este proyecto |
|---|---|---|
| CRM / fuente de verdad del cliente | **Pipedrive** | Ficha de paciente (nombre, ID, historial de consultas) |
| Casilla de soporte | **Gmail** | Derivación de casos a la guardia médica, con aprobación por link |
| Canal del equipo | **Slack** | Notificación al equipo de operaciones en casos de gravedad ALTA |

No se usó HubSpot ni Salesforce como CRM porque ambos requerían un proceso de registro de aplicación (Client ID/Secret vía consola de desarrollador) demasiado extenso para el alcance de este TP. Se optó por **Pipedrive**, que cumple el mismo rol de CRM con autenticación simple por Access Token, sin afectar el cumplimiento de los puntos de la rúbrica.

## Arquitectura general

```
[Chat del paciente]
        │
        ▼
[LLM — Triage y extracción de datos]
        │
        ▼
[Pipedrive — Search] → [IF: ¿existe el paciente?] → [Update / Create]
        │
        ▼
[Switch — Nivel de gravedad: ALTA / MEDIA / BAJA]
        │
   ┌────┼────┐
   ▼    ▼    ▼
Worker Worker Worker
 ALTA  MEDIA  BAJA
```

Cada Worker gestiona la derivación según el nivel de gravedad:

- **Worker 1 (ALTA):** notificación simultánea por Gmail (con botones de aprobar/rechazar) y Slack (canal `#operaciones-triage`), con espera de decisión humana (Wait, 15 min).
- **Worker 2 (MEDIA):** igual que ALTA, pero con una tercera opción de "Escalar a ALTA" si el médico decide que el caso es más grave de lo evaluado inicialmente.
- **Worker 3 (BAJA):** un LLM redacta una respuesta simple para el paciente, y se guarda como **borrador de Gmail** (no se envía automáticamente) para revisión humana antes de la emisión final.

Todos los casos, sea cual sea su resolución, quedan registrados en una hoja de Google Sheets como log de auditoría.

## Mapeo contra los 4 nodos clave de la rúbrica

### ① IF anti auto-reply
Implementado como una pieza demostrativa aislada: `Demo - Filtro Anti Auto-Reply.json`. Como el trigger principal del sistema es un chat (no una bandeja de correo entrante), este workflow separado muestra el mecanismo de forma autónoma: un `Gmail Trigger` conectado a un `IF` que evalúa, con combinador OR:

- `Subject` contiene "auto-reply"
- `Subject` contiene "out of office"
- `Subject` contiene "undeliverable"
- `From` contiene "no-reply@"

Probado en ambas ramas: un mail normal cae en `False` (sigue el flujo), y un mail con asunto "Auto-Reply: Fuera de la oficina" cae en `True` (corta el flujo, sin nodos conectados en esa salida).

### ② Look up antes del Create (evita duplicados)
Implementado en el **Manager**, antes del Switch de derivación:

`Search persons` (Pipedrive, busca por `id_paciente`) → `IF - Existe en CRM` → `Update person` (si existe) / `Create person` (si no existe).

Probado en ambos caminos con pacientes reales de prueba (creación de contacto nuevo y actualización de un contacto ya existente), confirmando que Pipedrive no genera duplicados.

### ③ Create Draft (Human-in-the-loop)
Implementado en el **Worker 3 (BAJA)**: el nodo de Gmail está configurado con `resource: draft`, por lo que la respuesta generada por el LLM se guarda como borrador en la bandeja de Gmail, sin enviarse automáticamente. Un humano debe revisarlo y enviarlo manualmente.

Para los niveles ALTA y MEDIA, el HITL se implementa con un mecanismo adicional (no excluyente): un nodo `Wait` que pausa la ejecución hasta que un humano hace clic en un link de aprobar/rechazar/escalar dentro del mail — un guardrail equivalente en espíritu, con decisión activa registrada.

### ④ Set de limpieza de payload
Implementado en el **Worker 1 (ALTA)**, antes de la notificación a Slack: el nodo `Set - Limpiar payload para Slack` reduce el objeto `triage` completo (que incluye el mensaje original del paciente y otros metadatos) a solo tres campos esenciales — síntoma, nivel de gravedad y nombre del paciente — antes de enviarlos al canal de Slack.

## Autenticación (OAuth2 / tokens)

- **Pipedrive:** credencial tipo API Token (Access Token), generado desde el perfil de usuario.
- **Gmail:** credencial OAuth2, autorizada contra la cuenta de Google.
- **Slack:** credencial tipo Access Token (Bot User OAuth Token), generado a través de una app de Slack instalada en el workspace, con scopes `chat:write` y `channels:read` (principio de menor privilegio: solo los permisos necesarios para leer canales y enviar mensajes).

## Archivos incluidos

| Archivo | Descripción |
|---|---|
| `Manager_Triage_Medico.json` | Workflow principal: recibe el chat, clasifica con LLM, gestiona el CRM, deriva según gravedad |
| `Worker1_Nivel_Grav_Alta.json` | Derivación urgente (Gmail + Slack + HITL) |
| `Worker2_Nivel_Grav_Media.json` | Derivación media (Gmail + HITL con escalamiento) |
| `Worker3_Nivel_Grav_Baja.json` | Respuesta automática con LLM, guardada como borrador (Create Draft) |
| `Demo_Filtro_Anti_AutoReply.json` | Pieza demostrativa aislada del filtro anti-loop |


