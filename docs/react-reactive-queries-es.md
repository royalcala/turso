# Queries Reactivas en React con Turso

Turso no tiene mecanismo nativo de notificaciones (no hay `onChange`, ni `update_hook`, ni eventos push). La base de datos es local — las queries son síncronas contra disco. Para reactividad en React, debes **implementarla manualmente** sobre el API de sync.

---

## 1. Lo Que Turso Te Da Hoy

```typescript
import { Database } from "@tursodb/sync";

const db = await Database.open("app.db", {
  remoteUrl: "libsql://my-db.turso.io",
  clientName: "my-app",
});

// Pull retorna boolean: ¿hubo cambios nuevos desde el servidor?
const hasChanges: boolean = await db.pull();

// Push sube cambios locales al servidor
await db.push();

// Stats te da metadata
const stats = await db.stats();
// stats.cdcOperations — cambios locales sin subir
// stats.lastPullUnixTime — timestamp del último pull
// stats.revision — token opaco de revisión
```

No hay `db.onChange(callback)`, `db.subscribe()`, ni `useSync()`. Tú construyes la reactividad.

---

## 2. Opción A: Polling Periódico (La Más Simple)

Ideal para la mayoría de apps. Un `setInterval` que hace pull cada N segundos.

```tsx
import { useEffect, useState, useCallback } from "react";

function useSync(db: Database, intervalMs = 5000) {
  const [lastPullAt, setLastPullAt] = useState<number | null>(null);

  const syncNow = useCallback(async () => {
    await db.push();
    const changed = await db.pull();
    if (changed) {
      setLastPullAt(Date.now());
    }
    await db.checkpoint();
  }, [db]);

  // Sync automático cada N segundos
  useEffect(() => {
    const id = setInterval(syncNow, intervalMs);
    return () => clearInterval(id);
  }, [syncNow, intervalMs]);

  // Sync al montar
  useEffect(() => { syncNow(); }, [syncNow]);

  return { lastPullAt, syncNow };
}

// Uso en un componente:
function Feed() {
  const db = useDatabase();
  const { lastPullAt } = useSync(db, 3000);

  // Re-query cuando lastPullAt cambia
  const posts = useQuery(
    ["posts", lastPullAt],
    () => db.query("SELECT * FROM posts ORDER BY created_at DESC LIMIT 100")
  );

  return posts.map(p => <Post key={p.id} post={p} />);
}
```

**Pros:** Simple, funciona.  
**Contras:** Polling innecesario si no hay cambios. Overhead de red en apps con poco tráfico.

---

## 2.1 Opción A Mejorada: Pull Condicional (Solo Si Hay Datos Locales)

No hagas pull si no hay nada local que compartir ni necesidad de recibir:

```tsx
function useSmartSync(db: Database, intervalMs = 5000) {
  const syncNow = useCallback(async () => {
    const stats = await db.stats();

    // Solo hace push si hay cambios locales pendientes
    if (stats.cdcOperations > 0) {
      await db.push();
    }

    // Siempre hace pull (podría haber cambios remotos)
    const changed = await db.pull();
    if (changed) {
      setLastPullAt(Date.now());
    }
  }, [db]);

  useEffect(() => {
    const id = setInterval(syncNow, intervalMs);
    return () => clearInterval(id);
  }, [syncNow, intervalMs]);

  useEffect(() => { syncNow(); }, []);
}
```

---

## 3. Opción B: Long Poll (Conexión Abierta, Más Eficiente)

Turso soporta **long polling** nativo. El servidor mantiene la conexión HTTP abierta hasta que haya cambios (o timeout). Esto evita polling innecesario:

```tsx
// Configurar al abrir la DB
const db = await Database.open("app.db", {
  remoteUrl: "libsql://my-db.turso.io",
  clientName: "my-app",
  longPollTimeoutMs: 15000,  // servidor espera hasta 15s por cambios
});

// Hook con long poll en segundo plano
function useSyncLongPoll(db: Database) {
  const [lastPullAt, setLastPullAt] = useState<number | null>(null);
  const running = useRef(false);

  const syncLoop = useCallback(async () => {
    running.current = true;

    while (running.current) {
      await db.push();
      const changed = await db.pull();  // se bloquea hasta 15s o cambios

      if (changed) {
        setLastPullAt(Date.now());
      }
      // pull() ya esperó 15s. Si no hubo cambios, reintenta
    }
  }, [db]);

  useEffect(() => {
    syncLoop();
    return () => { running.current = false; };
  }, [syncLoop]);

  return { lastPullAt };
}
```

**Importante:** `longPollTimeoutMs` **no está implementado en el servidor de testing** (`tursodb --sync-server`). Funciona con Turso Cloud. Para self-hosting, necesitas tu propio servidor con long poll.

---

## 4. Opción C: Pull Manual (On Focus, On Action)

No sincronices por tiempo — sincroniza en momentos clave:

```tsx
function useSyncOnFocus(db: Database) {
  const [lastPullAt, setLastPullAt] = useState<number | null>(null);

  const syncNow = useCallback(async () => {
    await db.push();
    const changed = await db.pull();
    if (changed) setLastPullAt(Date.now());
  }, [db]);

  // Pull al montar
  useEffect(() => { syncNow(); }, [syncNow]);

  // Pull cuando la app vuelve a primer plano
  useEffect(() => {
    const onFocus = () => syncNow();
    window.addEventListener("focus", onFocus);
    document.addEventListener("visibilitychange", () => {
      if (document.visibilityState === "visible") syncNow();
    });
    return () => {
      window.removeEventListener("focus", onFocus);
    };
  }, [syncNow]);

  return { lastPullAt, syncNow };
}

function Feed() {
  const db = useDatabase();
  const { lastPullAt, syncNow } = useSyncOnFocus(db);

  const posts = useQuery(
    ["posts", lastPullAt],
    () => db.query("SELECT * FROM posts ORDER BY created_at DESC LIMIT 100")
  );

  return (
    <div>
      <button onClick={syncNow}>Refrescar</button>
      {posts.map(p => <Post key={p.id} post={p} />)}
    </div>
  );
}
```

**Cuándo usar:** Apps donde los datos cambian poco entre sesiones (ej. perfil, configuración).

---

## 5. Opción D: CDC Local Como Trigger de Re-Render

Monitoreas la tabla `turso_cdc` directamente para detectar cambios locales (no remotos):

```tsx
function useLocalChanges(db: Database) {
  const [changeId, setChangeId] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      const row = db.query("SELECT MAX(change_id) as max_id FROM turso_cdc");
      const latest = row[0]?.max_id ?? 0;
      if (latest > changeId) {
        setChangeId(latest);
      }
    }, 1000);
    return () => clearInterval(id);
  }, [changeId]);

  return changeId;
}

// Uso: re-query cuando hay cambios locales (ej. el usuario insertó algo)
function Tasks() {
  const db = useDatabase();
  const changeId = useLocalChanges(db);

  const tasks = useQuery(
    ["tasks", changeId],
    () => db.query("SELECT * FROM tasks ORDER BY id DESC")
  );

  return tasks.map(t => <Task key={t.id} task={t} />);
}
```

**Ideal para:** UI optimista. El usuario hace INSERT y la UI se actualiza sin esperar el round-trip de sync.

---

## 6. Opción E: Integración con TanStack Query (React Query)

Si ya usas TanStack Query, puedes invalidar queries cuando sync trae cambios:

```tsx
import { useQueryClient, useQuery } from "@tanstack/react-query";

function useSyncWithQueryClient(db: Database) {
  const queryClient = useQueryClient();

  const syncNow = useCallback(async () => {
    const statsBefore = await db.stats();

    await db.push();
    const changed = await db.pull();

    if (changed) {
      // Invalida TODAS las queries — se re-ejecutarán contra datos nuevos
      queryClient.invalidateQueries();
    }
  }, [db, queryClient]);

  useEffect(() => {
    const id = setInterval(syncNow, 5000);
    return () => clearInterval(id);
  }, [syncNow]);

  return { syncNow };
}

function Feed() {
  const db = useDatabase();
  useSyncWithQueryClient(db);

  const { data: posts } = useQuery({
    queryKey: ["posts"],
    queryFn: () => db.query("SELECT * FROM posts ORDER BY created_at DESC LIMIT 100"),
    // La query se ejecuta local, es instantánea — no necesita staleTime largo
    staleTime: 0,
  });

  return posts?.map(p => <Post key={p.id} post={p} />);
}
```

**Ideal para:** Apps que ya usan TanStack Query y quieren integrar Turso sin reescribir el data layer.

---

## 7. Opción F: Sync Loop en Web Worker (Sin Bloquear UI)

Mueve el sync a un Web Worker para no saturar el thread principal:

```tsx
// sync-worker.ts
import { Database } from "@tursodb/sync";

let db: Database;
let running = false;

async function syncLoop() {
  running = true;
  while (running) {
    await db.push();
    const changed = await db.pull();
    if (changed) {
      postMessage({ type: "sync", timestamp: Date.now() });
    }
  }
}

self.onmessage = async (e) => {
  if (e.data.type === "start") {
    db = await Database.open(e.data.dbPath, e.data.config);
    syncLoop();
  }
  if (e.data.type === "stop") {
    running = false;
  }
};
```

```tsx
// App.tsx
function useSyncWorker(dbPath: string, dbConfig: object) {
  const [lastPullAt, setLastPullAt] = useState<number | null>(null);
  const workerRef = useRef<Worker | null>(null);

  useEffect(() => {
    const worker = new Worker("./sync-worker.ts");
    workerRef.current = worker;

    worker.postMessage({ type: "start", dbPath, config: dbConfig });
    worker.onmessage = (e) => {
      if (e.data.type === "sync") {
        setLastPullAt(e.data.timestamp);
      }
    };

    return () => {
      worker.postMessage({ type: "stop" });
      worker.terminate();
    };
  }, [dbPath]);

  return { lastPullAt };
}
```

