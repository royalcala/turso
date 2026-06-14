# Guia Final: Runtime AI-First con Svelte

## Objetivo
Este documento define la arquitectura propuesta para permitir que usuarios creen:
- modulos,
- formularios,
- paginas customizadas,
- componentes compuestos,
- vistas y acciones,

sin ejecutar codigo arbitrario del usuario en el cliente.

La idea central es:
- usar Svelte como runtime de render,
- guardar definiciones declarativas en JSON,
- sincronizarlas offline con la arquitectura local-first ya definida,
- permitir que una IA genere y edite esas definiciones,
- mantener el sistema seguro, portable y operable.

---

## 1) Decision central

La propuesta no es que el usuario o la IA escriban componentes Svelte libres.

La propuesta es:
- un DSL declarativo versionado,
- un renderer controlado en Svelte,
- un catalogo cerrado de primitives de UI,
- validacion estricta antes de publicar,
- sincronizacion offline de drafts y publicaciones.

En una frase:

el usuario describe la aplicacion, la plataforma la materializa.

---

## 2) Principios del runtime

### 2.1 Nada de codigo arbitrario
No se guarda ni ejecuta:
- Svelte dinamico del usuario,
- JavaScript arbitrario,
- expresiones libres sin sandbox,
- plugins de terceros ejecutables dentro del runtime principal.

### 2.2 Todo es declarativo
La UI, formularios, modulos y acciones se representan como documentos estructurados.

### 2.3 AI-first real
La IA no genera componentes ejecutables.
La IA genera documentos DSL validos.

### 2.4 Offline-first natural
Los documentos del runtime se pueden:
- editar offline,
- guardar como draft,
- sincronizar,
- validar,
- publicar,
- rehidratar localmente.

### 2.5 Seguridad de plataforma
La plataforma controla:
- que componentes existen,
- que acciones estan permitidas,
- que bindings de datos son validos,
- que eventos puede disparar una pagina,
- que permisos necesita cada accion.

---

## 3) Capas del sistema

### 3.1 Catalogo de componentes
Conjunto cerrado de primitives renderizables por Svelte.

Ejemplos:
- `heading`
- `text`
- `stack`
- `grid`
- `card`
- `table`
- `form`
- `text_input`
- `number_input`
- `select`
- `checkbox`
- `tabs`
- `metric`
- `chart`
- `modal`
- `kanban`
- `timeline`

Cada primitive tiene:
- `kind`
- `props_schema`
- `data_contract`
- `allowed_events`
- `capability_requirements`

### 3.2 DSL declarativo
Los usuarios y la IA crean documentos JSON que referencian primitives del catalogo.

Tipos principales de documento:
- `module_definition`
- `entity_schema`
- `page_definition`
- `form_definition`
- `view_definition`
- `action_definition`
- `api_contract_definition`
- `mcp_tool_definition`
- `workflow_definition`
- `theme_definition`

### 3.3 Renderer Svelte
El frontend carga documentos DSL y los interpreta con componentes Svelte internos.

El renderer se encarga de:
- resolver layouts,
- inyectar datos,
- evaluar visibilidad,
- mapear eventos,
- construir formularios,
- renderizar tablas y vistas,
- despachar acciones autorizadas.

### 3.4 Capa de validacion
Antes de guardar o publicar:
- se valida el JSON contra schema,
- se revisan referencias a entidades y campos,
- se revisan permisos requeridos,
- se revisan limites de complejidad,
- se rechazan estructuras no soportadas.

---

## 4) Modelo de documentos

### 4.1 `entity_schema`
Define el modelo logico de una entidad sin requerir `ALTER TABLE` por cada cambio.

Ejemplo:

```json
{
  "type": "entity_schema",
  "schema_id": "invoice",
  "version": 3,
  "title": "Invoice",
  "fields": [
    {
      "name": "customer_name",
      "type": "string",
      "required": true,
      "indexed": true
    },
    {
      "name": "amount",
      "type": "number",
      "required": true,
      "indexed": true
    },
    {
      "name": "status",
      "type": "enum",
      "values": ["draft", "sent", "paid"],
      "required": true,
      "indexed": true
    }
  ]
}
```

### 4.1.1 `module_definition`
Define un modulo funcional completo dentro del tenant.

Un modulo agrupa:
- entidades,
- paginas,
- formularios,
- rutas,
- acciones,
- vistas,
- contratos API,
- herramientas MCP.

Ejemplo:

```json
{
  "type": "module_definition",
  "module_id": "sales",
  "title": "Ventas",
  "version": 1,
  "entities": ["invoice", "customer"],
  "pages": ["sales_dashboard", "invoice_detail"],
  "forms": ["invoice_form"],
  "routes": ["sales_dashboard_route", "invoice_detail_route"],
  "actions": ["approve_invoice"],
  "api_contracts": ["sales_invoice_query"],
  "mcp_tools": ["get_invoice", "list_invoices"]
}
```

Regla:
- el modulo no ejecuta logica por si mismo,
- el modulo solo agrupa y versiona artefactos declarativos ya soportados por la plataforma.

### 4.2 `page_definition`
Describe una pagina completa.

Ejemplo:

```json
{
  "type": "page_definition",
  "page_id": "sales_dashboard",
  "title": "Ventas",
  "layout": {
    "kind": "stack",
    "children": [
      {
        "kind": "heading",
        "props": {
          "text": "Resumen de ventas"
        }
      },
      {
        "kind": "table",
        "props": {
          "source": "invoice",
          "columns": [
            { "field": "customer_name", "label": "Cliente" },
            { "field": "status", "label": "Estado" },
            { "field": "amount", "label": "Monto" }
          ]
        }
      }
    ]
  }
}
```

### 4.3 `form_definition`
Describe un formulario declarativo.

```json
{
  "type": "form_definition",
  "form_id": "invoice_form",
  "entity": "invoice",
  "fields": [
    {
      "name": "customer_name",
      "component": "text_input",
      "label": "Cliente",
      "required": true
    },
    {
      "name": "amount",
      "component": "number_input",
      "label": "Monto",
      "required": true
    },
    {
      "name": "status",
      "component": "select",
      "label": "Estado",
      "options": ["draft", "sent", "paid"]
    }
  ],
  "actions": [
    { "kind": "save_draft" },
    { "kind": "submit" }
  ]
}
```

### 4.4 `action_definition`
Describe una accion permitida por la plataforma.

```json
{
  "type": "action_definition",
  "action_id": "approve_invoice",
  "kind": "entity_mutation",
  "entity": "invoice",
  "requires": ["can_write"],
  "mutation": {
    "patch": [
      { "op": "replace", "path": "/status", "value": "paid" }
    ]
  }
}
```

### 4.5 `api_contract_definition`
Describe un contrato de API declarativo publicable desde frontend o IA.

La recomendacion es usar JSON Schema como base canonica para input y output.

No se recomienda permitir al usuario definir handlers arbitrarios.
El contrato debe mapear a capacidades soportadas por la plataforma:
- query de registros,
- lectura de entidad,
- mutacion declarativa,
- accion publicada,
- workflow permitido.

Ejemplo:

```json
{
  "type": "api_contract_definition",
  "contract_id": "sales_invoice_query",
  "version": 1,
  "transport": "http_json",
  "method": "POST",
  "path": "/api/runtime/sales/invoices/query",
  "capabilities_required": ["can_read"],
  "input_schema": {
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "type": "object",
    "properties": {
      "status": { "type": "string" },
      "limit": { "type": "integer", "minimum": 1, "maximum": 100 }
    },
    "additionalProperties": false
  },
  "output_schema": {
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "type": "object",
    "properties": {
      "items": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "record_id": { "type": "string" },
            "status": { "type": "string" },
            "amount": { "type": "number" }
          },
          "required": ["record_id", "status", "amount"]
        }
      }
    },
    "required": ["items"]
  },
  "binding": {
    "kind": "record_query",
    "entity": "invoice",
    "select": ["record_id", "status", "amount"],
    "filters": ["status"],
    "default_limit": 20
  }
}
```

### 4.6 `mcp_tool_definition`
Describe una herramienta MCP publicada por el tenant.

La idea correcta es que el MCP no ejecute codigo arbitrario definido por el usuario.
Debe exponer herramientas derivadas de contratos y acciones declarativas ya validadas.

Ejemplo:

```json
{
  "type": "mcp_tool_definition",
  "tool_name": "get_invoice",
  "title": "Obtener factura",
  "description": "Devuelve una factura por id",
  "input_schema": {
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "type": "object",
    "properties": {
      "invoice_id": { "type": "string" }
    },
    "required": ["invoice_id"],
    "additionalProperties": false
  },
  "output_schema": {
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "type": "object"
  },
  "binding": {
    "kind": "record_get",
    "entity": "invoice",
    "id_arg": "invoice_id"
  },
  "capabilities_required": ["can_read"]
}
```

---

## 5) Propuesta de almacenamiento

### 5.0 Regla de particion
En esta arquitectura, las paginas, formularios, rutas, schemas y registros publicados viven dentro de la DB de negocio del tenant.

