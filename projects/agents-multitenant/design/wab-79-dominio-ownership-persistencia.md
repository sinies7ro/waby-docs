# WAB-79 — Dominio, ownership y persistencia

> **Estado:** Documento de trabajo para revisión. No se ha iniciado formalmente la implementación.
> **Fase:** WAB-79 (Bloques B1, B2, B3).
> **Modo:** Solo lectura de fuentes; escritura exclusiva de este documento de trabajo.
> **Restricción:** No modificar código, schemas, tests, configuración, bases de datos, Linear ni GitHub.

---

## 1. Contexto, objetivo y alcance de WAB-79

### 1.1 Objetivo

Preparar una matriz técnica inicial que permita contrastar el modelo de dominio y los contratos aprobados durante WAB-78 con la implementación real de los servicios (CRM y ai-service). La actividad es exclusivamente de investigación, diagnóstico y diseño de dominio, ownership, persistencia e índices. No es una tarea de implementación.

### 1.2 Alcance (WAB-79)

- Definición del modelo de dominio objetivo (B2).
- Definición de persistencia, colecciones, campos, referencias e índices (B3).
- Diagnóstico técnico de la implementación actual (B1) y reconciliación con WAB-78 (R1â€“R10).
- Separación estricta de responsabilidades con fases posteriores.

### 1.3 Fuera de alcance de WAB-79

| Fase | Alcance | Motivo de exclusión |
|------|---------|---------------------|
| WAB-80 | Provisioning, autorización y permisos | `wrk.agents.*`, guards de autorización |
| WAB-81 | Runtime, ejecución y mensajería | Aplicación runtime de AgentDefinition vigente y sus estados, Outbox/Inbox, ARQ-22 implementación |
| WAB-82 | Operación de Knowledge, Skills y Rules | Runtime de Knowledge, Skills, Rules por instancia |
| WAB-84 | Migración operativa y rollback | Mapping, backfill, cutover de Bot/BotInstance legacy |
| WAB-85 | Créditos y observabilidad | `OrganizationAICredit`, `CreditLedger`, presupuestos (P4) |

---

## 2. Decisiones heredadas de WAB-78 (no reabrir)

| ID | Decisión | Origen |
|----|----------|--------|
| C1 | AgentDefinition: propiedad Waby, persistencia en AI Service DB | WAB-78 A.2, D.1.9 |
| C2 | AgentInstance: propiedad Workspace, persistencia en CRM DB | WAB-78 A.2, D.1.9 |
| C3 | AgentInstance almacena `workspaceId`; no se agrega `orgId` por inferencia | WAB-78 regla 4, D.3 |
| C4 | No existe unicidad `workspaceId + agentDefinitionId` | WAB-78 I16 |
| C5 | Cero o una `isDefault=true` por Workspace; cambio atómico | WAB-78 C.5, DEC-CM-08 |
| C6 | Skills y Rules pertenecen a AgentInstance, persisten en CRM | WAB-78 A.2, D.1.9 |
| C7 | Knowledge pertenece al Workspace, persiste en AI Service DB | WAB-78 A.2, D.1.9 |
| C8 | Plantillas `defaultSkills`/`defaultRules` se copian al crear instancia; sin herencia dinámica | WAB-78 A.2, C.7 |
| C9 | Reconciliado: AgentDefinition tiene identidad estable, sin versiones publicadas; nuevas ejecuciones usan la configuración vigente compartida por todas sus instancias vinculadas; las ejecuciones en curso conservan su snapshot. | WAB-78 DEC-CM-01, P10; D-W79-01 aprobada y reflejada en el canónico WAB-78 |
| C10 | AgentExecution conserva `effectiveConfigSnapshot` inmutable, sin secretos | WAB-78 P10, C.7 |
| C11 | CRM es autoridad única de ContactAssignment | WAB-78 D.1.5, E |
| C12 | Contacto usa `assignedType`, `ref_user_assigned`, `ref_agent_assigned` | WAB-78 D.1.5 |
| C13 | EvAssigned es discriminator de `Evento` en colección `eventos` de CRM | WAB-78 D.1.5 |
| C14 | Ciclo de vida P9: `draft`, `active`, `suspended`, `retirement_pending`, `retired` | WAB-78 P9 |
| C15 | Suspensión conserva ContactAssignment; no fallback automático | WAB-78 DEC-CM-07 |
| C16 | No routing persistente canal→instancia; no fallback por prioridad | WAB-78 DEC-CM-04 |
| C17 | P4 (créditos) no se rediseña en WAB-79 | WAB-78 regla 16 |

---

## 3. B1 — Diagnóstico técnico y evidencias

### 3.1 Estado verificado de WAB-78 y WAB-79 (Linear)

| Issue | Estado | Tipo | Dependencia |
|-------|--------|------|-------------|
| WAB-78 | Done (completado 08/11/2026) | Diseño previo a implementación | Padre: WAB-75 |
| WAB-79 | Backlog (no iniciado) | Implementación | Padre: WAB-75; depende de WAB-78 |

### 3.2 Entidades existentes en el código

