# Alternativas a Limbo/Turso DB y TinyBase

## Objetivo
Comparar alternativas reales para:
- motor de datos/sync (hoy: Limbo/Turso DB),
- store reactivo cliente (hoy: TinyBase),

considerando requisitos del proyecto:
- soberania de infraestructura,
- costos operativos bajos,
- local-first/offline-first,
- multi-tenant,
- UI declarativa AI-first,
- posibilidad de exponer API/MCP.

---

## 1) Baseline actual

### 1.1 Stack base
- DB y sync: Limbo/Turso DB
- Store UI: TinyBase
- Backend: Rust + Cedar
- Cliente: runtime declarativo

### 1.2 Fortalezas del baseline
- Control soberano fuerte.
- Costos infra bajos si el despliegue se mantiene simple.
- Excelente encaje con modelo de DB por tenant y workspace privado por usuario.
- Integracion limpia con write pipe transaccional y CAS.

### 1.3 Debilidades del baseline
- Menos ecosistema plug-and-play de SDUI/GUI generativa que el stack React/TanStack.
- TinyBase puede quedarse corto si quieres un engine de live query avanzado cross-collection.
- Parte de tooling de integraciones hay que construirlo a medida.

---

## 2) Alternativa A: Electric Sync + Postgres + TanStack DB

### 2.1 Que aporta
- Sync de datos desde Postgres con primitives de Electric.
- Buen encaje con TanStack DB para live queries y estado reactivo cliente.
- Ecosistema moderno para experiencias reactivas y apps colaborativas.

### 2.2 Ventajas
- Muy fuerte para reactividad end-to-end y fanout de lectura.
- Integracion natural con TanStack DB.
- Menor trabajo custom en ciertas piezas de sync de alto nivel.

### 2.3 Desventajas
- Incrementa dependencia de Postgres como pilar.
- Mayor superficie operativa que un stack embebido simple.
- Menor alineacion con objetivo de minima infraestructura (comparado con Limbo embebido puro).

### 2.4 Cuando conviene
- Cuando priorizas ecosistema reactivo y velocidad de features de realtime.
- Cuando aceptas una capa central de datos tipo Postgres en produccion.

---

## 3) Alternativa B: Durable Streams + runtime declarativo

### 3.1 Que aporta
- Primitive de streams persistentes, direccionables y realtime.
- Muy orientado a workloads de agentes y colaboracion basada en streams/eventos.

### 3.2 Ventajas
- Excelente para orquestacion y estados de agentes long-lived.
- Buen encaje para event sourcing y pipelines de sesiones.

### 3.3 Desventajas
- No reemplaza automaticamente una DB relacional de negocio.
- Para CRUD estructurado con queries ricas, normalmente necesitas componerlo con otra capa.
- Puede mover la complejidad a modelar todo en streams cuando tu dominio es mas tabular/relacional.

### 3.4 Cuando conviene
- Si el nucleo del producto son streams de eventos/agentes y colaboracion en tiempo real.
- Como complemento de DB de negocio, no necesariamente como reemplazo total.

---

## 4) Alternativa C: TanStack DB en lugar de TinyBase

### 4.1 Que aporta
- Store client-first reactivo con collections, live queries y optimistic mutations.
- Fuerte para UIs dinamicas con joins/filtros/agregaciones reactivas.

### 4.2 Ventajas
- Mejor ergonomia para consultas reactivas complejas que un store basico.
- Encaje natural con React y routing/data loading de ecosistema TanStack.
- Puede reducir codigo custom de cache/sincronizacion de UI.

### 4.3 Desventajas
- Tecnologia en etapa temprana (BETA al momento de esta evaluacion).
- Riesgo de cambios de API y churn de integracion.
- Requiere validar muy bien el backend connector elegido.

### 4.4 Cuando conviene
- Si migras frontend a React y necesitas un engine reactivo mas potente que TinyBase.
- Si aceptas riesgo controlado de adoptar una pieza joven del ecosistema.

---

## 5) Matriz comparativa (practica)

Escala: 1 (bajo) a 5 (alto)

| Opcion | Soberania | Complejidad operativa | Offline-first | Realtime reactivo | Riesgo de adopcion |
|---|---:|---:|---:|---:|---:|
| Limbo/Turso + TinyBase | 5 | 2 | 5 | 3 | 2 |
| Limbo/Turso + TanStack DB | 5 | 3 | 5 | 4 | 3 |
| Electric + TanStack DB | 3 | 4 | 4 | 5 | 3 |
| Durable Streams + DB relacional | 4 | 4 | 3 | 5 | 4 |

Notas:
- "Riesgo de adopcion" refleja madurez, cambios potenciales y costo de migracion.
- "Complejidad operativa" asume despliegue soberano sin servicios externos administrados.

---

## 6) Recomendacion por escenarios

### 6.1 Escenario 1: prioridad absoluta a soberania + costo minimo
Recomendacion:
- Mantener Limbo/Turso DB como nucleo.
- Mantener TinyBase en corto plazo.
- Evolucionar gradualmente el runtime declarativo y API/MCP por contratos.

### 6.2 Escenario 2: prioridad a DX de UI reactiva avanzada
Recomendacion:
- Mantener Limbo/Turso DB.
- Evaluar reemplazo de TinyBase por TanStack DB en un modulo piloto.
- Medir rendimiento y complejidad antes de migracion total.

### 6.3 Escenario 3: prioridad a ecosistema sync postgresql y fanout
Recomendacion:
- Evaluar Electric + TanStack DB.
- Aceptar tradeoff de mayor complejidad operativa.
- No adoptar si contradice el objetivo de infra minima.

### 6.4 Escenario 4: prioridad a agentes/event-streaming como centro del producto
Recomendacion:
- Incorporar Durable Streams como capa especializada.
- Mantener DB relacional para dominio estructurado.
- Evitar usar streams como sustituto unico de todo el modelo de negocio.

---

## 7) Propuesta concreta para este proyecto

Dado lo ya definido en RAO:

1. Mantener Limbo/Turso DB como data plane por tenant.
2. Mantener write pipe sincronico con CAS e idempotencia.
3. Mantener control plane minimo de identidad/memberships.
4. Mantener runtime declarativo para UI/API/MCP.
5. Si migras frontend a React, evaluar TanStack DB como reemplazo progresivo de TinyBase.
6. Tratar Electric y Durable Streams como opciones de expansion, no como cambio inmediato obligatorio.

---

## 8) Decision recomendada hoy

### Corto plazo
- No reemplazar DB principal.
- No hacer replatform completo de sync.
- Consolidar arquitectura actual y cerrar MVP del runtime declarativo.

### Mediano plazo
- Probar TanStack DB en un modulo aislado de UI para validar si supera TinyBase en tu caso real.
- Revaluar Electric o Durable Streams solo cuando haya evidencia de necesidad que el stack actual no cubra.

En resumen:
- Limbo/Turso sigue siendo base recomendada para soberania y costo.
- TinyBase puede mantenerse o evolucionar a TanStack DB de forma incremental.
- Electric y Durable Streams son alternativas valiosas, pero no obligatorias para el estado actual del proyecto.