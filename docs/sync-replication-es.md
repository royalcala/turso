# Arquitectura de Sincronización y Replicación de Turso

## Resumen Ejecutivo

Turso implementa un modelo de **replicación multi-primario (active-active)** basado en **CDC lógico (Change Data Capture)**. Cada cliente conectado es un par que puede leer y escribir localmente, y el motor de sincronización reconcilia los cambios bidireccionalmente con Turso Cloud o cualquier endpoint remoto.

**No hay un primario único.** Todos los clientes son pares simétricos que pueden hacer push y pull de cambios.

---

## 0. Conceptos Fundamentales: Páginas, WAL, Push y Pull

Para entender la sincronización de Turso, imagina que tu base de datos es un **cuaderno cuadriculado**.

### Analogía del Cuaderno

| Concepto | Analogía | Realidad Técnica |
|----------|----------|-----------------|
| **Base de datos** | Un cuaderno de 100 hojas | Archivo `app.db` |
| **Página** | Cada hoja del cuaderno | Bloque de 4096 bytes (4KB) |
| **Fila** | Una línea escrita en una hoja | Un registro (ej. `id=5, name='Ana'`) |
| **WAL** | Post-its pegados encima de las hojas | Archivo `app-wal` |
| **Checkpoint** | Copiar los post-its al cuaderno | Pasar frames del WAL al `.db` |

> **Una página (4KB) puede contener varias filas.** Por ejemplo, la hoja 7 del cuaderno puede tener 3 usuarios (Ana, Beto, Carlos) y la hoja 8 puede tener 5 tareas.

### Ejemplo Concreto: App de Tareas

Tienes una app de tareas con estas tablas:

```sql
CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT);
CREATE TABLE tasks (id INTEGER PRIMARY KEY, user_id INTEGER, title TEXT, done INTEGER);
```

Tu base de datos tiene 3 usuarios y 100 tareas. Ocupa 5 páginas (20KB).

---

### Push: Cómo Turso Envía Tus Cambios al Servidor

**Escenario:** El usuario Ana (en su celular, offline) marca la tarea #42 como completada:

```sql
UPDATE tasks SET done = 1 WHERE id = 42;
```

**Qué pasa internamente:**

```
PASO 1 — CDC captura el cambio:
  Turso detecta: "alguien modificó la fila 42 de la tabla tasks"
  Guarda en tabla interna turso_cdc:
    change_id: 128
    table_name: "tasks"
    change_type: UPDATE
    id: 42
    after: {"id": 42, "user_id": 3, "title": "Pagar renta", "done": 1}

PASO 2 — Push convierte CDC a SQL:
  Turso toma esa fila CDC y genera:
    UPDATE tasks SET done = 1 WHERE id = 42;

PASO 3 — Envía el SQL al servidor:
  POST /v2/pipeline
  Body: "BEGIN; UPDATE tasks SET done = 1 WHERE id = 42; COMMIT;"

PASO 4 — El servidor ejecuta el SQL:
  El servidor recibe la sentencia y la ejecuta en su propia copia de la DB.
  Ahora el servidor también tiene tasks.done = 1 para id=42.
```

**Lo importante del Push:** Turso **no envía páginas** — envía la **sentencia SQL exacta** que ejecutaste. Es como si le dictaras a alguien por teléfono: "en la tabla tasks, fila 42, cambia done a 1". La otra persona ejecuta la instrucción.

---

### Pull: Cómo Turso Recibe Cambios de Otros

**Escenario:** Carlos (otro usuario, en su laptop) agregó 3 tareas nuevas mientras Ana estaba offline. Ahora Ana abre la app y hace sync.

**Qué pasa internamente:**

```
PASO 1 — Ana hace pull:
  POST /pull-updates
  Body: { client_revision: "abc-123" }

  El servidor ve que Ana está en revisión "abc-123" y que
  el servidor tiene cambios más nuevos (las 3 tareas de Carlos).

PASO 2 — El servidor responde con PÁGINAS (no SQL):
  El servidor NO envía "INSERT INTO tasks VALUES (...)".
  Envía las páginas de 4KB que contienen los datos modificados:

  Respuesta:
    PageData { page_id: 4, encoded_page: [4096 bytes] }
    PageData { page_id: 5, encoded_page: [4096 bytes] }

  Estas páginas contienen:
    - Página 4: los datos nuevos de las tareas 101, 102
    - Página 5: la tabla sqlite_schema actualizada + parte de task 103

PASO 3 — Ana escribe las páginas en su WAL local:
  Las páginas se guardan en el archivo app-wal como "frames".
  El WAL de Ana ahora dice: "la página 4 ahora se ve así, la página 5 ahora se ve así".

PASO 4 — Ana aplica el WAL y re-ejecuta sus cambios locales:
  - Revierte sus cambios locales pendientes
  - Aplica los frames remotos (las 3 tareas de Carlos quedan visibles)
  - Re-ejecuta su UPDATE tasks SET done = 1 WHERE id = 42 encima
```

