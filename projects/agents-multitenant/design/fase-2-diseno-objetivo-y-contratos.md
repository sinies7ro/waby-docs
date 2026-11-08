# WAB-78 — Fase 2: Diseño objetivo y contratos

> **Estado documental actualizado al 07/11/2026.** Este documento es el diseño integral de Fase 2 reconciliado con las decisiones aprobadas en Linear. En caso de contradicción prevalecen ARQ-19/20/21, DEC-CM-01..08 y P4 aprobada. Los acuerdos conceptuales no implican implementación. WAB-78 permanece In Progress; no cerrar la fase ni iniciar WAB-79 sin autorización expresa.
>
> **Cambios de vigencia:** P1 sustituye el cupo comercial antiguo y deja sin vigencia I3 en lo relativo a límites de cantidad; se retiraron las reglas antiguas de cupos 1/2 y downgrade, routing persistente canal→instancia, permisos `platform.*`/`bot.configure`/`wrk.agent.use`, y la autoridad exclusiva de Waby sobre el ciclo de vida lógico de AgentInstance. Ver el [registro de reconciliación](./reconciliacion-vigencia-fase-2-wab-78.md).


**Padre:** WAB-75 · **Fase:** WAB-78 · **Estado:** Diseño (previo a implementación) · **Fecha:** 02/11/2026

**Regla de cierre:** WAB-78 NO se cierra ni WAB-79 se inicia hasta que el usuario revise y autorice explícitamente. Este documento es de solo diseño: no se modifica código, schemas, colecciones, resolvers, endpoints ni infraestructura.

**Actualización aprobada (03/11/2026):** P4 queda aprobada, incluida la política híbrida de facturación G.4.6 y un período de gracia de **30 días** para operaciones que permanezcan en `reconciliation_required` sin evidencia concluyente. Esta aprobación no cierra WAB-78 ni autoriza la implementación o el inicio de WAB-79.

---

## 1. Resumen ejecutivo

Esta fase diseña la transformación del modelo actual **Bot / BotInstance** en el modelo objetivo **AgentDefinition (Waby) + AgentInstance (Workspace)**. Waby administra definiciones, catálogo, proveedores/modelos/credenciales LLM, políticas y provisioning técnico; los usuarios autorizados del Workspace administran el ciclo de vida lógico y la configuración empresarial permitida de sus instancias. No existe cupo comercial de AgentInstance por Workspace ni por Organización. Knowledge pertenece al Workspace; Skills/Rules y configuración empresarial pertenecen a la instancia; los créditos se comparten en una única bolsa por Organización.

El diseño se basa en los hallazgos de la Fase 1 (WAB-77, [informe consolidado](../audits/wab-77/informe-consolidado-fase-1.md)) y toma como referencia las invariantes I1–I21 de WAB-75 §2. La antigua I3 relativa a cupos comerciales queda supersedida por P1 aprobada: no hay límite comercial de AgentInstance por Workspace ni Organización.

**Decisiones resueltas que no se reabren:** P1 (sin cupo comercial), P2 (reglas funcionales de asignación y routing consolidadas en DEC-CM-04/06/08), P3 (modelo híbrido ARQ-19 y permisos ARQ-20), P4 (política híbrida G.4.6 aprobada), P9 (cinco estados; pendientes solo los detalles de retiro indicados), P10 (snapshot inmutable por ejecución con referencias identificables/versionadas de configuración y Knowledge; permanecen pendientes la política de publicación, ciclo de versionado y retención de Knowledge). Permanecen abiertos los contratos técnicos enumerados al final y en Linear.

---

## A. Modelo de dominio

### A.1 Principio de diseño: reutilizar, adaptar o separar

La auditoría de Fase 1 confirmó que `Bot` mezcla definición + instancia. No hay entidad separada de definición. El modelo objetivo debe separar claramente qué es propiedad de Waby (definición) y qué es propiedad del Workspace (instancia).

**Entidades actuales reutilizables:**
- `Execution` → se conserva y amplía con campos de auditoría (I20, I21)
- `KnowledgeBase` → se conserva como `WorkspaceKnowledge` (compartido por workspace, I11)
- `Prompt` / `PromptVersion` → se reutilizan como base de `AgentDefinition` (versión de definición)
- `MemoryFact` → se reemplaza por `ConversationMemory` (ventana 20, I14)
- `BotCredential` → se elimina; credenciales de proveedores LLM centralizadas en Waby (I6)

**Entidades nuevas requeridas:**
- `AgentDefinition` (propiedad Waby)
- `AgentInstance` (propiedad Workspace)
- `SkillConfiguration` (por instancia, I12)
- `AgentRules` (por instancia, I13)
- `ConversationMemory` (por conversación/instancia, I14)
- `AgentExecutionLog` (auditoría por ejecución, I21)
- `OrganizationAICredit` / `CreditLedger` (bolsa única por Organización, I17/I20)

### A.2 Entidades, responsabilidades y cardinalidades

#### Contacto y asignaciones (CRM)

La asignación efectiva se mantiene en el documento Contacto de CRM mediante assignedType, ref_user_assigned y ref_agent_assigned, conforme a D.1.5. CRM es la única fuente de verdad.

EvAssigned es un discriminator de Evento en la colección eventos de CRM. Cada transición confirmada tiene identidad única y el historial no se replica a ai-service. La identidad de EvAssigned identifica la transición que autorizó una ejecución y resuelve ABA; no se sustituye por originMessageId.

**Compatibilidad histórica:** contactAssignmentHistory se conserva durante esta fase como artefacto/estructura legacy si está presente en los datos o modelos de origen. No se presupone su existencia en el código actual ni se lo considera fuente canónica de nuevas transiciones: ese rol corresponde a EvAssigned. No eliminarlo como parte de esta fase sin una decisión de migración explícita.

**Proyección runtime:** contact_assignments en ai-service es una proyección derivada de ContactAssignment, sincronizada desde CRM. No es autoridad editable ni reemplaza la validación contra CRM en los checkpoints que exigen autoridad de dominio.


#### AgentDefinition (Waby)
Propiedad de la plataforma Waby. Representa la definición lógica de un agente: personalidad base, prompt del sistema, Skills/Rules por defecto, proveedor y modelo recomendados por Waby.

| Campo | Tipo | Obligatorio | Descripción |
|-------|------|-------------|-------------|
| id | UUID | Sí | Identificador único |
| name | String | Sí | Nombre de la definición |
| description | String | No | Descripción |
| systemPrompt | String | Sí | Prompt base del sistema |
| provider | Enum | Sí | Proveedor LLM (openai, openrouter, nvidia, gemini) |
| model | String | Sí | Modelo LLM |
| temperature | Decimal | No | Temperatura (default 0.7) |
| maxTokens | Integer | No | Tokens máximos (default 4096) |
| topP | Decimal | No | Top-p (default 1) |
| frequencyPenalty | Decimal | No | Penalidad de frecuencia (default 0) |
| presencePenalty | Decimal | No | Penalidad de presencia (default 0) |
| defaultSkills | JSON[] | No | Plantilla inicial de Skills que se copia a la nueva AgentInstance al crearla; no se hereda dinámicamente |
| defaultRules | JSON[] | No | Plantilla inicial de Rules que se copia a la nueva AgentInstance al crearla; no se hereda dinámicamente |
| version | Integer | Sí | Versión de la definición |
| status | Enum | Sí | `draft` / `active` / `retired` |
| createdBy | String | Sí | Usuario de Waby que creó la definición |
| createdAt | Timestamp | Sí | Fecha de creación |
| updatedAt | Timestamp | Sí | Fecha de última actualización |
| retiredAt | Timestamp | No | Fecha de retiro |

**Cardinalidad:** Una AgentDefinition puede tener cero o más AgentInstance asociadas (1:N, sin unicidad workspaceId+agentDefinitionId per I16).

**Inicialización de AgentInstance:** al crear una instancia, `defaultSkills` y `defaultRules` de la versión de AgentDefinition utilizada se copian como configuración inicial en sus `SkillConfiguration` y `AgentRules`. Son plantillas de inicialización, no herencia dinámica: los cambios posteriores en los valores por defecto de AgentDefinition no modifican instancias existentes. Cada Workspace administra luego la configuración permitida de su instancia, sujeta a las políticas Waby.

#### AgentInstance (Workspace)
Propiedad de un Workspace. Representa una instancia ejecutable de una AgentDefinition asignada a un Workspace específico.

| Campo | Tipo | Obligatorio | Descripción |
|-------|------|-------------|-------------|
| id | UUID | Sí | Identificador único |
| agentDefinitionId | UUID | Sí | Referencia a AgentDefinition |
| workspaceId | String | Sí | Workspace propietario (per I2) |
| name | String | Sí | Nombre de la instancia (visible al cliente) |
| description | String | No | Descripción |
| status | Enum | Sí | `draft` / `active` / `suspended` / `retirement_pending` / `retired` (P9; detalle de retiro pendiente) |
| isDefault | Boolean | Sí | Si es la única instancia predeterminada del Workspace |

| additionalPrompt | String | No | Instrucción empresarial adicional, limitada y validada; subordinada al prompt central y a políticas Waby (DEC-CM-02) |
| skills | JSON[] | Sí | Skills configuradas para esta instancia (I12) |
| rules | JSON[] | Sí | Rules configuradas para esta instancia (I13) |
| knowledgeBaseIds | UUID[] | Sí | Referencias a WorkspaceKnowledge del workspace (I11) |
| createdBy | String | Sí | Usuario autorizado del Workspace que creó la instancia |
| createdAt | Timestamp | Sí | Fecha de creación |
| updatedAt | Timestamp | Sí | Fecha de última actualización |
| activatedAt | Timestamp | No | Fecha de activación |
| suspendedAt | Timestamp | No | Fecha de suspensión |
| retiredAt | Timestamp | No | Fecha de retiro |

**Cardinalidad:** Una AgentInstance pertenece a exactamente un Workspace (N:1). Un Workspace puede tener múltiples AgentInstance; no existe cupo comercial por plan.

#### AgentExecution
Ejecución individual de un agente para procesar un mensaje de usuario.

| Campo | Tipo | Obligatorio | Descripción |
|-------|------|-------------|-------------|
| id | UUID | Sí | Identificador único (execution_id, idempotency key) |
| organizationId | String | Sí | Organización propietaria del saldo consumido |
| workspaceId | String | Sí | Workspace asociado (dimensión de atribución) |
| agentInstanceId | UUID | Sí | Instancia que atendió la ejecución |
| conversationId | String | Sí | Conversación de origen |
| contactId | String | Sí | Contacto de origen |
| channel | String | Sí | Canal de entrada |
| input | String | Sí | Mensaje de entrada |
| promptSnapshotHash | String | Sí | SHA-256 del prompt efectivo construido (I21, P7) |
| effectiveConfigHash | String | Sí | SHA-256 de la configuración efectiva completa (AgentDefinition, Rules, Skills, capabilities, provider, modelo, routing, parámetros) (P10) |
| effectiveConfigSnapshot | JSON | Sí | Snapshot inmutable sin secretos. Incluye como mínimo referencias identificables y versionadas de AgentDefinition, Skills y SkillConfiguration, Rules, capabilities, Knowledge Bases, documentos y fragmentos incorporados al contexto; puede incluir valores efectivos adicionales para reproducibilidad técnica (P7, P10) |
| providerModelUsed | String | Sí | Proveedor y modelo realmente usado (I21) |
| inputTokens | Integer | Sí | Tokens de entrada |
| outputTokens | Integer | Sí | Tokens de salida |
| totalTokens | Integer | Sí | Total de tokens |
| totalCreditsDeducted | Decimal | Sí | Créditos descontados (I20) |
| currency | String | Sí | Moneda (default USD) |
| latencyMs | Integer | Sí | Latencia de ejecución (I21) |
| status | Enum | Sí | `pending` / `completed` / `failed` / `cancelled` |
| error | String | No | Mensaje de error si falló |
| startedAt | Timestamp | Sí | Fecha de inicio |
| completedAt | Timestamp | No | Fecha de completación |
| correlationData | JSON | Sí | traceId, executionId, correlationId |

**Cardinalidad:** Una AgentExecution pertenece a una AgentInstance (N:1), a un Workspace (N:1) y a una Organización (N:1). Una AgentInstance puede tener cero o más AgentExecution (1:N). `organizationId` debe coincidir con la Organización a la que pertenece el Workspace; no se confía en un organizationId recibido libremente en mensajes RabbitMQ o solicitudes internas.

**Nota de trazabilidad:** `prompt_snapshot_hash` identifica el prompt efectivo; `effective_config_hash` identifica el conjunto completo de configuración efectiva. Ninguno de estos hashes permite reconstruir la ejecución por sí solo; para reproducibilidad completa se requiere `effective_config_snapshot` (si se almacena) o las versiones inmutables de AgentDefinition, Rules y Skills referenciadas.

#### WorkspaceKnowledge
Knowledge perteneciente al Workspace, compartido por todas sus AgentInstance (I11).

| Campo | Tipo | Obligatorio | Descripción |
|-------|------|-------------|-------------|
| id | UUID | Sí | Identificador único |
| workspaceId | String | Sí | Workspace propietario |
| name | String | Sí | Nombre de la base de conocimiento |
| description | String | No | Descripción |
| status | Enum | Sí | `active` / `inactive` / `deleted` |
| embeddingConfig | JSON | Sí | Proveedor, modelo, dimensión de embeddings |
| vectorStoreConfig | JSON | Sí | Configuración del vector store |
| retrievalConfig | JSON | No | topK, threshold, maxTokens, rerankingEnabled |
| documentCount | Integer | No | Número de documentos indexados |
| createdAt | Timestamp | Sí | Fecha de creación |
| updatedAt | Timestamp | Sí | Fecha de última actualización |
| deletedAt | Timestamp | No | Fecha de eliminación lógica |

**Cardinalidad:** Un Workspace puede tener cero o más WorkspaceKnowledge (1:N). Todas las AgentInstance del mismo Workspace comparten los mismos WorkspaceKnowledge (I11).

#### SkillConfiguration
Skills pertenecientes individualmente a cada AgentInstance (I12). Sin herencia ni compartición implícita.

| Campo | Tipo | Obligatorio | Descripción |
|-------|------|-------------|-------------|
| id | UUID | Sí | Identificador único |
| agentInstanceId | UUID | Sí | Instancia propietaria |
| skillId | String | Sí | Identificador de la skill (catálogo Waby) |
| skillName | String | Sí | Nombre de la skill |
| enabled | Boolean | Sí | Si la skill está habilitada para esta instancia |
| config | JSON | No | Configuración específica de la skill para esta instancia |
| credentials | JSON | No | Referencias (IDs) a credenciales de integraciones empresariales configuradas y administradas por el Workspace; no almacena secretos ni credenciales LLM. Los secretos se gestionan mediante el mecanismo seguro de credenciales de la plataforma. |
| createdAt | Timestamp | Sí | Fecha de creación |
| updatedAt | Timestamp | Sí | Fecha de última actualización |

**Cardinalidad:** Una AgentInstance puede tener cero o más SkillConfiguration (1:N). Una SkillConfiguration pertenece a exactamente una AgentInstance (N:1).

#### AgentRules
Rules pertenecientes individualmente a cada AgentInstance (I13). Personalidad, tono, instrucciones y restricciones independientes por instancia.

| Campo | Tipo | Obligatorio | Descripción |
|-------|------|-------------|-------------|
| id | UUID | Sí | Identificador único |
| agentInstanceId | UUID | Sí | Instancia propietaria |
| ruleName | String | Sí | Nombre de la regla |
| content | String | Sí | Contenido de la regla |
| priority | Integer | No | Prioridad de aplicación de la regla |
| enabled | Boolean | Sí | Si la regla está habilitada para esta instancia |
| version | Integer | Sí | Versión de la regla |
| createdAt | Timestamp | Sí | Fecha de creación |
| updatedAt | Timestamp | Sí | Fecha de última actualización |

**Cardinalidad:** Una AgentInstance puede tener cero o más AgentRules (1:N). Una AgentRules pertenece a exactamente una AgentInstance (N:1).

#### ConversationMemory
Memoria operativa por conversación, asociada a AgentInstance. Solo mensajes `user`/`assistant`, máximo 20, sin resumen automático (I14).

| Campo | Tipo | Obligatorio | Descripción |
|-------|------|-------------|-------------|
| id | UUID | Sí | Identificador único |
| agentInstanceId | UUID | Sí | Instancia propietaria |
| conversationId | String | Sí | Conversación de la memoria |
| workspaceId | String | Sí | Workspace asociado |
| messages | JSON[] | Sí | Últimos 20 mensajes user/assistant (rol, contenido, timestamp) |
| messageCount | Integer | Sí | Número de mensajes en memoria (máximo 20) |
| createdAt | Timestamp | Sí | Fecha de creación |
| updatedAt | Timestamp | Sí | Fecha de última actualización |
| expiresAt | Timestamp | No | Fecha de expiración (retención) |

**Cardinalidad:** Una ConversationMemory pertenece a una AgentInstance y una conversación (N:1:N). Una conversación tiene exactamente una ConversationMemory por AgentInstance.

#### AgentExecutionLog
Log de auditoría técnica por ejecución (I21). Registra eventos y detalles cronológicos vinculados a `AgentExecution`; no sustituye `AgentExecution.effectiveConfigSnapshot`, que es la fuente de verdad de las referencias inmutables de configuración y Knowledge de P10. Sin exponer modelos/versiones/parámetros al cliente.

| Campo | Tipo | Obligatorio | Descripción |
|-------|------|-------------|-------------|
| id | UUID | Sí | Identificador único |
| executionId | UUID | Sí | Referencia a AgentExecution |
| agentInstanceId | UUID | Sí | Instancia ejecutora |
| workspaceId | String | Sí | Workspace asociado |
| eventType | Enum | Sí | `execution_started` / `execution_completed` / `execution_failed` / `execution_cancelled` / `credits_deducted` |
| promptSnapshotHash | String | Sí | Hash del prompt usado |
| providerModelUsed | String | Sí | Proveedor y modelo usado |
| inputTokens | Integer | Sí | Tokens de entrada |
| outputTokens | Integer | Sí | Tokens de salida |
| totalCreditsDeducted | Decimal | Sí | Créditos descontados en este evento |
| latencyMs | Integer | Sí | Latencia en ms |
| result | String | No | Resultado normalizado (sin detalles internos) |
| error | String | No | Error si falló |
| createdAt | Timestamp | Sí | Timestamp del evento |

**Cardinalidad:** Una AgentExecution puede tener cero o más AgentExecutionLog (1:N).

#### OrganizationAICredit
Bolsa única de créditos por Organización (I17, I20). Saldo acumulado compartido por todos los Workspaces de la Organización. Una Organización tiene exactamente una OrganizationAICredit.

| Campo | Tipo | Obligatorio | Descripción |
|-------|------|-------------|-------------|
| id | UUID | Sí | Identificador único |
| organizationId | String | Sí | Organización propietaria (único) |
| balance | Decimal | Sí | Saldo actual de créditos disponibles |
| currency | String | Sí | Moneda (default USD) |
| totalPurchased | Decimal | Sí | Total de créditos comprados |
| totalConsumed | Decimal | Sí | Total de créditos consumidos |
| committedBudget | Decimal | No | Presupuesto total comprometido por autorizaciones en `authorized`, `executing`, `settling` o `reconciliation_required`; equivale a la suma de `authorizedBudget` de esas autorizaciones |
| createdAt | Timestamp | Sí | Fecha de creación |
| updatedAt | Timestamp | Sí | Fecha de última actualización |

**Cardinalidad:** Una Organización tiene exactamente una OrganizationAICredit (1:1). Todos los Workspaces de la Organización comparten el mismo saldo.

#### ExecutionBudgetAuthorization
Autorización temporal de presupuesto para una ejecución lógica. Representa la capacidad económica temporalmente reservada para una AgentExecution específica dentro del saldo de su Organización. Una ejecución lógica puede tener múltiples autorizaciones a lo largo de su ciclo (por reintentos o recuperación de worker caído), pero solo una con status `authorized`, `executing`, `settling` o `reconciliation_required` a la vez.

| Campo | Tipo | Obligatorio | Descripción |
|-------|------|-------------|-------------|
| authorizationId | UUID | Sí | Identificador único de la autorización |
| organizationId | String | Sí | Organización propietaria del saldo |
| workspaceId | String | Sí | Workspace de origen |
| agentInstanceId | UUID | Sí | Instancia que ejecuta |
| executionId | UUID | Sí | Ejecución lógica |
| attemptId | String | Sí | Identifica el intento/worker propietario; cambia al recuperar o reasignar la ejecución. No es una clave de idempotencia financiera |
| authorizedBudget | Decimal | Sí | Presupuesto máximo autorizado para esta ejecución |
| consumedAmount | Decimal | Sí | Usage del proveedor reconocido y liquidado internamente de las invocaciones resueltas, sea facturable al cliente o absorbido por Waby |
| activeAllocatedBudget | Decimal | Sí | Suma del `assignedBudget` de todas las asignaciones con `status` en `('assigned', 'executing', 'settling', 'reconciliation_required')` de esta autorización |
| fencingGeneration | Integer | Sí | Generación vigente de esta autorización; incrementa solo al recuperar/reasignar la propiedad a otro worker. La renovación normal del lease la conserva |
| status | Enum | Sí | `requested`, `authorized`, `executing`, `settling`, `settled`, `expired`, `cancelled`, `reconciliation_required` |
| leaseExpiresAt | Timestamp | Sí | Vencimiento de la autorización |
| pricingVersion | String | No | Versión de tarifas utilizada para el cálculo |
| createdAt | Timestamp | Sí | Fecha de creación |
| updatedAt | Timestamp | Sí | Fecha de última actualización |

**Cardinalidad:** Una Organización puede tener cero o más ExecutionBudgetAuthorization (1:N). Una ejecución lógica puede tener cero o más autorizaciones (1:N, por reintentos), pero solo una con status `authorized`, `executing`, `settling` o `reconciliation_required` a la vez.

**Restricciones de integridad:**
- La combinación `(organizationId, executionId)` con `status` en `('authorized', 'executing', 'settling', 'reconciliation_required')` aparece como máximo una vez (una ejecución no puede tener dos autorizaciones que retengan capacidad simultáneamente).
- `fencingGeneration` se incrementa únicamente al recuperar/reasignar la autorización a otro worker. La renovación normal del lease conserva la generación; un worker con generación obsoleta no puede liquidar ni ampliar presupuesto.
- `authorizedBudget` debe ser mayor a cero cuando `status` es `authorized` o `executing`.
- `consumedAmount` nunca debe superar `authorizedBudget`.
- `consumedAmount + activeAllocatedBudget` nunca debe superar `authorizedBudget`.
- `activeAllocatedBudget` es igual a la suma de `assignedBudget` de todas las asignaciones con `status` en `('assigned', 'executing', 'settling', 'reconciliation_required')` de esta autorización.

#### ExecutionBudgetAllocation
Asignación temporal de presupuesto para una invocación individual dentro de una `ExecutionBudgetAuthorization`. Representa la capacidad económica temporalmente asignada a una invocación concreta al proveedor LLM (incluidos reintentos contra el mismo proveedor/modelo) dentro del presupuesto de una ejecución lógica. Una ejecución puede tener múltiples asignaciones simultáneas para invocaciones paralelas, pero la suma de sus `assignedBudget` activos no puede superar el `authorizedBudget` de su autorización padre.

| Campo | Tipo | Obligatorio | Descripción |
|-------|------|-------------|-------------|
| allocationId | UUID | Sí | Identificador único de la asignación |
| authorizationId | UUID | Sí | Referencia a la `ExecutionBudgetAuthorization` padre |
| executionId | UUID | Sí | Ejecución lógica a la que pertenece |
| invocationId | String | Sí | Identificador de la invocación concreta al proveedor LLM, incluidos reintentos contra el mismo proveedor/modelo |
| attemptId | String | Sí | Identifica el intento/worker propietario de esta invocación; cambia al recuperar o reasignar la asignación |
| assignedBudget | Decimal | Sí | Presupuesto asignado a esta invocación; permanece inmutable como dato histórico después de su liquidación |
| consumedAmount | Decimal | No | Usage reconocido por el proveedor para esta invocación; se registra incluso cuando el consumo sea absorbido por Waby |
| fencingGeneration | Integer | Sí | Generación vigente de esta asignación; incrementa solo al recuperar/reasignar la propiedad a otro worker. La renovación normal del lease la conserva |
| status | Enum | Sí | `assigned`, `executing`, `settling`, `settled`, `expired`, `cancelled`, `reconciliation_required` |
| leaseExpiresAt | Timestamp | Sí | Vencimiento de la asignación |
| createdAt | Timestamp | Sí | Fecha de creación |
| updatedAt | Timestamp | Sí | Fecha de última actualización |

**Cardinalidad:** Una `ExecutionBudgetAuthorization` puede tener cero o más `ExecutionBudgetAllocation` (1:N). Una asignación pertenece a exactamente una autorización (N:1). `activeAllocatedBudget` en la autorización padre representa la suma de `assignedBudget` de las asignaciones con estado `assigned`, `executing`, `settling` o `reconciliation_required`.

**Restricciones de integridad adicionales:**
- La combinación `(authorizationId, invocationId)` con `status` en `('assigned', 'executing', 'settling', 'reconciliation_required')` aparece como máximo una vez (una invocación no puede tener dos asignaciones activas simultáneas).
- `fencingGeneration` se incrementa únicamente al recuperar/reasignar la asignación a otro worker. La renovación normal del lease conserva la generación; un worker con generación obsoleta no puede asignar ni liberar presupuesto para esa invocación.
- `activeAllocatedBudget` equivale a la suma de `assignedBudget` de las asignaciones con `status` en `('assigned', 'executing', 'settling', 'reconciliation_required')` y no puede superar `authorizedBudget - consumedAmount`.
- El presupuesto disponible para nuevas asignaciones dentro de una autorización se calcula como `authorizedBudget - consumedAmount - activeAllocatedBudget`. La comparación y la actualización del contador persistido se realizan atómicamente.

