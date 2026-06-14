# Self-Hosting vs Turso Cloud: ¿Qué es Open Source y Qué No?

## Resumen

El motor de base de datos (`turso_core`) y el cliente de sync son **100% open source y totalmente funcionales**. Sin embargo, el servidor sync completo (auth, multi-tenancy, todos los endpoints) y la API de administración **no están en este repositorio** — son parte de Turso Cloud (pago).

---

## 1. Estructura del Repositorio

Este repositorio (`turso`) es una reescritura de SQLite en Rust. Contiene ~45 crates. **No es el mismo proyecto que `libsql`** (el fork de SQLite en C).

| Proyecto | Lenguaje | Descripción |
|----------|----------|-------------|
| `turso` (este repo) | Rust | Database engine + cliente sync + bindings |
| `libsql` | C | Fork de SQLite con modificaciones mínimas. Incluía `sqld` (servidor sync completo) |

Son dos proyectos distintos con propósitos diferentes.

---

## 2. Lo que SÍ es Open Source

### Motor de Base de Datos (`turso_core`)

El núcleo de la base de datos es completamente funcional sin dependencias externas:

| Módulo | Ubicación | Funcionalidad |
|--------|-----------|---------------|
| B-tree / Páginas | `core/storage/btree.rs` | Formato compatible con SQLite |
| WAL / Durabilidad | `core/storage/wal.rs` | Write-ahead log, checkpointing |
| MVCC | `core/mvcc/` | Concurrencia multiversión (experimental) |
| Ejecutor VDBE | `core/vdbe/execute.rs` | Intérprete de bytecode (12k LOC) |
| Compilador SQL | `core/translate/` | AST → bytecode, optimizador |
| CDC | `core/` + `sync/engine/src/database_tape.rs` | `PRAGMA capture_data_changes_conn` |
| FTS (Full Text Search) | `core/` | Búsqueda texto completo |
| Encriptación | `core/` + `perf/encryption/` | Encriptación a nivel de página |
| Extensiones | `extensions/` | crypto, regexp, csv, fuzzy, ipaddr, vector |

```bash
# El motor se puede usar como biblioteca embebida
cargo build
# O vía CLI interactiva
cargo run -q --bin tursodb -- -q
```

### Cliente de Sync (`turso_sync_engine`)

El motor de sincronización es una biblioteca cliente que se conecta a cualquier servidor compatible:

| Funcionalidad | Ubicación |
|---------------|-----------|
| Push de cambios locales | `sync/engine/src/database_sync_engine.rs` |
| Pull de cambios remotos | `sync/engine/src/database_sync_engine.rs` |
| Bootstrap (crear DB desde cero) | `sync/engine/src/database_sync_operations.rs` |
| CDC capture + replay | `sync/engine/src/database_tape.rs` |
| Partial sync (lazy loading) | `sync/engine/src/database_sync_lazy_storage.rs` |
| Protocolo wire V1 | `sync/engine/src/server_proto.rs` |

### SDK y Bindings Multi-Lenguaje

| Lenguaje | Directorio | Tecnología |
|----------|-----------|------------|
| Rust | `bindings/rust/` | Nativo |
| C | `sync/sdk-kit/turso_sync.h` | ABI C |
| Python | `bindings/python/` | PyO3 |
| JavaScript/Node | `bindings/js/` | NAPI |
| Go | `bindings/go/` | CGO |
| Java/Kotlin | `bindings/java/` | JNI |
| .NET | `bindings/dotnet/` | P/Invoke |
| WASM | `bindings/wasm/` | Wasm-bindgen |

Todos los bindings incluyen soporte para sync.

### Testing y Herramientas

| Herramienta | Descripción |
|-------------|-------------|
| `tursodb` CLI | REPL SQL + MCP server + sync server de testing |
| `make test` | TCL compat + sqlite3 + extensions + MVCC |
| `cargo test` | Tests unitarios y de integración Rust |
| Simuladores | Fault injection, concurrent, differential oracle |
| `scripts/diff.sh` | Comparar sqlite3 vs tursodb |

---

## 3. Lo que NO es Open Source (Solo en Turso Cloud)

### Servidor Sync Completo

El repositorio incluye `tursodb --sync-server` (`cli/sync_server.rs`, 703 líneas), pero es un **fixture de testing**, no un servidor production-grade. Implementa solo 2 de 6 endpoints:

| Endpoint | Implementado en `sync_server.rs` | Necesario para sync completo |
|----------|----------------------------------|------------------------------|
| `POST /v2/pipeline` | ✅ Sí (Hrana/JSON) | Push de cambios lógicos |
| `POST /pull-updates` | ⚠️ Solo raw encoding (sin zstd) | Pull de cambios |
| `GET /sync/{gen}/{start}/{end}` | ❌ No | Pull WAL legacy |
| `POST /sync/{gen}/{start}/{end}` | ❌ No | Push WAL legacy |
| `GET /info` | ❌ No | Metadata de DB |
| `GET /export/{gen}` | ❌ No | Bootstrap full |

Limitaciones del `--sync-server`:
- **Una sola DB** por proceso (no multi-tenant)
- **Sin autenticación** de ningún tipo
- **Sin zstd** (el cliente puede solicitarlo pero el servidor devuelve error)
- **Sin long polling** (no espera cambios nuevos)
- **Sin baton/session** para sesiones de push
- **Single-thread**, no diseñado para producción

### API de Administración

Los endpoints para gestionar tenants, grupos y bases de datos **no están implementados**:

```
POST   /v1/tenants/{name}                           # Crear tenant
POST   /v1/tenants/{name}/groups/{name}              # Crear grupo
POST   /v1/tenants/{name}/groups/{name}/databases/{n} # Crear DB
DELETE /v1/tenants/{name}/groups/{name}/databases/{n} # Eliminar DB
GET    /v1/tenants/{name}/...                         # Listar recursos
```

Estas rutas se referencian en los tests de integración (`bindings/rust/src/sync.rs`) apuntando a `ADMIN_URL = "http://localhost:8081"` — un servicio externo que no está en este repo.

### Infraestructura Cloud

| Componente | En este repo |
|------------|-------------|
| Autenticación (JWT, API keys) | ❌ Solo cliente envía `Bearer`; no hay server-side |
| Rate limiting / Quotas | ❌ |
| Billing | ❌ |
| Dashboard / Monitoring | ❌ |
| Database provisioning automático | ❌ |
| Deployment (docker-compose, Helm, K8s) | ❌ Solo `.devcontainer/docker-compose.yml` para desarrollo |
| Multi-tenancy server-side | ❌ |

---

## 4. ¿Qué Opciones Tengo para Self-Hosting?

### Opción A: Usar Turso Cloud para el Servidor Sync

La opción más simple. El motor y cliente son open source, pero el servidor sync lo provee Turso Cloud:

```
┌──────────────────────┐     ┌─────────────────────┐
│  Tu Aplicación       │     │  Turso Cloud         │
│  ┌────────────────┐  │     │  (servidor sync      │
│  │ turso_sync_     │  │ HTTP │  completo, auth,    │
│  │ engine (cliente)│──┼────►│  multi-tenancy,      │
│  │                │  │     │  API admin)          │
│  │ turso_core (db) │  │     └─────────────────────┘
│  └────────────────┘  │
└──────────────────────┘
```

- Ventajas: cero infraestructura que mantener
- Desventajas: dependencia del servicio cloud, costo

### Opción B: Construir Tu Propio Servidor Sync

Usas `turso_core` + `turso_sync_engine` como bibliotecas y construyes el servidor HTTP que falta:

```
┌──────────────────────┐     ┌──────────────────────────┐
│  Tu Aplicación       │     │  Tu Servidor Sync         │
│  ┌────────────────┐  │     │  ┌──────────────────────┐ │
│  │ turso_sync_     │  │ HTTP │  │ Implementar:         │ │
│  │ engine (cliente)│──┼────►│  │ • /pull-updates      │ │
│  │                │  │     │  │ • /v2/pipeline        │ │
│  │ turso_core (db) │  │     │  │ • /sync/...          │ │
│  └────────────────┘  │     │  │ • /info               │ │
└──────────────────────┘     │  │ • /export/...         │ │
                              │  │ • Auth (JWT/API key)  │ │
                              │  │ • Gestión de DBs      │ │
                              │  └──────────────────────┘ │
                              └──────────────────────────┘
```

Lo que necesitas implementar:
1. **Endpoints sync faltantes** — `/sync/`, `/info`, `/export/`, zstd para `/pull-updates`
2. **Autenticación** — Validación de JWT o API keys en cada request
3. **Gestión de DBs** — Un registry que mapee `tenant/db` → archivo `.db` en disco
4. **Long polling** — Mantener conexiones abiertas esperando cambios nuevos
5. **Concurrencia** — Múltiples DBs y conexiones simultáneas

El `sync_server.rs` puede servir como referencia parcial, pero necesitarás extenderlo significativamente.

