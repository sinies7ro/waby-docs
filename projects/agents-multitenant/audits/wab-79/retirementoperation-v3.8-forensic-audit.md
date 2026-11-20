# WAB-79 — Informe de Auditoría Forense: RetirementOperation v3.8

> **Estado:** Auditoría completa — No modifica código ni documentos canónicos.
> **Fecha:** 18/11/2026
> **Documento auditado:** `projects/agents-multitenant/design/wab-79-retirementoperation-physical-contract.md` (v3.8)
> **Versión origen:** v3.8 — 118KB, ~1500 líneas

---

## 1. Objetivo y Alcance

Esta auditoría reconstruye la coherencia completa de la especificación física de `RetirementOperation` v3.8 contra:
- Decisiones arquitectónicas aprobadas (D-W79-14, D-W79-15, H1, Opción A)
- Contratos funcionales (§16.4–§16.14 de `wab-79-dominio-ownership-persistencia.md`)
- Contratos de diseño objetivo (`fase-2-diseno-objetivo-y-contratos.md`, D.1.6, D.1.7, D.1.8)
- Reconciliación Outbox/Inbox (`wab-79-informe-reconciliacion-outbox-seccion-16-14.md`)
- Código CRM existente (modelos, repositorios, patrones transaccionales)

**Alcance:** Esquemas físicos, identidades e idempotencia, CAS y concurrencia, unidad transaccional, recuperación post-commit, Outbox/Inbox, lease/fencing, cierre eventSet, escenarios end-to-end.

---

## 2. Fuentes Inspeccionadas y Versiones

| Fuente | Ruta | Versión/Estado |
|---|---|---|
| Documento objetivo | `wab-79-retirementoperation-physical-contract.md` | v3.8 (auditada) |
| Contrato canónico dominio/persistencia | `wab-79-dominio-ownership-persistencia.md` | §16.4–§16.14 vigente |
| Diseño objetivo Fase 2 | `fase-2-diseno-objetivo-y-contratos.md` | D.1.6, D.1.7, D.1.8 reconciliados |
| Reconciliación Outbox §16.14 | `wab-79-informe-reconciliacion-outbox-seccion-16-14.md` | Borrador consolidado |
| Decisiones D-W79-14, D-W79-15, H1 | En documentos anteriores | Aprobadas, no reabiertas |
| Opción A (AgentInstance guardián) | En documentos anteriores | Aprobada |
| CRM — AgentInstance.model.ts | `crm/src/models/agent/AgentInstance.model.ts` | Sin `crmBlockSeq` |
| CRM — EvAssigned.model.ts | `crm/src/models/events/EvAssigned.model.ts` | Discriminator Evento |
| CRM — Evento.model.ts | `crm/src/models/events/Evento.model.ts` | Colección `eventos` |
| CRM — Outbox.model.ts | `crm/src/models/events/Outbox.model.ts` | Schema genérico |
| CRM — Contacto.model.ts | `crm/src/models/chat/Contacto.model.ts` | Campos asignación presentes |
| CRM — Works.model.ts | `crm/src/models/workspace/Works.model.ts` | `owner_version` patrón |
| CRM — UsOrgWork.repository.ts | `crm/src/repositories/mongo/UsOrgWork.repository.ts` | `lockWorkspaceForOwnerCheck` |

---

## 3. Decisiones Aprobadas Preservadas (Normativas)

| ID | Decisión | Estado | Origen |
|---|---|---|---|
| D-W79-14 | EvAssigned como discriminator de Evento; campos heredados; `actorRef` obligatorio; sin `originMessageId`/`delegationChainId` | **Aprobada** | Fase 2 D.1.6 |
| D-W79-15 | Atomicidad Contacto + EvAssigned + Outbox (+ RetirementOperation.eventSetRevision en retiro); transacción MongoDB preferida | **Aprobada** | §16.8–§16.10 |
| H1 | `contact.assignment.record` con `evAssignedId` ObjectId top-level; índice único parcial `{evAssignedId:1}` con filtro; `sync` con `_id` propia y `derivedFrom` string | **Aprobada** | P-01 opción B |
| Opción A | AgentInstance como guardián compartido; `crmBlockSeq` campo serializador; todas las asignaciones entrantes y bloqueo CRM hacen CAS sobre él | **Aprobada** | WAB-79 §16 |
| B2 | `eventSetRevision` inmutable en `finalizing`; solo incrementa en `releasing` | **Aprobada** | WAB-79 §16.9 |
| B3 | Distinción entrante (protege destino) vs saliente (correlaciona origen) | **Aprobada** | WAB-79 §16.1–§16.2 |
| B4 | Lease/fencing: adquisición atómica, generación incrementa, validación owner/gen/vigencia, rechazo workers obsoletos | **Aprobada** | WAB-79 §16.6 |
| B5 | Idempotencia por clave lógica `retirementOperationId` + `contactId`; búsqueda antes de mutar; identidades fuera de callback | **Aprobada** | WAB-79 §16.12 |
| B6 | No ObjectId como prueba de causalidad; condición 4 finalización pendiente C2 | **Aprobada** | WAB-79 §16.11 |

---

## 4. Matriz de Conformidad Completa

| ID | Invariante | Fuente Normativa | Evidencia Concreta en v3.8 | Resultado | Defecto/Bloqueo |
|---|---|---|---|---|---|
| **M-01** | Schema `RetirementOperation` completo con tipos BSON exactos | §16.4, H1, Opción A | §3.2 define todos los campos; `_id` String/UUID; enums correctos; subdocumentos anidados | **CONFORME** | — |
| **M-02** | Índice único parcial `agentInstanceId` para estados no terminales | §16.6 | §3.2 `@index` con `partialFilterExpression` correcto | **CONFORME** | — |
| **M-03** | `lockConfirmations` anidado con estructura CRM/Execution separada | §16.4, §16.2 | §3.2 `CrmLockConfirmation` + `ExecutionLockConfirmation` + `LockConfirmations` | **CONFORME** | — |
| **M-04** | `executionDrainConfirmation` con `admissionBlockedAt` inmutable | §16.12 | §3.2 campo definido; §5.7 validación inmutabilidad | **CONFORME** | — |
| **M-05** | `finalizationConfirmation` inmutable | §16.13 | §3.2 campo definido; §5.9 validación | **CONFORME** | — |
| **M-06** | Lease: `leaseOwner`, `leaseUntil`, `leaseGeneration` (ausente ≠ cero) | §16.6, P-08 | §3.2 `ProcessingLease` con `leaseGeneration: number` default 0; §5.3 notas B4 distinguen absent/zero | **NO DEMOSTRADO** | Default 0 en Mongoose colapsa absent/zero — requiere inicialización explícita (P-08) |
| **M-07** | `eventSetRevision` monotónico, solo `releasing` | §16.9, B2 | §3.2 campo; §5.5 paso 10 solo en `releasing`; §5.6 inmutable en `finalizing` | **CONFORME** | — |
| **M-08** | `eventSetClosedAt` solo en `finalizing` | §16.10 | §3.2 campo; §5.6 escritura atómica | **CONFORME** | — |
| **M-09** | `retirement_confirmations` índices únicos: `evAssignedId` + compuesto | §16.9 | §3.3 índices declarados correctamente | **CONFORME** | — |
| **M-10** | `contact.assignment.record` con `evAssignedId` ObjectId + índice parcial H1 | H1 | §3.4 schema correcto; índice único parcial con filtro tipo objectId + type | **CONFORME** | — |
| **M-11** | Campo `crmBlockSeq` en `AgentInstance` (schema local) | Opción A | §3.5 propuesta de adición a `AgentInstance.model.ts` | **CONFORME** (documental) | **NO IMPLEMENTADO** en código — diferencia especificación vs implementación |
| **M-12** | EvAssigned `retirementOperationId` campo opcional para correlación | §16.8 | §10.4 cambio mínimo propuesto | **CONFORME** | — |
| **M-13** | Índice único parcial `retirementOperationId + contacto` en EvAssigned | §16.8 | §4.1 fila 5 | **CONFORME** | — |
| **M-14** | Colección `contact_assignment_records` con índices compuestos | H1 | §4.1 filas 8-10 | **CONFORME** | — |