---

#### CreditLedger
Registro append-only de movimientos de créditos (P4). Idempotente por combinación `(organizationId, executionId)` para movimientos `deduct`. Los asientos nunca se modifican ni eliminan una vez registrados.

| Campo | Tipo | Obligatorio | Descripción |
|-------|------|-------------|-------------|
| id | UUID | Sí | Identificador único del movimiento (`movementId`) |
| organizationId | String | Sí | Organización propietaria de los créditos (ámbito contable) |
| workspaceId | String | Sí | Workspace que originó el movimiento (dimensión de atribución) |
| executionId | UUID | Sí | Referencia a AgentExecution (idempotency key para débitos) |
| refundRequestId | UUID | No | Para asientos `refund`: clave de idempotencia propia de la solicitud de devolución. Permite reintentar una devolución sin duplicarla. |
| movementId | UUID | Sí | Identificador único del asiento (generado al crear; para trazabilidad y auditoría) |
| originalMovementId | UUID | No | Para asientos `refund`: referencia directa al `movementId` del débito original compensado |
| originalExecutionId | UUID | No | Referencia complementaria al `executionId` asociado al débito original compensado |
| agentInstanceId | UUID | Sí | Instancia que generó el consumo |
| type | Enum | Sí | `deduct` / `refund` / `purchase` / `adjustment` |
| amount | Decimal | Sí | Cantidad de créditos |
| currency | String | Sí | Moneda (default USD) |
| description | String | Sí | Descripción del movimiento |
| status | Enum | Sí | `completed` / `failed` |
| createdAt | Timestamp | Sí | Fecha del movimiento |
| completedAt | Timestamp | No | Fecha de completación |

**Cardinalidad:** Una Organización puede tener cero o más CreditLedger (1:N). Para movimientos `deduct`, la combinación `(organizationId, executionId)` aparece como máximo una vez (idempotencia, P4). Para movimientos `refund`, la combinación `(organizationId, refundRequestId)` aparece como máximo una vez (idempotencia de la solicitud de devolución). Múltiples asientos `refund` pueden referenciar el mismo `originalMovementId` (devoluciones parciales o múltiples), y conservar el `originalExecutionId` como referencia complementaria. La suma de todos los refunds asociados a ese movimiento no puede superar su monto.

**Nota de integridad:** `CreditLedger` es append-only. Una reversión se representa como un asiento `refund` nuevo que referencia directamente al débito original vía `originalMovementId`; `originalExecutionId` se conserva como referencia complementaria. No existe `status: reversed` ni modificación de asientos existentes. Los movimientos `refund` y `purchase` no están sujetos a la restricción única de `(organizationId, executionId)`.

**Validación atómica de devoluciones:** Antes de registrar un asiento `refund`, el sistema verifica dentro de la misma transacción que `SUM(amount de refunds existentes para originalMovementId) + amount del nuevo refund <= amount del movimiento de débito original referenciado`. Esta verificación y la escritura ocurren en la misma transacción de MongoDB para prevenir sobre-devoluciones bajo concurrencia.

---

## B. Diagrama de entidades y relaciones (to-be)

```
WABY (PLATAFORMA)
  AgentDefinition
       │ 1:N
       ▼
WORKSPACE
  AgentInstance ───────────── AgentExecution
       │                           │
       ├── SkillConfiguration      └── AgentExecutionLog
       ├── AgentRules
       ├── ConversationMemory
       └── WorkspaceKnowledge (compartido por las instancias del Workspace)

ORGANIZACIÓN
  Organization
       ├── OrganizationAICredit (1:1)
       │        └── ExecutionBudgetAuthorization (1:N)
       │                 └── ExecutionBudgetAllocation (1:N)
       └── CreditLedger (append-only; workspaceId es dimensión de atribución)

CRM — AUTORIDAD DE ASIGNACIONES
  Contacto / ContactAssignment
       ├── EvAssigned (Evento discriminator; colección eventos)
       ├── Assignment Outbox (propagación recuperable)
       └── contactAssignmentHistory (compatibilidad legacy, si existe)

CRM — MENSAJERÍA ASÍNCRONA
  AgentMessageOutbox (CRM → AI Service)
  ExecutionResultInbox (AI Service → CRM; deduplicación durable)

AI SERVICE — MENSAJERÍA ASÍNCRONA
  AgentExecutionInbox (recepción durable)
       └── AgentExecution
              └── ExecutionResultOutbox (AI Service → CRM)
```

### Propiedad por capa

| Capa / dominio | Entidades | Propietario de negocio | Persistencia fuente de verdad |
|---|---|---|---|
| Waby (plataforma) | AgentDefinition y Prompt/PromptVersion asociados | Waby | AI Service DB, bajo control exclusivo Waby |
| Workspace | AgentInstance | Workspace | CRM DB |
| Workspace | WorkspaceKnowledge y metadatos de documentos | Workspace | AI Service DB; binarios en MinIO y vectores/índice derivados en Qdrant |
| Organización | OrganizationAICredit, CreditLedger, ExecutionBudgetAuthorization, ExecutionBudgetAllocation | Organización | CRM DB |
| AgentInstance | SkillConfiguration, AgentRules | AgentInstance | CRM DB |
| Conversación | ConversationMemory | AgentInstance + conversación | AI Service DB |
| Ejecución | AgentExecution, AgentExecutionLog | AgentInstance + Workspace | AI Service DB |
| Asignaciones CRM | Contacto/ContactAssignment, EvAssigned y Outbox de asignación | CRM | CRM DB |
| Proyección runtime | contact_assignments | Derivado de ContactAssignment CRM | AI Service DB; reconstruible desde CRM |
| Outbox de mensaje de origen CRM → AI Service (AgentMessageOutbox) | CRM | CRM | CRM DB | Mensaje de origen e intención de despacho se escriben en una misma unidad atómica; distinto del Outbox de asignación de D.1.8. |
| Inbox de ejecución (AgentExecutionInbox) y Outbox de resultado (ExecutionResultOutbox) | AI Service | AI Service | AI Service DB | Recepción durable/deduplicación vinculada a AgentExecution y publicación recuperable del resultado. |
| Inbox de resultado CRM ← AI Service (ExecutionResultInbox) | CRM | CRM | CRM DB | Recibo durable idempotente por eventId/executionId; Redis puede ser caché, no autoridad de deduplicación. |

### Restricciones de integridad

1. **No hay unicidad `workspaceId` + `agentDefinitionId`** (I16): múltiples instancias de una misma definición en un Workspace.
2. **Cantidad de instancias:** no existe cupo comercial por plan, Workspace u Organización (P1 aprobado). La cantidad de instancias es ilimitada comercialmente; el uso está condicionado por créditos organizacionales, entitlements y límites técnicos independientes.
3. **AgentInstance.workspaceId** es denormalizado para consultas rápidas; validado cross-tenant en escritura.
4. **WorkspaceKnowledge.workspaceId** — todas las instancias del mismo workspace comparten el mismo knowledge (I11).
5. **SkillConfiguration.agentInstanceId** y **AgentRules.agentInstanceId** — exclusivos por instancia, sin herencia (I12, I13).
6. **CreditLedger:** para movimientos `deduct`, la combinación `(organizationId, executionId)` es única (idempotencia, P4). Los movimientos `refund` y `purchase` no están sujetos a esta restricción única; varios refunds pueden asociarse al mismo `originalMovementId` para devoluciones parciales, conservando `originalExecutionId` como correlación de negocio. `originalMovementId` referencia directamente el movimiento de débito original en asientos `refund`; `originalExecutionId` permite asociarlo además con la ejecución de negocio.
7. **AgentExecution.id** = execution_id, usado como idempotency key en CreditLedger para movimientos `deduct` (P4).
8. **CreditLedger no tiene estado `reversed` ni `pending`:** los asientos son `completed` o `failed`. Una reversión es un asiento `refund` nuevo que referencia al débito original vía `originalMovementId`, conservando `originalExecutionId` como dato complementario.
9. **ExecutionBudgetAuthorization:** la combinación `(organizationId, executionId)` con `status` en `('authorized', 'executing', 'settling', 'reconciliation_required')` aparece como máximo una vez (una ejecución no puede tener dos autorizaciones que retengan capacidad simultáneamente). `fencingGeneration` se incrementa únicamente al recuperar/reasignar la autorización a otro worker; la renovación normal del lease conserva la generación.
10. **ExecutionBudgetAllocation:** la combinación `(authorizationId, invocationId)` con `status` en `('assigned', 'executing', 'settling', 'reconciliation_required')` aparece como máximo una vez (una invocación no puede tener dos asignaciones que retengan capacidad simultáneamente). `fencingGeneration` se incrementa únicamente al recuperar/reasignar la asignación a otro worker; la renovación normal del lease conserva la generación. `activeAllocatedBudget` debe coincidir con la suma de `assignedBudget` de las asignaciones con esos mismos estados y nunca puede superar `authorizedBudget` menos `consumedAmount`.
11. **OrganizationAICredit.committedBudget** debe ser siempre igual a la suma de `authorizedBudget` de todas las `ExecutionBudgetAuthorization` con status `authorized`, `executing`, `settling` o `reconciliation_required` para esa organización.

---

## C. Diseño del provisioning

### C.1 Asignación de instancias a Workspaces

Waby administra el catálogo y el provisioning técnico de plataforma. La creación y administración lógica de AgentInstance corresponden a usuarios humanos autorizados del Workspace, mediante operaciones CRM que validan membership, permisos, ownership y políticas de Waby (ARQ-19/20). El cliente nunca crea ni modifica AgentDefinition.

### C.2 Cantidad de AgentInstance y planes (P1 — decisión resuelta)

**Decisión vigente:** no existe límite comercial de cantidad de AgentInstance por Organización ni por Workspace. Los planes pueden diferenciar créditos incluidos, funcionalidades y capacidades, pero no establecen cupos comerciales de instancias.

El uso de IA queda condicionado por la bolsa única de créditos de la Organización definida en P4. Los límites técnicos de concurrencia, rate limits, almacenamiento y protección operativa son independientes de la cantidad comercial de agentes.

Quedan sustituidas y sin vigencia las reglas anteriores de cupos de una/dos instancias, exceso por downgrade, período de gracia de cinco días, selección automática de instancias excedentes y congelamiento de criterios durante esa gracia. No implementar `retirement_pending` como consecuencia de un downgrade comercial.

### C.3 Creación y administración de instancias

El modelo de autoridad es híbrido por operación (ARQ-19):

1. Waby publica y mantiene AgentDefinitions y el catálogo central.
2. Un usuario humano autorizado del Workspace crea una AgentInstance a partir de una definición publicada.
3. CRM valida membership, permiso `wrk.agents.*`, ownership del Workspace y estado de la definición.
4. CRM persiste la AgentInstance y su ownership `AgentInstance → Workspace → Organización`.
5. El usuario autorizado configura los parámetros empresariales permitidos, Skills/Rules habilitadas, integraciones permitidas y `additionalPrompt`.
6. Waby conserva bajo su control el provisioning técnico, proveedor/modelo/credenciales LLM, políticas obligatorias y operaciones de plataforma.
7. La creación y los cambios relevantes quedan auditados.

El Administrador de Organización no obtiene por ese solo rol permisos para administrar AgentInstances. Los servicios son ejecutores técnicos, no sustitutos del actor de negocio.

### C.4 Múltiples instancias de una misma definición

Se permiten múltiples AgentInstances derivadas de una misma AgentDefinition, incluso dentro del mismo Workspace. Cada AgentInstance mantiene su configuración empresarial y su propio estado. No existe routing persistente por canal ni asociación permanente canal → instancia. La selección de instancia se resuelve mediante ContactAssignment y, cuando el contacto no tiene asignación humana ni asignación a una AgentInstance, mediante la instancia predeterminada del Workspace conforme a DEC-CM-04/06/08.

### C.5 Concurrencia y restricciones de unicidad

- **Default por Workspace:** como máximo una AgentInstance puede tener `isDefault=true` por Workspace.
- **Cambio atómico:** al designar una nueva instancia predeterminada, el cambio debe preservar la unicidad dentro de una única transición lógica.
- **Asignación:** CRM es autoridad única de ContactAssignment; Contacto y AgentInstance deben pertenecer al mismo Workspace.
- **Concurrencia:** las operaciones de creación, cambio de default, asignación y transición de estado deben ser idempotentes y seguras ante concurrencia.
- **Sin cupo comercial:** no se crean contadores, índices ni bloqueos para imponer cupos 1/2 por plan.

### C.6 Suspensión, reactivación, retiro y eliminación

| Operación | Autoridad | Efecto |
|---|---|---|
| Crear/configurar AgentInstance | Usuario autorizado del Workspace; CRM valida y persiste | Crea o actualiza configuración empresarial dentro de la definición publicada y los límites de Waby |
| Provisioning técnico | Waby | Prepara recursos técnicos y aplica políticas de plataforma |
| Activar/suspender/reanudar | Usuario autorizado del Workspace, sujeto a políticas y validaciones de Waby | Cambia el ciclo de vida lógico |
| Retiro lógico | Usuario autorizado del Workspace | Deja de recibir asignaciones y ejecuciones conforme al contrato de retiro |
| Purga física | Waby bajo política | Elimina recursos físicos y dependencias conforme a retención/auditoría |
| Eliminación de Workspace | CRM coordina el ciclo de vida; servicios ejecutan limpieza de sus recursos | Purga coordinada, idempotente y auditable |

**Regla aprobada DEC-CM-07:** al suspender una AgentInstance con contactos asignados, se conservan los ContactAssignment activos. La instancia suspendida no ejecuta; esos contactos quedan sin atención automática. Al reactivarse esa misma instancia, puede reanudar la atención de sus contactos asignados. También se permite una reasignación manual explícita por un usuario autorizado. No se libera la asignación ni se aplica fallback automático a otra instancia.

La suspensión no equivale al retiro ni a la purga física. La eliminación de una AgentInstance no elimina Knowledge compartido del Workspace.

**Estados de referencia:** `draft`, `active`, `suspended`, `retirement_pending`, `retired`. Los detalles de reversibilidad de `retirement_pending`, quién puede cancelarlo y las condiciones de purga física deben permanecer como decisiones pendientes si no están resueltos en Linear. No asociar `retirement_pending` a downgrade o exceso de cupo.

### C.7 Consistencia de configuración (P10) — Instantánea inmutable por ejecución y trazabilidad

**Mecanismo aprobado (P10):** instantánea inmutable de configuración efectiva al comenzar cada ejecución.

- Al iniciar una ejecución, el sistema crea una **instantánea inmutable** de la configuración efectiva: AgentDefinition (versión activa), Rules, Skills, capabilities, knowledgeBaseIds y parámetros de ejecución de la AgentInstance.
- La ejecución en curso conserva sus definiciones hasta finalizar. Los cambios posteriores en AgentDefinition, Rules, Skills o capabilities se aplican únicamente a las **nuevas ejecuciones**, nunca a la ejecución en curso.
- Cambio en AgentDefinition: la instancia mantiene una referencia estable y las nuevas ejecuciones resuelven la última versión publicada (DEC-CM-01). El cambio es explícito, autorizado y auditado; si la nueva versión es incompatible con la configuración de una instancia, esa instancia se suspende hasta que un usuario autorizado la revise y corrija (DEC-CM-05). Los cambios relevantes se notifican a los Workspaces afectados.
- Proveedor, modelo, credenciales y parámetros técnicos base son administrados por Waby. La allowlist de parámetros configurables por AgentInstance queda vacía por defecto; no se expone un objeto libre de configuración del proveedor (DEC-CM-03).
- Cambio en SkillConfiguration o AgentRules → evento `agent_instance.updated` → afecta solo a nuevas ejecuciones; las ejecuciones en curso mantienen su instantánea.
- Sin flujo draft/release para el cliente (I17). Los cambios de Waby se propagan inmediatamente a nuevas ejecuciones.

**No se incluyen secretos ni valores sensibles en el snapshot.** Las credenciales se referencian por ID de versión, nunca se copian su valor al snapshot.

---

#### C.7.1 Dos hashes diferenciados para trazabilidad

Se definen dos hashes con propósitos distintos:

| Hash | Qué identifica | Campos que participan | Normalización antes de SHA-256 |
|------|---------------|----------------------|-------------------------------|
| `promptSnapshotHash` | Prompt efectivo construido | systemPrompt, Rules contenido, Skills contenido, knowledgeBaseIds (IDs solo, no contenido), mensajes de memoria, historial | Serialización JSON ordenada por clave, sin espacios, sin secretos |
| `effectiveConfigHash` | Configuración efectiva completa | AgentDefinition.id + version, agentInstanceId, skill IDs + versiones, rule IDs + versiones, capability IDs, provider, model, temperatura, maxTokens, knowledgeBaseIds | Serialización JSON ordenada por clave, sin espacios, sin secretos, sin credenciales |

**Nota importante:** Ningún hash por sí solo permite reconstruir una ejecución. `promptSnapshotHash` identifica el prompt; `effectiveConfigHash` identifica la configuración. Cada ejecución debe conservar `effectiveConfigSnapshot` con las referencias identificables y versionadas obligatorias. El acceso posterior a las versiones inmutables permite resolver esas referencias, pero no sustituye la persistencia del snapshot en la ejecución.

---

#### C.7.2 Configuración inmutable: snapshot persistido vs referencias mutables

El snapshot **no es solo una lista de IDs** que se vuelven a resolver contra registros mutables. Cada ejecución conserva referencias estables y versionadas de la configuración y del contexto realmente utilizados, de modo que los cambios posteriores no alteren la evidencia histórica.

`effectiveConfigSnapshot` es obligatorio y contiene, como mínimo:

- AgentDefinition: `id` y versión publicada.
- Skills y configuración efectiva: IDs y versiones de Skill y SkillConfiguration.
- Rules: IDs y versiones de las reglas aplicadas.
- Capabilities: identificadores y versión/revisión cuando el catálogo la exponga.
- Knowledge: IDs de las Knowledge Bases habilitadas, documentos recuperados con su versión y fragmentos efectivamente incorporados al contexto con sus IDs, versión/revisión disponible y hash de contenido.
- Referencias de proveedor/modelo y parámetros técnicos efectivos no secretos.

El snapshot puede incluir además una copia de los valores efectivos (por ejemplo, texto de Rules/Skills y prompt) para mejorar la reproducibilidad técnica. Esa copia adicional no sustituye las referencias identificables/versionadas obligatorias ni autoriza almacenar secretos.

---

#### C.7.3 Trazabilidad del contexto RAG

`knowledgeBaseIds` identifica las bases de Knowledge habilitadas para la instancia, pero **no** registra el contenido efectivamente recuperado durante la ejecución.

Para auditoría de qué contexto recibió la ejecución, se registra obligatoriamente en `AgentExecution.effectiveConfigSnapshot`. `AgentExecutionLog` puede registrar eventos y referencias de auditoría adicionales, pero no sustituye el snapshot obligatorio ni constituye una ubicación alternativa para sus referencias de Nivel 1 y Nivel 2:

- IDs de las fuentes/documentos recuperados de Qdrant y su versión publicada.
- IDs de los fragmentos efectivamente incorporados al prompt, con su versión/revisión disponible.
- Hashes de contenido de esos fragmentos para detectar cambios entre ejecuciones.
- Metadatos suficientes para identificar y auditar el contexto, sin duplicar indiscriminadamente todo el contenido sensible.

Si una fuente o fragmento no dispone de versionado propio, se conserva su identificador estable y el hash del contenido utilizado; no se debe omitir la referencia por falta de un campo de versión.

**Diferenciación:**
- **Configuración de Knowledge habilitada:** `knowledgeBaseIds` en el snapshot (qué bases estaban disponibles).
- **Contexto recuperado durante la ejecución:** IDs de fragmentos obtenidos de Qdrant en esa ejecución específica.
- **Contenido efectivamente incorporado al prompt:** Qué fragmentos llegaron al `promptSnapshotHash` (implícito en el hash, no se duplica el contenido).

---

#### C.7.4 Tres niveles de garantía de trazabilidad y reproducibilidad

| Aspecto | Nivel 1: Reconstrucción de configuración | Nivel 2: Trazabilidad del contexto | Nivel 3: Reproducción de ejecución |
|---------|------------------------------------------|--------------------------------------|------------------------------------|
| Qué se almacena | AgentDefinition.id+version, Rule.id+version, Skill.id+version, SkillConfiguration.id+version, capability IDs/versiones disponibles, provider/model, parámetros y knowledgeBaseIds | IDs/versiones de documentos y chunks recuperados; IDs y hashes de los fragmentos efectivamente incorporados al contexto | Copia adicional de valores efectivos (sin secretos), cuando se requiera mayor reproducibilidad |
| Puede reconstruir el prompt exacto | ❌ No, por referencias solamente | ❌ No, por referencias solamente | Puede reconstruir las condiciones de entrada; no garantiza la salida exacta |
| Puede reproducir respuesta LLM exacta | ❌ No | ❌ No | ❌ No (modelos no deterministas) |
| Costo de almacenamiento | Bajo | Medio | Mayor |
| Uso recomendado | Auditoría obligatoria de configuración por ejecución | Auditoría obligatoria del contexto RAG utilizado | Reproducción técnica adicional de condiciones de entrada (P7) |

**Decisión de diseño:** Los niveles 1 y 2 son obligatorios para cada ejecución y sus referencias se conservan en `AgentExecution.effectiveConfigSnapshot`, que es la fuente de verdad del snapshot inmutable. `AgentExecutionLog` registra eventos cronológicos y puede incluir referencias al snapshot, pero no lo sustituye ni almacena una copia alternativa de sus referencias obligatorias. La copia completa de valores efectivos del Nivel 3 es complementaria y puede optimizarse por almacenamiento, pero nunca se degradan ni omiten las referencias versionadas de configuración, documentos y fragmentos realmente utilizados.

**Limitación importante:** Ningún nivel garantiza reproducibilidad exacta de la respuesta LLM. Los modelos de lenguaje son no deterministas, los proveedores pueden cambiar sus modelos, y las herramientas externas pueden devolver resultados diferentes. El snapshot garantiza trazabilidad de la configuración de entrada (Nivel 1) y del contexto (Nivel 2), no reproducibilidad de la salida (Nivel 3 solo reproduce condiciones de entrada, no el resultado exacto).

**Alternativa considerada y descartada:** Relectura por cada paso del pipeline (riesgo de inconsistencia si la configuración cambia a mitad de ejecución). Invalidación de caché sin instantánea (riesgo de que una ejecución en curso use configuración nueva a mitad de su procesamiento).

---

## D. Contratos y autorización

### D.1 Responsabilidades del CRM y AI Service

La tabla siguiente describe la superficie de gestión y las responsabilidades funcionales. La asignación inequívoca de persistencia por entidad se fija en D.1.9; no inferir persistencia a partir de quién expone el resolver o administra la operación.

| Responsabilidad | CRM (Backend) | AI Service | Observaciones |
|-----------------|---------------|------------|---------------|
| Gestión de Workspaces y Organizaciones | ✅ | ❌ | Tenant → Workspace → Permission → Hierarchy → Operation |
| Gestión de AgentDefinition | ❌ (solo lectura) | ✅ (Waby-only) | Crear/editar/versionar/retirar definiciones |
| Gestión de AgentInstance | ✅ (autoridad de dominio y persistencia) | Ejecuta runtime/proyecciones derivadas | Usuarios autorizados del Workspace administran la instancia; Waby controla definición, políticas y provisioning técnico. |
| Gestión de Knowledge (KB + documentos) | ✅ (API, autorización y fachada) | ✅ (persistencia y procesamiento) | Workspace es propietario; AI Service DB conserva la fuente de verdad de WorkspaceKnowledge/documentos; Qdrant es índice derivado y MinIO almacena binarios. |
| Gestión de Skills/Rules por instancia | ✅ (permiso `wrk.agents.*`) | ❌ | Configuración empresarial dentro de allowlist y políticas Waby |
| Ejecución de agente (pipeline LLM) | ❌ | ✅ | Resolver instancia, cargar contexto, invocar proveedor |
| Asignación efectiva de Contacto (`assignedType`, `ref_user_assigned`, `ref_agent_assigned`) | ✅ (autoridad única) | ❌ (proyección operativa `contact_assignments` solo para runtime) | CRM decide quién tiene asignado el contacto. `ref_agent_assigned` es referencia tipada a `AgentInstance._id`. Sin `assignmentVersion`. |
| Historial funcional de asignaciones (`EvAssigned`) | ✅ (exclusivo en CRM, colección `eventos`) | ❌ (no se sincroniza) | Registra transiciones confirmadas de asignación. Identidad única de EvAssigned autoriza cada ejecución (resolución ABA). |
| Cálculo y descuento de créditos | ✅ (persistencia y autoridad transaccional de saldo, ledger y autorizaciones) | ✅ (ejecuta invocaciones y reporta usage) | OrganizationAICredit, CreditLedger, ExecutionBudgetAuthorization y ExecutionBudgetAllocation persisten en CRM. ai-service no es autoridad de saldo ni autoriza presupuesto por cuenta propia. |
| Auditoría por ejecución | ✅ (auditoría de negocio/financiera separada) | ✅ (fuente de verdad de AgentExecution y AgentExecutionLog) | CRM conserva CreditLedger y referencias financieras; no duplica el log operativo de ejecución. |
| Eliminación de Workspace | ✅ (delete-wrk extendido) | ✅ (limpia ai_service) | Knowledge, vectores, media, ejecuciones |

#### D.1.5 Modelo unificado de Contacto (decisión aprobada — WAB-78)

CRM mantiene el modelo unificado de asignación de Contacto con los campos `assignedType`, `ref_user_assigned` y `ref_agent_assigned`.

**Enum aprobado:**

