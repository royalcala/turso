# Guia de Decision: Local-First Sync con Turso/Limbo

## Objetivo de este documento
Este documento consolida todo lo discutido sobre:
- Como funciona el sync en este repositorio.
- Que implican CDC, partial sync, MCP y sync-server.
- Limites de seguridad/permisos en local-first.
- Comparativa de alternativas (Turso/Limbo, Electric/Zero/PowerSync, CouchDB/PouchDB).
- Propuesta final recomendada para combinar Limbo + TinyBase.
- Arquitectura recomendada para Loro docs (validacion, merge y proyeccion de lectura).

---

## 1) Conceptos base

### 1.1 Local-first
La app lee y escribe en una base local. El sync replica cambios entre cliente y servidor en segundo plano.

### 1.2 CDC vs MVCC
- CDC (Change Data Capture): registra cambios para poder hacer push incremental.
- MVCC: modelo de concurrencia/snapshots.
- En este proyecto, CDC y MVCC no estan pensados para usarse juntos en el mismo flujo.

### 1.3 No es "NoSQL"
Limbo/Turso es un motor SQL compatible con SQLite. Puede almacenar JSON, pero sigue siendo SQL.

---

## 2) Como funciona el sync aqui

### 2.1 Push (cliente -> servidor)
- Se toman cambios locales capturados por CDC.
- Se convierten en operaciones SQL replay.
- Se envian por `POST /v2/pipeline`.
- Se guarda cursor de ultimo cambio empujado.

### 2.2 Pull (servidor -> cliente)
- Se piden cambios desde una revision conocida.
- Se reciben paginas/WAL (nivel fisico, no "por tabla").
- Se aplican con secuencia segura (rollback temporal, apply remoto, replay local).

### 2.3 Cursores persistentes
El engine persiste automaticamente metadatos de sync (revision sincronizada, pistas de ultimo push/pull). No hay que inventar cursores manuales.

---

## 3) Partial sync explicado correctamente

### 3.1 Nivel real de granularidad
No opera por tabla de forma nativa. Opera principalmente por paginas fisicas.

### 3.2 Estrategias observadas
- `Prefix`: bootstrap parcial por rango/tamano.
- `Query`: bootstrap inicial seleccionado por query del servidor.

### 3.3 Punto clave
El filtro por query aplica al bootstrap inicial. En sincronizaciones posteriores el pull incremental sigue reglas de cambios fisicos y no garantiza filtro logico continuo por tabla/where.

---

## 4) Modos CLI importantes

### 4.1 `--sync-server`
Servidor HTTP del protocolo de sync (`/v2/pipeline`, `/pull-updates`).

### 4.2 `--mcp`
Servidor JSON-RPC por stdio para agentes de IA (tools de abrir DB, listar tablas, consultas y mutaciones).

### 4.3 Diferencia practica
- Sync-server: replicacion de datos.
- MCP: interfaz de herramientas para agentes.

---

## 5) Seguridad y permisos: realidad de local-first

### 5.1 Limitacion estructural
Si un dato llega al dispositivo, ese cliente lo puede leer. Por eso ACL fina en cliente (fila/columna) no es control fuerte.

### 5.2 Donde si aplicar seguridad fuerte
- Antes de sincronizar datos sensibles.
- En backend/proxy para validar push.
- En autenticacion/autorizacion central (ej. OpenFGA).

### 5.3 Si un cliente escribe algo no permitido
- El server/proxy debe rechazar en push.
- La central queda protegida.
- El cliente queda divergente localmente hasta reconciliar (rollback/correccion/reintento).

### 5.4 Implicacion clave: la DB local es no confiable
Aunque uses Limbo, sync y dos DB locales por usuario, el cliente siempre puede:
- editar la DB directamente,
- falsificar filas o blobs,
- saltarse validaciones de UI,
- reescribir metadatos locales.

Eso no es un bug del modelo; es una propiedad inevitable de cualquier sistema local-first.

La regla correcta es:
- lo local sirve para UX, trabajo offline y cache,
- lo remoto decide que entra a la verdad compartida,
- ninguna escritura del cliente se considera valida por existir en una DB sincronizada.

En otras palabras:
- `tenant_shared.db` en cliente es una replica materializada, no una autoridad.
- `user_workspace.db` en cliente es un workspace privado, no una prueba de validez.
- la unica autoridad es el backend que revalida y promueve.

---

## 6) OpenFGA en este stack

OpenFGA encaja como capa de autorizacion central:
- Decide si `user X` puede `accion Y` sobre `recurso Z`.
- El proxy traduce operaciones SQL/tools a checks de autorizacion.
- Se aplica especialmente en push y tools MCP de escritura.

No reemplaza:
- Parseo SQL (lo hace el proxy).
- Cifrado/TLS (infra).
- Segmentacion de lectura ya entregada en cliente.

### 6.1 Conclusion recomendada
Usaria OpenFGA como autoridad de permisos, pero no pondria la verdad de permisos dentro de `tenant_shared.db`.

Separacion recomendada:
- OpenFGA: motor autoritativo de decision (`can_read`, `can_write`, `can_manage_permissions`, etc.).
- Backend SQL: fuente operativa y auditable de cambios de permisos, membresias, roles y grants.
- `tenant_shared.db`: replica de datos compartidos del negocio, no fuente autoritativa de ACL.
- `user_workspace.db`: proyeccion privada de capacidades efectivas del usuario actual y, si aplica, vistas admin privadas.

