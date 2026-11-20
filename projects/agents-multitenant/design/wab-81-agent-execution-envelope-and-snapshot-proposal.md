

## 16. Reconciliacion del inbound y retiro del routing Bot legacy (19/11/2026)

Revisados `fase-2-diseno-objetivo-y-contratos.md`, `decisiones-operativas-agent-instance.md`, el complemento tecnico de asignaciones/EvAssigned y Linear WAB-81, se confirma:

- Bot/BotInstance legacy no llego a produccion; se reemplaza directamente por AgentDefinition/AgentInstance, sin migracion, backfill, cutover, rollback de datos ni compatibilidad funcional.
- Retirar routing/forwarding legacy, BotClient, modelos/contratos de dominio y referencias operativas que ya no correspondan. No mantener un camino Bot paralelo al Producer AgentInstance.
- Solo reutilizar infraestructura neutral (Outbox/Change Streams/workers/transacciones/session propagation) si se valida contra los contratos nuevos.
- Asignacion efectiva: `AgentInstance | human | unassigned`, con `assignedType=agent|human|unassigned`; CRM es autoridad. AI Service solo proyecta.
- Human vigente: no ejecutar Agent.
- AgentInstance vigente y ACTIVE: Producer resuelve esa instancia.
- Unassigned ante nuevo mensaje: aplicar DEC-CM-06/08; asignar default del Workspace solo si existe y esta ACTIVE. Sin default activa, no ejecutar IA y dejar disponible para atencion humana.
- No fallback a Bot ni a runtime legacy.
- Cada execution conserva `evAssignedId` causal; el checkpoint valida que la asignacion vigente corresponde a la transicion que autorizo el mensaje. No introducir `assignmentVersion`.

Este alcance reemplaza la propuesta previa de mantener un branch Bot legacy en MessageProcessor. La retirada del routing legacy se considera parte del reemplazo, no una migracion separada de datos.


## 17. Contratos de ejecución CRM Producer → Outbox → AI Service Admission (19/11/2026)

### 17.1 Principios y autoridades

- CRM es autoridad de tenancy, Contacto, asignación vigente y causalidad `EvAssigned`.
- Producer no acepta `organizationId`, `workspaceId`, `agentInstanceId` ni `evAssignedId` de un payload externo como autoridad: los deriva del contexto inbound validado y de la persistencia CRM.
- AI Service valida el envelope, pero no resuelve ni modifica asignaciones CRM.
- El transporte puede duplicar y reordenar; la Admission debe ser idempotente y durable.
- Una ejecución admitida se identifica por `executionId`; el mensaje de origen por `originMessageId`. Ambos son estables en reintentos.
- Human y Unassigned sin default ACTIVE son resultados legítimos sin ejecución IA. No constituyen error de routing ni habilitan fallback.

### 17.2 Envelope propuesto (contrato versionado)

```typescript
interface AgentExecutionEnvelopeV1 {
  schemaVersion: 1;
  messageId: string;              // originMessageId estable: CRM EvMensaje._id
  executionId: string;            // UUID generado por CRM Producer
  occurredAt: string;             // ISO-8601 UTC
  source: { provider: 'meta' | 'evo'; providerMessageId: string; crmEventId: string }; // providerMessageId = EvMensaje.wamid; crmEventId = EvMensaje._id
  tenancy: { organizationId: string; workspaceId: string };
  assignment: {
    contactId: string;
    conversationId: string;
    assignedType: 'agent';
    agentInstanceId: string;
    agentDefinitionId: string;
    evAssignedId: string;          // transición causal que autorizó ejecución
  };
  message: {
    channel: string;
    content: string;
    receivedAt: string;
  };
  runtimeSnapshot: AgentRuntimeSnapshot; // snapshot completo, inmutable, hash validado
}
```

Este envelope es propuesta para revisión contractual; no se declara compatible con código existente hasta que el schema Zod/DTO, publisher y ingress compartan la misma definición. El envelope no transporta secretos, credenciales ni policy inferida.

### 17.3 Reglas del Producer

1. Obtener inbound persistido y sus IDs estables; rechazar si no tiene identidad durable.
2. Resolver Workspace y Organización desde la relación CRM validada.
3. Leer Contacto y asignación actual. `human` → no ejecutar. `unassigned` → aplicar DEC-CM-06/08; asignar default únicamente si ACTIVE; si no, no ejecutar. `agent` → validar AgentInstance ACTIVE y Workspace propietario.
4. Validar que `ref_ev_assigned` apunta a EvAssigned vigente y que la evidencia causal corresponde al AgentInstance seleccionado. No introducir `assignmentVersion`.
5. Resolver snapshot autoritativo. Snapshot inválido, policy ausente, ownership inconsistente o configuración incompleta → fail closed, sin crear request IA.
6. Crear envelope y Outbox request en una única transacción CRM con IDs estables y clave idempotente por mensaje inbound. El Outbox no debe reutilizar el evento de AssignmentSync.

