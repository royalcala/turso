# Permisos y Seguridad con 1 DB por Org + Múltiples Usuarios

## El Problema

Con una DB por organización, **todos los usuarios de la misma org comparten el mismo archivo de base de datos** vía sync. El archivo `org_acme.db` en el dispositivo de cada empleado contiene **todos los datos de la organización**.

```
┌─────────────────────────────────────────────────────────────────┐
│  org_acme.db (sincronizada a TODOS los usuarios de Acme Corp)     │
│                                                                   │
│  ┌──────────────────┐   ┌──────────────┐   ┌──────────────────┐ │
│  │ users            │   │ tasks        │   │ salaries         │ │
│  │ Ana, Beto, Carlos│   │ todas las    │   │ Ana: $50k        │ │
│  │                  │   │ tareas de    │   │ Beto: $60k       │ │
│  │                  │   │ todos        │   │ Carlos: $55k     │ │
│  └──────────────────┘   └──────────────┘   └──────────────────┘ │
│                                                                   │
│  ┌────────────┐        ┌────────────┐        ┌────────────┐     │
│  │ Carlos     │        │ Beto       │        │ Ana (CEO)  │     │
│  │ (empleado) │        │ (manager)  │        │ (admin)    │     │
│  │            │        │            │        │            │     │
│  │ Su app.db  │        │ Su app.db  │        │ Su app.db  │     │
│  │ tiene TODO │        │ tiene TODO │        │ tiene TODO │     │
│  └────────────┘        └────────────┘        └────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

**Carlos puede potencialmente ver los salarios de todos si inspecciona el archivo.** La pregunta es: ¿cómo limitamos lo que cada usuario puede ver?

---

## ¿Qué Nivel de Protección Necesitas?

| Tipo de app | Riesgo si los datos se filtran entre colegas | Nivel de protección necesario |
|-------------|----------------------------------------------|------------------------------|
| App interna tipo Trello/Jira | Bajo — los datos no son secretos entre empleados | App-level filtering |
| CRM con datos de clientes | Medio — acceso por vendedor/región | App-level + row filtering |
| HR con salarios | Alto — cumplimiento legal | Datos sensibles en API, no en sync |
| Salud/Finanzas/Legal | Crítico — regulaciones estrictas | No usar sync para esos datos |

---

## Opción 1: Filtrado a Nivel de Aplicación (Para riesgo bajo)

El archivo contiene todos los datos, pero tu app solo muestra lo que corresponde:

```typescript
// El empleado Carlos solo ve SUS tareas
function TasksForUser(userId: number) {
  return db
    .select()
    .from(tasks)
    .where(eq(tasks.assignedTo, userId))  // ← filtro en WHERE
    .orderBy(desc(tasks.createdAt));
}

// El manager Beto ve las tareas de su equipo
function TasksForTeam(teamId: number) {
  return db
    .select()
    .from(tasks)
    .innerJoin(users, eq(tasks.assignedTo, users.id))
    .where(eq(users.teamId, teamId));
}
```

**¿Contra qué protege?**
- ✅ Errores de UI — la app no muestra datos que no debe
- ✅ Usuarios normales — no ven lo que no les corresponde

**¿Contra qué NO protege?**
- ❌ Usuario malicioso que abre DevTools y consulta el archivo
- ❌ Usuario que extrae `app.db` y lo abre con otra herramienta
- ❌ Bug en el código que omite el WHERE accidentalmente

**Cuándo usarlo:** Apps de productividad interna (tareas, proyectos, notas) donde los datos no son secretos entre compañeros de trabajo.

---

## Opción 2: Tablas Separadas para Datos No-Sync (Para riesgo medio/alto)

Divide los datos en dos categorías:

```
┌──────────────────────────────────────────────────────────────────┐
│  DATOS QUE SE SINCRONIZAN (app.db)                                │
│  ✅ tasks — tareas, proyectos, comentarios                        │
│  ✅ users — nombre, email, avatar, equipo                         │
│  ✅ projects — proyectos, milestones                               │
│  ✅ notifications — notificaciones del usuario                     │
│                                                                   │
│  DATOS QUE NUNCA SE SINCRONIZAN (solo vía API al servidor)        │
│  ❌ salaries — salarios, bonos                                    │
│  ❌ performance_reviews — evaluaciones                             │
│  ❌ hr_documents — documentos de RH                                │
│  ❌ billing — facturación, pagos                                  │
│  ❌ audit_logs — logs de auditoría                                │
└──────────────────────────────────────────────────────────────────┘
```

```typescript
// ── Datos locales (sync) ──
const myTasks = await db
  .select()
  .from(tasks)
  .where(eq(tasks.assignedTo, myUserId))
  .all();