Eso significa:
- dentro de `tenant_shared.db`, `tenant_id` es implicito por contexto de base de datos,
- no hace falta repetir `tenant_id` en todas las tablas internas del runtime del tenant,
- `tenant_id` solo vuelve a ser necesario en tablas compartidas o en workspaces privados que puedan contener drafts de multiples tenants para un mismo usuario.

### 5.1 Tipos de documento a sincronizar
Documentos del runtime:
- `module_definition`
- `entity_schema`
- `page_definition`
- `form_definition`
- `view_definition`
- `action_definition`
- `api_contract_definition`
- `mcp_tool_definition`
- `workflow_definition`
- `theme_definition`

Documentos de datos reales:
- `records`
- `draft_records`
- `published_records`

### 5.2 Separacion entre compartido y privado

#### `tenant_shared.db`
Contiene:
- modulos publicados,
- schemas publicados,
- paginas publicadas,
- rutas publicadas,
- formularios publicados,
- vistas publicadas,
- acciones publicadas,
- contratos API publicados,
- herramientas MCP publicadas,
- temas publicados,
- datos aprobados compartidos.

#### `user_workspace.db`
Contiene:
- drafts de modulos,
- drafts de paginas,
- drafts de rutas,
- drafts de formularios,
- drafts de contratos API,
- drafts de herramientas MCP,
- prototipos generados por IA,
- cambios pendientes de publicar,
- previews privadas,
- borradores de registros,
- proyecciones de capacidades del usuario.

Nota:
- como un usuario puede pertenecer a varios tenants, `user_workspace.db` si puede requerir `tenant_id` para separar drafts privados por org.

### 5.3 Estructura base sugerida

#### Documentos publicados dentro de la DB del tenant

```sql
CREATE TABLE runtime_documents (
  doc_type TEXT NOT NULL,
  doc_id TEXT NOT NULL,
  status TEXT NOT NULL,                 -- draft | published | archived
  content_json TEXT NOT NULL CHECK (json_valid(content_json)),
  schema_version INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  updated_by TEXT NOT NULL,
  PRIMARY KEY (doc_type, doc_id, status)
);

CREATE INDEX idx_runtime_documents_type_status
  ON runtime_documents (doc_type, status, updated_at DESC);
```

#### Rutas publicadas dentro de la DB del tenant

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

CREATE INDEX idx_runtime_routes_lookup
  ON runtime_routes (is_published, priority DESC, updated_at DESC);
```

Regla:
- cada tenant define sus propias rutas dentro de su propia DB,
- no existe colision entre tenants porque no comparten esa base,
- el shell Svelte resuelve la ruta contra la `runtime_routes` del tenant activo.

#### Drafts privados en `user_workspace.db`

```sql
CREATE TABLE runtime_document_drafts (
  tenant_id TEXT NOT NULL,
  doc_type TEXT NOT NULL,
  doc_id TEXT NOT NULL,
  content_json TEXT NOT NULL CHECK (json_valid(content_json)),
  based_on_version INTEGER,
  updated_at INTEGER NOT NULL,
  updated_by TEXT NOT NULL,
  PRIMARY KEY (tenant_id, doc_type, doc_id)
);

CREATE TABLE runtime_route_drafts (
  tenant_id TEXT NOT NULL,
  route_id TEXT NOT NULL,
  route_pattern TEXT NOT NULL,
  page_id TEXT NOT NULL,
  priority INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  updated_by TEXT NOT NULL,
  PRIMARY KEY (tenant_id, route_id)
);
```

### 5.4 Datos reales de negocio
Los registros reales siguen viviendo como documentos JSON, pero con proyecciones indexadas.

Ejemplo:

```sql
CREATE TABLE records (
  entity_type TEXT NOT NULL,
  record_id TEXT NOT NULL,
  record_json TEXT NOT NULL CHECK (json_valid(record_json)),
  status TEXT,
  owner_user_id TEXT,
  updated_at INTEGER NOT NULL,
  updated_by TEXT NOT NULL,
  version INTEGER NOT NULL,
  PRIMARY KEY (entity_type, record_id)
);

CREATE INDEX idx_records_listing
  ON records (entity_type, status, updated_at DESC);
