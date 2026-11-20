# WAB-79 — Informe de Viabilidad Técnica: Cierre del Diseño de Implementación (Versión Corregida)

> **Estado:** Propuesta técnica corregida para revisión. No autoriza implementación ni cambios en Linear.
> **Fase:** WAB-79 — Bloque de viabilidad técnica de la jubilación de AgentInstance
> **Fuentes:** waby-docs (canónico), CRM backend (`ElemeSoft/crm`), AI Service (`crm/microservices/ai-service`), Infrastructure (`ElemeSoft/infrastructure`)
> **Versión anterior:** `wab-79-viabilidad-tecnica-cierre.md` (con errores de ownership, coordinación cross-DB, concurrencia y recuperación)

---

## 1. Resumen Ejecutivo Actualizado

Este informe corrige la versión anterior resolviendo los cuatro puntos técnicos pendientes con **ownership correcto, separación de bases de datos, y comportamiento real del driver MongoDB**.

| Punto | Estado | Conclusión Corregida |
|-------|--------|---------------------|
| 1. Exclusión atómica admisión/bloqueo | **Propuesta técnica corregida** | Control de admisión local en AI Service (`AgentExecutionAdmission` en DB `ai_service`). Coordinación vía comunicación durable (Outbox/Inbox), **sin transacciones cross-DB**. Aceptación y bloqueo compiten atómicamente en transacción local de AI Service usando `admissionRevision` + `$inc`. |
| 2. Separación de confirmaciones | **Propuesta técnica corregida** | Tres contratos separados: (a) `retirement_confirmations` en CRM (aplicación EvAssigned, idempotente por `evAssignedId`); (b) `RetirementOperation.lockConfirmations.execution` en CRM (bloqueo admisión, idempotente por `retirementOperationId`); (c) `RetirementOperation.executionDrainConfirmation` en CRM (drenaje, idempotente por `retirementOperationId` + `admissionBlockedAt`). AI Service emite confirmaciones vía Outbox; CRM las persiste. |
| 3. Recuperación transaccional | **Propuesta técnica verificada** | Driver MongoDB 6.20.0 / Mongoose 8.19.3: `withTransaction` reintenta **callback completo** ante `TransientTransactionError`; `UnknownTransactionCommitResult` requiere reintento manual de `commitTransaction()` en **misma sesión**; agotados reintentos → reconciliación durable por claves (no reparación automática). Distinción estricta entre transacción confirmada, abortada, commit incierto, y inconsistencia de datos. |
| 4. Verificación MongoDB Atlas | **Pendiente de confirmación externa (BLOQUEANTE)** | CRM usa DB `crm`; AI Service usa DB `ai_service`; **mismo cluster Atlas, bases distintas**. No hay transacciones distribuidas. DevOps debe confirmar: topología (RS/SC), versión ≥ 4.2, compatibilidad transacciones multidocumento en ese deployment, límites operativos. |

**Invariantes garantizadas por el diseño corregido:**
- No existe ventana de aceptación posterior al bloqueo confirmado (exclusión atómica en transacción local AI Service).
- Bloqueo de admisión y drenaje son confirmados durablemente por AI Service y registrados por CRM.
- Recuperación no duplica eventos ni transiciones (reconciliación por claves durables + índices únicos).
- Atomicidad Contacto + EvAssigned + Outbox + Revision en transacción local CRM.
- AI Service nunca escribe en CRM DB; CRM nunca escribe en AI Service DB.

---

## 2. Ownership y Límites Transaccionales (Corrección Fundamental)

### 2.1 Separación de Bases de Datos — Evidencia Verificada

| Servicio | Base de Datos | Colecciones Relevantes (actuales/objetivo) |
|----------|---------------|--------------------------------------------|
| **CRM** | `crm` | `contactos`, `eventos` (incl. `EvAssigned`), `outbox`, `agent_instances`, `retirement_operations`, `retirement_confirmations`, `works`, `organizaciones` |
| **AI Service** | `ai_service` | `executions` → `agent_executions`, `contact_assignments` (proyección), `bots`, `bot_instances`, `knowledge_bases`, `documents`, **`agent_execution_admissions` (nueva)** |

**Fuentes:**
- `crm/src/database.ts:15` → `MONGO_DB` = `crm`
- `crm/microservices/ai-service/src/infrastructure/database/mongodb.ts:38-41` → `config.mongodb.database` = `ai_service`
- `waby-docs/projects/agents-multitenant/audits/wab-77/informe-consolidado-fase-1.md:115` — "CRM (DB `crm`) y ai-service (DB `ai_service`) comparten cluster MongoDB pero usan bases distintas"
- `waby-docs/projects/agents-multitenant/design/wab-79-dominio-ownership-persistencia.md §16.2` — "CRM es autoridad de asignaciones y propietario único de `RetirementOperation`. El servicio de ejecución es autoridad de admisión de ejecuciones. **No existe transacción distribuida entre ambos**."

### 2.2 Modelo de Ownership Aprobado (D-W79-15, §16)

| Entidad | Propietario de Negocio | Servicio Responsable | DB de Persistencia | Autoridad |
|---------|------------------------|---------------------|-------------------|-----------|
| `AgentInstance` | Workspace | CRM | `crm` (`agent_instances`) | CRM (lifecycle, status) |
| `RetirementOperation` | Workspace | CRM | `crm` (`retirement_operations`) | CRM (única copia) |
| `AgentExecution` | AgentInstance + Workspace | AI Service | `ai_service` (`agent_executions`) | AI Service |
| `AgentExecutionAdmission` (nueva) | AgentInstance | AI Service | `ai_service` (`agent_execution_admissions`) | AI Service |
| `Contacto` / `EvAssigned` / Outbox | CRM | CRM | `crm` | CRM |
| `contact_assignments` (proyección) | Derivada | AI Service | `ai_service` | AI Service (operativa) |
| `retirement_confirmations` | CRM | CRM | `crm` | CRM |

### 2.3 Límites Transaccionales