### 17.4 Outbox y publisher

- Evento dedicado: `agent.execution.requested` (nombre provisional hasta reconciliar con nomenclatura canónica de Outbox).
- Persistir envelope completo y hash canónico junto con identidad/correlación; estado pendiente, lease/retry y límite de intentos según el patrón Outbox existente.
- Publicación RabbitMQ durable, persistent message, ConfirmChannel publisher confirm. Marcar enviado solamente tras confirm positivo; confirm negativo, timeout o desconexión deja evento reintentable.
- La entrega puede ser at-least-once; la Admission deduplica. No afirmar exactly-once de transporte.
- El publisher no debe bloquear el procesamiento inbound en una llamada no durable a AI Service.

### 17.5 AI Service ingress y Admission

- Endpoint/consumer autenticado de servicio; validar schemaVersion, campos requeridos, tipos, tenancy/IDs, timestamp, snapshot identity y configHash antes de Admission.
- No provisionar/reabrir `AgentInstanceExecutionControl` desde ingress. El control debe existir OPEN por lifecycle/reconciler; falta control → rechazo fail-closed.
- Admission transaccional: localizar control por `agentInstanceId + workspaceId + admissionState=open`; incrementar `admissionSequence` en el mismo documento que usa el bloqueo WAB-79; insertar `AgentExecution` y `AgentExecutionInbox` en la misma transacción.
- Persistir `admissionGeneration` y `admissionSequence` en ambas entidades. Si BLOCKED gana la competencia, no se admite. Si Admission gana, queda durable y el drain debe contarla como non-terminal.
- Duplicado con mismo originMessageId/executionId e identidad/hash idénticos → devolver resultado idempotente y la generación/secuencia originales. Reutilización con identidad o payload diferente → conflicto; nunca crear segunda ejecución.
- El hash cubre envelope canonicalizado; snapshot hash también se valida contra su contenido según contrato de Snapshot.

### 17.6 Snapshot canónico y versionado

- `configHash`: SHA-256 de JSON canónico del contenido efectivo, excluyendo campos volátiles de identidad (`snapshotId`, `capturedAt`) y el propio hash. Orden estable para objetos; arrays conservan orden semántico definido por contrato.
- `snapshotVersion`: versión monotónica del snapshot/config efectivo de AgentInstance; no puede derivarse de `LLMRoute.version`. Su fuente y mecanismo de incremento deben definirse contra persistencia CRM existente antes de implementación.
- `snapshotId`: identificador estable para el snapshot materializado; no generar otro ID en cada retry si el contenido y la resolución pertenecen al mismo request de ejecución.
- El AI Service debe recalcular hash y validar identidad (organization/workspace/instance/definition), versión positiva, ruta y status ACTIVE; mismatch → rechazo.
- La política de capabilities/limits debe proceder de un resolver autoritativo. No usar defaults vacíos ni aceptar policy proporcionada por usuario/mensaje inbound.

### 17.7 Dependencias que deben resolverse antes de declarar contrato cerrado

1. Confirmar fuente autoritativa y schema de `Policy/Capabilities/Limits` en CRM/Waby.
2. Confirmar definición de `conversationId` e identidad durable del inbound en el MessageProcessor actual.
3. Confirmar si Outbox genérico soporta payload, retries, leases y publisher confirm para el nuevo evento, sin colisionar con AssignmentSync/Retirement.
4. Confirmar provisioning de `AgentInstanceExecutionControl` para instancias ACTIVE y comportamiento de instancias suspendidas/retiradas.
5. Confirmar el consumidor real de `AgentExecutionInbox` y su handoff al ExecutionEngine; el worker existente no se considera runtime completo por sí mismo.
6. Confirmar semántica de deduplicación cuando originMessageId y executionId apuntan a dos registros distintos (conflicto), así como manejo de carreras por índices únicos/transacción.

### 17.8 Secuencia de implementación

1. Cerrar schema y validator compartidos del envelope/snapshot.
2. Resolver policy autoritativa y snapshot canonical/version source.
3. Implementar Producer CRM y Outbox dedicado.
4. Implementar ingress autenticado + Admission idempotente.
5. Retirar Bot routing/forwarding/client/domain legacy sin fallback.
6. Conectar Inbox admitida al runtime AgentInstance/ExecutionEngine existente.
7. Revisar diff, actualizar Linear y documentar bloqueos; no ejecutar tests/build/typecheck/lint sin autorización explícita.


