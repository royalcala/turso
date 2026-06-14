# Migraciones de Schema en Turso

## Resumen

Las migraciones en Turso **se replican automáticamente** como cualquier otro cambio de datos, vía CDC. Sin embargo, requieren estrategias específicas por ser un sistema multi-primario (active-active). **La recomendación es aplicar migraciones desde un solo nodo** y dejar que el sync las propague.

---

## 1. Cómo Funcionan las Migraciones en Turso

### 1.1 Las Migraciones Son DDL, y el DDL Son Datos

Cuando ejecutas `CREATE TABLE`, `ALTER TABLE`, o `DROP TABLE`, SQLite modifica la tabla interna `sqlite_schema`. Turso tiene CDC habilitado (`PRAGMA capture_data_changes_conn`), que **captura cada escritura a `sqlite_schema`** como una fila en `turso_cdc`:

| Operación DDL | Qué se captura en CDC | Tipo CDC |
|---------------|----------------------|----------|
| `CREATE TABLE t(...)` | `INSERT` en `sqlite_schema` (type=table, name=t, sql=CREATE...) | Insert |
| `ALTER TABLE t ADD COLUMN z` | `UPDATE` en `sqlite_schema` (nuevo sql con la columna) | Update |
| `DROP TABLE t` | `DELETE` en `sqlite_schema` (type=table, name=t) | Delete |
| `CREATE INDEX idx ON t(x)` | `INSERT` en `sqlite_schema` (type=index, name=idx) | Insert |

### 1.2 Flujo de una Migración

```
Cliente A ejecuta: ALTER TABLE users ADD COLUMN phone TEXT
         │
         ▼
┌─────────────────────────────────────────────────────┐
│ 1. CDC captura UPDATE en sqlite_schema (tabla users) │
│ 2. Push al servidor: envía la sentencia SQL          │
│    "ALTER TABLE users ADD COLUMN phone TEXT"          │
│ 3. Servidor ejecuta → columna existe en remoto       │
└─────────────────────────────────────────────────────┘
         │
         ▼  (Cliente B hace pull)
┌─────────────────────────────────────────────────────┐
│ 4. Cliente B descarga frames WAL con el cambio       │
│ 5. Revierte sus cambios locales                      │
│ 6. Aplica frames remotos (schema remoto)             │
│ 7. Bump schema_version → fuerza re-preparación       │
│ 8. Re-ejecuta sus cambios locales encima             │
│ 9. Schema sincronizado en Cliente B                  │
└─────────────────────────────────────────────────────┘
```

### 1.3 Idempotencia Automática

Turso maneja idempotencia para evitar errores si una migración se ejecuta dos veces:

| DDL | Mecanismo de Idempotencia |
|-----|--------------------------|
| `CREATE TABLE/INDEX/VIEW/TRIGGER` | El replay generator reescribe la sentencia con `IF NOT EXISTS` |
| `ALTER TABLE ADD COLUMN` | Verifica `pragma_table_info` — si la columna ya existe, la omite |
| `DROP TABLE/INDEX/VIEW` | Sin idempotencia automática — debes manejarlo en tu migración |

**Archivo clave:** `sync/engine/src/database_replay_generator.rs`

---

## 2. ¿Dónde Aplicar las Migraciones?

### Opción A: Desde un Solo Nodo (Recomendado)

```
┌─────────────────┐
│  Nodo Migrador   │  ← SOLO este nodo aplica migraciones
│  (CI/CD o admin) │
└────────┬────────┘
         │ ALTER TABLE users ADD COLUMN phone TEXT
         ▼
┌─────────────────┐     sync     ┌─────────────────┐
│   Servidor       │─────────────│  Cliente A       │
│   (remoto)       │             │  (recibe schema) │
└─────────────────┘             └─────────────────┘
         │     sync
         ▼
┌─────────────────┐
│  Cliente B       │
│  (recibe schema) │
└─────────────────┘
```

**Ventajas:**
- Sin conflictos entre versiones de migración
- Control total del orden de migraciones
- Rollback claro (solo un autor)

**Desventajas:**
- El nodo migrador debe tener conectividad para hacer push
- Necesitas un proceso de deploy que ejecute migraciones antes de desplegar código nuevo

### Opción B: Cualquier Nodo Puede Migrar (No Recomendado)

```
Cliente A: ALTER TABLE users ADD COLUMN phone TEXT
Cliente B: ALTER TABLE users ADD COLUMN email TEXT
         │                              │
         ▼                              ▼
    ┌─────────────────────────────────────────┐
    │  Ambos cambios llegan al servidor        │
    │  → Se ejecutan en orden de llegada       │
    │  → Ambos clientes reciben ambos cambios  │
    │  → El schema converge                    │
    └─────────────────────────────────────────┘
```