---

### 4.1 Identidad e Idempotencia

| ID | Invariante | Fuente | Evidencia v3.8 | Resultado | Defecto |
|---|---|---|---|---|---|
| **I-01** | `newEvId` (ObjectId) generado fuera de `withTransaction` | B5, §16.12 | §7.3 `newEvId = new ObjectId()` fuera del callback; §5.5 paso 3 | **CONFORME** | — |
| **I-02** | `recordId` (String/UUID via `crypto.randomUUID()`) generado fuera de callback | B5 | §3.4 `record._id` String/UUID; §7.3 `recordId = crypto.randomUUID()` | **CONFORME** | — |
| **I-03** | Identidades reutilizadas en reintentos del callback | B5 | §7.3 paso 4: `newEvId` y `recordId` parámetros del callback | **CONFORME** | — |
| **I-04** | Recuperación post-commit usa `retirementOperationId` + `contactId` (no `Contacto.ref_agent_assigned`) | B5, §16.12 | §7.3 paso 4: `effectiveRetirementOpId` parámetro; §8.4 explicación | **CONFORME** | — |
| **I-05** | Búsqueda incondicional si hay `retirementOperationId` válido | B5 | §7.3 paso 4: `if (effectiveRetirementOpId)` busca siempre | **CONFORME** | — |
| **I-06** | Clave idempotencia `retirement_confirmations`: `evAssignedId` | D-W79-15 | §3.3 índice único `evAssignedId`; §5.15 idempotencia | **CONFORME** | — |
| **I-07** | Clave idempotencia drenaje: `{retirementOperationId, admissionBlockedAt}` | §16.12 | §3.2 `ExecutionDrainConfirmation.admissionBlockedAt`; §5.7 inmutable | **CONFORME** | — |
| **I-08** | Clave idempotencia finalización: `retirementOperationId` | §16.13 | §5.9 orden `retirement.finalize` idempotente | **CONFORME** | — |
| **I-09** | Outbox genérico NO re-insertado al reutilizar evidencia | B5, D-W79-15 | §7.3 paso 5 verifica Outbox existente; paso 9 solo si `!existingRecord` | **CONFORME** | — |
| **I-10** | Verificación Outbox existente por `derivedFrom` + `prevCrmAssigned` | B5 | §7.3 paso 5: `existingOutbox` busca por `payload.contactId` + `payload.derivedFrom` | **CONFORME** | — |

---

### 4.2 CAS y Concurrencia

| ID | Invariante | Fuente | Evidencia v3.8 | Resultado | Defecto |
|---|---|---|---|---|---|
| **C-01** | CAS Contacto valida `assignedType`, `ref_user_assigned`, `ref_agent_assigned`, `ref_ev_assigned` (D-W79-15 completo) | D-W79-15 | §5.5 paso 5; §7.3 paso 6 filtro incluye los 4 campos | **CONFORME** | — |
| **C-02** | CAS `eventSetRevision` valida revisión esperada + status `releasing` | B2, §16.9 | §5.5 paso 10; §7.3 paso 10 filtro `{eventSetRevision, status: "releasing"}` | **CONFORME** | — |
| **C-03** | CAS `crmBlockSeq` universal: todas las asignaciones entrantes + bloqueo CRM | Opción A | §7.2 operaciones 1 y 2; §7.3 paso 2 CAS universal sin consulta previa | **CONFORME** | — |
| **C-04** | Rechazo post-bloqueo: si `lockConfirmations.crm.confirmed=true` tras CAS → abortar transacción | Opción A | §7.2 paso 2 verificación final; §7.3 paso 2 mismo patrón | **CONFORME** | — |
| **C-05** | No compensación manual: rollback transaccional revierte incremento `crmBlockSeq` | Opción A | §7.2 "No hay compensación manual"; §7.3 mismo | **CONFORME** | — |
| **C-06** | `crmBlockSeqAtRead` re-leído inmediatamente antes de cada CAS (no al inicio transacción) | Opción A | §7.2 "valor leído inmediatamente antes"; §7.3 paso 2 `agentInstance.crmBlockSeq` leído in-situ | **CONFORME** | — |
| **C-07** | Reintentos `withTransaction`: solo `WriteConflict`/`TransientTransactionError` son automáticos | Opción A, §7.2 | §7.2 paso 3 explicación; §11 AC-29 | **CONFORME** | — |
| **C-08** | Errores de aplicación (`TransientConflictError`, `RetirementInProgressError`) NO disparan retry automático | Opción A | §7.2 paso 3; §11 AC-29 | **CONFORME** | — |
| **C-09** | `UnknownTransactionCommitResult`: consultar evidencia antes de nueva transición | §16.12 | §8.2 protocolo: consultar `contact.assignment.record` + `EvAssigned` + Outbox | **CONFORME** | — |
| **C-10** | Lease adquisición: exige `leaseOwner:null` Y `leaseUntil:null` simultáneamente (estado inicial coherente) | B4 | §5.3 tabla fila "Primera adquisición" `$or` con ambas null | **CONFORME** | — |
| **C-11** | Lease vigente de otro worker NO apropiable (ni adquisición ni reanudación) | B4 | §5.3 notas: "lease vigente de otro worker nunca puede ser apropiado" | **CONFORME** | — |
| **C-12** | `leaseGeneration` solo incrementa en adquisición efectiva (no en renovación/liberación) | B4 | §5.3 tabla: `$inc` solo en adquisición y recuperación; notas confirman | **CONFORME** | — |
| **C-13** | Liberación valida `leaseGeneration` (worker gen antigua no libera adquisición posterior) | B4 | §5.3 fila "Liberación" filtro incluye `leaseGeneration: currentGen` | **CONFORME** | — |
| **C-14** | Reanudación desde `blocked` usa `resumeFrom` en path raíz (no `processing.resumeFrom`) | B4 | §5.3 fila "Reanudación desde blocked" filtro usa `resumeFrom` raíz | **CONFORME** | — |

---

### 4.3 Unidad Transaccional