```typescript
enum AssignedType {
  HUMAN = 'human',
  AGENT = 'agent',
  UNASSIGNED = 'unassigned'
}
```

**Invariantes:**

| assignedType | ref_user_assigned | ref_agent_assigned |
|---|---|---|
| `human` | UOW obligatorio | null |
| `agent` | null | AgentInstance obligatoria (`AgentInstance._id`) |
| `unassigned` | null | null |

**Reglas:**
- `ref_agent_assigned` es una referencia tipada a `AgentInstance._id` en CRM.
- Ownership: `AgentInstance → Workspace → Organización`.
- La asignación a agent solo se permite cuando contacto e instancia pertenecen al mismo Workspace.
- No introducir `assignmentVersion` en Contacto.
- El modelo completo de AgentInstance (campos, índices, ciclo de vida) corresponde a WAB-79. No diseñar ni implementar ahora toda esa entidad.

**Mapeo de migración:**
- UOW presente → `human`
- UOW nulo → `unassigned`
- `ref_agent_assigned` nulo para todos los contactos existentes

#### D.1.6 EvAssigned — historial funcional de asignaciones (decisión aprobada — WAB-78)

`EvAssigned` se implementa como discriminator de `Evento` en la colección `eventos` existente, exclusivamente en CRM.

**Finalidad:** Registrar las transiciones confirmadas de asignación y conservar el historial funcional en CRM.

**Debe permitir conocer:**
- Responsable anterior y nuevo.
- Tipo de cada responsable (`AssignedType` aprobado).
- Actor que produjo la transición.
- Operación que la originó (`operationType`, propuesta pendiente — separado de `AssignedType`).
- Motivo, cuando corresponda.
- Mensaje de origen, cuando corresponda (`originMessageId` para trazabilidad/idempotencia, no sustituye la identidad de EvAssigned).
- Cadena de delegación, cuando corresponda.
- Momento de confirmación.

**Decisiones:**
- `EvAssigned` se persiste exclusivamente en CRM.
- No se sincroniza el historial de `EvAssigned` con ai-service.
- No se reconstruye retrospectivamente el historial anterior.
- Cada transición confirmada tiene un identificador único.
- La identidad de `EvAssigned` se utiliza para identificar la asignación que autorizó una ejecución (resolución ABA).
- `operationType` es una propuesta pendiente de aprobación, no una decisión aprobada por el usuario.

#### D.1.7 Resolución de ABA con identidad de EvAssigned (decisión aprobada — WAB-78)

No introducir `assignmentVersion` en Contacto.

La identidad única de `EvAssigned` debe identificar qué transición autorizó cada ejecución:

| Transición | Responsable | EvAssigned |
|---|---|---|
| Recepción de M1 | Agent A | E1 |
| Takeover humano | Human | E2 |
| Desasignación humana | Unassigned | E3 |
| Recepción de M2 | Agent A | E4 |

La ejecución originada por M1 conserva E1. Aunque Agent A vuelva a ser responsable mediante E4, la ejecución antigua de M1 no recupera autorización.

`originMessageId` sirve para trazabilidad e idempotencia del mensaje de origen; no sustituye la identidad de la transición `EvAssigned`.

#### D.1.8 Atomicidad y persistencia

**Requisito funcional:** los cambios de Contacto, `EvAssigned` y Outbox deben preservar una única transición lógica consistente. No deben quedar estados parciales que permitan una asignación efectiva sin su historial correspondiente o sin el mecanismo de propagación recuperable.

**Solución técnica preferida:** actualizar Contacto, persistir `EvAssigned` y registrar Outbox dentro de una misma transacción MongoDB.

**Validación pendiente:** todavía no se verificó la compatibilidad de esa solución con la infraestructura real de MongoDB Atlas, incluyendo soporte de transacciones, sesiones y compatibilidad de las colecciones involucradas.

Las llamadas HTTP a ai-service deben producirse después del commit mediante Outbox. No mantener transacciones abiertas durante operaciones de red.

Si la transacción no resulta viable, deberá diseñarse y aprobarse una alternativa recuperable que garantice consistencia, idempotencia y reconciliación. Esa alternativa es trabajo futuro sujeto a diseño y aprobación; no se diseña en esta fase.

`EvAssigned` nunca se envía como historial a ai-service. Si ai-service necesita conocer el identificador de la transición que autoriza una ejecución para validar ABA, puede recibir únicamente ese identificador como metadata de la proyección operativa. Eso no constituye sincronizar el historial funcional.

#### D.1.9 Matriz normativa de ownership y persistencia (ARQ-14)

La propiedad de negocio, la superficie de API y la persistencia son dimensiones distintas. Esta matriz fija el servicio que conserva la fuente de verdad de cada entidad del modelo objetivo. La presencia de una fachada GraphQL en CRM no implica que CRM persista la entidad; una proyección o índice derivado tampoco se convierte en fuente de verdad.

| Entidad / recurso | Propietario de negocio | API / gestión | Persistencia fuente de verdad | Copias derivadas y límites |
|---|---|---|---|---|
| AgentDefinition, Prompt, PromptVersion asociados | Waby | Operaciones Waby-only en AI Service; CRM solo consulta/consume | **AI Service DB** | CRM no mantiene copia editable. Son definiciones globales, sin orgId ni workspaceId. |
| AgentInstance | Workspace | CRM GraphQL y autorización CRM | **CRM DB** | AI Service consume la configuración necesaria para runtime; cualquier caché/proyección es descartable y no editable como autoridad. |
| WorkspaceKnowledge, KnowledgeDocument y metadatos de ingestión | Workspace | CRM expone la API y valida permisos/ownership; AI Service ejecuta ingestión/procesamiento | **AI Service DB** (knowledge_bases y documentos equivalentes) | Se conserva la persistencia actual en AI Service, evitando una migración de autoridad no acordada. MinIO almacena binarios; Qdrant y chunks/embeddings son artefactos derivados reconstruibles. |
| SkillConfiguration, AgentRules | AgentInstance | CRM GraphQL, bajo wrk.agents.* | **CRM DB** | AI Service los consume para componer la configuración efectiva; no mantiene una copia editable paralela. |
| ConversationMemory | AgentInstance + conversación | AI Service runtime | **AI Service DB** | No se duplica como memoria operativa en CRM. El historial conversacional de CRM, si existe, es un recurso distinto. |
| AgentExecution, AgentExecutionLog | AgentInstance + Workspace | AI Service runtime | **AI Service DB** | CRM recibe eventos/resultados y mantiene sus registros de negocio/financieros propios; no replica el log operativo como segunda autoridad. |
| OrganizationAICredit, CreditLedger, ExecutionBudgetAuthorization, ExecutionBudgetAllocation | Organización | CRM y sus operaciones financieras | **CRM DB** | AI Service solicita autorización y reporta usage; no persiste ni modifica el saldo/ledger como autoridad. |
| Contacto, ContactAssignment, EvAssigned, Outbox de asignación | CRM / Workspace según ownership del recurso | CRM | **CRM DB** | EvAssigned no se sincroniza como historial. La proyección contact_assignments de AI Service es derivada. |
| contact_assignments | Derivado de ContactAssignment | AI Service runtime | **AI Service DB**, como proyección | No es fuente de verdad; debe poder reconciliarse/reconstruirse desde CRM. |
| AgentMessageOutbox | CRM | CRM | **CRM DB** | Mensaje de origen e intención de despacho CRM → AI Service en una misma unidad atómica; distinto del Outbox de asignación. |
| AgentExecutionInbox | AI Service | AI Service | **AI Service DB** | Recepción durable/idempotente vinculada a un único AgentExecution y originMessageId. |
| ExecutionResultOutbox | AI Service | AI Service | **AI Service DB** | Intención durable de publicar el resultado terminal; se confirma solo tras confirmación y enrutamiento válido del broker. |
| ExecutionResultInbox | CRM | CRM | **CRM DB** | Recibo durable idempotente por eventId/executionId; Redis puede ser caché, no autoridad de deduplicación. |
| Vectores, embeddings e índice de Knowledge | Derivado de WorkspaceKnowledge | AI Service ingestión/RAG | **Qdrant** (índice vectorial) y almacenamiento técnico de AI Service para chunks/metadatos de ingestión | No son fuente de verdad del documento ni de su ownership; pueden regenerarse desde el documento fuente. |
| Binarios de documentos Knowledge | Workspace | CRM autoriza; AI Service procesa | **MinIO** (object storage) | El registro de propiedad/estado del documento permanece en AI Service DB; MinIO no es autoridad de dominio. |
| Credenciales de proveedores LLM | Waby | Gestión exclusiva Waby | **Almacén seguro de secretos administrado por Waby**, fuera de las colecciones de dominio CRM/AI Service | Solo referencias/IDs versionados pueden aparecer en snapshots; nunca secretos. |

**Consecuencias de la frontera:**

1. CRM es autoridad de AgentInstance, SkillConfiguration, AgentRules, ContactAssignment y del ledger/créditos. AI Service es autoridad de AgentDefinition, Knowledge técnico, ConversationMemory y estado/log operativo de AgentExecution.
2. WorkspaceKnowledge tiene propietario de negocio Workspace y API/controles de acceso en CRM, pero su persistencia canónica se conserva en AI Service DB. Esta separación explícita mantiene el patrón existente de fachada CRM y lectura/procesamiento AI Service; no autoriza dos colecciones editables para la misma entidad.
3. Los vectores Qdrant, chunks y contact_assignments son proyecciones/artefactos derivados. Deben poder reconstruirse desde su fuente de verdad y no pueden decidir ownership, permisos ni asignación.
4. AgentExecution en AI Service y los movimientos CreditLedger en CRM son entidades distintas: la primera registra el ciclo técnico; los segundos son la autoridad financiera. La correlación se hace por executionId, organizationId y workspaceId, sin duplicar un ledger en AI Service.
5. La persistencia de Contacto + EvAssigned + Outbox de asignación en CRM queda sujeta al requisito de atomicidad de D.1.8 y a la verificación de Atlas; ARQ-14 no convierte esa verificación de infraestructura (ARQ-06) en una decisión pendiente de ownership.

### D.2 Operaciones GraphQL/HTTP necesarias

#### Operaciones de plataforma Waby

| Operación | Autoridad | Límite |
|---|---|---|
| Crear, editar, versionar y retirar AgentDefinition | Waby | Solo catálogo central |
| Administrar proveedores, modelos y credenciales LLM | Waby | No configurable libremente por Workspace |
| Administrar políticas obligatorias | Waby | No sobrescribibles por cliente ni LLM |
| Provisioning técnico y purga física | Waby | Bajo política y auditoría |

#### Operaciones de Workspace

| Operación | Permiso/autoridad |
|---|---|
| Consultar AgentInstances del Workspace | `wrk.agents.read` |
| Crear/configurar/activar/suspender/reanudar/retiro lógico de AgentInstance | `wrk.agents.*`, con validación contextual CRM |
| Configurar Skills, Rules, `additionalPrompt` e integraciones permitidas | `wrk.agents.*`, sujeto a allowlist y políticas Waby |
| Asignar, reasignar o liberar contactos manualmente | `wrk.workspaces.update` |
| Gestionar Knowledge | Permisos de Knowledge existentes; no se sustituyen por `wrk.agents.*` |

#### Operaciones internas CRM ↔ ai-service

Las operaciones internas usan el contrato ARQ-21: JWT de servicio firmados RS256, credenciales diferenciadas por dirección (CRM → ai-service y ai-service → CRM), token de corta duración por petición y scopes mínimos por operación. Claims base: iss, sub, aud, iat, nbf, exp, jti, scope, orgId, workspaceId, agentInstanceId e initiatedBy; la obligatoriedad de cada claim depende del flujo. El receptor valida firma y algoritmo, issuer, subject esperado, audience, vigencia, scope y contexto tenant/recurso; CRM vuelve a validar ownership y estado actuales. initiatedBy solo aporta auditoría, no es principal ni autorización. Rotación/revocación de claves, anti-replay, autenticación RabbitMQ, inventario exhaustivo de operaciones/scopes y pruebas negativas son endurecimiento operativo de ARQ-21, diferido a WAB-80/WAB-81. La entrega durable e idempotencia del flujo RabbitMQ se especifican separadamente en D.6 (ARQ-22).

La asignación inicial automática es una operación interna autenticada, validada y auditada de CRM. ai-service puede mantener una proyección operativa derivada de ContactAssignment, pero no una autoridad editable paralela.

**Permisos ARQ-20:** declarar explícitamente `wrk.agents.*` y `wrk.agents.read` en `permission.enum.ts`. Workspace Owner hereda `wrk.agents.*` desde `wrk.*.*`; Workspace Manager recibe `wrk.agents.*` explícito; Workspace Agent recibe `wrk.agents.read`. No introducir `bot.configure`, `wrk.agent.use`, `wrk.credits.read` ni permisos `platform.*` como permisos de administración de AgentInstance.

### D.3 Validación orgId/workspaceId

**En toda operación de escritura sobre recursos de agente:**

1. El contexto JWT resuelve `orgId` y `workspaceId` (x-org-id, x-wrk-id headers).
2. `assertResourceOwnership` (T3) valida que el recurso pertenece al workspace del contexto.
3. Para AgentInstance: `workspaceId` de la instancia debe coincidir con `x-wrk-id` del contexto.
4. Para WorkspaceKnowledge: `workspaceId` debe coincidir con `x-wrk-id` del contexto.
5. Para AgentDefinition: propiedad de Waby, no se valida contra workspace del cliente.

**En el punto de entrada de ejecución (RabbitMQ):**
- `workspaceId` del payload debe validarse contra el `workspaceId` de la AgentInstance resuelta.
- Si no coinciden, rechazar con `FORBIDDEN` (corregir hallazgo de Fase 1: workspaceId no validado contra el bot).

### D.4 Separación Waby/Workspace

| Operación | Waby | Workspace |
|---|---|---|
| AgentDefinition y catálogo central | Administra y publica | Solo consume definiciones publicadas |
| Proveedor/modelo/credenciales LLM | Administra exclusivamente | No configura credenciales ni parámetros técnicos libres |
| Políticas obligatorias | Administra y aplica | No puede sobrescribirlas |
| AgentInstance y configuración empresarial | Control de plataforma, políticas y provisioning técnico | Usuarios autorizados administran ciclo de vida lógico y configuración permitida |
| ContactAssignment | CRM conserva autoridad de dominio | Usuarios autorizados operan con `wrk.workspaces.update` |
| Knowledge | Mantiene controles de plataforma | Se administra con permisos de Knowledge existentes |

### D.5 Permisos de AgentInstance — ARQ-20

La administración de AgentInstance utiliza `wrk.agents.*`; la lectura utiliza `wrk.agents.read`. Los permisos se declaran explícitamente en `permission.enum.ts`. Workspace Owner hereda `wrk.agents.*` mediante `wrk.*.*`; Workspace Manager recibe `wrk.agents.*` explícito; Workspace Agent recibe `wrk.agents.read`.

No existe un permiso separado `bot.configure` para Skills/Rules de instancia. `wrk.agents.*` no autoriza modificar AgentDefinitions, políticas ni recursos centrales de Waby, y no sustituye membership, ownership ni autorización contextual de Capabilities. La asignación manual de contactos conserva el permiso `wrk.workspaces.update`.

### D.6 Contrato de entrega durable e idempotencia del flujo asíncrono (ARQ-22)

#### D.6.1 Alcance y garantía

Este contrato cubre los dos tramos del ciclo de ejecución: CRM publica un mensaje de origen para iniciar la ejecución en AI Service; AI Service publica el resultado terminal para que CRM lo procese.

La garantía es **entrega al menos una vez (at-least-once) con consumidores idempotentes y recuperación durable**. No se promete exactly-once a través de MongoDB, RabbitMQ, proveedores LLM y efectos externos: una confirmación puede perderse después de que el receptor haya procesado el mensaje y provocar una redelivery. La obligación es que esa redelivery no cree una segunda ejecución lógica ni duplique efectos de negocio.

La autenticación/autorización del transporte y los scopes de servicio pertenecen a ARQ-21. Este contrato define durabilidad, identidad, ack, reintentos y deduplicación; no crea un mecanismo de autenticación RabbitMQ paralelo.

#### D.6.2 Identidades no intercambiables

| Identificador | Autoridad / origen | Función |
|---|---|---|
| originMessageId | CRM, asignado una sola vez al mensaje de origen | Identifica el mensaje entrante lógico. Debe permanecer estable en reintentos, republicaciones y redeliveries. |
| executionId | AI Service, asignado al aceptar por primera vez originMessageId | Identifica la única ejecución lógica asociada al mensaje. Se conserva en recuperación/reintentos; no se genera otro por redelivery. |
| eventId | Servicio publicador del evento | Identifica un evento de integración concreto y permite deduplicar su publicación/consumo. Se conserva al republicar el mismo evento. |
| evAssignedId | CRM, identidad de la transición de asignación | Identifica la transición que autorizó la ejecución y resuelve ABA. No sustituye originMessageId ni executionId. |

originMessageId debe ser una identidad canónica y estable de CRM. Si el identificador externo del canal no es globalmente único, se conserva junto con el sistema/canal de origen y se define una clave compuesta estable, al menos acotada por workspaceId. El contenido normalizado del mensaje puede llevar un hash para detectar reutilización conflictiva de la misma clave.

#### D.6.3 Publicación CRM → AI Service

1. CRM persiste el mensaje de origen y la intención de despacharlo de forma durable en una misma unidad atómica (transacción o mecanismo recuperable equivalente). La publicación se representa mediante un registro Outbox de entrada ligado al mensaje; no se reutiliza el Outbox de asignaciones de D.1.8 como si fuera el mismo evento de dominio.
2. El publicador envía el mensaje a un exchange/cola durable con entrega persistente. Debe comprobar la confirmación positiva del broker y detectar mensajes no enrutables (por ejemplo, mediante publicación obligatoria/retorno o mecanismo equivalente). Si el cliente AMQP informa backpressure (publish devuelve false), el publicador respeta el flujo y espera drain o el mecanismo equivalente antes de continuar; esto no sustituye la confirmación del broker.
3. El Outbox solo se marca publicado después de confirmación positiva y enrutamiento válido. Fallos, timeout o confirmación incierta dejan el registro recuperable para reintento; no se absorben como éxito.
4. La republicación conserva originMessageId y la identidad del evento. Una confirmación incierta puede producir duplicados físicos en RabbitMQ; la deduplicación del consumidor es obligatoria.

El envelope de entrada incluye como mínimo eventId, eventType, schemaVersion, occurredAt, correlationId, originMessageId, organizationId, workspaceId, agentInstanceId, conversationId, contactId, channel y el contenido normalizado necesario. Los campos tenant/recurso se validan contra CRM y el contexto autenticado; no se confía en valores libres del payload.

#### D.6.4 Recepción, persistencia e idempotencia en AI Service

1. El consumidor usa ack manual. Antes de confirmar el mensaje RabbitMQ, valida el envelope y persiste una recepción durable vinculada a originMessageId y a un único executionId.
2. La unicidad lógica de recepción se aplica, como mínimo, por (workspaceId, channel, originMessageId) —o clave equivalente si CRM garantiza un ID global—. La restricción física/índice se concreta en WAB-81, pero la unicidad es parte obligatoria del contrato.
3. La persistencia de la recepción y su asociación a la ejecución lógica debe ser atómica. Si la escritura no se confirma, el consumidor no hace ack: reintenta/reencola según política. No se acepta continuar en memoria sin registro durable.
4. Si llega una redelivery con la misma clave y el mismo contenido lógico, el consumidor recupera la recepción/ejecución existente y hace ack sin crear otra ejecución ni volver a invocar el pipeline. Si la misma clave llega con contenido incompatible, se registra conflicto y se envía a cuarentena/DLQ; no se sobrescribe ni se ejecuta de nuevo.
5. **Momento del ack de entrada:** AI Service confirma el mensaje RabbitMQ después de que la recepción y la ejecución lógica hayan quedado persistidas de forma durable y recuperable, no simplemente porque el handler terminó sin lanzar una excepción. No necesita mantener el mensaje RabbitMQ sin ack durante toda la llamada LLM: la ejecución durable asume la responsabilidad de continuar/recuperarse.
6. Un worker puede recuperar una ejecución aceptada tras caída/reinicio usando la misma executionId, con control de ownership/lease/fencing de intento. Recuperar el worker no crea otra ejecución lógica.
7. Si el resultado de una invocación LLM queda incierto (timeout/desconexión después de enviar la solicitud), no se reintenta ciegamente como si no hubiera ocurrido. Se aplica la reconciliación de invocaciones y usage de P4, preservando invocationId/estado económico y evitando duplicar efectos cuando el proveedor no permite determinar el resultado.

#### D.6.5 Publicación AI Service → CRM y consumo idempotente

1. Al finalizar una ejecución, AI Service persiste el estado/resultado terminal y la intención de publicar el evento correspondiente como una única unidad atómica de persistencia (Transactional Outbox o mecanismo atómico equivalente, recuperable y verificable en WAB-81). No puede existir un resultado terminal confirmado sin una intención durable de publicarlo.
2. El publicador de resultados utiliza mensajes persistentes y confirmación positiva del broker. Solo marca el Outbox como publicado después de confirmación y enrutamiento válido. Ante backpressure respeta drain/flow control; ante fallo o resultado incierto conserva el evento para reintento; nunca convierte un fallo de publicación en éxito.
3. eventId y executionId permanecen estables en cada republicación. CRM registra un recibo durable de inbox con unicidad por eventId y executionId y persiste la aceptación del resultado junto con la intención de cualquier despacho posterior en una misma unidad atómica o mecanismo recuperable equivalente, antes del ack. Redis puede acelerar la deduplicación, pero no es su autoridad ni puede operar fail-open. La deduplicación actual basada solo en Redis SET NX no satisface este contrato.
4. execution.completed solo representa una ejecución completada. Las ejecuciones failed o cancelled no se publican como completadas; la semántica del evento respeta los contratos de cancelación/invalidez de ARQ-07.
5. CRM conserva su autoridad para validar ownership, asignación vigente y evAssignedId antes de cualquier efecto externo (ARQ-05/08 y E.4.11/E.4.12). La aceptación durable del evento y la creación de una intención durable de despacho deben preceder al ack; los envíos a canales externos conservan su propia identidad/idempotencia y política de reconciliación. Si el proveedor externo no ofrece idempotencia y el resultado de aceptación es incierto, no se promete exactly-once ni se reintenta ciegamente.

#### D.6.6 Reintentos, DLQ y recuperación

- Errores transitorios de publicación/consumo se reintentan con backoff y límites operativos; no se permite un ciclo de requeue inmediato e infinito.
- Mensajes inválidos, incompatibles o que agoten la política de reintentos se envían a una Dead Letter Queue (DLQ) con su identidad original y causa; no se descartan silenciosamente.
- La recuperación/replay desde DLQ conserva originMessageId, executionId si ya fue asignado y eventId del evento original. No genera una identidad nueva para sortear la deduplicación.
- Los Outbox pendientes, inboxes aceptados y ejecuciones no terminales deben ser recuperables tras reinicio del proceso. La retención/operación concreta de DLQ, backoff, leases y límites se especifica en WAB-81.
- El sistema debe poder distinguir al menos: pendiente de publicar, confirmado por broker, recibido durablemente, en proceso, terminal y en cuarentena/DLQ. La transición física y el modelo de estados se detallan en WAB-81 sin debilitar estas garantías.

#### D.6.7 Criterios de aceptación del contrato

1. Caída del publicador antes de confirmación: el Outbox permanece pendiente y se reintenta.
2. Confirmación de broker incierta o caída después de publicar: la republicación puede duplicar el mensaje físico, pero no crea otra ejecución lógica.
3. Redelivery de entrada: mismo originMessageId recupera el mismo executionId; no se vuelve a invocar el pipeline si ya está aceptado/procesado.
4. Caída del worker tras ack de entrada: la ejecución durable puede recuperarse sin depender de una nueva entrega del mensaje original.
5. Fallo del publicador de resultado: el resultado terminal conserva su Outbox pendiente hasta confirmación.
6. Redelivery del resultado: el inbox durable de CRM reconoce eventId/executionId ya aceptados, no repite la aceptación lógica y luego confirma el mensaje. La entrega al canal externo es una frontera posterior con su propia idempotencia/reconciliación.
7. Una clave repetida con contenido incompatible se pone en cuarentena y no ejecuta.
8. Una ejecución cancelada/fallida nunca se interpreta como execution.completed.
9. Los fallos ambiguos de invocación LLM respetan P4; no se promete exactly-once del proveedor ni se duplica consumo por reintento ciego.

**Límite de fase:** WAB-78 fija estas garantías y responsabilidades. La implementación de Outbox/inbox, índices, confirmaciones RabbitMQ, consumidores, reintentos, DLQ, leases, recuperación y pruebas de fallo corresponde a WAB-81. La autenticación del broker y el inventario de scopes continúan bajo ARQ-21/WAB-80/WAB-81. No se modifica código ni infraestructura en esta fase.

## E. Runtime y routing

### E.1 Flujo completo de ejecución

