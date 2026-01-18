# Información sobre el Flujo de Errores de Formato en Tools - OpenCode

## Resumen del Flujo

Cuando una tool falla por validación de formato, el sistema sigue este camino:

---

### 1. **Validación en `Tool.define()`**

**Archivo:** `packages/opencode/src/tool/tool.ts:56-67`

```ts
toolInfo.execute = async (args, ctx) => {
  try {
    toolInfo.parameters.parse(args) // ← Validación Zod
  } catch (error) {
    if (error instanceof z.ZodError && toolInfo.formatValidationError) {
      throw new Error(toolInfo.formatValidationError(error), { cause: error })
    }
    throw new Error(
      `The ${id} tool was called with invalid arguments: ${error}.\nPlease rewrite the input so it satisfies the expected schema.`,
      { cause: error },
    )
  }
  // ... continúa ejecución
}
```

---

### 2. **Personalización de Mensaje de Error (Opcional)**

**Archivo:** `packages/opencode/src/tool/batch.ts:22-31`

```ts
formatValidationError(error) {
  const formattedErrors = error.issues
    .map((issue) => {
      const path = issue.path.length > 0 ? issue.path.join(".") : "root"
      return `  - ${path}: ${issue.message}`
    })
    .join("\n")

  return `Invalid parameters for tool 'batch':\n${formattedErrors}\n\nExpected payload format:\n  [{"tool": "tool_name", "parameters": {...}}, {...}]`
}
```

---

### 3. **Captura en el Procesador**

**Archivo:** `packages/opencode/src/session/processor.ts:196-221`

```ts
case "tool-error": {
  const match = toolcalls[value.toolCallId]
  if (match && match.state.status === "running") {
    await Session.updatePart({
      ...match,
      state: {
        status: "error",           // ← Estado establecido a error
        input: value.input,
        error: (value.error as any).toString(),
        time: {
          start: match.state.time.start,
          end: Date.now(),
        },
      },
    })
    // ... manejo de permisos rechazados
  }
  break
}
```

---

### 4. **Conversión a Formato para Modelo**

**Archivo:** `packages/opencode/src/session/message-v2.ts:533-541`

```ts
if (part.state.status === "error")
  assistantMessage.parts.push({
    type: ("tool-" + part.tool) as `tool-${string}`,
    state: "output-error", // ← Estado para el modelo
    toolCallId: part.callID,
    input: part.state.input,
    errorText: part.state.error, // ← Mensaje de error
    callProviderMetadata: part.metadata,
  })
```

---

### 5. **Estados de ToolState**

**Archivo:** `packages/opencode/src/session/message-v2.ts:266-283`

```ts
export const ToolStateError = z.object({
  status: z.literal("error"),
  input: z.record(z.string(), z.any()),
  error: z.string(), // ← El mensaje de error
  metadata: z.record(z.string(), z.any()).optional(),
  time: z.object({
    start: z.number(),
    end: z.number(),
  }),
})
```

---

### 6. **Publicación de Evento (Opcional)**

**Archivo:** `packages/opencode/src/session/processor.ts:359`

```ts
input.assistantMessage.error = error
Bus.publish(Session.Event.Error, {
  sessionID: input.assistantMessage.sessionID,
  error: input.assistantMessage.error,
})
```

---

## Notificaciones cuando una Tool falla por formato

Cuando una tool falla por validación de formato, se publican los siguientes eventos:

---

### 1. **`MessageV2.Event.PartUpdated`**

**Archivo:** `packages/opencode/src/session/index.ts:401-409`

```ts
export const updatePart = fn(UpdatePartInput, async (input) => {
  const part = "delta" in input ? input.part : input
  const delta = "delta" in input ? input.delta : undefined
  await Storage.write(["part", part.messageID, part.id], part)
  Bus.publish(MessageV2.Event.PartUpdated, {
    part,
    delta,
  })
  return part
})
```

Este evento se publica **siempre** que se actualiza cualquier part, incluyendo tools con error.

**Contenido del evento:**

```ts
{
  type: "message.part.updated",
  properties: {
    part: ToolPart,  // ← Tiene state.status === "error" y state.error
    delta?: string
  }
}
```

---

### 2. **`Session.Event.Error`** (en algunos casos)

**Archivo:** `packages/opencode/src/session/processor.ts:359`

```ts
input.assistantMessage.error = error
Bus.publish(Session.Event.Error, {
  sessionID: input.assistantMessage.sessionID,
  error: input.assistantMessage.error,
})
```

Este evento se publica cuando:

- Hay un error irrecuperable en el processor
- El error no puede ser reintentado

**Contenido del evento:**

```ts
{
  type: "session.error",
  properties: {
    sessionID: string
    error: { name: string; message: string; ... }
  }
}
```