| # | Entidad (dominio / código) | Servicio / DB | Archivo y símbolo | Estado |
|---|---------------------------|---------------|-------------------|--------|
| 1 | Organization / `Organizacion` | CRM, MongoDB `organizaciones` | `crm/src/models/org/Organizacion.model.ts` | Hecho observado |
| 2 | Workspace / `Works` | CRM, MongoDB `workspaces` | `crm/src/models/workspace/Works.model.ts` | Hecho observado |
| 3 | Contacto / `Contacto` | CRM, MongoDB `contactos` | `crm/src/models/chat/Contacto.model.ts` | Parcialmente confirmado (falta `ref_agent_assigned`) |
| 4 | Outbox / `Outbox` | CRM, MongoDB | `crm/src/models/events/Outbox.model.ts` | Hecho observado |
| 5 | BotResource / `BotResource` | CRM, MongoDB `bot_resources` | `crm/src/modules/bot-resources/bot-resources.model.ts` | Hecho observado |
| 6 | BotResourceLink / `BotResourceLink` | CRM, MongoDB `bot_resource_links` | `crm/src/modules/bot-resources/bot-resource-links.model.ts` | Hecho observado |
| 7 | Bot / `Bot` | ai-service, MongoDB `bots` | `crm/microservices/ai-service/src/domain/entities/bot.ts` | Parcialmente confirmado (modelo legacy) |
| 8 | BotInstance / `BotInstance` | ai-service, MongoDB `bot_instances` | `crm/microservices/ai-service/src/domain/entities/bot-instance.ts` | Parcialmente confirmado (modelo legacy) |
| 9 | Prompt / `Prompt` | ai-service, MongoDB `prompts` | `crm/microservices/ai-service/src/domain/entities/prompt.ts` | Hecho observado |
| 10 | PromptVersion / `PromptVersion` | ai-service, MongoDB `prompt_versions` | `crm/microservices/ai-service/src/domain/entities/prompt-version.ts` | Hecho observado |
| 11 | KnowledgeBase / `KnowledgeBase` | ai-service, MongoDB `knowledge_bases` | `crm/microservices/ai-service/src/domain/entities/knowledge-base.ts` | Hecho observado |
| 12 | Document / `DocumentRecord` | ai-service, MongoDB `documents` | `crm/microservices/ai-service/src/infrastructure/database/models/document.model.ts` | Hecho observado |
| 13 | Execution / `Execution` | ai-service, MongoDB `executions` | `crm/microservices/ai-service/src/domain/entities/execution.ts` | Parcialmente confirmado (falta snapshot P10) |
| 14 | ExecutionEvent / `ExecutionEvent` | ai-service, MongoDB `execution_events` | `crm/microservices/ai-service/src/infrastructure/database/models/execution-event.model.ts` | Hecho observado |
| 15 | MemoryFact / `MemoryFact` | ai-service, MongoDB `memory_facts` | `crm/microservices/ai-service/src/domain/entities/memory-fact.ts` | Hecho observado |
| 16 | MemoryRecord / `MemoryRecord` | ai-service, MongoDB `memory_records` | `crm/microservices/ai-service/src/infrastructure/database/models/memory-record.model.ts` | Hecho observado |
| 17 | ContactAssignment / `ContactAssignment` (proyección) | ai-service, MongoDB `contact_assignments` | `crm/microservices/ai-service/src/domain/entities/contact-assignment.ts` | Parcialmente confirmado |

### 3.3 Entidades que NO existen en el código

`AgentDefinition`, `AgentInstance`, `SkillConfiguration`, `AgentRules`, `ConversationMemory`, `AgentExecutionLog`, `OrganizationAICredit`, `CreditLedger`, `ExecutionBudgetAuthorization`, `ExecutionBudgetAllocation`, `WorkspaceKnowledge` (como entidad nombrada).

### 3.4 Brechas identificadas (G1â€“G12)

Brechas conocidas (ausencia esperada de entidades objetivo) y brechas reales (diferencias entre implementación y contrato). Ver sección 4 del informe de reconciliación R1â€“R10.

---

## 4. B2 — Modelo de dominio y ownership

### 4.1 Diagrama de entidades y relaciones

```
WABY (PLATAFORMA)
  AgentDefinition (AI Service DB)
        â”‚ 1:N
        â–¼
WORKSPACE
  AgentInstance (CRM DB)
        â”‚
        â”œâ”€â”€ 1:N SkillConfiguration (CRM DB)
        â”œâ”€â”€ 1:N AgentRules (CRM DB)
        â”œâ”€â”€ 1:N ConversationMemory (AI Service DB)
        â””â”€â”€ 1:N AgentExecution (AI Service DB)
                  â”‚
                  â””â”€â”€ 1:N AgentExecutionLog (AI Service DB)

WORKSPACE â”€â”€ 1:N â”€â”€ WorkspaceKnowledge (AI Service DB)
      â”‚
      â””â”€â”€ compartido por todas las AgentInstances del Workspace

ORGANIZACIÃ“N â”€â”€ 1:N â”€â”€ Workspace (CRM DB)

CRM — AUTORIDAD DE ASIGNACIONES
  Contacto (CRM DB)
        â”œâ”€â”€ assignedType (human | agent | unassigned)
        â”œâ”€â”€ ref_user_assigned (Ref<UsOrgsWorks>)
        â””â”€â”€ ref_agent_assigned (Ref<AgentInstance>)
  Evento (CRM DB) â† EvAssigned es discriminator, no colección propia
```

## 5. B3 — Persistencia, colecciones, campos y referencias

### 5.1 Matriz de ownership y persistencia

| Entidad | Propietario de negocio | Servicio responsable | DB de persistencia | Colección | Autoritativa / proyección |
|---------|----------------------|---------------------|-------------------|-----------|--------------------------|
| Organización | Waby | CRM | CRM DB | `organizaciones` | Autoritativa |
| Workspace | Organización | CRM | CRM DB | `workspaces` | Autoritativa |
| AgentDefinition | Waby | ai-service | AI Service DB | `agent_definitions` | Autoritativa; identidad estable y configuración vigente sin versiones publicadas (D-W79-01, reconciliada con WAB-78) |
| AgentInstance | Workspace | CRM | CRM DB | `agent_instances` | Autoritativa |
| SkillConfiguration | AgentInstance | CRM | CRM DB | `skill_configurations` | Autoritativa |
| AgentRules | AgentInstance | CRM | CRM DB | `agent_rules` | Autoritativa |
| KnowledgeBase (concepto WorkspaceKnowledge) | Workspace | ai-service | AI Service DB | `knowledge_bases` (existente) | Autoritativa |
| Document (concepto KnowledgeDocument) | Workspace | ai-service | AI Service DB | `documents` (existente) | Autoritativa |
| AgentExecution | AgentInstance + Workspace | ai-service | AI Service DB | `agent_executions` | Autoritativa |
| AgentExecutionLog | AgentInstance + Workspace | ai-service | AI Service DB | `agent_execution_logs` | Registro operativo autoritativo (no sustituye el snapshot de AgentExecution) |
| ConversationMemory | AgentInstance + conversación | ai-service | AI Service DB | `conversation_memory` | Operativa |
| Contacto | CRM | CRM | CRM DB | `contactos` | Autoritativa |
| ContactAssignment | CRM | CRM | CRM DB / AI Service DB | `contact_assignments` (proyección en ai-service) | Autoritativa (CRM) / Proyección (ai-service) |
| EvAssigned | CRM | CRM | CRM DB | `eventos` (discriminator) | Autoritativa |
| Outbox de asignación | CRM | CRM | CRM DB | `outbox` | Mecanismo de propagación |

### 5.2 Definición de colecciones y campos

#### 5.2.1 `agent_instances` (CRM DB)