```
Entrada (RabbitMQ / HTTP interno)
        │
        ▼
┌─────────────────────────┐
│ 1. Validar mensaje      │  Zod schema: workspaceId, agentInstanceId
│    (workspaceId,        │    obligatorios. contactId, content.
│     agentInstanceId)    │                                                   │
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│ 2. Validar ownership    │  workspaceId del payload vs agentInstance.
│    workspaceId vs       │  Si no coinciden → FORBIDDEN.
│    agentInstance        │                                                   │
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│ 3. Resolver instancia   │  Buscar AgentInstance por id + workspaceId.
│    + verificar estado   │  status debe ser 'active'. Si no → rechazar.
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│ 4. Vigencia G1          │  AgentInstance activa, contacto no asignado a humano,
│    (fail-closed si      │    no hay cambio de asignación. Si MongoDB no
│    MongoDB no está      │    está disponible o no es posible comprobar
│    disponible)          │    la vigencia, no ejecutar respuestas ni
│                         │    acciones que dependan de esa validación.                                                   │
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│ 5. Cargar configuración │  AgentDefinition + AgentInstance (Skills,
│    del agente           │  Rules, knowledgeBaseIds, additionalPrompt).
│                         │  Instantánea inmutable al iniciar la ejecución   │
│                         │  (P10): crear instantánea de configuración       │
│                         │  efectiva; calcular promptSnapshotHash y         │
│                         │  effectiveConfigHash. La ejecución conserva sus  │
│                         │  definiciones hasta finalizar.                   │
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│ 6. Cargar Knowledge     │  RAG filtrado por workspaceId (BR-KB-001).
│    (RAG)                │  Qdrant error → error visible, no [] silencioso.
│                         │  No exigir agentInstanceId como filtro (I11).   │
│                         │  Registrar IDs de fragmentos recuperados para   │
│                         │  trazabilidad RAG (P10).                        │
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│ 7. Cargar Skills/Rules  │  SkillConfiguration y AgentRules de la instancia.
│    por instancia        │  Filtrar capabilities por enabledCapabilities.  │
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│ 8. Cargar memoria       │  ConversationMemory: últimos 20 mensajes
│    (ventana 20)         │  user/assistant, sin resumen automático.        │
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│ 9. Componer contexto    │  [System Prompt] + [RAG Workspace] + [20 msgs]
│    y validar políticas  │  user/assistant. Policies de plataforma        │
│    (Policy Engine)      │  prevalecen sobre Rules/Skills/Knowledge.     │
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│ 10. Solicitar presupuesto│  Solicitar presupuesto máximo autorizado para  │
│    de ejecución autorizado │  la ejecución (OrganizationAICredit.committedB-  │
│                         │  udget). Verificar saldo disponible de la Orga-  │
│                         │  nización. Si insuficiente → rechazar.           │
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│ 11. Invocar proveedor   │  Provider/modelo centralizados (Waby). Credencial
│    LLM y ejecutir tools │  gestionada por Waby, nunca expuesta al cliente.│
│                         │  Capabilities filtradas por instancia (I12).     │
│                         │  Output filter antes de enviar al usuario.     │
│                         │  Antes de cada invocación facturable, asignar   │
│                         │   presupuesto de invocación desde el presupuesto│
│                         │   autorizado de la ejecución (fencingGeneration   │
│                         │   vigente).                                       │
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│ 12. Vigencia G2         │  Revalidar antes de completar. Fail-closed.   │
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│ 13. Liquidar créditos   │  Débito append-only en CreditLedger (idempotency│
│    directos (deduct)    │  key = execution_id). Descontar de              │
│                         │  OrganizationAICredit.balance (consumo real)   │
│                         │   y decrementar committedBudget por el          │
│                         │   monto autorizado. La diferencia no usada      │
│                         │   queda liberada. Reintentos sin descuentos    │
│                         │   duplicados.                                   │
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│ 15. Persistir ejecución │  AgentExecution + AgentExecutionLog (campos     │
│    y auditoría          │  I21: promptSnapshotHash, effectiveConfigHash,  │
│                         │  effectiveConfigSnapshot, provider_model_used,  │
│                         │  tokens, créditos, latencia, resultado).        │
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│ 16. Actualizar memoria  │  ConversationMemory: agregar mensaje user +     │
│    y publicar resultado │    respuesta assistant (máximo 20, FIFO).       │
└─────────────────────────┘
```

**Garantías de mensajería:** la entrada y salida de este flujo siguen D.6 (ARQ-22): CRM publica desde Outbox; AI Service hace ack tras aceptación durable en inbox/ejecución; el resultado se publica desde Outbox con confirmación; CRM consume idempotentemente. El ack de entrada no queda ligado a mantener abierta la entrega RabbitMQ durante toda la ejecución LLM.

### E.2 Resolución de AgentInstance

La resolución no utiliza routing persistente canal → instancia ni `AgentInstance.channelAssignment`. El canal se conserva como contexto del mensaje entrante y de Execution, no como asociación permanente de routing.

CRM aplica la precedencia de DEC-CM-06:

1. Si existe una asignación humana vigente, la IA no interviene.
2. Si existe una asignación vigente a una AgentInstance activa del mismo Workspace, esa instancia conserva la atención.
3. Si existe una asignación vigente a una AgentInstance inactiva, suspendida o eliminada, se conserva el ContactAssignment pero se impide la atención automática. No hay fallback a otra instancia ni al default. La atención solo se recupera mediante reactivación o reasignación explícita.
4. Si el contacto no tiene asignación humana ni asignación a una AgentInstance, ante un nuevo mensaje entrante CRM intenta asignar la única AgentInstance predeterminada del Workspace, únicamente si está activa.
5. Si no existe una predeterminada activa, el contacto permanece sin asignación.
6. No existe fallback a otra instancia por prioridad ni selección alternativa.

### E.3 Validación y unicidad

- CRM resuelve y valida ContactAssignment; los identificadores aportados por el cliente o el mensaje no son autoridad.
- Contacto y AgentInstance deben pertenecer al mismo Workspace.
- Como máximo una instancia puede tener `isDefault=true` por Workspace; el cambio es atómico.
- Los workers y ai-service vuelven a validar la asignación vigente antes de procesar o despachar.
- Una suspensión conserva la asignación y no habilita fallback automático.
- Una transición de asignación invalida ejecuciones incompatibles; la identidad de EvAssigned permite identificar la transición autorizante conforme al contrato aprobado.

### E.4 Recepción automática, delegación, asignación y transferencia de AgentInstance (P2)

Esta sección consolida los acuerdos funcionales de P2 sobre recepción automática, asignaciones, delegaciones, distribución de equipos, transferencia fallida, desasignación, takeover humano, ciclos de delegación y seguridad de concurrencia. Son acuerdos de diseño; no modifican código, modelos, schemas, permisos ni roles.

#### E.4.1 Agente predeterminado

**Acordado:**

- Una AgentInstance puede ser marcada como predeterminada por un usuario autorizado del Workspace.
- Puede existir cero o una instancia predeterminada por Workspace; nunca más de una.
- El cambio de `isDefault` es atómico.
- La creación de una instancia no obliga a que el Workspace tenga siempre una predeterminada. Si no existe una predeterminada activa, el contacto permanece sin asignación.

- Cambiar `isDefault` no migra ni altera ContactAssignment existentes (DEC-CM-08).

**Decisión aprobada DEC-CM-06/08:** no existe selección entre varias predeterminadas ni fallback por prioridad.





#### E.4.2 Recepción automática

**Acordado:**

- Solo los contactos sin asignación pueden entrar en el mecanismo de recepción automática.
- La evaluación ocurre ante un nuevo mensaje entrante del contacto.
- Waby comprueba la disponibilidad operativa de IA de la Organización y la disponibilidad de la AgentInstance predeterminada.
- La asignación debe confirmarse antes de despachar el mensaje al runtime.
- La ausencia de un agente receptor disponible deja el contacto sin asignación para atención humana.

#### E.4.3 Comportamiento de asignaciones humanas (conservado)

Se conserva el comportamiento actual de Waby:

- Si un humano responde a un contacto sin asignación, queda automáticamente asignado a sí mismo.
- La asignación debe establecerse antes de completar el envío del mensaje.
- Un humano puede tomar el control de una conversación atendida por IA.
- El takeover detiene la atención de IA.

#### E.4.4 Delegaciones y transferencias

Una AgentInstance puede realizar las siguientes acciones dentro de las capacidades concedidas por Waby:

- Asignarse un contacto sin responsable cuando esté autorizada para recibirlo.
- Transferir a otro usuario humano que ya pertenezca al Workspace.
- Transferir a un equipo del mismo Workspace.
- Transferir a otra AgentInstance activa del mismo Workspace.
- Desasignarse y liberar el contacto.

Los agentes IA **no pueden**:

- Invitar usuarios.
- Crear usuarios.
- Modificar membresías.
- Modificar roles o permisos.
- Crear equipos.
- Administrar la estructura del Workspace.

El LLM decide el destino lógico de la delegación. Waby valida el destino y ejecuta la asignación.

**Pendiente de auditoría técnica:** qué permisos actuales `wrk.teams.*` permiten administrar la configuración de equipos para distribución.

#### E.4.5 Distribución de equipos

**Dirección de diseño acordada:**

- La estrategia se configura por equipo por un administrador autorizado del Workspace.
- Round Robin es la estrategia predeterminada.
- Least Open Conversation es una alternativa.
- Waby selecciona al integrante concreto; no delega esa decisión de distribución al LLM.
- La estrategia debe considerar pertenencia y elegibilidad del usuario.

**Propuesta acordada (pendiente de confirmación expresa):**

- Disponibilidad online como filtro opcional por equipo.
- Inicialmente desactivado, para conservar el comportamiento actual.
- Límite de conversaciones abiertas como capacidad configurable, sujeto a diseño.

**Pendiente de auditoría técnica:** determinar qué permisos actuales `wrk.teams.*` permiten administrar esta configuración.

#### E.4.6 Transferencia fallida y fallback

Cuando una AgentInstance solicita transferir un contacto a un destino:

1. Waby verifica la elegibilidad del destino.
2. Si el destino es un equipo, resuelve un integrante conforme a la estrategia configurada.
3. Si no hay destinatarios elegibles dentro del destino solicitado, no se inventa una instancia alternativa ni se aplica fallback por prioridad. Se aplica únicamente una alternativa humana/equipo si forma parte explícita de la cadena de delegación autorizada; si no existe destino elegible, el contacto queda `unassigned`.
4. La AgentInstance de origen deja de responder.

La transferencia no debe dejar al agente de origen como responsable cuando el proceso de delegación agotó sus alternativas. Un intento fallido no debe iniciar inmediatamente otra recepción automática para el mismo evento que lo originó.

#### E.4.7 Desasignación y nueva recepción automática

Cuando una AgentInstance se desasigna:

- El contacto queda `unassigned`.
- La instancia deja de procesar mensajes nuevos de ese contacto.
- Se invalidan las ejecuciones pendientes que ya no sean válidas.
- No se ejecuta inmediatamente el agente predeterminado.
- El contacto queda disponible para atención humana.
- Un nuevo mensaje entrante puede iniciar una nueva evaluación de recepción automática.

La desasignación no debe confundirse con un evento de mensaje entrante.

#### E.4.8 Takeover humano (invariante)

Cuando un humano toma un contacto atendido por IA:

- El responsable pasa a ser el usuario humano.
- La IA deja de procesar mensajes nuevos del contacto.
- Se invalidan las ejecuciones pendientes de la instancia anterior.
- Los workers deben verificar la asignación vigente antes de procesar y despachar.
- Una respuesta generada bajo una transición de asignación anterior (identidad de `EvAssigned` distinta a la vigente en CRM) debe rechazarse.

El historial de conversación permanece disponible para los usuarios autorizados.

**Notas de implementación:**

- No basta con instruir al LLM para que no responda. La garantía debe implementarse en backend y runtime.

#### E.4.9 Ciclos de delegación y eventos actuales

**Dirección de diseño:**

Se propone utilizar el mecanismo existente de eventos de Waby para detectar ciclos de asignación y delegación.

**Elementos que deben investigarse en una auditoría posterior:**

- Eventos `assigned` almacenados junto a los mensajes.
- Eventos de inicio de conversación.
- Eventos de cierre de conversación.
- Orden e idempotencia de los eventos.
- Identificación de actor, asignación anterior y nueva.
- Relación entre el evento original y la cadena de delegación.
- Concurrencia y reintentos.

**Regla funcional acordada:**

Cuando se detecta un ciclo de delegaciones, el contacto queda `unassigned`, termina la cadena actual y vuelve a la cola. No se vuelve a ejecutar la recepción automática por el mismo evento. Un nuevo mensaje entrante puede iniciar una nueva evaluación.

No diseñar todavía nuevos eventos ni mecanismos alternativos antes de auditar los existentes.

#### E.4.10 Seguridad y concurrencia

**Invariantes consolidados:**

- Una única asignación efectiva de atención por contacto.
- Solo el responsable vigente puede procesar mensajes mediante IA.
- Todas las transferencias validan ownership y pertenencia al mismo Workspace.
- Los cambios de Contacto, `EvAssigned` y Outbox deben preservar una única transición lógica consistente. La solución preferida es persistir los tres elementos en una misma transacción MongoDB. Su viabilidad está pendiente de verificar en la infraestructura real de MongoDB Atlas, incluyendo soporte de transacciones, sesiones y compatibilidad de las colecciones involucradas.
- Los cambios de responsable invalidan ejecuciones incompatibles con la nueva asignación.
- Los workers RabbitMQ vuelven a validar antes de procesar o enviar mensajes.
- Los reintentos no generan asignaciones ni respuestas duplicadas.
- Las transferencias deben quedar auditadas.

**Descartado:** no se introduce `assignmentVersion` ni un mecanismo de fencing/versionado de asignación. La identidad única de `EvAssigned` identifica la transición que autorizó cada ejecución y permite compararla contra la asignación vigente de CRM antes del despacho o efecto externo.

#### E.4.11 Cancelación de ejecuciones e invalidación de resultados (decisión aprobada — WAB-78)

Cuando una AgentInstance pierde la asignación de un contacto:

- Se deben cancelar sus ejecuciones activas para ese contacto.
- Se deben invalidar los trabajos pendientes.
- Se debe intentar abortar llamadas LLM en curso cuando el transporte y proveedor lo permitan.
- Una ejecución que no pueda cancelarse físicamente debe quedar invalidada.
- Ningún resultado tardío sin autorización puede producir efectos externos.

Esto aplica a:
- `agent → human`
- `agent A → agent B`
- `agent → unassigned`

**Dos garantías obligatorias y distintas:**
1. **Cancelación operativa:** interrumpe la ejecución activa (aborta LLM, cancela pasos pendientes, libera recursos). Si falla, la ejecución puede seguir generándose.
2. **Invalidación del resultado:** rechaza el resultado aunque el proveedor lo haya completado. Se aplica en G3/G4 antes de despachar.

El consumo de una llamada LLM abortada se trata conforme a P4. No reabrir la política de facturación.

**Nota de vigencia (06/11/2026):** la afirmación histórica de que ai-service no estaba disponible localmente quedó obsoleta: el repositorio crm contiene microservices/ai-service. Esta reconciliación no repite su auditoría ni afirma que la cancelación o el soporte AbortController estén implementados; esos mecanismos siguen requiriendo diseño y verificación específica.

#### E.4.12 Vigencia G3/G4 y contrato de despacho (decisión aprobada — WAB-78)

CRM es la autoridad de vigencia. Antes de cualquier efecto externo, se debe comprobar:

- Contacto con `assignedType = agent`.
- `ref_agent_assigned` coincide con la AgentInstance ejecutora.
- Identidad `EvAssigned` vigente coincide con la que autorizó la ejecución.
- Contacto e instancia pertenecen al mismo Workspace.
- Instancia sigue en estado que permite ejecutar.

**En G3 (CRM):** puede consultar su fuente local directamente.
**En G4 (otro servicio):** necesita un mecanismo interno confiable para consultar CRM o una validación equivalente cuya autoridad sea CRM.

Si no se puede confirmar la vigencia, no se despacha el resultado.

---

## F. Composición del contexto

### F.1 Jerarquía de instrucciones (orden de prioridad)

El contexto enviado al proveedor LLM sigue este orden estricto:

| Prioridad | Componente | Origen | Puede ser anulado por cliente? |
|-----------|------------|--------|-------------------------------|
| 1 | System Prompt / Persona | AgentDefinition (Waby) | ❌ No |
| 2 | Reglas de plataforma (Policy Engine) | Waby (central) | ❌ No |
| 3 | Rules de la instancia | AgentRules (I13) | Solo con `wrk.agents.*` |
| 4 | Knowledge del Workspace | WorkspaceKnowledge (RAG, I11) | ⚠️ Cliente gestiona documentos, pero no puede anular políticas |
| 5 | Skills de la instancia | SkillConfiguration (I12) | Solo con `wrk.agents.*` |
| 6 | Memoria conversacional | ConversationMemory (últimos 20 user/assistant, I14) | ❌ No (solo lectura del historial) |
| 7 | Mensaje de usuario actual | Entrada del cliente | ✅ Sí (contenido del mensaje) |

### F.2 Prevención de prompt injection

**Problema identificado en Fase 1:** Los fragmentos de Knowledge viajaban con rol `system` (`openai.mapper.ts:26-32`), misma prioridad que instrucciones del sistema, permitiendo que un documento malicioso sobrescriba instrucciones de plataforma.

**Solución en el diseño objetivo:**

1. **Knowledge viaja como rol `user` o rol dedicado `knowledge`**, nunca como `system`. El system prompt (prioridad 1 y 2) tiene precedencia absoluta sobre el contenido de Knowledge.
2. **Policy Engine como capa intermedia:** Antes de enviar al proveedor, el sistema valida que el contenido generado no viole políticas de plataforma. Las políticas no pueden ser anuladas por instrucciones, skills, documentos o reglas empresariales (I8).
3. **Output filter:** El contenido de respuesta del proveedor se filtra antes de enviarlo al usuario (corregir hallazgo R11 de Fase 1).
4. **Separación de fronteras de confianza:** Contenido de cliente (mensajes, documentos de Knowledge) nunca tiene prioridad de sistema.

### F.3 Evitar que contenido empresarial modifique permisos o políticas

- Las políticas de plataforma (I8) se aplican como capa de validación antes y después de la generación, no como parte del prompt enviable al modelo.
- Skills y Rules de la instancia (prioridades 3 y 5) pueden influir en el tono y comportamiento, pero no pueden anular políticas de plataforma.
- Knowledge (prioridad 4) es contexto informativo, nunca instrucción de sistema.
- El system prompt de AgentDefinition es inmutable para el cliente (solo Waby lo edita).

---

## G. Proveedores, créditos y telemetría

### G.1 Routing centralizado de modelos (I18)

**Principio:** Waby administra proveedor, modelo e hiperparámetros. El cliente no los selecciona ni los configura.

| Componente | Responsable | Configurable por cliente? |
|------------|-------------|--------------------------|
| Proveedor LLM | Waby (central) | ❌ No |
| Modelo LLM | Waby (central) | ❌ No |
| Temperatura | Waby (central) | ❌ No |
| Max tokens | Waby (central) | ❌ No |
| TopP, penalties | Waby (central) | ❌ No |
| Credenciales API de proveedores LLM | Waby (central, cifradas) | ❌ No |
| Credenciales de integraciones empresariales | Workspace, mediante usuarios autorizados y solo para integraciones permitidas | Sí, dentro de la allowlist y políticas Waby |

**Implementación:** Tabla de rutas central y versionada (configuración, no código) que resuelve un único proveedor y modelo LLM activo por AgentDefinition (P8). La estructura podrá admitir proveedores de respaldo en el futuro, pero en esta fase cada AgentDefinition utiliza un solo LLM: no hay rotación ni fallback entre proveedores. Los reintentos técnicos, si se habilitan, se realizan contra el mismo proveedor/modelo. Esta configuración nunca se expone por API al cliente.

### G.2 Credenciales de proveedores LLM administradas por Waby (I6)

- Las API keys de proveedores LLM son gestionadas exclusivamente por Waby.
- Cifradas en reposo (AES-256-GCM o equivalente).
- Nunca expuestas al cliente, ni en respuestas de API, ni en logs.
- La `aiCredential` per-bot del modelo actual se elimina; se reemplaza por credenciales centralizadas de Waby asociadas al routing de proveedores.

Esta regla se limita a credenciales de proveedores LLM y no transfiere a Waby la administración de credenciales de integraciones empresariales del Workspace.

### G.2.1 Credenciales de integraciones empresariales administradas por Workspace

- Los usuarios autorizados del Workspace configuran, actualizan y revocan las credenciales de las integraciones empresariales que Waby habilita para sus AgentInstance.
- Waby define la allowlist de integraciones, las capacidades disponibles y las políticas de seguridad; no administra las cuentas ni decide los valores de las credenciales empresariales del cliente.
- `SkillConfiguration.credentials` contiene únicamente referencias a credenciales (IDs), nunca sus valores secretos. La resolución y el uso de esos secretos se realizan mediante el mecanismo seguro de credenciales de la plataforma.
- Los valores secretos de credenciales empresariales nunca se copian a `effectiveConfigSnapshot`, prompts, logs ni metadatos de ledger. Cuando sea necesaria trazabilidad, puede conservarse únicamente una referencia identificable/versionada no secreta.

### G.3 Cálculo de tokens

| Métrica | Descripción | Origen |
|---------|-------------|--------|
| inputTokens | Tokens de entrada (prompt + contexto) | Contador de tokens del proveedor |
| outputTokens | Tokens de salida (respuesta) | Contador de tokens del proveedor |
| totalTokens | inputTokens + outputTokens | Calculado |
| totalCreditsDeducted | Créditos descontados | Conversión tokens → créditos |

### G.4 Bolsa única de créditos por Organización (I17, I20)

**OrganizationAICredit:** Un saldo por Organización (1:1). No hay bolsas por Workspace ni por instancia. Todos los Workspaces de una Organización comparten el mismo saldo de créditos. Un Workspace puede consumir todo el saldo disponible de su Organización y dejar temporalmente sin créditos a los demás Workspaces. No se implementan límites individuales de consumo por Workspace en esta etapa.

**Autoridad del saldo:** `OrganizationAICredit` reside en la base de datos del CRM. Es la única fuente de verdad del saldo disponible. El AI Service lee y debita este saldo a través de operaciones internas, pero nunca posee una copia local del saldo. `organizationId` es el ámbito contable; `workspaceId` es una dimensión de atribución solamente.

---

#### G.4.1 Flujo de presupuesto y liquidación final post-ejecución (P4)

El flujo de créditos de una ejecución lógica se compone de autorización de capacidad, ejecución, clasificación individual de invocaciones y una única liquidación financiera final:

1. **Autorización del presupuesto:** Antes de invocar al proveedor LLM, el sistema solicita un presupuesto máximo para la ejecución. Dentro de una transacción MongoDB verifica que `OrganizationAICredit.balance - OrganizationAICredit.committedBudget >= presupuesto_solicitado`. Si hay disponibilidad, incrementa `committedBudget` y crea una `ExecutionBudgetAuthorization` con `status = 'authorized'` y su `fencingGeneration` inicial. Si no hay disponibilidad, rechaza la ejecución sin invocar al proveedor.
2. **Ejecución e invocaciones:** El agente procesa la interacción. Cada invocación al proveedor LLM tiene su propio `invocationId` y asignación de presupuesto; los reintentos contra el mismo proveedor/modelo conservan el `executionId` y utilizan otro `invocationId`; no se contempla fallback entre proveedores.
3. **Reconocimiento de consumo:** Se obtiene el usage informado por el proveedor para cada invocación y se convierte a créditos mediante la versión de tarifa (`pricingVersion`) aplicable.
4. **Clasificación financiera:** Cada invocación se clasifica conforme a G.4.6: consumo confirmado y facturable, consumo confirmado absorbido por Waby, o consumo desconocido pendiente de reconciliación. La validez técnica del resultado se evalúa independientemente del resultado del despacho al canal.
5. **Reconciliación:** Si alguna invocación mantiene un resultado externo desconocido, permanece en `reconciliation_required` y conserva su capacidad comprometida durante el período de gracia aprobado de **30 días** desde su entrada en ese estado. Al vencer sin evidencia suficiente, se excluye únicamente ese consumo desconocido y se libera la asignación correspondiente.
6. **Liquidación final única:** Cuando todas las invocaciones estén resueltas —incluidas las que se cierren por vencimiento del período de gracia— se consolida únicamente el consumo facturable. Si el importe facturable es mayor que cero, se registra un único `deduct` por `(organizationId, executionId)` y se actualiza `OrganizationAICredit.balance` en la misma transacción que libera el presupuesto comprometido de la autorización. Si el importe facturable es cero, no se crea un `deduct` de importe cero y se cierra/libera la autorización cuando todas sus asignaciones estén resueltas.
7. **Atomicidad e idempotencia:** La escritura del ledger, la actualización del saldo y la liberación final del presupuesto comprometido se realizan de forma atómica. Los reintentos de liquidación usan el mismo `executionId` y no generan débitos duplicados.

La aceptación del proveedor del canal, `delivered` y `read` no son requisitos universales de facturación. El resultado del despacho se utiliza para atribuir responsabilidades conforme a G.4.6.

---

#### G.4.2 Protección contra sobreconsumo bajo concurrencia (alternativa B)

**El problema:** Una validación previa de saldo seguida de una transacción atómica de débito **no es suficiente** para prevenir sobreconsumo bajo concurrencia. Escenario:

- Una Organización tiene 10 créditos.
- Tres ejecuciones simultáneas de Workspaces diferentes (E1 en WS1, E2 en WS2, E3 en WS3) leen el mismo saldo de 10 créditos.
- Las tres pasan la validación previa.
- Las tres invocan al proveedor LLM y consumen tokens.
- Al liquidar, E1 debita 4 créditos (saldo → 6), E2 debita 4 créditos (saldo → 2), E3 intenta debitar 4 créditos pero solo quedan 2.

Si E3 no puede completar su débito por saldo insuficiente, ya ha consumido recursos del proveedor sin que la Organización pueda pagarlos.

**Mecanismo aprobado (ADR-002/ADR-006): presupuesto máximo autorizado por ejecución y asignación por invocación**

Para permitir concurrencia entre Workspaces sin bloquear toda la Organización, cada ejecución solicita y obtiene un **presupuesto máximo autorizado** antes de invocar al proveedor. Cada ejecución tiene su propia autorización de presupuesto con un `fencingGeneration` independiente. El presupuesto se registra como capacidad comprometida temporalmente en `OrganizationAICredit.committedBudget`, y cada autorización individual se rastrea en `ExecutionBudgetAuthorization`.

**Flujo de autorización de presupuesto:**

