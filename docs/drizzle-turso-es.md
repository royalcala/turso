# Usar Drizzle ORM con Turso (Frontend y Backend)

Turso —la reescritura en Rust de SQLite— funciona con Drizzle ORM tanto en frontend (DB embebida local) como en backend (Turso Cloud vía HTTP). La clave: usar el driver correcto para cada lado.

---

## 1. ¿Por Qué Dos Drivers Distintos?

Turso opera en dos modos según dónde se ejecuta:

```
┌─────────────────────────────────────────────────────────────────────┐
│  FRONTEND (dispositivo del usuario)                                  │
│                                                                      │
│  ┌─────────────┐      ┌───────────────────┐                         │
│  │ app.db       │◄────│ @tursodatabase/   │  DB local en disco      │
│  │ (archivo)    │      │ sync              │  API: better-sqlite3    │
│  └─────────────┘      └────────┬──────────┘                         │
│                                │                                    │
│                     Drizzle ORM (better-sqlite3)                    │
│                                                                      │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  │
│                                                                      │
│  BACKEND (servidor Node/Bun/Deno)                                    │
│                                                                      │
│  ┌──────────────┐     ┌───────────────────┐                         │
│  │ Turso Cloud   │◄────│ @libsql/client    │  HTTP a servidor remoto │
│  │ o sqld        │     │                   │  API: @libsql/client   │
│  └──────────────┘     └────────┬──────────┘                         │
│                                │                                    │
│                     Drizzle ORM (libsql)                            │
└─────────────────────────────────────────────────────────────────────┘
```

| | Frontend | Backend |
|---|---|---|
| **Motor** | `turso_core` (Rust) via WASM o native | `turso_core` o libSQL (C) en servidor |
| **Conexión** | Directa al archivo `app.db` | HTTP a Turso Cloud o sqld |
| **Paquete npm** | `@tursodatabase/sync` | `@libsql/client` |
| **API que expone** | `better-sqlite3` (`run`, `get`, `all`, `prepare`) | `@libsql/client` (`execute`, `batch`) |
| **Driver Drizzle** | `drizzle-orm/better-sqlite3` | `drizzle-orm/libsql` |

---

## 2. Setup Compartido: Schema de Drizzle

Crea el schema una vez. Ambos lados lo comparten:

```typescript
// shared/schema.ts
import { sqliteTable, integer, text } from "drizzle-orm/sqlite-core";

export const users = sqliteTable("users", {
  id: integer("id").primaryKey({ autoIncrement: true }),
  name: text("name").notNull(),
  email: text("email").notNull().unique(),
  createdAt: integer("created_at").notNull().default(new Date().getTime()),
});

export const posts = sqliteTable("posts", {
  id: integer("id").primaryKey({ autoIncrement: true }),
  userId: integer("user_id").references(() => users.id),
  title: text("title").notNull(),
  body: text("body").notNull(),
  createdAt: integer("created_at").notNull().default(new Date().getTime()),
});
```

---

## 3. Backend: Drizzle + Turso Cloud (HTTP)

```typescript
// backend/db.ts
import { drizzle } from "drizzle-orm/libsql";
import { createClient } from "@libsql/client";
import * as schema from "../shared/schema";

const client = createClient({
  url: process.env.TURSO_DB_URL!,       // libsql://my-db.turso.io
  authToken: process.env.TURSO_AUTH_TOKEN!,
});

const db = drizzle(client, { schema });

export { db };
```

```typescript
// backend/api.ts
import { db } from "./db";
import { users, posts } from "../shared/schema";
import { eq, desc } from "drizzle-orm";

// Obtener posts recientes
export async function getRecentPosts(limit = 100) {
  return db
    .select()
    .from(posts)
    .orderBy(desc(posts.createdAt))
    .limit(limit);
}

// Crear usuario
export async function createUser(name: string, email: string) {
  return db.insert(users).values({ name, email }).returning();
}

// Posts de un usuario con join
export async function getUserPosts(userId: number) {
  return db
    .select({
      postId: posts.id,
      title: posts.title,
      userName: users.name,
    })
    .from(posts)
    .innerJoin(users, eq(posts.userId, users.id))
    .where(eq(users.id, userId));
}
```