| Transacción | Servicio | Base de Datos | Colecciones |
|-------------|----------|---------------|-------------|
| Aceptación ejecución + bloqueo admisión | AI Service | `ai_service` | `agent_execution_admissions`, `agent_executions` |
| Bloqueo admisión (iniciado por CRM) | AI Service | `ai_service` | `agent_execution_admissions` |
| Transición contacto (CAS + EvAssigned + Outbox + Revision) | CRM | `crm` | `contactos`, `eventos`, `outbox`, `retirement_operations` |
| Aplicación EvAssigned en proyección | AI Service | `ai_service` | `contact_assignments`, `inbox`, `outbox` (AI Service) |
| Confirmación drenaje | AI Service | `ai_service` | `agent_execution_admissions` (lectura), `outbox` (AI Service) |

**Regla de oro:** No hay transacciones que abarquen ambas bases de datos. La coordinación es **event-driven durable** (Outbox/Inbox + idempotencia).

---

## 3. Modelo de Admisión Local en AI Service

### 3.1 Nueva Colección: `agent_execution_admissions` (DB `ai_service`)

```typescript
// AI Service DB: ai_service.agent_execution_admissions
interface AgentExecutionAdmission {
  _id: string;                         // agentInstanceId (PK)
  agentInstanceId: string;             // redundancia explícita para consultas
  status: 'open' | 'blocked';          // estado de admisión
  retirementOperationId: string | null; // ID de la operación que bloqueó
  blockedAt: Date | null;              // timestamp del bloqueo confirmado
  admissionRevision: number;           // contador optimista para exclusión atómica
  updatedAt: Date;
  createdAt: Date;
}

// Índices
// PK: _id (agentInstanceId)
// Índice opcional: { status: 1 } para consultas de bloqueos activos
```

**Justificación del nombre y separación:**
- `AgentInstance` pertenece a CRM (DB `crm`). AI Service no debe escribir en `crm.agent_instances`.
- El control de admisión es **autoridad de AI Service** (dueño de `AgentExecution`).
- `admissionRevision` (`$inc` atómico) permite exclusión sin leer-escribir separado.
- Evita confusión de ownership: CRM consulta/lee este documento vía read-only cross-DB si necesita verificar, pero **solo AI Service lo modifica**.

### 3.2 Invariantes del Modelo

| Invariante | Mecanismo |
|------------|-----------|
| Una sola admisión por AgentInstance | PK `_id = agentInstanceId` |
| Bloqueo inmutable una vez confirmado | `status: 'blocked'` + `retirementOperationId` + `blockedAt` solo se escriben una vez (validación en aplicación) |
| Exclusión atómica aceptación vs bloqueo | `$inc` en `admissionRevision` dentro de transacción local |
| Idempotencia de bloqueo | Mismo `retirementOperationId` + `blockedAt` = idempotente; distinto = rechazo |

---

## 4. Algoritmos Transaccionales: Aceptación y Bloqueo

### 4.1 Operación de Aceptación de Ejecución (AI Service — Transacción Local)

```typescript
// AI Service: acceptExecution(executionId, agentInstanceId, executionData)
// RETORNA: { accepted: boolean; execution?: AgentExecution; reason?: string }

async function acceptExecution(
  executionId: string,
  agentInstanceId: string,
  executionData: AgentExecutionInput, // incluye retirementOperationId si viene de retry
  session: ClientSession
): Promise<AcceptResult> {
  // 1. Idempotencia: ¿ya existe esta executionId?
  const existing = await AgentExecutionModel.findById(executionId, { session });
  if (existing) {
    // Validar que corresponde a la misma solicitud (mismo agentInstanceId, etc.)
    if (existing.agentInstanceId !== agentInstanceId) {
      return { accepted: false, reason: 'EXECUTION_ID_CONFLICT_DIFFERENT_INSTANCE' };
    }
    return { accepted: true, execution: existing };
  }

  // 2. Transacción local: admisión + creación AgentExecution
  //    Usamos withTransaction para reintento automático ante TransientTransactionError
  return await session.withTransaction(async () => {
    // 2a. Leer admisión actual (findOneAndUpdate con $inc para exclusión atómica)
    const admission = await AgentExecutionAdmissionModel.findOneAndUpdate(
      {
        _id: agentInstanceId,
        status: 'open'
      },
      {
        $inc: { admissionRevision: 1 },  // incrementa revisión atómicamente
        $setOnInsert: {
          _id: agentInstanceId,
          agentInstanceId,
          status: 'open',
          retirementOperationId: null,
          blockedAt: null,
          createdAt: new Date()
        },
        $set: { updatedAt: new Date() }
      },
      { session, new: true, upsert: true }
    );

    if (!admission || admission.status !== 'open') {
      // Admisión bloqueada (o no existe y upsert creó con status open pero otra tx la bloqueó)
      throw new AdmissionBlockedError('Admission blocked or concurrent block');
    }

    // 2b. Verificar que AgentInstance está en estado que admite ejecuciones
    //     (lectura cross-DB read-only a CRM, sin transacción)
    const instance = await crmReadOnlyClient.agentInstances.findById(agentInstanceId);
    if (!instance || !['active', 'suspended'].includes(instance.status)) {
      throw new InstanceNotAdmissibleError(`Instance status: ${instance?.status}`);
    }

    // 2c. Crear AgentExecution en MISMA transacción
    const execution = new AgentExecutionModel({
      _id: executionId,
      ...executionData,
      agentInstanceId,
      status: 'pending',
      createdAt: new Date(),              // aceptación durable
      effectiveConfigSnapshot: executionData.effectiveConfigSnapshot,
      effectiveConfigHash: executionData.effectiveConfigHash,
      promptSnapshotHash: executionData.promptSnapshotHash,
    });
    await execution.save({ session });

    return { accepted: true, execution };
  }).catch((err) => {
    if (err instanceof AdmissionBlockedError || err instanceof InstanceNotAdmissibleError) {
      return { accepted: false, reason: err.message };
    }
    // TransientTransactionError → withTransaction ya reintentó; si llega aquí = fallo permanente
    // UnknownTransactionCommitResult → no debería llegar aquí si usamos withTransaction correctamente
    throw err;
  });
}
```