**Lo importante del Pull:** Turso **no recibe SQL** — recibe **páginas completas de 4KB** que ya contienen los datos modificados. Es como recibir por fax las hojas del cuaderno de Carlos con sus tachaduras y anotaciones, y pegarlas encima de tu cuaderno. Es más rápido que ejecutar 100 INSERTs uno por uno, pero **no puedes filtrar qué viene dentro de cada página**.

---

### ¿Por Qué Push Es SQL y Pull Es Páginas?

| | Push (envías) | Pull (recibes) |
|---|---|---|
| **Formato** | SQL (`INSERT INTO...`) | Páginas 4KB (datos binarios) |
| **Filtrable** | ✅ Sí — puedes decidir qué tablas/filas enviar | ❌ No — recibes la página entera con todo lo que contiene |
| **Velocidad** | Lento (cada SQL se parsea y ejecuta) | Rápido (los 4KB se copian directo al WAL) |
| **Por qué** | Necesitas control sobre QUÉ compartir | Necesitas velocidad al recibir muchos cambios |

**Ejemplo de por qué el push es filtrable pero el pull no:**

Imagina que tienes una tabla `user_sessions` que solo debe existir localmente. Con `tables_ignore = ["user_sessions"]`:

- **Push**: Turso ve un cambio en `user_sessions`, lo ignora, no genera SQL. ✅ No se comparte.
- **Pull**: El servidor envía la página 12 que contiene datos de `tasks` Y de `user_sessions`. Turso escribe la página entera en tu WAL local. ❌ No puede separar lo que viene dentro de una página.

---

### Partial Sync: No Descargar Toda la DB de Golpe

**Escenario:** Tu app tiene 50,000 tareas históricas (200MB de DB). Un usuario nuevo instala la app. No quieres que espere a descargar 200MB antes de poder usarla.

**Solución — Partial Sync con Prefix:**

```
Configuración: partial_sync_opts = Prefix { length: 1048576 }  (1MB inicial)

Estado inicial del archivo local app.db:
┌─────────────────────┬───────────────────────────────────────┐
│ Descargado (1MB)    │ HUECO — sin datos                     │
│ Páginas 0 a 255     │ Páginas 256 a 50,000                  │
│ Contiene: schema,   │ Contiene: tareas antiguas, logs,      │
│ usuarios, primeras  │ datos que el usuario aún no necesita   │
│ 200 tareas          │                                       │
└─────────────────────┴───────────────────────────────────────┘

El usuario abre la app:
  - Ve su perfil (página 12, ya descargada) ✅ instantáneo
  - Ve las primeras 200 tareas (páginas 20-40, ya descargadas) ✅ instantáneo
  - Hace scroll hacia abajo, necesita la tarea #5000
    → Esa tarea está en la página 256 (HUECO)
    → Turso detecta el hueco, descarga SOLO la página 256 del servidor
    → 4KB descargados en milisegundos, la tarea aparece
```

**Partial Sync con Query:**

```
Configuración: partial_sync_opts = Query { query: "SELECT * FROM tasks WHERE user_id = 5" }

El servidor ejecuta ese SQL, determina qué páginas toca (digamos 20, 45, 89, 200),
y envía SOLO esas páginas en el bootstrap inicial.

El resto de páginas se descargan bajo demanda igual que con Prefix.
```

### Escenario Completo: Feed de Últimos 100 Eventos

Este es el caso de uso más común en apps: una línea de tiempo o feed que muestra los items más recientes.

**Setup:** Una app de red social. La tabla `posts` tiene 500,000 registros históricos (~2GB). La app solo muestra los últimos 100 posts en el feed.

```sql
CREATE TABLE posts (id INTEGER PRIMARY KEY, user_id INTEGER, body TEXT, created_at INTEGER);
```

**Día 1 — Primera apertura de la app (bootstrap):**

```
┌─────────────────────────────────────────────────────────────────┐
│ El usuario instala la app. No hay archivo app.db local.          │
│                                                                  │
│ Configuración: partial_sync_opts = Prefix { length: 524288 }     │
│ Bootstrap: descarga los primeros 512KB (128 páginas).            │
│                                                                  │
│ app.db local después del bootstrap:                              │
│ ┌──────────────┬───────────────────────────────────────────┐    │
│ │ 128 páginas  │ HUECO (1.999GB sin descargar)              │    │
│ │ schema,      │ Las 499,900 posts viejos están acá         │    │
│ │ posts recien │ pero NO se descargaron                     │    │
│ │tes, usuarios │                                            │    │
│ └──────────────┴───────────────────────────────────────────┘    │
│                                                                  │
│ La app ejecuta: SELECT * FROM posts ORDER BY created_at DESC     │
│                 LIMIT 100;                                       │
│                                                                  │
│ Turso necesita leer los índices + datos de los 100 posts más     │
│ recientes. Si esos posts están en páginas ya descargadas         │
│ (dentro de los primeros 512KB), la query es instantánea.         │
│ Si alguno cayó en un hueco, descarga solo esa página (4KB).      │
│                                                                  │
│ Resultado: ~600KB descargados en total (no 2GB).                 │
└─────────────────────────────────────────────────────────────────┘
```

