# Guia Final: Arquitectura Local-First Multi-Tenant

## Objetivo
Este documento resume solo la arquitectura elegida.

No incluye comparativas historicas, alternativas descartadas ni ramas exploratorias.

El alcance final es:
- backend en Rust,
- almacenamiento servidor con Limbo/libSQL,
- frontend en TypeScript/React,
- datos estructurados y relacionales,
- UX local-first con sincronizacion,
- permisos evaluados en backend,
- sin dependencias externas adicionales para colas o autorizacion remota.

---

## 1) Decisiones cerradas

### 1.1 Modelo de datos
- No se usan CRDTs.
- No se usa Loro.
- El modelo de concurrencia es control optimista con `version` monotona.
- El estado de negocio se modela como `snapshot actual + audit log`.

### 1.2 Autorizacion
- La autorizacion se evalua en backend con `cedar-policy` embebido en Rust.
- La UI usa una proyeccion de capacidades para renderizar vistas y acciones.
- La UI no es autoridad de seguridad.

### 1.3 Infraestructura
- No se usa Redis ni Valkey.
- No se usa una cola asincrona externa.
- El write pipe es sincronico, atomico y transaccional sobre SQLite/Limbo.

### 1.4 Topologia local en cliente
- Cada usuario tiene dos dominios de datos locales.
- `tenant_shared.db`: replica local del estado canonico compartido del tenant.
- `user_workspace.db`: base privada del usuario, sincronizada entre sus dispositivos.

### 1.5 Identidad y membresias
- La identidad no se duplica dentro de cada tenant.
- Existe una capa minima compartida para `users`, `sessions` y `tenant_memberships`.
- Un mismo `user_id` puede pertenecer a multiples tenants u organizaciones.
- Los datos de negocio y el sync siguen aislados por tenant.

### 1.6 Regla de confianza
- Toda DB local del cliente es manipulable.
- Ningun dato del cliente se considera valido por existir localmente.
- Solo el backend promueve cambios al estado compartido.

---

## 2) Principios del sistema

### 2.1 Verdad compartida
La verdad compartida vive en servidor.

Se separa en dos dominios:
- control plane minimo compartido para identidad, sesiones y memberships,
- data plane aislado por tenant para negocio, snapshots, historial y sync.

### 2.2 Estado local
El cliente mantiene estado local para:
- lectura offline,
- UI optimista,
- drafts,
- outbox,
- vistas de capacidades.

### 2.3 Validacion
Todo write del cliente se revalida en servidor antes de persistirse.

### 2.4 Concurrencia
La concurrencia se controla con compare-and-swap sobre `version`.

---

## 3) Topologia final

### 3.1 Cliente

#### `tenant_shared.db`
Replica local del estado aprobado del tenant.

Se usa para:
- lectura offline,
- queries locales,
- indices,
- navegacion de recursos compartidos.

No se usa para:
- staging,
- autoridad,
- permisos autoritativos,
- drafts privados.

#### `user_workspace.db`
Base privada del usuario, sincronizada entre sus dispositivos.

Se usa para:
- drafts,
- outbox,
- errores de validacion,
- estados pendientes,
- capacidades efectivas del usuario,
- vistas privadas de administracion si aplica.

### 3.2 Servidor
El servidor mantiene:
- una DB minima compartida de identidad y memberships,
- snapshots materializados por entidad,
- historial append-only de cambios,
- tabla de idempotencia,
- tablas locales de grants y permisos efectivos por tenant,
- evaluacion de Cedar,
- sync hacia cliente de estados aprobados y proyecciones privadas.

### 3.3 Control plane minimo compartido
La capa compartida no almacena datos de negocio del tenant.

Solo mantiene:
- usuarios globales,
- sesiones,
- memberships usuario-tenant,
- invitaciones,
- metadata operativa minima.

Tablas sugeridas:

```sql
CREATE TABLE users (
  user_id TEXT PRIMARY KEY,
  email TEXT NOT NULL UNIQUE,
  password_hash TEXT NOT NULL,
  status TEXT NOT NULL,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL
);

CREATE TABLE sessions (
  session_id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  expires_at INTEGER NOT NULL,
  created_at INTEGER NOT NULL,
  revoked_at INTEGER,
  FOREIGN KEY (user_id) REFERENCES users(user_id)
);

CREATE TABLE tenant_memberships (
  tenant_id TEXT NOT NULL,
  user_id TEXT NOT NULL,
  role TEXT NOT NULL,
  status TEXT NOT NULL,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  PRIMARY KEY (tenant_id, user_id),
  FOREIGN KEY (user_id) REFERENCES users(user_id)
);
```

### 3.3.1 Esquema recomendado del control plane