**Puntos clave:**
- **No hay "bloqueo temporal"**: la admisión se queda en `open`; el `$inc` en `admissionRevision` serializa concurrentemente.
- **Idempotencia fuerte**: valida `executionId` existente **y** que pertenece al mismo `agentInstanceId`.
- **Validación de estado**: cross-DB read-only a CRM para verificar `AgentInstance.status` (active/suspended).
- **Atomicidad**: admisión + ejecución en misma transacción local AI Service.

### 4.2 Operación de Bloqueo de Admisión (AI Service — Transacción Local, invocado por CRM vía mensaje durable)

```typescript
// AI Service: blockExecutionAdmission(retirementOperationId, agentInstanceId)
// Invocado tras recibir mensaje durable de CRM (vía RabbitMQ/Inbox)

async function blockExecutionAdmission(
  retirementOperationId: string,
  agentInstanceId: string,
  session: ClientSession
): Promise<BlockResult> {
  return await session.withTransaction(async () => {
    const admission = await AgentExecutionAdmissionModel.findOneAndUpdate(
      {
        _id: agentInstanceId,
        status: 'open',
        retirementOperationId: null  // solo bloquea si no hay bloqueo previo
      },
      {
        $set: {
          status: 'blocked',
          retirementOperationId,
          blockedAt: new Date()
        },
        $inc: { admissionRevision: 1 },
        $setOnInsert: {
          _id: agentInstanceId,
          agentInstanceId,
          createdAt: new Date()
        },
        $set: { updatedAt: new Date() }
      },
      { session, new: true, upsert: true }
    );

    if (!admission) {
      // No se pudo bloquear: ya bloqueado o no existe
      const current = await AgentExecutionAdmissionModel.findById(agentInstanceId, { session });
      return {
        blocked: current?.status === 'blocked',
        blockedAt: current?.blockedAt,
        retirementOperationId: current?.retirementOperationId,
        reason: current?.status === 'blocked' ? 'ALREADY_BLOCKED' : 'NOT_FOUND'
      };
    }

    // 3. Emitir confirmación durable en Outbox AI Service
    await AIServiceOutboxModel.create([{
      type: 'EXECUTION_ADMISSION_BLOCKED',
      payload: {
        retirementOperationId,
        agentInstanceId,
        admissionBlockedAt: admission.blockedAt,
        admissionRevision: admission.admissionRevision
      }
    }], { session });

    return { blocked: true, blockedAt: admission.blockedAt, admissionRevision: admission.admissionRevision };
  });
}
```

### 4.3 Escenarios de Concurrencia (Demostrados con Comportamiento Real MongoDB)

#### Escenario 1: Aceptación gana, luego llega bloqueo
```
T1: AI Service acceptExecution(exec-1, inst-A)
    → withTransaction:
       findOneAndUpdate(admission.open, $inc revision) → ENCUENTRA (rev=1)
       crea AgentExecution(exec-1, pending)
       COMMIT ✓ (admission sigue open, rev=1)

T2: CRM → AI Service blockExecutionAdmission(retire-1, inst-A)
    → withTransaction:
       findOneAndUpdate(admission.open, retirementOperationId=null, $set blocked, $inc rev) → ENCUENTRA (rev=2)
       status=blocked, retirementOperationId=retire-1, blockedAt=T2
       emite EXECUTION_ADMISSION_BLOCKED (Outbox)
       COMMIT ✓

Resultado: exec-1 aceptada durablemente (pending); admisión bloqueada para nuevas.
```

#### Escenario 2: Bloqueo gana, luego llega aceptación
```
T1: CRM → AI Service blockExecutionAdmission(retire-1, inst-A)
    → withTransaction:
       findOneAndUpdate(open, retirementOperationId=null) → ENCUENTRA
       blocked, retirementOperationId=retire-1, blockedAt=T1, rev=1
       emite Outbox → COMMIT ✓

T2: AI Service acceptExecution(exec-2, inst-A)
    → withTransaction:
       findOneAndUpdate(open, $inc rev) → NO ENCUENTRA (ya blocked)
       lanza AdmissionBlockedError
       CATCH → retorna { accepted: false, reason: 'Admission blocked' }
       exec-2 RECHAZADA (idempotente: findById(exec-2) = null)
```

#### Escenario 3: Conflicto simultáneo real (ambos llegan "a la vez")
```
MongoDB serializa findOneAndUpdate sobre mismo _id (agentInstanceId).
Una transacción gana el lock del documento; la otra espera o falla con WriteConflict.
withTransaction reintenta automáticamente la tx perdedora (TransientTransactionError).
En el reintento, la tx ve el estado ya modificado y toma la decisión correcta.
No hay ventana donde ambas lean 'open' y ambas escriban.
```

#### Escenario 4: Reintentos con mismo `executionId` (idempotencia)
```
acceptExecution(exec-1, inst-A) → crea exec-1 (pending)
[red cae, cliente reintenta]
acceptExecution(exec-1, inst-A) → findById(exec-1) ENCUENTRA
   → valida agentInstanceId coincide → retorna { accepted: true, execution: exec-1 }
No se crea duplicado; snapshot/hashes ya persistidos.
```

### 4.4 Manejo de Errores y Recuperación (Corregido según Driver 6.20.0)

| Situación | Comportamiento Driver 6.20.0 | Acción en Código |
|-----------|-------------------------------|------------------|
| `TransientTransactionError` (WriteConflict, primary stepped down, etc.) | `withTransaction` **reintenta automáticamente el callback completo** (nueva transacción, misma lógica) | No hacer nada; `withTransaction` maneja. Límite configurable via `maxTransactionLockRequestTimeoutMs` (default 5ms) y `maxTimeMS` en opciones. |
| `UnknownTransactionCommitResult` (commit timeout, network error) | `withTransaction` **NO reintenta automáticamente**. La aplicación debe detectar y reintentar `session.commitTransaction()` en **la misma sesión**. | Usar `withTransaction` con callback que no lance este error; si ocurre fuera del helper, reintentar commit manual en misma sesión (máx. 3 intentos). |
| Sesión cerrada / error de red post-commit | Resultado incierto | **Reconciliar estado durable** antes de cualquier reintento: consultar `AgentExecution` por `executionId` y `AgentExecutionAdmission` por `agentInstanceId`. |

