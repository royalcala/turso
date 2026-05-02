# Turso Sync - Mapa Mental (Markmap)

## Objetivo
- Sincronizar base local con remoto
- Mantener cambios locales y remotos consistentes
- Permitir flujo incremental (no full sync cada vez)

## Capas principales

### 1) Motor de Sync (cliente)
- Ruta: sync/engine
- Responsabilidad:
  - Bootstrap inicial
  - Push de cambios locales
  - Pull de cambios remotos
  - Aplicacion segura de cambios remotos

### 2) Capa SDK/Bindings
- Ruta: sync/sdk-kit
- Responsabilidad:
  - Exponer API de alto nivel (create/open/connect/push/pull/checkpoint/stats)
  - Encapsular asincronia e IO

### 3) Servidor de protocolo Sync
- Ruta: cli/sync_server.rs
- Endpoints:
  - POST /v2/pipeline
  - POST /pull-updates

## Flujo completo

### Inicializacion
- create/open del sync engine
- Lectura de metadata local
- Si no hay estado local valido:
  - Bootstrap desde remoto
  - Persistencia de revision inicial

### Escrituras locales
- Aplicacion escribe en DB local
- CDC captura INSERT/UPDATE/DELETE/DDL
- Los cambios quedan en tabla de CDC

### Push (local -> remoto)
- Lee cambios desde CDC
- Convierte cambios en operaciones SQL replay
- Envia lotes por /v2/pipeline
- Actualiza punteros de sincronizacion por cliente

### Pull (remoto -> local)
- Solicita cambios con revision cliente
- Recibe paginas WAL/protobuf via /pull-updates
- Guarda frames/paginas en archivo temporal

### Apply remoto en local
- Checkpoint/estado WAL seguro
- Rollback temporal de cambios locales necesarios
- Aplicar cambios remotos
- Re-jugar cambios locales pendientes
- Actualizar metadata de revision

## CDC y su papel

### Activacion
- PRAGMA capture_data_changes_conn(...)
- En el tape se usa modo full para replay robusto

### Datos capturados
- Tipo de cambio
- Tabla
- Rowid
- before/after/updates (segun modo)
- En v2: identificador transaccional y commits explicitos

### Integracion con sync
- Fuente primaria para push logico
- Permite reconstruir SQL de replay por fila

## Estado y metadata

### Archivo de metadata
- Guarda:
  - client_unique_id
  - revision sincronizada
  - timestamps de ultimo push/pull
  - hints de ultimo change_id
  - configuracion guardada (remote_url, partial sync)

### Tabla turso_sync_last_change_id
- Mapea cliente -> pull_gen/change_id
- Ayuda a incrementalidad e idempotencia

## Protocolos y transporte

### Protocolo V1 (actual)
- Pull de paginas por protobuf
- Push por pipeline JSON

### Endpoints usados por cliente
- /pull-updates
- /v2/pipeline

### Legacy
- Existe compatibilidad legacy en engine
- Camino principal recomendado: V1

## Operaciones operativas

### checkpoint
- Reduce WAL local y consolida estado
- Debe coordinarse con push/pull

### stats
- cdc_operations pendientes
- tamano de WAL principal
- tamano de WAL de revert
- bytes enviados/recibidos

## Restricciones y consideraciones
- CDC no compatible con MVCC
- Push/Pull/Checkpoint no deben correr concurrentemente sin coordinacion
- El replay debe respetar fronteras transaccionales
- Mantener consistencia de schema version durante apply

## Archivos clave para navegar
- sync/engine/src/database_sync_engine.rs
- sync/engine/src/database_sync_operations.rs
- sync/engine/src/database_tape.rs
- sync/engine/src/server_proto.rs
- sync/sdk-kit/src/rsapi.rs
- sync/sdk-kit/turso_sync.h
- cli/sync_server.rs
- core/translate/pragma.rs
- core/translate/emitter/mod.rs
- core/vdbe/execute.rs

## Receta de uso (mental model)
- 1. create/open
- 2. connect y escribir local
- 3. push periodico
- 4. pull periodico
- 5. apply
- 6. checkpoint
- 7. stats para observar salud

## Debug rapido
- Ver si CDC esta activo y con modo correcto
- Revisar tabla de CDC y change_id
- Ver punteros en turso_sync_last_change_id
- Confirmar revision en metadata
- Revisar WAL local antes/despues de checkpoint
- Ver logs de /pull-updates y /v2/pipeline