### 17.9 Inventario inicial de Bot legacy (CRM y AI Service)

La búsqueda física confirma que el legacy no se limita a `BotRoutingPolicy`/`forwardToAIService`:

- CRM: `BotRouting.policy`, `bot-history`, queue publisher/message-received DTO, `aiVigencia.repo`, outbound/completion consumer, BotResourceLink GraphQL/model/repository/rules-skills service y referencias en Contacto.
- AI Service: DTO/entities Bot/BotInstance, configuration/execution/process-message/prompt-management/vigencia services, use-cases CRUD/readiness/credentials/prompt assignment, repositories/models/mappers, `bot-runtime-resolver`, runtime/context/engine steps, provider credential adapter, message-received handler, capability executor, memory extractor, composition root y route `bot.routes`.
- Tests existentes contienen contratos legacy; deben reemplazarse por tests de AgentInstance y no conservar comportamiento Bot como fallback.

La eliminación será por dependencias y ownership: primero Producer/Admission y handoff AgentInstance funcional; después retirar los módulos/routes/modelos Bot no utilizados y reemplazar las integraciones necesarias. No borrar infraestructura neutral (providers, capabilities, RAG, memory, execution engine, Rabbit, Outbox) por coincidencia textual; debe validarse su contrato y adaptarse al runtime AgentInstance. Esta lista es inventario inicial, no autorización para borrar todos los archivos mencionados sin revisar dependencias.

## 18. Physical contract closure findings (implementation inspection)

The following findings are grounded in the current CRM and AI Service repository, and supersede any proposal-level assumptions where they differ:

- `EvMensaje._id` is the durable CRM origin-message identity. `wamid` is provider metadata and is not globally unique; the CRM model documents its uniqueness as scoped by `(wamid, contacto)`.
- `EvMensaje.ref_conversation` is the persisted CRM conversation reference. `EvMensaje.contacto` is the contact reference. `Evento.workspace` is the workspace reference.
- Assignment evidence is represented by `ContactAssignmentRecord`, including `evAssignedId`, `contactId`, `workspaceId`, `prevAssignedType/ref`, `newAssignedType/ref`, actor and operation. `AssignedType` is `human | agent | unassigned`. The execution request must preserve the causal `evAssignedId`; it must not synthesize an assignment version.
- `AgentRuntimeSnapshot` currently defines identity (`organizationId`, `workspaceId`, `agentInstanceId`, `agentDefinitionId`, `snapshotId`, `snapshotVersion`, `capturedAt`, `configHash`), definition route, instance resources, and policy (`allowedCapabilities`, `limits`). Its validator checks identity, metadata, active instance, route and policy shape, but does not calculate/verify `configHash` or establish authoritative source/version semantics.
- CRM inbound persistence returns the saved `mensaje`; the producer must derive origin identity and conversation from this persisted record, not raw webhook values. Do not forward messages where persistence was blocked or failed.
- Existing `contactAssignmentClient.getActive()` is a read of AI Service assignment state and does not, by itself, provide CRM causal evidence. The Producer must resolve the CRM-authoritative assignment evidence and reconcile it with the assignment projection before requesting an execution snapshot.

### Contract blockers still open

1. Determine a durable and race-safe CRM lookup for the current causal `EvAssigned`/`ContactAssignmentRecord` matching the active AgentInstance assignment; do not use an invented `assignmentVersion`.
2. Confirm the authoritative CRM runtime-snapshot resolver and policy/capability/limit source. Current snapshot contract is only a transport interface; identity validation alone is not a complete snapshot validator.
3. Establish snapshot version semantics and canonical config hash calculation/verification. Existing assignment-snapshot version controls are for assignment snapshot paging and must not be reused as AgentInstance runtime config versions.
4. Confirm the CRM Outbox supports execution-request delivery semantics (publisher confirms, retry/lease ownership, terminal handling) or design the extension explicitly; existing Outbox event handling is not evidence that this new event is covered.
5. Confirm AI Service has a dedicated execution-request ingress and that the existing `AgentExecutionInboxWorker` is wired to a real AgentInstance runtime; current worker accepts a runtime interface and is not proof of the target runtime connection.

Implementation remains fail-closed: absent assignment evidence, active AgentInstance, authoritative snapshot or valid admission control, no execution is admitted.

### Additional snapshot implementation finding