| Campo | Tipo | Obligatorio | Referencia | Índice |
|-------|------|-------------|------------|--------|
| `_id` | String / UUID | Sí | — | PK |
| `agentDefinitionId` | String / UUID | Sí | AgentDefinition (AI Service DB, cross-service) | `{agentDefinitionId: 1}` |
| `workspaceId` | String | Sí | Workspace (CRM DB) | `{workspaceId: 1}` |
| `name` | String | Sí | — | — |
| `description` | String | No | — | — |
| `status` | Enum (`draft`, `active`, `suspended`, `retirement_pending`, `retired`) | Sí | — | `{status: 1}` |
| `isDefault` | Boolean | Sí (default false) | — | Ãšnico parcial: `{workspaceId: 1, isDefault: 1}` donde `isDefault=true` |
| `additionalPrompt` | String | No | — | — |
| `knowledgeBaseIds` | String[] / UUID[] | Sí (default `[]`) | KnowledgeBase (AI Service DB, cross-service; compartidas por Workspace) | — |
| `createdBy` | String | Sí | — | — |
| `createdAt` | Date | Sí | — | — |
| `updatedAt` | Date | Sí | — | — |
| `activatedAt` | Date | No | — | — |
| `suspendedAt` | Date | No | — | — |
| `retiredAt` | Date | No | — | — |

**Restricciones:**
- `workspaceId` obligatorio.
- No existe índice único `workspaceId + agentDefinitionId` (I16 prohibido).
- No existe `orgId` (regla C3).
- Validación cross-tenant: `workspaceId` debe pertenecer a la Organización del contexto autenticado.

**Contrato:** WAB-78 A.2, C.5, DEC-CM-08, P9.

#### 5.2.2 `skill_configurations` (CRM DB)

| Campo | Tipo | Obligatorio | Referencia | Índice |
|-------|------|-------------|------------|--------|
| `_id` | String / UUID | Sí | — | PK |
| `agentInstanceId` | String / UUID | Sí | AgentInstance | `{agentInstanceId: 1}` |
| `skillId` | String | Sí | Catálogo Waby | — |
| `skillName` | String | Sí | — | — |
| `enabled` | Boolean | Sí | — | — |
| `config` | JSON | No | — | — |
| `credentials` | JSON | No | Referencias a credenciales, sin secretos | — |
| `createdAt` | Date | Sí | — | — |
| `updatedAt` | Date | Sí | — | — |

**Restricciones:**
- Pertenencia a AgentInstance (1:N).
- Sin herencia dinámica de `defaultSkills` de AgentDefinition.

**Contrato:** WAB-78 A.2, D.1.9.

#### 5.2.3 `agent_rules` (CRM DB)

| Campo | Tipo | Obligatorio | Referencia | Índice |
|-------|------|-------------|------------|--------|
| `_id` | String / UUID | Sí | — | PK |
| `agentInstanceId` | String / UUID | Sí | AgentInstance | `{agentInstanceId: 1}` |
| `ruleName` | String | Sí | — | — |
| `content` | String | Sí | — | — |
| `priority` | Integer | No | — | — |
| `enabled` | Boolean | Sí | — | — |
| `version` | Integer | Sí | — | — |
| `createdAt` | Date | Sí | — | — |
| `updatedAt` | Date | Sí | — | — |

**Restricciones:**
- Pertenencia a AgentInstance (1:N).
- No confundir con `BotResource` (catálogo global por workspace).

**Contrato:** WAB-78 A.2, D.1.9.

#### 5.2.4 `agent_definitions` (AI Service DB)

| Campo | Tipo | Obligatorio | Índice |
|-------|------|-------------|--------|
| `_id` | String / UUID | Sí | PK |
| `name` | String | Sí | — |
| `description` | String | No | — |
| `systemPrompt` | String | Sí | — |
| `provider` | Enum | Sí | — |
| `model` | String | Sí | — |
| `temperature` | Decimal | No | — |
| `maxTokens` | Integer | No | — |
| `topP` | Decimal | No | — |
| `frequencyPenalty` | Decimal | No | — |
| `presencePenalty` | Decimal | No | — |
| `defaultSkills` | JSON[] | No | — |
| `defaultRules` | JSON[] | No | — |
| `status` | Enum (`draft`, `active`, `suspended`, `retired`) | Sí | `{status: 1}` |
| `createdBy` | String | Sí | — |
| `createdAt` | Date | Sí | — |
| `updatedAt` | Date | Sí | — |
| `retiredAt` | Date | No | — |

**Restricciones:**
- `_id` es la identidad lógica estable de la definición; no se introduce una entidad de versiones ni un campo `version`.
- Los cambios en la configuración vigente de AgentDefinition se reflejan en todas las AgentInstance que la referencian, sin modificar `agentDefinitionId`.
- `draft` no es ejecutable; `active` permite nuevas ejecuciones; `suspended` bloquea nuevas ejecuciones de todas las instancias vinculadas; `retired` es terminal y no ejecutable.
- El runtime debe validar el estado vigente de AgentDefinition antes de iniciar cada ejecución. Las ejecuciones ya iniciadas conservan su snapshot; por defecto no se cancelan retroactivamente.
- Las modificaciones de configuración deben propagarse a todas las instancias vinculadas; cualquier caché debe invalidarse o refrescarse para que ninguna ejecución nueva use indefinidamente una configuración obsoleta.
- `defaultSkills`/`defaultRules` son plantillas de inicialización, no herencia dinámica.
- D-W79-01 elimina el versionado publicado de AgentDefinition y queda reconciliada con DEC-CM-01/C9 en el documento canónico WAB-78. La configuración vigente se aplica a nuevas ejecuciones de todas las instancias vinculadas; el snapshot de cada ejecución preserva los valores efectivos utilizados.

**Contrato:** WAB-78 A.2, D.1.9; D-W79-01 reconciliada con DEC-CM-01/C9 y P10/C.7.

#### 5.2.5 `knowledge_bases` (AI Service DB)

| Campo | Tipo | Obligatorio | Referencia | Índice |
|-------|------|-------------|------------|--------|
| `_id` | String / UUID | Sí | — | PK |
| `workspaceId` | String | Sí | Workspace (CRM DB) | `{workspaceId: 1}` |
| `name` | String | Sí | — | — |
| `description` | String | No | — | — |
| `status` | Enum (`active`, `inactive`, `deleted`) | Sí | `{status: 1}` |
| `embeddingConfig` | JSON | Sí | — | — |
| `vectorStoreConfig` | JSON | Sí | — | — |
| `retrievalConfig` | JSON | No | — | — |
| `documentCount` | Integer | No | — | — |
| `createdAt` | Date | Sí | — | — |
| `updatedAt` | Date | Sí | — | — |
| `deletedAt` | Date | No | — | — |