La razon es simple:
- si metes todas las asignaciones de permisos en `tenant_shared.db`, las replicas compartidas del tenant terminan filtrando demasiado contexto interno,
- y aunque las metas ahi, seguirian sin ser confiables porque el cliente puede adulterarlas.

### 6.2 Que deberia editar la interfaz
La interfaz no deberia editar permisos escribiendo directo a una tabla sincronizada.

La UI deberia llamar endpoints del backend, por ejemplo:
- `POST /tenants/:tenant_id/members/:user_id/roles`
- `POST /documents/:doc_id/grants`
- `DELETE /documents/:doc_id/grants/:grant_id`

Flujo recomendado:
1. Un admin abre la pantalla de permisos.
2. La UI muestra una proyeccion leible de permisos actuales.
3. El admin hace un cambio.
4. La UI envia comando al backend.
5. El backend valida `can_manage_permissions` en OpenFGA.
6. Si pasa, actualiza tablas SQL de ACL/membresia.
7. El backend actualiza tuples en OpenFGA.
8. El backend publica proyecciones nuevas para UI.

### 6.3 Modelo recomendado de datos de permisos
Mantendria dos capas:

**A. Tablas SQL autoritativas y auditables**
```sql
CREATE TABLE tenant_memberships (
  tenant_id TEXT NOT NULL,
  user_id TEXT NOT NULL,
  role TEXT NOT NULL,
  status TEXT NOT NULL,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  PRIMARY KEY (tenant_id, user_id)
);

CREATE TABLE document_grants (
  grant_id TEXT PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  doc_id TEXT NOT NULL,
  subject_type TEXT NOT NULL,   -- user | role | group
  subject_id TEXT NOT NULL,
  relation TEXT NOT NULL,       -- reader | editor | owner
  created_by TEXT NOT NULL,
  created_at INTEGER NOT NULL
);

CREATE TABLE permission_audit_log (
  event_id TEXT PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  actor_user_id TEXT NOT NULL,
  action TEXT NOT NULL,
  resource_type TEXT NOT NULL,
  resource_id TEXT NOT NULL,
  payload_json TEXT NOT NULL,
  created_at INTEGER NOT NULL
);
```

**B. OpenFGA tuples derivadas**
- `user:123 relation editor document:abc`
- `user:123 relation admin tenant:t1`
- `role:manager relation editor document:abc`

SQL te da auditoria, reporting y facilidad de administracion.
OpenFGA te da la respuesta autoritativa de permiso en runtime.

### 6.4 Que si sincronizar hacia cliente
No sincronizaria una tabla global de "todos los permisos de todos" a `tenant_shared.db`.

Sincronizaria proyecciones minimas segun uso:

**Para todos los usuarios, en `tenant_shared.db`:**
- catalogos compartidos no sensibles,
- metadata de recursos,
- quizas roles disponibles del tenant (`owner`, `editor`, `viewer`),
- quizas politicas descriptivas de UI, no decisiones autoritativas.

**Para el usuario actual, en `user_workspace.db`:**
- `my_capabilities(resource_id, can_read, can_write, can_share, can_admin)`
- `my_memberships(tenant_id, role)`
- `permission_change_requests` si quieres que los cambios de permisos tambien sean workflow.

**Para admins, tambien en `user_workspace.db`:**
- `admin_permission_view(subject, resource, relation, inherited_from)`
- solo si ese admin realmente tiene `can_manage_permissions`.

Esto resuelve dos cosas:
- la UI puede renderizar botones y pantallas rapido sin consultar al backend cada vez,
- pero el backend sigue siendo quien decide de verdad.

### 6.5 Validacion de cambios de datos con OpenFGA
Cada write del usuario deberia validar al menos:
1. `can_read` o visibilidad del recurso base.
2. `can_write` sobre el documento o tabla afectada.
3. `can_manage_permissions` si intenta cambiar ACL.
4. Reglas de dominio adicionales fuera de OpenFGA.

OpenFGA responde la pregunta de autorizacion.
El backend todavia debe validar:
- que el documento pertenezca al tenant correcto,
- que la version base sea valida,
- que el payload sea integro,
- que el cambio no viole reglas semanticas.

### 6.6 Conclusion practica para tu caso
La conclusion concreta seria:
- `tenant_shared.db` para datos compartidos del negocio.
- `user_workspace.db` para drafts, outbox y capacidades privadas del usuario.
- permisos autoritativos en backend (`tenant_memberships`, `document_grants`, audit log).
- OpenFGA como motor de decision.
- UI de permisos mediante comandos al backend, no editando tablas sync como fuente de verdad.

### 6.7 OpenFGA vs Cedar vs CASL: comparacion real

#### OpenFGA
Fortalezas:
- Excelente para relaciones de permisos tipo ReBAC (usuario, rol, recurso, herencia).
- Muy bueno para colaboracion multi-tenant (owner/editor/viewer, shares, grants por doc/folder).
- API simple para checks en runtime (`can user X do Y on Z?`).
- Facil de integrar como guard en write pipe.

Debilidades:
- Menos expresivo que un motor ABAC completo para reglas muy ricas de atributos/tiempo/contexto.
- Para reglas complejas de negocio necesitas combinar con validaciones del backend.