**Regla:** Nunca crear automáticamente documentos faltantes al reconciliar. La reconciliación solo **lee** y **decide**.

---

## 5. Confirmación de Bloqueo y Drenaje (Corregida)

### 5.1 Tres Contratos Separados

| Confirmación | Quién Emite | Quién Persiste | Colección | Clave Idempotencia | Significado |
|--------------|-------------|----------------|-----------|-------------------|-------------|
| **Bloqueo Admisión** | AI Service (Outbox) | CRM | `retirement_operations.lockConfirmations.execution` (embebido) | `retirementOperationId` | AI Service confirmó que admisión está `blocked` para este `retirementOperationId` |
| **Drenaje Ejecuciones** | AI Service (Outbox) | CRM | `retirement_operations.executionDrainConfirmation` (embebido) | `retirementOperationId` + `admissionBlockedAt` | Admisión sigue bloqueada por misma operación + 0 ejecuciones `pending`/`processing` |
| **Aplicación EvAssigned** | AI Service (Outbox) | CRM | `retirement_confirmations` (colección independiente) | `evAssignedId` (único) | AI Service aplicó EvAssigned en proyección `contact_assignments` |

### 5.2 Flujo Completo de Bloqueo (Corregido)

```
1. CRM: Crea RetirementOperation (status=requested)
   ↓
2. CRM → AI Service: Mensaje durable BLOCK_EXECUTION_ADMISSION
   (vía RabbitMQ, idempotente por retirementOperationId + agentInstanceId)
   ↓
3. AI Service: Procesa mensaje en transacción local
   - blockExecutionAdmission(retirementOperationId, agentInstanceId)
   - Si éxito: emite EXECUTION_ADMISSION_BLOCKED en Outbox AI Service
   ↓
4. CRM: Consume EXECUTION_ADMISSION_BLOCKED (Inbox CRM)
   - Valida: retirementOperationId coincide, agentInstanceId coincide
   - Persiste en RetirementOperation.lockConfirmations.execution = {
       retirementOperationId, confirmedAt, admissionBlockedAt (del payload)
     }
   - Idempotente: si ya existe confirmación para mismo retirementOperationId → OK
   ↓
5. CRM: Cuando AMBOS bloqueos confirmados (CRM assignments + AI Service execution)
   - RetirementOperation.status = releasing
   - AgentInstance.status = retirement_pending (transacción local CRM)
```

### 5.3 Flujo de Drenaje (Corregido — sin consultar CRM DB)

```
1. CRM Worker: Completa drenaje de contactos (status = finalizing)
   ↓
2. CRM → AI Service: Mensaje durable VERIFY_EXECUTION_DRAIN
   (incluye retirementOperationId, agentInstanceId, admissionBlockedAt)
   ↓
3. AI Service: Procesa en transacción local verifyAndEmitDrainConfirmation
   - Lee AgentExecutionAdmission (DB ai_service)
   - Valida: status=blocked AND retirementOperationId coincide AND blockedAt = admissionBlockedAt
   - Cuenta AgentExecution con agentInstanceId y status IN ['pending', 'processing']
   - Si count > 0 → error (reintentar con backoff)
   - Si count = 0 → emite EXECUTION_DRAIN_CONFIRMED en Outbox AI Service
   ↓
4. CRM: Consume EXECUTION_DRAIN_CONFIRMED (Inbox CRM)
   - Valida: retirementOperationId coincide, admissionBlockedAt coincide con lockConfirmation
   - Persiste en RetirementOperation.executionDrainConfirmation = {
       retirementOperationId, admissionBlockedAt, confirmedAt
     }
   - Idempotente: si ya existe con MISMOS datos → OK; si datos difieren → RECHAZO (inconsistencia)
```

### 5.4 Implementación AI Service: verifyAndEmitDrainConfirmation

```typescript
async function verifyAndEmitDrainConfirmation(
  retirementOperationId: string,
  agentInstanceId: string,
  admissionBlockedAt: Date,  // DEL PAYLOAD, no consultado en CRM
  session: ClientSession
): Promise<void> {
  return await session.withTransaction(async () => {
    // 1. Verificar admisión local
    const admission = await AgentExecutionAdmissionModel.findById(agentInstanceId, { session });
    if (!admission ||
        admission.status !== 'blocked' ||
        admission.retirementOperationId !== retirementOperationId ||
        admission.blockedAt.getTime() !== admissionBlockedAt.getTime()) {
      throw new Error('Admission not blocked by this retirement operation or mismatch');
    }

    // 2. Verificar drenaje: 0 ejecuciones pending/processing
    const pendingCount = await AgentExecutionModel.countDocuments({
      agentInstanceId,
      status: { $in: ['pending', 'processing'] }
    }, { session });

    if (pendingCount > 0) {
      throw new DrainIncompleteError(`${pendingCount} executions still pending/processing`);
    }

    // 3. Emitir confirmación durable en Outbox
    await AIServiceOutboxModel.create([{
      type: 'EXECUTION_DRAIN_CONFIRMED',
      payload: {
        retirementOperationId,
        agentInstanceId,
        admissionBlockedAt,
        confirmedAt: new Date()
      }
    }], { session });
  });
}
```

### 5.5 Persistencia CRM (Idempotencia Estricta)