**Día 1 — El usuario cierra la app:**

```
┌─────────────────────────────────────────────────────────────────┐
│ La app se cierra pero app.db NO se borra.                        │
│ Queda en disco:                                                  │
│   app.db       → ~600KB (páginas descargadas + huecos)           │
│   app-wal      → vacío o con pocos cambios locales               │
│   app-info     → metadata de sync (revisión "rev-001")           │
└─────────────────────────────────────────────────────────────────┘
```

**Día 2 — El usuario vuelve a abrir la app:**

```
┌─────────────────────────────────────────────────────────────────┐
│ PASO 1 — Conectar a la DB existente:                             │
│   Turso abre app.db (el mismo archivo del Día 1).                │
│   Todas las páginas descargadas ayer siguen ahí.                 │
│   La query LIMIT 100 lee datos locales (sin red). ✅             │
│                                                                  │
│ PASO 2 — Pull incremental:                                       │
│   POST /pull-updates { client_revision: "rev-001" }              │
│                                                                  │
│   El servidor compara: "rev-001" vs estado actual "rev-050".     │
│   En 24 horas se crearon 45 posts nuevos.                        │
│   Esos 45 posts modificaron ~8 páginas.                          │
│                                                                  │
│   El servidor envía SOLO esas 8 páginas (~32KB).                 │
│   NO envía los 500,000 posts viejos ni las 128 páginas           │
│   que ya descargaste ayer.                                       │
│                                                                  │
│ PASO 3 — Aplicar cambios:                                        │
│   Las 8 páginas se escriben al WAL local.                        │
│   Turso hace replay de cambios locales pendientes.               │
│                                                                  │
│ PASO 4 — Query local:                                            │
│   SELECT * FROM posts ORDER BY created_at DESC LIMIT 100;        │
│   Lee datos locales actualizados. Sin llamada HTTP. ✅           │
│                                                                  │
│ Resultado: ~32KB descargados. Feed actualizado en milisegundos.  │
└─────────────────────────────────────────────────────────────────┘
```

**Día 30 — El usuario abre la app después de un mes sin usarla:**

```
┌─────────────────────────────────────────────────────────────────┐
│ app.db sigue en disco con lo descargado el Día 1.                │
│                                                                  │
│ Pull incremental:                                                │
│   client_revision: "rev-001" (la última vez que sincronizó)      │
│   server_revision: "rev-890"                                     │
│                                                                  │
│   En 30 días: ~2,000 posts nuevos. Modificaron ~300 páginas.     │
│   El servidor envía ~300 páginas (~1.2MB).                       │
│                                                                  │
│   Sigue SIN enviar los 500,000 posts viejos.                     │
│                                                                  │
│ Resultado: ~1.2MB descargados en ~2 segundos.                    │
│ La query LIMIT 100 sigue siendo local e instantánea. ✅          │
└─────────────────────────────────────────────────────────────────┘
```

**Lo fundamental de este escenario:**

| Qué pasa | Respuesta |
|----------|-----------|
| ¿Se vuelve a descargar toda la DB al reabrir? | **No.** El archivo persiste. Solo se descargan cambios nuevos. |
| ¿El partial sync se aplica cada vez? | **No.** Partial sync es solo para el bootstrap inicial. Después es pull incremental. |
| ¿La query `LIMIT 100` toca la red? | **No.** La query se ejecuta 100% local contra el archivo `app.db`. |
| ¿Qué pasa si no abro la app en 6 meses? | Pull incremental trae solo las páginas modificadas desde tu última sync. Si son muchas, puede pesar, pero nunca se re-descarga la DB completa. |
| ¿Y si desinstalo y reinstalo? | Ahí sí: bootstrap desde cero (partial sync aplica de nuevo). |

### ¿Qué Es Una "Revisión" y Cómo Sabe El Servidor Qué Enviar?

En los ejemplos de arriba mencioné `client_revision: "rev-001"`. Esto es importante: **la revisión NO es un query SQL ni tiene nada que ver con `SELECT ... LIMIT 100`.** Son dos sistemas independientes.

**La revisión es un marcador interno de sync** — piensa en ello como el número de versión de un documento de Google Docs o el hash de un commit de Git. Le dice al servidor: "ya tengo todo hasta este punto, mándame solo lo nuevo".