---

## 4. Frontend: Drizzle + Turso Embebido (Local)

```typescript
// frontend/db.ts
import { drizzle } from "drizzle-orm/better-sqlite3";
import { connect } from "@tursodatabase/sync";
import * as schema from "../shared/schema";

// Abrir DB local con sync
const conn = await connect({
  path: "app.db",
  url: process.env.TURSO_DB_URL!,         // URL del remoto para sync
  authToken: process.env.TURSO_AUTH_TOKEN!,
  clientName: "my-app",
  bootstrapIfEmpty: true,
  partialBootstrapStrategyPrefix: 524288, // 512KB iniciales
});

const db = drizzle(conn, { schema });

export { db, conn };
```

```typescript
// frontend/feed.tsx
import { useEffect, useState } from "react";
import { db, conn } from "./db";
import { posts, users } from "../shared/schema";
import { eq, desc } from "drizzle-orm";

export function Feed() {
  const [feedPosts, setFeedPosts] = useState<typeof posts.$inferSelect[]>([]);

  // Sync al montar y cada 5 segundos
  useEffect(() => {
    const syncAndQuery = async () => {
      await conn.push();
      const changed = await conn.pull();

      if (changed) {
        const rows = await db
          .select()
          .from(posts)
          .orderBy(desc(posts.createdAt))
          .limit(100);

        setFeedPosts(rows);
      }
    };

    syncAndQuery();
    const id = setInterval(syncAndQuery, 5000);
    return () => clearInterval(id);
  }, []);

  return (
    <div>
      {feedPosts.map((post) => (
        <Post key={post.id} post={post} />
      ))}
    </div>
  );
}
```

---

## 5. Migraciones con Drizzle Kit

Drizzle Kit genera migraciones desde tu schema. Las migraciones se aplican de forma distinta según el lado:

### Generar migración

```bash
npx drizzle-kit generate --schema=./shared/schema.ts --out=./migrations
```

Esto crea `migrations/0000_initial.sql` con el SQL de tu schema.

### Aplicar en backend (Turso Cloud)

```bash
# Via CLI de Turso
turso db shell my-db < migrations/0000_initial.sql

# O programáticamente en deploy
turso db shell $TURSO_DB_URL --auth-token $TURSO_AUTH_TOKEN < migrations/0000_initial.sql
```

### Aplicar en frontend (DB local)

```typescript
// frontend/migrate.ts
import { conn, db } from "./db";
import fs from "node:fs";

// Ejecutar migración en la DB local
const migration = fs.readFileSync("./migrations/0000_initial.sql", "utf-8");
conn.exec(migration);
```

Con bootstrap, el frontend recibe el schema del servidor. Las migraciones locales son para desarrollo o para cuando el frontend es el migration author.

---

## 6. Transacciones con Drizzle

```typescript
// Frontend (DB local, transacción síncrona en Rust)
import { db } from "./db";
import { users, posts } from "../shared/schema";

await db.transaction(async (tx) => {
  const [user] = await tx.insert(users).values({ name: "Ana", email: "ana@example.com" }).returning();
  await tx.insert(posts).values({ userId: user.id, title: "Hola", body: "Primer post" });
});

// Backend (Turso Cloud, transacción via HTTP)
import { db } from "../backend/db";

await db.transaction(async (tx) => {
  const [user] = await tx.insert(users).values({ name: "Ana", email: "ana@example.com" }).returning();
  await tx.insert(posts).values({ userId: user.id, title: "Hola", body: "Primer post" });
});
```

Ambos lados usan la misma API de Drizzle. La diferencia es interna: el frontend ejecuta en Rust/WASM local; el backend envía SQL por HTTP.

---

## 7. Drizzle en Ambos Lados con Sync (Flujo Completo)