The current `POST /agent-runtime/snapshots` endpoint in AI Service is not yet a trustworthy effective-snapshot resolver:

- The route accepts `agentInstance` and `policy` in the request body instead of resolving their authoritative CRM sources.
- It checks that the AgentDefinition exists and is active and resolves its LLMRoute, but does not establish that the AgentInstance belongs to the supplied Workspace/Definition.
- It generates a new `snapshotId` on each request.
- It sets `snapshotVersion` to `LLMRoute.version`, which is not an effective AgentInstance configuration version.
- `configHash` is calculated using `JSON.stringify(effective)` and includes volatile `snapshotId` and `capturedAt`; it is not a stable canonical hash of effective configuration.

This endpoint must be corrected or replaced by an authoritative resolver before CRM Producer integration. Do not add a second snapshot implementation or accept caller-supplied policy as authoritative.

## 19. Reconciliation with canonical WAB-78/WAB-79 decisions (20/11/2026)

Review of the canonical Phase 2 design, assignment/EvAssigned technical complement, effective configuration/capabilities design, Outbox reconciliation and Linear WAB-81/WAB-79 confirms that several prior blockers can be narrowed without inventing a new domain contract.

### 19.1 Causal assignment identity — resolved contractually, physical checkpoint remains

- `Contacto.ref_ev_assigned` is the canonical pointer to the latest confirmed assignment transition, including release/unassigned transitions; it is not cleared after first assignment.
- `EvAssigned` is a CRM `Evento` discriminator. Its `_id` identifies the transition; `contacto`, `workspace`, `createdAt` are inherited. The approved own fields are `prevAssignedType/ref`, `newAssignedType/ref`, `actorType/ref`, `operationType`, `reason`.
- The producer must read the persisted Contacto assignment tuple and `ref_ev_assigned`, then resolve that exact EvAssigned by ID. It must validate contact/workspace and the event's `newAssignedType/ref` against the persisted current tuple and target AgentInstance. It must not query “latest event by timestamp” as a substitute for the pointer or create `assignmentVersion`.
- If the pointer/event is missing, the tuple does not match, or the target is not ACTIVE/owned by that Workspace, fail closed (no execution). For `human`, do not execute. For `unassigned`, apply DEC-CM-06/08 and only execute after a confirmed automatic transition to an ACTIVE default; use the new confirmed EvAssigned pointer.
- The exact race-safe implementation checkpoint is still physical: read/validate Contacto and causal event in a consistent transaction/snapshot or use an explicit CAS checkpoint against the assignment tuple and pointer before writing the execution Outbox. If the inbound handler performs automatic assignment, the Producer must use the committed result of that transition, not a stale pre-transition read. The contract is closed; this implementation checkpoint remains to be verified in code.

### 19.2 Effective configuration and snapshot — canonical sources clarified

Phase 2 D.1.9 ownership and CTX-P3/P4 define the source composition:

- AgentDefinition, central Prompt/Skills/Rules and LLMRoute are Waby-owned in AI Service DB. AgentDefinition references the current LLMRoute by stable `llmRouteId`; it has no published version pin.
- AgentInstance, SkillConfiguration, AgentRules and its KnowledgeBase references are CRM-owned Workspace configuration. AI Service resolves/consumes them for runtime; no editable duplicate is authoritative.
- KnowledgeBase/Documents and their ingestion/RAG artifacts are AI Service-owned technical persistence, but Workspace ownership and access are validated through CRM context.
- LLMRoute's `version` is the approved revision of that route's provider/model/allowlisted parameters and must be captured. It is not an AgentInstance snapshot version.
- The effective snapshot must include the full effective configuration described by P10: stable AgentDefinition identity, `llmRouteId` and route version, effective provider/model/parameters, Skill identities/release or Workspace content hash, SkillConfiguration, Rules, authorized capabilities, Knowledge references and any documents/chunks actually incorporated into context. It excludes secrets.

No canonical contract requires a monotonic `snapshotVersion` field for AgentInstance. Remove this invented requirement from the proposed envelope/snapshot. Snapshot identity is carried by a stable `snapshotId` for the materialized execution request, while reproducibility/integrity is provided by `effectiveConfigHash`/`configHash` and the immutable snapshot content. A retry of the same Outbox request must reuse its persisted snapshot and IDs; a new execution request may materialize a new snapshot even when configuration content is unchanged.

Hash contract: canonical SHA-256 over the effective snapshot content, with deterministic object-key ordering and semantically defined array ordering; exclude volatile envelope metadata (`snapshotId`, capture timestamp) and the hash field itself. Include all effective configuration values/references that affect execution; never include secrets. The AI Service validator recomputes and compares the hash, verifies the organization/workspace/instance/definition ownership chain, current allowed status, route reference/version and schema. Hash equality is not authorization: current capability authorization/revocation remains enforced at the execution point.