```
┌─────────────────────────────────────────────────────────────────┐
│                DOS SISTEMAS SEPARADOS                            │
│                                                                  │
│  SISTEMA 1: Sync (entre cliente y servidor, automático)          │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ El motor de sync guarda en archivo app-info:              │    │
│  │   synced_revision: "rev-001"                             │    │
│  │                                                          │    │
│  │ Cuando hace pull:                                        │    │
│  │   POST /pull-updates                                     │    │
│  │   Body: { client_revision: "rev-001" }                   │    │
│  │                                                          │    │
│  │ El servidor responde:                                    │    │
│  │   "Aquí están todas las páginas modificadas desde        │    │
│  │    rev-001 hasta ahora (rev-050)"                        │    │
│  │   → envía ~8 páginas (32KB)                              │    │
│  │                                                          │    │
│  │ Después del pull, app-info se actualiza:                 │    │
│  │   synced_revision: "rev-050"                             │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  SISTEMA 2: Queries de la app (entre tu app y la DB local)       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Tu código de aplicación hace:                             │    │
│  │   db.query("SELECT * FROM posts ORDER BY created_at      │    │
│  │             DESC LIMIT 100")                             │    │
│  │                                                          │    │
│  │ Esto se ejecuta contra el archivo app.db local.           │    │
│  │ No habla con el servidor. No sabe de revisiones.          │    │
│  │ Solo lee lo que ya está en disco (sync ya lo trajo).      │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

**En resumen:**

| | Revisión de sync | Query SQL de la app |
|---|---|---|
| **Quién la usa** | El motor de sync internamente | Tu código de aplicación |
| **Para qué sirve** | Marcador: "¿qué cambios nuevos hay?" | Leer datos: "¿cuáles son los últimos 100 posts?" |
| **Va al servidor** | ✅ Sí, en cada pull | ❌ No, se ejecuta local |
| **Dónde se guarda** | Archivo `app-info` | — |
| **El servidor la usa para** | Saber qué páginas enviar | Nada (el servidor no ejecuta este SQL salvo en partial sync con `Query`) |

**En el ejemplo del feed:**

```
Día 2, el usuario abre la app:

1. PRIMERO el motor de sync hace pull:
   "Servidor, dame cambios desde rev-001 hasta hoy"
   → recibe 32KB de páginas, las aplica

2. DESPUÉS tu app ejecuta su query:
   "SELECT * FROM posts ORDER BY created_at DESC LIMIT 100"
   → lee los datos locales que YA fueron actualizados en el paso 1
   → devuelve los 100 posts más recientes (instantáneo, sin red)
