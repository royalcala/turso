# Arquitectura Híbrida: Turso (Servidor) + LiveStore (Cliente) + Event Sourcing

## La Idea

Usar Turso como base de datos SQL en el servidor, y LiveStore (basado en event sourcing / CRDT) como state manager reactivo en el cliente, conectados por una capa de eventos que filtra por permisos.

```
┌───────────────────────────────────────────────────────────────────┐
│  SERVIDOR                                                          │
│                                                                   │
│  ┌──────────────┐     ┌────────────────────┐                      │
│  │ Turso DB      │────►│ Event Emitter       │                     │
│  │ (source of    │     │ (tras cada INSERT,   │                    │
│  │  truth)       │     │  UPDATE, DELETE en   │                    │
│  │               │     │  Turso, emite evento) │                   │
│  └──────────────┘     └─────────┬──────────┘                      │
│                                 │                                  │
│                    ┌────────────▼──────────┐                      │
│                    │ Permission Filter      │                     │
│                    │ (filtra eventos por    │                     │
│                    │  user_id, team, role)  │                     │
│                    └────────────┬──────────┘                      │
│                                 │                                  │
│  ┌──────────────────────────────▼───────────────────────────┐    │
│  │ Command Handler (recibe writes del cliente)               │    │
│  │  1. Valida permisos                                       │    │
│  │  2. Escribe en Turso                                      │    │
│  │  3. Retorna confirmación / error                          │    │
│  └──────────────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────────────┘
         │                          │
         │ WebSocket / SSE / HTTP   │ (eventos + comandos)
         │                          │
┌────────▼──────────────────────────▼──────────────────────────────┐
│  CLIENTE (navegador / móvil)                                      │
│                                                                    │
│  ┌───────────────────────┐     ┌─────────────────┐               │
│  │ LiveStore              │     │ Event Sync      │               │
│  │ ┌───────────────────┐ │     │ Client           │               │
│  │ │ local state       │◄┼─────┤ recibe eventos   │               │
│  │ │ (reactivo, offline)│ │     │ del servidor     │               │
│  │ │                   │ │     └─────────────────┘               │
│  │ │ Solo datos que    │ │                                        │
│  │ │ el usuario puede  │ │     ┌─────────────────┐               │
│  │ │ ver               │┼─────►│ Command Sender   │               │
│  │ └───────────────────┘ │     │ envía acciones   │               │
│  └───────────────────────┘     │ al servidor      │               │
│                                └─────────────────┘               │
│                                                                    │
│  Si el usuario extrae el storage:                                  │
│  → Solo ve eventos que YA le llegaron (sus datos permitidos)      │
│  → No ve datos de otros usuarios                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 1. ¿Por Qué Event Sourcing?

**Turso sync nativo = páginas WAL** → el cliente recibe TODO. No filtra por usuario.

**Event sourcing** → el servidor emite eventos semánticos. El cliente solo recibe eventos que le corresponden.

| | Turso Sync Nativo | Event Sourcing Custom |
|---|---|---|
| **Qué viaja** | Páginas WAL (4KB binario) | Eventos JSON (`task_created`, `salary_updated`) |
| **Filtrable** | ❌ No | ✅ Por usuario, rol, equipo |
| **El cliente tiene** | Archivo DB completo | Solo eventos autorizados |
| **Offline writes** | ✅ Sí (DB local) | ✅ Sí (eventos en cola local) |
| **Conflictos** | Last-writer-wins | A nivel de aplicación (tú decides) |
| **Complejidad** | Baja (viene hecho) | Alta (tú lo construyes) |

---

## 2. Cómo Funciona: Server → Client (Lectura)

El servidor emite un evento cada vez que algo cambia en Turso:

```typescript
// server/events.ts
import { db } from "./turso-db";

