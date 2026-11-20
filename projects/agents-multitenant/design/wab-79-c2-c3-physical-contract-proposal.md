# WAB-79 — Propuesta de cierre físico C2/C3

**Estado:** propuesta técnica para reconciliación; no sustituye decisiones canónicas ni autoriza por sí sola despliegues/migraciones.
**Fecha:** 19/11/2026
**Ámbito:** contratos CRM ↔ AI Service para RetirementOperation y admisión/drenaje de AgentExecution.

## 1. Invariantes

1. CRM es autoridad de AgentInstance, Contacto, EvAssigned, RetirementOperation y assignment history.
2. AI Service es autoridad de AgentExecution, AgentExecutionInbox, ExecutionAdmission y resultados de ejecución.
3. Los eventos de coordinación no transportan historial EvAssigned; cada transición sólo proyecta su identidad `evAssignedId` y metadata necesaria.
4. At-least-once delivery; deduplicación durable antes de ACK; los efectos de negocio y el Inbox processed state deben confirmarse atómicamente en la DB receptora.
5. No se usa ObjectId como cursor causal ni se declara drain por comparación de IDs.
6. No se realizan llamadas de red dentro de transacciones MongoDB.
7. Todo rechazo por instancia/Workspace/operación mismatched es fail-closed.

## 2. Envelope propuesto (P-09)

```ts
interface RetirementMessageEnvelope<T> {
  schemaVersion: 1;
  messageId: string; // UUID estable por intención lógica
  messageType: 'agent.assignment.transition' | 'retirement.execution-drain.confirmed' | 'retirement.finalize' | 'retirement.finalization.confirmed';
  occurredAt: string; // ISO-8601 UTC
  source: 'crm' | 'ai-service';
  workspaceId: string;
  agentInstanceId: string;
  retirementOperationId?: string;
  evAssignedId?: string; // metadata de transición, no historial
  eventSetRevision?: number; // secuencia CRM monotónica por operación
  payload: T;
}
```

`messageId` se persiste y reutiliza en cada retry. Los consumidores validan `schemaVersion`, origen, tenant, AgentInstance y correlación de operación antes de cualquier efecto.

## 3. Outbox (P-01/P-02)

Cada intención durable se inserta en la misma transacción que el cambio fuente. El Outbox conserva el modelo genérico existente, pero añade/estandariza campos físicos para la identidad contractual: `messageId`, `type`, `payload`, `status`, `attempts`, `createdAt`, `sentAt`, `lastError`. Índice único sobre `messageId`; `type` permanece como routing intent. `retirementOperationId` se incluye en envelope/payload para correlación; no es la clave única global porque una operación emite múltiples mensajes.

El publisher debe usar publisher confirms, marcar `sent` únicamente tras confirmación del broker y reintentar con backoff. Mensajes inválidos o permanentemente rechazados se mueven a DLQ con alerta y no se descartan silenciosamente.

## 4. Inbox durable (P-03)

Colección AI Service `agent_execution_inbox` (mensajería de ejecución) y CRM Inbox para mensajes de coordinación RetirementOperation, separadas por autoridad/DB. Campos mínimos: `messageId` único, `messageType`, `schemaVersion`, `workspaceId`, `agentInstanceId`, `retirementOperationId` nullable, `receivedAt`, `status` (`received|processing|processed|retry|dead_letter`), `attempts`, `processedAt`, `lastError`, lease/fencing para workers. El mensaje original se conserva de forma limitada según política de retención pendiente P-06.

Recepción: persistir durablemente antes ACK Rabbit. Procesamiento: transacción local incluye dedupe/claim, mutación de proyección/entidad y estado processed + Outbox de respuesta. Si falla, rollback y retry; no ACK como processed.

## 5. Causalidad y bootstrap de `contact_assignments` (P-04 reconciliado)

El contrato canónico WAB-79 §16.9 ya define causalidad por contacto, no una secuencia global: `evAssignedId` identifica la transición, `previousEvAssignedId` identifica el predecesor esperado y `lastProcessedEvAssignedId` conserva el último evento aplicado. No se introducirá `assignmentProjectionSequence`, ni se reutilizarán `eventSetRevision` o `crmBlockSeq` como cursor de proyección.