```

### 5.5 Modelo de rutas
Las rutas de negocio del runtime no se representan como archivos Svelte individuales.

Se representan como documentos y filas de ruteo dentro de la DB del tenant.

Ejemplo de `page_definition`:

```json
{
  "type": "page_definition",
  "page_id": "sales_dashboard",
  "title": "Ventas",
  "route": "/ventas/dashboard",
  "layout": {
    "kind": "stack",
    "children": [
      {
        "kind": "heading",
        "props": { "text": "Dashboard" }
      }
    ]
  }
}
```

Ejemplo de patron dinamico:

```json
{
  "route_pattern": "/ventas/facturas/:invoiceId",
  "page_id": "invoice_detail",
  "route_kind": "pattern",
  "priority": 100
}
```

Regla operativa:
- SvelteKit solo necesita un shell estatico y una ruta catch-all,
- el matching real de paginas se hace contra `runtime_routes` de la DB del tenant activo,
- por eso no hace falta un archivo fisico por pagina creada por usuario o IA.

---

## 6) Runtime Svelte

### 6.1 Enfoque
Svelte no se usa como lenguaje libre generado por el usuario.
Svelte se usa como motor interno para renderizar el DSL.

### 6.2 Estructura recomendada del renderer

Capas:
- `Registry`: mapa de `kind -> Svelte component`
- `Renderer`: recibe nodo DSL y despacha al componente correcto
- `DataBinder`: resuelve datos y colecciones
- `ActionDispatcher`: ejecuta acciones permitidas
- `CapabilityGate`: decide visibilidad y habilitacion
- `FormEngine`: controla estado, validacion y drafts

### 6.3 Patron de renderer

Ejemplo conceptual:

```ts
type UiNode = {
  kind: string;
  props?: Record<string, unknown>;
  children?: UiNode[];
};
```

El renderer:
- recibe `UiNode`,
- busca el `kind` en un registry,
- monta el componente Svelte correspondiente,
- le pasa `props`, `children`, `context`, `capabilities` y `bindings`.

### 6.4 Ventaja de Svelte aqui
Svelte encaja bien porque:
- el runtime puede ser liviano,
- el mapping declarativo a componentes es simple,
- el costo de reactividad es bajo,
- es muy bueno para UIs generadas por configuracion.

### 6.5 Builder de modulos y contratos
El frontend puede ofrecer builders para:
- crear modulos,
- crear schemas,
- crear paginas,
- crear rutas,
- crear formularios,
- crear contratos API,
- crear herramientas MCP.

Todos esos builders deben escribir documentos declarativos y guardarlos como draft en `user_workspace.db`.

El backend solo publica artefactos que pasen validacion.

---

## 7) AI-first real

### 7.1 Rol de la IA
La IA debe producir:
- modulos,
- definiciones de entidades,
- formularios,
- paginas,
- contratos API,
- herramientas MCP,
- acciones,
- filtros,
- layouts,
- automatizaciones declarativas.

No debe producir:
- Svelte libre,
- JavaScript libre,
- SQL arbitrario,
- cambios directos de schema fisico,
- codigo ejecutable que corra sin control de plataforma.

### 7.2 Flujo de generacion
1. Usuario describe lo que quiere.
2. IA genera DSL estructurado.
3. El sistema valida schema y referencias.
4. Se guarda como draft en `user_workspace.db` asociado al tenant sobre el que el usuario esta trabajando.
5. Se renderiza preview local.
6. Si el usuario publica, pasa al backend.
7. El backend valida permisos, consistencia y limites.
8. Se publica en la DB del tenant y se sincroniza hacia `tenant_shared.db`.

### 7.3 Publicacion de contratos API y MCP
Los contratos API y las herramientas MCP siguen el mismo flujo de draft y publicacion.

Reglas:
- se validan con JSON Schema,
- se valida que el binding apunte a capacidades soportadas,
- se valida que el usuario tenga permiso de publicacion,
- se valida que no exponga datos fuera del tenant activo,
- se valida que las capacidades requeridas sean compatibles con Cedar.

Una vez publicados:
- el runtime HTTP puede exponer endpoints derivados,
- el runtime MCP puede anunciar herramientas derivadas,
- ambos usan el mismo modelo de permisos y bindings.

### 7.4 Publicacion segura
Toda publicacion debe validar:
- que el documento sea DSL valido,
- que no use primitives no soportadas,
- que las referencias a entidades existan,
- que las acciones requeridas sean legales,
- que el usuario tenga permiso de publicar.

---

## 8) Permisos y visibilidad

### 8.1 Principio
La UI puede ocultar cosas segun capacidades, pero la autoridad sigue estando en backend.

### 8.2 Uso de capacidades
Cada pagina, vista o accion puede declarar requisitos como:
- `can_read`
- `can_write`
- `can_manage_permissions`
- `can_publish_runtime`

### 8.3 Ejemplo

```json
{
  "kind": "button",
  "props": {
    "label": "Publicar"
  },
  "visible_if": {
    "capability": "can_publish_runtime"
  },
  "action": {
    "kind": "publish_document",
    "target": "page_definition:sales_dashboard"
  }
}
```

### 8.4 Permisos sobre contratos y herramientas
No todo usuario debe poder crear o publicar integraciones.

Capacidades sugeridas:
- `can_manage_runtime`
- `can_publish_runtime`
- `can_manage_api_contracts`
- `can_invoke_runtime_api`
- `can_invoke_runtime_mcp`

Regla:
- crear un draft puede requerir una capacidad,
- publicar puede requerir una capacidad mas fuerte,
- invocar el endpoint o la tool tambien se revalida en runtime.

---

## 9) API y MCP del runtime

### 9.1 Principio
La UI no es la unica forma de usar el sistema.

Los artefactos publicados del runtime deben poder consumirse por:
- interfaz Svelte,
- clientes HTTP,
- clientes MCP,
- agentes de IA.

### 9.2 Capa canonica
La capa canonica para describir interfaces debe ser declarativa.

Recomendacion:
- JSON Schema para `input_schema` y `output_schema`,
- bindings declarativos a operaciones soportadas,
- metadata de capacidades y permisos,
- versionado de contratos.

No recomiendo usar OpenAPI completo como artefacto canonico editable por usuario.
Puede generarse despues como export o vista derivada.

### 9.3 Modelo ideal
El modelo ideal seria:

1. El usuario o la IA crea `api_contract_definition`.
2. El sistema lo valida.
3. El backend publica un endpoint runtime derivado.
4. Si aplica, el backend publica tambien una `mcp_tool_definition` derivada.
5. HTTP y MCP comparten schemas, permisos y binding.

### 9.4 Ejemplo de exposicion HTTP

Endpoint derivado:

```text
POST /api/runtime/sales/invoices/query
```

El runtime:
- autentica,
- resuelve tenant activo,
- valida input contra `input_schema`,
- revisa capacidades,
- ejecuta el binding permitido,
- valida output contra `output_schema`,
- responde JSON.

### 9.5 Ejemplo de exposicion MCP

Tool derivada:

```json
{
  "name": "sales_get_invoice",
  "inputSchema": {
    "type": "object",
    "properties": {
      "invoice_id": { "type": "string" }
    },
    "required": ["invoice_id"]
  }
}
```

El runtime MCP:
- autentica contexto de agente o usuario,
- resuelve tenant activo,
- valida input,
- ejecuta binding,
- valida output,
- devuelve resultado.

### 9.6 Que si permitir desde frontend
Desde frontend si permitiria crear:
- modulos,
- paginas,
- formularios,
- rutas,
- esquemas,
- acciones declarativas,
- contratos API declarativos,
- herramientas MCP declarativas.

Desde frontend no permitiria crear:
- handlers backend libres,
- SQL libre,
- JavaScript arbitrario,
- herramientas MCP con codigo ejecutable custom,
- webhooks sin sandbox o sin capacidades controladas.

---

## 10) Lo que si haria y lo que no

### 10.1 Si haria
- DSL declarativo versionado
- runtime controlado en Svelte
- catalogo cerrado de primitives
- drafts privados + publicacion explicita
- JSON para documentos con validacion fuerte
- proyecciones indexadas para consultas reales
- IA generando estructura y configuracion
- contratos API y tools MCP basados en JSON Schema
- endpoints y tools derivados de bindings permitidos

### 10.2 No haria
- ejecutar Svelte arbitrario del usuario
- guardar funciones JS libres en DB
- permitir SQL arbitrario generado por IA
- usar JSON sin schemas ni validacion
- mezclar drafts privados con artefactos publicados
- permitir handlers backend libres definidos por usuario
- convertir MCP en un runtime de plugins arbitrarios

---

## 11) Conclusion final

La recomendacion final es:

1. Usar Svelte como runtime interno, no como lenguaje libre del usuario.
2. Definir un DSL declarativo para paginas, formularios, entidades y acciones.
3. Extender ese DSL a modulos, contratos API y herramientas MCP.
4. Guardar esos artefactos como documentos JSON sincronizables offline.
5. Separar drafts privados en `user_workspace.db` y publicaciones en la DB del tenant replicada en `tenant_shared.db`.
6. Usar JSON Schema como base para contratos HTTP y MCP.
7. Permitir IA-first sobre documentos estructurados, no sobre codigo arbitrario.
8. Mantener proyecciones indexadas para no depender de JSON puro en todo.

En resumen:

no construiria una plataforma donde el usuario programa Svelte;
ni una plataforma donde el usuario programa handlers backend arbitrarios;
construiria una plataforma donde el usuario y la IA describen productos, interfaces y acciones, y la plataforma los renderiza y expone de forma segura con Svelte, HTTP y MCP.