### Opción C: Usar `sqld` del Proyecto libSQL Original

El proyecto original `libsql` (C) incluía `sqld`, un servidor sync completo y open source. Es un proyecto separado de este repositorio pero implementa el mismo protocolo wire:

```
https://github.com/tursodatabase/libsql
```

`sqld` implementa todos los endpoints sync, auth, y multi-tenancy. El cliente de sync de este repositorio (`turso_sync_engine`) puede conectarse a `sqld`. Sin embargo, ten en cuenta que:

- `sqld` usa el motor libSQL (fork de SQLite en C), no `turso_core` (Rust)
- Los dos motores son compatibles a nivel de protocolo wire
- `sqld` está en mantenimiento; el desarrollo activo está en este repo (Rust)

---

## 5. Protocolo de Transporte: ¿Polling, Long Polling, SSE, WebSockets?

Tanto Turso Cloud como `sqld` usan exclusivamente **HTTP/1.1 con long polling**. No hay SSE, no hay WebSockets.

### Analogía del Restaurante

Imagina que eres un mesero y quieres saber cuándo la cocina tiene un plato listo para servir. Hay 4 formas:

```
POLLING (peor opción)
─────────────────────
Tú:      "¿Ya está listo?"  ──►  Cocinero: "No."
(3 seg)  "¿Ya está listo?"  ──►  Cocinero: "No."
(3 seg)  "¿Ya está listo?"  ──►  Cocinero: "No."
(3 seg)  "¿Ya está listo?"  ──►  Cocinero: "Sí, aquí está."
Problema: Molestas 4 veces, la mayoría inútiles.

LONG POLLING (buena opción, la que usa Turso)
─────────────────────────────────────────────
Tú:      "Avísame cuando esté listo (pero no tardes más de 15 seg)" ──► Cocinero: "..."
(5 seg)                                                                    Cocinero: "¡Listo!"
                                                                        ◄── Cocinero: "Aquí está."
Si en 15 segundos no hay nada:
                                                                        ◄── Cocinero: "Nada aún."
Tú (inmediatamente): "Avísame cuando esté listo (otros 15 seg)" ──►

Ventaja: Solo molestas cuando realmente hay algo o cuando expira el tiempo.

SSE (Server-Sent Events, buena opción pero no usada)
─────────────────────────────────────────────────────
Tú le dices al cocinero: "Cada vez que tengas algo, me lo mandas por este tubo."
El cocinero te manda cada plato en cuanto está listo, uno tras otro, por la misma conexión.
Tú nunca preguntas — solo recibes.

WebSocket (mejor opción técnica, pero más compleja)
────────────────────────────────────────────────────
Tú y el cocinero instalan un teléfono dedicado (conexión permanente bidireccional).
Tú puedes decirle: "Necesito 2 cafés" y el cocinero: "Aquí tienes 1 café".
Hablan los dos en ambas direcciones, sin límite de mensajes.
```

### Comparativa Real

| | Polling | Long Polling | SSE | WebSocket |
|---|---|---|---|---|
| **¿Cómo funciona?** | GET cada N segundos | POST, servidor espera cambios o timeout | Conexión abierta, servidor envía eventos | Conexión bidireccional permanente |
| **Latencia** | Hasta N segundos | ~50ms (en cuanto hay cambio) | ~50ms | ~50ms |
| **Requests vacíos** | Muchos (cada N seg) | 1 cada 15s si no hay cambios | 0 (conexión persistente) | 0 (conexión persistente) |
| **Conexiones abiertas** | 0 entre requests | 1 por cliente activo | 1 por cliente | 1 por cliente |
| **Complejidad servidor** | Trivial | Baja (timeout + canal) | Media (retry, reconexión) | Alta (frames, ping/pong, reconexión) |
| **Funciona con HTTP/1.1** | ✅ | ✅ | ✅ | ❌ (requiere upgrade) |
| **Funciona con proxies/CDN** | ✅ | ✅ | ⚠️ (algunos proxy bufferean) | ⚠️ (algunos proxy no soportan) |
| **Funciona en WASM** | ✅ | ✅ | ❌ (no hay EventSource) | ❌ (no hay WebSocket) |
| **Funciona en React Native** | ✅ | ✅ | ❌ (no nativo) | ✅ |
| **Funciona en Edge Functions** | ✅ | ⚠️ (timeout corto) | ❌ | ❌ |
| **Costo de red (datos móviles)** | Medio (muchos requests) | Bajo (1 cada 15s) | Muy bajo | Muy bajo |
| **Costo de batería** | Medio | Bajo | Bajo | Bajo |