**Restricciones:**
- Workspace 1:N WorkspaceKnowledge.
- Compartido por todas las AgentInstances del mismo Workspace.
- No duplicar autoridad en CRM.

**Contrato:** WAB-78 A.2, D.1.9, I11.

**Evidencia actual:** `KnowledgeBase` en ai-service (`knowledge-base.model.ts`) cumple esta semántica con `workspaceId`.

#### 5.2.6 `documents` (AI Service DB)

| Campo | Tipo | Obligatorio | Referencia | Índice |
|-------|------|-------------|------------|--------|
| `_id` | String / UUID | Sí | — | PK |
| `workspaceId` | String | Sí | Workspace | `{workspaceId: 1}` |
| `knowledgeBaseId` | String / UUID | Sí | WorkspaceKnowledge | `{knowledgeBaseId: 1}` |
| `name` | String | Sí | — | — |
| `mimeType` | String | Sí | — | — |
| `size` | Number | Sí | — | — |
| `status` | Enum (`pending`, `processing`, `completed`, `failed`) | Sí | `{knowledgeBaseId: 1, status: 1}` |
| `createdAt` | Date | Sí | — | — |
| `updatedAt` | Date | Sí | — | — |

**Contrato:** WAB-78 D.1.9 (Knowledge en AI Service DB).

**Evidencia actual:** `Document` en ai-service (`document.model.ts`).

#### 5.2.7 `agent_executions` (AI Service DB)

| Campo | Tipo | Obligatorio | Referencia | Índice |
|-------|------|-------------|------------|--------|
| `_id` | String / UUID | Sí | — | PK |
| `workspaceId` | String | Sí | Workspace | `{workspaceId: 1}` |
| `organizationId` | String / UUID | Sí | Organización (derivada de Workspace) | `{organizationId: 1}` |
| `agentInstanceId` | String / UUID | Sí | AgentInstance | `{agentInstanceId: 1}` |
| `conversationId` | String | Sí | — | `{conversationId: 1, startedAt: -1}` |
| `contactId` | String | Sí | Contacto | — |
| `channel` | String | No | — | — |
| `input` | String | No | — | — |
| `correlationData` | Objeto (traceId, executionId, correlationId) | Sí | — | — |
| `agentDefinitionId` | String / UUID | Sí | Identidad estable de AgentDefinition | `{agentDefinitionId: 1}` |
| `promptSnapshotHash` | String | Sí | — | — |
| `effectiveConfigHash` | String | Sí | — | — |
| `effectiveConfigSnapshot` | JSON | Sí | — | — |
| `providerModelUsed` | String | Sí | — | — |
| `inputTokens` | Integer | Condicional al finalizar | — | — |
| `outputTokens` | Integer | Condicional al finalizar | — | — |
| `totalTokens` | Integer | Condicional al finalizar | — | — |
| `totalCreditsDeducted` | Decimal | Condicional por estado; semántica WAB-85 | — | — |
| `currency` | String | Condicional por estado; semántica WAB-85 | — | — |
| `latencyMs` | Integer | Condicional al finalizar | — | — |
| `status` | Enum (`pending`, `completed`, `failed`, `cancelled`) | Sí | `{status: 1}` |
| `error` | String | No | — | — |
| `startedAt` | Date | Sí | — | `{conversationId: 1, startedAt: -1}` |
| `createdAt` | Date | Sí | — | — |
| `completedAt` | Date | No | — | — |

**Restricciones:**
- `effectiveConfigSnapshot` obligatorio, inmutable, sin secretos.
- `promptSnapshotHash` y `effectiveConfigHash` obligatorios por ejecución.
- No sustituir el snapshot por hashes.
- `organizationId` obligatorio para atribución/ownership de la ejecución según WAB-78 A.2; no se agrega a AgentInstance. La semántica de créditos sigue en WAB-85.

**Contrato:** WAB-78 P10, C.7.

#### 5.2.8 `agent_execution_logs` (AI Service DB)

| Campo | Tipo | Obligatorio | Referencia | Índice |
|-------|------|-------------|------------|--------|
| `_id` | String / UUID | Sí | — | PK |
| `executionId` | String / UUID | Sí | AgentExecution | `{executionId: 1}` |
| `agentInstanceId` | String / UUID | Sí | AgentInstance | — |
| `workspaceId` | String | Sí | Workspace | `{workspaceId: 1}` |
| `eventType` | Enum | Sí | — | `{eventType: 1}` |
| `promptSnapshotHash` | String | Sí | — | — |
| `providerModelUsed` | String | Sí | — | — |
| `inputTokens` | Integer | Sí | — | — |
| `outputTokens` | Integer | Sí | — | — |
| `totalCreditsDeducted` | Decimal | Sí | — | — |
| `latencyMs` | Integer | Sí | — | — |
| `result` | String | No | — | — |
| `error` | String | No | — | — |
| `createdAt` | Date | Sí | — | `{executionId: 1, createdAt: 1}` |

**Restricciones:**
- AgentExecution 1:N AgentExecutionLog.
- No sustituye `AgentExecution.effectiveConfigSnapshot`.

**Contrato:** WAB-78 A.2, I21.

#### 5.2.9 `conversation_memory` (AI Service DB)

| Campo | Tipo | Obligatorio | Referencia | Índice |
|-------|------|-------------|------------|--------|
| `_id` | String / UUID | Sí | — | PK |
| `agentInstanceId` | String / UUID | Sí | AgentInstance | `{agentInstanceId: 1}` |
| `conversationId` | String | Sí | — | `{agentInstanceId: 1, conversationId: 1}` único |
| `workspaceId` | String | Sí | Workspace | `{workspaceId: 1}` |
| `messages` | JSON[] | Sí | — | — |
| `messageCount` | Integer | Sí | — | — |
| `createdAt` | Date | Sí | — | — |
| `updatedAt` | Date | Sí | — | — |
| `expiresAt` | Date | No | — | — |

**Restricciones:**
- Máximo 20 mensajes user/assistant.
- Una conversación tiene exactamente una ConversationMemory por AgentInstance.

**Contrato:** WAB-78 A.2, I14.

#### 5.2.10 `contactos` (CRM DB) — cambios propuestos