1. **Solicitud de presupuesto:** Antes de invocar al proveedor, el worker solicita un presupuesto máximo de créditos para la ejecución.
2. **Verificación atómica:** El sistema verifica que `OrganizationAICredit.balance - OrganizationAICredit.committedBudget >= presupuesto_solicitado`. Si es suficiente, incrementa `committedBudget` por el monto solicitado y registra la autorización.
3. **Ejecutar:** El agente procesa el mensaje con el presupuesto autorizado.
4. **Control durante ejecución:** Si la ejecución alcanza su presupuesto autorizado, se detiene la generación.
5. **Liquidación:** Al completar, se descuenta el consumo real de `OrganizationAICredit.balance` y se decrementa `committedBudget` por el monto autorizado (la diferencia no utilizada queda liberada).
6. **Idempotencia:** `execution_id` es la llave de idempotencia. Una misma `execution_id` no puede generar dos movimientos `deduct`.

**Ventajas de esta alternativa:**

- **Concurrencia:** Múltiples Workspaces de la misma Organización pueden ejecutar agentes simultáneamente, cada uno con su propio presupuesto autorizado.
- **Sin lock de Organización:** No se bloquea toda la Organización durante llamadas a proveedores externos.
- **Sin hold del ledger:** No se introducen movimientos `hold`/`release` en `CreditLedger`. El `committedBudget` es un campo de capacidad temporalmente asignada, no un movimiento contable.
- **Protección contra sobreconsumo:** La suma de presupuestos comprometidos nunca puede exceder el saldo disponible.

**Limitación:** Si una ejecución necesita más créditos de los autorizados inicialmente, debe solicitar un presupuesto adicional (sujeto a disponibilidad).

---

#### G.4.2.1 Adquisición de presupuesto y control de concurrencia

**Solicitud de presupuesto de ejecución:** Antes de invocar al proveedor, el AI Service solicita un presupuesto máximo de créditos para la ejecución. La solicitud incluye el monto máximo estimado.

**Verificación atómica:** Dentro de una transacción de MongoDB, el sistema verifica que `OrganizationAICredit.balance - OrganizationAICredit.committedBudget >= presupuesto_solicitado`. Si es suficiente, incrementa `committedBudget` por el monto solicitado y registra la autorización vinculada al `execution_id`.

**Liberación de presupuesto de ejecución:** Al liquidar la ejecución, se decrementa `committedBudget` por el monto autorizado y se descuenta el consumo real de `balance`. La diferencia no utilizada queda liberada para otras ejecuciones.

**Recuperación ante fallos:** Si un worker cae después de autorizar presupuesto pero antes de liquidar, el `committedBudget` permanece comprometido hasta que se libere. Se puede implementar un proceso de reconciliación periódica que libere presupuestos de ejecuciones abandonadas (por ejemplo, después de un tiempo de expiración).

---

#### G.4.2.1.1 Presupuesto por invocación

Cada `ExecutionBudgetAllocation` representa el presupuesto máximo asignado a una invocación concreta al proveedor LLM. Los reintentos contra el mismo proveedor/modelo dentro de una ejecución lógica conservan el `executionId` y usan un `invocationId` distinto por cada llamada real al proveedor. Las ejecuciones HTTP de Skills y otras operaciones no-LLM quedan fuera de P4.

**Presupuesto disponible para nuevas invocaciones:**

`availableInvocationBudget = authorizedBudget - consumedAmount - activeAllocatedBudget`

La fórmula utiliza el contador persistido `activeAllocatedBudget` de la autorización padre; no depende de una suma de asignaciones calculada fuera de la transacción.

**Asignación de presupuesto:**

1. Antes de invocar al proveedor LLM, el runtime solicita un `assignedBudget` para el `invocationId`.
2. En una transacción/actualización condicional se valida la autorización padre (estado, propietario, `attemptId`, lease y `fencingGeneration`) y la asignación, si ya existe.
3. Solo si `availableInvocationBudget >= assignedBudgetSolicitado`, se incrementa `activeAllocatedBudget` por el presupuesto solicitado y se crea la `ExecutionBudgetAllocation` en estado `assigned`.
4. La suma de asignaciones activas nunca puede superar `authorizedBudget - consumedAmount`.

**Liquidación de una invocación resuelta:**

- Se registra el usage efectivamente reconocido por el proveedor en `ExecutionBudgetAllocation.consumedAmount`.
- En una operación atómica se incrementa `ExecutionBudgetAuthorization.consumedAmount` por el usage reconocido y se decrementa `activeAllocatedBudget` por el `assignedBudget` completo de esa asignación.
- La asignación pasa a `settled`. El valor histórico de `assignedBudget` no se modifica.
- La diferencia no utilizada entre `assignedBudget` y el usage reconocido vuelve a estar disponible para nuevas invocaciones de la misma ejecución mediante la fórmula anterior.
- El consumo reconocido que G.4.6 clasifique como absorbido por Waby también consume capacidad de la autorización; sin embargo, no se incluye en el `billableAmount` que se debita a la Organización.

**Invocaciones sin resultado concluyente:**

- Una asignación en `reconciliation_required` conserva íntegro su `assignedBudget` dentro de `activeAllocatedBudget`; no se libera capacidad por un timeout de resultado desconocido.
- Si la invocación se reconcilia, se aplica la liquidación normal con usage confirmado.
- Si transcurren los 30 días naturales aprobados desde su entrada en `reconciliation_required` sin evidencia suficiente, se excluye el consumo desconocido, la asignación se cierra como `expired` y se decrementa `activeAllocatedBudget` por su `assignedBudget` completo. No se modifica el `assignedBudget` histórico ni se genera un débito por consumo no confirmado.
- Los estados `assigned`, `executing`, `settling` y `reconciliation_required` retienen capacidad; `settled`, `cancelled` y `expired` no la retienen. `cancelled`/`expired` solo liberan antes del vencimiento de gracia cuando existe evidencia suficiente de que no hay consumo externo pendiente.

**Liquidación de la ejecución:** La autorización padre conserva su `committedBudget` mientras esté en `authorized`, `executing`, `settling` o `reconciliation_required`. El débito final de la ejecución espera hasta que todas sus asignaciones tengan un estado terminal (`settled`, `cancelled` o `expired`). Entonces se calcula el importe facturable conforme a G.4.6, se registra como máximo un `deduct` por `executionId` y se libera el `authorizedBudget` completo de `committedBudget` en la misma transacción final. Si el importe facturable es cero, se cierra la autorización y libera presupuesto sin crear un asiento `deduct` de importe cero.

---

#### G.4.2.2 Fencing por autorización y asignación individual

**Ámbito del fencing:** Cada `ExecutionBudgetAuthorization` y cada `ExecutionBudgetAllocation` tiene su propio `fencingGeneration`. No existe un token global por Organización ni un lock exclusivo de Organización.

**Incremento de generación:** La generación se incrementa únicamente cuando una recuperación/reasignación entrega la propiedad del recurso a un nuevo worker. Una renovación normal del lease, una ampliación de presupuesto solicitada por el mismo propietario o un retry financiero idempotente no incrementan la generación por sí solos. Un nuevo intento de propietario (`attemptId`) se registra al recuperar/reasignar el trabajo; `attemptId` y `fencingGeneration` no son claves de idempotencia del ledger.

**Validación de escrituras:** Toda operación sobre una asignación valida dentro de la misma transacción MongoDB tanto el `fencingGeneration` de la asignación como el de su autorización padre, junto con el recurso, `workerId`/propietario, `attemptId`, estado y lease vigente. Las operaciones sobre una autorización validan los datos equivalentes de esa autorización. Una condición que no coincide provoca el rechazo de la escritura y el rollback de la transacción.

**Recuperación:** Si Worker A posee una autorización con `fencingGeneration = 5` y el sistema recupera/reasigna esa autorización a Worker B, la generación pasa a `6`; las escrituras posteriores de Worker A condicionadas a generación `5` son rechazadas. Una renovación normal del lease de Worker A conserva la generación `5` y no lo invalida.

**Límite del fencing:** El fencing impide que un worker obsoleto realice escrituras financieras en MongoDB. No cancela una solicitud externa que el proveedor LLM ya haya aceptado ni revierte un consumo que haya ocurrido fuera de Waby; esos resultados deben resolverse mediante idempotencia y reconciliación conforme a G.4.6.

---

#### G.4.2.3 Alcance de la exclusión (operaciones cubiertas)

El mecanismo de control de presupuesto por Organización se aplica a **todas** las operaciones capaces de modificar el saldo de la Organización:

- Liquidaciones de ejecuciones (`deduct` en `CreditLedger`).
- Devoluciones y compensaciones (`refund` en `CreditLedger`).
- Compras o acreditaciones de créditos (`purchase` en `CreditLedger`).
- Ajustes administrativos de saldo.
- Cualquier otro débito o crédito que pueda introducirse posteriormente.

Ninguna operación que afecte el saldo puede ignorar el mecanismo de control de presupuesto. Todas deben verificar la disponibilidad de saldo antes de modificar `OrganizationAICredit.balance`.

---

#### G.4.2.4 Integración con la arquitectura existente

**Infraestructura disponible:** Waby ya dispone de Redis y RabbitMQ. La autoridad del saldo y el presupuesto comprometido residen en MongoDB (`OrganizationAICredit` y `ExecutionBudgetAuthorization`). No se introduce infraestructura nueva.

**Alternativas comparadas:**

| Alternativa | Ventajas | Desventajas | Decisión |
|-------------|----------|-------------|----------|
| **Presupuesto máximo autorizado por ejecución (B)** | Permite concurrencia entre Workspaces; no bloquea toda la Organización; sin hold del ledger | Requiere estimación de presupuesto máximo por ejecución; complejidad en control de consumo durante ejecución | **Aceptado** |
| **Lock exclusivo por Organización** | Simple de implementar; garantiza saldo no negativo | Serializa TODAS las ejecuciones de una Organización; bajo throughput; bloquea durante llamadas a proveedores | Descartado |
| **Consumo concurrente con sobreconsumo acotado** | Máxima concurrencia | No garantiza límite económico; requiere política comercial de sobreconsumo | Descartado como garantía técnica |
| **Lock por Workspace (anterior)** | Simple | No protege la unidad económica correcta (Organización); serializa innecesariamente dentro de una Organización | Descartado |

**Transacción de liquidación:** La liquidación final (escritura en `CreditLedger` + actualización de `OrganizationAICredit.balance` y decremento de `committedBudget`) usa una **transacción corta de MongoDB** (no abierta durante la llamada al proveedor).

---

#### G.4.2.5 Orden de operaciones: ejecución, despacho y liquidación

El orden operativo debe permitir clasificar la responsabilidad sobre el despacho antes de calcular el importe facturable final:

1. **Ejecución del pipeline:** El agente ejecuta las invocaciones LLM autorizadas y recopila el usage informado por el proveedor para cada `invocationId`.
2. **Validación de autorización y fencing:** Antes de registrar resultados financieros, se verifica que la autorización y las asignaciones utilizadas sigan vigentes y que el worker conserve el `fencingGeneration` correspondiente.
3. **Clasificación del resultado:** Se determina si cada invocación produjo un resultado técnicamente válido, parcial válido o un fallo técnico sin resultado válido, conforme a G.4.6.
4. **Despacho y atribución:** Cuando corresponda enviar una respuesta, se registra el resultado del intento de despacho y se distingue un fallo de integración/código propio de Waby de un fallo del proveedor externo del canal. Si Waby impide el despacho por un fallo propio, el consumo correspondiente no es facturable; si falla el proveedor externo del canal, el consumo LLM confirmado se mantiene facturable conforme a G.4.6.
5. **Persistencia y reconciliación:** Se conservan `AgentExecution`, `AgentExecutionLog`, usage por invocación y resultados de despacho necesarios para auditar la decisión. Si una invocación mantiene usage o resultado externo desconocido, permanece en `reconciliation_required`, retiene su asignación y no se liquida todavía el débito final de la ejecución.
6. **Vencimiento de reconciliación:** Si la invocación continúa sin evidencia concluyente al vencer los **30 días** desde su entrada en `reconciliation_required`, se excluye únicamente ese consumo no confirmado y se libera la capacidad de esa asignación. La evidencia tardía no produce un débito retroactivo automático.
7. **Liquidación financiera final:** Solo cuando todas las invocaciones estén resueltas se calcula el total facturable. Si es mayor que cero, se crea un único `deduct` por `executionId` y se actualiza `OrganizationAICredit.balance` junto con la liberación de `committedBudget` en una transacción MongoDB corta. Si no hay importe facturable, se cierra la autorización y libera la capacidad sin crear un débito de importe cero.

**Consecuencias de fallar en cada punto:**

| Punto de fallo | Consecuencia |
|---------------|--------------|
| Fallo al validar presupuesto o fencing antes de liquidar | No se registra un débito con una autorización obsoleta. El consumo eventualmente incurrido se trata según G.4.6 y no se factura sin autorización válida. |
| Timeout LLM con usage/resultado desconocido | La invocación permanece en `reconciliation_required`; conserva su capacidad durante 30 días y bloquea la liquidación final de esa ejecución mientras siga pendiente. |
| Fallo propio de Waby que impide despachar una respuesta técnicamente válida | El resultado LLM sigue siendo válido, pero el consumo correspondiente es absorbido por Waby; no se incluye en el débito al cliente. |
| Fallo del proveedor externo del canal después de una respuesta LLM válida | El consumo LLM confirmado se mantiene facturable; el fallo se registra como incidencia de entrega. |
| Fallo en la transacción de liquidación | Se revierte la transacción completa; no hay débito ni liberación parcial. Puede reintentarse con el mismo `executionId`. |
| Error interno posterior a una liquidación correcta que no afectó el resultado recibido | No anula el débito. Si se identifica posteriormente un fallo compensable conforme a G.4.6, se registra un asiento `refund` separado. |

**No confundir:** `committedBudget` es capacidad operativa comprometida, no un movimiento contable `hold` en `CreditLedger`. El débito financiero se registra una única vez, al final, por el consumo agregado que G.4.6 determina como facturable.

---

#### G.4.3 Modalidad On-Demand (administrada por Waby)

**Saldo disponible:** `OrganizationAICredit.balance - OrganizationAICredit.committedBudget` (créditos disponibles después de descontar el presupuesto comprometido por ejecuciones en curso).

**Límite adicional permitido (On-Demand):** Ninguno por defecto. Cada ejecución está limitada al costo estimado máximo definido por Waby (ej. máximo X créditos por ejecución). Si el saldo disponible es menor que el costo estimado máximo, la ejecución se rechaza.

**Prevención de sobreconsumo en On-Demand:** El mecanismo de presupuesto máximo autorizado (G.4.2) garantiza que la suma de presupuestos comprometidos nunca exceda el saldo disponible de la Organización. Mientras una ejecución tenga presupuesto autorizado, otras ejecuciones de Workspaces de la misma Organización pueden obtener su propio presupuesto siempre que quede saldo disponible.

**Cuando se alcanza el límite:** Si una ejecución alcanza su presupuesto máximo autorizado, se detiene la generación y se factura solo lo consumido hasta ese punto.

**Ejecuciones en curso cuando se agota el saldo:** Las ejecuciones que ya tienen presupuesto autorizado se completan y se liquidan normalmente. Las nuevas ejecuciones se rechazan por saldo insuficiente o por no poder obtener presupuesto autorizado.

**Consumo incluido vs On-Demand:** Todo el consumo se descuenta del saldo de la Organización. On-Demand es un modelo de facturación por uso real (no por suscripción fija), administrado exclusivamente por Waby. No existe categoría de "consumo adicional" separada por Workspace.

---

#### G.4.4 Identidad del movimiento e idempotencia

**Diferenciación de identidades:**

| Identidad | Qué representa | ¿Cambia en reintento? | Uso |
|-----------|---------------|----------------------|-----|
| `executionId` | Identificador de la ejecución lógica (AgentExecution.id) | No | Idempotency key para liquidación de débitos |
| `movementId` | Identificador único del asiento en CreditLedger (UUID generado al crear) | No | Clave primaria del asiento |
| `authorizationId` | Identificador de la autorización de presupuesto (ExecutionBudgetAuthorization.id) | No | Identifica la autorización de presupuesto asociada a una ejecución |
| `allocationId` | Identificador de la asignación de presupuesto (ExecutionBudgetAllocation.id) | No | Identifica una asignación de presupuesto para una invocación específica |
| `invocationId` | Identificador de una invocación concreta al proveedor LLM, incluidos reintentos contra el mismo proveedor/modelo | No | Identifica una invocación individual y su consumo dentro de una ejecución |
| `attemptId` | Intento/propietario de procesamiento de una ejecución o invocación | Cambia cuando una recuperación/reasignación entrega el recurso a otro worker | Permite identificar el intento propietario; no es una clave de idempotencia financiera |
| `fencingGeneration` | Número incremental por autorización o asignación | Incrementa solo al recuperar o reasignar la propiedad a un nuevo worker; una renovación normal del lease conserva la generación | Identifica la generación vigente de una autorización o asignación; usado en validación de liquidación |
| `refundRequestId` | Para asientos `refund`: clave de idempotencia propia de la solicitud de devolución | No | Permite reintentar una devolución sin duplicarla |
| Idempotency key (deduct) | Combinación `(organizationId, executionId)` | No | Restricción única en CreditLedger para movimientos `deduct` |
| Idempotency key (refund) | Combinación `(organizationId, refundRequestId)` | No | Restricción única en CreditLedger para movimientos `refund` |
| `originalMovementId` | Para asientos `refund`: referencia al `movementId` del débito original compensado | No | Identifica inequívocamente qué asiento del ledger se compensa |
| `originalExecutionId` | Referencia complementaria al `executionId` del débito original compensado | No | Relaciona el movimiento con su ejecución de negocio |

**Reglas de idempotencia:**

1. **Un débito por ejecución lógica:** Una misma `(organizationId, executionId)` no puede generar más de un movimiento `deduct` en `CreditLedger`. Si se intenta crear un segundo asiento `deduct` con la misma clave, se rechaza.

2. **Reintentos de liquidación:** Si una liquidación falla y se reintenta con el mismo `execution_id`, el sistema verifica si ya existe un asiento `completed` para esa combinación. Si existe, no se genera un nuevo asiento.

3. **Devoluciones idempotentes:** Una devolución (`refund`) usa `refundRequestId` como clave de idempotencia propia. Una misma `(organizationId, refundRequestId)` no puede generar más de un movimiento `refund`. Si se intenta crear un segundo asiento `refund` con la misma `refundRequestId`, se rechaza.

4. **Asignación de presupuesto por invocación:** Una misma `(authorizationId, invocationId)` con `status` en `('assigned', 'executing', 'settling', 'reconciliation_required')` no puede generar más de una `ExecutionBudgetAllocation` que retenga capacidad. Si se intenta crear un segundo asiento de asignación con la misma clave, se rechaza.

5. **Alcance de las restricciones únicas:** La restricción única `(organizationId, executionId)` aplica **solo a movimientos `deduct`**. La restricción única `(organizationId, refundRequestId)` aplica **solo a movimientos `refund`**. Los movimientos `purchase` no están sujetos a restricción única de ejecución. Esto evita bloquear accidentalmente operaciones legítimas.

**Reintentos técnicos del mismo proveedor:**
- Un reintento técnico contra el mismo proveedor/modelo dentro de la misma ejecución lógica conserva el `executionId` y utiliza otro `invocationId`; sus consumos se clasifican individualmente y se consolidan en el único débito final de esa ejecución. No se contempla cambiar a un proveedor alternativo.
- Solo una nueva ejecución de negocio independiente utiliza otro `executionId`, con su propia autorización y su propio débito final.
- Si una ejecución falla después de haber debitado créditos, no se modifica el débito original; cualquier compensación aprobada se registra mediante un nuevo asiento `refund` conforme a G.4.5 y G.4.6.

**Consistencia de transacción, ledger y saldo:**
- `CreditLedger` y `OrganizationAICredit` residen en la **misma base de datos del CRM** (colecciones en el mismo deployment de MongoDB). La transacción atómica de MongoDB abarca ambas escrituras en una sola operación `commitTransaction`.
- **Condiciones para la atomicidad:** Ambas colecciones deben estar en el mismo clúster de MongoDB y en el mismo deployment de la aplicación CRM. No se asume atomicidad entre bases de datos independientes, servicios distintos, o clústeres de MongoDB separados.
- **No se asume atomicidad entre bases de datos independientes o servicios distintos.** La autoridad del saldo reside en `OrganizationAICredit` (CRM). El AI Service ejecuta el débito a través de operación interna (`deductCredits`) que escribe al CRM.
- Si el AI Service cae después de escribir en `CreditLedger` pero antes de confirmar la transacción, la transacción se revierte y no hay débito parcial.
- Si el AI Service cae después de confirmar la transacción pero antes de notificar al cliente, el débito persiste y es idempotente (reintento con mismo `execution_id` no duplica).

**Validación atómica de devoluciones parciales:** Antes de registrar un asiento `refund`, el sistema verifica dentro de la misma transacción que `SUM(amount de refunds existentes para originalMovementId) + amount del nuevo refund <= amount del movimiento de débito original referenciado`. Esta verificación y la escritura ocurren en la misma transacción de MongoDB para prevenir sobre-devoluciones bajo concurrencia.

---

#### G.4.5 Ledger append-only y devoluciones

**`CreditLedger` es append-only:** Los asientos nunca se modifican ni se eliminan una vez registrados.

**Tipos de movimiento:**
- `deduct`: Débito por ejecución facturable.
- `refund`: Devolución de créditos (asiento compensatorio, nuevo registro).
- `purchase`: Compra de créditos (saldo añadido).
- `adjustment`: Ajuste administrativo del saldo, cuando esté habilitado.

**Estado `status`:**
- `completed`: El movimiento se aplicó correctamente.
- `failed`: El movimiento falló (no se aplicó).
- **`reversed` eliminado:** No se usa. Las reversiones son asientos `refund` nuevos, no cambios de estado de asientos existentes.

**Devoluciones parciales y múltiples:**

- **Devolución completa:** Un asiento `refund` por el monto total del débito original.
- **Devolución parcial:** Un asiento `refund` por un monto menor al débito original.
- **Múltiples devoluciones:** Se pueden registrar varios asientos `refund` para el mismo `originalMovementId`. La suma de todos los asientos `refund` asociados a ese movimiento **no puede superar** el monto originalmente debitado.

**Garantía de no sobre-devolución:** Antes de registrar un asiento `refund`, el sistema verifica que `suma(refunds_existentes) + monto_nuevo <= monto_debito_original`. Esta verificación ocurre dentro de la misma transacción atómica que escribe el asiento `refund`.

**Referencia al débito original:** Cada asiento `refund` lleva `originalMovementId` que apunta al `movementId` del débito compensado y `originalExecutionId` como referencia complementaria a la ejecución asociada. El débito original permanece inmutable en el ledger.

**Decisión sobre devoluciones:** El modelo permite devoluciones completas y parciales, con múltiples asientos `refund` sobre un mismo débito. Si el producto no necesita devoluciones parciales en una fase inicial, esta capacidad puede dejarse sin usar sin afectar la integridad del modelo.

---

#### G.4.6 Evento facturable, fallos y política de facturación

**Principio rector.** Se distinguen cinco conceptos que no se confunden:
1. **Consumo técnico reconocido:** usage/tokens informados por el proveedor LLM y convertidos a créditos según `pricingVersion`.
2. **Validez técnica del resultado:** determina si la salida del proveedor cumple el contrato técnico de la invocación y puede procesarse como resultado. Es independiente del éxito o fallo del despacho al canal.
3. **Despacho al canal:** permite determinar si el mensaje fue aceptado por el proveedor del canal y atribuir la responsabilidad de un fallo de envío.
4. **Entrega efectiva y lectura:** estados posteriores al despacho, sujetos a las capacidades de cada canal (no todos los canales los exponen de forma fiable).
5. **Liquidación financiera:** determina el importe que corresponde debitar a la Organización.

La aceptación del canal, el estado `delivered` y la confirmación de lectura **no son requisitos universales** para reconocer el consumo LLM. Tampoco el despacho exitoso es, por sí solo, una condición general de facturación; su resultado sirve para atribuir responsabilidades y determinar eventuales compensaciones.

**Política aplicable (híbrida):** se debita al cliente el consumo técnico confirmado por el proveedor LLM por invocación, salvo cuando no exista un resultado técnicamente válido por un fallo técnico del proveedor LLM o por un fallo propio de Waby que impida producir el resultado o completar el despacho bajo responsabilidad de Waby, en cuyo caso Waby absorbe ese consumo. Waby no absorbe consumo provocado por el cliente ni por fallos de servicios externos ajenos a la generación LLM, siempre que exista evidencia suficiente para atribuir la causa.

**Criterios técnicos verificables de clasificación del resultado:**

| Clasificación | Definición técnica | Tratamiento financiero |
|---|---|---|
| Resultado válido | El proveedor devuelve una respuesta que cumple el contrato técnico de la invocación y puede procesarse como resultado. | Facturable según consumo confirmado, sujeto a la atribución del despacho definida en esta sección. |
| Fallo técnico sin resultado válido | Error técnico del proveedor, respuesta vacía, estructura inválida o interrupción que no produce un resultado técnicamente aprovechable. No incluye un rechazo estructurado válido ante una solicitud prohibida atribuible al cliente. | No facturable al cliente; Waby absorbe el consumo confirmado. |
| Resultado parcial válido | Existe una salida parcial que cumple el contrato técnico de la invocación y puede procesarse como resultado parcial. | Se reconoce únicamente el usage confirmado por el proveedor para esa invocación, convertido mediante `pricingVersion`. |
| Disconformidad con calidad o pertinencia | Resultado técnicamente válido que el cliente considera insuficiente o inapropiado, sin fallo técnico comprobado. | No genera reembolso automático. Queda fuera de la liquidación automática y, si corresponde, se trata mediante una política comercial o de soporte independiente. |