### 19.3 Policy, capabilities and limits — source composition closed; physical resolver pending

The canonical effective-config rule is `CapabilitiesEffective = PlatformAllowed ∩ DefinitionAllowed ∩ InstanceEnabled ∩ CurrentAuthorization`. Skills do not grant executable capability. Platform policies are mandatory and cannot be overridden by Definition, Instance, Rules, Skills, Knowledge or user content. LLM technical configuration comes only from Waby-managed LLMRoute.

The snapshot's policy/capability/limit data must be resolved from authoritative platform policy/catalog and current authorization, AgentDefinition, CRM AgentInstance configuration, and Organization/Workspace entitlement/limits as applicable. It must not be caller-supplied as trusted policy. Exact persisted schema, resolver owner and source for every limit remain unverified; do not invent empty/default allowlists or limits. Resolver must fail closed when required authoritative data is unavailable.

### 19.4 Execution Outbox — canonical delivery contract found

Phase 2 D.1.9 already defines distinct ownership:
- `AgentMessageOutbox`: CRM source-message intent for CRM → AI Service, durable in CRM DB, distinct from assignment Outbox.
- `AgentExecutionInbox`: durable/idempotent receipt in AI Service, linked to one execution and originMessageId.
- `ExecutionResultOutbox`: durable terminal-result intent in AI Service; confirmed only after broker confirmation and valid routing.
- `ExecutionResultInbox`: durable/idempotent receipt in CRM; Redis may cache but is not the dedup authority.

D.6 requires the inbound source intent and durable receipt/acceptance to be persisted before ACK, and acceptance to preserve the immutable snapshot/hash. Delivery is at-least-once with durable deduplication; do not claim transport exactly-once. Existing CRM generic Outbox is not proof of this execution context's schema/state machine. Its `PENDING/SENT/FAILED` model and worker must be extended/reused only after verifying confirm, routing, retry, uncertain publish, lease/ownership and terminal retention semantics. `SENT` must mean the contractually defined broker confirmation/routing outcome, not merely `channel.publish()` returning true.

### 19.5 Closure and implementation checkpoint

Contract decisions now grounded in canonical documents: causal pointer/event identity; effective configuration ownership/composition; P10 snapshot contents; capabilities intersection; AgentExecution snapshot/hash fields; separate AgentMessageOutbox/ExecutionInbox/ExecutionResultOutbox/ResultInbox ownership; durable-before-ACK and at-least-once delivery.

Physical verification still required before Producer code: (a) assignment transition result/consistent CAS checkpoint in current CRM inbound flow, (b) authoritative runtime snapshot resolver and source lookup for each field, (c) concrete policy/capability/limits schema/resolver, (d) execution Outbox publisher confirmation and retry/lease implementation, and (e) admission control provisioning + actual runtime handoff. These are implementation inspections/checkpoints, not reasons to reopen the approved architecture.

### 19.6 Correction to the earlier proposed envelope and runtime-snapshot transport

Section 17 is a proposal and is superseded where it mentions `snapshotVersion` as AgentInstance runtime configuration version. The canonical `AgentExecution` contract uses `effectiveConfigHash` and `effectiveConfigSnapshot`, plus `promptSnapshotHash`; it captures the snapshot when durable acceptance occurs, before acknowledging inbound. It captures `LLMRoute.version`, not an AgentInstance snapshot counter. The runtime transport identity should therefore be reviewed to remove `snapshotVersion` and align naming with the canonical `effectiveConfig*` fields rather than introducing a parallel version/hash vocabulary.

The proposed envelope's `runtimeSnapshot` is transport metadata for durable acceptance; it does not imply that the CRM must own AI Service entities or that CRM may authoritatively supply platform policy. The ingress must validate ownership and authenticity and persist the effective snapshot as part of durable acceptance. If the architecture resolves some fields through AI Service authority, the CRM Producer must call the authenticated resolver before writing the request Outbox; that resolver must return a stable materialized result tied to the request identity and be verified by ingress.

### 19.7 Outbox code inspection checkpoint

The current CRM Outbox is generic (`messageId`, `type`, `payload`, `PENDING/SENT/FAILED`, `attempts`, `lastError`, timestamps; sparse unique `messageId`). The current worker/change stream handles workspace deletion, assignment sync, assignment transition and retirement events. The retry poller is scoped to assignment sync and coordination events. It does not yet route `agent.execution.requested`.