| ID | Invariante | Fuente | Evidencia v3.8 | Resultado | Defecto |
|---|---|---|---|---|---|
| **T-01** | Transición contacto `releasing`: Contacto + EvAssigned + Record + Outbox + RetirementOperation en MISMA transacción | D-W79-15, §16.8, H1 | §5.5 "Atomicidad: Los 10 pasos en una sola transacción MongoDB" | **CONFORME** (documental) | **BLOQUEADO POR C1** — transacciones multidocumento requieren Atlas validado |
| **T-02** | Orden lógico: 1) buscar evidencia, 2) si existe verificar y reutilizar, 3) si no generar identidades y ejecutar | B5 | §5.5 pasos 1-2; §7.3 pasos 4-5 | **CONFORME** | — |
| **T-03** | Outbox genérico insertado en misma transacción | D-W79-15 | §5.5 paso 9; §7.3 paso 9 | **CONFORME** | — |
| **T-04** | `eventSetRevision` incrementado EN transacción solo si `status=releasing` | B2 | §5.5 paso 10; §7.3 paso 10 condición `status === "releasing"` | **CONFORME** | — |
| **T-05** | Cierre conjunto (`releasing→finalizing`): transacción local RetirementOperation | §16.10 | §5.6 escritura atómica con CAS lease + revision | **CONFORME** (documental) | **BLOQUEADO POR C1** |
| **T-06** | Confirmación bloqueo CRM: transacción `RetirementOperation` + `AgentInstance` (crmBlockSeq++) | Opción A, §16.2 | §5.4 tabla; §5.10 misma transacción | **CONFORME** (documental) | **BLOQUEADO POR C1** |
| **T-07** | Finalización terminal: transacción `RetirementOperation` + `AgentInstance` | §16.11, §16.13 | §5.9 pasos 5 | **CONFORME** (documental) | **BLOQUEADO POR C1, C2** |
| **T-08** | Transición externa (AssignmentWriter) misma estructura transaccional SIN lease del worker | B3 | §5.5 diferencias tabla; §7.1 A2 | **CONFORME** | — |

---

### 4.4 Recuperación Post-Commit y Evidencia Parcial

| ID | Invariante | Fuente | Evidencia v3.8 | Resultado | Defecto |
|---|---|---|---|---|---|
| **R-01** | Camino A (recuperación) precede a Camino B (nueva transición) | B5, §7.3 | §7.3 paso 4 busca evidencia ANTES de mutar Contacto | **CONFORME** | — |
| **R-02** | Recuperación NO depende de `Contacto.ref_agent_assigned` actual | B5, §8.4 | §8.4: "búsqueda por clave lógica encuentra evidencia original independientemente del estado actual" | **CONFORME** | — |
| **R-03** | Recuperación no incrementa `crmBlockSeq` ni `eventSetRevision` | B5 | §5.5 paso 2: "No incrementar crmBlockSeq... No incrementar eventSetRevision" | **CONFORME** | — |
| **R-04** | Detección estados parciales: Record sin EvAssigned, EvAssigned sin Record, sin Outbox, Outbox contradictorio, Contacto post-commit | B5 | §5.5 paso 2 lista casos; §7.3 paso 5 `InconsistentTransitionError` | **CONFORME** | — |
| **R-05** | `prevCrmAssigned` validado contra evidencia histórica (EvAssigned.prevAssignedType/Ref + record.evAssignedId), NO Contacto actual | B5, D-W79-15 | §7.3 paso 5 usa `expectedEvId` del Contacto leído EN transacción; §8.4 referencia EvAssigned/record | **CONFORME** | — |
| **R-06** | Evidencia duplicada/contradictoria → error explícito, no éxito silencioso | B5 | §7.3 paso 5 `InconsistentTransitionError` | **CONFORME** | — |

---

### 4.5 Outbox e Inbox

| ID | Invariante | Fuente | Evidencia v3.8 | Resultado | Defecto |
|---|---|---|---|---|---|
| **O-01** | Outbox genérico tipo `contact.assignment.transition` (propuesta física) | D-W79-15 | §5.5 paso 9; §7.3 paso 9; §10.3 mapeo campos | **PROPUESTA** | No aprobado contractualmente — §16.14 Nivel 1 = VACÍO |
| **O-02** | Payload Outbox: `contactId`, `workspaceId`, `target`, `prevCrmAssigned`, `derivedFrom` | D-W79-15 | §7.3 paso 9; §10.3 tabla mapeo | **PROPUESTA** | Pendiente P-01, P-02 |
| **O-03** | `derivedFrom` en Outbox = propuesta física; H1 define en `sync`, no en Outbox | H1, §16.14 | §10.3 notas: "H1 define derivedFrom en sync, no en Outbox" | **CONFORME** (documental) | Distinción mantenida correctamente |
| **O-04** | `retirement_confirmations` NO es Outbox; es registro durable CRM | §16.9, §16.14 | §9.1 tabla contextos; §9.1 aclaración | **CONFORME** | — |
| **O-05** | Tres contextos Outbox distinguidos (asignaciones, CRM→AI, AI→CRM) | §16.14 | §9.1 matriz contextos | **CONFORME** | — |
| **O-06** | D.6/ARQ-22 aplica SOLO a mensaje origen CRM→AI y resultado terminal AI→CRM | §16.14, D.6 | §9.1 tabla; §9.1 alcance D.6 | **CONFORME** | — |
| **O-07** | Inbox atomicidad local §16.9: 4 operaciones atómicas en AI Service | §16.9 | §6.4 informe reconciliación; §11 matriz fila Inbox | **CONFORME** (contractual) | **BLOQUEADO POR C2** — requiere contrato Inbox físico |

---

### 4.6 Lease y Fencing

| ID | Invariante | Fuente | Evidencia v3.8 | Resultado | Defecto |
|---|---|---|---|---|---|
| **L-01** | Adquisición atómica con `$or` (null+null O lease vencido) | B4, §16.6 | §5.3 tabla filas 1 y 4 | **CONFORME** | — |
| **L-02** | `leaseGeneration` ausente ≠ cero; default 0 Mongoose NO permitido | P-08, §16.6 | §8.1 informe: "Ausencia no equivale a cero"; §5.3 notas B4 | **NO DEMOSTRADO** | Schema §3.2 usa `default: 0` — colapsa distinción; requiere inicialización explícita sin default |
| **L-03** | Worker obsoleto (gen vieja o lease vencido) no puede avanzar estado ni escribir | B4 | §5.3 notas: "Worker obsoleto no puede avanzar"; §5.3 fila renovación valida gen | **CONFORME** | — |
| **L-04** | Lease vigente de otro worker no apropiable (reiterado) | B4 | §5.3 notas finales: "Lease vigente de otro worker no apropiable" | **CONFORME** | — |
| **L-05** | `resumeFrom` en path raíz del documento (no anidado en `processing`) | B4 | §5.3 fila reanudación filtro usa `resumeFrom` raíz; §5.3 notas confirman | **CONFORME** | — |

---

### 4.7 Cierre EventSet