**Matriz de comportamiento por situación:**

| Situación | Consumo técnico reconocido | Resultado válido | Despacho | ¿Débito al cliente? | Tratamiento |
|---|---|---|---|---|---|
| Error antes de despachar una invocación al proveedor LLM | No hay consumo de esa invocación | No aplica | No aplica | No | No hay uso que facturar. |
| Proveedor LLM devuelve error y confirma que no hubo usage | Consumo reconocido cero | No | No aplica | No | No hay uso que facturar. |
| Fallo técnico del proveedor LLM sin resultado válido | Puede existir consumo confirmado | No | No aplica | No | Waby absorbe el consumo confirmado del proveedor. |
| Solicitud inválida o prohibida atribuible al cliente | Puede existir consumo confirmado | Puede existir un rechazo estructurado conforme al contrato | Puede existir | Sí | El cliente asume el consumo confirmado cuando existe evidencia técnica suficiente de que la solicitud inválida/prohibida es atribuible al cliente. |
| Rechazo estructurado del proveedor conforme a su contrato | Puede existir consumo confirmado | Sí, si cumple el contrato técnico | Puede existir | Sí, cuando corresponde a una solicitud atribuible al cliente | Se trata como resultado válido; no se confunde automáticamente con un error técnico del proveedor. |
| Resultado técnicamente válido generado y despachado al canal | Consumo confirmado | Sí | Sí | Sí | Débito por consumo confirmado. |
| Resultado técnicamente válido generado, falla el despacho por un fallo propio de Waby | Consumo confirmado | Sí | No | No | El resultado sigue siendo técnicamente válido; el despacho falló por responsabilidad de Waby y Waby absorbe el consumo correspondiente. |
| Resultado técnicamente válido generado, falla el despacho por el proveedor externo del canal | Consumo confirmado | Sí | No | Sí (consumo LLM) | El débito por consumo LLM se mantiene; el fallo del proveedor del canal se trata como incidencia de entrega y no anula el crédito por consumo LLM. |
| Fallo interno de Waby durante la generación o procesamiento que impide obtener un resultado técnicamente válido | Puede existir consumo confirmado | No | No aplica | No | Waby absorbe el consumo correspondiente. |
| Resultado válido generado y despachado correctamente; error interno posterior de Waby que no afectó el resultado recibido | Consumo confirmado | Sí | Sí | Sí | El error posterior no anula el débito. |
| Timeout con resultado parcial válido confirmado | Consumo parcial confirmado | Sí (parcial) | Puede existir | Sí | Se factura el usage realmente reconocido para esa invocación, convertido mediante `pricingVersion`. |
| Timeout, consumo confirmado, pero no existe resultado válido por fallo técnico del proveedor | Consumo confirmado | No | No aplica | No | Waby absorbe ese consumo. |
| Timeout, consumo confirmado, pero Waby interrumpe la generación/procesamiento antes de producir un resultado técnicamente válido | Consumo confirmado | No | No aplica | No | Waby absorbe el consumo correspondiente al fallo bajo su responsabilidad. El caso de resultado válido con fallo de despacho está cubierto por la fila específica de despacho fallido por Waby. |
| Timeout sin confirmar consumo ni conocer el resultado | Consumo potencial no confirmado | Desconocido | Desconocido | No (pendiente) | `reconciliation_required`; se conserva la capacidad comprometida y no se liquida el débito final de la ejecución mientras siga pendiente. |
| Reintento contra el mismo proveedor dentro de la misma ejecución lógica | Usage por `invocationId` | Según cada invocación | Según cada invocación | Solo el consumo clasificado como facturable se agrega al débito final | Se conserva el mismo `executionId`; cada invocación utiliza su propio `invocationId`. |
| Nueva ejecución de negocio independiente | Usage por `invocationId` | Según cada invocación | Según cada invocación | Solo el consumo facturable se agrega al débito final de esa ejecución | Utiliza su propia autorización y clave de ejecución. |
| Disconformidad con calidad o pertinencia sin fallo técnico | Consumo confirmado | Sí (técnicamente válido) | Puede existir | Sí | No genera reembolso automático; se trata vía política comercial o soporte independiente. |

**Atribución de responsabilidades por fallo:**

| Origen del fallo | Ejemplo | ¿Waby absorbe el consumo? |
|---|---|---|
| Fallo de integración o código de Waby | Error interno que impide obtener un resultado técnicamente válido o completar el despacho bajo responsabilidad de Waby | Sí, para el consumo correspondiente. |
| Fallo del proveedor externo del canal | Canal WhatsApp caído o error de API de Evolution/Meta | No anula el débito por consumo LLM confirmado; se trata como incidencia de entrega. |
| Fallo del proveedor LLM | Error del modelo, respuesta vacía, interrupción sin resultado técnicamente válido | Sí, cuando no existe resultado técnicamente válido. |
| Prompt o entrada del cliente | Solicitud prohibida o mal formada atribuible al cliente | No, si existe evidencia técnica suficiente; el cliente asume el consumo confirmado. |

El estado técnico `failed` por sí solo no determina el tratamiento financiero. La validez del resultado y la atribución del despacho son dimensiones distintas.

**Liquidación e idempotencia:**
- Cada `invocationId` se clasifica individualmente en: consumo confirmado y facturable, consumo confirmado absorbido por Waby, o consumo desconocido pendiente de reconciliación.
- Las invocaciones facturables, las absorbidas por Waby y las pendientes de reconciliación se distinguen entre sí.
- La existencia de una invocación pendiente no elimina ni modifica el consumo facturable de las invocaciones ya resueltas.
- Sin embargo, el débito final único por `executionId` espera hasta que todas las invocaciones de esa ejecución estén resueltas.
- Mientras exista una invocación en `reconciliation_required`, se conserva el compromiso de capacidad correspondiente a esa operación y no se realiza la liquidación final de la ejecución.
- Al resolverse todas las invocaciones, se consolida el consumo facturable y se registra como máximo un único `deduct` por `executionId`; si el total facturable es cero, no se crea un débito de importe cero.
- Si vence el período de gracia sin evidencia suficiente, se excluye únicamente el consumo desconocido, se libera la capacidad correspondiente y se liquida el consumo facturable confirmado de las demás invocaciones, una vez que todas las invocaciones de la ejecución tengan un tratamiento final.
- No se introducen débitos parciales, segundos débitos ni modificaciones del diseño append-only.
- La clave idempotente del débito es `(organizationId, executionId, type=deduct)`.
- El consumo de una respuesta parcial válida se calcula exclusivamente con el usage realmente reconocido por el proveedor para esa invocación, convertido mediante `pricingVersion`. No se infiere a partir de la longitud de la respuesta, el porcentaje generado o el presupuesto asignado.
- Si un fallo se conoce **antes del débito**, el importe facturable se determina directamente según esta política (cero, consumo confirmado parcial o consumo confirmado completo); no se crea un asiento `deduct` por importes que ya se sabe que no son facturables.
- Si un fallo compensable se conoce **después del débito**, se registra un asiento `refund` idempotente con `originalMovementId`, `originalExecutionId` y `refundRequestId`. La suma de los refunds no puede superar el débito original. La validación del límite de devolución y la actualización del saldo se serializan de forma segura en una transacción MongoDB.

**Reembolsos y compensaciones:** cuando un fallo atribuible a Waby o al proveedor LLM sin resultado válido se identifica después de un débito y corresponde una compensación conforme a esta política, se crea un asiento `refund` nuevo con `originalMovementId`, `originalExecutionId` y `refundRequestId`. Los reembolsos son asientos nuevos, append-only; no existe estado `reversed`.

**Reconciliación de proveedor:** un timeout cuyo resultado externo es desconocido requiere reconciliación basada en capacidades reales del proveedor (request ID, consulta de usage/operación, respuesta tardía o soporte del proveedor). No se promete cancelar llamadas ya aceptadas. No se inventan capacidades de consulta de uso ni reconciliación que el proveedor no ofrezca.

**Operaciones sin información concluyente (período de gracia):** el período de gracia aprobado es de **30 días naturales**, contados desde la entrada de la invocación en `reconciliation_required`. Durante ese período, la operación conserva su asignación y capacidad comprometida; no se liquida ni se libera capacidad prematuramente. Al vencer los 30 días sin evidencia suficiente, se excluye el consumo no confirmado, se libera la capacidad correspondiente y se registra evidencia de auditoría. La ejecución podrá liquidarse cuando todas sus invocaciones tengan un tratamiento final; el consumo confirmado y facturable de las demás invocaciones se conserva. La evidencia posterior no habilita un débito retroactivo automático; cualquier ajuste posterior requiere autorización administrativa/comercial expresa.

**Alcance de P4:** P4 controla exclusivamente créditos por consumo de proveedores LLM. Las ejecuciones HTTP externas, Skills, APIs externas y otras operaciones no-LLM quedan fuera de esta política, salvo una decisión futura explícita y separada.

### G.5 Logs de auditoría (I21)

Cada ejecución genera `AgentExecution` + `AgentExecutionLog` con:

| Campo | Descripción |
|-------|-------------|
| execution_id | Identificador único (llave de idempotencia) |
| organization_id | Organización propietaria del saldo |
| agent_instance_id | Instancia que atendió |
| workspace_id | Workspace asociado (dimensión de atribución) |
| prompt_snapshot_hash | SHA-256 del prompt efectivo normalizado |
| effective_config_hash | SHA-256 de la configuración efectiva completa (P10) |
| effective_config_snapshot | Copia persistida de la configuración efectiva (sin secretos) para reproducibilidad (P7, P10) |
| provider_model_used | Proveedor y modelo realmente usado |
| input_tokens / output_tokens / total_tokens | Conteo de tokens |
| total_credits_deducted | Créditos descontados |
| latency_ms | Latencia de ejecución |
| resultado normalizado | Resultado sin detalles internos de routing LLM |

**No se exponen modelos/versiones/parámetros al cliente** (I9, I21).

**Nota de trazabilidad:** `prompt_snapshot_hash` identifica el prompt; `effective_config_hash` identifica la configuración completa. Ninguno permite reconstruir la ejecución por sí solo; para reproducibilidad completa se requiere `effective_config_snapshot` o las versiones inmutables de AgentDefinition, Rules y Skills referenciadas.

### G.6 Métricas por instancia y Organización (I20, P5)

| Nivel | Métrica | Fórmula | Ventana |
|-------|---------|---------|---------|
| Organización | Créditos consumidos | Suma de totalCreditsDeducted | Mes calendario UTC (o 30 días móviles, P5) |
| Organización | Saldo disponible | `OrganizationAICredit.balance` | Tiempo real |
| Workspace | Créditos consumidos | Suma de totalCreditsDeducted por `workspace_id` | Mes calendario UTC |
| Workspace | Ejecuciones | Conteode `AgentExecution` por `workspace_id` | Mes calendario UTC |
| AgentInstance | Créditos consumidos | Suma de totalCreditsDeducted por `agent_instance_id` | Mes calendario UTC |
| AgentInstance | Ejecuciones | Conteo de `AgentExecution` por `agent_instance_id` | Mes calendario UTC |
| AgentInstance | Tokens utilizados | Suma de totalTokens por `agent_instance_id` | Mes calendario UTC |
| AgentInstance | Latencia y resultados | Promedio de `latencyMs`, conteo de `status` | Mes calendario UTC |

Los desgloses por Workspace y AgentInstance son estadísticas y atribución. No representan saldos individuales ni límites comerciales por Workspace.

---

## H. Migración y compatibilidad

### H.1 Matriz conservar / adaptar / separar / reemplazar / retirar

| Elemento actual | Acción | Justificación | Invariantes |
|-----------------|--------|---------------|-------------|
| `Bot` (definición + instancia mezcladas) | Separar → `AgentDefinition` (Waby) + `AgentInstance` (Workspace) | `Bot` mezcla definición e instancia (Fase 1) | I1, I2, I16 |
| `BotInstance` | Adaptar → `AgentInstance` (propiedad Workspace) | Ya existe, cambiar propiedad y campos | I2, I16 |
| `ModelConfiguration` por bot | Reemplazar → routing central Waby | Configurable por bot hoy, debe ser Waby | I18 |
| Credencial per-bot (D*-cred) | Reemplazar → credenciales LLM centralizadas Waby | Cliente gestiona API keys de proveedor LLM hoy (violación I6) | I6 |
| `enabledCapabilities` global en pipeline | Separar → filtrar por instancia | `findAll()` carga todas (Fase 1, R7) | I8, I12 |
| RAG filtro `config.workspaceId` | Conservar → `WorkspaceKnowledge` | Cumple I11 (compartido por workspace) | I11 |
| `memory-manager` (30 facts) | Reemplazar → `ConversationMemory` (ventana 20) | No existe ventana 20 (Fase 1, R10) | I14 |
| `Execution` (workspaceId, model, status) | Conservar + ampliar → campos I20, I21 | Necesario para auditoría y créditos | I20, I21 |
| `delete-wrk` (solo DB crm) | Adaptar → limpiar ai_service, Qdrant, MinIO | No toca recursos de agente hoy (Fase 1, R14) | Invariante 13 |
| Sistema de créditos | Crear → `OrganizationAICredit` + `CreditLedger` | No existe hoy (Fase 1, R1) | I17, I20, I21 |
| Policy Engine | Crear → políticas no delegables | No existe hoy (Fase 1, R8) | I8 |
| Output filter | Crear → filtrar respuesta antes de enviar | No existe hoy (Fase 1, R11) | I8, I9 |

**Frontera de persistencia objetivo (ARQ-14):** AgentInstance y SkillConfiguration/AgentRules persisten en CRM; AgentDefinition, WorkspaceKnowledge/documentos, ConversationMemory y AgentExecution/AgentExecutionLog persisten en AI Service DB. Qdrant es índice derivado, MinIO almacena binarios y CRM conserva los créditos/ledger. La matriz normativa completa está en D.1.9. La persistencia de Knowledge se mantiene en AI Service DB; no se introduce una migración de fuente de verdad a CRM.

### H.2 Correspondencia Bot → AgentDefinition / AgentInstance

| Campo de Bot actual | Destino en modelo objetivo | Notas |
|---------------------|---------------------------|-------|
| id | AgentInstance.id | La instancia hereda el ID del bot o se genera nuevo |
| workspaceId | AgentInstance.workspaceId | Denormalizado, validado cross-tenant |
| name | AgentInstance.name | Nombre visible al cliente |
| model (provider, name, temp, etc.) | AgentDefinition (campos de proveedor) | Waby centraliza, migra a definición |
| aiCredential | Eliminado | Credenciales de proveedores LLM centralizadas en Waby |
| enabledCapabilities | SkillConfiguration (por instancia) | Migrado como skills de la instancia |
| knowledgeBaseIds | AgentInstance.knowledgeBaseIds | Referencias a WorkspaceKnowledge |
| promptId / activePromptVersionId | AgentDefinition (prompt del sistema) | La definición lleva el prompt base |
| status | AgentInstance.status | `active` / `inactive` → `active` / `suspended` |
| temperature, maxIterations | AgentDefinition + AgentInstance override | Hiperparámetros centralizados en definición |

### H.3 Migración de BotInstance

Cada `BotInstance` actual se migra a `AgentInstance`:
- `botId` → `agentDefinitionId` (el Bot se convierte en AgentDefinition)
- `workspaceId` → mantenido (denormalizado)
- Campos sobrescribibles (modelConfig, temperature, systemPrompt, enabledCapabilities) → migrados a SkillConfiguration / AgentRules / additionalPrompt de la instancia

### H.4 Knowledge y Qdrant

- `knowledge_bases` → `WorkspaceKnowledge` (misma estructura, propiedad de workspace)
- Vectores Qdrant: migrados manteniendo filtro por `workspaceId` (BR-KB-001 ya cumple I11)
- Al eliminar una WorkspaceKnowledge: invalidar embeddings asociados en Qdrant (purge automático)
- No exigir `agentInstanceId` como filtro en Qdrant (I11)

### H.5 Skills y Rules

- `bot_resources` / `bot_resource_links` → `SkillConfiguration` y `AgentRules` (por instancia)
- Cada recurso vinculado a un bot se migra como skill/rule de la instancia correspondiente
- Sin herencia ni compartición implícita entre instancias (I12, I13)

### H.6 MemoryFact

- `memory_facts` (30 facts por contacto) → `ConversationMemory` (ventana 20 mensajes user/assistant por conversación)
- Los facts existentes se pueden conservar como metadata histórica, pero la memoria operativa usa la nueva estructura
- Sin resumen automático (I14)

### H.7 Credenciales

- `aiCredential` per-bot → Eliminado. Credenciales de proveedores LLM gestionadas centralmente por Waby.
- Los clientes ya no aportan API keys (I6).

### H.8 Execution y costos

- `execution` → `AgentExecution` + `AgentExecutionLog` (campos I21 adicionales)
- `execution.cost` (estimación legacy) → `totalCreditsDeducted` (débito real)
- Campos nuevos: `prompt_snapshot_hash`, `provider_model_used`, `latency_ms`

### H.9 Compatibilidad temporal

- Durante la migración, ambos modelos pueden coexistir: los bots existentes siguen funcionando mientras se migran a AgentDefinition/AgentInstance.
- El frontend debe mostrar solo instancias habilitadas al cliente (no las definiciones).
- Las APIs internas de ai-service (`/api/bots/*`) se deprecian gradualmente a medida que se migran a las nuevas operaciones GraphQL.

### H.10 Backup y rollback

- Backup completo de MongoDB (crm + ai_service) antes de iniciar la migración.
- Backup de vectores Qdrant y buckets MinIO afectados.
- El backup es un punto de recuperación y evidencia; no autoriza por sí solo restaurar una copia anterior sobre datos que ya recibieron asignaciones o transiciones confirmadas.
- Antes de activar asignaciones de AgentInstance, puede recuperarse el modelo Bot/BotInstance si se verifican las condiciones previas de H.11. Después de la activación, cualquier rollback debe cumplir las condiciones e invariantes de H.11; si la restauración implica pérdida de asignaciones, historial o cambios concurrentes, se detiene y se corrige hacia adelante.
- Migración por workspace (por lotes) para reducir riesgo y permitir validación incremental.

### H.11 Migración del modelo unificado de Contacto (WAB-78 — orden aprobado)

**Orden aprobado:**

1. Incorporar campos opcionales `assignedType` y `ref_agent_assigned` en `Contacto`.
2. Adaptar lecturas y escrituras manteniendo compatibilidad con el modelo anterior.
3. Desplegar código compatible en CRM, ai-service y workers.
4. Ejecutar backfill idempotente: UOW presente → `human`; UOW nulo → `unassigned`; `ref_agent_assigned = null` para todos los contactos existentes.
5. Verificar invariantes post-backfill.
6. Activar routing de AgentInstance basado en el nuevo modelo.
7. Retirar compatibilidad antigua solo cuando ningún consumidor dependa del modelo anterior.

**Mapeo aprobado:**
- UOW presente → `human`
- UOW nulo → `unassigned`
- `ref_agent_assigned` nulo para todos los contactos existentes

**Nota:** El usuario confirmó que no hubo asignaciones productivas anteriores de bots (solo una prueba posteriormente eliminada). No ejecutar la migración en esta fase.

**Condiciones e invariantes de rollback:**

- **Antes de activar asignaciones de AgentInstance:** el rollback del código nuevo solo puede ejecutarse tras verificar que no existen contactos asignados a agentes ni ejecuciones/trabajos pendientes que dependan del nuevo modelo. Se preserva íntegramente `ref_user_assigned`; los campos nuevos solo pueden retirarse si ningún consumidor desplegado los necesita.
- **Después de activar asignaciones de AgentInstance:** no se permite volver a una versión que solo entienda `ref_user_assigned`. El destino del rollback debe seguir interpretando ambos tipos de asignación o existir una conversión explícita, validada y sin pérdida para cada contacto afectado. No se permite convertir silenciosamente asignaciones `agent` en `human` o `unassigned`, ni descartar referencias a AgentInstance.
- **Cambios concurrentes durante la migración o reversión:** las nuevas transiciones de asignación deben quedar serializadas/bloqueadas durante la ventana crítica o registrarse de forma que puedan reconciliarse idempotentemente. El snapshot inicial no puede sobrescribir cambios legítimos ocurridos después de su captura.
- **Integridad y verificación:** antes de continuar se comparan los registros y conteos por tipo de asignación, se validan ownership Contacto–Workspace–AgentInstance, ausencia de referencias huérfanas y conservación del historial `EvAssigned`. La verificación debe cubrir también ejecuciones y trabajos pendientes vinculados a la asignación.
- **Condición de parada:** si no puede garantizarse la representación íntegra de todas las asignaciones, la conservación del historial o la reconciliación de cambios concurrentes, se detiene el rollback. Se mantiene una versión compatible y se corrige hacia adelante; no se restaura una copia antigua de datos que pueda perder asignaciones o transiciones confirmadas.

**Detalles operativos pendientes para WAB-79:** ventana exacta, mecanismo de serialización/reconciliación, procedimiento de ejecución y criterios cuantitativos de aceptación. No ejecutar la migración ni el rollback en WAB-78.

---

## I. Estrategia de implementación

### I.1 Orden de fases posteriores

La implementación se organiza por dependencias arquitectónicas. Los estados y el orden operativo vigentes se consultan en Linear; el diseño de esta fase no cambia esos estados.

| Issue | Alcance | Estado |
|---|---|---|
| WAB-79 | Dominio, ownership, persistencia e índices | Backlog; bloqueada por WAB-78 |
| WAB-80 | Provisioning y autorización | Backlog |
| WAB-81 | Runtime y composición de contexto | Backlog |
| WAB-82 | Knowledge compartido y Skills/Rules por agente | Backlog |

Las decisiones P1/P2/P3 ya no son decisiones de producto pendientes: están sustituidas por ARQ-19/20 y DEC-CM-01..08. P4 está aprobada. Los detalles técnicos pendientes de WAB-78 deben cerrarse antes de autorizar la fase siguiente. No iniciar WAB-79 sin autorización expresa.

### I.2 No fragmentar artificialmente

El trabajo se agrupa por responsabilidad arquitectónica (datos, provisioning, runtime, centralización, frontend, migración, créditos, pruebas), no por carpetas ni microservicios. Cada fase puede tocar código en CRM, ai-service y frontend según sea necesario.

---

## J. ADRs (Architecture Decision Records)

### ADR-001: Separación AgentDefinition / AgentInstance

**Estado:** Dirección conceptual aprobada; detalle de persistencia diferido a WAB-79.
**Fecha de decisión:** 06/11/2026
**Decisión:** Separar el modelo actual Bot (que mezcla definición e instancia) en dos entidades: AgentDefinition (propiedad Waby) y AgentInstance (propiedad Workspace).

**Alternativas consideradas:**
1. Mantener Bot tal cual y agregar campos para distinguir definición/instancia — Rechazado: no resuelve la mezcla de propiedad, no permite unicidad adecuada, ni separación Waby/cliente.
2. Crear AgentDefinition como entidad separada y adaptar BotInstance a AgentInstance — Aceptado: separación clara de propiedad y cumplimiento de I1, I2, I16.

**Consecuencias:**
- Requiere migración de datos de Bot a AgentDefinition + AgentInstance.
- Los clientes ya no crean ni editan definiciones (solo instancias).
- Waby gestiona proveedor, modelo y credenciales LLM centralizadas.

---

### ADR-002: Bolsa única de créditos por Organización, presupuesto comprometido y liquidación final

**Estado:** Aprobado por el usuario el 03/11/2026
**Fecha de decisión:** 03/11/2026
**Decisión:** La Organización es propietaria de una única bolsa `OrganizationAICredit`, compartida por todos sus Workspaces. `CreditLedger` es append-only. Antes de ejecutar se compromete capacidad de forma operativa mediante `committedBudget` y `ExecutionBudgetAuthorization`; no se crean asientos contables `hold`/`release`. Cada invocación LLM obtiene una `ExecutionBudgetAllocation` propia. Los consumos se reconocen por invocación, pero se registra como máximo un débito final por `executionId`, una vez que todas las invocaciones están resueltas.

**Alternativas consideradas:**
1. Créditos por instancia — Rechazado: viola I17 (bolsa única por Organización).
2. Créditos por Workspace — Rechazado: los créditos pertenecen a la Organización y son compartidos por todos sus Workspaces.
3. Asientos `hold` → `deduct` en `CreditLedger` — Rechazado: añade estados contables intermedios y riesgo de holds huérfanos. El control de concurrencia se implementa mediante presupuesto operativo comprometido fuera del ledger.
4. Débito directo con solo validación previa — Rechazado como garantía completa: no evita que ejecuciones concurrentes consuman recursos del proveedor antes de liquidar.
5. Lock exclusivo por Organización durante toda la ejecución — Rechazado: serializaría todas las ejecuciones de una Organización y reduciría la concurrencia entre Workspaces.
6. Presupuesto máximo autorizado por ejecución y asignación por invocación — Aceptado: reserva capacidad de forma operativa, permite concurrencia entre Workspaces y limita el consumo antes de invocar proveedores.