Rabbit connector creates a ConfirmChannel. Existing coordination publisher uses publisher confirms, and Outbox marks SENT after that publish helper returns. This is evidence of a reusable mechanism, not proof that execution-request routing, durable queue configuration, publisher confirmation semantics, uncertain publish handling, consumer ACK, deduplication and retry/retention are complete for the new context.

For execution requests, add a dedicated event handler and poller eligibility only after defining its routing/queue and terminal/retry policy. Do not silently send it through assignment or retirement handlers. Preserve the execution request in CRM Outbox and make its payload stable across retries. The AI Service durable Inbox is the deduplication boundary; publish/consume is at-least-once.

## 20. Snapshot implementation closure attempt (20/11/2026)

### Contract decisions closed
- AgentInstance runtime snapshot has no canonical monotonic `snapshotVersion`; remove it from the transport contract.
- `LLMRoute.version` is captured as route revision only.
- Canonical integrity field is `effectiveConfigHash` (aligning with AgentExecution `effectiveConfigHash/effectiveConfigSnapshot`), calculated as SHA-256 over deterministic effective config, excluding volatile identity/capture metadata and the hash itself.
- Snapshot materialization must be request-stable: retries reuse persisted snapshot and identity; a new execution request can materialize a new snapshot.
- Effective composition follows �19.2; policy follows �19.3 and is never trusted from caller input.

### Physical implementation status
The inspected AI Service route `POST /agent-runtime/snapshots` is currently unsafe to integrate: it accepts `agentInstance` and `policy` from caller, validates only AgentDefinition active and LLMRoute existence, does not verify AgentInstance Workspace/Definition ownership, generates snapshotId per call, uses LLMRoute.version as snapshotVersion, and hashes JSON including volatile identity fields. The CRM client and AI Service contract also still expose `snapshotVersion/configHash` legacy names.

The physical CRM AgentInstance model confirms Workspace, Definition, status, additionalPrompt and knowledgeBaseIds; however, the authoritative resolver for AgentRules, SkillConfiguration, platform capabilities, current authorization and Organization/Workspace limits has not been established in the inspected code. No safe resolver can be completed without those sources. Do not adapt the current endpoint by accepting caller-supplied policy or defaulting capabilities/limits.

### Next implementation prerequisite (narrow and actionable)
1. Locate and verify authoritative models/services for AgentDefinition, AgentRules, SkillConfiguration, capabilities/current authorization and entitlement/limits.
2. Replace the snapshot DTO/contract vocabulary (`snapshotVersion/configHash`) with canonical `effectiveConfigHash/effectiveConfigSnapshot` and captured `LLMRoute.version`; add deterministic canonical hash verification in the trusted resolver/ingress.
3. Resolve ownership and ACTIVE state from CRM-authoritative AgentInstance/Workspace/Definition records, and resolve Waby-owned Definition/LLMRoute/resources from AI Service authority.
4. Materialize and persist request-stable snapshot before CRM execution Outbox creation; ensure retry returns the same snapshot identity/content.
5. Only then connect Producer to resolver. Missing authoritative source remains fail-closed.

This closes the design ambiguity but does not claim snapshot implementation complete. No production code was changed in this checkpoint; no tests/build/typecheck/lint were run.

### Source verification update
The CRM physical models/services now verify two snapshot components:
- `AgentRules` (`agent_rules`) is keyed by `agentInstanceId`; effective rule content/name/priority is Workspace configuration. `enabled` is optional in the physical model, so resolver semantics must be reconciled with the canonical contract before inclusion/exclusion (do not silently assume a default).
- `SkillConfiguration` (`skill_configurations`) is keyed by `agentInstanceId + source + ownerId + skillId`; it records source, owner, skill identity/name, optional enabled and config. Workspace ownership is validated by CRM services. Resolver must filter/interpret enabled and validate the Waby/workspace skill source against authoritative catalogs/releases; CRM rows alone do not establish executable skill content or capabilities.

This narrows the unresolved source inventory: AgentInstance, AgentRules and SkillConfiguration are physically available in CRM. Still not established: Waby AgentDefinition/current Prompt/Skill/Rule catalog and release/hash data in AI Service, capability catalog and current authorization resolver, and Organization/Workspace entitlement/limit source. Therefore snapshot remains blocked at the trusted resolver/policy portion; no code change was made.

### �20.3 Clarification � capabilities, authorization and limits are already contractually defined