### ¿Por Qué Turso Eligió Long Polling?

```
┌─────────────────────────────────────────────────────────────────┐
│  Long polling es el "punto dulce" entre simplicidad y eficiencia │
│                                                                  │
│  ✅ Simplicidad:                                                  │
│     Es literalmente un HTTP POST normal. El servidor solo tarda  │
│     en responder. Cualquier backend lo soporta sin librerías.     │
│                                                                  │
│  ✅ Latencia casi igual a WebSocket:                              │
│     En cuanto la DB tiene un cambio, el servidor despierta y      │
│     responde. No hay que esperar al próximo intervalo de poll.    │
│                                                                  │
│  ✅ Menos requests que polling:                                   │
│     Solo se re-conecta cada 15 segundos si no hay actividad,      │
│     vs cada 1-3 segundos que haría el polling normal.             │
│                                                                  │
│  ✅ Portabilidad total:                                           │
│     Funciona en WASM, React Native, proxies, edge functions.      │
│     SSE y WebSocket no funcionan en todos estos entornos.         │
│                                                                  │
│  ❌ La única desventaja real:                                     │
│     Un request vacío cada 15 segundos si no hay cambios.          │
│     Para una app con 100k usuarios activos = 6.6k req/segundo.    │
│     Para una app con 1k usuarios = 66 req/segundo. Trivial.       │
└─────────────────────────────────────────────────────────────────┘
```

### ¿Qué Es El Request Vacío Cada 15 Segundos?

```
Sin actividad:
  Hora 0s:  Cliente → POST /pull-updates (long_poll_timeout_ms=15000)
  Hora 15s: Servidor → "sin cambios" (body vacío, ~200 bytes)
  Hora 15s: Cliente → POST /pull-updates (long_poll_timeout_ms=15000)
  Hora 30s: Servidor → "sin cambios"
  ...

Con actividad:
  Hora 0s:  Cliente → POST /pull-updates (long_poll_timeout_ms=15000)
  Hora 3s:  Llega un cambio → Servidor responde inmediato (~32KB de páginas)
  Hora 3s:  Cliente → POST /pull-updates (long_poll_timeout_ms=15000)
  Hora 8s:  Llega otro cambio → Servidor responde inmediato
```

El costo es mínimo: ~200 bytes cada 15 segundos por cliente conectado. Para referencia, un WebSocket tiene un ping/pong de ~50 bytes cada 30 segundos — comparable.

### ¿Vale La Pena Implementar SSE o WebSocket En Vez de Long Polling?

**Para el caso de uso de sync de base de datos: NO.**

| Razón | Explicación |
|-------|-------------|
| La latencia ya es baja | Long poll responde en ~50ms cuando hay cambios. SSE/WS no serían perceptiblemente más rápidos para el usuario. |
| El patrón es request/response | El cliente pregunta "¿qué cambió desde rev-001?" — esto es naturalmente un request. SSE/WS son para streams de eventos, no para preguntas puntuales. |
| Cada pull es transaccional | El pull forma parte de una transacción de sync (revertir → aplicar → replay). Meter SSE/WS no simplifica nada. |
| La complejidad no se justifica | Implementar y mantener WebSocket (reconexión, backoff, ping/pong, state management) para ahorrar 200 bytes cada 15 segundos no compensa. |
| Portabilidad | Si tu app corre en WASM o edge functions, SSE/WS simplemente no funcionan. Long poll sí. |

### Tabla de Costos Reales (Ejemplo: App con 10,000 Usuarios Conectados)

| Protocolo | Requests/minuto al servidor | Datos/minuto (sin cambios) | Complejidad |
|-----------|---------------------------|---------------------------|-------------|
| Polling (3s) | 200,000 | 40MB | Trivial |
| Polling (10s) | 60,000 | 12MB | Trivial |
| **Long Polling (15s)** | **40,000** | **8MB** | **Baja** |
| SSE | ~0 (conexiones abiertas) | ~5MB (keepalive) | Media |
| WebSocket | ~0 (conexiones abiertas) | ~2MB (ping/pong) | Alta |


---

## 6. Comparativa de Opciones