```

Son dos pasos secuenciales: primero sync (trae datos nuevos), después query (lee datos locales). La revisión no aparece en el query ni el query en la revisión.

### ¿El Pull Trae Toda La Base de Datos o Solo un Subconjunto?

Esta es la pregunta central, y la respuesta es matizada.

**¿Trae toda la base de datos desde cero?**

**No.** El pull envía `client_revision: "rev-001"` y el servidor responde: "solo las páginas que cambiaron desde `rev-001` hasta hoy". Si la DB tiene 500,000 posts y desde tu última sync solo se crearon 45, el servidor envía ~8 páginas (32KB). No reenvía lo que ya tienes.

**¿Trae solo los datos que me interesan (un subconjunto)?**

**No.** El servidor envía **todas** las páginas modificadas en ese intervalo, sin filtrar por tabla, fila o query. Si desde `rev-001` alguien modificó la tabla `posts` Y la tabla `logs` Y la tabla `users`, recibes las páginas de las tres tablas, incluso si tu app solo muestra `posts`.

```
┌──────────────────────────────────────────────────────────────────┐
│  Imagina esto:                                                    │
│                                                                   │
│  Desde tu último sync (rev-001):                                  │
│    - 45 posts nuevos (tu app los muestra en el feed)              │
│    - 200 filas de logs del sistema (tu app NUNCA las muestra)     │
│    - 3 usuarios cambiaron su foto de perfil                       │
│                                                                   │
│  El pull te trae:                                                 │
│    Página 4:  contiene 5 posts nuevos + 3 logs                    │
│    Página 5:  contiene 8 posts nuevos + 12 logs                   │
│    Página 6:  contiene las 3 fotos de perfil                      │
│    Página 7:  contiene 15 posts nuevos + 50 logs                  │
│    Página 12: contiene 10 posts nuevos + 20 logs                  │
│    Página 18: contiene 7 posts nuevos + 115 logs                  │
│    ...                                                            │
│                                                                   │
│  NO puedes pedir: "solo mándame los posts, no los logs".          │
│  Las páginas mezclan datos de distintas tablas.                   │
│  Si una página tiene 1 post Y 10 logs, recibes los 10 logs.       │
└──────────────────────────────────────────────────────────────────┘
```

**¿Cuánto se descarga entonces? — Comparación realista:**

| Escenario | Datos en DB total | Datos nuevos desde último sync | Lo que se descarga en el pull |
|-----------|-------------------|-------------------------------|-------------------------------|
| Feed, abro cada hora | 2GB (500k posts) | 5 posts nuevos | ~1-3 páginas (~4-12KB) |
| Feed, abro cada día | 2GB (500k posts) | 200 posts + otros cambios | ~30-50 páginas (~120-200KB) |
| Feed, no abro en 1 mes | 2GB (500k posts) | 6000 posts + otros cambios | ~500-800 páginas (~2-3MB) |
| Chat, mensajes cada minuto | 100MB (50k msgs) | 300 mensajes nuevos | ~40-60 páginas (~160-240KB) |
| App colaborativa, 10 usuarios offline | 50MB | 500 cambios de 10 usuarios | ~80-120 páginas (~320-480KB) |

En todos los casos, **lo descargado es proporcional a cuánto cambió la DB, no al tamaño total de la DB.** Nunca se re-descarga la base completa si la revisión está al día.

**En resumen:**

| Pregunta | Respuesta |
|----------|-----------|
| ¿Re-descarga toda la DB? | No, solo las páginas modificadas desde tu última revisión |
| ¿Filtra por query SQL? | No, el pull no entiende queries de aplicación |
| ¿Filtra por tabla? | No, si una página tiene datos de varias tablas modificadas, vienen todos |
| ¿Puedo evitar descargar los logs? | Sí, con `tables_ignore` evitas que se **suban** (push), pero en el **pull** no puedes evitarlo |
| ¿Y si mi DB tiene 10GB? | Solo descargas lo que cambió desde tu último sync. Si abres cada hora, son KBs, no GBs |


### Resumen: Cuándo Usas Cada Concepto

| Situación | Concepto relevante |
|-----------|-------------------|
| "Quiero que mi tabla `logs` nunca salga del dispositivo" | Push lógico + `tables_ignore` |
| "Dos usuarios editan la misma tarea offline" | Pull físico + replay de cambios locales |
| "Mi DB pesa 500MB y quiero que abra rápido" | Partial sync (Prefix o Query) |
| "Quiero filtrar qué filas se comparten por tenant" | Push lógico + `use_transform` (pero pull recibe todo) |
| "Hice 100 cambios offline, quiero sync" | Push lógico (100 sentencias SQL al servidor) |
| "Otros hicieron 500 cambios, quiero recibirlos" | Pull físico (~50 páginas de 4KB descargadas) |

---

## 1. Modelo de Replicación: Multi-Primario (Active-Active)

Turso NO es master-replica tradicional. Es un sistema **peer-to-peer lógico** donde:

- **Todos los clientes son iguales**: cualquier nodo puede leer y escribir localmente sin coordinación previa.
- **Sincronización bidireccional**: cada cliente empuja sus cambios locales al remoto (`push`) y descarga cambios remotos hacia local (`pull`).
- **Resolución de conflictos optimista**: los cambios locales se revierten antes de aplicar cambios remotos, y luego se re-ejecutan encima del nuevo estado.
- **Sin líder designado**: no hay elección de líder, ni failover, ni promoción de réplica.

### Estados posibles de un cliente

| Estado | Descripción |
|--------|-------------|
| **Offline** | Lee y escribe localmente. Los cambios se acumulan en la tabla CDC y en el WAL local. |
| **Push** | Envía cambios locales al remoto como sentencias SQL lógicas. |
| **Pull** | Descarga cambios remotos, revierte trabajo local, aplica remoto, re-ejecuta local encima. |
| **Sync** | `push()` seguido de `pull()` — sincronización completa. |
| **Idle** | Sin cambios pendientes en ninguna dirección. |

---

## 2. Cómo Funciona la Sincronización

### 2.1 Captura de Cambios: CDC (Change Data Capture)

La base de cada instancia habilita CDC mediante el PRAGMA:

```sql
PRAGMA capture_data_changes_conn('full,turso_cdc');
```

Esto crea una tabla interna `turso_cdc` que captura **cada INSERT/UPDATE/DELETE** como una fila con:

| Columna | Descripción |
|---------|-------------|
| `change_id` | ID monotónico incremental |
| `change_time` | Timestamp Unix |
| `change_txn_id` | ID de transacción (v2) |
| `change_type` | -1=delete, 0=update, 1=insert, 2=commit |
| `table_name` | Nombre de la tabla afectada |
| `id` | RowID de la fila |
| `before` | Registro binario del estado anterior |
| `after` | Registro binario del nuevo estado |
| `updates` | Columnas modificadas (solo para updates) |

### 2.2 Flujo de Push (Empujar cambios locales al remoto)

```
1. Leer tabla turso_cdc local (DatabaseChangesIterator)
2. Para cada fila CDC:
   a. Saltar tablas en tables_ignore
   b. Saltar la tabla de bookkeeping de sync
   c. Convertir CDC → DatabaseTapeRowChange → sentencia SQL (DatabaseReplayGenerator)
   d. Opcionalmente aplicar transformación de usuario (keep/skip/rewrite)