---

## 8. Comparativa de Opciones

| Opción | Red | Latencia | CPU | Mejor para |
|--------|-----|----------|-----|-----------|
| **A: Polling** | Alta (requests cada N seg) | N segundos max | Baja | Prototipos, apps simples |
| **B: Long Poll** | Baja (conexión persistente) | ~50ms | Muy baja | Producción, apps con tráfico |
| **C: Manual** | Muy baja (solo on demand) | Depende del usuario | Mínima | Apps de lectura, perfiles |
| **D: CDC local** | Cero | Instantáneo | Mínima | UI optimista local |
| **E: TanStack Query** | Media | N segundos | Baja | Apps con data layer existente |
| **F: Web Worker** | Depende de A/B | Depende de A/B | Cero en main thread | Apps que no pueden bloquear UI |

---

## 9. Recomendación

```
┌─────────────────────────────────────────────────────────────────┐
│  RECOMENDACIÓN GENERAL                                           │
│                                                                  │
│  Para producción, combina:                                       │
│                                                                  │
│  1. LONG POLL (opción B) como mecanismo base de sync             │
│     → Eficiente, baja latencia, bajo overhead de red              │
│     → pull() se bloquea hasta que hay cambios o timeout           │
│                                                                  │
│  2. QUERY INVALIDATION (opción E) para re-renderizar              │
│     → TanStack Query o similar con invalidateQueries()            │
│     → O un key compartido que cambia en cada sync                 │
│                                                                  │
│  3. CDC LOCAL (opción D) para UI optimista                       │
│     → El usuario ve su cambio inmediatamente                      │
│     → Sin esperar el round-trip de sync                          │
│                                                                  │
│  4. MANUAL PULL (opción C) en onFocus + botón refresh            │
│     → Buen UX: sincroniza al volver a la app                     │
│                                                                  │
│  Para self-hosting sin long poll:                                 │
│  → Usa polling (opción A) con intervalo de 3-5 segundos          │
│  → Ajusta según tráfico de tu app                                │
└─────────────────────────────────────────────────────────────────┘
```

---

## 10. Hook Completo de Ejemplo (Producción)

```tsx
import { useEffect, useState, useCallback, useRef } from "react";
import type { Database } from "@tursodb/sync";

interface UseSyncOptions {
  db: Database;
  longPollMs?: number;
  onSync?: (hasChanges: boolean) => void;
}

export function useTursoSync({ db, longPollMs = 15000, onSync }: UseSyncOptions) {
  const [lastSyncAt, setLastSyncAt] = useState<number | null>(null);
  const [isSyncing, setIsSyncing] = useState(false);
  const running = useRef(true);

  const syncNow = useCallback(async () => {
    if (isSyncing) return;
    setIsSyncing(true);
    try {
      await db.push();
      const changed = await db.pull();

      if (changed) {
        const now = Date.now();
        setLastSyncAt(now);
        onSync?.(true);
      } else {
        onSync?.(false);
      }
    } finally {
      setIsSyncing(false);
    }
  }, [db, isSyncing, onSync]);

  // Loop de sync en segundo plano (long poll o polling)
  useEffect(() => {
    running.current = true;

    const loop = async () => {
      while (running.current) {
        await syncNow();
        // si no hay long poll, espera entre pulls
        if (longPollMs === 0) {
          await new Promise(r => setTimeout(r, 3000));
        }
        // si hay long poll, pull() ya esperó — reintenta inmediato
      }
    };

    loop();
    return () => { running.current = false; };
  }, [syncNow, longPollMs]);

  // Sync al montar y en focus
  useEffect(() => {
    syncNow();
    const onFocus = () => syncNow();
    window.addEventListener("focus", onFocus);
    return () => window.removeEventListener("focus", onFocus);
  }, [syncNow]);

  return { lastSyncAt, isSyncing, syncNow };
}
```

Uso:

```tsx
function App() {
  const db = useDatabase();
  const { lastSyncAt } = useTursoSync({ db, longPollMs: 15000 });

  return (
    <Feed key={lastSyncAt} db={db} />
  );
}
```

---

## 11. Limitaciones y Consideraciones

| Limitación | Impacto |
|------------|---------|
| **No hay eventos push nativos** | Debes implementar polling o long poll |
| **Long poll solo en Turso Cloud** | Self-hosting requiere polling manual |
| **No hay update_hook en SQLite** | No puedes suscribirte a cambios a nivel motor |
| **CDC es solo local** | `turso_cdc` solo captura tus propios cambios, no los remotos hasta después del pull |
| **No hay conflicto con queries** | Las queries son locales, no bloquean el sync ni viceversa |

**En resumen:** Turso te da la base de datos local con sync. La reactividad la construyes tú encima con polling, long poll, y patrones estándar de React. No es Firebase ni Supabase Realtime — es una BD embebida que se sincroniza. La buena noticia es que las queries son instantáneas (locales), así que re-renderizar es barato.