| Criterio | Turso Cloud | Construir servidor propio | Usar `sqld` (libSQL) |
|----------|-------------|---------------------------|----------------------|
| **Esfuerzo inicial** | Bajo | Muy alto | Medio |
| **Mantenimiento** | Cero | Alto (tú mantienes) | Medio (sqld en mantenimiento) |
| **Motor de BD** | `turso_core` (Rust) | `turso_core` (Rust) | libSQL (C) |
| **Endpoints sync** | Completos | Tú los implementas | Completos |
| **Auth** | Incluido | Tú lo implementas | Incluido |
| **Multi-tenancy** | Incluido | Tú lo implementas | Incluido |
| **Costo** | Pago por uso | Infraestructura propia | Infraestructura propia |
| **Control** | Limitado | Total | Total |
| **Soporte** | Oficial Turso | Ninguno | Comunidad |

---

## 7. Endpoints Sync: Referencia Completa

Para referencia si decides construir tu propio servidor, estos son todos los endpoints que el cliente de sync espera:

### Protocolo V1 (moderno, recomendado)

| Endpoint | Método | Body | Respuesta | Uso |
|----------|--------|------|-----------|-----|
| `/v2/pipeline` | POST | JSON: `PipelineReqBody` con `StreamRequest[]` | JSON: `PipelineRespBody` con `StreamResult[]` | Push de cambios lógicos (SQL) |
| `/pull-updates` | POST | Protobuf: `PullUpdatesReqProtoBody` | Stream Protobuf: `PullUpdatesRespProtoBody` + `PageData[]` | Pull de cambios + bootstrap |

### Protocolo Legacy (V0)

| Endpoint | Método | Body | Respuesta | Uso |
|----------|--------|------|-----------|-----|
| `/sync/{gen}/{start}/{end}` | GET | — | Binario: frames WAL | Pull de frames WAL |
| `/sync/{gen}/{start}/{end}[/{baton}]` | POST | Binario: frames WAL | Binario: frames WAL | Push de frames WAL |
| `/info` | GET | — | JSON: `DbSyncInfo` (`current_generation`) | Metadata |
| `/export/{gen}` | GET | — | Binario: archivo DB completo | Bootstrap full |

### Detalle: `/pull-updates` Request (Protobuf)

```protobuf
message PullUpdatesReqProtoBody {
    PageUpdatesEncodingReq encoding = 1;  // Raw=0, Zstd=1
    string server_revision = 2;           // revisión deseada del servidor
    string client_revision = 3;           // revisión actual del cliente
    uint32 long_poll_timeout_ms = 4;      // esperar cambios nuevos
    bytes server_pages_selector = 5;      // RoaringBitmap de páginas
    string server_query_selector = 7;     // SQL para seleccionar páginas
    bytes client_pages = 6;               // páginas del cliente
}
```

### Detalle: `/pull-updates` Response (Protobuf Stream)

```
PullUpdatesRespProtoBody {
    server_revision: string
    db_size: uint64
    raw_encoding / zstd_encoding: ...
}
seguido de N × PageData {
    page_id: uint64
    encoded_page: bytes
}
```

### Detalle: `/v2/pipeline` Request/Response (JSON)

```json
// Request
{
  "baton": "optional_session_token",
  "requests": [
    {"type": "execute", "stmt": {"sql": "BEGIN IMMEDIATE"}, "want_rows": false},
    {"type": "batch", "batch": {"steps": [
      {"stmt": {"sql": "INSERT INTO users VALUES (...)"}},
      {"stmt": {"sql": "UPDATE users SET ... WHERE id = 5"}}
    ]}},
    {"type": "execute", "stmt": {"sql": "COMMIT"}, "want_rows": false}
  ]
}

// Response
{
  "baton": "optional_session_token",
  "base_url": null,
  "results": [
    {"type": "execute", "result": {"cols": [], "rows": []}},
    {"type": "batch", "result": {"step_results": [
      {"result": {"cols": [], "rows": [], "affected_row_count": 1}},
      {"result": {"cols": [], "rows": [], "affected_row_count": 1}}
    ]}},
    {"type": "execute", "result": {"cols": [], "rows": []}}
  ]
}
```

---

## 8. Conclusión

**El motor y el cliente son 100% open source.** Puedes usar `turso_core` como base de datos embebida sin depender de nada externo, y `turso_sync_engine` para conectar clientes entre sí o a un servidor sync.

**El servidor sync completo no es open source.** Si necesitas multi-tenancy con auth y gestión de DBs, tus opciones son:

1. **Turso Cloud** — la opción más simple; pagas por el servicio gestionado
2. **Construir tu propio servidor** — viable si tienes recursos de infraestructura; usas `turso_core` + `turso_sync_engine` como building blocks
3. **Usar `sqld` del proyecto libSQL** — servidor sync open source ya existente, pero basado en el motor C (no en `turso_core` Rust)
