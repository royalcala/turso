# Alternativa Recomendada: AI-First con React + TanStack Router

## Objetivo
Definir una alternativa recomendada para la capa frontend AI-first usando React en lugar de Svelte, manteniendo:
- soberania de infraestructura,
- costos operativos bajos,
- backend Rust como autoridad,
- runtime declarativo basado en documentos JSON,
- soporte offline-first,
- exposicion HTTP/MCP sin depender de la interfaz.

---

## 1) Resumen ejecutivo

Para este contexto, la alternativa recomendada es:

1. Vite + React SPA
2. TanStack Router
3. Runtime declarativo (DSL + JSON Schema)
4. Backend Rust (Axum/Actix) + Limbo/libSQL + Cedar
5. Sin servidor Node.js en produccion

Motivo principal:
- reduce friccion para Server-Driven UI y Generative UI,
- tiene ecosistema maduro para UI dinamica,
- mantiene el mismo modelo de seguridad y datos ya definido.

---

## 2) Decision de stack

### 2.1 Frontend
- `React` para renderer de arboles UI dinamicos en runtime.
- `TanStack Router` para enrutamiento tipado y resolucion de rutas dinamicas desde DB.
- `Vite` para build estatico de la SPA.

### 2.2 Backend
- Rust para auth, autorizacion, validacion y write pipe.
- Cedar embebido para decisiones de permiso.
- DBs de tenant para artefactos publicados y datos de negocio.
- Workspace privado por usuario para drafts.

### 2.3 Runtime
- No se ejecuta codigo React arbitrario del usuario.
- El usuario/IA produce documentos declarativos validados.
- El renderer React interpreta ese DSL con un registry de componentes permitidos.

---

## 3) Por que React + TanStack Router encaja bien

### 3.1 Generative UI y SDUI
React se adapta naturalmente a:
- transformar JSON DSL en arbol de componentes,
- renderizar layouts dinamicos,
- componer formularios y vistas en runtime.

### 3.2 Enrutamiento dinamico
TanStack Router permite:
- mantener rutas de infraestructura estaticas,
- resolver rutas de negocio desde `runtime_routes`,
- trabajar con tipado fuerte en params y loaders,
- integrar guards por capacidad y tenant activo.

### 3.3 Operacion soberana
Con Vite SPA:
- solo se sirven estaticos en produccion,
- no necesitas contenedores Node para SSR,
- el backend Rust mantiene control total de auth, permisos y datos.

---

## 4) Rutas: modelo operativo

### 4.1 Rutas estaticas minimas
- `/login`
- `/select-tenant`
- `/app`
- `/app/*` (catch-all de runtime)

### 4.2 Rutas de negocio declarativas
Las rutas publicadas viven por tenant en la DB de negocio:

```sql
CREATE TABLE runtime_routes (
  route_id TEXT PRIMARY KEY,
  route_pattern TEXT NOT NULL UNIQUE,
  page_id TEXT NOT NULL,
  route_kind TEXT NOT NULL,             -- static | pattern
  priority INTEGER NOT NULL,
  is_published INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  updated_by TEXT NOT NULL
);
```

### 4.3 Flujo de resolucion
1. El usuario entra a `/app/...`.
2. El loader resuelve sesion y tenant activo.
3. Busca ruta en `runtime_routes` del tenant.
4. Carga `page_definition`.
5. Renderiza con `PageRenderer`.

---

## 5) Modelo declarativo (sin codigo arbitrario)

### 5.1 Artefactos principales
- `module_definition`
- `entity_schema`
- `page_definition`
- `form_definition`
- `action_definition`
- `api_contract_definition`
- `mcp_tool_definition`

### 5.2 Regla de seguridad
No se permite:
- JSX/TS libre generado por usuario,
- handlers backend arbitrarios,
- SQL libre,
- plugins MCP ejecutables sin sandbox.

Todo contrato se valida contra JSON Schema y bindings permitidos.

---

## 6) Offline-first y separacion de estados

### 6.1 Publicado (tenant)
En la DB del tenant se guardan:
- rutas publicadas,
- paginas publicadas,
- formularios publicados,
- contratos API/MCP publicados,
- datos de negocio aprobados.

### 6.2 Drafts (usuario)
En `user_workspace.db` se guardan:
- drafts de paginas,
- drafts de rutas,
- drafts de contratos,
- previews privadas,
- cambios pendientes de publicar.

Como un usuario puede pertenecer a multiples orgs, los drafts privados incluyen `tenant_id`.

---

## 7) API y MCP sin UI

### 7.1 Principio
Los contratos publicados se consumen por:
- interfaz web,
- clientes HTTP,
- herramientas MCP,
- agentes IA.

### 7.2 Base canonica
Usar JSON Schema para:
- `input_schema`
- `output_schema`

El contrato apunta a un `binding.kind` permitido por plataforma.

### 7.3 Publicacion derivada
1. Usuario/IA crea contrato (draft).
2. Backend valida schema + permisos + binding.
3. Se publica en tenant DB.
4. Runtime HTTP expone endpoint derivado.
5. Runtime MCP expone tool derivada.

HTTP y MCP comparten el mismo contrato y control de permisos.

---

## 8) Integracion con permisos

Toda operacion del runtime valida:
- sesion/token,
- tenant activo,
- membership activa,
- capacidad requerida,
- reglas Cedar.

Capacidades sugeridas:
- `can_manage_runtime`
- `can_publish_runtime`
- `can_manage_api_contracts`
- `can_invoke_runtime_api`
- `can_invoke_runtime_mcp`

---

## 9) Recomendacion final

Esta alternativa es recomendada cuando priorizas:
- entrega rapida,
- ecosistema maduro para UI dinamica,
- tooling AI-first actual,
- despliegue simple con backend Rust sirviendo estaticos.

Decision final de esta alternativa:

1. React + TanStack Router para runtime de UI declarativa.
2. DSL + JSON Schema como contrato canónico.
3. Backend Rust + Cedar como autoridad.
4. DB por tenant para publicado, workspace privado para drafts.
5. Exposicion HTTP/MCP derivada de contratos validados.

En resumen:

no se programa interfaz o API con codigo libre del usuario;
se describen artefactos declarativos y la plataforma los ejecuta de forma segura.