A subsequent cross-check of the canonical Phase 2 design, `configuracion-efectiva-y-capacidades.md`, Linear WAB-80 and WAB-81 confirms these are not undefined product/design concepts. Their contract exists:
- `CapabilitiesEffective = PlatformAllowed n DefinitionAllowed n InstanceEnabled n CurrentAuthorization`.
- Platform mandatory policy and Waby capability catalog are authoritative; AgentInstance cannot elevate capabilities; Skill/Prompt content is not authorization.
- WAB-81 requires resolving Organization/Workspace, instance ownership, state, permissions and limits before execution; missing/untrusted context fails closed.
- WAB-80 defines Workspace/Organization ownership checks and authorization boundary for AgentInstance administration; the runtime authorization/capability gate is a separate enforcement concern.
- Phase 2 effective-configuration design defines Organization plan/entitlement ownership and technical policy boundaries. P4 credit authorization is already specified and must not be conflated with general runtime capability limits.

The remaining gap is not absence of a design. It is physical implementation/source mapping: identify which existing services/catalogs/entitlement models provide each contract value, and verify runtime enforcement. No new product decision or new design document is required. This clarification supersedes the earlier �20 wording that broadly described these as unresolved design sources.


### 20.4 Physical source inspection — capabilities, authorization and limits (19/11/2026)

Focused inspection of CRM `src` and AI Service `microservices/ai-service/src` confirms the following:

**Sources that exist and can be resolved physically**
- CRM `AgentInstance` (`src/models/agent/AgentInstance.model.ts`, `AgentInstance.service.ts`, repository): Workspace, AgentDefinition ID, lifecycle status, `additionalPrompt`, `knowledgeBaseIds`. Service validates Workspace→Organization ownership and instance→Workspace ownership for its operations.
- CRM `AgentRules` (`src/models/agent/AgentRules.model.ts`, `AgentRules.service.ts`, repository): `agentInstanceId`, `ruleName`, `content`, optional `priority`, optional `enabled`. The model does not establish a canonical enabled default; resolver must follow contract/verified behavior.
- CRM `SkillConfiguration` (`src/models/agent/SkillConfiguration.model.ts`, service/repository): instance, source (`waby|workspace`), owner, skill ID/name, optional enabled/config. CRM validates Workspace ownership and workspace ownerId for `source=workspace`; it does not provide executable catalog content/release or capability authorization.
- AI Service `AgentDefinitionModel` / repository: Waby-owned definition identity, `systemPrompt`, stable `llmRouteId`, defaultSkills/defaultRules and lifecycle status. Physical schema has no allowed-capabilities field/catalog mapping.
- AI Service `LLMRouteModel` / repository: route provider/model/version and allowlisted numeric technical parameters. Route version is material route revision.
- AI Service `CapabilityRegistry` in `infrastructure/capabilities/capability-registry.ts`, composition root: registers the currently implemented runtime capabilities. This registry is a runtime implementation registry, not the canonical Waby platform authorization catalog and does not by itself represent PlatformAllowed, DefinitionAllowed, InstanceEnabled and CurrentAuthorization separately.

**Sources not established by this inspection**
- Waby authoritative platform capability catalog/mandatory policy and the mapping from catalog entries to the runtime CapabilityRegistry.
- AgentDefinition allowed-capability source (no field/schema mapping found in the inspected AI Service AgentDefinition model/domain entity).
- Current actor/Workspace authorization resolver for runtime capability use. CRM workspace/organization ownership checks in AgentInstance/Rules/Skills administration are not equivalent to current execution authorization.
- Organization/Workspace entitlement and runtime limits mapping for AgentInstance executions. CRM has Organization subscription (`Suscri`/`TipSuscri`) and plan models/services, but no verified mapping from those records to the WAB-81 runtime capability/limit contract was found. Organization AI credit authorization is a distinct P4 contract and must not be substituted for generic capability/limit resolution.

**Conclusion / implementation decision**
The sources are sufficient to read the CRM-owned instance resources and Waby-owned definition/route, but not to calculate the complete authoritative policy tuple `PlatformAllowed ∩ DefinitionAllowed ∩ InstanceEnabled ∩ CurrentAuthorization`, nor to derive required runtime limits/entitlements. Implementing the snapshot resolver now would require inventing a catalog, authorization source or defaults, which violates the approved contract. Therefore no production-code change is made in this checkpoint. The blocker is narrowed to the specific authoritative source/mapping above, not a design gap. Continue by locating the existing Waby catalog/authorization/entitlement implementation (including other ElemeSoft repositories if applicable); if no source exists, report that concrete absence in WAB-81 and request the minimum implementation decision rather than adding a parallel catalog or permissive defaults.

Inspection only: no tests/build/typecheck/lint/commit/push were run.