Envelope de transición incluye `evAssignedId`, `previousEvAssignedId` (nullable explícito), `contactId`, estado resultante (`assignedType`, `assigneeId`, `status`) y metadata contractual. `eventSetRevision` puede acompañar eventos de retiro para el cierre de conjunto, pero no ordena el stream global de asignaciones.

### Aplicación causal por contacto

- Si `lastProcessedEvAssignedId === previousEvAssignedId`, aplicar transición y avanzar a `evAssignedId`.
- Si `lastProcessedEvAssignedId === evAssignedId`, transición ya aplicada: idempotencia.
- Si el evento es futuro (predecesor no coincide), persistir como deferred/retry; no aplicarlo ni ACK como processed.
- Si CRM/proyección ya reflejan `evAssignedId`, reconocerlo como aplicado sin mutar estado de nuevo.
- Si el predecesor no puede reconciliarse, solicitar snapshot autoritativo de ese contacto; nunca ordenar por ObjectId o timestamp.

### Bootstrap inicial

La proyección target no migra ni traduce filas legacy, según D-W79-04. Se construye desde snapshot autoritativo CRM de contactos actualmente asignados a AgentInstance:

1. CRM entrega `contactId`, `workspaceId`, `assignedType=agent`, `assigneeId=agentInstanceId` y `ref_ev_assigned` (evento vigente), más estado de snapshot consistente.
2. AI Service marca el AgentInstance projection como `initializing`; los eventos recibidos se persisten en Inbox y quedan deferred.
3. Snapshot se aplica con upsert condicional sólo a contactos sin estado inicializado; cada fila activa queda con `evAssignedId=ref_ev_assigned` y `lastProcessedEvAssignedId=ref_ev_assigned`.
4. Se registra bootstrap completado durablemente; no se borran ni sobreescriben transiciones procesadas concurrentemente.
5. Se libera el procesamiento causal del Inbox. Eventos ya incluidos en snapshot se reconocen por identidad; eventos posteriores aplican por predecessor.
6. Contactos que no aparecen en snapshot no se materializan como `unassigned`; ausencia de asignación activa se representa por ausencia de fila activa. Si existe fila local activa no incluida, se reconcilia contra CRM y se marca released sin inventar evento.

Snapshot/Bootstrap requiere consistencia de lectura CRM y protección contra carrera con eventos. Preferir snapshot de lectura consistente; si no se puede garantizar, bootstrap debe usar per-contact reconciliation y conservar eventos deferred hasta que cada contacto se reconcilie. No se necesita cursor global.

### Recuperación

- Inbox recibido/deferred permanece durable durante reinicios.
- Procesamiento retoma por contacto a partir de `lastProcessedEvAssignedId`.
- Huecos causales consultan CRM para obtener estado/evento predecesor faltante; no descartan eventos futuros.
- Rebuild explícito reemplaza proyección mediante nuevo snapshot versionado; no borra Inbox sin resolver ni confirmaciones.
- Retención del Inbox y snapshots queda pendiente P-06; nunca purgar mensajes causales aún no aplicados/confirmados.

## 6. C3 — ExecutionAdmission en AI Service DB

La autoridad de AgentExecution y su admisión reside en AI Service DB. Colección agent_instance_execution_controls, clave única agentInstanceId, campos workspaceId, admissionState (open|blocked), retirementOperationId, admissionGeneration, admissionSequence, blockedAt y updatedAt. El control debe provisionarse cuando AgentInstance pasa a estado ejecutable; falta de control es fail-closed.

**Admitir:** transacción AI Service condiciona el control open e incrementa admissionSequence (escritura sobre el documento guardián), valida Workspace y crea AgentExecution durable pending con identidad estable y evAssignedId obligatorio. Commit conjunto. Si blocked, no crear ejecución. La escritura guardián hace competir admisión y bloqueo bajo conflicto Mongo, evitando snapshot write skew.