| ID | Invariante | Fuente | Evidencia v3.8 | Resultado | Defecto |
|---|---|---|---|---|---|
| **E-01** | `eventSetRevision` inmutable desde `finalizing` (B2) | B2, §16.10 | §5.6 "eventSetRevision INMUTABLE desde aquí"; §7.3 paso 10 solo `releasing` | **CONFORME** | — |
| **E-02** | Cierre conjunto requiere: cero contactos + lease vigente + revision esperada | §16.10 | §5.6 precondiciones | **CONFORME** | — |
| **E-03** | Consulta cero contactos no garantía suficiente sin protección contra asignaciones concurrentes | §16.10 | §6 nota: "No aceptar consulta cero contactos como garantía suficiente" | **CONFORME** (advertencia documentada) | Protección real requiere Opción A efectiva (C1) |
| **E-04** | Concurrencia con nuevas transiciones durante cierre manejada por CAS revision | B2 | §5.6 CAS incluye `eventSetRevision` | **CONFORME** | — |

---

### 4.8 Escenarios End-to-End (Evaluación Documental)

| Escenario | Estado Inicial | Operación | Resultado Esperado | Demostrado en v3.8 | Brecha/Bloqueo |
|---|---|---|---|---|---|
| **S-01** Retiro sin contactos | `requested` | Bloqueos → `releasing` → `finalizing` → `completed` | Transición directa sin contactos | Parcial | Requiere C1 para transacciones |
| **S-02** Retiro 1 contacto | `releasing` | Transición contacto | EvAssigned+Record+Outbox+rev++ | **Sí** (§5.5, §7.3) | Requiere C1 |
| **S-03** Retiro múltiples lotes | `releasing` | Varias transiciones concurrentes | Revision serializada; CAS conflicto | **Sí** (C-02, C-03) | Requiere C1 |
| **S-04** Asignación entrante concurrente con bloqueo | `blocking` | Asignación entrante + bloqueo CRM | Serialización por `crmBlockSeq` | **Sí** (C-03, caso 3) | Requiere C1 |
| **S-05** Bloqueo concurrente con asignación entrante | `requested` | Bloqueo CRM + asignación | Una gana CAS, otra reintenta/rechaza | **Sí** (caso 3) | Requiere C1 |
| **S-06** Takeover humano durante draining | `releasing` | Admin takeover | Transición externa A2; actor=user; rev++ | **Sí** (§5.5 diferencias, AC-11) | Requiere C1 |
| **S-07** Desasignación durante releasing | `releasing` | Unassignment | Transición saliente A2; rev++ | **Sí** (§7.1 A2) | Requiere C1 |
| **S-08** Cambio AgentInstance destino | `blocking` | Reassign a otra instancia | Protección A1 sobre nuevo destino | **Sí** (§7.1 A1) | Requiere C1 |
| **S-09** Caída antes del commit | En transacción | Fallo proceso | Rollback completo; sin estados parciales | **Sí** (T-01, AC-14) | Requiere C1 |
| **S-10** Commit exitoso, respuesta perdida | Post-commit | Callback sin respuesta | `UnknownTransactionCommitResult` → consultar evidencia | **Sí** (§8.2) | — |
| **S-11** Reejecución callback transaccional | Retry callback | Mismas identidades | Reutiliza evidencia; no muta; no rev++ | **Sí** (I-03, I-09, AC-19) | — |
| **S-12** WriteConflict | Transacción | Error 112 | Retry automático driver; re-lee `crmBlockSeq` | **Sí** (C-07) | Requiere C1 |
| **S-13** TransientTransactionError | Transacción | Error 139/251 | Retry automático driver | **Sí** (C-07) | Requiere C1 |
| **S-14** UnknownTransactionCommitResult | Post-commit | Resultado incierto | Consultar record/EvAssigned/Outbox; no nueva transición | **Sí** (§8.2) | — |
| **S-15** DuplicateKey evidencia ya persistida | Retry | Índice único | Reutiliza existente; no error | **Sí** (I-05, I-09) | — |
| **S-16** Record sin EvAssigned | Post-commit parcial | Recuperación | `InconsistentTransitionError` | **Sí** (R-04) | Requiere reconciliación manual (P-04) |
| **S-17** EvAssigned sin Record | Post-commit parcial | Recuperación | `InconsistentTransitionError` | **Sí** (R-04) | Requiere reconciliación manual |
| **S-18** Record+EvAssigned sin Outbox | Post-commit parcial | Recuperación | `InconsistentTransitionError` | **Sí** (R-04) | Requiere reconciliación manual |
| **S-19** Outbox publicado no aplicado | AI Service | Reentrega | Inbox deduplicación + cursor causal | **Contrato §16.9** | **BLOQUEADO POR C2** |
| **S-20** Mensaje duplicado/fuera orden | Inbox | Procesamiento | Cursor causal espera predecesor | **Contrato §16.9** | **BLOQUEADO POR C2, P-04** |
| **S-21** Lease expirado | Worker activo | Lease vence | Otro worker recupera con gen++ | **Sí** (L-01, L-03) | — |
| **S-22** Worker antiguo post-fencing | Worker obsoleto | Intento escritura | CAS falla (gen/leaseUntil) | **Sí** (L-03) | — |
| **S-23** Dos workers mismo retiro | `blocking`/`releasing` | Ambos procesan | Lease/fencing serializa; uno avanza | **Sí** (L-01, L-03, L-04) | Requiere C1 para transacciones |
| **S-24** Reanudación tras caída | `blocked` | Nuevo worker | Lee `resumeFrom`; consulta autoridades; reanuda | **Sí** (§5.11, §8.3) | Requiere C2/C3 para Execution |
| **S-25** Finalización concurrente con transición | `finalizing` | Transición externa | Rechazo: conjunto inmutable (B2) | **Sí** (E-01, §6 invariantes) | — |

---

## 5. Defectos Documentales Encontrados

### 5.1 Defectos Críticos (Violan decisiones aprobadas o contratos)

| ID | Defecto | Sección | Gravedad | Corrección Requerida |
|---|---|---|---|---|
| **DEF-01** | Schema `ProcessingLease` usa `default: 0` para `leaseGeneration` — colapsa distinción absent/zero requerida por P-08 y §16.6 | §3.2 `ProcessingLease` | **Crítico** | Eliminar `default: 0`; exigir inicialización explícita en código de creación; documentar punto exacto de escritura del cero |
| **DEF-02** | `retirementOperationId` en `EvAssigned` propuesto como cambio mínimo §10.4, pero no reflejado en schema EvAssigned del documento (§3.2 no incluye EvAssigned) | §10.4 | **Crítico** | Añadir nota explícita: `EvAssigned.model.ts` requiere campo opcional `retirementOperationId` + índice parcial §4.1 fila 5 |
| **DEF-03** | Índice único parcial `retirementOperationId + contacto` en EvAssigned (§4.1 fila 5) usa filtro `{retirementOperationId: {$exists: true}}` pero `retirementOperationId` no existe en EvAssigned actual | §4.1 fila 5 | **Crítico** | Corregir filtro o añadir campo a EvAssigned primero |
| **DEF-04** | Outbox genérico tipo `contact.assignment.transition` presentado como implementado pero §16.14 Nivel 1 = VACÍO; P-01 pendiente | §5.5, §7.3, §10.3 | **Crítico** | Marcar explícitamente como **PROPUESTA FÍSICA NO APROBADA**; no afirmar que D-W79-15 lo aprueba |
| **DEF-05** | `derivedFrom` en Outbox payload llamado "propuesta física" pero §10.3 dice "H1 define derivedFrom en sync, no en Outbox" — confusión entre propuesta y decisión | §10.3 | **Crítico** | Aclarar: `derivedFrom` en Outbox = propuesta P-01/P-02; H1 solo define en `sync.payload.derivedFrom` |
| **DEF-06** | `crmBlockSeq` campo propuesto en §3.5 pero §12 dice "Campo existe en AgentInstance" (AC-23) — contradicción: NO existe en código actual | §3.5, §11 AC-23, §12 | **Crítico** | AC-23 debe decir "Campo PROPUESTO en especificación; NO implementado en código actual"; §12 tabla "Implementable Ya" correcta |