---

### 3. **`Session.Event.ToolUnavailable`** (NUEVO)

**Archivo:** `packages/opencode/src/session/processor.ts:171-184`

```ts
if (value.toolName === "invalid") {
  try {
    const invalidInput = typeof value.input === "string" ? JSON.parse(value.input) : value.input
    if (invalidInput.tool && invalidInput.error) {
      await Bus.publish(Session.Event.ToolUnavailable, {
        sessionID: input.sessionID,
        toolName: invalidInput.tool,
        error: invalidInput.error,
      })
    }
  } catch {
    // ignore parsing errors
  }
}
```

Este evento se publica cuando:

- El modelo intenta llamar a una tool que no está disponible
- El SDK de ai-sdk convierte el tool call a "invalid" con los detalles del error
- Ocurre en `experimental_repairToolCall` en `session/llm.ts:183-190`

**Contenido del evento:**

```ts
{
  type: "session.tool_unavailable",
  properties: {
    sessionID: string
    toolName: string  // ej: "write"
    error: string     // ej: "Model tried to call unavailable tool 'write'"
  }
}
```

**Definición del evento:**

```ts
// En session/index.ts
ToolUnavailable: BusEvent.define(
  "session.tool_unavailable",
  z.object({
    sessionID: z.string(),
    toolName: z.string(),
    error: z.string(),
  }),
),
```

---

## Diagrama del Flujo

```
┌─────────────────┐
│  LLM responde   │
│  con tool call  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Ejecuta tool   │
│  parameters.parse│
└────────┬────────┘
         │
    ¿ZodError?
         │
    ┌────┴────┐
    │         │
   SÍ         NO
    │         │
    ▼         ▼
┌────────┐  ┌─────────────────┐
│ format-│  │ Continúa        │
│Validation│  │ ejecución       │
│Error?   │  └─────────────────┘
└────┬────┘
     │
    SÍ
     │
     ▼
┌─────────────────────────────────┐
│ Lanza Error con mensaje         │
│ personalizado o genérico        │
└────────┬────────────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│ processor.ts captura "tool-error"│
│ y actualiza part con status:    │
│ "error" y error: "mensaje"      │
└────────┬────────────────────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌─────────┐  ┌─────────────────────┐
│Session. │  │ MessageV2.          │
│Event.Error│ │ Event.PartUpdated  │ ← SIEMPRE
│(opcional)│  │ (estado "error")    │
└─────────┘  └─────────────────────┘
```

---

## Cómo suscribirse a estos eventos

```ts
import { Bus } from "@/bus"
import { MessageV2, Session } from "@/session"

// Para errores de tool específicos
Bus.subscribe(MessageV2.Event.PartUpdated, (event) => {
  if (event.part.type === "tool" && event.part.state.status === "error") {
    console.log("Tool falló:", event.part.tool)
    console.log("Error:", event.part.state.error)
  }
})

// Para errores de sesión generales
Bus.subscribe(Session.Event.Error, (event) => {
  console.log("Error en sesión:", event.sessionID)
  console.log("Detalles:", event.error)
})

// Para tools no disponibles (NUEVO)
Bus.subscribe(Session.Event.ToolUnavailable, (event) => {
  console.log("Tool no disponible:", event.toolName)
  console.log("Error:", event.error)
  console.log("Sesión:", event.sessionID)
})
```

---

## Archivos relevantes

| Archivo                 | Línea   | Descripción                                           |
| ----------------------- | ------- | ----------------------------------------------------- |
| `tool/tool.ts`          | 56-67   | Validación y lanzamiento de errores                   |
| `tool/batch.ts`         | 22-31   | Personalización de mensaje de error                   |
| `session/processor.ts`  | 196-221 | Manejo de "tool-error"                                |
| `session/processor.ts`  | 359     | Publicación de Session.Event.Error                    |
| `session/processor.ts`  | 171-184 | Publicación de ToolUnavailable (NUEVO)                |
| `session/index.ts`      | 401-409 | `updatePart` publica `PartUpdated`                    |
| `session/index.ts`      | 349-355 | `updateMessage` publica `Updated`                     |
| `session/index.ts`      | 121-127 | Definición de `Session.Event.Error`                   |
| `session/index.ts`      | 128-136 | Definición de `Session.Event.ToolUnavailable` (NUEVO) |
| `session/llm.ts`        | 183-190 | `experimental_repairToolCall` convierte a "invalid"   |
| `session/message-v2.ts` | 266-283 | Definición de `ToolStateError`                        |
| `session/message-v2.ts` | 398-427 | Definición de eventos de message                      |
| `session/message-v2.ts` | 533-541 | Conversión a formato para el modelo                   |