| Campo | Tipo | Obligatorio | Referencia | Índice |
|-------|------|-------------|------------|--------|
| `ref_agent_assigned` | Ref<AgentInstance> | No | AgentInstance (CRM DB) | `{ref_agent_assigned: 1}` parcial donde no null |
| `assignedType` | Enum (`human`, `agent`, `unassigned`) | Sí | — | — |

**Invariantes:**
- Si `assignedType=agent`: `ref_agent_assigned` obligatorio, `ref_user_assigned=null`.
- Si `assignedType=human`: `ref_user_assigned` obligatorio, `ref_agent_assigned=null`.
- Si `assignedType=unassigned`: ambos null.
- Contacto y AgentInstance deben pertenecer al mismo Workspace.

**Contrato:** WAB-78 D.1.5, E.

**Evidencia actual:** `Contacto` tiene `ref_user_assigned` pero no `ref_agent_assigned` (`Contacto.model.ts` línea 157).

#### 5.2.11 `contact_assignments` (AI Service DB — proyección)

| Campo | Tipo | Obligatorio | Índice |
|-------|------|-------------|--------|
| `id` | String | Sí | PK |
| `contactId` | String | Sí | `{contactId: 1, status: 1}` único parcial donde `status='active'` |
| `workspaceId` | String | Sí | `{workspaceId: 1, status: 1}` |
| `assigneeType` | Enum de proyección legacy (`user`, `bot`) con mapeo explícito al vocabulario objetivo (`human`, `agent`, `unassigned`) | Sí | `{assigneeType: 1, assigneeId: 1, status: 1}` |
| `assigneeId` | String | Sí | — |
| `status` | Enum (`active`, `released`) | Sí | — |
| `createdAt` | Date | Sí | `{contactId: 1, createdAt: -1}` |

**Restricciones:**
- Cero o una asignación activa por contacto.
- Proyección derivada de CRM; no es autoridad editable.

**Contrato:** WAB-78 D.1.9, DEC-CM-06.

---

## 6. Índices, restricciones de integridad y concurrencia

### 6.1 Índices propuestos

| Colección | Índice | Tipo | Propósito |
|-----------|--------|------|-----------|
| `agent_instances` | `{workspaceId: 1, isDefault: 1}` | Ãšnico parcial (`isDefault=true`) | DEC-CM-08: como máximo una predeterminada por Workspace |
| `agent_instances` | `{agentDefinitionId: 1}` | Simple | Consultas por definición |
| `agent_instances` | `{workspaceId: 1, status: 1}` | Compuesto | Consultas activas por workspace |
| `skill_configurations` | `{agentInstanceId: 1}` | Simple | Consultas por instancia |
| `agent_rules` | `{agentInstanceId: 1}` | Simple | Consultas por instancia |
| `knowledge_bases` | `{workspaceId: 1, status: 1}` | Compuesto | Knowledge activo por workspace |
| `documents` | `{knowledgeBaseId: 1, status: 1}` | Compuesto | Documentos por KB y estado |
| `agent_executions` | `{agentInstanceId: 1}` | Simple | Ejecuciones por instancia |
| `agent_executions` | `{workspaceId: 1}` | Simple | Ejecuciones por workspace |
| `agent_executions` | `{conversationId: 1, startedAt: -1}` | Compuesto | Historial por conversación |
| `agent_executions` | `{status: 1}` | Simple | Ejecuciones por estado |
| `agent_execution_logs` | `{executionId: 1, createdAt: 1}` | Compuesto | Logs por ejecución ordenados |
| `conversation_memory` | `{agentInstanceId: 1, conversationId: 1}` | Ãšnico compuesto | Una memoria por conversación y AgentInstance |
| `contactos` | `{ref_agent_assigned: 1}` | Simple (parcial donde no null) | Consultas por agente asignado |
| `contactos` | `{ref_user_assigned: 1}` | Simple | Consultas por usuario asignado |
| `contact_assignments` | `{contactId: 1, status: 1}` | Ãšnico parcial (`status='active'`) | Cero o una activa por contacto |

**Nota:** Ningún índice declarado se asume aplicado en producción sin verificación. La existencia de un índice en esta matriz es una propuesta de schema, no evidencia de despliegue.

**Limitaciones de evidencia B1:** no se verificó la aplicación de índices en producción ni el soporte efectivo de transacciones multidocumento en MongoDB Atlas. La estrategia de atomicidad de `isDefault` queda pendiente de validación de infraestructura.

### 6.2 Reglas de integridad y concurrencia

#### 6.2.1 Cambio atómico de `isDefault`

- Como máximo una `AgentInstance` puede tener `isDefault=true` por Workspace.
- El cambio de predeterminada debe ser atómico: desactivar la anterior y activar la nueva en una misma transición lógica.
- No altera asignaciones existentes de Contacto.
- La nueva predeterminada solo se usa para contactos sin asignación activa al evaluar un nuevo mensaje entrante.

**Contrato:** WAB-78 DEC-CM-08.

#### 6.2.2 Validación cross-tenant

Cadena: `AgentInstance.workspaceId → Workspace → Organización`.

- Al crear o actualizar `AgentInstance`, validar que el `workspaceId` pertenece a la Organización del contexto autenticado.
- No confiar en `organizationId` recibido libremente en payloads.
- No almacenar `orgId` en `AgentInstance` (regla C3).

**Contrato:** WAB-78 D.3.

#### 6.2.3 Integridad de Contacto y asignación

- `ref_agent_assigned` es referencia tipada a `AgentInstance._id`.
- Contacto y AgentInstance deben pertenecer al mismo Workspace.
- CRM es autoridad única; `contact_assignments` en ai-service es proyección derivada vía Outbox.
- `EvAssigned` identifica la transición que autorizó cada ejecución; permite comparar contra la asignación vigente antes del despacho.

**Contrato:** WAB-78 D.1.5, E, DEC-CM-06.

#### 6.2.4 Eliminación y referencias huérfanas

- `retired` representa retiro lógico terminal de `AgentInstance`; no equivale a purga física. La retención y eventual purga física de la instancia y sus datos asociados debe respetar las políticas de Waby y los contratos de retención; no eliminar registros de ejecución ni snapshots por cascada.
- Eliminación de `WorkspaceKnowledge`: no afecta a `AgentInstance` que la referencien (referencias pueden volverse obsoletas; gestión en WAB-82).
- `BotResourceLink` con `botId` string plano sin Ref (cross-DB): riesgo de links huérfanos si se elimina un Bot en ai-service; gestión en GC (fuera de alcance WAB-79).