```typescript
// ── shared/schema.ts (mismo archivo en ambos proyectos) ──
import { sqliteTable, integer, text } from "drizzle-orm/sqlite-core";

export const todos = sqliteTable("todos", {
  id: integer("id").primaryKey({ autoIncrement: true }),
  title: text("title").notNull(),
  done: integer("done", { mode: "boolean" }).notNull().default(false),
});

// ── frontend/db.ts ──
import { drizzle } from "drizzle-orm/better-sqlite3";
import { connect } from "@tursodatabase/sync";
import * as schema from "../shared/schema";

const conn = await connect({
  path: "app.db",
  url: "libsql://my-db.turso.io",
  authToken: "...",
});
export const db = drizzle(conn, { schema });
export { conn };

// ── frontend/TodoList.tsx ──
import { db, conn } from "./db";
import { todos } from "../shared/schema";
import { eq, desc } from "drizzle-orm";

async function addTodo(title: string) {
  // INSERT local (el CDC lo captura automáticamente)
  await db.insert(todos).values({ title });
  // Push inmediato al servidor
  await conn.push();
}

async function loadTodos() {
  // Pull de cambios nuevos
  await conn.push();
  await conn.pull();
  // Query local
  return db.select().from(todos).orderBy(desc(todos.id)).limit(50);
}

async function toggleTodo(id: number) {
  const [todo] = await db.select({ done: todos.done }).from(todos).where(eq(todos.id, id));
  await db.update(todos).set({ done: !todo.done }).where(eq(todos.id, id));
  await conn.push();
}

// ── backend/api.ts ──
import { createClient } from "@libsql/client";
import { drizzle } from "drizzle-orm/libsql";
import * as schema from "../shared/schema";

const client = createClient({
  url: process.env.TURSO_DB_URL!,
  authToken: process.env.TURSO_AUTH_TOKEN!,
});
const db = drizzle(client, { schema });

// API: listar todos (lee del servidor)
export async function listTodos() {
  return db.select().from(schema.todos).orderBy(desc(schema.todos.id)).limit(100);
}
```

---

## 8. Compatibilidad Verificada

El repositorio de Turso contiene un test que valida Drizzle con el motor Rust:

```typescript
// bindings/javascript/packages/native/promise.test.ts:7
test('drizzle-orm', async () => {
    const conn = await connect(path);
    const db = drizzle(conn);
    await db.run('CREATE TABLE t(x, y)');
    await db.run(sql`INSERT INTO t VALUES (${1}, 'hello')`);
    const rows = await db.all('SELECT * FROM t');
    expect(rows.length).toBe(1);
});
```

Esto prueba `drizzle-orm/better-sqlite3` con el motor `turso_core` (Rust) vía los bindings nativos. Funciona.

---

## 9. Limitaciones

| Limitación | Detalle |
|------------|---------|
| **Frontend no usa `drizzle-orm/libsql`** | La API del driver no coincide. Usa `better-sqlite3`. |
| **`execute()` de libsql no existe en frontend** | Usa `run()`, `get()`, `all()` en su lugar. |
| **Transacciones son locales** | `db.transaction()` en frontend es una transacción SQLite local, no distribuida. |
| **Schema version** | Si el schema cambia (migración en backend), los prepared statements del frontend se recompilan automáticamente al hacer pull. |
| **Drizzle Kit genera SQL** | Las migraciones son SQL estándar — se aplican igual en ambos lados. |

---

## 10. Resumen

| Pregunta | Respuesta |
|----------|-----------|
| ¿Drizzle funciona con Turso Rust? | ✅ Sí, verificado en test del repo |
| ¿Driver para frontend? | `drizzle-orm/better-sqlite3` |
| ¿Driver para backend? | `drizzle-orm/libsql` + `@libsql/client` |
| ¿Mismo schema? | ✅ Sí, compartes `schema.ts` |
| ¿Migraciones? | ✅ Drizzle Kit genera SQL; aplicas en backend o ambos |
| ¿Transacciones? | ✅ `db.transaction()` funciona en ambos lados |
| ¿Sync + Drizzle? | ✅ Haces `conn.pull()`, luego query con Drizzle normalmente |