3. Agrupar sentencias SQL en batches
4. Enviar batch vía HTTP POST /v2/pipeline (protocolo Hrana)
   con BEGIN IMMEDIATE ... COMMIT wrapper
5. Actualizar turso_sync_last_change_id en el remoto
```

**Granularidad de push**: a nivel de **fila individual**, transformada a sentencias SQL (INSERT/UPDATE/DELETE/UPSERT).

### 2.3 Flujo de Pull (Descargar cambios del remoto a local)

```
Fase 0 - Pull:
1. POST /pull-updates con client_revision actual
2. Servidor responde con protobuf (header + PageData páginas WAL)
3. Páginas se escriben a archivo temporal {db}-changes

Fases 1-6 - Apply:
4. Checkpoint pasivo del WAL local al archivo de base de datos
5. Abrir conexión revert-DB (WAL separado para rollback)
6. Revertir cambios locales desde el WAL principal después del watermark
7. Revertir cambios locales usando frames del revert-DB (deshacer trabajo local no-checkpointed)
8. Aplicar frames WAL remotos del archivo changes al WAL principal
9. Incrementar schema version para forzar re-preparación de statements
10. Leer pull_gen + last_change_id del estado remoto
11. Recolectar cambios locales ocurridos desde el último sync (de CDC)
12. Re-ejecutar cambios locales encima del estado remoto (replay)
13. Commit de la transacción combinada
```

### 2.4 Flujo de Bootstrap (Crear base de datos desde cero)

```
Legacy:
  GET /info → GET /export/{generation} → escribir a archivo

V1 (full):
  POST /pull-updates (selector vacío) → stream de páginas

V1 (partial sync - Prefix):
  POST /pull-updates (page_range_bitmap para primeros N bytes) → stream de páginas

V1 (partial sync - Query):
  POST /pull-updates (server_query_selector) → servidor elige páginas → stream
```

### 2.5 Protocolo de Red

| Endpoint | Propósito | Formato |
|----------|-----------|---------|
| `/pull-updates` | Pull de cambios (V1) | Protobuf |
| `/v2/pipeline` | Push de cambios (V1) | Hrana/JSON (SQL batches) |
| `/sync/{gen}/{start}/{end}` | Pull/Push WAL frames (legacy) | Binario WAL |
| `/export/{gen}` | Bootstrap full (legacy) | Binario DB |
| `/info` | Metadata del servidor | JSON |

---

## 3. Archivos por Base de Datos Sincronizada

Cada base de datos sincronizada mantiene estos archivos:

| Archivo | Propósito |
|---------|-----------|
| `{db}` | Archivo principal de la base de datos |
| `{db}-wal` | WAL principal (almacena cambios locales) |
| `{db}-wal-revert` | WAL de reversión (información de undo para cambios locales) |
| `{db}-info` | Metadatos de sync (JSON): client_unique_id, synced_revision, watermarks, timestamps |
| `{db}-changes` | Archivo temporal para frames WAL remotos descargados |

---

## 4. ¿Se Puede Sincronizar Solo Queries, Tablas o Filas?

### 4.1 Sincronización Selectiva de Tablas

**SÍ.** Se pueden excluir tablas específicas del push mediante:

```rust
DatabaseSyncEngineOpts {
    tables_ignore: vec!["temp_data".to_string(), "cache".to_string()],
    // ...
}
```

Las tablas en `tables_ignore` no se enviarán al remoto durante el push. Sin embargo, los cambios en esas tablas aún se capturan en CDC y permanecen locales.

### 4.2 Transformación de Cambios por Fila

**SÍ.** Se puede habilitar un hook de transformación que decide por cada fila:

```rust
DatabaseSyncEngineOpts {
    use_transform: true,
    // ...
}
```

El hook `DatabaseRowTransformResult` puede devolver:
- `Keep`: sincronizar la fila normalmente
- `Skip`: excluir esta fila del sync
- `Rewrite(DatabaseStatementReplay)`: reemplazar con una sentencia SQL diferente

### 4.3 Partial Sync (Sincronización Parcial / Lazy Loading)

**SÍ.** Se puede configurar sincronización parcial donde no se descarga toda la base de datos, sino solo las páginas necesarias bajo demanda:

```rust
DatabaseSyncEngineOpts {
    partial_sync_opts: Some(PartialSyncOpts {
        bootstrap_strategy: Some(PartialBootstrapStrategy::Prefix { length: 1048576 }),
        // o: PartialBootstrapStrategy::Query { query: "SELECT * FROM users".to_string() }
        segment_size: 131072,  // 128KB por defecto
        prefetch: true,
    }),
    // ...
}
```

**Estrategias de bootstrap parcial:**

| Estrategia | Descripción |
|------------|-------------|
| `Prefix { length }` | Descarga solo los primeros N bytes de la DB y el resto se carga lazy |
| `Query { query }` | El servidor ejecuta un SQL para determinar qué páginas incluir en el bootstrap |

**Cómo funciona lazy loading:**
1. El archivo de base de datos usa sparse files (huecos/`fallocate` con `FALLOC_FL_PUNCH_HOLE` en Linux)
2. Cuando una lectura encuentra un hueco (página no descargada), `LazyDatabaseStorage` detecta la página faltante
3. El motor hace una petición HTTP para descargar solo esa página
4. La página se escribe en el archivo, rellenando el hueco

### 4.4 ¿Sincronizar solo ciertas queries?

**NO directamente.** La sincronización no opera a nivel de queries individuales. Opera a nivel de cambios de datos (CDC). Sin embargo, el partial sync con `Query` permite especificar una query SQL para determinar qué datos iniciales descargar.

---

## 5. ¿Siempre Se Debe Sincronizar Toda la DB?

**NO necesariamente.** Hay tres mecanismos de granularidad:

| Mecanismo | Nivel | Qué controla |
|-----------|-------|--------------|
| `tables_ignore` | **Tabla** | Qué tablas NO se sincronizan |
| `use_transform` | **Fila** | Qué filas individuales se incluyen/excluyen/reescriben |
| `partial_sync_opts` | **Página** | Qué páginas de la DB se descargan inicialmente; el resto bajo demanda |

**Combinación de estrategias:**
- Puedes usar `partial_sync_opts` para hacer bootstrap solo de una parte de la DB y cargar el resto lazy
- Puedes usar `tables_ignore` para que ciertas tablas nunca salgan del cliente local
- Puedes usar `use_transform` para filtrar filas específicas (ej. datos sensibles, PII, etc.)

---

## 6. Arquitectura Interna del Motor

### 6.1 Componentes Clave

```
┌──────────────────────────────────────────────────────────┐
│                    Turso Cloud / Remote                   │
│  /pull-updates (protobuf)  /v2/pipeline (Hrana/JSON)     │
│  /sync/{gen}/{start}/{end}  /export/{gen}  /info         │
└──────┬──────────────────────────────────────┬────────────┘
       │           HTTP (SyncEngineIo)        │
       │                                      │