DDL mas completo para produccion:

```sql
CREATE TABLE users (
  user_id TEXT PRIMARY KEY,
  email TEXT NOT NULL UNIQUE,
  password_hash TEXT NOT NULL,
  status TEXT NOT NULL,                 -- active | invited | suspended
  display_name TEXT,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL
);

CREATE TABLE tenants (
  tenant_id TEXT PRIMARY KEY,
  slug TEXT NOT NULL UNIQUE,
  display_name TEXT NOT NULL,
  status TEXT NOT NULL,                 -- active | suspended
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL
);

CREATE TABLE tenant_memberships (
  tenant_id TEXT NOT NULL,
  user_id TEXT NOT NULL,
  role TEXT NOT NULL,
  status TEXT NOT NULL,                 -- active | invited | revoked
  joined_at INTEGER,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  PRIMARY KEY (tenant_id, user_id),
  FOREIGN KEY (tenant_id) REFERENCES tenants(tenant_id),
  FOREIGN KEY (user_id) REFERENCES users(user_id)
);

CREATE TABLE sessions (
  session_id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  active_tenant_id TEXT,
  expires_at INTEGER NOT NULL,
  created_at INTEGER NOT NULL,
  revoked_at INTEGER,
  FOREIGN KEY (user_id) REFERENCES users(user_id),
  FOREIGN KEY (active_tenant_id) REFERENCES tenants(tenant_id)
);

CREATE TABLE tenant_invitations (
  invitation_id TEXT PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  email TEXT NOT NULL,
  invited_role TEXT NOT NULL,
  invited_by_user_id TEXT NOT NULL,
  status TEXT NOT NULL,                 -- pending | accepted | expired | revoked
  expires_at INTEGER NOT NULL,
  created_at INTEGER NOT NULL,
  accepted_at INTEGER,
  FOREIGN KEY (tenant_id) REFERENCES tenants(tenant_id),
  FOREIGN KEY (invited_by_user_id) REFERENCES users(user_id)
);

CREATE INDEX idx_tenant_memberships_user
  ON tenant_memberships (user_id, status, updated_at DESC);

CREATE INDEX idx_sessions_user
  ON sessions (user_id, expires_at DESC);

CREATE INDEX idx_tenant_invitations_tenant_email
  ON tenant_invitations (tenant_id, email, status);
```

Regla operativa:
- `users` y `sessions` son globales.
- `tenant_memberships` define a que orgs pertenece un usuario.
- el tenant activo de una sesion se puede fijar al hacer login o cambiar despues.

### 3.3.2 Flujo de autenticacion y seleccion de tenant

Cuando un usuario puede pertenecer a varias orgs, el flujo correcto no es asumir un tenant unico en login.

Flujo recomendado:

1. `POST /v1/auth/login`
   El usuario envia `email` y `password`.

2. El backend valida credenciales contra `users`.

3. El backend consulta `tenant_memberships` activas de ese `user_id`.

4. Si el usuario pertenece a un solo tenant:
   crea sesion con `active_tenant_id` y devuelve contexto completo.

5. Si el usuario pertenece a varios tenants:
   crea sesion sin tenant activo o con tenant por defecto y devuelve lista de orgs elegibles.

6. El cliente llama `POST /v1/auth/select-tenant` para fijar el tenant activo si hace falta.

7. A partir de ahi, cada request autenticada se resuelve con:
   - `user_id` global,
   - `session_id`,
   - `active_tenant_id`,
   - membership valida para ese tenant.

### 3.3.3 Contrato sugerido de login

Request:

```json
{
  "email": "ana@empresa.com",
  "password": "<secret>"
}
```

Respuesta si el usuario tiene multiples orgs:

```json
{
  "ok": true,
  "session": {
    "session_id": "sess_01J...",
    "user_id": "user_123",
    "active_tenant_id": null,
    "expires_at": 1777689999999
  },
  "user": {
    "user_id": "user_123",
    "email": "ana@empresa.com",
    "display_name": "Ana"
  },
  "memberships": [
    {
      "tenant_id": "tenant_a",
      "slug": "inmobiliaria-norte",
      "display_name": "Inmobiliaria Norte",
      "role": "admin"
    },
    {
      "tenant_id": "tenant_b",
      "slug": "club-tenis-sur",
      "display_name": "Club de Tenis Sur",
      "role": "viewer"
    }
  ]
}
```

Seleccion de tenant:

```json
POST /v1/auth/select-tenant

{
  "tenant_id": "tenant_a"
}
```

Respuesta:

```json
{
  "ok": true,
  "session": {
    "session_id": "sess_01J...",
    "user_id": "user_123",
    "active_tenant_id": "tenant_a",
    "expires_at": 1777689999999
  },
  "membership": {
    "tenant_id": "tenant_a",
    "role": "admin",
    "status": "active"
  }
}
```