**Ventajas:**
- Sin punto único de migración
- Flexibilidad total

**Desventajas:**
- Riesgo de conflictos si dos clientes migran la misma tabla con cambios incompatibles
- Orden de aplicación impredecible
- Difícil hacer rollback
- Sin `IF NOT EXISTS` para DROP — puede fallar

### Opción C: Migraciones por Coordinador (Avanzado)

```
┌────────────────────┐
│  Coordinador         │
│  - Liderazgo (lease) │
│  - Solo el líder     │
│    aplica migraciones│
└────────┬───────────┘
         │
    ┌────▼────┐
    │ ¿Soy    │
    │ líder?  │
    └────┬────┘
     Sí  │  No
    ┌────▼────┐   ┌─────────┐
    │ Aplicar │   │ Esperar │
    │ migración│   │ sync    │
    └─────────┘   └─────────┘
```

Útil si quieres alta disponibilidad sin punto fijo de migración, pero añade complejidad.

---

## 3. Estrategia Recomendada: Single Migration Author

### 3.1 Principios

1. **Un solo autor**: migraciones solo desde CI/CD o un servicio de migración
2. **Idempotentes**: toda migración debe poder ejecutarse múltiples veces sin error
3. **Orden estricto**: las migraciones se aplican en secuencia, como en cualquier proyecto SQL
4. **Push inmediato**: después de migrar, hacer push para propagar a todos los clientes
5. **Código tolerante**: el código de aplicación debe funcionar con el schema anterior Y el nuevo durante la transición

### 3.2 Flujo de Deploy

```
┌──────────────────────────────────────────────────────┐
│  1. CI/CD ejecuta migración en el servidor remoto     │
│     (conexión directa, sin cliente sync)              │
│                                                       │
│  2. Migración es idempotente:                         │
│     ALTER TABLE users ADD COLUMN IF NOT EXISTS phone  │
│     (o verificando existencia antes)                  │
│                                                       │
│  3. Todos los clientes hacen pull → reciben schema    │
│                                                       │
│  4. Schema version se incrementa →                    │
│     prepared statements se recompilan                 │
│                                                       │
│  5. Código nuevo se despliega (ya sabe del schema)    │
└──────────────────────────────────────────────────────┘
```

### 3.3 Implementación de Ejemplo

```rust
struct MigrationRunner {
    db: Database,
}

impl MigrationRunner {
    async fn run_migration(&self, sql: &str) -> Result<()> {
        self.db.execute_batch(sql)?;
        Ok(())
    }

    async fn migrate_v1(&self) -> Result<()> {
        self.run_migration("CREATE TABLE IF NOT EXISTS migrations (version INTEGER PRIMARY KEY, applied_at TEXT)").await?;
        self.run_migration("CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, name TEXT, email TEXT)").await?;
        self.run_migration("INSERT OR IGNORE INTO migrations VALUES (1, datetime('now'))").await?;
        Ok(())
    }

    async fn migrate_v2(&self) -> Result<()> {
        let has_column = self.db
            .query("SELECT COUNT(*) FROM pragma_table_info('users') WHERE name = 'phone'")?
            .get::<i64>(0)? > 0;

        if !has_column {
            self.db.execute("ALTER TABLE users ADD COLUMN phone TEXT")?;
        }
        self.db.execute("INSERT OR IGNORE INTO migrations VALUES (2, datetime('now'))")?;
        Ok(())
    }
}
```

---

## 4. Idempotencia: Cómo Escribir Migraciones Seguras

### 4.1 CREATE — Usa IF NOT EXISTS

```sql
-- ✅ Seguro (idempotente)
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY,
    name TEXT
);

CREATE INDEX IF NOT EXISTS idx_users_name ON users(name);

CREATE VIEW IF NOT EXISTS active_users AS
    SELECT * FROM users WHERE active = 1;
```

### 4.2 ALTER TABLE ADD COLUMN — Verifica Primero

```sql
-- ✅ Seguro (idempotente con verificación)
-- En código:
SELECT COUNT(*) FROM pragma_table_info('users') WHERE name = 'phone';
-- Si count = 0:
ALTER TABLE users ADD COLUMN phone TEXT;
```

Turso ya maneja esto automáticamente durante el **replay** en sync, pero en tus propias migraciones manuales debes hacerlo explícitamente.

### 4.3 DROP — Verifica Existencia

```sql
-- ✅ Seguro (idempotente)
-- En código:
SELECT COUNT(*) FROM sqlite_schema WHERE type = 'table' AND name = 'old_table';
-- Si count = 1:
DROP TABLE old_table;

-- O usa una tabla de tracking de migraciones:
-- Solo ejecutar si la versión de migración no se ha aplicado
```

### 4.4 Tabla de Tracking de Migraciones

