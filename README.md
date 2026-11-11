# Práctica 2: El Protocolo Model Context Protocol (MCP) a Bajo Nivel

## 1. Descripción

Esta práctica profundiza en el funcionamiento interno del **Model Context Protocol (MCP)** implementando y analizando la comunicación cliente-servidor a bajo nivel mediante el transporte estándar `stdio` y mensajes formateados bajo la especificación **JSON-RPC 2.0**.

A través de esta práctica se explora cómo un cliente se conecta directamente a un servidor MCP (`@modelcontextprotocol/server-filesystem`), gestiona el ciclo de vida del protocolo (fases de negociación, inicialización y notificación), lista herramientas, invoca ejecuciones y maneja las distintas categorías de respuestas y errores que el estándar contempla.

* **Materia:** Desarrollo de Agentes Inteligentes (ACD-2504).
* **Entorno operativo:** Windows 11 (x64) con Node.js v24.22.1 y npm/npx v11.19.1.
* **Servidor MCP evaluado:** `@modelcontextprotocol/server-filesystem` (v2.3.0).
* **Transporte:** `stdio` (comunicación interproceso mediante entrada y salida estándar en formato JSON delimitado por saltos de línea).

---

## 2. Estructura del Proyecto

```text
practica2-mcp/
├── cliente_mcp.mjs          # Cliente MCP mínimo en Node.js sin dependencias externas
├── mcp.json                 # Configuración del servidor filesystem para clientes compatibles
├── README.md                # Documentación técnica y análisis comparativo de errores
├── datos/
│   └── calificaciones.csv   # Dataset de prueba para lectura de archivos
├── evidencias/              # Capturas de pantalla de la inspección con MCP Inspector (Actividad A)
│   ├── Captura de pantalla 2026-11-11 094738.png
│   ├── Captura de pantalla 2026-11-11 095019.png
│   ├── Captura de pantalla 2026-11-11 095444.png
│   ├── Captura de pantalla 2026-11-11 095513.png
│   └── Captura de pantalla 2026-11-11 095828.png
└── salida/
    ├── historial_inspector.json # Registro de mensajes exportado desde MCP Inspector
    └── traza.jsonl              # Traza completa de la conversación generada por cliente_mcp.mjs (13 mensajes)
```

---

## 3. Arquitectura del Cliente Mínimo (`cliente_mcp.mjs`)

El cliente implementado en `cliente_mcp.mjs` no requiere librerías externas y opera directamente sobre los módulos nativos de Node.js (`child_process`, `readline`, `fs`):

1. **Lanzamiento del servidor (`spawn`):** Inicia el servidor `npx -y @modelcontextprotocol/server-filesystem .` como proceso hijo con canales de entrada y salida tubería (`stdio: ['pipe', 'pipe', 'inherit']`).
2. **Serialización y envío JSON-RPC:**
   - `solicitar(method, params)`: Asigna un identificador incremental `id`, envía el mensaje con `"jsonrpc": "2.0"` a `stdin` y guarda una promesa en un mapa de pendientes.
   - `notificar(method)`: Envía un mensaje unidireccional sin campo `id`, el cual no espera respuesta del servidor.
3. **Recepción y procesamiento asíncrono:** Lee la salida `stdout` línea por línea mediante `readline`, deserializa el JSON, lo registra en `salida/traza.jsonl` y resuelve la promesa correspondiente al `id` del mensaje.
4. **Cierre de conexión:** Envía `EOF` a la entrada del servidor (`servidor.stdin.end()`), finalizando ordenadamente el subproceso.

---

## 4. Actividad C: Análisis de Respuestas y Diferenciación de Errores

En la **Actividad C**, se modificó la conversación del cliente agregando tres solicitudes antes del cierre del servidor (paso 5):
1. **Lectura correcta:** Llamada a `tools/call` con la herramienta `read_text_file` sobre `datos/calificaciones.csv`.
2. **Lectura prohibida:** Llamada a `tools/call` con `read_text_file` sobre una ruta externa al directorio delimitado (`C:/Windows/win.ini`).
3. **Método inexistente:** Envío de una solicitud con un método JSON-RPC no registrado en el servidor (`herramientas/inventada`).

La ejecución del cliente produjo un total de **13 mensajes** en `salida/traza.jsonl`. A continuación se analizan en detalle las dos solicitudes que fallaron.

---

### 4.1 Respuestas Fallidas (Copiadas Textualmente de la Traza)

#### Respuesta 1: Lectura fuera del directorio permitido (ID 5)

* **Solicitud enviada (Línea 10 de `salida/traza.jsonl`):**
```json
{"direccion":"CLIENTE -> SERVIDOR","mensaje":{"jsonrpc":"2.0","id":5,"method":"tools/call","params":{"name":"read_text_file","arguments":{"path":"C:/Windows/win.ini"}}}}
```