**Bloquear:** transacción condiciona open, incrementa admissionGeneration y establece blocked + retirementOperationId; si blocked por la misma operación, idempotente; otra operación, conflicto. Ambos caminos escriben el control.

**Drenar:** consultar AgentExecution autoritativas pending|processing asociadas a instancia; no usar contadores ni Inbox-only state. Recuperar ejecuciones abandonadas y alcanzar terminal. Confirmación drain sólo si control blocked por misma operación y cero no terminales.

**Finalizar:** retirement.finalize acepta misma operación/generation; transacción Inbox processed + terminal confirmation Outbox. Control permanece blocked; no reapertura automática.

## 7. Compatibilidad con runtime

No crear segundo ExecutionEngine. Adaptar motor genérico existente para AgentDefinition/AgentInstance y AgentExecution. Entidades legacy Bot/Execution no se reinterpretan silenciosamente; compatibilidad histórica se mantiene por lectura/migración separada.

## 8. Requisitos de prueba (ronda consolidada al finalizar)

- Carrera admit vs block: exactamente una transacción gana; no execution accepted after block.
- Duplicados Outbox/Inbox: un efecto.
- Publisher crash pre/post confirm: retry sin pérdida/duplicación de efecto.
- Inbox crash pre/post commit: recovery.
- Hueco causal: futuro deferred y luego orden correcto.
- Confirmation duplicada/mismatched operation/generation: idempotente/rechazo.
- Worker stale fencing.
- Drain con pending/processing y recovery.
- Finalization no reabre admission.

## 9. Pendientes físicos

- P-06 retención/purga, retry bounds, DLQ.
- Confirmar si Outbox/Inbox CRM existentes se extienden o bounded-context collections.
- SchemaVersion policy y snapshot consistency API CRM.
- Verificar AgentExecution target/runtime WAB-81 antes de cableado.

## 10. Evidencia de revisión CRM (19/11/2026)

- EvAssigned tiene retirementOperationId, sin sequence global.
- ContactAssignmentRecord tiene evAssignedId, retirementOperationId, derivedFromRecordId, sin sequence.
- AgentInstance.crmBlockSeq serializa bloqueo CRM/asignaciones entrantes; no es cursor de proyección.
- CRM Outbox genérico no tiene identidad contractual ni publisher confirm.
- Proyección AI legacy usa vocabulario user|bot; contrato target reemplaza sin migración legacy.

No reutilizar sequence global. Cursor target se deriva por contacto del chain de EvAssigned.

## 11. Implementación local incremental (sin pruebas intermedias)

### AI Service

- Modelos AgentExecution (estados D-W79-05), AgentExecutionInbox, ExecutionResultOutbox y AgentInstanceExecutionControl.
- MongoAgentExecutionAdmissionRepository: admit incrementa admissionSequence y crea AgentExecution.pending en transacción; block CAS open→blocked incrementa admissionGeneration sobre el mismo control; missing control fail-closed.
- retirement_coordination_inbox/outbox: messageId, envelope, state, lease/fencing.
- Inbox persistence-before-ACK, transactional local effect + processed + response Outbox; retry.
- ConfirmChannel AI Service, confirmed publisher, consumer re-registration, durable coordination queue.
- DurableMessageOutboxPublisher y AgentAssignmentInboxWorker.
- Target contact_assignments projection y causal apply por previousEvAssignedId; bootstrap per-contact snapshot seed.

### CRM

- Outbox messageId opcional + unique sparse.
- AgentInstanceAssignmentWriterService: Contact CAS, EvAssigned, ContactAssignmentRecord, retirementOperationId/eventSetRevision cuando sale de AgentInstance en releasing, Outbox envelope; transaction conjunta.
- Publisher ConfirmChannel para coordinación; Outbox handler de assignment/retirement messages.
- RetirementMessageInbox durable y consumer de confirmations; assignment applied confirmation, execution drain confirmation y terminal finalization con validaciones.
- RetirementOperationRepository: findResumable y closeEventSet CAS/transaction.
- RetirementOperationWorker: releaseAssignments dinámico por batches y completeRetirement (drain, confirmation set, Inbox pending, finalization request).
- RetirementExecutionAdmissionAdapter: durable Outbox request + espera Inbox confirmation.
- Runner local integrado al arranque CRM.