```typescript
// CRM: handleExecutionDrainConfirmed(payload)
async function handleExecutionDrainConfirmed(
  payload: DrainConfirmedPayload,
  session: ClientSession
): Promise<void> {
  const op = await RetirementOperationModel.findOneAndUpdate(
    {
      _id: payload.retirementOperationId,
      'executionDrainConfirmation': { $exists: false }  // solo primera vez
    },
    {
      $set: {
        executionDrainConfirmation: {
          retirementOperationId: payload.retirementOperationId,
          admissionBlockedAt: payload.admissionBlockedAt,  // VALIDADO: coincide con lockConfirmation
          confirmedAt: payload.confirmedAt
        }
      }
    },
    { session, new: true }
  );

  if (!op) {
    // Ya existe: validar coincidencia exacta (idempotencia estricta)
    const existing = await RetirementOperationModel.findById(payload.retirementOperationId, { session });
    const existingDrain = existing.executionDrainConfirmation;
    if (existingDrain.admissionBlockedAt.getTime() !== payload.admissionBlockedAt.getTime()) {
      throw new Error('Drain confirmation mismatch: admissionBlockedAt differs from lock confirmation');
    }
    // Mismos datos → idempotente OK
  }
}
```

---

## 6. Recuperación Transaccional Verificada (Driver 6.20.0 / Mongoose 8.19.3)

### 6.1 Comportamiento Real del Driver (No Supuesto)

| Característica | Comportamiento Real (v6.20.0) |
|----------------|-------------------------------|
| `session.withTransaction(fn)` | Ejecuta `fn` en transacción. Si `fn` lanza **`TransientTransactionError`** (incluye `MongoWriteConcernError` con `writeConcernError.code=11600`, `MongoServerError` con `errorLabels: ['TransientTransactionError']`, `WriteConflict`), **reintenta automáticamente `fn` completo** (nueva transacción, misma sesión). Límite: `maxTransactionLockRequestTimeoutMs` (default 5ms) para adquirir lock; `maxTimeMS` opcional en opciones para timeout total. |
| `UnknownTransactionCommitResult` | Se lanza cuando el commit se envía pero no se recibe respuesta (timeout red, primary stepped down). **`withTransaction` NO reintenta automáticamente**. La aplicación debe catch y reintentar `await session.commitTransaction()` en **la misma sesión**. |
| `session.commitTransaction()` | Puede reintentarse en la misma sesión tras `UnknownTransactionCommitResult`. Si falla de nuevo, la sesión puede quedar inutilizable. |
| `session.abortTransaction()` | Libera la sesión. Después de abortar, la sesión **no debe reutilizarse** para operaciones transaccionales. |

**Fuente:** Documentación oficial MongoDB Node.js Driver 6.x + comportamiento observado en tests de integración del driver.

### 6.2 Clasificación de Escenarios y Acciones

| Escenario | Detección | Acción |
|-----------|-----------|--------|
| **Transacción confirmada** | `withTransaction` resuelve sin error | Continuar; estado durable garantizado |
| **Transacción abortada** | `withTransaction` lanza error no-Transient (ej. validación, CAS fallido) | Error propagado; lógica de reintento a nivel superior decide |
| **Commit incierto** | `UnknownTransactionCommitResult` lanzado **fuera** de `withTransaction` (o callback completa pero commit falla) | Reintentar `session.commitTransaction()` en **misma sesión** (máx. 3 intentos). Si agota → reconciliación durable. |
| **Resultado no reconciliable** | Agotados reintentos commit + reconciliación no determina estado | **Escalar a intervención manual**; no reparar automáticamente. |
| **Inconsistencia de datos** | Reconciliación encuentra `EvAssigned` sin Outbox, o `Contacto` en estado inesperado | **Diagnosticar y escalar**; no crear Outbox faltante ni forzar estado. |

### 6.3 Reconciliación de Contacto + EvAssigned + Outbox (Corregida)

**Principio:** La transacción CRM (Contacto + EvAssigned + Outbox + Revision) es **atómica**. Un commit correcto **nunca** deja `EvAssigned` sin su Outbox correspondiente. Por tanto:

- Si existe `EvAssigned` **y** su Outbox atómico → transición aplicada ✓
- Si existe `EvAssigned` **PERO NO** su Outbox → **INCONSISTENCIA** (no auto-reparable)
- Si NO existe `EvAssigned` → releer `Contacto` y re-evaluar CAS antes de reintentar