* **Respuesta recibida del servidor (Línea 11 de `salida/traza.jsonl`):**
```json
{"direccion":"SERVIDOR -> CLIENTE","mensaje":{"result":{"content":[{"type":"text","text":"Access denied - path outside allowed directories: C:\\Windows\\win.ini not in C:\\AgentesInt\\practica2-mcp"}],"isError":true},"jsonrpc":"2.0","id":5}}
```

En formato legible:
```json
{
  "jsonrpc": "2.0",
  "id": 5,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Access denied - path outside allowed directories: C:\\Windows\\win.ini not in C:\\AgentesInt\\practica2-mcp"
      }
    ],
    "isError": true
  }
}
```

---

#### Respuesta 2: Método RPC no existente (ID 6)

* **Solicitud enviada (Línea 12 de `salida/traza.jsonl`):**
```json
{"direccion":"CLIENTE -> SERVIDOR","mensaje":{"jsonrpc":"2.0","id":6,"method":"herramientas/inventada","params":{}}}
```

* **Respuesta recibida del servidor (Línea 13 de `salida/traza.jsonl`):**
```json
{"direccion":"SERVIDOR -> CLIENTE","mensaje":{"jsonrpc":"2.0","id":6,"error":{"code":-32601,"message":"Method not found"}}}
```

En formato legible:
```json
{
  "jsonrpc": "2.0",
  "id": 6,
  "error": {
    "code": -32601,
    "message": "Method not found"
  }
}
```

---

### 4.2 Explicación y Diferenciación Técnica de los Errores

| Criterio | Solicitud 2 (Lectura prohibida - ID 5) | Solicitud 3 (Método inventado - ID 6) |
| :--- | :--- | :--- |
| **Categoría del error** | **Error de ejecución de herramienta** (*Tool Execution Error*) | **Error de protocolo** (*Protocol / JSON-RPC Error*) |
| **Capa donde ocurre** | **Capa de aplicación / lógica de negocio** del servidor MCP | **Capa de transporte / enrutamiento** JSON-RPC 2.0 |
| **Estado a nivel JSON-RPC** | **Éxito (200 OK equivalente)**: La transacción RPC se completó normalmente | **Fallo**: La transacción RPC no se pudo resolver ni enrutar |
| **Estructura raíz de la respuesta** | Contiene el objeto `"result"` (no existe `"error"`) | Contiene el objeto `"error"` (no existe `"result"`) |
| **Campos distintivos** | `"isError": true` y `"content"` dentro de `"result"` | `"code": -32601` y `"message": "Method not found"` dentro de `"error"` |
| **Causa del fallo** | La herramienta `read_text_file` validó que la ruta solicitada (`C:/Windows/win.ini`) viola la política de contención (*allowed directories*) del servidor | El despachador RPC del servidor no tiene ninguna función o manejador registrado para el método `herramientas/inventada` |
| **Significado para un agente inteligente** | El agente puede leer el texto en `content` para comprender por qué falló su acción e intentar una ruta válida o una alternativa | El cliente/agente envió un comando inválido que rompe la especificación del protocolo; debe corregir el método invocado |

---

### 4.3 Análisis de los Campos de Reconocimiento

#### ¿En qué campos se reconoció el Error de Ejecución de Herramienta (ID 5)?
1. **Presencia de `"result"` y ausencia de `"error"` a nivel raíz:** Para el protocolo JSON-RPC, el servidor procesó exitosamente la solicitud. La llamada al método `tools/call` fue recibida, comprendida y atendida.
2. **Campo `"isError": true` dentro de `"result"`:** Es el estándar definido por la especificación de MCP para indicar que la herramienta no pudo cumplir su objetivo. A diferencia de una excepción que tumbaría el canal de comunicación, MCP modela los fallos de herramientas como respuestas válidas que informan el error al cliente o modelo.
3. **Campo `"content"` dentro de `"result"`:** Alberga un bloque de tipo `"text"` con el mensaje `"Access denied - path outside allowed directories: C:\\Windows\\win.ini not in C:\\AgentesInt\\practica2-mcp"`. Este contenido permite a un modelo de lenguaje o agente razonar sobre el motivo del rechazo sin que falle la infraestructura de comunicación.

#### ¿En qué campos se reconoció el Error de Protocolo (ID 6)?
1. **Presencia de `"error"` a nivel raíz:** La respuesta carece por completo del campo `"result"`. La especificación JSON-RPC 2.0 estipula que una respuesta debe contener exclusivamente `"result"` o `"error"`, nunca ambos.
2. **Campo `"error.code": -32601`:** Es el código numérico reservado estándar en la especificación JSON-RPC 2.0 para `Method not found` (Método no encontrado). Indica un fallo en el protocolo de comunicación antes de que cualquier lógica de negocio o herramienta sea invocada.
3. **Campo `"error.message": "Method not found"`:** Cadena textual estándar que documenta el fallo sintáctico o semántico de la solicitud RPC enviada por el cliente.

---

## 5. Reproducción y Ejecución

Para ejecutar el cliente y regenerar la traza:

```bash
node cliente_mcp.mjs
```

El resultado en consola mostrará el intercambio completo de los 13 mensajes, guardándose simultáneamente en `salida/traza.jsonl`.