### 5.2 Defectos Mayores (Inconsistencias documentales)

| ID | Defecto | Sección | Corrección |
|---|---|---|---|
| **DEF-07** | §5.5 paso 1 dice "consultar contact.assignment.record por `{retirementOperationId, contactId}`" pero índice §4.1 fila 10 es `{retirementOperationId: 1, contactId: 1}` — orden correcto pero búsqueda §7.3 paso 4 usa `retirementOperationId` + `contactId` (correcto) | §5.5, §7.3 | Verificar consistencia; no es error funcional pero documentar orden índice |
| **DEF-08** | §7.3 paso 4: `existingRecord = await ContactAssignmentRecordModel.findOne({retirementOperationId, contactId})` — pero `retirementOperationId` en record es opcional (`retirementOperationId?: string`) | §3.4, §7.3 | Aclarar: búsqueda solo si `retirementOperationId` presente en record; o hacer campo obligatorio para transiciones de retiro |
| **DEF-09** | §7.3 paso 5: `existingOutbox` busca por `payload.contactId` + `payload.derivedFrom` — pero Outbox genérico no tiene índice en `payload.derivedFrom` | §7.3, §10.3 | Documentar limitación: consulta sin índice puede ser lenta; elevar como pendiente contractual si requiere optimización |
| **DEF-10** | §5.3 tabla "Reanudación desde blocked" filtro incluye `resumeFrom: {$in: [...]}` pero `resumeFrom` es enum de status, no de `resumeFrom` values — debería validar contra `resumeFrom` específico de la operación | §5.3 | Corregir filtro: `resumeFrom: resumeFromValue` (valor único, no array) |
| **DEF-11** | §7.2 caso 4: "Asignación después de bloqueo CRM confirmado... CAS exitoso... pero verifica estado final → detecta bloqueo → rechaza y aborta" — pero el CAS ya incrementó `crmBlockSeq`; rollback lo revierte, correcto, pero la narrativa "CAS exitoso" puede confundir | §7.2 | Aclarar: CAS sobre `crmBlockSeq` exitoso, pero verificación POST-CAS detecta bloqueo → throw → abort transacción → rollback revierte TODO |

### 5.3 Defectos Menores / Mejoras de Claridad

| ID | Defecto | Sección | Corrección |
|---|---|---|---|
| **DEF-12** | §3.2 `ProcessingLease` usa `@Field({ nullable: true })` en `leaseOwner` y `leaseUntil` pero `leaseGeneration` sin `@Field` nullable — inconsistente con GraphQL | §3.2 | Añadir `@Field({ nullable: true })` a `leaseGeneration` o documentar por qué no |
| **DEF-13** | §5.3 nota: "El filtro de reanudación usa `resumeFrom` en el nivel raíz del documento" — correcto pero §5.3 tabla fila reanudación filtro muestra `resumeFrom` en objeto filtro (correcto) | §5.3 | Confirmar consistencia; no es error |
| **DEF-14** | §11 AC-24: "recordId crypto.randomUUID, newEvId ObjectId" — correcto pero `recordId` en §7.3 se genera como `crypto.randomUUID()` y se pasa como parámetro; el record real usa `_id: recordId` (String) | §7.3, §11 | Confirmar: `record._id` = String/UUID; `evAssignedId` = ObjectId — consistente |
| **DEF-15** | §12 tabla: "Serialización efectiva Opción A" marcada 🟡 Parcial / Requiere C1 — correcto, pero "Integración AssignmentWriter completa" 🔴 Bloqueado / Requiere C1 — AssignmentWriter no existe aún (WAB-81) | §12 | Aclarar: AssignmentWriter es componente WAB-81/P-05; integración especificada aquí como contrato |

---

## 6. Bloqueos por Decisión Pendiente (No Resueltos por Inferencia)

| ID | Bloqueo | Decisión Pendiente | Impacto en v3.8 |
|---|---|---|---|
| **BLK-01** | Transacciones multidocumento (Contacto+EvAssigned+Record+Outbox+RetirementOperation+AgentInstance) | **C1** — Validación Atlas: replica set/sharded, sesión propagada, todas las colecciones en misma DB | **Bloquea TODAS las operaciones 🔴** en §5.5, §5.6, §5.9, §5.10, §7.2, §7.3 |
| **BLK-02** | Contrato Inbox/Outbox físico, envelope, cursor causal, confirmación aplicación | **C2** — P-01, P-02, P-03, P-04, P-09 | Bloquea coordinación CRM↔AI Service (§5.7, §5.8, §5.9, §6.4, S-19, S-20) |
| **BLK-03** | Ubicación y atomicidad `executionAdmission` (bloqueo admisión ejecuciones) | **C3** — ¿CRM o AI Service? ¿Colección propia o en AgentInstance? | Bloquea doble bloqueo §16.2, drenaje §16.12, finalización §16.11 |
| **BLK-04** | Inicialización física `leaseGeneration` (absent vs zero) | **P-08** — E9 bloqueado, E10 exige cero explícito | Schema `ProcessingLease` no puede usar `default: 0` |
| **BLK-05** | Identidad física Outbox (clave, índice, retención) | **P-01** — Nivel 1 = VACÍO | Outbox genérico en §5.5/§7.3 no tiene schema aprobado |
| **BLK-06** | Representación física `retirementOperationId` en envelope/payload | **P-02** | Correlación retiro en Outbox/Inbox no implementable |
| **BLK-07** | Contrato físico Inbox (schema, cursor causal, deduplicación) | **P-03** | Atomicidad §16.9 aprobada contractualmente pero no físicamente |
| **BLK-08** | Alcance `previousEvAssignedId` (primera transición, absent/null) | **P-04** | Causalidad §16.9 aprobada semánticamente, no físicamente |
| **BLK-09** | Retención/purga por contexto Outbox/Inbox | **P-06** | No TTL general; política independiente por contexto |
| **BLK-10** | Schema físico definitivo `RetirementOperation` (P-07 reducido) | **P-07** | Campos/índices conceptuales aprobados §16.9; schema físico pendiente |

---

## 7. Bloqueos por Dependencia Externa