### 20.5 Repository scope correction (19/11/2026)

The workspace repository inventory was clarified by the owner: backend and microservices are in `F:\ElemenSoft\crm`; shared backend infrastructure/DevOps is in `F:\ElemenSoft\infrastructure`; application frontend is in `F:\ElemenSoft\WABY_frontend`; `F:\ElemenSoft\wabyland` is a landing page, not a product backend. This checkpoint searched the relevant backend, infrastructure and frontend repositories. Infrastructure search found no capability/authorization/entitlement implementation beyond deployment configuration; frontend search found no runtime authority (frontend permissions are presentation/route concerns only). The landing-page repository is excluded from runtime source-of-truth investigation. No additional product repositories are presumed; external repositories may be searched only if a concrete dependency warrants it and owner authorization permits.

### 20.6 CRM targeted search: authoritative limits, authorization and capability sources

Physical inspection completed in crm (not infrastructure, frontend or landing page):

- src/models/billing/TipSuscri.model.ts defines Organization subscription-plan limits: macs_max, roadcast_max, contacts_max, users_max, workflows_max, workspaces_max, and optional storage_max. No AI-agent execution, capability, skill, token, credit or runtime quota field exists in this model.
- src/models/billing/Suscri.model.ts binds a subscription to TipSuscri, with ctiva, estado, and lifecycle timestamps. It is not an AI execution authorization source by itself.
- src/models/org/TipAccessOrg.model.ts and src/models/workspace/TipAccessWorks.model.ts store role permission strings. Workspace permissions are namespace strings and defaults; src/auth/WrkHierarchy.ts explicitly handles role hierarchy, not permissions. These are user-operation authorization mechanisms, not AgentInstance runtime capability policy.
- AI Service search found legacy Bot/BotInstance.enabledCapabilities and CapabilityRegistry/CapabilityExecutor, but these are legacy Bot-domain implementation and runtime registration. They do not establish the Waby PlatformAllowed ∩ DefinitionAllowed ∩ InstanceEnabled ∩ CurrentAuthorization policy for AgentInstance. Do not reuse or translate legacy Bot capability values without a canonical mapping.

Conclusion: this targeted search found no physical authoritative source for the WAB-81 capability policy tuple or AI execution limits/entitlements. Existing CRM subscription limits are generic product limits and cannot be mapped to AI runtime limits by inference. No resolver implementation can safely materialize the complete contract from current sources without inventing authority/defaults. Production code remains unchanged.

Next required source/decision: establish or identify the authoritative Waby capability catalog and definition-level allowed-capability policy, instance-enabled policy, current execution authorization source, and AI-specific entitlement/limit contract (including relation to Organization credits/P4). Once these source contracts exist, resolver implementation can proceed; until then fail-closed behavior remains required.

### 20.7 Investigation closure and minimum implementation boundary (20/11/2026)

The scoped source investigation is closed. CRM, AI Service, shared infrastructure and WABY frontend were searched for existing authoritative capability policy, runtime authorization and AI entitlement/limit sources. The only physical sources found are instance resources (CRM), Waby AgentDefinition/LLMRoute (AI Service), generic Organization subscription limits and user-role permissions. None implements the full WAB-81 policy tuple or AI runtime limits. Legacy Bot capability configuration is not an acceptable authority.

**Closure conclusion:** this is a concrete implementation gap, not an open repository-search task and not a reason to reopen the approved WAB-81 design. The snapshot resolver must remain fail-closed until the authoritative inputs exist. Do not create an ad hoc parallel catalog, infer AI limits from generic plan fields, accept caller-supplied policy, or repurpose legacy Bot capabilities.

**Minimum implementation boundary when the gap is addressed:**
1. Waby-owned capability catalog and mandatory platform policy, with explicit mapping to implemented AI Service runtime capabilities.
2. AgentDefinition-owned allowlist/policy source and CRM AgentInstance-enabled configuration, each resolved from their actual owners.
3. An authenticated current-authorization decision at execution time (including revocation semantics); instance ownership validation alone is not this decision.
4. AI execution entitlement/limits source and explicit mapping to Organization/Workspace billing plus P4 credit authorization, without conflating the contracts.
5. A trusted resolver that materializes the effective intersection and limits; snapshot validator recomputes canonical hash and verifies ownership/schema, while execution enforces current authorization.

The investigation checkpoint is closed and its evidence recorded. WAB-81 itself remains In Progress because its broader runtime, producer, admission and ExecutionEngine handoff acceptance criteria are not complete. No production code changed; no tests/build/typecheck/lint/commit/push were run.