#### Cedar Policy
Fortalezas:
- Lenguaje de policy muy expresivo (ABAC potente).
- Muy bueno cuando reglas dependen de muchos atributos/contexto (region, horario, plan, entorno, tags).
- Politicas versionables y auditables como artefacto.

Debilidades:
- Mayor complejidad operacional y de modelado para casos donde ReBAC simple ya cubre.
- Si lo usas como unico motor para colaboracion documental, puede ser mas costoso de operar que OpenFGA.

#### CASL (JS/TS)
Fortalezas:
- Muy util para controlar vistas/acciones en frontend (mostrar/ocultar botones, rutas, componentes).
- Excelente DX en cliente para construir abilities por usuario.

Debilidades:
- No es autoridad de seguridad.
- El cliente puede adulterar estado local, por lo que CASL solo sirve para UX.

#### Conclusiones practicas
- Si tu problema principal es colaboracion por relaciones (tenant/member/document/role), OpenFGA encaja mejor como nucleo.
- Si tu problema principal son politicas ABAC complejas y cambiantes, Cedar puede complementar o, en algunos casos, reemplazar parte del stack.
- CASL conviene casi siempre para UX de frontend, pero nunca como enforcement final.

### 6.8 Recomendacion para este proyecto

Para tu caso actual (local-first multi-tenant, docs colaborativos, write pipe con validacion), recomendaria:

1. OpenFGA como autoridad principal de autorizacion en backend.
2. CASL en frontend para render de vistas y acciones.
3. Backend siempre valida en cada write aunque la UI o CASL permitan la accion.
4. Cedar solo si aparecen reglas ABAC complejas que OpenFGA + validaciones normales ya no cubran bien.

Patron de ejecucion recomendado:
1. Frontend calcula abilities con CASL desde `my_capabilities`.
2. Usuario intenta accion.
3. Request llega al backend.
4. Backend consulta OpenFGA (`can_write`, `can_manage_permissions`, etc.).
5. Backend aplica reglas de dominio (version base, integridad, negocio).
6. Si todo pasa, persiste y publica.

### 6.9 Como aplicar reglas a vistas frontend sin romper seguridad

No uses decisiones en vivo de OpenFGA directamente para cada render de componente offline.
Mejor:

1. Generar proyeccion de capacidades por usuario en backend.
2. Sincronizar esa proyeccion hacia `user_workspace.db`.
3. Construir abilities CASL desde esa proyeccion.
4. Renderizar UI con CASL.
5. Revalidar siempre en backend al ejecutar comandos.

Tabla sugerida de proyeccion para UI:

```sql
CREATE TABLE my_capabilities (
  tenant_id TEXT NOT NULL,
  user_id TEXT NOT NULL,
  resource_type TEXT NOT NULL,
  resource_id TEXT NOT NULL,
  can_read INTEGER NOT NULL,
  can_write INTEGER NOT NULL,
  can_share INTEGER NOT NULL,
  can_manage_permissions INTEGER NOT NULL,
  policy_version TEXT,
  updated_at INTEGER NOT NULL,
  PRIMARY KEY (tenant_id, user_id, resource_type, resource_id)
);
```

Esta tabla sirve para UX y decisiones de interfaz.
No reemplaza checks del backend.

---

## 7) Agent-first vs Policy-first

Concluson: no conviene delegar seguridad al agente.

Patron recomendado:
1. Agente interpreta intencion.
2. Policy layer valida permiso y reglas.
3. Solo entonces se ejecuta en DB/sync.

---

## 8) Comparativa de enfoques (resumen)

### 8.1 Turso/Limbo (SQLite local-first)
Fuerte en:
- Offline real.
- Costo bajo cuando hay particion por tenant/usuario.
- Simplicidad operativa de datos locales.

Debil en:
- Permisos logicos finos continuos por query/fila en pull.

### 8.2 Electric / Zero / PowerSync (centrados en PG y sync por slices)
Fuerte en:
- Sync por subconjuntos logicos.
- Flujos con permisos dinamicos complejos.

Costo/tension:
- Mayor complejidad y/o costo infra para casos simples por tenant.

### 8.3 CouchDB + PouchDB
Fuerte en:
- Replicacion documental madura.
- Muy util para CRUD offline document-centric.

Debil en:
- Menor ergonomia SQL relacional avanzada (joins/reporting complejo).

---

## 9) Costos (orden relativo bajo supuesto tenant-simple)

Bajo el supuesto de:
- Datos por tenant.
- Auth simple.
- Sin reglas complejas por fila/columna.

Orden esperado de costo de infraestructura:
1. Turso/Limbo OSS self-host (usualmente mas bajo).
2. Electric/PowerSync/Zero con PG central (usualmente mayor, segun carga y modelo).

Nota: en TCO real, el costo de equipo/operacion puede superar el de VMs.

---

## 10) Propuesta final: Limbo + TinyBase (recomendada)

La idea es buena, con ajuste de diseno para escalar sin dolor.

### 10.1 Rol de cada componente
- Limbo: fuente de verdad persistente + sync.
- TinyBase: estado reactivo de UI, derivaciones y UX instantanea.

### 10.2 Modelo de datos recomendado (hibrido)
Evitar "solo una tabla JSON" sin estructura.

Propuesta minima:

#### Tabla `documents`
- `id` (PK)
- `tenant_id`
- `doc_type`
- `doc` (JSON)
- `version`
- `created_at`
- `updated_at`
- `deleted_at` (nullable)