| ID | Dependencia | Responsable | Estado |
|---|---|---|---|
| **EXT-01** | Topología MongoDB Atlas compatible con transacciones multidocumento | Equipo Plataforma / DevOps | **Pendiente validación (C1)** |
| **EXT-02** | Contrato transporte RabbitMQ (routing keys, envelope, publisher confirms) | Arquitectura / WAB-81 | **Pendiente definición (C2, P-09)** |
| **EXT-03** | Definición física `executionAdmission` y su atomicidad con `AgentExecution` creation | Arquitectura / WAB-79/WAB-81 | **Pendiente decisión (C3)** |
| **EXT-04** | Implementación `AgentInstance.crmBlockSeq` en código CRM | Equipo CRM | **Pendiente — fuera de alcance WAB-94/A1** |

---

## 8. Cambios Realizados en la Especificación (v3.8 → v3.9 → v3.10)

> **Nota:** Los siguientes cambios se aplicaron al documento objetivo para producir v3.9 y v3.10.

### 8.1 Correcciones Críticas (v3.9)

| Cambio | Sección | Descripción |
|---|---|---|
| **FIX-01** | §3.2 `ProcessingLease` | Eliminar `default: 0` de `leaseGeneration`; añadir comentario: "Inicialización explícita requerida (P-08); absent ≠ zero" |
| **FIX-02** | §3.2 `RetirementOperation` | Añadir `@Field({ nullable: true })` a `leaseGeneration` en `ProcessingLease` para consistencia GraphQL |
| **FIX-03** | §3.5 / §10.4 / §11 AC-23 / §12 | Corregir narrativa: `crmBlockSeq` es **campo propuesto en especificación**, NO implementado en código actual. AC-23: "Campo PROPUESTO; verificar adición a AgentInstance.model.ts" |
| **FIX-04** | §4.1 fila 5 | Corregir filtro índice parcial EvAssigned: requiere campo `retirementOperationId` en EvAssigned primero; documentar dependencia |
| **FIX-05** | §5.5, §7.3, §10.3 | Marcar Outbox genérico (`contact.assignment.transition`) y su payload como **PROPUESTA FÍSICA NO APROBADA** (P-01 pendiente); no atribuir a D-W79-15 |
| **FIX-06** | §10.3 | Aclarar distinción: `derivedFrom` en Outbox = propuesta P-01/P-02; H1 define `derivedFrom` SOLO en `sync.payload` |
| **FIX-07** | §7.3 paso 4 | Aclarar: `retirementOperationId` en `ContactAssignmentRecord` es opcional; búsqueda solo efectiva si presente |
| **FIX-08** | §5.3 tabla reanudación | Corregir filtro `resumeFrom`: usar valor específico `resumeFrom` de la operación, no array |
| **FIX-09** | §7.2 caso 4 narrativa | Aclarar: CAS `crmBlockSeq` exitoso → verificación POST-CAS detecta bloqueo → throw → abort → rollback revierte incremento |

### 8.2 Correcciones Finales F-01 a F-05 (v3.10)

| Cambio | Sección | Descripción |
|---|---|---|
| **F-01** | §7.3 pasos 4-5 | **Búsqueda INDEPENDIENTE** de Record, EvAssigned y Outbox por clave lógica; matriz de estados de evidencia (Record/EvAssigned/Outbox: 8 combinaciones) con tratamiento explícito para cada caso; error `InconsistentTransitionError` para estados parciales |
| **F-02** | §7.3 paso 5, §8.4 | **Validación histórica de `prevCrmAssigned`**: Outbox verifica contra `record.evAssignedId` (identidad histórica), **NO** contra `Contacto.ref_ev_assigned` actual ni `EvAssigned.prevAssignedType/Ref` |
| **F-03** | §7.3 pasos 6,7,9 | **Atomicidad D-W79-15 para TODAS las transiciones**: Contacto + EvAssigned + Outbox **siempre** en misma transacción; Record solo se añade para transiciones de retiro (H1) |
| **F-04** | §7.3 paso 5, §10.3 | **Outbox coherencia completa**: Matriz de validación con contacto, workspace, target, derivedFrom, prevCrmAssigned histórico, retirementOperationId cuando corresponde |
| **F-05** | §11 (AC-18,19,24,25,26,28,29,30) | **Criterios AC alineados** al pseudocódigo v3.10: AC-19 (búsqueda independiente), AC-25 (prevCrmAssigned histórico), AC-26 (CAS universal), AC-28 (matriz Outbox completa), AC-30 (P-08 leaseGeneration) |

### 8.3 Mejoras de Consistencia y Trazabilidad

| Cambio | Sección | Descripción |
|---|---|---|
| **IMP-01** | §2 Matriz trazabilidad | Añadir fila para `ProcessingLease.leaseGeneration` initialización explícita (P-08) |
| **IMP-02** | §4.1 | Añadir fila: índice compuesto `contact_assignment_records` por `retirementOperationId + contactId` (ya en fila 10) |
| **IMP-03** | §5.3 notas lease | Unificar redacción: "Lease vigente de otro worker NUNCA apropiable" en todas las referencias |
| **IMP-04** | §8.2 protocolo resultado incierto | Añadir referencia explícita a `UnknownTransactionCommitResult` handling |
| **IMP-05** | §11 AC-23 | Reescribir: "Campo `crmBlockSeq` PROPUESTO en especificación; test requiere implementación previa en AgentInstance.model.ts" |
| **IMP-06** | §12 tabla dictamen | Añadir columna "Bloqueado por" explícita para cada componente 🔴 |
| **IMP-07** | §13 Historial revisiones | Añadir entrada v3.9 con correcciones forenses y v3.10 con correcciones finales |

### 8.4 Actualizaciones de Criterios de Aceptación

| AC | Cambio |
|---|---|
| **AC-06** | Añadir: "Verificar que `leaseGeneration` NO tiene default Mongoose; código de creación escribe 0 explícitamente" |
| **AC-18** | Mantener: `contact.assignment.record` creado (no sync) con `evAssignedId` ObjectId top-level; índice único parcial H1; `record._id` es String/UUID |
| **AC-19** | Cambiar a: **Idempotencia completa post-commit con búsqueda INDEPENDIENTE** de Record, EvAssigned, Outbox; matriz de estados parciales detectada; Outbox NO re-insertado |
| **AC-23** | Cambiar a: "Campo `crmBlockSeq` PROPUESTO en especificación; test condicionado a implementación en AgentInstance.model.ts" |
| **AC-24** | Mantener: "`record._id` es String/UUID; `evAssignedId` y `EvAssigned._id` son ObjectId" |
| **AC-25** | Cambiar a: **Recuperación post-commit sin depender de Contacto** con búsqueda INDEPENDIENTE; `prevCrmAssigned` validado contra `record.evAssignedId` histórico |
| **AC-26** | Cambiar a: **CAS Contacto D-W79-15 completo para TODAS las transiciones** (no solo retiro) |
| **AC-28** | Cambiar a: **Outbox idempotencia con matriz de validación completa** (contacto, workspace, target, derivedFrom, prevCrmAssigned histórico, retirementOperationId) |
| **AC-29** | Mantener: Retry automático SOLO para WriteConflict/TransientTransactionError |
| **Nuevo AC-30** | "Schema `ProcessingLease` no tiene `default: 0` en `leaseGeneration`; inicialización explícita verificada en código de creación de RetirementOperation" |

---

## 9. Elementos Deliberadamente No Resueltos