### No cerrado todavía

- Provisioning del execution-control al lifecycle AgentInstance; sin control, admission/block falla cerrado.
- Snapshot CRM API consistente y orquestación formal del bootstrap.
- AgentExecution/AgentExecutionInbox/ExecutionResultOutbox aún no conectados al runtime genérico; lifecycle pending→terminal, startedAt y recuperación durable siguen pendientes.
- AssignmentWriter aún requiere reconciliación de contratos físicos y cableado con todos los paths de asignación; no sustituye todavía todos los writers legacy.
- Confirmation consumer, adapter y worker deben validarse contra el contrato físico v3.10 y el schema actual.
- Retry bounds/DLQ/retention P-06 y publisher confirmations CRM↔AI Service requieren validación final.
- No se ejecutaron pruebas; quedan reservadas para el cierre consolidado solicitado.

Estos cambios son código local no desplegado y no constituyen criterios Done.

## 12. Reconciliación D-W79-05 (corrección del incremento local)

La implementación target de AgentExecution respeta el estado lógico: pending/completed/failed/cancelled. No se utiliza processing en AgentExecution; recepción, claim, lease, retries y recuperación pertenecen a AgentExecutionInbox. AgentExecutionInbox y ExecutionResultOutbox son colecciones separadas de retirement_coordination_inbox/outbox. AgentExecution conserva createdAt/startedAt/finishedAt, channel/input, evAssignedId obligatorio, métricas/result y effectiveConfigSnapshot.

El control de admisión cuenta AgentExecution pending y AgentExecutionInbox no terminales (received/processing/retry) para el drain. Runtime todavía debe cablear el mismo ExecutionEngine genérico, completar AgentExecution y publicar ExecutionResultOutbox. No se ha implementado ese wiring.


## 13. Fencing de assignment y finalización

AssignmentWriter requiere lease token vigente del RetirementOperation cuando sale de AgentInstance en releasing; el CAS de eventSetRevision incluye owner, generation y leaseUntil. El worker transmite el token. Transiciones durante blocking/finalizing/blocked se rechazan. El request de finalización usa messageId por lease generation; CRM valida owner/generation/leaseUntil. Confirmación stale se consume como processed-stale sin finalizar ni mantener Inbox pendiente.


## 14. Revisión física incremental — 19/11/2026

### Correcciones locales no validadas

- AI Service `MongoAgentContactAssignmentProjectionRepository.bootstrapContact`: se elimina el upsert condicionado por `bootstrapVersion < snapshotVersion`, que no coincide con documentos sin el campo. Bootstrap ahora usa lectura, CAS para proyección activa no inicializada, inserción y reconciliación de duplicate-key; nunca sobrescribe un cursor causal ya procesado ni filas released.
- AI Service `MongoDurableMessageRepository`: persistInbox detecta messageId reutilizado con identidad contractual distinta; Inbox/Outbox aplica máximo local de 10 intentos y estado dead_letter.
- CRM `RetirementMessageInbox` incorpora lease/fencing fields, processing/dead_letter y attempts. Consumer coordination intenta claim durable, valida identidad al redelivery, confirma processed dentro de la transacción, y clasifica stale finalization cuando la operación ya no está finalizing.

### Limitaciones que siguen abiertas

- El bootstrap todavía carece de un snapshot provider CRM consistente y de control durable por AgentInstance para inicialización global; la corrección per-contact no cierra la carrera snapshot/eventos.
- No se ejecutaron tests, build, typecheck ni validación de runtime.
- Runtime AgentExecution sigue sin wiring: MessageReceivedHandler/BotRuntimeResolver/ExecutionContext/ExecutionEngine continúan Bot-oriented y legacy Execution persistence. No simular Bot identity ni crear engine paralelo.
- AgentExecution/ExecutionAdmission provisioning y worker de execution inbox, snapshot al aceptar, ejecución del pipeline genérico y lifecycle terminal siguen pendientes.
- AssignmentWriter aún no sustituye todos los caminos de asignación.
- Estos cambios locales no constituyen cierre de C2/C3/P-01/P-03/P-04/P-06/P-09 ni aceptación WAB-95/WAB-81.