Indices sugeridos:
- `(tenant_id, doc_type, updated_at)`
- `(tenant_id, id)`

#### Tabla `edges` (relaciones)
- `tenant_id`
- `from_id`
- `to_id`
- `rel_type`
- `created_at`

### 10.3 Regla practica
- Campos de filtro frecuente (status, owner_id, etc.) conviene exponerlos como columnas indexables (o generadas), no solo dentro del JSON.

### 10.4 Flujo recomendado
1. TinyBase aplica optimismo de UI.
2. Persistir transaccionalmente en Limbo.
3. Sync push/pull desde Limbo.
4. Rehidratar TinyBase desde Limbo al reconectar.

### 10.5 Beneficio
- Mantienes flexibilidad documental.
- Conservas potencia SQL para rendimiento y mantenimiento.
- Preparas el sistema para crecer sin rehacer todo.

---

## 11) Roadmap pragmatico sugerido

### Fase 1
- Implementar `documents` + indices base.
- Sync funcional por tenant.
- TinyBase como cache reactivo.

### Fase 2
- Agregar `edges` para relaciones persistentes.
- Agregar control de version por documento.
- Politica de reconciliacion simple (last-write-wins por doc_type).

### Fase 3
- Proxy de seguridad (allowlist SQL/tooling).
- Integrar OpenFGA si crecen roles y politicas.
- Promover campos calientes de JSON a columnas indexadas.

---

## 12) Backend Write Pipe: Consumo y validación de cambios

### 12.1 El problema
Cliente envia cambios (POST /v2/write). ¿Como el servidor los consume, valida y promociona al estado canonico?

Opciones: sync puro vs async con queue. Aqui resumo ambas.

### 12.2 Opcion A: Síncrono + `pending_changes` en Canonical DB (MVP recomendado)

**Ventaja principal**: Transaccional, auditoria built-in, sin infra extra.

**Schema:**
```sql
CREATE TABLE pending_changes (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  document_id TEXT NOT NULL,
  loro_ops BLOB NOT NULL,           -- serialized Loro ops
  status TEXT DEFAULT 'pending',    -- 'pending' | 'published' | 'rejected'
  validation_error TEXT,             -- error msg si rejected
  created_at INTEGER NOT NULL,
  promoted_at INTEGER,               -- timestamp of promotion
  FOREIGN KEY(document_id) REFERENCES documents(id),
  FOREIGN KEY(user_id) REFERENCES users(id)
);

CREATE INDEX idx_pending_status ON pending_changes(status);
CREATE INDEX idx_pending_user ON pending_changes(user_id, created_at DESC);
CREATE INDEX idx_pending_doc ON pending_changes(document_id, status);
```

**Flujo HTTP (pseudocode):**
```
POST /v2/write
  Request: {
    document_id: "doc_123",
    user_id: "user_abc",
    loro_ops: <bytes>,
    change_id: "ch_1234"  -- idempotency key
  }

Handler:
  1. BEGIN TRANSACTION
  
  2. OpenFGA.check(user_id, "write", document_id)
     → If denied: 
       INSERT pending_changes(status='rejected', validation_error='unauthorized')
       Return { status: 'rejected', reason: 'unauthorized' }
  
  3. Leur merge validation:
     current_state = SELECT loro_state FROM documents WHERE id = document_id
     merged = loreland::merge(current_state, loro_ops)
     If error:
       INSERT pending_changes(status='rejected', validation_error=error)
       Return { status: 'rejected', reason: 'merge_conflict' }
  
  4. Promote to canonical (for MVP, synchronously):
     INSERT pending_changes(
       id=change_id,
       document_id, user_id, loro_ops,
       status='pending'
     )
     
     UPDATE documents SET
       loro_state = merged,
       version = version + 1,
       updated_by = user_id,
       updated_at = now()
     WHERE id = document_id
     
     UPDATE pending_changes SET
       status = 'published',
       promoted_at = now()
     WHERE id = change_id
     
     INSERT changeset_index(document_id, version, timestamp, changeset)
       VALUES(document_id, version+1, now(), merged)
  
  5. COMMIT
  
  Response: {
    status: 'published',
    change_id,
    version,
    published_at
  }
```

**Ventajas:**
- ✅ Atomic (todo o nada).
- ✅ Auditoria: puedes ver intent + resultado en `pending_changes`.
- ✅ Query capability: "dame mis cambios del último día", "rechazos por validación";
- ✅ Rollback built-in (historial completo).
- ✅ Sin dependencias externas.

**Desventajas:**
- ❌ Cliente espera validación (síncrono).
- ❌ Si Loro merge es intensivo, high latency.
- ❌ No ideal si cambios vienen rápido en cascada.

### 12.3 Opcion B: Asíncrono + Valkey Queue (escala)

Cuando los cambios llegan rápido o merge es caro, desacopla:

**Cliente:**
```
POST /v2/write → respuesta inmediata
  Response: { 
    status: 'queued',
    change_id,
    expected_promotion_ms: 100
  }
  
Luego cliente:
  → Muestra "pending..." en UI (optimistic).
  → Espera /v2/sync o long-poll para confirmación de `published`.
```