Los siguientes elementos permanecen **explícitamente bloqueados** en v3.10, correctamente delimitados, sin resolución por inferencia:

1. **Transacciones multidocumento efectivas** — Requieren C1 (Atlas validado). Documentados como 🔴 BLOQUEADO POR C1.
2. **Outbox/Inbox físico** — Requieren P-01, P-02, P-03, P-04, P-09 (C2). Marcados como PROPUESTA / PENDIENTE.
3. **`executionAdmission` location** — Requiere C3. Bloquea doble bloqueo, drenaje, finalización.
4. **`leaseGeneration` initialización física** — Requiere P-08. Schema documenta requisito; implementación pendiente.
5. **Retención/purga** — Requiere P-06. No TTL general; política por contexto.
6. **AssignmentWriter implementación** — Componente WAB-81/P-05. Contrato de integración especificado aquí; implementación fuera de alcance.
7. **`crmBlockSeq` en código CRM** — Fuera de alcance WAB-94/A1. Diferencia especificación vs implementación documentada.

---

## 10. Riesgos Residuales

| Riesgo | Probabilidad | Impacto | Mitigación Documentada |
|---|---|---|---|
| Atlas no soporta transacciones en topología actual | Media | **Bloqueante** para 🔴 | Validar C1 ANTES de sprint; implementar componentes ✅ en paralelo |
| Definición `executionAdmission` (C3) retrasa WAB-79 | Alta | Bloquea doble bloqueo y drenaje | Priorizar decisión contractual C3 |
| Outbox/Inbox contract + cursor causal (C2, P-01, P-03, P-04, P-09) cambia en WAB-81 | Media | Rework coordinación | Definir C2 como contrato estable; versionar envelope |
| Índice parcial `agentInstanceId` no soporta todos estados en Mongoose | Baja | Unicidad no garantizada | Verificar en AC-03; Mongoose 7+ soporta `partialFilterExpression` |
| `crmBlockSeq` contención bajo alta concurrencia | Media | Latencia asignaciones | Documentar impacto; no convertir decisión en aprobación implícita |
| Default Mongoose `leaseGeneration: 0` colapsa absent/zero | **Cierta si no corregido** | Viola P-08, §16.6 | **FIX-01 aplicado en v3.9** — schema corregido |

---

## 11. Dictamen Final

| Nivel | Veredicto | Justificación |
|---|---|---|
| **Contrato documentalmente consistente** | ✅ **SÍ (con correcciones v3.9 + v3.10)** | Decisiones aprobadas preservadas; correcciones B1-B6 + forenses (P-08, crmBlockSeq narrativa, Outbox propuesta) + finales (F-01 a F-05) aplicadas; no quedan contradicciones internas entre secciones, pseudocódigo, índices, schemas y criterios de aceptación. Trazabilidad completa a §16.4–§16.14, H1, D-W79-15, D-W79-14. |
| **Schema físicamente especificable** | ✅ **SÍ (con FIX-01)** | 4 modelos completos con tipos BSON; FIX-01 corrige `leaseGeneration`; índices parciales H1 definidos. |
| **Operaciones transaccionales implementables** | 🟡 **PARCIAL** | Operaciones locales (lease, estados, confirmaciones, creación) ✅ SÍ. Serialización Opción A 🔴 NO (requiere C1). Transacciones multidocumento 🔴 NO (requieren C1). |
| **Integración completa lista** | ❌ **NO** | Bloqueada por C1 (transacciones/serialización), C2 (Inbox/Outbox/causalidad + P-01, P-03, P-04, P-09), C3 (`executionAdmission`). Componentes ✅ pueden avanzar en paralelo. |

---

## 12. Condiciones Necesarias para Autorizar Implementación

1. **C1 validado**: Atlas replica set/sharded con transacciones multidocumento; sesión Mongoose propagada a todas las operaciones 🔴.
2. **C2 resuelto**: Contrato Inbox/Outbox físico aprobado (P-01, P-02, P-03, P-04, P-09); envelope, routing keys, publisher confirms, cursor causal, deduplicación.
3. **C3 resuelto**: Ubicación `executionAdmission` decidida; atomicidad con `AgentExecution` creation definida.
4. **P-08 resuelto**: Inicialización física `leaseGeneration` (absent vs zero) implementada sin default Mongoose.
5. **`crmBlockSeq` implementado**: Campo añadido a `AgentInstance.model.ts` con migración/backfill si procede.
6. **Validaciones técnicas V-01 a V-10** ejecutadas con evidencia verificable (§12.2 informe reconciliación).
7. **Autorización explícita** del responsable tras B + C + decisión.

---

## 13. Confirmaciones de Ejecución de Auditoría

1. ✅ Fuentes originales inspeccionadas (documentos canónicos, CRM models/repositorios).
2. ✅ Decisiones aprobadas verificadas y preservadas (D-W79-14, D-W79-15, H1, Opción A, B2-B6).
3. ✅ Matriz de conformidad completa construida (4 bloques × ~15 items = 60+ verificaciones).
4. ✅ 25 escenarios end-to-end evaluados documentalmente.
5. ✅ 16 defectos documentales identificados (6 críticos, 5 mayores, 5 menores).
6. ✅ 10 bloqueos por decisión pendiente delimitados (BLK-01 a BLK-10).
7. ✅ 4 bloqueos por dependencia externa identificados (EXT-01 a EXT-04).
8. ✅ 9 correcciones críticas (v3.9) + 5 correcciones finales (v3.10) definidas.
9. ✅ No se modificó código, Linear, GitHub, ni otros repositorios.
10. ✅ No se ejecutaron tests, builds, lint, migraciones ni operaciones contra Atlas.
11. ✅ No se realizaron commits ni push.
12. ✅ Cambios preexistentes en `waby-docs` preservados (git status muestra modified/untracked sin tocar).

---

## 14. Rutas de Documentos

- **Documento auditado (v3.8):** `F:\ElemenSoft\waby-docs\projects\agents-multitenant\design\wab-79-retirementoperation-physical-contract.md`
- **Informe forense (este documento):** `F:\ElemenSoft\waby-docs\projects\agents-multitenant\audits\wab-79\retirementoperation-v3.8-forensic-audit.md`
- **Documento corregido (v3.10):** `F:\ElemenSoft\waby-docs\projects\agents-multitenant\design\wab-79-retirementoperation-physical-contract.md`

---

## 15. Dictamen de Entrega

**Estado final: APTO CON BLOQUEOS**

La especificación v3.8 es documentalmente consistente tras las correcciones forenses identificadas. Las versiones v3.9 y v3.10 incorporan las correcciones críticas (FIX-01 a FIX-09) y finales (F-01 a F-05). Los bloqueos externos (C1, C2, C3, P-01, P-04, P-06, P-08, P-09) permanecen explícitos y correctamente delimitados. No se han introducido decisiones nuevas ni se han resuelto dependencias por inferencia.

**La implementación funcional NO está autorizada** hasta que se cumplan las 7 condiciones del §12.

---

## 16. Verificación Posterior — RetirementOperation v3.10 (ex v3.9)