Regla de seguridad:
- el cliente puede proponer `tenant_id`,
- pero el backend solo lo acepta si existe `tenant_memberships(tenant_id, user_id)` activa,
- y despues todas las operaciones de negocio se ejecutan contra la DB de ese tenant.

### 3.4 Tenant DB de negocio
Cada tenant mantiene su propia DB para negocio y sync.

Dentro de esa DB viven:
- `entities`,
- `change_history`,
- `write_idempotency`,
- `document_grants` y otras tablas de negocio del tenant,
- proyecciones para sync compartido y privado.

---

## 4) Modelo de confianza y seguridad

### 4.1 Regla fundamental
El cliente puede:
- editar sus DB locales,
- falsificar payloads,
- reintentar requests,
- modificar UI o saltarse CASL.

Por lo tanto:
- `tenant_shared.db` no es autoridad,
- `user_workspace.db` no es autoridad,
- la UI no es autoridad,
- CASL no es autoridad.

La identidad del usuario tampoco se deduce desde el cliente.
El backend la obtiene desde sesion o token firmado y luego resuelve la membership del tenant activo.

### 4.2 Enforcements reales
La seguridad real ocurre en backend:
- autenticacion,
- resolucion de `user_id` global,
- resolucion de membership activa para el tenant,
- evaluacion de Cedar,
- validacion de tenant,
- validacion de integridad de payload,
- validacion de `base_version`,
- idempotencia,
- reglas de dominio.

---

## 5) Permisos

### 5.1 Motor de autorizacion
Se usa `cedar-policy` embebido dentro del backend Rust.

Esto permite:
- soberania de infraestructura,
- baja latencia,
- versionado de politicas,
- evitar una llamada de red externa para autorizar.

### 5.2 Rol de la UI
La UI renderiza acciones y vistas a partir de capacidades proyectadas.

Puede decidir:
- mostrar u ocultar botones,
- habilitar o deshabilitar acciones,
- abrir o no ciertas vistas.

Pero toda accion real se vuelve a validar en backend.

### 5.3 Proyeccion de capacidades
La proyeccion efectiva del usuario se sincroniza a `user_workspace.db`.

Esta proyeccion se calcula a partir de:
- `user_id` global,
- membership del usuario en el tenant activo,
- grants locales del tenant,
- reglas Cedar vigentes.

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

### 5.4 Administracion de permisos
La interfaz no modifica permisos escribiendo directo a una DB sincronizada.

La interfaz manda comandos al backend.

Ejemplos:
- `POST /tenants/:tenant_id/members/:user_id/roles`
- `POST /documents/:doc_id/grants`
- `DELETE /documents/:doc_id/grants/:grant_id`

### 5.5 Datos autoritativos de permisos
El backend mantiene dos niveles de datos autoritativos:
- memberships en la capa compartida,
- grants y auditoria especifica dentro de cada tenant DB.

```sql
CREATE TABLE document_grants (
  grant_id TEXT PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  doc_id TEXT NOT NULL,
  subject_type TEXT NOT NULL,
  subject_id TEXT NOT NULL,
  relation TEXT NOT NULL,
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

---

## 6) Modelo de persistencia del dominio

### 6.1 Snapshot materializado
La tabla principal de negocio contiene el estado actual por entidad dentro de cada tenant DB.

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
```

### 6.2 Audit log append-only
Cada cambio aprobado queda registrado en historial dentro de la DB del tenant correspondiente.

```sql
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
  patch_format TEXT NOT NULL,
  delta_json TEXT NOT NULL CHECK (json_valid(delta_json)),
  resulting_json TEXT NOT NULL CHECK (json_valid(resulting_json)),
  policy_version TEXT,
  created_at INTEGER NOT NULL
);

CREATE INDEX idx_change_history_entity_version
  ON change_history (tenant_id, entity_type, entity_id, new_version);

CREATE INDEX idx_change_history_tenant_seq
  ON change_history (tenant_id, change_seq);
```

### 6.3 Idempotencia
La idempotencia se persiste en una tabla dedicada dentro de la DB del tenant.

```sql
CREATE TABLE write_idempotency (
  tenant_id TEXT NOT NULL,
  actor_user_id TEXT NOT NULL,
  idempotency_key TEXT NOT NULL,
  request_hash TEXT NOT NULL,
  entity_type TEXT NOT NULL,
  entity_id TEXT NOT NULL,
  status TEXT NOT NULL,
  http_status INTEGER NOT NULL,
  response_json TEXT NOT NULL CHECK (json_valid(response_json)),
  created_at INTEGER NOT NULL,
  completed_at INTEGER,
  PRIMARY KEY (tenant_id, actor_user_id, idempotency_key)
);

CREATE INDEX idx_write_idempotency_entity
  ON write_idempotency (tenant_id, entity_type, entity_id, created_at DESC);
```