// Evento tras crear tarea
async function createTask(data: CreateTaskInput, userId: number) {
  // 1. Escribir en Turso
  const [task] = await db.insert(tasks).values(data).returning();
  
  // 2. Emitir evento
  const event: TaskCreatedEvent = {
    type: "task_created",
    payload: { id: task.id, title: task.title, assignedTo: task.assignedTo },
    userId, // quién lo creó
    timestamp: Date.now(),
  };
  
  // 3. Guardar en event log (para replay, audit, etc.)
  await db.insert(eventLog).values({
    type: event.type,
    payload: JSON.stringify(event.payload),
    userId: event.userId,
    timestamp: event.timestamp,
  });
  
  // 4. Broadcast filtrado a clientes relevantes
  await broadcastEvent(event);
}
```

**Filtrado por permisos:**

```typescript
// server/broadcast.ts
function getRecipientsForEvent(event: TaskEvent): number[] {
  // Admin ve todo
  if (event.type === "task_created") {
    // El creador y los miembros de su equipo lo ven
    const teamMembers = getTeamMembers(event.payload.assignedTo);
    return [event.userId, ...teamMembers, ADMIN_USER_ID];
  }
  
  if (event.type === "salary_updated") {
    // Solo admins y managers del equipo
    return [ADMIN_USER_ID, ...getManagersOfTeam(event.payload.teamId)];
  }
  
  return [event.userId]; // default: solo el creador
}
```

---

## 3. Cómo Funciona: Client → Server (Escritura)

El cliente envía un comando, no escribe directamente en Turso:

```typescript
// client/commands.ts
async function createTask(title: string) {
  const command = {
    type: "create_task",
    payload: { title, assignedTo: currentUserId },
    clientId: generateUUID(),  // idempotency key
  };
  
  // 1. Optimista: aplicar al estado local inmediatamente
  liveStore.mutate((state) => {
    state.tasks.push({
      id: `pending_${command.clientId}`,  // id temporal
      title: command.payload.title,
      assignedTo: command.payload.assignedTo,
      _status: "pending",  // marcador de estado
    });
  });
  
  // 2. Enviar comando al servidor
  const result = await sendCommand(command);
  
  if (result.ok) {
    // 3. Confirmar: reemplazar id temporal con id real
    liveStore.mutate((state) => {
      const task = state.tasks.find(t => t.id === `pending_${command.clientId}`);
      if (task) {
        task.id = result.taskId;
        task._status = "synced";
      }
    });
  } else {
    // 4. Revertir en caso de error
    liveStore.mutate((state) => {
      state.tasks = state.tasks.filter(t => t.id !== `pending_${command.clientId}`);
    });
    showError(result.error);
  }
}
```

**Servidor recibe el comando:**

```typescript
// server/commands.ts
async function handleCreateTask(command: CreateTaskCommand, userId: number) {
  // 1. Validar permisos
  if (!canCreateTask(userId, command.payload.assignedTo)) {
    return { ok: false, error: "Forbidden" };
  }
  
  // 2. Idempotencia (evitar duplicados si el cliente reenvía)
  const existing = await db.select()
    .from(eventLog)
    .where(eq(eventLog.clientId, command.clientId))
    .get();
  if (existing) {
    return { ok: true, taskId: JSON.parse(existing.payload).id };
  }
  
  // 3. Escribir en Turso
  const [task] = await db.insert(tasks)
    .values({
      title: command.payload.title,
      assignedTo: command.payload.assignedTo,
      createdBy: userId,
    })
    .returning();
  
  // 4. Emitir evento + guardar en event log
  await emitAndLogEvent({
    type: "task_created",
    payload: { id: task.id, title: task.title, assignedTo: task.assignedTo },
    userId,
    clientId: command.clientId,
  });
  
  return { ok: true, taskId: task.id };
}
```

---

## 4. Transporte: WebSocket, SSE o HTTP Long Poll

Para eventos en tiempo real, WebSocket es lo más natural:

```typescript
// server/ws.ts
import { WebSocketServer } from "ws";

const wss = new WebSocketServer({ port: 8080 });

// Mapa: userId → WebSocket
const connections = new Map<number, WebSocket>();

wss.on("connection", (ws, req) => {
  const userId = authenticate(req); // validar token
  connections.set(userId, ws);
  
  ws.on("message", async (data) => {
    const command = JSON.parse(data.toString());
    const result = await handleCommand(command, userId);
    ws.send(JSON.stringify({ type: "command_result", ...result }));
  });
  
  ws.on("close", () => connections.delete(userId));
  
  // Enviar snapshot inicial
  const initialEvents = await getEventsForUser(userId);
  ws.send(JSON.stringify({ type: "snapshot", events: initialEvents }));
});

// Broadcast de eventos filtrados
export function broadcastEvent(event: ServerEvent) {
  const recipients = getRecipientsForEvent(event);
  for (const userId of recipients) {
    const ws = connections.get(userId);
    if (ws) {
      ws.send(JSON.stringify({ type: "event", event }));
    }
  }
}
```

```typescript
// client/livestore-adapter.ts
import { LiveStore } from "@livestore/livestore";