**Backend queue:**
```
Write handler (rápido):
  1. OpenFGA check (sync, fail-fast).
  2. LPUSH cola:${document_id}  { change_id, user_id, loro_ops_json }
  3. Return { status: 'queued', change_id }

Background worker (consume cola):
  WHILE true:
    FOR each_document_id:
      changes = RPOP cola:${document_id}  [count=100]
      
      FOR each change:
        merged = Loro.merge(...)
        If valid:
          DB: UPDATE documents + INSERT published
          Publish event → client sync channels
        Else:
          DB: INSERT pending_changes(status='rejected', error=...)
```

**Ventajas:**
- ✅ Respuesta inmediata al cliente.
- ✅ Puede procesar cambios en batch (mas eficiente).
- ✅ Tolerancia a picos de tráfico (colas absorben).

**Desventajas:**
- ❌ Valkey/Redis infrastructure.
- ❌ Eventual consistency (delay antes de publish).
- ❌ Más complejo: retry logic, ordering, edge cases.

### 12.4 Estrategia recomendada

**Para MVP:** Opcion A (síncrono).

**Por qué:**
1. Empieza sin extras.
2. Mide latencias reales.
3. Agrega async solo si lo necesitas.
4. Transicion facil: enqueue en lugar de hacer sync = 1 cambio.

**Cuando migrar a B:**
- POST /v2/write latency > 200ms consistentemente.
- Rate de cambios > X por segundo (mide cuando lo veas).
- Observas contention en DB durante escritura.

### 12.5 Topologia recomendada: dos DB locales por usuario

La idea de tener dos DB por `tenant/user` es buena y, para tu caso, es mas solida que un simple overlay efimero.

Modelo recomendado en cliente:
- `tenant_shared.db`: replica local de solo lectura del estado canonico del tenant. Todos los usuarios del tenant convergen a esta misma verdad compartida.
- `user_workspace.db`: base privada del usuario, sincronizada entre sus dispositivos, donde viven drafts, cambios pendientes, rechazos, cola de salida y metadatos de reconciliacion.

Lectura en la app:
- La UI no lee solo una DB. Lee una composicion logica de ambas.
- Base para consultas globales e indices: `tenant_shared.db`.
- Base para estado personal y trabajo en progreso: `user_workspace.db`.

Ventajas de este modelo:
- Drafts y pendientes sobreviven entre dispositivos del mismo usuario.
- No mezclas datos privados/transitorios con la replica compartida del tenant.
- Si una validacion falla, el cambio sigue existiendo en la DB privada del usuario con estado y motivo.
- Puedes hacer sync independiente: una DB baja estado compartido; la otra sincroniza workspace personal.

Restricciones importantes:
- `tenant_shared.db` debe tratarse como proyeccion canonica, no como area de staging.
- `user_workspace.db` no debe promocionarse sola a verdad compartida. Todo lo que pase a canonico debe cruzar el write pipe.
- Si un draft depende de datos del tenant, guarda referencia estable (`doc_id`, `base_version`) para detectar cuando el compartido cambio debajo del draft.

Implicacion de seguridad:
- Que ambas DB sincronicen no las vuelve confiables.
- El cliente puede adulterar cualquiera de las dos.
- El write pipe debe recomputar permiso, integridad, version base, idempotencia y reglas de dominio antes de aceptar.

### 12.6 Cliente: Flujo de reconciliación

Independiente de sync/async, cliente necesita reconciliar:

```
tenant_shared.db:
  documents(id, canonical_blob, version, updated_at)

user_workspace.db:
  drafts(id, doc_id, local_blob, base_version, status)
  outbox(change_id, doc_id, payload, status)
  validation_log(change_id, status, error)

Local outbox (antes):
  [
    { id: 'ch_1', loro_ops: <bytes>, status: 'sent' },
    { id: 'ch_2', loro_ops: <bytes>, status: 'sent' }
  ]

Después de respuesta de /v2/write:
  - Si status='published': remove de outbox, UI actualiza.
  - Si status='rejected': mostrar error, marcar para retry/manual.
  - Si status='queued': esperar confirmación via sync.

En siguiente /v2/pull-updates:
  Recibes cambios canonicos (incluyendo los tuyos merged).
  Reconcilias `tenant_shared.db` contra `user_workspace.db`:
    - Si tu cambio esta en canonical con tu user_id → confirm.
    - Si canonical diverge del draft basado en `base_version` → conflicto o rebase local.
    - Si fue rechazado → mantienes draft en `user_workspace.db` con razon y opcion de corregir/reintentar.
```

### 12.7 Diagrama completo

```
┌─ CLIENT ──────────────────────────────────────┐
│  tenant_shared.db   → replica canonica         │
│  user_workspace.db  → drafts/outbox personal   │
│                                                │
│  User edits                                     │
│  → write en user_workspace.db                   │
│  → Optimistic UI update                         │
│  → POST /v2/write desde outbox                  │
└─────────────┬──────────────────────────────────┘
              │
              ↓
┌─ SERVER /v2/write ─────────────────────────────────────┐
│  1. OpenFGA.check(user, 'write', doc)                   │
│  2. Loro.merge(current, ops) → validate                 │
│  3. BEGIN TXN                                            │
│  4. INSERT pending_changes(pending)                      │
│  5. UPDATE documents(loro_state, version++)             │
│  6. UPDATE pending_changes(published)                    │
│  7. INSERT changeset_index (for sync)                    │
│  8. COMMIT                                               │
│  9. Return { status: published, version: 42 }           │
└─────────────┬───────────────────────────────────────────┘
              │
              ↓
┌─ CLIENT: Reconciliation ───────────────────────────────┐
│  tenant_shared.db recibe canonical v42                 │
│  user_workspace.db marca draft/outbox como synced      │
│  Si hay divergence: rebase local o conflicto de UX     │
└─────────────────────────────────────────────────────────┘
```