// ── Datos sensibles (API, nunca en app.db) ──
const mySalary = await fetch("/api/salaries/me", {
  headers: { Authorization: `Bearer ${sessionToken}` },
}).then((r) => r.json());

// ── Manager ve salarios de su equipo (API, verifica permisos en servidor) ──
async function getTeamSalaries(teamId: number) {
  return fetch(`/api/salaries/team/${teamId}`, {
    headers: { Authorization: `Bearer ${sessionToken}` },
  }).then((r) => r.json());
}
```

**Arquitectura:**

```
┌─────────────────────┐     ┌──────────────────────┐
│  Cliente             │     │  Servidor             │
│                      │     │                       │
│  ┌──────────────┐   │     │  ┌─────────────────┐  │
│  │ app.db (sync) │◄──┼─────┼─┤ tasks, projects  │  │
│  │ tasks,        │   │     │  │ users, etc.     │  │
│  │ projects,     │   │     │  └─────────────────┘  │
│  │ users         │   │     │                       │
│  └──────────────┘   │     │  ┌─────────────────┐  │
│                      │     │  │ PostgreSQL/     │  │
│  ┌──────────────┐   │     │  │ Turso server-db  │  │
│  │ API calls     │───┼────►│  │ salaries,       │  │
│  │ salaries,     │   │     │  │ HR, billing     │  │
│  │ HR data       │   │     │  └─────────────────┘  │
│  └──────────────┘   │     │                       │
└─────────────────────┘     └──────────────────────┘
```

**Ventaja:** Los datos sensibles **nunca existen en el archivo del cliente**. Es imposible extraerlos.

**Desventaja:** Esos datos no están disponibles offline. Necesitas el servidor para consultarlos.

---

## Opción 3: Datos de Otros Usuarios Filtrados en el Servidor (Para riesgo medio)

En vez de sync completo, **el servidor filtra qué datos envía a cada usuario**. Esto requiere construir tu propio servidor o extender el existente:

```
┌─────────────────────────────────────────────────────────────────┐
│  Servidor Sync (modificado)                                       │
│                                                                   │
│  POST /pull-updates { client_revision: "...", userId: "carlos" } │
│                                                                   │
│  El servidor:                                                     │
│  1. Prepara el WAL con todos los cambios                          │
│  2. FILTRA las páginas: solo incluye páginas que contienen        │
│     datos que Carlos tiene permiso de ver                         │
│  3. Envía solo esas páginas a Carlos                              │
│                                                                   │
│  ⚠️ ESTO NO EXISTE HOY en Turso Cloud ni en sqld.                 │
│  Tendrías que construirlo.                                       │
└─────────────────────────────────────────────────────────────────┘
```

**Problemas:**
- Filtrar a nivel de página es complejo (una página puede contener datos de varios usuarios)
- Rompe el modelo de sync estándar
- Muy difícil de implementar correctamente

**No recomendado.** Mejor usar Opción 2 (separar tablas) que es más simple y segura.

---

## Opción 4: Encriptación a Nivel de Aplicación (Para riesgo alto, pero necesitas sync offline)

Encriptas campos sensibles antes de guardarlos en la DB. Solo los usuarios autorizados tienen la llave para desencriptar:

```typescript
import { encryptForUser, decryptForUser } from "./crypto";

// ── Guardar dato sensible (se ejecuta en el cliente que escribe) ──
async function setSalary(userId: number, amount: number) {
  // Encriptar con la llave pública de la org (todos los managers pueden leer)
  const encrypted = await encryptForRole("manager", JSON.stringify({ userId, amount }));
  
  await db.insert(salaries).values({
    userId,
    encryptedData: encrypted,  // en disco: blob cifrado
  });
}

