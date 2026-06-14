# Arquitectura Multi-Tenant (Multi-Org) con Turso

## Resumen de la Recomendación

**Usa una base de datos por organización.** Es la única opción viable para aislamiento de datos con el modelo de sincronización de Turso.

> Para entender qué partes son open source vs qué requiere Turso Cloud, consulta [self-hosting-es.md](self-hosting-es.md).

---

## 1. Por Qué NO Debes Usar una Columna `org_id` en una Sola DB

### El Problema Fundamental: El Pull Descarga Todo

El protocolo de sincronización de Turso tiene una asimetría crítica:

| Dirección | ¿Se puede filtrar? | Mecanismo |
|-----------|-------------------|-----------|
| **Push** (local → remoto) | ✅ Sí | `tables_ignore`, `use_transform` |
| **Pull** (remoto → local) | ❌ No | No existe ningún filtro row-level o tenant-level |

Cuando un cliente hace **pull** (`POST /pull-updates`), el servidor envía **todas** las páginas WAL con **todos** los cambios confirmados. El `PullUpdatesReqProtoBody` solo permite:

- `server_pages_selector` — RoaringBitmap de páginas (filtro a nivel de página, no de fila)
- `server_query_selector` — SQL para seleccionar páginas en bootstrap parcial (no para filtrar filas en pulls incrementales)
- `long_poll_timeout_ms` — timeout de espera

**No hay campo para identificar al tenant ni para filtrar filas por `org_id`.**

### Consecuencia: Fuga de Datos Inevitable

```
Cliente Org A (solo debería ver datos de Org A)
         │
         │  POST /pull-updates (client_revision="abc")
         ▼
┌────────────────────┐
│   Servidor Remoto  │
│                    │
│  WAL contiene:     │
│  - INSERT org_a... │  ← Todos los cambios de todas las orgs
│  - UPDATE org_b... │    se envían en las mismas páginas WAL
│  - DELETE org_c... │
│  - INSERT org_a... │
└────────────────────┘
         │
         │  Respuesta: PageData con TODAS las páginas WAL
         ▼
Cliente Org A recibe datos de Org A, Org B, Org C, ...
```

Aunque uses `use_transform` para evitar que Org A **suba** datos de Org B, Org A **descargará** datos de Org B en cada pull. No hay forma de evitarlo con el protocolo actual.

---

## 2. Opción Recomendada: Una DB por Organización

```
┌──────────────────────────────────────────────────┐
│                  Turso Cloud                      │
│                                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │ org_a.db │  │ org_b.db │  │ org_c.db │  ...   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘        │
│       │              │              │              │
└───────┼──────────────┼──────────────┼──────────────┘
        │              │              │
   ┌────▼────┐    ┌────▼────┐    ┌────▼────┐
   │Org A    │    │Org B    │    │Org C    │
   │Cliente  │    │Cliente  │    │Cliente  │
   │(solo    │    │(solo    │    │(solo    │
   │org_a.db)│    │org_b.db)│    │org_c.db)│
   └─────────┘    └─────────┘    └─────────┘
```

### Ventajas

| Ventaja | Descripción |
|---------|-------------|
| **Aislamiento total** | Cada org solo sincroniza su DB; imposible filtrar datos entre orgs |
| **Seguridad** | No hay riesgo de bug o config errónea que exponga datos entre tenants |
| **Escalabilidad independiente** | Cada org puede tener su propio plan, cuotas, retención |
| **Operaciones aisladas** | Backup, restore, schema migration por org sin afectar a otros |
| **Modelo nativo de Turso** | Así está diseñado el producto; máxima compatibilidad con el ecosistema |

### Desventajas y Mitigaciones

| Desventaja | Mitigación |
|------------|------------|
| Muchas DBs que gestionar | Automatizar con API de Turso (crear/eliminar DBs programáticamente) |
| Sin queries cross-org directas | DB separada de analytics/agregación (ver sección 4) |
| Schema migrations × N orgs | Pipeline de migración automatizado que itera sobre todas las DBs |
| Costo potencial de DBs vacías | Turso cobra por uso; DBs sin tráfico tienen costo mínimo |

---

## 3. Opción Híbrida: DB por Org + DB Compartida (No Sincronizada)

Si necesitas queries cross-org (ej. analytics, admin panel), puedes complementar con una DB compartida **no sincronizada**:

```
┌─────────────────────────────────────────────────────┐
│                   Turso Cloud                        │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │ org_a.db │  │ org_b.db │  │ analytics.db      │  │
│  │(sync ON) │  │(sync ON) │  │(sync OFF - solo   │  │
│  │          │  │          │  │ servidor central)  │  │
│  └──────────┘  └──────────┘  └──────────────────┘  │
│                                                      │
└──────────────────────────────────────────────────────┘
        │              │                                  
   ┌────▼────┐    ┌────▼────┐                            
   │Org A    │    │Org B    │   El admin panel / ETL    
   │Cliente  │    │Cliente  │   consulta analytics.db   
   └─────────┘    └─────────┘   directamente (sin sync) 
```

- **org_N.db**: sincronizadas con los clientes de cada org (partial sync opcional)
- **analytics.db**: solo existe en el servidor, sin sincronización. Un proceso ETL o triggers materializan datos agregados desde las DBs de orgs.

---

## 4. Opción Híbrida: DB por Org con Partial Sync

Para orgs que necesitan acceso offline/lento, puedes reducir la cantidad de datos que cada cliente descarga inicialmente:

```rust
DatabaseSyncEngineOpts {
    partial_sync_opts: Some(PartialSyncOpts {
        bootstrap_strategy: Some(PartialBootstrapStrategy::Query {
            query: "SELECT 1".to_string(), // bootstrap mínimo
        }),
        segment_size: 131072,
        prefetch: false, // carga bajo demanda, no en background
    }),
    // ...
}
```

Esto hace que el cliente solo descargue las páginas que realmente lee, reduciendo el bootstrap inicial. Pero **no es un mecanismo de seguridad** — el cliente puede eventualmente leer (y por tanto descargar) toda la DB de su org. Para aislamiento entre orgs, solo sirve la DB separada.

---

## 5. Comparativa de Opciones

| Criterio | Una DB con `org_id` | Una DB por org | DB por org + analytics.db |
|----------|---------------------|----------------|---------------------------|
| **Aislamiento de datos** | ❌ Imposible | ✅ Total | ✅ Total |
| **Queries cross-org** | ✅ Fácil (misma DB) | ❌ Imposible directo | ✅ Vía analytics.db |
| **Complejidad operativa** | Baja (1 schema) | Media (N schemas) | Alta (N+1 schemas, ETL) |
| **Escalabilidad** | Baja (todos compiten) | Alta (independiente) | Alta |
| **Compatibilidad con Turso** | ❌ El protocolo no lo permite | ✅ Nativo | ✅ Nativo |
| **Costo** | 1 DB | N DBs | N+1 DBs |

---

## 6. Implementación: Ejemplo de Gestión Multi-Tenant

### 6.1 Resolución de DB por Tenant

```rust
struct TenantDbManager {
    data_dir: PathBuf,
}

impl TenantDbManager {
    fn db_name_for_org(&self, org_id: &str) -> String {
        format!("org_{}", org_id)
    }

    fn db_path_for_org(&self, org_id: &str) -> PathBuf {
        self.data_dir.join(self.db_name_for_org(org_id))
    }

    async fn get_or_create_db(&self, org_id: &str) -> Result<Database> {
        let path = self.db_path_for_org(org_id);
        if !path.exists() {
            std::fs::File::create(&path)?;
            self.run_migrations(org_id).await?;
        }
        Ok(Database::open(path)?)
    }
}
```

### 6.2 Configuración del Cliente de Sync

```c
// Cada org abre SOLO su propia DB
turso_sync_config_t config = {
    .path = "/data/org_acme-corp.db",
    .remote_url = "libsql://org_acme-corp.turso.io",   // o tu servidor sync
    .client_name = "org_acme-corp",
    .bootstrap_if_empty = true,
};
```

### 6.3 Migraciones Automatizadas

```bash
for db_file in /data/org_*.db; do
  tursodb "$db_file" < migrations/0001_init.sql
done
```

---

## 7. Conclusión

| Si necesitas... | Usa... |
|-----------------|--------|
| Aislamiento de datos entre orgs | **Una DB por org** (obligatorio) |
| Queries cross-org (admin, analytics) | DB adicional no sincronizada + ETL |
| Reducir datos descargados por cliente | `partial_sync_opts` con `Query` o `Prefix` |
| Filtrar qué tablas se sincronizan | `tables_ignore` (solo afecta push) |
| Transformar/filtrar filas en push | `use_transform` con keep/skip/rewrite |

**La arquitectura recomendada es: una base de datos por tenant/organización.** No es una limitación arbitraria — es una consecuencia directa de cómo funciona el protocolo de sincronización (push filtrable, pull no filtrable). Cualquier intento de usar una sola DB con columna `org_id` resultará en que todos los clientes descarguen los datos de todos los tenants en cada pull.

Para opciones de self-hosting vs Turso Cloud, consulta [self-hosting-es.md](self-hosting-es.md).