### 12.8 Contrato exacto recomendado: LWW + Audit Log + Cedar

Para el escenario ajustado de este proyecto:
- sin CRDT,
- sin OpenFGA remoto,
- sin Valkey/Redis,
- con `cedar-policy` embebido en backend,
- y con datos estructurados/relacionales,

el write pipe recomendado es sincronico, atomico y basado en control de concurrencia optimista con `version` entera monotona.

#### 12.8.1 Regla de concurrencia
No confiar en `updated_at` del cliente como autoridad.

La condicion de escritura correcta es:
- el cliente manda `base_version`,
- el backend hace `UPDATE ... WHERE version = :base_version`,
- si no actualiza exactamente una fila, responde `409 Conflict`.

`updated_at` queda como metadata de auditoria/ordenacion del servidor, no como arbitro de concurrencia.

#### 12.8.2 Esquema SQL minimo de produccion

```sql
CREATE TABLE entities (
  tenant_id TEXT NOT NULL,
  entity_type TEXT NOT NULL,
  entity_id TEXT NOT NULL,
  data_json TEXT NOT NULL CHECK (json_valid(data_json)),
  version INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  updated_by TEXT NOT NULL,
  deleted_at INTEGER,
  PRIMARY KEY (tenant_id, entity_type, entity_id)
);

CREATE INDEX idx_entities_tenant_type_updated
  ON entities (tenant_id, entity_type, updated_at DESC);

CREATE TABLE change_history (
  change_seq INTEGER PRIMARY KEY AUTOINCREMENT,
  change_id TEXT NOT NULL UNIQUE,
  tenant_id TEXT NOT NULL,
  entity_type TEXT NOT NULL,
  entity_id TEXT NOT NULL,
  actor_user_id TEXT NOT NULL,
  idempotency_key TEXT NOT NULL,
  base_version INTEGER NOT NULL,
  new_version INTEGER NOT NULL,
  patch_format TEXT NOT NULL,              -- 'json_patch'
  delta_json TEXT NOT NULL CHECK (json_valid(delta_json)),
  resulting_json TEXT NOT NULL CHECK (json_valid(resulting_json)),
  policy_version TEXT,
  created_at INTEGER NOT NULL
);

CREATE INDEX idx_change_history_entity_version
  ON change_history (tenant_id, entity_type, entity_id, new_version);

CREATE INDEX idx_change_history_tenant_seq
  ON change_history (tenant_id, change_seq);

CREATE TABLE write_idempotency (
  tenant_id TEXT NOT NULL,
  actor_user_id TEXT NOT NULL,
  idempotency_key TEXT NOT NULL,
  request_hash TEXT NOT NULL,
  entity_type TEXT NOT NULL,
  entity_id TEXT NOT NULL,
  status TEXT NOT NULL,                    -- 'processing' | 'applied' | 'conflict' | 'rejected'
  http_status INTEGER NOT NULL,
  response_json TEXT NOT NULL CHECK (json_valid(response_json)),
  created_at INTEGER NOT NULL,
  completed_at INTEGER,
  PRIMARY KEY (tenant_id, actor_user_id, idempotency_key)
);

CREATE INDEX idx_write_idempotency_entity
  ON write_idempotency (tenant_id, entity_type, entity_id, created_at DESC);
```

Notas:
- `entities` es el snapshot materializado que si baja a `tenant_shared.db`.
- `change_history` es append-only y sirve para auditoria, diff y pull incremental.
- `write_idempotency` evita duplicar efectos y tambien cachea respuestas `200`, `403` o `409` para reintentos seguros.
- `resulting_json` en `change_history` aumenta storage, pero simplifica depuracion, replay parcial y respuestas de conflicto. Si mas adelante pesa demasiado, se puede quitar y reconstruir desde `entities` + patches.

#### 12.8.3 Estructura recomendada para `delta_json`

Recomendacion: usar `JSON Patch` (RFC 6902) como formato canonico.

Ventajas:
- estandar conocido,
- representa solo campos mutados,
- TinyBase/React puede inspeccionar paths cambiados,
- sirve para auditoria y para construir UI de conflicto,
- evita guardar snapshot completo anterior en cada write.

Ejemplo:

```json
[
  { "op": "replace", "path": "/status", "value": "approved" },
  { "op": "replace", "path": "/amount", "value": 12500 },
  { "op": "add", "path": "/tags/-", "value": "urgent" }
]
```

No recomiendo guardar solo "payload completo nuevo" como delta principal, porque:
- pierdes granularidad de auditoria,
- empeora la resolucion de conflictos,
- dificulta explicar al usuario que cambio exactamente.

Si quieres mejor DX en backend, puedes aceptar payload completo desde cliente y convertirlo internamente a JSON Patch antes de persistir en `change_history`.

#### 12.8.4 Request HTTP recomendado