// ── Leer dato sensible (se ejecuta en el cliente que lee) ──
async function getSalary(userId: number): Promise<number | null> {
  // Solo funciona si el usuario tiene la llave de manager
  const rows = await db
    .select({ encryptedData: salaries.encryptedData })
    .from(salaries)
    .where(eq(salaries.userId, userId));
  
  if (rows.length === 0) return null;
  
  try {
    const decrypted = await decryptForRole("manager", rows[0].encryptedData);
    return JSON.parse(decrypted).amount;
  } catch {
    return null;  // no tienes la llave → no puedes leer
  }
}
```

```
┌──────────────────────────────────────────────────────────────────┐
│  app.db en el dispositivo de Carlos (empleado sin permiso)         │
│                                                                   │
│  salaries:                                                        │
│  ┌──────────┬───────────────────────────────────────────────┐    │
│  │ user_id  │ encrypted_data                                 │    │
│  ├──────────┼───────────────────────────────────────────────┤    │
│  │ 1 (Ana)  │ AES("Ana:$50k", key=manager_pub_key)          │    │
│  │ 2 (Beto) │ AES("Beto:$60k", key=manager_pub_key)         │    │
│  │ 3 (Carlos)│ AES("Carlos:$55k", key=manager_pub_key)       │    │
│  └──────────┴───────────────────────────────────────────────┘    │
│                                                                   │
│  Carlos tiene el archivo. Pero NO tiene la llave de manager.      │
│  → Los datos son un blob cifrado ilegible para él.               │
│                                                                   │
│  Ana (CEO) tiene la llave de manager.                             │
│  → Su app desencripta y ve los salarios.                         │
└──────────────────────────────────────────────────────────────────┘
```

**Cuándo usarlo:** Cuando necesitas que TODOS los datos estén offline pero solo algunos usuarios puedan leer ciertas columnas. Ejemplo: app de salud donde el doctor puede ver historiales completos pero el paciente solo ve los suyos.

**Complejidad:** Alta. Necesitas manejar llaves, rotación, y distribución segura de llaves.

---

## Recomendación Práctica

```
┌─────────────────────────────────────────────────────────────────┐
│                    ESTRATEGIA POR CAPA                           │
│                                                                  │
│  CAPA 1: Datos compartidos de la org (sync en app.db)            │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ tasks, projects, comments, notifications, chat          │     │
│  │ → Se filtran con WHERE en el código                     │     │
│  │ → Ej: WHERE assigned_to = $currentUserId                │     │
│  │ → Protege contra errores de UI, no contra forense       │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                  │
│  CAPA 2: Datos con permisos (sync en app.db, verificados)        │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ user_profiles, team_settings, project_config             │     │
│  │ → Middleware de permisos en cada query                   │     │
│  │ → Ej: function getTeamData(teamId) {                    │     │
│  │         if (!userBelongsToTeam(user, teamId)) throw;     │     │
│  │         return db.select().from(teams)...                │     │
│  │       }                                                  │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                  │
│  CAPA 3: Datos sensibles (NUNCA en app.db, solo API)             │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ salaries, HR, billing, compliance, PII, secrets         │     │
│  │ → Servidor tradicional con auth                          │     │
│  │ → El cliente hace fetch("/api/...") con token            │     │
│  │ → Imposible extraer porque nunca llegan al dispositivo   │     │
│  └────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Código: Capa de Permisos para Queries Locales

```typescript
// shared/permissions.ts
import { db } from "./db";
import { and, eq, inArray, sql } from "drizzle-orm";
import type { User } from "./auth";

// El usuario actual (obtenido de sesión/token)
let currentUser: User | null = null;
export function setCurrentUser(user: User) { currentUser = user; }

// ── Helper: query con filtro de permisos ──
export async function queryWithPermission<T>(
  queryFn: () => Promise<T>,
  requiredRole?: string
): Promise<T> {
  if (!currentUser) throw new Error("Not authenticated");
  if (requiredRole && currentUser.role !== requiredRole) {
    throw new Error("Insufficient permissions");
  }
  return queryFn();
}

// ── Tasks: solo las asignadas a tu equipo o a ti ──
export async function getVisibleTasks() {
  return queryWithPermission(async () => {
    if (currentUser!.role === "admin") {
      return db.select().from(tasks).all();
    }
    if (currentUser!.role === "manager") {
      return db
        .select()
        .from(tasks)
        .innerJoin(users, eq(tasks.assignedTo, users.id))
        .where(eq(users.teamId, currentUser!.teamId))
        .all();
    }
    // employee: solo sus tareas
    return db
      .select()
      .from(tasks)
      .where(eq(tasks.assignedTo, currentUser!.id))
      .all();
  });
}

// ── Salaries: SIEMPRE via API, nunca local ──
export async function getMySalary(): Promise<number> {
  const res = await fetch("/api/salaries/me", {
    headers: { Authorization: `Bearer ${currentUser!.token}` },
  });
  if (!res.ok) throw new Error("Forbidden");
  return res.json();
}
```

---

## Resumen

| Necesitas... | Usa... |
|-------------|--------|
| Evitar que usuarios vean datos de otros en la UI | WHERE en queries + capa de permisos |
| Evitar que un usuario malicioso extraiga datos del archivo | Datos sensibles en API (no en sync) |
| Offline + datos sensibles con permisos | Encriptación a nivel de aplicación (capa 4) |
| Simplicidad máxima, riesgo bajo | Una DB por org, WHERE en queries, confiar en los empleados |
| Cumplimiento regulatorio (GDPR, HIPAA, SOC2) | Separar datos regulados en API; auditarlos ahí |

**La DB por org con múltiples usuarios es viable siempre y cuando:** identifiques qué datos realmente necesitan protección fuerte y los mantengas fuera del sync, usando una API tradicional para esos datos sensibles.