**Consecuencias:**
- `OrganizationAICredit.balance` es la autoridad del saldo. `organizationId` es el ámbito contable; `workspaceId` y `agentInstanceId` son dimensiones de atribución.
- `committedBudget` incluye autorizaciones en estado `authorized`, `executing`, `settling` o `reconciliation_required`; no es un movimiento de `CreditLedger`.
- `activeAllocatedBudget` en cada autorización es la suma persistida de `assignedBudget` de asignaciones en `assigned`, `executing`, `settling` o `reconciliation_required`.
- El presupuesto disponible para nuevas invocaciones es `authorizedBudget - consumedAmount - activeAllocatedBudget`.
- La liquidación de cada invocación actualiza el consumo reconocido y libera su asignación, sin modificar el `assignedBudget` histórico.
- El débito final espera hasta que todas las invocaciones estén resueltas; el débito único usa `(organizationId, executionId, type=deduct)`. Si el importe facturable es cero, no se crea un asiento de débito de importe cero.
- Las operaciones `deduct`, `refund`, `purchase` y los ajustes que modifican el saldo se realizan con controles atómicos en MongoDB. `CreditLedger` y `OrganizationAICredit` residen en el mismo deployment para que la transacción abarque ambas escrituras.
- La política de facturación híbrida, los casos de absorción por Waby, las respuestas parciales y la reconciliación se definen en G.4.6. P4 cubre exclusivamente consumo LLM; Skills HTTP y operaciones no-LLM quedan fuera.
- El período de gracia para operaciones en `reconciliation_required` es de **30 días naturales** desde la entrada en ese estado. Al vencimiento sin evidencia suficiente se excluye el consumo desconocido y se libera su capacidad; no hay débito retroactivo automático.
- Los reintentos técnicos contra el mismo proveedor/modelo conservan `executionId` y usan un `invocationId` distinto por llamada real al proveedor. No se habilita fallback ni rotación entre proveedores. Una ejecución de negocio independiente utiliza un nuevo `executionId`.


---

### ADR-006: Presupuesto autorizado por ejecución con fencing por autorización e invocación (P4)

**Estado:** Aprobado por el usuario el 03/11/2026
**Fecha de decisión:** 03/11/2026
**Decisión:** Cada ejecución lógica obtiene una `ExecutionBudgetAuthorization` y cada invocación LLM una `ExecutionBudgetAllocation`. Cada recurso tiene su propio `fencingGeneration`. La generación cambia solo cuando la propiedad se recupera/reasigna a otro worker; la renovación normal del lease conserva la generación. Una escritura sobre una asignación valida tanto su fencing como el de la autorización padre.

**Alternativas consideradas:**
1. Solo transacción atómica de débito, sin control de presupuesto — Rechazado: no evita que ejecuciones concurrentes consuman recursos del proveedor antes de liquidar.
2. Lock exclusivo por Organización durante toda la ejecución — Rechazado: serializa ejecuciones entre Workspaces de una misma Organización.
3. Asientos `hold` → `deduct` en el ledger — Rechazado: el presupuesto se controla operacionalmente mediante `committedBudget`, no con asientos contables intermedios.
4. Sobreconsumo concurrente acotado — Rechazado como garantía técnica; solo sería posible como política comercial explícita separada.
5. Presupuesto autorizado por ejecución + asignaciones por invocación con fencing individual — Aceptado.

**Mecanismo aprobado:**
- **Autorización:** validar atómicamente `OrganizationAICredit.balance - committedBudget >= authorizedBudgetSolicitado`; incrementar `committedBudget` y crear la autorización.
- **Asignación:** validar atómicamente `authorizedBudget - consumedAmount - activeAllocatedBudget >= assignedBudgetSolicitado`; incrementar `activeAllocatedBudget` y crear la asignación.
- **Liquidación de invocación:** sumar el usage reconocido a `consumedAmount` de la autorización, restar el `assignedBudget` completo de `activeAllocatedBudget` y marcar la asignación `settled`. El `assignedBudget` histórico no se modifica; la parte no consumida vuelve a estar disponible.
- **Operación desconocida:** `reconciliation_required` retiene el `assignedBudget` íntegro durante 30 días naturales. Sin evidencia al vencimiento, se excluye el consumo desconocido y se libera la asignación; no se cobra retroactivamente sin autorización expresa.
- **Liquidación final:** cuando todas las asignaciones de la ejecución están en estado terminal, se agrega el importe facturable de sus invocaciones y se registra como máximo un débito por `executionId`; se libera el presupuesto comprometido de la autorización.
- **Fencing:** recuperar/reasignar a un nuevo worker incrementa `fencingGeneration`; renovación normal del lease no lo incrementa. La validación de una asignación también comprueba el fencing y ownership de la autorización padre.
- **Límite del mecanismo:** fencing evita escrituras financieras obsoletas en MongoDB, pero no cancela llamadas externas ya aceptadas por proveedores; estas se resuelven con idempotencia y reconciliación.

**Alcance:** esta ADR gobierna únicamente presupuesto y concurrencia para consumo de proveedores LLM en P4. Las ejecuciones HTTP externas, Skills y otras operaciones no-LLM quedan excluidas.


---

### ADR-003: Knowledge como rol no-sistema en el contexto

**Estado:** Contrato conceptual aprobado en el diseño de seguridad; implementación diferida a WAB-81/WAB-82.
**Fecha de decisión:** 06/11/2026
**Decisión:** Los fragmentos de Knowledge viajan como rol `user` o `knowledge` dedicado, nunca como `system`. El system prompt (AgentDefinition + Policy Engine) tiene precedencia absoluta.

**Alternativas consideradas:**
1. Mantener Knowledge como `system` (modelo actual) — Rechazado: riesgo de prompt injection (R8 de Fase 1), viola I8.
2. Knowledge como rol `knowledge` dedicado con procesamiento especial — Aceptado: separación clara de fronteras de confianza, system prompt tiene precedencia.

**Consecuencias:**
- Requiere modificar openai.mapper.ts y BuildPromptStep.
- Policy Engine como capa de validación antes/después de la generación.

---

### ADR-004: Resolución de instancia por ContactAssignment y default del Workspace

**Estado:** Reemplazado por DEC-CM-04, DEC-CM-06 y DEC-CM-08 (06/11/2026).

**Decisión vigente:** no existe routing persistente canal → instancia ni entidad AgentRouting. CRM es autoridad única de ContactAssignment. Ante un nuevo mensaje de un contacto sin asignación humana ni asignación a una AgentInstance, CRM asigna la AgentInstance predeterminada solo si está activa; si no existe o no está activa, el contacto queda sin asignación. Como máximo una AgentInstance puede ser default por Workspace, con cambio atómico. No hay fallback a otra instancia por prioridad.

**Consecuencias:** retirar `channelAssignment` y `priority` como mecanismos de resolución. El canal sigue disponible como contexto del mensaje y de Execution. Los workers validan la asignación vigente antes de cualquier efecto externo.

---

### ADR-005: Instantánea inmutable de configuración efectiva por ejecución (P10)

**Estado:** Principio y referencias identificables/versionadas obligatorias por ejecución aprobados; política de publicación, ciclo de versionado/retención de Knowledge y detalles de persistencia pendientes.
**Fecha de decisión:** 03/11/2026
**Decisión:** Al comenzar cada ejecución, se crea una instantánea inmutable de la configuración efectiva (AgentDefinition versión activa, Rules, Skills, capabilities, knowledgeBaseIds y parámetros de ejecución de la AgentInstance). La ejecución en curso conserva sus definiciones hasta finalizar; los cambios posteriores se aplican únicamente a nuevas ejecuciones.

Se definen **tres niveles de garantía** para trazabilidad y reproducibilidad:

**Nivel 1 — Reconstrucción de configuración:**
Permite conocer la configuración efectiva con la que se inició una ejecución. Incluye referencias identificables y versionadas de AgentDefinition, Rules, Skills y SkillConfiguration, capabilities, routing, provider/model y parámetros efectivos. Se conserva obligatoriamente en `effectiveConfigSnapshot`; `effectiveConfigHash` permite verificar su integridad.

**Nivel 2 — Trazabilidad del contexto:**
Permite conocer qué información se utilizó durante la ejecución y es obligatorio por ejecución:
- Knowledge Bases habilitadas (`knowledgeBaseIds` en el snapshot).
- Fuentes recuperadas (IDs estables de documentos/chunks de Qdrant) y su versión publicada cuando exista.
- IDs de fragmentos efectivamente incorporados al contexto, con versión/revisión disponible y hash de contenido.
- Referencias para recuperar el contenido original, cuando esté disponible y no haya sido eliminado por políticas de retención.
- Si una fuente o fragmento no dispone de versionado propio, conservar su ID estable y el hash del contenido utilizado.

**Nivel 3 — Reproducción de ejecución:**
Permite intentar volver a ejecutar el proceso reconstruyendo el mismo contexto y condiciones. **Nota:** No se garantiza reproducibilidad exacta de la respuesta LLM, ya que los modelos de lenguaje son no deterministas, los proveedores pueden cambiar sus modelos, y las herramientas externas pueden devolver resultados diferentes. El Nivel 3 busca reproducir las *condiciones de entrada*, no la *salida exacta*.

**Alternativas consideradas:**
1. Relectura de configuración en cada paso del pipeline — Rechazado: riesgo de inconsistencia si la configuración cambia a mitad de ejecución.
2. Invalidación de caché sin instantánea — Rechazado: una ejecución en curso podría usar configuración nueva a mitad de su procesamiento.
3. Instantánea inmutable al inicio de cada ejecución — Aceptado: consistencia garantizada durante el ciclo de vida de la ejecución, cambios propagados inmediatamente a nuevas ejecuciones, sin flujo draft/release para el cliente (I17).

**Consecuencias:**
- Se definen dos hashes diferenciados:
  - `promptSnapshotHash` (SHA-256 del prompt efectivo construido) — identifica el prompt enviado al modelo.
  - `effectiveConfigHash` (SHA-256 de la configuración efectiva completa: AgentDefinition.id+version, skill IDs+versiones, rule IDs+versiones, capability IDs, provider, modelo, parámetros) — identifica la configuración efectiva.
- Se almacena obligatoriamente `effectiveConfigSnapshot` (JSON sin secretos) con las referencias identificables/versionadas de configuración y Knowledge requeridas en los Niveles 1 y 2. La copia adicional de valores efectivos para reproducibilidad técnica (Nivel 3, P7) puede optimizarse por almacenamiento, sin eliminar las referencias obligatorias.
- Las ejecuciones en curso no se ven afectadas por cambios de AgentDefinition, Rules, Skills o Knowledge mientras están en proceso.
- Requiere almacenar por ejecución la versión de AgentDefinition, revisiones de Skills/SkillConfiguration y Rules, y las referencias versionadas de documentos y fragmentos Knowledge efectivamente utilizados.
- Sin flujo draft/release para el cliente (I17).
- **Trazabilidad RAG (Nivel 2):** `knowledgeBaseIds` identifica las bases habilitadas, pero no el contenido efectivamente recuperado. Se registran obligatoriamente IDs de documentos y fragmentos de Qdrant, versiones disponibles y hashes de los fragmentos incorporados al contexto.
- **No se incluyen secretos ni valores sensibles en el snapshot.** Las credenciales se referencian por ID de versión.
- **Limitación importante:** Un hash o snapshot de configuración **no permite reconstruir exactamente una respuesta LLM**. La no determinismo de los modelos, cambios en proveedores, y datos operativos que cambian impiden la reproducción exacta de la salida. El snapshot garantiza trazabilidad de la configuración de entrada, no reproducibilidad de la salida.

---

## K. Roadmap de implementación con dependencias

La secuencia de implementación se mantiene como planificación posterior y debe seguir las dependencias reales registradas en Linear. Las decisiones P1, P2 y P3 ya no son bloqueantes de diseño: sus acuerdos vigentes están documentados en ARQ-19/20 y DEC-CM-01..08. P4 está aprobada. Los detalles técnicos que siguen pendientes no deben convertirse en nuevas decisiones de producto.

| Bloque | Alcance | Estado de planificación |
|---|---|---|
| WAB-79 — Fase 3 | Dominio, ownership, persistencia e índices | Backlog; bloqueada por WAB-78 |
| WAB-80 — Fase 4 | Provisioning y autorización | Backlog |
| WAB-81 — Fase 5 | Runtime y composición de contexto | Backlog |
| WAB-82 — Fase 6 | Knowledge compartido y Skills/Rules por agente | Backlog |

La secuencia y las dependencias operativas se consultan en Linear. No iniciar WAB-79 ni fases posteriores hasta que WAB-78 sea revisada y autorizada expresamente.

**Decisiones vigentes:** ARQ-19/20/21, DEC-CM-01..08 y P4.
**Pendientes de contrato para completar WAB-78:**
- **ARQ-14 — ownership de persistencia: resuelto en D.1.9 y B.** Se distinguen propietario de negocio, superficie API y servicio persistente para cada entidad y sus proyecciones.
- **ARQ-22 — entrega asíncrona e idempotencia: resuelto como contrato en D.6.** Se fijan entrega at-least-once, identidades estables, aceptación durable antes del ack, Outbox de resultado con confirmación, deduplicación, recuperación y DLQ. La implementación queda en WAB-81.
- **Verificación cruzada final:** comprobar que el inventario A.2, diagrama B, matriz D.1.9, D.6, H.1, ADRs y escenarios de integración no se contradigan; no repetir diagnósticos D1–D5 ni reabrir decisiones.

**Trabajo técnico diferido (no son decisiones de producto pendientes):** la matriz exhaustiva operación × emisor × receptor × audience × scope × claims, rotación/revocación y anti-replay de JWT, autenticación RabbitMQ, pruebas negativas y despliegue de ARQ-21; verificación de transacciones MongoDB Atlas y su implementación para ARQ-06; implementación de invalidación/cancelación y validaciones de vigencia para ARQ-07/08. Se ejecuta en las fases técnicas correspondientes, manteniendo los contratos ya aprobados.

**Decisiones no reabribles:** ARQ-19/20/21 como dirección y contrato base; DEC-CM-01..08; P1/P2/P4/P9/P10 y reconciliación D5-A/B/C. La falta de implementación no revoca su aprobación. Los detalles de reversibilidad/purga de P9 y versionado/retención de Knowledge de P10 solo permanecen abiertos en la medida exacta en que no estén cubiertos por las decisiones registradas; no usar esa frase genérica para reabrir P9/P10.

## L. Criterios de finalización de Fase 2

- [x] Arquitectura objetivo documentada
- [x] Modelo de dominio y relaciones (Sección A, B)
- [x] Contratos entre servicios (Sección D)
- [x] Diseño de autorización y aislamiento (Sección D)
- [x] Diseño de provisioning (Sección C)
- [x] Diseño de runtime y contexto (Sección E, F)
- [x] Diseño de créditos y telemetría (Sección G)
- [x] Plan de migración (Sección H)
- [x] ADRs correspondientes (Sección J)
- [x] Roadmap de implementación con dependencias (Sección I, K)

**Ruta canónica prevista:** `projects/agents-multitenant/design/fase-2-diseno-objetivo-y-contratos.md` en `sinies7ro/waby-docs`.

**Estado de cierre documental (07/11/2026):** ARQ-14 y ARQ-22 tienen contrato de diseño documentado en D.1.9/B y D.6, respectivamente. La verificación cruzada entre inventario, diagramas, contratos, ADRs y los 15 escenarios D3 quedó completada documentalmente el 07/11/2026. Las pruebas de integración no se consideran ejecutadas. El diseño queda listo para revisión y autorización expresa del usuario; esta verificación no cierra WAB-78 por sí sola.

**Siguiente paso:** presentar el diseño completo al usuario para revisión y autorización expresa. WAB-78 permanece In Progress y WAB-79 no se inicia hasta esa autorización.

---
*Documento de diseño exclusivamente. No se modifica código, schemas, colecciones, resolvers, endpoints ni infraestructura durante esta fase.*

---

# Anexo técnico integrado — P4: presupuesto por ejecución e invocación

Este anexo forma parte del diseño integral de P4 y desarrolla contratos, fórmulas, transiciones, fencing, leases, idempotencia, recuperación y reconciliación. No constituye una segunda fuente normativa: prevalecen ADR-002, ADR-006 y G.4.6. La política de facturación híbrida y el período de gracia de 30 días están aprobados; cualquier mención histórica a una política pendiente queda superada.

### Desarrollo técnico de P4 — presupuesto por ejecución e invocación

**Fecha:** 2026-11-03
**Estado:** Integrado en el diseño canónico de Fase 2 en projects/agents-multitenant/design/fase-2-diseno-objetivo-y-contratos.md.
**Tarea:** WAB-78 — Fase 2
**Alcance:** P4 — presupuesto de créditos, concurrencia, fencing, asignaciones, liquidación y trazabilidad.

> **Alcance:** desarrollo técnico integrado en el documento canónico de Fase 2. Esta sección no debe mantenerse ni citarse como archivo independiente.

---

## 1. Objetivo y decisiones ya aprobadas

P4 define cómo Waby autoriza, controla, mide y liquida el consumo de créditos de ejecuciones de agentes en una plataforma multitenant.

Las siguientes decisiones conceptuales y económicas ya están aprobadas y no deben volver a presentarse como pendientes de aprobación:

1. **La Organización es propietaria del saldo.** Existe una única bolsa de créditos por `organizationId`, compartida por todos los Workspaces de esa Organización.
2. **El Workspace es atribución, no propietario del saldo.** No existe una bolsa independiente por Workspace en esta etapa.
3. **No hay límites individuales por Workspace en esta etapa.** Un Workspace podría consumir la bolsa completa de la Organización, sujeto a los límites económicos administrados por Waby.
4. **La concurrencia entre Workspaces está permitida.** Dos Workspaces de una misma Organización pueden ejecutar agentes simultáneamente.
5. **La protección contra sobreconsumo es organizacional.** La admisión de ejecuciones concurrentes debe proteger el saldo compartido de la Organización.
6. **Cada ejecución tiene un presupuesto máximo autorizado.** Cada invocación facturable dentro de esa ejecución también debe obtener un presupuesto máximo propio.
7. **El fencing es individual por autorización y por asignación.** No existe un fencing token global por Organización ni un token global por Worker.
8. **La liquidación contable ocurre después de conocer el consumo reconocido de la ejecución.** El ledger es append-only.
9. **No se crean asientos financieros `hold`/`release`.** Sí existe un compromiso operativo temporal de capacidad, representado por `OrganizationAICredit.committedBudget` y las autorizaciones activas. Ese compromiso no es un movimiento contable.
10. **El Workspace y la AgentInstance se conservan como dimensiones de atribución.** Las métricas pueden desglosarse por Organización, Workspace, AgentInstance y ejecución, sin crear saldos separados.
11. **Alertas, límites organizacionales y On-Demand son administrados por Waby.** No se fijan aquí porcentajes comerciales concretos como 90 %, 100 % o 200 %.

P4 y su política híbrida de facturación G.4.6 están aprobadas. Este desarrollo técnico no acredita implementación ni cierra WAB-78: las garantías deberán verificarse con pruebas de integración y concurrencia en las fases correspondientes.

## 2. Principios de diseño

- `OrganizationAICredit.balance` es la fuente de verdad del saldo de créditos de la Organización.
- `OrganizationAICredit.committedBudget` representa capacidad ya comprometida por autorizaciones de ejecución que todavía retienen presupuesto.
- `ExecutionBudgetAuthorization.authorizedBudget` limita el presupuesto total de una ejecución.
- `ExecutionBudgetAllocation.assignedBudget` limita el presupuesto de una invocación concreta.
- Los presupuestos autorizados y asignados son importes históricos. No se reducen para reflejar liquidaciones parciales.
- El consumo reconocido se acumula en `consumedAmount`; la capacidad no consumida se libera mediante los contadores de presupuesto comprometido.
- Las operaciones que compiten por saldo o presupuesto se protegen con escrituras condicionales y transacciones MongoDB. Una consulta previa seguida de una escritura no constituye una garantía de concurrencia.
- Redis puede ayudar con coordinación, notificaciones o leases auxiliares, pero no sustituye la validación transaccional en MongoDB.
- El fencing evita que un propietario obsoleto escriba en el estado propio de Waby. No cancela una operación externa que un proveedor ya haya aceptado.
- El consumo técnico reconocido por el proveedor y el importe que Waby decide cobrar al cliente son conceptos separados; su tratamiento comercial/contable sigue la política híbrida aprobada en G.4.6.

## 3. Modelo de dominio

Todos los importes de presupuesto y consumo usan la misma unidad de créditos y una representación decimal exacta. No deben usarse `float`/`number` binarios como representación persistente de cantidades contables.

### 3.1 `OrganizationAICredit`

Entidad 1:1 con la Organización.

| Campo | Propósito |
|---|---|
| `organizationId` | Organización propietaria de la cuenta. Identidad única. |
| `balance` | Saldo actual de créditos de la Organización. |
| `currency` | Unidad monetaria o unidad de crédito definida por el modelo de negocio. |
| `totalPurchased` | Acumulado informativo de créditos comprados, si se conserva como contador derivado. |
| `totalConsumed` | Acumulado informativo de consumo reconocido/liquidado, según la semántica contable que defina el ledger. |
| `committedBudget` | Suma del `authorizedBudget` de las autorizaciones que actualmente retienen capacidad. |
| `createdAt`, `updatedAt` | Auditoría temporal. |

**No incluir `lastFencingToken` en esta entidad.** El fencing no es global por Organización. La generación vigente vive en cada `ExecutionBudgetAuthorization` y cada `ExecutionBudgetAllocation`.

Invariantes:

- `balance >= 0`.
- `committedBudget >= 0`.
- `balance - committedBudget >= 0`.
- `committedBudget` coincide con la suma de `authorizedBudget` de las autorizaciones que retienen capacidad, según los estados de §6.
- Las actualizaciones de `balance` y `committedBudget` deben ser transaccionales con la operación que las origina.

### 3.2 `CreditLedger`

Ledger financiero append-only propiedad de la Organización.

Cada asiento debe poder atribuirse, cuando corresponda, a `organizationId`, `workspaceId`, `agentInstanceId` y `executionId`.

Campos conceptuales:

| Campo | Propósito |
|---|---|
| `movementId` | Identificador único e inmutable del asiento. |
| `organizationId` | Ámbito contable y propietario del saldo. |
| `workspaceId` | Dimensión de atribución; nunca propietario de la bolsa. |
| `agentInstanceId` | Instancia que originó el consumo, si aplica. |
| `executionId` | Ejecución asociada a un débito, si aplica. |
| `type` | Tipo de movimiento: `purchase`, `deduct`, `refund` y `adjustment`. |
| `amount` | Importe del asiento. |
| `refundRequestId` | Identidad idempotente propia de una solicitud de devolución. |
| `originalMovementId` | Referencia directa al `movementId` del débito original compensado por un refund. |
| `originalExecutionId` | Referencia complementaria al `executionId` asociado al débito original. |
| `idempotencyKey` | Identidad estable de la operación, distinta del fencing token. |
| `pricingVersion` | Versión de tarifas aplicada, cuando corresponda. |
| `metadata` | Metadatos auditables no secretos. |
| `createdAt` | Fecha de creación. |

Reglas:

- Un asiento ya confirmado nunca se modifica ni elimina.
- Una devolución crea un asiento `refund` compensatorio; no transforma el débito original en `reversed`.
- La idempotencia de un débito de ejecución se garantiza por Organización + ejecución lógica + tipo de movimiento `deduct`.
- La idempotencia de una devolución se garantiza por Organización + `refundRequestId`.
- Las compras y los ajustes administrativos tienen identificadores idempotentes propios, por ejemplo el identificador de pago del proveedor o un `adjustmentRequestId`.
- `fencingGeneration` no forma parte de ninguna clave idempotente.
- La suma de devoluciones asociadas a un débito no puede superar el débito neto original. Cada devolución se registra como un asiento separado.
- No se crean movimientos `hold` ni `release` en el ledger.

### 3.3 `ExecutionBudgetAuthorization`

Una autorización representa el presupuesto máximo comprometido para una ejecución lógica.

| Campo | Tipo conceptual | Propósito |
|---|---|---|
| `authorizationId` | UUID | Identidad de la autorización. |
| `organizationId` | ID | Organización propietaria del saldo. |
| `workspaceId` | ID | Workspace de atribución. |
| `agentInstanceId` | ID | Instancia que ejecuta. |
| `executionId` | ID | Ejecución lógica. |
| `attemptId` | ID | Intento operativo actual; no es clave idempotente financiera. |
| `authorizedBudget` | Decimal | Presupuesto máximo autorizado para la ejecución. |
| `consumedAmount` | Decimal | Consumo real reconocido acumulado de las invocaciones resueltas. |
| `activeAllocatedBudget` | Decimal | Contador persistido de `assignedBudget` de las asignaciones que retienen capacidad. |
| `fencingGeneration` | Integer | Generación vigente de propiedad de esta autorización. |
| `ownerId` / `workerId` | ID | Propietario operativo actual, según el contrato de ejecución. |
| `status` | Enum | Estado de la autorización. |
| `leaseExpiresAt` | Timestamp | Vencimiento del lease actual. |
| `pricingVersion` | String | Versión de tarifas fijada para la ejecución, si aplica. |
| `createdAt`, `updatedAt` | Timestamp | Auditoría temporal. |

Restricciones:

- `authorizedBudget > 0` mientras la autorización retenga capacidad.
- `consumedAmount >= 0`.
- `activeAllocatedBudget >= 0`.
- `consumedAmount + activeAllocatedBudget <= authorizedBudget`.
- `activeAllocatedBudget` es igual a la suma de `assignedBudget` de las asignaciones que retienen capacidad.
- Una ejecución lógica no puede tener dos autorizaciones activas que comprometan presupuesto simultáneamente.
- La recuperación de una autorización no crea una segunda autorización activa ni duplica `committedBudget`.
- La liquidación de una invocación no modifica `authorizedBudget`.
- Una ampliación explícita de presupuesto, si se admite, es una operación separada, idempotente y transaccional (§5.5); nunca debe confundirse con la liquidación de una invocación.

### 3.4 `ExecutionBudgetAllocation`

Una asignación representa el presupuesto máximo para una invocación concreta: llamada LLM, herramienta facturable u otra operación que pueda consumir créditos. No presupone fallback entre proveedores LLM.