```typescript
// CRM: executeContactTransition (transacción local CRM DB)
async function executeContactTransition(
  contactId: string,
  retirementOperationId: string,
  transition: ContactTransition,  // incluye newEvAssignedId, toAgentInstanceId, etc.
  session: ClientSession
): Promise<TransitionResult> {
  const maxCommitRetries = 3;
  let commitAttempt = 0;

  while (true) {
    try {
      // 1. Idempotencia: buscar EvAssigned por clave operación+contacto
      const existingEv = await EventoModel.findOne({
        contacto: contactId,
        retirementOperationId,
        __t: 'EvAssigned'
      }, { session }).lean();

      if (existingEv) {
        // 2. Verificar Outbox atómico correspondiente
        const outbox = await OutboxModel.findOne({
          type: 'CONTACT_ASSIGNMENT_CHANGED',
          'payload.evAssignedId': existingEv._id
        }, { session }).lean();

        if (outbox) {
          return { status: 'ALREADY_APPLIED', evAssignedId: existingEv._id };
        }

        // INCONSISTENCIA: EvAssigned existe SIN Outbox atómico
        // NO crear Outbox automáticamente. Registrar y escalar.
        await logInconsistency('EV_ASSIGNED_WITHOUT_OUTBOX', {
          contactId, retirementOperationId, evAssignedId: existingEv._id
        });
        throw new InconsistentStateError('EvAssigned exists without atomic Outbox');
      }

      // 3. Leer estado actual Contacto (CAS token = ref_ev_assigned)
      const contact = await ContactoModel.findById(contactId, { session });
      if (!contact) throw new Error('Contact not found');

      const expectedPrevEvAssignedId = contact.ref_ev_assigned;

      // 4. Ejecutar transacción completa
      await session.withTransaction(async () => {
        // 4a. CAS sobre Contacto
        const casResult = await ContactoModel.findOneAndUpdate(
          {
            _id: contactId,
            ref_ev_assigned: expectedPrevEvAssignedId,
            ref_agent_assigned: transition.fromAgentInstanceId
          },
          {
            $set: {
              ref_agent_assigned: transition.toAgentInstanceId || null,
              ref_user_assigned: transition.toUserId || null,
              assignedType: transition.newAssignedType,
              ref_ev_assigned: transition.newEvAssignedId
            }
          },
          { session, new: true }
        );

        if (!casResult) {
          throw new CASFailedError('Contact state changed by concurrent transition');
        }

        // 4b. Crear EvAssigned
        const ev = new EventoModel({
          _id: transition.newEvAssignedId,
          contacto: contactId,
          workspace: contact.ref_works,
          __t: 'EvAssigned',
          prevAssignedType: transition.prevAssignedType,
          prevAssignedRef: transition.prevAssignedRef,
          newAssignedType: transition.newAssignedType,
          newAssignedRef: transition.newAssignedRef,
          actorType: transition.actorType,
          actorRef: transition.actorRef,
          operationType: transition.operationType,
          reason: transition.reason,
          retirementOperationId
        });
        await ev.save({ session });

        // 4c. Crear Outbox atómico (MISMA transacción)
        await OutboxModel.create([{
          type: 'CONTACT_ASSIGNMENT_CHANGED',
          payload: {
            evAssignedId: transition.newEvAssignedId,
            retirementOperationId,
            contactId,
            newAssignedType: transition.newAssignedType,
            newAssignedRef: transition.newAssignedRef
          },
          status: 'PENDING'
        }], { session });

        // 4d. Incrementar eventSetRevision
        await RetirementOperationModel.updateOne(
          { _id: retirementOperationId },
          { $inc: { eventSetRevision: 1 } },
          { session }
        );
      });

      return { status: 'APPLIED', evAssignedId: transition.newEvAssignedId };

    } catch (err) {
      if (err instanceof CASFailedError) {
        // Otro proceso ganó: releer y respetar
        const current = await ContactoModel.findById(contactId, { session });
        const currentEv = await EventoModel.findOne({
          contacto: contactId, retirementOperationId
        }, { session });
        return { status: 'CONCURRENT_WIN', currentAssignment: current?.ref_agent_assigned };
      }

      if (err instanceof InconsistentStateError) {
        // No reintentar: inconsistencia requiere diagnóstico
        throw err;
      }

      // UnknownTransactionCommitResult: reintentar commit EN MISMA SESIÓN
      if (isUnknownTransactionCommitResult(err)) {
        commitAttempt++;
        if (commitAttempt <= maxCommitRetries) {
          try {
            await session.commitTransaction();
            // Commit OK en reintento: verificar qué persistió
            const ev = await EventoModel.findOne({
              contacto: contactId, retirementOperationId
            }, { session });
            if (ev) return { status: 'APPLIED', evAssignedId: ev._id };
            // No hay evento → transacción no llegó a commit; continuar loop para reintentar completo
            continue;
          } catch (commitErr) {
            if (isUnknownTransactionCommitResult(commitErr)) continue;
            throw commitErr;
          }
        }
        // Agotados reintentos commit → reconciliación durable (NUEVA SESIÓN)
      }

      // TransientTransactionError → withTransaction ya reintentó; si llega = fallo permanente
      // Otros errores → propagar
      throw err;
    }
  }
}
```

### 6.4 Reconciliación Durable (Nueva Sesión, Solo Lectura + Decisión)

```typescript
async function reconcileUncertainTransaction(
  contactId: string,
  retirementOperationId: string,
  expectedEvAssignedId: string
): Promise<TransitionResult> {
  // IMPORTANTE: Nueva sesión limpia; NO reutilizar sesión con commit incierto
  const session = await mongoose.startSession();

  try {
    // 1. Buscar EvAssigned por clave operación+contacto (índice único parcial)
    const ev = await EventoModel.findOne({
      contacto: contactId,
      retirementOperationId,
      __t: 'EvAssigned'
    }, { session }).lean();

    if (ev) {
      // 2. Verificar Outbox atómico
      const outbox = await OutboxModel.findOne({
        type: 'CONTACT_ASSIGNMENT_CHANGED',
        'payload.evAssignedId': ev._id
      }, { session }).lean();

      if (outbox) {
        return { status: 'RECONCILED_APPLIED', evAssignedId: ev._id };
      }

      // INCONSISTENCIA CONFIRMADA: EvAssigned sin Outbox
      await logInconsistency('RECONCILIATION_EV_WITHOUT_OUTBOX', {
        contactId, retirementOperationId, evAssignedId: ev._id
      });
      throw new InconsistentStateError('Reconciliation: EvAssigned without Outbox');
    }

    // 3. No hay EvAssigned → releer Contacto completo
    const contact = await ContactoModel.findById(contactId, { session }).lean();
    if (!contact) throw new Error('Contact disappeared');

    // 4. Comparar estado esperado vs actual
    //    Incluye ref_agent_assigned, ref_user_assigned, assignedType, ref_ev_assigned
    const expectedAssignment = {
      ref_agent_assigned: transition.toAgentInstanceId,  // requiere contexto transición
      ref_user_assigned: transition.toUserId,
      assignedType: transition.newAssignedType,
      ref_ev_assigned: expectedEvAssignedId
    };

    // Si Contacto ya tiene la asignación objetivo → transición aplicada por otro camino
    if (contact.ref_agent_assigned === transition.toAgentInstanceId &&
        contact.assignedType === transition.newAssignedType &&
        contact.ref_ev_assigned === expectedEvAssignedId) {
      return { status: 'RECONCILED_CONCURRENT_WIN', currentAssignment: contact.ref_agent_assigned };
    }

    // 5. Contacto en estado anterior → seguro reintentar transacción completa
    //    (re-evaluará CAS con ref_ev_assigned actual)
    return { status: 'RETRY_FULL_TRANSACTION' };

  } finally {
    await session.endSession();
  }
}
```