## 7. Correspondencia entre modelos legacy y objetivo

| Legacy (actual) | Objetivo (WAB-79) | Notas |
|-----------------|-------------------|-------|
| `Bot` (ai-service) | `AgentDefinition` (AI Service DB) + parte de `AgentInstance` | Separar definición de instancia |
| `BotInstance` (ai-service) | `AgentInstance` (CRM DB) | Mover persistencia a CRM; agregar `isDefault`, estados P9, `additionalPrompt` |
| `BotResource` / `BotResourceLink` (CRM) | `SkillConfiguration` + `AgentRules` (CRM DB) | Pasar de catálogo global a configuración por instancia |
| `KnowledgeBase` (ai-service) | `KnowledgeBase` (AI Service DB; concepto WorkspaceKnowledge) | Mantener colección y modelo físico existentes; nombre conceptual no implica renombrado |
| `Document` (ai-service) | `Document` (AI Service DB; concepto KnowledgeDocument) | Mantener colección y modelo físico existentes; nombre conceptual no implica renombrado |
| `Execution` (ai-service) | `AgentExecution` (AI Service DB) | Agregar `promptSnapshotHash`, `effectiveConfigHash`, `effectiveConfigSnapshot` |
| `ExecutionEvent` (ai-service) | `AgentExecutionLog` (AI Service DB) | Registro operativo autoritativo; revisar alineación con P10 sin tratarlo como proyección derivada |
| `MemoryFact` / `MemoryRecord` (ai-service) | `ConversationMemory` (AI Service DB) | Nueva entidad por AgentInstance+conversación |
| `Contacto` (CRM) | `Contacto` (CRM) | Agregar `ref_agent_assigned`, `assignedType` |
| `ContactAssignment` (ai-service proyección) | `ContactAssignment` (ai-service proyección) | Sin cambios estructurales; CRM sigue siendo autoridad |
| `EvAssigned` (CRM evento) | `EvAssigned` (discriminator de Evento en `eventos`) | Mantener como discriminator; el contrato requiere datos suficientes para identificar transición, actor, momento y contexto |

## 8. Cuestiones técnicas pendientes de resolución

Las siguientes cuestiones técnicas permanecen abiertas sin inventar decisiones. La identidad estable y ausencia de versiones publicadas de AgentDefinition ya están aprobadas mediante D-W79-01 y reconciliadas con DEC-CM-01/C9 en el documento canónico WAB-78; no deben reabrirse en WAB-79:

1. **Relación entre AgentDefinition, routing y parámetros de proveedor/modelo controlados por Waby.**
2. **Índice único compuesto para ConversationMemory por AgentInstance y conversación.**
3. **Vocabulario objetivo de ContactAssignment, sin perpetuar el término legacy `bot`.**
4. **Campos e índices de AgentExecution, incluyendo `createdAt` y campos que solo se completan al finalizar.**
5. **Relación conceptual y física entre WorkspaceKnowledge/KnowledgeBase y KnowledgeDocument/Document.**
6. **Referencias seguras a credenciales en SkillConfiguration, sin almacenar secretos.**
7. **Cardinalidad e índice de SkillConfiguration por AgentInstance y skillId.**
8. **Semántica del versionado de AgentRules.**
9. **Autoridad y función de AgentExecutionLog.**
10. **Validación de pertenencia de KnowledgeDocument y KnowledgeBase al mismo Workspace.**
11. **Unificación de los índices de asignación de Contacto.**
12. **Distinción entre campos heredados de Evento y campos propios de EvAssigned.**
13. **Estrategia técnica de atomicidad de `isDefault`.**
14. **Tratamiento de `BotResourceLink` y referencias legacy, respetando el alcance de WAB-84.**

**Clasificación de cada cuestión:**
- Decisión ya aprobada: aplicar tal cual (no reabrir).
- Requisito heredado de WAB-78: implementar según contrato.
- Propuesta técnica: definir en WAB-79, sujeto a aprobación.
- Cuestión pendiente: requiere decisión humana antes de implementar.
- Evidencia no verificada: requiere inspección adicional.

## 9. Decisiones y propuestas de diseño registradas durante WAB-79

| ID | Decisión | Estado | Origen |
|----|----------|--------|--------|
| D-W79-01 | Identidad estable de AgentDefinition sin versiones publicadas; los cambios se reflejan en todas las instancias vinculadas y los estados `draft`, `suspended` y `retired` bloquean nuevas ejecuciones. Solo `active` permite nuevas ejecuciones. | Aprobada por el usuario y reconciliada en el canónico WAB-78 (DEC-CM-01/C9, P10/C.7) | Revisión WAB-79, 09/11/2026 |

**Trazabilidad:** D-W79-01 fue aprobada para el diseño WAB-79 y reconciliada en el documento canónico WAB-78 (`fase-2-diseno-objetivo-y-contratos.md`), en DEC-CM-01/C9 y el contrato de snapshot P10/C.7. La referencia estable es `AgentDefinition._id`; no se mantiene `agentDefinitionVersion`.

## 10. Criterios de aceptación para autorizar implementación

Antes de iniciar la implementación de WAB-79, debe verificarse:

1. **Esquema `AgentDefinition` definido** en AI Service DB con identidad estable, estado global y configuración vigente, sin versiones publicadas (D-W79-01 ya reconciliada con WAB-78).
2. **Esquema `AgentInstance` definido** en CRM DB con `workspaceId`, `agentDefinitionId`, `isDefault`, estados P9, sin `orgId`, sin unicidad `workspaceId + agentDefinitionId` (contratos C2, C3, C4, C5).
3. **Índice único parcial `{workspaceId: 1, isDefault: 1}`** donde `isDefault=true` (contrato C5).
4. **Esquemas `SkillConfiguration` y `AgentRules`** en CRM DB con `agentInstanceId` (contrato C6).
5. **Persistencia de Knowledge** en AI Service DB mediante las colecciones físicas existentes `knowledge_bases` y `documents`, con ownership por Workspace (contrato C7).
6. **Esquema `AgentExecution`** en AI Service DB con `promptSnapshotHash`, `effectiveConfigHash`, `effectiveConfigSnapshot` obligatorios (contrato C10).
7. **Campo `ref_agent_assigned`** en `Contacto` de CRM con invariantes de `assignedType` (contrato C12).
8. **Validación cross-tenant** `AgentInstance.workspaceId → Workspace → Organización` documentada (contrato C3, D.3).
9. **Matriz de ownership y persistencia** revisada y aprobada (sección 5.1).
10. **Cuestiones pendientes de la sección 8** clasificadas como: decisión requerida, propuesta técnica o evidencia no verificada.