```sql
CREATE TABLE IF NOT EXISTS _migrations (
    version INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    applied_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

Cada migración verifica esta tabla antes de ejecutarse:

```rust
fn migration_already_applied(db: &Database, version: i64) -> bool {
    db.query("SELECT 1 FROM _migrations WHERE version = ?", params![version])
        .map(|rows| rows.count() > 0)
        .unwrap_or(false)
}

fn apply_migration(db: &Database, version: i64, name: &str, sql: &str) -> Result<()> {
    if migration_already_applied(db, version) {
        println!("Migration {version} ({name}) already applied, skipping");
        return Ok(());
    }
    db.execute_batch(sql)?;
    db.execute("INSERT INTO _migrations (version, name) VALUES (?, ?)", params![version, name])?;
    Ok(())
}
```

---

## 5. Conflictos de Migraciones

### 5.1 Conflictos Manejados Automáticamente

| Escenario | Resultado |
|-----------|-----------|
| Dos clientes crean la misma tabla | ✅ Segunda ejecución es no-op (`IF NOT EXISTS`) |
| Dos clientes añaden la misma columna | ✅ Segunda ejecución es no-op (verifica `pragma_table_info`) |
| Un cliente crea tabla, otro hace pull | ✅ Recibe el CREATE con `IF NOT EXISTS` en el replay |

### 5.2 Conflictos NO Manejados — Debes Evitarlos

| Escenario | Riesgo | Solución |
|-----------|--------|----------|
| Dos clientes añaden columnas con diferente tipo (`z TEXT` vs `z INTEGER`) | El primero que llegue gana; el segundo se ignora silenciosamente. Schema no coincide con lo esperado en un cliente | Single migration author |
| `DROP TABLE` concurrente con escrituras | La tabla se elimina mientras otro cliente escribe en ella | No hacer DROP en producción activa; usar soft-delete o migraciones coordinadas |
| `ALTER TABLE RENAME COLUMN` | No tiene idempotencia automática | Verificar existencia antes; single author |

---

## 6. Migraciones en Arquitectura Multi-Org

Si sigues la recomendación de [una DB por organización](multi-tenant-es.md), cada org tiene su propia base de datos. Las migraciones deben aplicarse a todas ellas.

### 6.1 Migración en Paralelo

```rust
async fn migrate_all_orgs(orgs: &[String], migration_sql: &str) -> Result<()> {
    let tasks: Vec<_> = orgs.iter().map(|org_id| {
        let db_path = format!("/data/org_{}.db", org_id);
        let sql = migration_sql.to_string();
        tokio::spawn(async move {
            let db = Database::open(&db_path)?;
            db.execute_batch(&sql)?;
            Ok::<_, Error>(())
        })
    }).collect();

    for task in tasks {
        task.await??;
    }
    Ok(())
}
```

### 6.2 Con Turso Cloud

```bash
# Listar todas las DBs de org y migrar
turso db list --filter "org_*" | while read db; do
  echo "Migrando $db..."
  turso db shell "$db" < migrations/0002_add_phone.sql
done
```

### 6.3 Con Self-Hosting

```bash
for db in /data/org_*.db; do
  echo "Migrando $db..."
  tursodb "$db" < migrations/0002_add_phone.sql
done
```

---

## 7. Resumen de Opciones

| Estrategia | Esfuerzo | Riesgo de Conflictos | Recomendado para |
|------------|----------|---------------------|------------------|
| **Single author (CI/CD)** | Bajo | Mínimo | ✅ La mayoría de los casos |
| **Cualquier nodo** | Bajo | Alto (DDL conflictivo) | ❌ No recomendado |
| **Coordinador con lease** | Alto | Bajo | HA estricta con multi-writer |

---

## 8. Checklist de Migración

Antes de aplicar una migración en producción:

- [ ] La migración es idempotente (puede ejecutarse N veces)
- [ ] Los CREATE usan `IF NOT EXISTS`
- [ ] Los ALTER verifican existencia de columna
- [ ] Los DROP verifican existencia del objeto
- [ ] Se usa tabla `_migrations` para tracking
- [ ] El código de aplicación tolera el schema anterior durante la transición
- [ ] La migración se probó en un entorno de staging con sync activo
- [ ] Si es multi-org, el script itera sobre todas las DBs
- [ ] Se hace push inmediato después de migrar para propagar a clientes

---

## 9. Conclusión

**Las migraciones se replican automáticamente** vía CDC — no necesitas aplicarlas en cada cliente manualmente. **Aplica migraciones desde un solo punto** (CI/CD, servidor de migración), asegúrate de que sean **idempotentes**, y deja que el sync propague los cambios de schema al resto de réplicas. Turso incrementa el `schema_cookie` automáticamente tras cada pull, forzando la recompilación de prepared statements.