### AssignmentWriter — integración CRM incremental

- `ContactoService.updateContactoByID` ahora deriva cambios reales de `ref_user_assigned` a `AgentInstanceAssignmentWriterService`; elimina de ese método el sync HTTP legacy de assignment. El update genérico excluye `ref_user_assigned`, `assignedType`, `ref_agent_assigned` y `ref_ev_assigned` para evitar writes paralelos.
- Mutation `assignUserToContact` se enruta al mismo writer.
- Nueva mutation `assignContactToAgentInstance(contactId, agentInstanceId|null)`: asignación/reasignación/liberación de AgentInstance, ownership de Contact/Workspace y estado activo del target validados; actor y operationType registrados.
- Permisos ARQ-20 añadidos a enum (`wrk.agents.read`, `wrk.agents.*`) y Manager default (`wrk.agents.*`).
- Resolver legacy `AiContactAssignment.resolver.ts` renombrado `AiContactAssignment.legacy.ts` para excluirlo del loader dinámico `.resolver.ts`; su modelo/servicio aún requieren revisión de referencias y limpieza.
- El AssignmentWriter valida target humano dentro del Workspace y deriva assignment type legacy desde refs cuando `assignedType` falta; CAS incorpora refs de assignee.

No se ejecutaron pruebas ni compilación. Requiere revisión consolidada de paths secundarios de asignación, compatibilidad de permisos y GraphQL antes de aceptar.


### ExecutionAdmission — revisión incremental

- `MongoAgentExecutionAdmissionRepository.countNonTerminal` ahora cuenta AgentExecution `pending|processing`, no sólo `pending`, además de Inbox `received|processing|retry`. Esto evita que el drain confirme mientras una ejecución ya pasó a processing.
- Sigue faltando el provisioning del control OPEN desde el lifecycle de AgentInstance y el runtime que actualiza AgentExecution terminal + resultado Outbox e Inbox processed en la misma unidad durable.


### CRM Outbox retry fallback

- `outboxRetry.worker` poller was extended to process `agent.assignment.transition` and `retirement.*` pending events as fallback to the change stream, with the existing exponential backoff and max-attempt terminal FAILED behavior. Publisher still marks SENT only after ConfirmChannel callback/drain. This is local unvalidated code; broker DLQ configuration and operational alerting remain open.


### CRM Outbox change stream coverage

- `Outbox.stream.ts` now matches `agent.assignment.transition` and the explicit coordination request types `retirement.execution-admission.block`, `retirement.execution-drain.requested`, and `retirement.finalize`, in addition to legacy workspace-delete/AssignmentSync. The retry poller is the fallback when the change stream misses events.


### Bootstrap/recovery — scaffolding durable

- Se creó control de bootstrap por Workspace `agent_assignment_projection_bootstrap_controls`, snapshotVersion/status/cursor y CAS de páginas.
- Repository bootstrap permite inicializar proyecciones active/released con cursor `refEvAssignedId`, evitando snapshot que omita el último evento de contactos liberados.
- Se creó `AgentAssignmentProjectionBootstrapService` con provider port para snapshot consistente CRM y paginación idempotente. El adapter CRM/provider no está implementado.
- Assignment Inbox ahora tiene estado durable `deferred`: mientras bootstrap no esté completed o un predecessor cursor no esté disponible, no consume intentos y no aplica el efecto. Reconcile conflicts sí siguen retry bounded/dead_letter.
- El worker fail-closed gatea assignment transitions por control completed. Falta proveer snapshot consistente y ejecutar bootstrap lifecycle; sin eso los eventos quedan deferred y no hay proyección activa.


### AgentExecution lifecycle/snapshot