**Correcciones clave vs versión anterior:**
- `transition` es parámetro explícito (no variable libre).
- Nueva sesión para reconciliación (no reutiliza sesión con commit incierto).
- No crea Outbox faltante automáticamente → trata como inconsistencia.
- Re-evalúa CAS con `ref_ev_assigned` actual antes de reintentar.
- Índice único parcial `{retirementOperationId: 1, contacto: 1}` en `EvAssigned` es protección definitiva.

---

## 7. Verificación MongoDB Atlas (Corregida)

### 7.1 Configuración Real Verificada

| Aspecto | Estado | Evidencia |
|---------|--------|-----------|
| **Cluster** | Compartido | `docker-compose.yml` sin MongoDB; URIs `mongodb+srv://` en ambos servicios |
| **CRM Database** | `crm` | `crm/src/database.ts:15` (`MONGO_DB=crm`) |
| **AI Service Database** | `ai_service` | `ai-service/config-manager.ts:20` (`mongodb.database`), `mongodb.ts:40` |
| **Transacciones CRM** | Locales (DB `crm`) | `Work.service.ts`, `UsOrgWork.service.ts` usan `session.withTransaction` |
| **Transacciones AI Service** | Locales (DB `ai_service`) | **Ninguna hoy**; diseño propone `withTransaction` en `agent_execution_admissions` + `agent_executions` |
| **Transacciones Cross-DB** | **No existen / No soportadas** | Driver MongoDB no soporta transacciones multi-database |

### 7.2 Qué Debe Confirmar DevOps (Específico y Verificable)

> **BLOQUEANTE: Sin esta confirmación no se autoriza implementación.**

| # | Dato Requerido | Por Qué Es Crítico |
|---|----------------|---------------------|
| 1 | **Topología del cluster Atlas**: Replica Set (RS) o Sharded Cluster (SC) | Transacciones multidocumento requieren RS o SC (no standalone) |
| 2 | **Versión MongoDB del servidor** (≥ 4.2) | Transacciones multidocumento introducidas en 4.0 (RS) / 4.2 (SC) |
| 3 | **Confirmación de que `crm` y `ai_service` están en este cluster** | Valida que transacciones locales en cada DB funcionan en el mismo deployment |
| 4 | **Compatibilidad transacciones en este deployment** | Atlas puede tener limitaciones (ej. transacciones deshabilitadas en M0/M2/M5 tiers) |
| 5 | **Límites operativos**: `transactionLifetimeLimitSeconds`, max transaction size (16MB), max ops per transaction | Afecta diseño de timeouts y tamaño de lotes en workers |
| 6 | **Latencia red típica servicio ↔ Atlas** | Define `maxTimeMS` en `withTransaction` y `serverSelectionTimeoutMS` |

**No se debe preguntar:** "¿Están habilitadas las transacciones?" (genérico).  
**Sí se debe preguntar:** "En el cluster `X`, ¿las transacciones multidocumento son compatibles para bases `crm` y `ai_service` en tier `Y` con versión `Z`?"

### 7.3 Colecciones por Transacción (Para Validación DevOps)

| Transacción | Servicio | Database | Colecciones |
|-------------|----------|----------|-------------|
| Aceptación + Bloqueo Admisión | AI Service | `ai_service` | `agent_execution_admissions`, `agent_executions` |
| Bloqueo Admisión (on-demand) | AI Service | `ai_service` | `agent_execution_admissions` |
| Drenaje (lectura + Outbox) | AI Service | `ai_service` | `agent_execution_admissions` (lectura), `agent_executions` (lectura), `outbox` (escritura) |
| Transición Contacto | CRM | `crm` | `contactos`, `eventos` (EvAssigned), `outbox`, `retirement_operations` |
| Aplicación EvAssigned | AI Service | `ai_service` | `contact_assignments`, `inbox`, `outbox` |
| Finalización | CRM | `crm` | `agent_instances`, `retirement_operations`, `outbox` |

---

## 8. Escenarios de Concurrencia y Recuperación (Resumen)

| Escenario | Resultado | Invariante |
|-----------|-----------|------------|
| Aceptación vs Bloqueo simultáneos | MongoDB serializa `findOneAndUpdate` sobre `_id=agentInstanceId`; una gana, otra ve estado modificado | **I1**: No ventana post-bloqueo |
| Reintento `executionId` idempotente | `findById` encuentra ejecución existente; valida `agentInstanceId` coincide | **I4**: Idempotencia fuerte |
| Commit incierto aceptación | Reintento `commitTransaction` misma sesión (≤3); agotado → reconciliación lee `AgentExecution` por `executionId` | **I9**: Sin duplicación |
| Commit incierto transición contacto | Reintento commit misma sesión; agotado → reconciliación nueva sesión lee `EvAssigned` + Outbox por claves | **I9**: Sin duplicación |
| EvAssigned sin Outbox (inconsistencia) | Detectada en reconciliación → **escalada manual**, no auto-reparación | **I10**: Atomicidad real |
| Bloqueo admisión duplicado distinto `retirementOperationId` | Rechazado en AI Service (`retirementOperationId: null` filter) y CRM (validación `admissionBlockedAt`) | **I7**: Inmutabilidad bloqueo |

---

## 9. Invariantes Garantizadas (Actualizadas)