| Campo | Tipo conceptual | Propósito |
|---|---|---|
| `allocationId` | UUID | Identificador de la asignación. |
| `authorizationId` | UUID | Autorización padre. |
| `executionId` | ID | Ejecución lógica. |
| `invocationId` | ID | Identidad estable de la invocación lógica. |
| `attemptId` | ID | Intento operativo actual. |
| `assignedBudget` | Decimal | Presupuesto máximo asignado a la invocación. |
| `consumedAmount` | Decimal | Consumo real reconocido de la invocación. |
| `fencingGeneration` | Integer | Generación vigente de propiedad de esta asignación. |
| `ownerId` / `workerId` | ID | Propietario operativo actual. |
| `status` | Enum | Estado de la asignación. |
| `leaseExpiresAt` | Timestamp | Vencimiento del lease actual. |
| `createdAt`, `updatedAt` | Timestamp | Auditoría temporal. |

Restricciones:

- Una asignación pertenece a exactamente una autorización.
- `assignedBudget > 0` al crear la asignación.
- `consumedAmount >= 0`.
- `assignedBudget` es histórico e inmutable después de asignar la invocación. No se decrementa durante la liquidación.
- Una invocación lógica no puede tener dos asignaciones activas que reserven capacidad al mismo tiempo.
- Un retry o una recuperación no puede crear una asignación paralela mientras el resultado de la invocación previa sea desconocido.
- La idempotencia de la asignación se basa en la identidad estable de autorización + invocación; `attemptId` y `fencingGeneration` identifican propiedad/ejecución, no sustituyen esa identidad.

### 3.5 `AgentExecution`

La ejecución debe conservar al menos:

- `organizationId`: propietario del saldo.
- `workspaceId`: atribución.
- `agentInstanceId`: instancia responsable.
- `executionId`: identidad lógica estable.
- Referencia a la autorización de presupuesto.
- `promptSnapshotHash` y `effectiveConfigHash`, según el contrato de snapshots de P10.
- Metadatos de invocaciones, consumo reconocido, tokens, tarifa aplicada y resultado operativo.

No registrar API keys, credenciales ni secretos del proveedor en snapshots, logs o metadatos de ledger.

## 4. Fórmulas e invariantes de presupuesto

### 4.1 Admisión organizacional

Antes de autorizar una ejecución, el presupuesto organizacional no comprometido es:

```text
availableOrganizationBudget = OrganizationAICredit.balance
                            - OrganizationAICredit.committedBudget
```

Una autorización de presupuesto `requestedBudget` sólo puede admitirse cuando:

```text
availableOrganizationBudget >= requestedBudget
```

La verificación y el incremento de `committedBudget` deben ser atómicos.

### 4.2 Presupuesto disponible dentro de una ejecución

El presupuesto disponible para una nueva invocación es:

```text
availableInvocationBudget = authorizedBudget
                          - consumedAmount
                          - activeAllocatedBudget
```

Una nueva asignación sólo puede admitirse cuando:

```text
requestedInvocationBudget <= availableInvocationBudget
```

El runtime utiliza el contador persistido `activeAllocatedBudget`. No hace una consulta `SUM(assignedBudget)` como mecanismo de exclusión concurrente.

### 4.3 Invariantes obligatorias

```text
OrganizationAICredit.balance >= 0
OrganizationAICredit.committedBudget >= 0
OrganizationAICredit.balance - OrganizationAICredit.committedBudget >= 0

ExecutionBudgetAuthorization.consumedAmount >= 0
ExecutionBudgetAuthorization.activeAllocatedBudget >= 0
ExecutionBudgetAuthorization.consumedAmount
  + ExecutionBudgetAuthorization.activeAllocatedBudget
  <= ExecutionBudgetAuthorization.authorizedBudget

ExecutionBudgetAuthorization.activeAllocatedBudget
  = SUM(ExecutionBudgetAllocation.assignedBudget
        para asignaciones que retienen capacidad)

OrganizationAICredit.committedBudget
  = SUM(ExecutionBudgetAuthorization.authorizedBudget
        para autorizaciones que retienen capacidad)
```

Las dos igualdades de sumatoria son invariantes de reconciliación. No sustituyen los contadores persistidos ni se usan como única protección en la ruta caliente de asignación.

## 5. Contratos transaccionales

Todas las operaciones siguientes se ejecutan con sesiones/transacciones MongoDB. Las escrituras deben ser condicionales y comprobar el estado y la generación actuales. Los errores de conflicto deben producir retry controlado e idempotente, no una segunda operación lógica.

### 5.1 T1 — Autorizar presupuesto de ejecución

1. Recibir `organizationId`, `workspaceId`, `agentInstanceId`, `executionId`, `requestedBudget` e identidad idempotente de la solicitud.
2. Iniciar transacción.
3. Leer/actualizar condicionalmente `OrganizationAICredit`, exigiendo `balance - committedBudget >= requestedBudget`.
4. Incrementar `committedBudget` por `requestedBudget`.
5. Crear `ExecutionBudgetAuthorization` con `authorizedBudget = requestedBudget`, `consumedAmount = 0`, `activeAllocatedBudget = 0`, generación inicial y estado `authorized`.
6. Confirmar la actualización de la cuenta y la creación de autorización en la misma transacción.
7. Si la solicitud idempotente ya existe, devolver la autorización existente sin volver a comprometer presupuesto.

La autorización no crea un asiento `hold` en `CreditLedger`.

### 5.2 T2 — Asignar presupuesto a una invocación

Este es el mecanismo obligatorio para evitar sobreasignación entre invocaciones concurrentes de una misma ejecución.

1. Recibir `authorizationId`, `invocationId`, `requestedInvocationBudget`, `ownerId/workerId`, `attemptId` y generación vigente.
2. Iniciar transacción.
3. Actualizar condicionalmente la autorización padre verificando:
   - estado apto para ejecutar;
   - lease vigente;
   - propietario e `attemptId` válidos;
   - `fencingGeneration` igual al valor esperado;
   - `authorizedBudget - consumedAmount - activeAllocatedBudget >= requestedInvocationBudget`.
4. Incrementar `activeAllocatedBudget` por `requestedInvocationBudget` mediante `$inc`/operación equivalente protegida por el filtro condicional.
5. Crear `ExecutionBudgetAllocation` con `assignedBudget = requestedInvocationBudget`, `consumedAmount = 0`, generación inicial y estado `assigned`.
6. Confirmar la actualización del padre y la creación de la asignación en la misma transacción.
7. Si la creación falla, la transacción también revierte el incremento del contador padre.
8. Ante una repetición de la misma solicitud idempotente, devolver la asignación existente; no incrementar nuevamente el contador.

**Prohibido:** calcular un `SUM` de asignaciones activas, comprobar disponibilidad fuera de una transacción y posteriormente insertar una asignación. Dos workers podrían observar el mismo espacio disponible y comprometerlo dos veces.

### 5.3 T3 — Liquidar una invocación

Al resolver una invocación cuyo resultado y consumo son conocidos:

1. Iniciar transacción.
2. Validar autorización y asignación: estado, propietario, `attemptId`, lease y generaciones vigentes.
3. Obtener el consumo real reconocido de esa invocación en créditos.
4. Comprobar que no se liquide dos veces la misma invocación. La identidad de liquidación debe ser estable y no depender del fencing token.
5. Incrementar `ExecutionBudgetAuthorization.consumedAmount` por el consumo reconocido.
6. Decrementar `ExecutionBudgetAuthorization.activeAllocatedBudget` por el `assignedBudget` completo de la asignación resuelta.
7. Registrar el consumo en `ExecutionBudgetAllocation.consumedAmount`.
8. Cambiar la asignación a `settled`.
9. Confirmar todas las operaciones atómicamente.

**No modificar `ExecutionBudgetAuthorization.authorizedBudget`.**
**No modificar `ExecutionBudgetAllocation.assignedBudget`.**

Ejemplo:

| Concepto | Antes | Después |
|---|---:|---:|
| `authorizedBudget` | 100 | 100 |
| `consumedAmount` autorización | 20 | 27 |
| `activeAllocatedBudget` | 30 | 10 |
| `assignedBudget` de la invocación que termina | 20 | 20 |
| `consumedAmount` de esa asignación | 0 | 7 |

La asignación liberada tenía 20 créditos asignados y consumió 7. Se libera el presupuesto asignado completo de 20 en el contador activo y se agregan 7 al consumo reconocido. La capacidad neta disponible aumenta en 13.

### 5.4 T4 — Renovar lease

- Una renovación normal conserva el `fencingGeneration` vigente.
- La renovación valida que el propietario y la generación sigan siendo los actuales.
- Actualiza sólo la expiración del lease y los metadatos temporales requeridos.
- No crea una nueva generación ni altera importes de presupuesto.

### 5.5 T5 — Ampliar una autorización

Si el producto permite ampliar el presupuesto máximo de una ejecución:

1. La ampliación se trata como una operación nueva, con `extensionRequestId` idempotente.
2. Se valida propietario, generación, estado y lease de la autorización.
3. Se verifica a nivel Organización que `balance - committedBudget >= extensionAmount`.
4. En una única transacción se incrementa `OrganizationAICredit.committedBudget` y `ExecutionBudgetAuthorization.authorizedBudget` por el mismo delta.
5. Una repetición de `extensionRequestId` no vuelve a ampliar el presupuesto.

Esta operación es la única que puede aumentar explícitamente `authorizedBudget`. La liquidación de invocaciones nunca lo modifica. No se permite reducirlo como efecto de liquidar consumo.

### 5.6 T6 — Liquidación final de ejecución

La ejecución sólo puede liquidarse cuando:

- no quedan asignaciones en estados que retienen presupuesto;
- no quedan resultados externos desconocidos;
- `activeAllocatedBudget = 0`;
- el consumo reconocido de las invocaciones fue consolidado idempotentemente;
- se dispone de la política de facturación aplicable al resultado de la ejecución.

Dentro de una única transacción MongoDB:

1. Validar autorización, propietario, generación y estado.
2. Verificar que no existan asignaciones pendientes ni `reconciliation_required`.
3. Obtener el consumo reconocido consolidado de la autorización.
4. Calcular el importe facturable según `pricingVersion` y la política comercial/contable aprobada.
5. Insertar un asiento `deduct` idempotente para Organización + ejecución lógica.
6. Actualizar `OrganizationAICredit.balance` por el importe efectivamente facturable.
7. Decrementar `OrganizationAICredit.committedBudget` por el `authorizedBudget` completo de esa autorización, no sólo por el excedente.
8. Cambiar la autorización a `settled`.
9. Confirmar ledger, saldo, liberación de compromiso y estado en la misma transacción.

El importe debitado no debe superar silenciosamente el presupuesto máximo autorizado. Si el consumo reconocido o el cálculo de créditos exceden el límite autorizado, la ejecución entra en una ruta de excepción/reconciliación; no se debe aplicar un débito adicional no autorizado sin una regla explícita.

Ejemplo de una ejecución con presupuesto autorizado de 100 y débito final de 50:

| Concepto | Antes | Después |
|---|---:|---:|
| Saldo de Organización | 1.000 | 950 |
| `authorizedBudget` | 100 | 100 |
| `consumedAmount` | 50 | 50 |
| `activeAllocatedBudget` | 0 | 0 |
| `committedBudget` de Organización | 100 | 0 |
| Asiento `deduct` | — | 50 |

Se libera el compromiso de 100 completo porque terminó la ejecución. Se debitan 50 porque ese es el importe facturable del ejemplo. No se debitan 100 por el mero hecho de haberlos autorizado.

### 5.7 T7 — Compras, refunds y ajustes

Las operaciones que no pertenecen al ciclo de vida de una ejecución no deben exigir un `fencingGeneration` de ejecución/asignación.

- **Compra:** idempotencia por identificador estable de pago/compra; actualización de saldo y asiento `purchase` en una transacción.
- **Refund:** idempotencia por `refundRequestId`; asiento compensatorio append-only; validación de que el total devuelto no supere el débito original.
- **Ajuste administrativo:** idempotencia por `adjustmentRequestId`, identidad del actor y motivo auditables, transacción de saldo y asiento append-only.

Estas operaciones tienen sus propios controles de concurrencia de saldo y no dependen del fencing de una ejecución particular.

## 6. Estados y retención de presupuesto

### 6.1 Estados de `ExecutionBudgetAuthorization`

| Estado | Significado | ¿Retiene `committedBudget`? |
|---|---|---:|
| `requested` | Solicitud creada, todavía no comprometida | No |
| `authorized` | Presupuesto comprometido y disponible para la ejecución | Sí |
| `executing` | Ejecución en curso | Sí |
| `settling` | Liquidación final en curso | Sí |
| `reconciliation_required` | Resultado/consumo externo incierto; requiere reconciliación | Sí |
| `settled` | Ejecución liquidada y compromiso liberado | No |
| `cancelled` | Autorización cancelada tras demostrar que no hubo consumo potencial | No |
| `expired` | Autorización vencida sin despacho ni consumo potencial | No |

Una autorización cuyo resultado externo es desconocido no puede pasar a `expired` para liberar presupuesto. Debe permanecer o pasar a `reconciliation_required`.

### 6.2 Estados de `ExecutionBudgetAllocation`

| Estado | Significado | ¿Retiene `activeAllocatedBudget`? |
|---|---|---:|
| `assigned` | Presupuesto asignado, invocación aún no resuelta | Sí |
| `executing` | Invocación en ejecución | Sí |
| `settling` | Resultado conocido, liquidación en curso | Sí |
| `reconciliation_required` | Resultado externo/consumo desconocido | Sí |
| `settled` | Consumo reconocido y presupuesto asignado liberado | No |
| `cancelled` | Cancelación segura, antes de cualquier consumo potencial | No |
| `expired` | Venció sin despacho ni consumo potencial | No |

Los estados que retienen presupuesto son exactamente:

```text
assigned, executing, settling, reconciliation_required
```

Las definiciones, fórmulas, restricciones de unicidad, cardinalidades e índices deben usar la misma lista. No puede haber una sección que omita `settling` o `reconciliation_required`.

`cancelled` y `expired` sólo liberan presupuesto cuando se demostró que no existe una operación externa aceptada ni consumo potencial pendiente. El vencimiento del lease por sí solo no es esa demostración.

### 6.3 Transiciones de recuperación

- Una asignación `assigned` que nunca fue despachada puede cancelarse/expirar de forma segura.
- Una asignación `executing` con resultado conocido pasa a `settling` y luego `settled`.
- Una asignación `executing` cuyo resultado externo es desconocido pasa a `reconciliation_required`.
- Una asignación `reconciliation_required` conserva íntegramente su `assignedBudget` hasta que la reconciliación determine consumo y resultado.
- La autorización padre conserva su `authorizedBudget` comprometido mientras alguna asignación siga reteniendo capacidad o el consumo de la ejecución sea incierto.

## 7. Fencing, leases y concurrencia

### 7.1 Ámbito

Cada `ExecutionBudgetAuthorization` tiene su propia generación de fencing. Cada `ExecutionBudgetAllocation` tiene su propia generación de fencing. No hay `lastFencingToken` compartido por Organización ni token global de Worker.

### 7.2 Reglas de generación

- La creación establece una generación inicial.
- La renovación normal del lease conserva la generación.
- La recuperación o reasignación efectiva de propietario incrementa la generación del recurso cuya propiedad cambia.
- La recuperación no libera presupuesto ni crea una autorización/asignación paralela.
- Un worker con generación obsoleta no puede liquidar, ampliar, cancelar ni modificar estado/importes.

### 7.3 Validación

Las escrituras protegidas deben comprobar en MongoDB, dentro de una transacción o mediante una escritura condicional atómica:

- identificador del recurso;
- `fencingGeneration` esperado;
- `ownerId`/`workerId` esperado;
- `attemptId` esperado;
- estado compatible con la operación;
- lease vigente, cuando corresponda.

Para una operación sobre una asignación se valida también la autorización padre y su generación vigente. Para una operación sobre el presupuesto organizacional se protege además el contador organizacional mediante una actualización condicional.

Redis puede facilitar la elección de propietario o la expiración de leases, pero una lectura de Redis seguida de una escritura MongoDB no es una garantía de fencing. La autoridad final de aceptación/rechazo debe residir en las escrituras protegidas de MongoDB.

### 7.4 Límite de la garantía

El fencing impide que un worker obsoleto modifique el estado de Waby. No puede deshacer ni cancelar una llamada LLM/tool que un proveedor externo ya aceptó. Por eso, ante un resultado externo desconocido, se conserva el presupuesto y se inicia reconciliación; no se libera por el simple vencimiento del lease.

## 8. Idempotencia: identidades separadas

| Operación | Identidad idempotente | No usar como identidad |
|---|---|---|
| Autorizar ejecución | Organización + `executionId` + solicitud de autorización estable | `fencingGeneration` |
| Ampliar autorización | `extensionRequestId` | `attemptId` o generación |
| Asignar invocación | `authorizationId` + `invocationId` | `fencingGeneration` |
| Liquidar invocación | `allocationId` + identidad estable del resultado/liquidación | Generación renovada |
| Debitar ejecución | Organización + `executionId` + tipo `deduct` | `workerId`, lease o generación |
| Devolver créditos | Organización + `refundRequestId` | `executionId` por sí solo |
| Comprar créditos | ID estable de compra/pago | Fencing de ejecución |
| Ajuste administrativo | `adjustmentRequestId` | Fencing de ejecución |

Una recuperación cambia el propietario/generación, no la identidad económica de la ejecución ni de la invocación. Reintentar un mensaje no crea otro débito o asignación si la operación lógica ya existe.

## 9. Límites de responsabilidad CRM / AI Service

**CRM (autoridad del dominio financiero):**

- Es propietario de `OrganizationAICredit`, `CreditLedger`, `ExecutionBudgetAuthorization` y `ExecutionBudgetAllocation`, o del servicio transaccional que los persiste.
- Valida ownership de Organización/Workspace/AgentInstance.
- Autoriza y compromete presupuesto mediante transacciones MongoDB.
- Crea/liquida asignaciones y aplica fencing en escrituras protegidas.
- Liquida el débito final, actualiza saldo y libera el compromiso de Organización atómicamente.
- Expone contratos internos idempotentes para autorización, ampliación, asignación, liquidación y reconciliación.

**AI Service (orquestación de ejecución):**

- Solicita autorización y presupuesto de invocación al CRM.
- Ejecuta LLM/tools dentro del presupuesto recibido.
- Reporta consumo real reconocido y resultado por invocación.
- Conserva `executionId`, `invocationId`, `attemptId` y correlación de autorización/asignación.
- No modifica directamente `OrganizationAICredit.balance`, `committedBudget` ni `CreditLedger`.
- No decide por sí solo la política comercial de cobro cuando la respuesta no se entrega o el resultado externo es desconocido.

El contrato entre servicios debe ser idempotente y permitir consultas/reintentos sin duplicar compromisos ni débitos. Evitar que AI Service tenga una segunda lógica de saldo que compita con la fuente de verdad del CRM.

## 10. Flujo runtime de referencia

1. El CRM valida Organización, Workspace, AgentInstance y estado de ejecución.
2. El runtime solicita autorización del presupuesto máximo para `executionId`.
3. El CRM compromete capacidad en `OrganizationAICredit.committedBudget` y crea la autorización en una transacción.
4. Antes de cada invocación facturable, el runtime solicita presupuesto específico.
5. El CRM incrementa condicionalmente `activeAllocatedBudget` y crea la asignación en una transacción.
6. El AI Service ejecuta la invocación dentro del límite asignado.
7. Cuando el resultado es conocido, informa consumo real; el CRM acumula `consumedAmount`, libera el `assignedBudget` completo del contador activo y marca la asignación `settled` en una transacción.
8. El runtime puede solicitar nuevas asignaciones mientras exista presupuesto disponible.
9. Si necesita ampliar el presupuesto total, solicita una ampliación explícita; el CRM valida el saldo organizacional y actualiza `committedBudget` + `authorizedBudget` atómicamente.
10. Al terminar, se verifica que no queden asignaciones reteniendo capacidad ni operaciones externas desconocidas.
11. Se aplica la política de facturación aprobada, se crea el asiento `deduct`, se descuenta el saldo por el importe facturable, se libera el `authorizedBudget` completo de `committedBudget` y se cierra la autorización, todo en la misma transacción.
12. Si hay una operación externa de resultado desconocido, no se ejecuta el paso de liberación/cierre como si la ejecución estuviera resuelta; se conserva el compromiso y se activa reconciliación.

## 11. Reconciliación y observabilidad

Deben existir procesos idempotentes de reconciliación para detectar drift entre:

- `OrganizationAICredit.committedBudget` y la suma de autorizaciones que retienen capacidad;
- `ExecutionBudgetAuthorization.activeAllocatedBudget` y la suma de asignaciones que retienen capacidad;
- consumo reconocido de la autorización y consumo consolidado de sus asignaciones;
- débitos `CreditLedger` y cambios de `OrganizationAICredit.balance`.

La reconciliación no debe corregir contadores a ciegas durante una operación concurrente. Debe usar transacciones/CAS, identificar la discrepancia, conservar evidencia y evitar liberar capacidad cuando exista incertidumbre sobre un proveedor externo.

Metadatos mínimos de auditoría:

- `organizationId`, `workspaceId`, `agentInstanceId`, `executionId`;
- `authorizationId`, `allocationId`, `invocationId`, `attemptId`;
- `fencingGeneration` aplicada, propietario y expiración del lease;
- `authorizedBudget`, `activeAllocatedBudget`, `assignedBudget` y `consumedAmount` antes/después de cada transición relevante;
- `pricingVersion`, tokens/usage informado por proveedor, créditos reconocidos y resultado de invocación;
- clave idempotente o identificador de la solicitud, sin secretos.

Las métricas se pueden agregar por Organización, Workspace, AgentInstance, proveedor/modelo interno y ventana temporal. Los desglose por Workspace/AgentInstance no crean saldos ni límites individuales.

## 13. Impacto en migración, eliminación y dependencias

- `WorkspaceAICredit` se reemplaza por `OrganizationAICredit` como única cuenta de saldo, si el modelo anterior llegó a persistirse.
- `CreditLedger.organizationId` identifica al propietario contable; `workspaceId` permanece como atribución histórica.
- Antes de retirar o migrar un Workspace, deben resolverse sus ejecuciones/autorizaciones/asignaciones pendientes. No se puede borrar una autorización con consumo externo desconocido para liberar capacidad.
- El proceso `delete-wrk` no debe borrar el saldo ni el ledger de la Organización.
- Los asientos financieros append-only y su atribución histórica deben preservarse según la política de retención/auditoría, aunque se eliminen recursos operativos del Workspace.
- La eliminación de datos operativos del Workspace debe coordinarse con reconciliación y cierre de ejecuciones; las referencias históricas de contabilidad deben seguir siendo interpretables.
- La auditoría de Fase 1 informó que no había un sistema de créditos productivo que requiriera migración histórica. Antes de ejecutar migración, comprobar esa premisa contra el estado real del entorno; no asumirla sin evidencia actual.

## 14. ADRs que deben alinearse

### ADR-002 — Modelo de créditos y liquidación

Debe expresar:

- pool único de créditos por Organización;
- Workspace como atribución;
- compromiso temporal mediante `committedBudget`, sin asientos `hold`/`release`;
- autorización máxima por ejecución y asignación máxima por invocación;
- liquidación por consumo reconocido y política de facturación aprobada;
- débito append-only idempotente;
- concurrencia entre Workspaces sin lock exclusivo por Organización.

No describir el mecanismo anterior de créditos por Workspace ni afirmar ausencia de todo tipo de reserva: no hay reserva contable `hold`, pero sí capacidad operativa comprometida.

### ADR-006 — Fencing, leases y recuperación

Debe expresar:

- fencing por autorización individual y asignación individual;
- renovación normal conserva generación;
- recuperación/reasignación incrementa generación;
- validación transaccional/condicional en MongoDB;
- Redis no es por sí solo autoridad para aceptar escrituras;
- lease vencido con operación externa desconocida conserva presupuesto en `reconciliation_required`;
- el fencing no cancela llamadas externas ya aceptadas;
- operaciones de compra/refund/ajuste no usan fencing de una ejecución.

Eliminar referencias a un `lastFencingToken` global en `OrganizationAICredit` y a fencing por Organización/Worker.

## 15. Criterios de aceptación P4

| Criterio | Condición para considerarlo definido |
|---|---|
| Pool compartido | Una Organización tiene un saldo; dos Workspaces pueden ejecutar en paralelo contra ese saldo. |
| Admisión organizacional | `balance - committedBudget` se valida y actualiza atómicamente al crear una autorización. |
| Presupuesto por ejecución | Una ejecución no puede consumir más que `authorizedBudget` sin una ampliación explícita y autorizada. |
| Presupuesto por invocación | No se pueden crear asignaciones concurrentes que superen el presupuesto disponible de la autorización. |
| Contador persistido | `activeAllocatedBudget` se incrementa/decrementa atómicamente en la misma transacción que crea/liquida una asignación. |
| Liquidación por invocación | Suma consumo real a `consumedAmount` y libera el `assignedBudget` completo; no modifica `authorizedBudget` ni `assignedBudget`. |
| Liquidación final | Débito, saldo, liberación del `authorizedBudget` completo y cierre se confirman atómicamente. |
| Fencing | Renovación conserva generación; recuperación aumenta generación; escrituras verifican generación y propiedad actuales. |
| Aislamiento de ejecuciones | La recuperación/liquidación de una autorización no invalida otra autorización de la misma Organización. |
| Incertidumbre externa | Timeout/lease vencido no libera capacidad mientras el resultado externo sea desconocido. |
| Idempotencia | Reintentos no duplican autorizaciones, asignaciones, débitos, compras ni refunds. |
| Ledger | Asientos append-only; refunds son asientos compensatorios. |
| Responsabilidades | CRM es autoridad transaccional del saldo; AI Service ejecuta y reporta consumo. |
| Política de facturación | Los casos de generación/entrega/fallo/timeout quedan cubiertos por la política híbrida aprobada en G.4.6. |
| Reconciliación | Hay procedimientos seguros para comparar contadores y resolver resultados externos inciertos. |

Los criterios aquí descritos representan contratos de diseño, no evidencia de implementación. Su cumplimiento deberá verificarse mediante pruebas de integración y concurrencia en las fases correspondientes.