- AgentExecution status model now includes `processing`; repository start CAS pending→processing and terminal completion/failure/cancellation uses non-terminal guards.
- Admission input requires `effectiveConfigSnapshot`; transaction persists it before returning admitted and duplicate-origin validates the same snapshot.
- Drain count includes pending+processing executions plus durable Inbox nonterminal states.
- Snapshot resolution and runtime wiring remain missing; this is a contract scaffold, not an executable AgentInstance runtime.


### Execution result Outbox

- `MongoAgentExecutionRepository` now persists terminal AgentExecution and `ExecutionResultOutbox` in one Mongo transaction with stable result messageId, preserving admission snapshot.
- Added publisher-confirm result Outbox worker and startup/shutdown timer wiring.
- Still no runtime adapter calling markStarted/complete/fail, nor Inbox+terminal+resultOutbox atomic effect for message processing. Therefore result lifecycle is scaffold only, not closed C3.


### Execution Inbox worker scaffolding

- AgentExecution Inbox repository adds pending scan, fenced lease, bounded retries/dead_letter.
- `AgentExecutionInboxWorker` accepts an `AgentExecutionRuntime` port, starts pending execution, and routes terminal result through atomic Inbox+AgentExecution+ResultOutbox transaction.
- No concrete runtime adapter is wired to the existing generic ExecutionEngine; worker is scaffolding and must not be activated without AgentInstance resolver/config snapshot adapter.
- Drain includes Inbox dead_letter as unresolved and therefore fail-closed.

## 15. Reconciliación de lifecycle AgentExecution — 19/11/2026

Se corrige el registro incremental anterior que había agregado `processing` a AgentExecution. D-W79-05 mantiene estados lógicos `pending|completed|failed|cancelled`; `processing`, claim, lease, attempts y recuperación pertenecen exclusivamente a AgentExecutionInbox.

- `AgentExecutionRepository.markStarted` sólo registra `startedAt` manteniendo el estado lógico `pending`.
- Completion/failure/cancellation condicionan terminalización a `status=pending`.
- `countNonTerminal` verifica únicamente AgentExecution `pending`, bajo control admission bloqueado por la misma RetirementOperation. No suma Inbox, porque admisión persiste AgentExecution + Inbox en una transacción y sumar ambos duplicaría cada ejecución aceptada.
- Dead-letter de una solicitud no puede dejar AgentExecution pending indefinidamente: el worker debe terminalizarla como failed mediante el mismo efecto durable; cualquier dead-letter sin ejecución debe tratarse como inconsistencia y bloquear/reconciliar, no confundirse con una ejecución de dominio.

Esto corrige el código local no validado y la descripción de drain; no cambia D-W79-05 ni autoriza tests intermedios.

## 16. Estado WAB-81 — runtime target pendiente de contrato físico

Linear WAB-81 describe el runtime AgentInstance-aware, pero el código local aún no define un envelope de ejecución CRM→AI Service ni la forma contractual de `effectiveConfigSnapshot`. El producer CRM de `agent.execution.requested`, consumer/admission ingress y concrete runtime adapter no existen conectados. El resolver Bot actual y fallback sintético son incompatibles con el target y no deben reutilizarse como fallback.

La implementación target requiere reconciliar en WAB-81 el snapshot sin secretos (identidad definition/instance/workspace, configuración efectiva controlada por Waby, prompt base + instrucciones de instancia, ruta provider/model, límites, capabilities autorizadas, Knowledge/Rules/Skills y hash/versiones), y el endpoint/adapter de autoridad que lo resuelve. Hasta definirlo, runtime target debe permanecer fail-closed.

## 17. Corrección de drain y bloqueo bootstrap — revisión actual

La implementación local reconciliada con D-W79-05 cuenta únicamente `AgentExecution.status=pending` para drain; Inbox procesa leases/retries y no se suma a AgentExecution, ya que admission crea ambos registros en una transacción. Un Inbox dead-letter sin AgentExecution asociado es inconsistencia que requiere reconciliación, no debe tratarse como estado lógico de ejecución.