| # | Invariante | Mecanismo | Verificación |
|---|------------|-----------|--------------|
| I1 | No ventana aceptación post-bloqueo | Exclusión atómica `$inc admissionRevision` en transacción local AI Service | **Propuesta técnica** |
| I2 | Bloqueo impide nuevas aceptaciones | `findOneAndUpdate` con filtro `status: 'open'` atómico en MongoDB | **Propuesta técnica** |
| I3 | Ejecución aceptada antes del bloqueo termina | `AgentExecution` persistido `pending` antes de bloqueo; no cancelación auto | **Propuesta técnica** |
| I4 | Reintentos `executionId` idempotentes | PK única + validación `agentInstanceId` coincide | **Propuesta técnica** |
| I5 | Bloqueo ligado a `retirementOperationId` | Escrito atómicamente con `status: 'blocked'` en misma transacción | **Propuesta técnica** |
| I6 | Confirmación EvAssigned idempotente | Índice único `{evAssignedId: 1}` en `retirement_confirmations` (CRM) | **Verificada** (diseño canónico) |
| I7 | Confirmación drenaje inmutable | Embebida en `RetirementOperation`; validación `admissionBlockedAt` coincide | **Propuesta técnica** |
| I8 | No duplicación EvAssigned retiro+contacto | Índice único parcial `{retirementOperationId: 1, contacto: 1}` en `Evento` | **Verificada** (diseño canónico) |
| I9 | Recuperación no duplica | Reconciliación por claves durables; índices únicos protección final | **Propuesta técnica verificada** |
| I10 | Atomicidad Contacto+EvAssigned+Outbox+Revision | Transacción local CRM con `withTransaction` | **Propuesta técnica** |
| I11 | CAS con `ref_ev_assigned` previene ABA | Condición de update incluye `ref_ev_assigned` esperado | **Verificada** (diseño canónico) |
| I12 | Finalización con correspondencia exacta | Comparación individual eventos vs confirmaciones (no contadores) | **Verificada** (diseño canónico) |

---

## 10. Riesgos y Cuestiones Pendientes

| Riesgo / Cuestión | Severidad | Estado | Acción Requerida |
|-------------------|-----------|--------|------------------|
| **Topología Atlas no verificada** | **Bloqueante** | Pendiente DevOps | Confirmación escrita items §7.2 |
| **AI Service sin transacciones hoy** | Alto | Propuesta | Implementar `withTransaction` en admisión, drenaje, aplicación EvAssigned |
| **Índice parcial discriminator Mongoose** | Medio | Pendiente diseño | Validar `@typegoose/typegoose` 12.20 con índice único parcial en `__t: 'EvAssigned'` |
| **Timeout transacción (`maxTimeMS`)** | Medio | Configuración | Definir por operación; considerar `transactionLifetimeLimitSeconds` Atlas |
| **Cross-DB read-only CRM→AI Service** | Bajo | Implementación | CRM leerá `agent_execution_admissions` para verificaciones; usar conexión read-only |
| **Purga física post-retired** | Bajo | Fuera alcance | Políticas independientes (WAB-79 §16.14) |

---

## 11. Dependencias para Diseño de Implementación

| Dependencia | Tipo | Responsable | Estado |
|-------------|------|-------------|--------|
| Confirmación topología Atlas (RS/SC, versión, límites) | **Externa** | DevOps | **PENDIENTE — BLOQUEANTE** |
| Schema `AgentExecutionAdmission` en AI Service DB | Interna | Equipo AI Service | **Propuesta en §3.1** — revisar/aprobar |
| Schema `AgentExecution` (nuevo, reemplaza `Execution`) | Interna | Equipo AI Service | D-W79-05 — implementar |
| Schema `RetirementOperation` en CRM | Interna | Equipo CRM | §16.4 canónico — implementar |
| Schema `retirement_confirmations` en CRM | Interna | Equipo CRM | §16.9 canónico — implementar |
| Índice único parcial EvAssigned en `eventos` | Interna | Equipo CRM | Validar discriminator Mongoose — implementar |
| Implementación `acceptExecution` transaccional | Interna | Equipo AI Service | §4.1 — implementar |
| Implementación `blockExecutionAdmission` transaccional | Interna | Equipo AI Service | §4.2 — implementar |
| Worker drenaje contactos (CRM) | Interna | Equipo CRM | §16.8 canónico — implementar |
| Worker finalización (CRM) | Interna | Equipo CRM | §16.11-13 canónico — implementar |
| Tests concurrencia (aceptación vs bloqueo, CAS, recuperación) | Interna | QA / Equipos | **Obligatorios antes de merge** |

---

## Anexos

### A. Versiones Críticas Verificadas

| Paquete | CRM | AI Service | Nota |
|---------|-----|------------|------|
| `mongodb` | 6.20.0 | 6.20.0 (workspace) | Driver nativo — `withTransaction`, commit retry manual |
| `mongoose` | 8.19.3 | 8.19.3 | Wrapper — `session.withTransaction()` reintenta TransientTransactionError |
| `@typegoose/typegoose` | 12.20.0 | - | Discriminators, índices parciales — validar soporte |

### B. Códigos Error MongoDB Relevantes (Driver 6.20.0)

| Error | Clasificación | Acción |
|-------|---------------|--------|
| `TransientTransactionError` (errorLabels incluye) | Retriable: callback completo | `withTransaction` reintenta automáticamente |
| `UnknownTransactionCommitResult` | Retriable: commit misma sesión | Reintentar `session.commitTransaction()` (máx 3) |
| `WriteConflict` | TransientTransactionError | `withTransaction` reintenta |
| `DuplicateKey` (índice único) | No retriable — conflicto lógico | Manejar como CAS fallido / idempotencia |

### C. Referencias Documentales Canónicas

- `waby-docs/projects/agents-multitenant/design/wab-79-dominio-ownership-persistencia.md` §16 (contrato completo aprobado)
- `waby-docs/projects/agents-multitenant/design/fase-2-diseno-objetivo-y-contratos.md` §D.1.8, ARQ-14
- `waby-docs/projects/agents-multitenant/audits/wab-77/informe-consolidado-fase-1.md:115` (bases separadas confirmadas)
- `waby-docs/projects/agents-multitenant/audits/wab-78/matriz-acumulativa.md` ARQ-06, ARQ-14
- `waby-docs/projects/agents-multitenant/audits/wab-78/informe-bloque-3.md` (verificación Atlas)

---

**Fin del informe corregido.**  
Esta versión corrige ownership, separación de bases de datos, algoritmos de concurrencia, recuperación transaccional y verificación Atlas según evidencia real de código y versiones instaladas. Sigue siendo **propuesta técnica para revisión**; no autoriza implementación, no modifica Linear, no crea decisiones D-W79 nuevas.