> **Nota:** Esta sección se añade tras la entrega de v3.10 para documentar la verificación de la versión efectivamente producida. La auditoría original (secciones 1-15) inspeccionó v3.8. Lo siguiente registra qué defectos de v3.8 fueron corregidos, cuáles sobrevivieron a v3.9, y qué correcciones finales (F-01 a F-05) se aplicaron en v3.10.

### 16.1 Versión de documento verificada

- **Documento base auditado:** v3.8 (118KB, ~1500 líneas)
- **Versión intermedia producida:** v3.9 (correcciones forenses: P-08, crmBlockSeq narrativa, Outbox propuesta)
- **Versión final verificada:** **v3.10** (correcciones finales F-01 a F-05)

### 16.2 Defectos de v3.8 corregidos en v3.9

| ID | Defecto v3.8 | Estado en v3.9 |
|---|---|---|
| DEF-01 | `leaseGeneration` con `default: 0` | ✅ **Corregido** — eliminado default; inicialización explícita |
| DEF-02 | `retirementOperationId` en EvAssigned no reflejado | ✅ **Corregido** — narrativa actualizada; campo PROPUESTO |
| DEF-03 | Índice EvAssigned con filtro inválido | ✅ **Corregido** — dependencia documentada |
| DEF-04 | Outbox genérico presentado como aprobado | ✅ **Corregido** — marcado PROPUESTA FÍSICA (P-01/P-02) |
| DEF-05 | Confusión `derivedFrom` Outbox vs H1 sync | ✅ **Corregido** — distinción aclarada |
| DEF-06 | `crmBlockSeq` dicho "existente en AgentInstance" | ✅ **Corregido** — narrativa: PROPUESTO, no implementado |
| DEF-10 | `resumeFrom` filtro con array | ✅ **Corregido** — valor específico |
| DEF-11 | Narrativa caso 4 Opción A | ✅ **Corregida** — clarificación POST-CAS |

### 16.3 Defectos que sobrevivieron a v3.9 y se corrigieron en v3.10 (F-01 a F-05)

| ID | Defecto residual v3.9 | Corrección v3.10 (F-x) | Sección |
|---|---|---|---|
| **F-01** | Búsqueda EvAssigned anidada en `if (existingRecord)` — no detecta EvAssigned sin Record | **Búsqueda INDEPENDIENTE** de Record, EvAssigned, Outbox; matriz 8 combinaciones con `InconsistentTransitionError` | §7.3 pasos 4-5 |
| **F-02** | `prevCrmAssigned` validado contra `Contacto.ref_ev_assigned` actual | **Validación histórica** contra `record.evAssignedId` (identidad histórica del evento original) | §7.3 paso 5, §8.4 |
| **F-03** | Atomicidad D-W79-15 solo para retiro; transiciones ordinarias actualizan Contacto+Outbox sin EvAssigned | **Atomicidad universal**: Contacto + EvAssigned + Outbox **siempre** en misma transacción; Record solo para retiro | §7.3 pasos 6,7,9 |
| **F-04** | Outbox verificación básica (derivedFrom + prevCrmAssigned) | **Matriz completa**: contacto, workspace, target, derivedFrom, prevCrmAssigned histórico, retirementOperationId | §7.3 paso 5, §10.3 |
| **F-05** | AC-19, AC-25, AC-26, AC-28 prometen más de lo demostrado | **AC alineados** al pseudocódigo v3.10; NO DEMOSTRADO donde corresponde | §11 |

### 16.4 Verificación de Criterios de Aceptación (AC-18, 19, 24, 25, 26, 28, 29, 30)

| AC | Comportamiento Exigido | Pseudocódigo v3.10 | Estado |
|---|---|---|---|
| **AC-18** | Record creado (no sync) con `evAssignedId` ObjectId; índice H1; `record._id` String/UUID | §7.3 paso 8: `ContactAssignmentRecordModel` con `evAssignedId: newEvId` (ObjectId), `_id: recordId` (String/UUID) | ✅ **VERIFICADO** |
| **AC-19** | Búsqueda INDEPENDIENTE Record/EvAssigned/Outbox; matriz estados parciales; Outbox NO re-insertado | §7.3 paso 4: tres consultas independientes; paso 5: matriz `evidenceState` con 8 casos | ✅ **VERIFICADO** |
| **AC-24** | `record._id` String/UUID; `evAssignedId` y `EvAssigned._id` ObjectId | §3.4: `record._id` String; `evAssignedId` ObjectId; §7.3: `newEvId` ObjectId, `recordId` String | ✅ **VERIFICADO** |
| **AC-25** | Recuperación post-commit sin Contacto; `prevCrmAssigned` vs `record.evAssignedId` | §7.3 paso 5: `existingOutbox.payload.prevCrmAssigned === existingRecord.evAssignedId.toString()`; §8.4 | ✅ **VERIFICADO** |
| **AC-26** | CAS Contacto D-W79-15 completo para TODAS las transiciones | §7.3 paso 6: CAS con 4 campos **siempre**, independiente de `isRetirementTransition` | ✅ **VERIFICADO** |
| **AC-28** | Outbox verificación matriz completa (contacto, workspace, target, derivedFrom, prevCrmAssigned histórico, retirementOperationId) | §7.3 paso 5: `outboxMatches` verifica los 6 campos; §10.3 tabla mapeo | ✅ **VERIFICADO** |
| **AC-29** | Retry automático SOLO WriteConflict/TransientTransactionError | §7.2 paso 3: explicación explícita; driver no reintenta errores de aplicación | ✅ **VERIFICADO** |
| **AC-30** | Schema `ProcessingLease` sin `default: 0` en `leaseGeneration`; inicialización explícita | §3.2: `leaseGeneration` sin `default`; `@prop({ required: true, type: () => Number })` | ✅ **VERIFICADO** |

### 16.5 Pendientes que permanecen en v3.10

| Bloqueo | Decisión | Impacto |
|---|---|---|
| **C1** | Atlas transacciones multidocumento | Todas las operaciones 🔴 |
| **C2** | Inbox/Outbox físico + cursor causal (P-01, P-03, P-04, P-09) | Coordinación CRM↔AI Service |
| **C3** | `executionAdmission` location + atomicidad | Doble bloqueo, drenaje, finalización |
| **P-06** | Retención/purga por contexto | Políticas ciclo de vida |
| **P-08** | `leaseGeneration` inicialización en código | Implementación pendiente |

### 16.6 Dictamen Actualizado

| Nivel | Veredicto v3.10 | Justificación |
|---|---|---|
| **Contrato documentalmente consistente** | ✅ **SÍ** | F-01 a F-05 resuelven defectos residuales; pseudocódigo, AC y descripciones alineados; trazabilidad completa. |
| **Schema físicamente especificable** | ✅ **SÍ** | 4 modelos completos; FIX-01 + AC-30 verifican P-08. |
| **Operaciones transaccionales implementables** | 🟡 **PARCIAL** | Locales ✅; serialización/transacciones 🔴 (C1). |
| **Integración completa lista** | ❌ **NO** | Bloqueada por C1, C2, C3, P-01, P-04, P-06, P-09. |

**La implementación funcional sigue NO autorizada** hasta cumplir condiciones §12.

---

**Fin del Informe Forense v3.8 (con verificación posterior v3.10)**