```json
POST /v2/write
Content-Type: application/json

{
  "idempotency_key": "01JTN6M6K8YJ3Q4V6R5B2Y9QXZ",
  "tenant_id": "tenant_123",
  "entity_type": "invoice",
  "entity_id": "inv_987",
  "base_version": 7,
  "patch_format": "json_patch",
  "delta": [
    { "op": "replace", "path": "/status", "value": "approved" },
    { "op": "replace", "path": "/approvedBy", "value": "user_42" }
  ],
  "client_context": {
    "device_id": "device_a1",
    "session_id": "sess_44",
    "submitted_at": 1777681200123
  }
}
```

Campos clave:
- `idempotency_key`: unica por intento logico de write.
- `base_version`: version sobre la que el cliente baso su cambio.
- `delta`: patch canonico de mutacion.
- `client_context`: opcional para trazabilidad, no para autoridad.

#### 12.8.5 Respuesta `200 OK`

```json
{
  "ok": true,
  "result": "applied",
  "change": {
    "change_id": "chg_01JTN6P0BX8QW6Y2S3EXAMPLE",
    "change_seq": 4821,
    "idempotency_key": "01JTN6M6K8YJ3Q4V6R5B2Y9QXZ"
  },
  "entity": {
    "tenant_id": "tenant_123",
    "entity_type": "invoice",
    "entity_id": "inv_987",
    "version": 8,
    "updated_at": 1777681200456,
    "updated_by": "user_42",
    "data": {
      "status": "approved",
      "approvedBy": "user_42",
      "amount": 12500
    }
  }
}
```

#### 12.8.6 Respuesta `409 Conflict` cuando falla CAS

La respuesta debe darle a TinyBase suficiente informacion para:
- ver que su base quedo vieja,
- mostrar que cambio en servidor desde esa base,
- comparar con su draft local,
- permitir rebase o resolver manualmente.

```json
{
  "ok": false,
  "error": {
    "code": "version_conflict",
    "message": "The entity was updated by another writer before this mutation was applied.",
    "entity_ref": {
      "tenant_id": "tenant_123",
      "entity_type": "invoice",
      "entity_id": "inv_987"
    },
    "idempotency_key": "01JTN6M6K8YJ3Q4V6R5B2Y9QXZ",
    "base_version": 7,
    "current_version": 10,
    "server_entity": {
      "version": 10,
      "updated_at": 1777681255123,
      "updated_by": "user_99",
      "data": {
        "status": "paid",
        "approvedBy": "user_31",
        "amount": 12500,
        "paymentRef": "tr_889"
      }
    },
    "attempted_delta": [
      { "op": "replace", "path": "/status", "value": "approved" },
      { "op": "replace", "path": "/approvedBy", "value": "user_42" }
    ],
    "server_changes_since_base": [
      {
        "change_id": "chg_01JTN6XX1",
        "version": 8,
        "actor_user_id": "user_31",
        "created_at": 1777681210000,
        "delta": [
          { "op": "replace", "path": "/approvedBy", "value": "user_31" }
        ]
      },
      {
        "change_id": "chg_01JTN6XX2",
        "version": 9,
        "actor_user_id": "user_31",
        "created_at": 1777681220000,
        "delta": [
          { "op": "replace", "path": "/status", "value": "paid" }
        ]
      },
      {
        "change_id": "chg_01JTN6XX3",
        "version": 10,
        "actor_user_id": "user_99",
        "created_at": 1777681255000,
        "delta": [
          { "op": "add", "path": "/paymentRef", "value": "tr_889" }
        ]
      }
    ],
    "conflicting_paths": [
      "/status",
      "/approvedBy"
    ],
    "suggested_action": "rebase_and_retry"
  }
}
```

Con eso, el frontend puede:
1. tomar su draft local,
2. comparar contra `server_entity.data`,
3. resaltar `conflicting_paths`,
4. ofrecer "usar version del servidor", "reaplicar mi cambio", o merge manual.

#### 12.8.7 Respuesta `403 Forbidden`

```json
{
  "ok": false,
  "error": {
    "code": "forbidden",
    "message": "You do not have permission to update this entity.",
    "entity_ref": {
      "tenant_id": "tenant_123",
      "entity_type": "invoice",
      "entity_id": "inv_987"
    },
    "policy_version": "2026-05-02.1"
  }
}
```

#### 12.8.8 Flujo transaccional del backend

Pseudoflujo en Rust + SQLite/Limbo:

```text
BEGIN IMMEDIATE;

1. Intentar reservar idempotency key en write_idempotency.
   - Si ya existe:
     a. mismo request_hash -> devolver response_json cacheada.
     b. distinto request_hash -> 409 idempotency_key_reused.

2. Evaluar Cedar para autorizacion.
   - Si falla: guardar response_json 403 en write_idempotency y COMMIT.

3. Leer fila actual de entities.

4. Si current.version != base_version:
   - construir payload 409 con server_entity + server_changes_since_base.
   - guardar response_json 409 en write_idempotency.
   - COMMIT.

5. Aplicar JSON Patch al snapshot actual.

6. UPDATE entities
   SET data_json = :new_json,
       version = version + 1,
       updated_at = :now,
       updated_by = :actor_user_id
   WHERE tenant_id = :tenant_id
     AND entity_type = :entity_type
     AND entity_id = :entity_id
     AND version = :base_version;

7. INSERT INTO change_history (...).

8. Guardar response_json 200 en write_idempotency.

COMMIT;
```