## 11. Historial de cambios

| Versión | Fecha | Cambio | Autor |
|---------|-------|--------|-------|
| 1.0 | 08/11/2026 | Creación del documento consolidado B1/B2/B3 | Agente OpenCode |
| 1.1 | 08/11/2026 | Pasada correctiva: identidad/estado global de AgentDefinition, eliminación propuesta de versionado, consistencia de persistencia/índices, snapshot, ownership y limpieza de duplicados | ChatGPT (revisión) |
| 1.2 | 09/11/2026 | Reconciliación cruzada D-W79-01 con WAB-78 DEC-CM-01/C9 y P10/C.7; actualización de referencias pendientes, alcance WAB-81 y criterios de aceptación | ChatGPT (revisión) |
| 1.3 | 09/11/2026 | Ajuste de alcance y trazabilidad operativa tras la reconciliación cruzada con el documento canónico WAB-78 | ChatGPT (revisión) |

---

## Alcance y restricciones operativas de esta actualización

- Se actualizaron conjuntamente este documento WAB-79 y el documento canónico WAB-78 para mantener reconciliado D-W79-01.
- No se modificó código, modelos, schemas, tests ni configuraciones.
- No se modificó Linear (issues, estados ni dependencias); WAB-79 conserva su estado Backlog.
- No se repitió la auditoría B1 ni se declararon índices aplicados en producción sin evidencia.
- Se mantuvo separada la regla de dominio de su mecanismo técnico de implementación.



## 16. Retiro coordinado de AgentInstance

> **Estado:** Contrato de diseño aprobado en la sesión WAB-79 del 11/11/2026. No implica implementación. Este apartado concreta P9 y prevalece sobre formulaciones anteriores incompatibles con el retiro coordinado.

### 16.1 Principios y lifecycle

El retiro es lógico y coordinado; no equivale a eliminación física inmediata ni agrega un booleano `deleted`. La AgentInstance pasa a `retirement_pending` sólo después de confirmar durablemente ambos bloqueos: CRM impide nuevas asignaciones y el servicio propietario de ejecución impide nuevas ejecuciones. Al completar todas las verificaciones pasa a `retired`, estado terminal que conserva identidad y referencias históricas (EvAssigned y AgentExecution).

La retención/purga de configuración, credenciales/conexiones, Knowledge, Skills, Rules, objetos MinIO e historiales tiene políticas independientes todavía no definidas. El retiro lógico no las elimina automáticamente.

### 16.2 Doble bloqueo y recuperación

CRM es autoridad de asignaciones y propietario único de `RetirementOperation`. El servicio de ejecución es autoridad de admisión de ejecuciones. No existe transacción distribuida entre ambos.

La operación usa un `retirementOperationId` estable. Se solicitan los dos bloqueos; cada autoridad persiste y confirma durablemente el mismo ID. Sólo cuando ambas confirmaciones están persistidas se pasa a `releasing` y AgentInstance pasa a `retirement_pending`. Los bloqueos pueden existir antes del cambio de lifecycle. Una confirmación parcial mantiene la operación en `blocking` o `blocked`; no se agrega otro estado a AgentInstance.

Solicitudes/reintentos son idempotentes. Ante respuesta perdida o recuperación, CRM consulta a cada autoridad por el mismo `retirementOperationId`; no infiere fallo por timeout ni crea otro ID. Si CRM o el servicio de ejecución no están disponibles, no se considera confirmado el bloqueo faltante.

### 16.3 Estrategia y destino

Cada operación define una única estrategia para todos los contactos: `unassign`, `reassign_user` a un usuario elegible del mismo Workspace, o `reassign_agent` a otra AgentInstance del mismo Workspace. El destino se valida al inicio; no se permite auto-reasignación. Estrategia y destino son inmutables.

Si el destino B se vuelve inactivo después de iniciar A→B, la instrucción no cambia: se continúa intentando transferir a B, sin cambiarlo silenciosamente, crear una operación hija ni bloquear el resto por ese cambio. Los contactos ya transferidos permanecen en B. La garantía del retiro de A es que ninguno siga asignado a A al completar.

### 16.4 Modelo conceptual de RetirementOperation

CRM mantiene la única `RetirementOperation`; AI Service no persiste una copia de la operación de retiro. Campos conceptuales:

| Campo | Semántica |
|---|---|
| `_id` | Identidad estable `retirementOperationId` |
| `workspaceId`, `agentInstanceId` | Ámbito |
| `actor` | `actorType` y `actorRef`, compatibles con EvAssigned |
| `strategy` | Tipo y `destinationId`, inmutables |
| `status`, `resumeFrom` | Estado y etapa de recuperación |
| `blockedReason` | Código estable y detalles opcionales |
| `lockConfirmations` | Confirmaciones durables CRM/servicio de ejecución y fecha |
| `processing` | `leaseOwner`, `leaseUntil`, `leaseGeneration` |
| `eventSetRevision`, `eventSetClosedAt` | Coordinación y cierre del conjunto esperado |
| `lastProgressAt`, `attempts` | Progreso/reintentos |
| `createdAt`, `updatedAt`, `completedAt` | Marcas temporales |

Es un modelo conceptual, no un schema final listo para implementar.

### 16.5 Estados e invariantes

| Estado | Invariante |
|---|---|
| `requested` | Operación creada y destino validado; bloqueo aún no iniciado |
| `blocking` | Solicita o recupera ambos bloqueos |
| `releasing` | Ambos bloqueos confirmados; AgentInstance en `retirement_pending`; drenaje de contactos |
| `finalizing` | Cero asignaciones actuales; conjunto de eventos cerrado; verifica ejecuciones y proyección |
| `blocked` | Recuperable; conserva causa, progreso, bloqueos y `resumeFrom` |
| `completed` | Todas las condiciones verificadas; terminal |

Transiciones permitidas: `requested → blocking → releasing → finalizing → completed`; desde `blocking`, `releasing` o `finalizing` se puede pasar a `blocked`; `blocked` vuelve a la etapa indicada por `resumeFrom`. No hay `aborting` ni `aborted`; una vez iniciado el retiro no se cancela. `completed` no tiene salidas. AgentInstance pasa a `retired` sólo al completar la operación.