┌──────▼──────────────────────────────────────▼────────────┐
│               DatabaseSyncEngine<IO>                     │
│  ┌─────────────────────────────────────────────────┐    │
│  │  DatabaseTape (CDC capture)                      │    │
│  │  ├─ DatabaseChangesIterator (leer CDC)           │    │
│  │  ├─ DatabaseReplaySession (replay cambios)       │    │
│  │  └─ DatabaseWalSession (sesión WAL)              │    │
│  ├─────────────────────────────────────────────────┤    │
│  │  DatabaseReplayGenerator (CDC → SQL)             │    │
│  ├─────────────────────────────────────────────────┤    │
│  │  WalSession (WAL bajo nivel: begin/read/insert)  │    │
│  ├─────────────────────────────────────────────────┤    │
│  │  LazyDatabaseStorage (partial sync storage)      │    │
│  └─────────────────────────────────────────────────┘    │
│  DatabaseMetadata ({db}-info): estado de sync            │
└──────────────────────────────────────────────────────────┘
       │
┌──────▼───────────────────────────────────────────────────┐
│              turso_sync_sdk_kit (C ABI)                  │
│  ┌─────────────────────────────────────────────────┐    │
│  │  TursoDatabaseSync (API alto nivel Rust)         │    │
│  │  SyncEngineIoQueue (cola de IO asíncrona)        │    │
│  │  Bindings C (.h) → Rust/WASM/Go/Python/JS/.NET  │    │
│  └─────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

### 6.2 Máquina de Estados con Corutinas

El motor usa **corutinas cooperativas** (`genawaiter`) en lugar de async/await de Rust. El patrón es:

```rust
Coroutine → yield SyncEngineIoResult::IO → caller drives IO → resume coroutine
```

Esto permite que el motor sea portable a entornos sin runtime async (WASM, embebidos) y que el caller controle exactamente cuándo se ejecuta IO.

### 6.3 Estados del Motor de Sync

Cada operación (push, pull, bootstrap, checkpoint) es una corutina que avanza por fases numeradas:

| Operación | Fases |
|-----------|-------|
| **Push** | 0→1: Leer CDC → enviar batches → commit |
| **Pull** | 0: Pull frames, 1-6: Rollback → Apply → Replay → Commit |
| **Bootstrap** | 0: Descargar → escribir → inicializar metadata |
| **Sync** | Push completo → Pull completo |

### 6.4 Metadata de Sync (`{db}-info`)

```json
{
  "version": "v1",
  "client_unique_id": "turso-client-abc123",
  "synced_revision": {
    "V1": { "revision": "abc-def-ghi" }
  },
  "revert_since_wal_salt": [1, 2],
  "revert_since_wal_watermark": 42,
  "last_pushed_change_id_hint": 100,
  "last_pushed_pull_gen_hint": 5,
  "last_pull_unix_time": 1718234567,
  "last_push_unix_time": 1718234560,
  "saved_configuration": {
    "remote_url": "libsql://my-db.turso.io"
  }
}
```