**Bloqueo operacional de bootstrap:** `AgentAssignmentInboxWorker` fail-closed gatea `agent.assignment.transition` hasta que el bootstrap por Workspace esté `completed`. El servicio y repository son scaffolding, pero no existe snapshot provider CRM ni endpoint autenticado con snapshot consistente. Por lo tanto, las transiciones nuevas pueden quedar `deferred` y la confirmación `retirement.assignment.applied` no se emite; RetirementOperation no puede completar el conjunto de confirmaciones. No retirar el gate ni inventar un cursor/snapshot: implementar el contrato de snapshot con autoridad CRM, consistencia y autenticación acordadas.

**C3 lifecycle:** el control `agent_instance_execution_controls` fail-closed si falta, pero su provisioning OPEN no está conectado al lifecycle CRM de AgentInstance. Sin producer/contract de provisioning, una instancia activa no puede admitir ejecución target.

Estas limitaciones son blockers de integración y acceptance, no se resuelven con pruebas ni fallback legacy.

## 18. Lease fencing — revisión local adicional (19/11)

- AI Service retirement Inbox, execution Inbox, durable coordination Outbox e execution result Outbox: terminal/retry updates ahora condicionan status + leaseOwner + leaseGeneration + `leaseUntil > now`; execution Inbox worker renueva lease y entrega AbortSignal al runtime port.
- CRM retirement coordination consumer: commit procesado y retry requiere lease vigente además de owner/generation. La rama stale-finalization ya no marca processed anticipadamente; deja la clasificación `lastError` y permite el update común fenced.
- Drain confirmation CRM aplica CAS correlacionado por Workspace/AgentInstance/operation/status/lock confirmation/admissionBlockedAt, y reconoce duplicate confirmation sólo si la evidencia durable coincide.
- ExecutionAdmission duplicate snapshot comparación usa JSON canónico para evitar conflicto por orden de propiedades; AgentExecution startedAt es idempotente y no se reescribe en retries.

Cambios locales no validados. No se ejecutaron tests/build/typecheck.

## 19. crmBlockSeq serializa transiciones salientes — revisión adicional

`AgentInstanceAssignmentWriterService` ahora incrementa el guard `crmBlockSeq` de la AgentInstance de origen cuando una transición sale de una asignación Agent. El mismo documento guardián que utiliza `RetirementCrmLockService.blockAssignments` fuerza conflicto/retry entre lock CRM y assignment outgoing. La salida autorizada del worker sólo se permite cuando la operación está `releasing` y el token operationId/workerId/generation/leaseUntil es vigente. Asignaciones entrantes siguen usando el guard de destino.

Cambio local no validado; no modifica el contrato Opción A.

## 20. Inbox messageId reuse / response validation

AI Service and CRM coordination Inbox duplicate detection now compare causal identifiers (`evAssignedId`, `previousEvAssignedId`, `eventSetRevision`) and canonical payload in addition to messageType/schema/source/workspace/instance/operation. CRM response consumer validates schemaVersion, source `ai-service`, type, workspace/operation/instance and occurredAt before durable claim. Local changes unvalidated.

## 21. Cierre del eventSetRevision

CRM finalization ahora verifica `expectedEvAssignedEvents.length === eventSetRevision` antes de comprobar confirmations 1:1, tanto en RetirementOperation worker como en el consumer finalization transaction. Bajo el contrato, cada salida de assignment en `releasing` incrementa revision y crea EvAssigned con operationId en la misma transacción. Cambio local no validado.

## 22. Confirmación causal de AssignmentWriter

CRM `retirement.assignment.applied` ahora exige `eventSetRevision` entero positivo y valida que RetirementOperation coincida con workspace/AgentInstance y que su revisión durable sea >= la revisión del evento, además de EvAssigned con operationId. Confirmation se persiste idempotente por EvAssigned. Cambio local no validado.

## 23. Token de finalización

AI Service `retirement.finalize` request dispatch ahora requiere `leaseOwner` string y `leaseGeneration` integer antes de responder; bloque timestamp debe parsear y corresponder al control bloqueado. La autoridad CRM aún valida lease vigente y generation antes de terminalizar. Cambio local no validado.