Códigos estables de `blockedReason`: `CRM_LOCK_UNAVAILABLE`, `EXECUTION_LOCK_UNAVAILABLE`, `DESTINATION_INVALID` (al iniciar), `CONTACT_TRANSITION_CONFLICT`, `PROJECTION_NOT_CAUGHT_UP`, `PENDING_EXECUTIONS`; pueden llevar detalles opcionales. Fallos transitorios se reintentan con backoff; problemas permanentes requieren intervención administrativa. No se cambia destino automáticamente.

### 16.6 Unicidad, lease y fencing

Sólo puede existir una operación no terminal por AgentInstance. Se requiere índice único parcial conceptual sobre `agentInstanceId` para `requested`, `blocking`, `releasing`, `finalizing` y `blocked`; `completed` libera la unicidad. La creación es idempotente por `retirementOperationId`; una solicitud concurrente perdedora recibe conflicto y puede consultar la existente.

Lease conceptual:
```ts
processing: {
  leaseOwner: string | null,
  leaseUntil: Date | null,
  leaseGeneration: number
}
```
Adquisición atómica; cada nueva adquisición efectiva incrementa generación; renovación conserva generación; lease vencido puede recuperarse. Toda escritura valida owner, generación y vigencia. Worker obsoleto no puede avanzar estado ni escribir progreso. Lease/fencing no reemplaza CAS ni idempotencia. La recuperación conserva estrategia, destino, bloqueos y transiciones confirmadas; consulta autoridades con el mismo ID, reanuda desde `resumeFrom` y no repite efectos confirmados. Contadores son informativos, no fuente de verdad.

### 16.7 Admisión de ejecuciones

La aceptación de ejecución y el bloqueo compiten atómicamente sobre el control durable de admisión:
- Si gana la aceptación, se crea durablemente `AgentExecution` en `pending` con su `executionId`; debe terminar normalmente.
- Si gana el bloqueo, la nueva ejecución se rechaza.
- Reintentar el mismo `executionId` es idempotente.
- No se cancelan automáticamente ejecuciones aceptadas antes del bloqueo.

Campo conceptual:
```ts
executionAdmission: {
  status: 'open' | 'blocked',
  retirementOperationId: string | null,
  blockedAt: Date | null
}
```
La topología y transacción concretas deben garantizar la exclusión entre admisión y creación durable de AgentExecution; una lectura seguida de escritura separada no basta.

### 16.8 Drenaje dinámico y CAS por contacto

El worker consulta contactos que siguen asignados a A, toma un lote acotado, relee y procesa cada contacto individualmente, y vuelve a consultar el conjunto vivo hasta obtener cero. Después realiza verificación final independiente contra el estado autoritativo de CRM. No depende de cursor histórico ni de contadores.

Cada transición valida mediante CAS la asignación actual a A y que `ref_ev_assigned` coincida con el evento previo esperado. Cada contacto se procesa en transacción MongoDB independiente con Contacto + EvAssigned + Outbox atómicos. Si otra transición válida ya movió o liberó el contacto, el retiro respeta el resultado, no duplica evento ni revierte la decisión.

La regla lógica aprobada es como máximo una transición de retiro por contacto y operación. La clave exacta del índice y su convivencia con transiciones externas quedan para concreción técnica posterior.

### 16.9 Correlación, causalidad y confirmación durable

Toda transición válida que saque un contacto de A mientras el retiro de A esté activo queda correlacionada con ese `retirementOperationId`, aunque la inicie otro administrador/proceso. La correlación no altera `actorType`, `actorRef` ni `operationType`: reflejan al actor y operación reales. El conjunto esperado incluye tanto transiciones del worker como transiciones concurrentes externas que sacaron contactos de A.

CRM obtiene los eventos esperados de los EvAssigned correlacionados, no de un preconteo ni de contadores. `SENT` en Outbox no significa aplicado.

AI Service usa Inbox durable y cursor causal: aplica sólo cuando el cursor coincide con `previousEvAssignedId`; eventos futuros esperan al predecesor y eventos obsoletos no revierten la proyección. `lastProcessedEvAssignedId` conserva la última transición aplicada incluso tras liberar; la asignación activa queda nula al liberar.

En una transacción local de AI Service, al aplicar cada evento: actualiza `contact_assignments`, avanza `lastProcessedEvAssignedId`, marca Inbox aplicado y crea confirmación durable en su Outbox. La confirmación significa aplicación real en la proyección, no recepción ni publicación. AI Service reintenta hasta que CRM la persista; CRM guarda confirmación idempotentemente por `evAssignedId` y la vincula a RetirementOperation.

### 16.10 Cierre del conjunto esperado

Mientras la operación esté en `releasing`, toda transición que saque un contacto de A escribe el mismo documento RetirementOperation en la misma transacción que Contacto, EvAssigned y Outbox, incrementando `eventSetRevision`. Esto incluye transiciones concurrentes externas.

Para pasar a `finalizing`, una transacción valida estado `releasing`, lease/fencing vigente y cero contactos asignados a A; cierra el conjunto y registra `eventSetClosedAt`. Tanto el alta de eventos como el cierre escriben el mismo documento RetirementOperation. Una colisión debe abortar/reintentar y volver a validar. En `finalizing` el conjunto es inmutable; sólo pueden seguir llegando confirmaciones pendientes.

### 16.11 Condiciones de finalización

CRM sólo marca `completed` y AgentInstance `retired` cuando verifica: (1) cero contactos actualmente asignados a A; (2) ninguna AgentExecution aceptada pendiente o en procesamiento; (3) todos los EvAssigned del conjunto cerrado aplicados y confirmados durablemente por AI Service; (4) ningún evento causal pendiente de la operación en Inbox; y (5) confirmaciones persistidas por CRM para todos los eventos esperados. Se consulta el estado real, no contadores.

Si AI Service no está disponible o la proyección no alcanzó el último evento, la operación permanece en `finalizing` o `blocked`, conserva bloqueos y reintenta. No se marca `retired` por timeout ni por Outbox `SENT`.

### 16.12 Detalles pendientes de concreción técnica

Sin reabrir las reglas aprobadas, quedan por definir: índices/clave exacta de idempotencia por contacto-operación y confirmación; schema final y ubicación del control de admisión; nombres definitivos, payloads y contratos de transporte Inbox/Outbox; propagación de sesión y topología MongoDB para atomicidad; y política independiente de retención/purga física. No se incorpora telemetría, dashboards ni nuevos registros de observabilidad, que están fuera del alcance. No se autoriza implementación ni cambios en Linear.

---

**Fin del documento.**