---

## 7. Opciones de Configuración Completas

| Opción | Tipo | Descripción |
|--------|------|-------------|
| `remote_url` | `Option<String>` | URL del remoto (`libsql://...`, `https://...`, `http://...`) |
| `client_name` | `String` | Prefijo para el ID único del cliente |
| `tables_ignore` | `Vec<String>` | Tablas a excluir del push |
| `use_transform` | `bool` | Habilitar transformación de filas (keep/skip/rewrite) |
| `wal_pull_batch_size` | `u64` | Tamaño de batch para pull legacy |
| `long_poll_timeout` | `Option<Duration>` | Timeout de long polling para esperar cambios nuevos |
| `protocol_version_hint` | Enum | `Legacy` o `V1` |
| `bootstrap_if_empty` | `bool` | Si true, bootstrap desde remoto si no hay DB local |
| `reserved_bytes` | `usize` | Bytes reservados por página (para encriptación remota) |
| `partial_sync_opts` | `Option<PartialSyncOpts>` | Sincronización parcial/lazy |
| `remote_encryption_key` | `Option<String>` | Clave de encriptación base64 para Turso Cloud |
| `push_operations_threshold` | `Option<usize>` | Partir push en batches de N+ operaciones |
| `pull_bytes_threshold` | `Option<usize>` | Chunkear descarga bootstrap en requests de N+ bytes |

---

## 8. Bindings y Soporte Multi-Lenguaje

El crate `turso_sync_sdk_kit` expone una API C (`turso_sync.h`) que sirve como base para todos los bindings:

| Lenguaje | Directorio | Tecnología |
|----------|-----------|------------|
| **Rust** | `bindings/rust/` | Nativo |
| **C** | `sync/sdk-kit/turso_sync.h` | ABI C |
| **Python** | `bindings/python/` | PyO3 |
| **JavaScript/Node** | `bindings/js/` | NAPI |
| **Go** | `bindings/go/` | CGO |
| **Java/Kotlin** | `bindings/java/` | JNI |
| **.NET** | `bindings/dotnet/` | P/Invoke |
| **WASM** | `bindings/wasm/` | Wasm-bindgen |

---

## 9. Resolución de Conflictos

El sistema usa un enfoque **optimista con replay**:

1. Antes de aplicar cambios remotos, los cambios locales se **revienen** (se deshacen)
2. Se aplican los cambios remotos encima del estado base
3. Los cambios locales se **re-ejecutan** (replay) encima del nuevo estado
4. Si `remote_pull_gen == local_pull_gen`, solo se re-ejecutan cambios posteriores al último `change_id` conocido por el remoto

Esto significa que **el último escritor gana** en caso de conflicto sobre la misma fila.

---

## 10. Limitaciones y Consideraciones

1. **Sin resolución de conflictos semántica**: no hay merge a nivel de aplicación. Si dos clientes modifican la misma fila, el último en sincronizar sobrescribe.
2. **Transacciones no distribuidas**: cada transacción es local. No hay commit atómico跨多个节点.
3. **WAL debe ser checkpointed**: si el WAL crece mucho, el rendimiento de sync se degrada.
4. **Partial sync es por página**: 4KB de granularidad mínima. No se puede bajar solo una celda específica de una página.
5. **Tablas ignoradas son solo para push**: `tables_ignore` evita que se suban, pero no evita que se descarguen del remoto. Para filtrar en ambas direcciones, usar `use_transform`.
6. **DDL es sincronizado**: CREATE/ALTER/DROP se capturan como cambios en `sqlite_schema` y se replican con `IF NOT EXISTS` para idempotencia.

---

## 11. Referencia de Archivos

| Archivo | Propósito |
|---------|-----------|
| `sync/engine/src/types.rs` | Tipos core: DatabaseChange, DatabaseMetadata, PartialSyncOpts |
| `sync/engine/src/database_sync_engine.rs` | Motor principal: push, pull, bootstrap, checkpoint |
| `sync/engine/src/database_sync_operations.rs` | Operaciones HTTP y WAL de bajo nivel |
| `sync/engine/src/database_tape.rs` | Abstracción CDC: captura, iteración, replay |
| `sync/engine/src/database_replay_generator.rs` | Conversión CDC → SQL |
| `sync/engine/src/database_sync_lazy_storage.rs` | Partial sync: carga lazy de páginas |
| `sync/engine/src/server_proto.rs` | Tipos del protocolo wire (protobuf + Hrana) |
| `sync/engine/src/wal_session.rs` | Manipulación WAL de bajo nivel |
| `sync/sdk-kit/src/rsapi.rs` | API Rust de alto nivel |
| `sync/sdk-kit/turso_sync.h` | API pública en C |
| `cli/sync_server.rs` | Servidor sync embebido para testing local |