const store = new LiveStore({ tasks: [], users: [] });

const ws = new WebSocket("wss://api.example.com/sync?token=...");

ws.onmessage = (msg) => {
  const data = JSON.parse(msg.data);
  
  if (data.type === "snapshot") {
    // Carga inicial: aplicar todos los eventos
    for (const event of data.events) {
      applyEvent(store, event);
    }
  }
  
  if (data.type === "event") {
    // Evento nuevo: aplicar al estado local
    applyEvent(store, data.event);
  }
  
  if (data.type === "command_result") {
    // Confirmación/rechazo de un comando enviado
    handleCommandResult(data);
  }
};

// Enviar comandos
function sendCommand(command: Command) {
  ws.send(JSON.stringify(command));
}
```

---

## 5. El Event Log en Turso

El event log es opcional pero muy útil. Te da audit trail, replay, y permite a clientes nuevos obtener el estado inicial sin hacer snapshot de toda la DB:

```sql
CREATE TABLE event_log (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  type TEXT NOT NULL,         -- 'task_created', 'task_updated', 'task_deleted'
  payload TEXT NOT NULL,       -- JSON con los datos del evento
  user_id INTEGER NOT NULL,    -- quién generó el evento
  client_id TEXT,              -- idempotency key del cliente
  timestamp INTEGER NOT NULL DEFAULT (unixepoch())
);

CREATE INDEX idx_event_log_timestamp ON event_log(timestamp);
```

**Replay para un cliente nuevo:**

```typescript
async function getEventsForUser(userId: number, since?: number): Promise<ServerEvent[]> {
  const rows = await db
    .select()
    .from(eventLog)
    .where(since ? gt(eventLog.timestamp, since) : undefined)
    .orderBy(eventLog.timestamp)
    .all();
  
  // Filtrar eventos que este usuario puede ver
  return rows
    .map(row => ({
      type: row.type,
      payload: JSON.parse(row.payload),
      userId: row.userId,
      timestamp: row.timestamp,
      clientId: row.clientId,
    }))
    .filter(event => canUserSeeEvent(userId, event));
}
```

---

## 6. ¿Por Qué LiveStore y No Solo WebSocket + Estado?

LiveStore aporta:

- **Reactivity declarativa**: componentes se suscriben a slices del estado y re-renderizan automáticamente cuando cambian
- **Mutaciones con undo/redo**: el historial de eventos permite deshacer cambios
- **Persistencia local**: el estado sobrevive refrescos de página (localStorage/IndexedDB)
- **Optimistic updates**: aplicas cambios al estado local antes de que el servidor confirme
- **CRDT opcional**: para conflictos offline → online

Sin LiveStore, tendrías que construir todo esto manualmente con `useState` + `useEffect` + IndexedDB. LiveStore te da una abstracción lista sobre eventos.

---

## 7. Comparativa Final

| Arquitectura | DB en cliente | Permisos reales | Offline | Complejidad |
|-------------|--------------|----------------|---------|------------|
| **Turso sync nativo** | app.db completa | ❌ No (el archivo tiene todo) | ✅ Total | Baja |
| **Turso solo backend + API REST** | Nada | ✅ Sí (API filtra) | ❌ No | Media |
| **Turso + LiveStore + eventos** | Solo eventos autorizados | ✅ Sí (filtrado server-side) | ✅ Sí (eventos locales) | Alta |
| **Turso + TinyBase + eventos** | Solo datos sincronizados | ✅ Sí (filtrado server-side) | ✅ Sí | Media-Alta |

---

## 8. ¿Vale la Pena?

**Sí**, si necesitas todo esto junto:
- ✅ Offline funcional (no solo caché, sino writes offline)
- ✅ Permisos granulares (cada usuario solo recibe sus datos)
- ✅ Reactividad en tiempo real (UI se actualiza sola)
- ✅ Audit trail (event log inmutable en Turso)

**No**, si:
- ❌ Tu app es solo lectura para la mayoría de usuarios
- ❌ No necesitas offline completo (basta con un caché HTTP)
- ❌ Los usuarios de una misma org confían entre sí (no necesitas filtrar)

La complejidad extra de event sourcing solo se justifica si los permisos + offline son requisitos duros. Si no, Turso sync nativo con datos sensibles en API separada (documento `permissions-org-users-es.md`) es mucho más simple.