#### 12.8.9 Reglas practicas de produccion

1. No aceptar `base_version` nula salvo en `create` inicial.
2. Usar hash del request para detectar reuse ilegal de `idempotency_key`.
3. No usar reloj cliente para resolver conflictos.
4. Mantener `change_seq` monotona para pull incremental.
5. Si el payload es grande, limitar numero maximo de operaciones JSON Patch por request.
6. Para deletes, usar soft-delete (`deleted_at`) y registrar patch explicito.

---

## 13) Decision corta

Para tu caso descrito (tenant simple, costo bajo, local-first real):
- Si, combinar Limbo + TinyBase es una muy buena estrategia.
- Pero hazlo con esquema hibrido (JSON + columnas/indices + edges), no JSON-only.

---

## 14) Arquitectura recomendada para Loro docs (CRDT)

Si usas Loro para colaboracion en documentos, conviene separar ingest de operaciones y lectura consolidada.

### 14.1 Patron recomendado
No pensar en "dos bases porque si", sino en Command/Projection:
- Capa de escritura/ingest (command side): recibe operaciones CRDT.
- Capa de merge: combina operaciones y construye estado canonico.
- Capa de lectura (projection side): consultas rapidas de estado consolidado.

Con tu requisito de producto, agregar una cuarta vista: tracking por usuario.
- Cada usuario debe poder ver solo sus cambios: pendientes, aceptados, rechazados y publicados.
- El tenant comparte el estado final consolidado, no necesariamente todo el historial operativo de cada usuario.

### 14.2 Fuente de verdad
La fuente de verdad debe ser el log de operaciones CRDT (no solo el snapshot final).

Guardar ambos:
- Log de operaciones (auditoria, replay, depuracion, idempotencia).
- Snapshot mergeado por documento (lectura y sync eficientes).

### 14.3 Validaciones por etapa

#### Antes de aceptar operacion (ingest)
- Auth del usuario.
- Permiso por tenant/documento.
- Formato y tamano del blob.
- Idempotencia por `operation_id`.

#### Antes/durante merge
- Causalidad (clock/version vector segun modelo).
- Deteccion de repetidos.
- Integridad de documento base.

#### Despues de merge
- Reglas de negocio semanticas.
- Marcar rechazo/compensacion si no pasa validaciones de dominio.

### 14.4 Modelo de tablas sugerido

#### `loro_ops` (ingest de operaciones)
- `op_id` (PK)
- `tenant_id`
- `doc_id`
- `actor_id`
- `op_blob`
- `clock_or_version`
- `created_at`
- `status` (`pending|applied|rejected`)

Nota:
- `status=applied` aqui significa "mergeado internamente".
- Conviene separar "mergeado" de "publicado a read model" para trazabilidad fina.

Indices sugeridos:
- `(tenant_id, doc_id, created_at)`
- `(tenant_id, status, created_at)`

#### `loro_docs` (estado consolidado)
- `tenant_id`
- `doc_id` (PK compuesto con tenant)
- `merged_blob`
- `merged_version`
- `updated_at`

#### `loro_user_sync_status` (tracking por usuario)
- `tenant_id`
- `user_id`
- `op_id`
- `doc_id`
- `ingest_status` (`received|validated|rejected`)
- `merge_status` (`pending|merged|failed`)
- `publish_status` (`pending|published`)
- `reject_reason` (nullable)
- `received_at`
- `updated_at`

Indices sugeridos:
- `(tenant_id, user_id, updated_at)`
- `(tenant_id, user_id, publish_status, updated_at)`
- `(tenant_id, op_id)`

### 14.5 Sync hacia clientes
Para distribuir cambios a clientes:
- Opcion A: enviar deltas de operaciones nuevas.
- Opcion B: enviar snapshot mergeado por documento.

En general:
- Deltas para colaboracion en vivo eficiente.
- Snapshots para bootstrap/recovery.

Para tu caso de UX por usuario:
- El cliente del usuario consulta su cola en `loro_user_sync_status` para ver "que ya se acepto" y "que sigue en transito".
- El estado compartido del tenant se lee de `loro_docs` (solo lectura para consumo general).

Flujo sugerido:
1. Usuario envia cambio -> `loro_ops` + `loro_user_sync_status(ingest_status=received)`.
2. Validador marca `validated` o `rejected` (con `reject_reason`).
3. Worker de merge aplica CRDT y marca `merge_status=merged`.
4. Proyeccion publica en `loro_docs` y marca `publish_status=published`.
5. Cliente recibe confirmacion por su `user_id` y actualiza UI de "enviado/aceptado/publicado".

### 14.6 Una base vs dos bases

#### Empezar (recomendado)
Una sola base con separacion logica (`loro_ops` y `loro_docs`).

#### Escalar despues
Separar fisicamente ingest/merge y read-model cuando:
- sube mucho el volumen,
- necesitas aislar cargas,
- o quieres topologias multi-region.

### 14.7 Conclusion practica
Si tu duda era "es mejor una DB de lectura buena y otra de escrituras para validar/mergear?":
- Si, el patron es valido.
- Pero el diseno correcto es log de operaciones como verdad + proyeccion mergeada para lectura.
- Si ademas necesitas visibilidad por usuario, agrega tracking explicito por `user_id` y estado de transito.
- Esa combinacion da seguridad operativa, trazabilidad y rendimiento.