### 6.4 Formato de delta
El formato canonico de `delta_json` es `JSON Patch`.

```json
[
  { "op": "replace", "path": "/status", "value": "approved" },
  { "op": "replace", "path": "/amount", "value": 12500 },
  { "op": "add", "path": "/tags/-", "value": "urgent" }
]
```

---

## 7) Write pipe final

### 7.1 Regla de concurrencia
No se usa el reloj del cliente para resolver conflictos.

La regla correcta es:
- el cliente envia `base_version`,
- el backend actualiza solo si la fila sigue en esa version,
- si no coincide, responde `409 Conflict`.

### 7.2 Semantica de escritura
El write pipe es:
- sincronico,
- atomico,
- transaccional,
- con control de concurrencia optimista.

Cada write debe quedar ligado a:
- `user_id` global,
- `session_id` o token valido,
- `tenant_id` activo,
- membership valida en ese tenant.

### 7.3 Request de `POST /v2/write`

La identidad del actor no viaja como autoridad en el body.
El backend la deriva desde la sesion o JWT y del tenant activo resuelto en control plane.

```json
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

### 7.4 Respuesta `200 OK`

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

### 7.5 Respuesta `409 Conflict`

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

### 7.6 Respuesta `403 Forbidden`

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

### 7.7 Flujo transaccional

```text
0. Resolver sesion o JWT.
1. Obtener `user_id` global.
2. Resolver `tenant_id` activo.
3. Verificar membership activa del usuario en ese tenant.

BEGIN IMMEDIATE;

4. Reservar idempotency key en write_idempotency.
   - Si ya existe con mismo request_hash -> devolver respuesta cacheada.
   - Si ya existe con distinto request_hash -> conflicto de idempotencia.

5. Evaluar Cedar.
   - Si falla -> guardar respuesta 403 y COMMIT.

6. Leer estado actual de entities.

7. Si current.version != base_version:
   - construir respuesta 409,
   - guardar respuesta en write_idempotency,
   - COMMIT.

8. Aplicar JSON Patch al snapshot actual.

9. UPDATE entities ... WHERE version = :base_version.

10. INSERT INTO change_history (...).

11. Guardar respuesta 200 en write_idempotency.

COMMIT;
```

---

## 8) Flujo de sincronizacion

### 8.1 Push de usuario
El cliente edita sobre `user_workspace.db` y envia su outbox al backend.

### 8.2 Pull compartido
El cliente recibe snapshots aprobados en `tenant_shared.db`.

### 8.3 Pull privado
El cliente recibe tambien su estado privado:
- capacidades,
- estados pendientes,
- resultados de validacion,
- metadata de resolucion.

### 8.4 Reconciliacion
Cuando el servidor acepta un cambio:
- el cliente purga ese write del outbox,
- actualiza el draft local si hace falta,
- recibe el nuevo snapshot en `tenant_shared.db`.

Cuando el servidor rechaza o detecta conflicto:
- el draft queda en `user_workspace.db`,
- se guarda el error,
- la UI ofrece rebase, retry o resolucion manual.

---

## 9) Reglas operativas

1. No aceptar `base_version` nula salvo en creacion inicial.
2. No usar timestamps del cliente como autoridad.
3. Persistir idempotencia para `200`, `403` y `409`.
4. Mantener `change_seq` monotona para pull incremental.
5. Limitar el numero de operaciones JSON Patch por request.
6. Usar soft delete con `deleted_at`.
7. Mantener transacciones cortas.
8. Hacer particion natural por tenant si la carga crece.
9. No duplicar identidad por tenant si un usuario puede pertenecer a varias orgs.
10. Resolver siempre el tenant activo a partir de la sesion o token y la membership del usuario.

---

## 10) Conclusion final

La arquitectura final queda asi:
- una capa minima compartida para identidad, sesiones y memberships,
- `tenant_shared.db` para lectura compartida del estado aprobado,
- `user_workspace.db` para drafts, outbox y estado privado del usuario,
- DBs de negocio aisladas por tenant para snapshots, historial e idempotencia,
- backend Rust con Limbo/libSQL como plano de control y plano de datos,
- `cedar-policy` embebido como motor de autorizacion,
- control de concurrencia por `version` con compare-and-swap,
- audit log append-only con `JSON Patch`,
- write pipe sincronico y atomico,
- cliente usado para UX y offline, nunca como autoridad.