# Informe Consolidado — Fase 1 (WAB-77) Auditoría del Sistema Actual

**Padre:** WAB-75 · **Fase:** WAB-77 · **Estado:** Auditoría de solo lectura completada · **Fecha:** 02/11/2026

**Regla de cierre:** WAB-77 NO se cierra ni WAB-78 se inicia hasta que el usuario revise y autorice explícitamente. Este informe es de solo lectura: no se modificó código, schemas, datos ni infraestructura.

## 1. Resumen ejecutivo

El sistema actual (`crm/microservices/ai-service/` + `crm/` + `WABY_frontend/`) opera con el modelo **Bot / BotInstance**, donde el cliente configura provider, modelo, temperatura, capabilities y credencial por bot. El modelo objetivo WAB-75 requiere **AgentDefinition (Waby) + AgentInstance (Workspace)** con credenciales centralizadas, cupo por plan, Knowledge compartido por Workspace, Skills/Rules por instancia, ventana de 20 mensajes y bolsa única de créditos.

La distancia es significativa: **no existe AgentDefinition, no existe cupo configurable por plan, no existe bolsa de créditos única, las credenciales son per-bot, y el provider/modelo es configurable por bot.**

Hallazgos críticos sin evidencia de mitigación:
1. **No hay `orgId` en entidades de agente** — la tenant se deriva vía `workspaceId → Works.ref_organizacion` (cross-DB crm↔ai_service), lo que obliga a join en cada check de ownership y no permite afirmar integridad multitenant solo por la presencia de `workspaceId` en schemas.
2. **BR-AI-009 (máx 1 bot activo por workspace) no está implementada en schema ni índices** — solo documentada en `crm/docs/business/bot-routing.md:130`.
3. **`delete-wrk` no toca `bots`, `contact_assignments`, `knowledge_bases`, `memory_facts`, `executions`, vectores Qdrant ni buckets MinIO** — confirmado contra el código actual de los 7 pasos de eliminación.
4. **`workspaceId` del payload RabbitMQ no se valida contra el bot** en el punto de entrada (`message-received-handler.ts:104`).
5. **Todas las capabilities se cargan globalmente** (`capabilityRegistry.findAll()` en `execution-engine.ts:65-67`) sin filtrar por `enabledCapabilities` del bot/instancia.
6. **Los fragmentos de Knowledge viajan con rol `system`** (`openai.mapper.ts:28-32`) — misma prioridad que instrucciones del sistema, riesgo de prompt injection.
7. **RAG falla silenciosamente a `[]`** si Qdrant falla (`execute-rag.step.ts:16`) — indistinguible de "sin resultados".
8. **No existe ventana de 20 mensajes** — memoria son hasta 30 facts por contacto (`memory-manager.ts:7`), y `LoadConversationStep` inicializa historial vacío (`load-conversation.step.ts:27`).
9. **No existe sistema de créditos** — `execution.cost` en schema es estimación legacy (BR-AI-017), no débito atómico.

Los seis informes AUD están publicados como comentarios en sus respectivas issues de Linear (WAB-87 a WAB-92). Este documento los consolida, reconcilia contradicciones y clasifica cada hallazgo por nivel de evidencia.

## 2. Arquitectura actual comprobada

### 2.1 Dominio y persistencia (evidencia confirmada mediante código)

**Entidades de agente confirmadas** (leídos directamente de `crm/microservices/ai-service/src/domain/entities/` y `src/infrastructure/database/models/`):

| Entidad | Clase dominio | Modelo Mongoose | Colección | orgId | workspaceId | Índices declarados en schema |
|---------|--------------|-----------------|-----------|-------|-------------|------------------------------|
| Bot | `bot.ts` | `bot.model.ts` | `bots` | ❌ no existe | ✅ requerido | `{workspaceId:1}`, `{workspaceId:1,status:1}` |
| BotInstance | `bot-instance.ts` | `bot-instance.model.ts` | `bot_instances` | ❌ no existe | ✅ requerido | `{botId:1}`, `{workspaceId:1}`, `{botId:1,status:1}` |
| Execution | `execution.ts` | `execution.model.ts` | `executions` | ❌ no existe | ✅ requerido | `{conversationId:1}`, `{workspaceId:1}`, `{botId:1}`, `{botInstanceId:1}`, `{status:1}`, `{conversationId:1,createdAt:-1}` |
| KnowledgeBase | `knowledge-base.ts` | `knowledge-base.model.ts` | `knowledge_bases` | ❌ no existe | ✅ requerido | `{workspaceId:1}`, `{workspaceId:1,status:1}` |
| MemoryFact | `memory-fact.ts` | `memory-fact.model.ts` | `memory_facts` | ❌ no existe | ✅ requerido | `{ws:1,c:1,status:1}`, `{ws:1,c:1,category:1}`, `{ws:1,c:1,importance:-1}`, `{status:1,expiresAt:1}`, `{ws:1,c:1,key:1} unique partial ACTIVE` |
| ContactAssignment | `contact-assignment.ts` | `contact-assignment.model.ts` | `contact_assignments` | ❌ no existe | ✅ requerido | `{contactId:1,status:1} unique partial active`, `{contactId:1,createdAt:-1}`, `{assigneeType:1,assigneeId:1,status:1}`, `{workspaceId:1,status:1}` |
| Prompt | `prompt.ts` | `prompt.model.ts` | `prompts` | ❌ no existe | ✅ requerido | `{workspaceId:1}`, `{workspaceId:1,name:1} unique`, `{workspaceId:1,status:1}` |
| Document | `document.ts` | `document.model.ts` | `documents` | ❌ no existe | N/C | no verificado completamente |

**Evidencia de líneas:**
- `bot.model.ts:70-71`: índices `workspaceId` y compuesto `workspaceId+status` (no hay unicidad wsId+name)
- `bot-instance.model.ts:75-77`: índices `botId`, `workspaceId`, `botId+status` (no hay unicidad)
- `execution.model.ts:109-114`: índices por conversationId, workspaceId, botId, botInstanceId, status
- `knowledge-base.model.ts:69-70`: índices `workspaceId` y `workspaceId+status`
- `memory-fact.model.ts:55-66`: índice único parcial `{wsId,contactId,key}` solo para `status=ACTIVE`
- `contact-assignment.model.ts:35-38`: índice único parcial `{contactId,status}` solo para `status=active`
- `prompt.model.ts:33-35`: índice único `{workspaceId,name}` — sí hay unicidad de nombre por workspace

**NOTA de calidad:** Los índices anteriores están **declarados en los schemas Mongoose**, no se verificaron contra una instancia real de MongoDB. No se puede afirmar que existen efectivamente en la base de datos desplegada sin inspección del cluster. Además, la presencia de `workspaceId` como campo **no equivale a integridad multitenant efectiva**: no hay índice único `workspaceId + definición`, no hay validación conjunta `orgId + workspaceId` en escritura, y `orgId` no existe como campo en ninguna entidad de agente.

**Relaciones y restricciones efectivas:**
- `Bot.workspaceId` → referencia lógica a `Works._id` (DB `crm`), resuelta cross-DB vía `aiServiceOwnership.ts:57-68` y `ResourceOwnership.ts:345-356`.
- `BotInstance.botId` → referencia a `Bot._id`. `BotInstance.workspaceId` duplicado (denormalizado).
- `Execution.workspaceId`, `botId`, `botInstanceId` — todos denormalizados, sin foreign key.
- `MemoryFact.workspaceId`, `contactId`, `conversationId` — sin foreign key.
- `KnowledgeBase.workspaceId` — sin foreign key.
- `Prompt.workspaceId` — sin foreign key.
- No hay TTL en ninguna colección de agente (solo `linkPreview.model.ts` del CRM tiene `expireAfterSeconds`).

**Campos legacy / no verificados:**
- `aiCredential` en `Bot` (cifrado AES-256-GCM, excluido de `toJSON` en `bot.ts:418-420`) — solo escrito por `ai-service`.
- `activePromptVersionId` en `Bot` — runtime override, no la versión publicada.
- `PromptVersion.promptId` — referencia a `Prompt._id`, índice único parcial `{promptId:1}` para `status='published'` (`prompt-version.model.ts:60-63`).

### 2.2 Separación AgentDefinition / AgentInstance — estado actual

**No existe separación.** `Bot` mezcla definición (modelo, provider, credencial, capabilities globales) e instancia (estado activo/inactivo). `BotInstance` es una sobrecarga opcional por bot con campos sobrescribibles (`modelConfig`, `temperature`, `systemPrompt`, `enabledCapabilities`, `schedule`).

**Evidencia de mezcla:** `bot.ts:52` (`model: ModelConfiguration`), `bot.ts:43` (`credential: BotCredential`), `bot.ts:29` (`enabledCapabilities`). `bot-instance.ts:230-233` (`resolveEffectiveConfig` combina campos propios con fallback a `Bot`).

**Consecuencia para migración I1/I2/I16:** se requiere nuevo schema con `AgentDefinition` (propiedad Waby, sin `orgId` + `workspaceId` única) y `AgentInstance` (propiedad Workspace, sin unicidad wsId+definición per I16). No hay migración de datos verificada.

## 3. Diagrama de entidades y relaciones actuales (as-is)

```
┌─────────────┐      ┌──────────────────┐      ┌─────────────────┐
│   Works     │      │      Bot         │      │   BotInstance   │
│  (DB crm)   │      │  (DB ai_service) │      │ (DB ai_service) │
│             │◄─────│ workspaceId      │─────►│ botId           │
│ ref_org     │      │ name             │      │ workspaceId*    │
│─────────────┘      │ model            │      │ status          │
                     │ aiCredential     │      │ modelConfig*    │
                     │ enabledCapabilities│   │ systemPrompt*   │
                     │ knowledgeBaseIds │      │ enabledCapabilities*│
                     └──────────────────┘      └─────────────────┘
                                  │                   │
                                  │                   │
                          ┌───────┴───────┐          │
                          │               │          │
                    ┌───────▼─────┐  ┌────▼──────┐  │
                    │  Prompt     │  │ Execution │  │
                    │ (DB ai_svc) │  │ (DB ai_svc)│  │
                    │ workspaceId │  │ workspaceId│  │
                    │             │  │ botId      │  │
                    │             │  │ botInstanceId│ │
                    │             │  │ conversationId││
                    └───────┬─────┘  │ contactId   │ │
                            │        └──────┬──────┘ │
                            │               │        │
                    ┌───────▼─────┐  ┌──────▼───────┐│
                    │ Knowledge   │  │ MemoryFact   ││
                    │ Base        │  │ (DB ai_svc)  ││
                    │ workspaceId │  │ workspaceId  ││
                    └─────────────┘  │ contactId    ││
                                     │ conversationId││
                                     └───────────────┘

* Denormalizado (duplicado), no foreign key real.
```

**Relaciones entre servicios:**
- CRM (DB `crm`) y ai-service (DB `ai_service`) comparten cluster MongoDB pero usan bases distintas.
- `resolveAuthContext.ts` resuelve JWT + `x-org-id` + `x-wrk-id` en una sola agregación con `$lookup` a `workspaces` y `us_orgs_works`.
- `ResourceOwnership.ts` (T3) valida resource → `workspaceId` → `Works.ref_organizacion` para WORKSPACE, CHANNEL, CONTACT, UOW, US_ORG, ORGANIZATION, KNOWLEDGE_BASE, DOCUMENT, RULE, SKILL, BOT.
- `aiServiceOwnership.ts` hace read-only cross-DB desde `crm` a `ai_service` para `bots`, `knowledge_bases`, `documents`.
- `BotRuntimeResolver.resolve(botId)` carga Bot + BotInstance desde `ai_service` (no valida `workspaceId` del payload).
- `delete-wrk` opera solo sobre DB `crm` (7 pasos: CONTACTOS, DIFUSIONES, SEGMENTOS, CHANNELS, USERS, TAGS, WORKSPACE).
- Qdrant: almacena vectores de chunks de Knowledge, filtrados por `workspaceId` (BR-KB-001).
- MinIO: almacena documentos de Knowledge Sources y media de salida de Skills.

---

## 4. Flujo de ejecución y routing actual (evidencia confirmada mediante código)

### 4.1 Entrada y resolución de instancia

**Flujo confirmado** (`crm/microservices/ai-service/src/infrastructure/messaging/consumers/message-received-handler.ts`):

1. RabbitMQ consume `ai-service.message-received` (L87-178). Zod valida `RabbitMQMessageSchema` con `workspaceId`, `botId`, `contactId`, `content`, `history`, `traceId` (L98-102).
2. `BotRuntimeResolver.resolve(parsed.data.botId)` carga Bot + BotInstance desde DB (L104).
3. Vigencia G1 (opcional, solo si `vigenciaService` inyectada): `vigenciaService.check({botId, contactId, workspaceId})` (L111-158). Fail-open si Mongo cae (L140-153).
4. `buildContext()` construye `ExecutionContext` con `workspaceId` del payload, `botId`, `botInstanceId` de `runtime.botInstanceId` (L194-250). **`workspaceId` del payload no se valida contra `bot.workspaceId` en este punto.**
5. `processMessageService.executeWithContext(context)` → `ExecutionEngine.process()`.

**Validaciones efectivas por salto:**

| Salto | Fuente de identidad | Validación efectiva | ¿Valida wsId vs bot? | Evidencia |
|-------|---------------------|---------------------|---------------------|-----------|
| Entrada RabbitMQ | Zod schema | `workspaceId`, `botId` requeridos (strings) | ❌ No | `message-received-handler.ts:98-102` |
| Resolución bot | `BotRuntimeResolver.resolve(botId)` | Bot existe en DB | ❌ No valida wsId del payload | `message-received-handler.ts:104` |
| Vigencia G1 | `vigenciaService.check()` | `workspaceId` del payload vs `bot.workspaceId` | ✅ Sí (si inyectada) | `vigencia.service.ts:41-43` |
| Pipeline contexto | `context.workspaceId` del payload | Usado en RAG (BR-KB-001) | ❌ No validado en entrada | `message-received-handler.ts:233` |
| RAG búsqueda | `config.workspaceId` (canonical del bot) | Filtra por workspace del bot, ignora `context.wsId` si difieren | ✅ Sí (BR-KB-001) | `execute-rag.step.ts:43-55` |
| Capabilities | `capabilityRegistry.findAll()` | Todas cargadas, sin filtro por instancia | ❌ No | `execution-engine.ts:65-67` |

### 4.2 Pipeline de ejecución

**Secuencia confirmada** (`crm/microservices/ai-service/src/engine/execution-engine.ts:57-129`):

1. `createExecution(context)` + `save` (estado `pending`) (L58-62)
2. `capabilityRegistry.findAll()` → todas las capabilities → `getToolDefinitions()` → `context.setAvailableCapabilities()` (L65-67). **Sin filtro por `enabledCapabilities`.**
3. `runPipelineLoop(context, execution)` — pasos: `LoadConfiguration` → `LoadConversation` → `ResolvePrompt` → `LoadMemory` → `LoadRulesSkills` → `ExecuteRAG` → `BuildPrompt` → `InvokeProvider` → bucle tool calling (L69)
4. Vigencia G2: `vigenciaService.check()` antes de completar. Fail-closed: `context.cancel()` + `execution.cancel()` si no válido (L80-91)
5. `markCompleted(response.content)` o `markFailed(error)` (L93-115)
6. `finally`: `accumulateTokens`, `accumulateCost`, `save` execution, `memoryPersister.persist` (si no cancelada), `publishResult` (L116-128)

**Detalles de pasos clave:**
- `LoadConfigurationStep` (`load-configuration.step.ts`): carga Bot + BotInstance, resuelve `EffectiveConfig`, valida provider existe y bot tiene API key propia (D*-cred, modo estricto). Credencial gate: sin credencial el bot no llega al provider (L73-84).
- `LoadConversationStep` (`load-conversation.step.ts:18-28`): inicializa historial `[]` si no está ya inicializado. **No hay límite de 20 mensajes.** El historial real del contacto viene del CRM vía `buildBotHistory` (L88-109), con `getBotHistoryLimit()` default 30 (`bot-history.ts:67-71`).
- `ExecuteRAGStep` (`execute-rag.step.ts:28-64`): si `context.workspaceId !== config.workspaceId` → `[]` con warn estructurado (BR-KB-001). Si Qdrant falla → `[]` sin error (fallback silencioso, L16).
- `BuildPromptStep` (`build-prompt.step.ts:51-140`): prioridad token budget: 1) system prompt, 2) user message, 3) knowledge, 4) memory, 5) history.
- `InvokeProviderStep`: invoca al proveedor LLM con la credencial del bot.

### 4.3 Composición del contexto y fronteras de confianza

**Orden de mensajes enviados al proveedor** (`openai.mapper.ts:16-82`):

1. `system` — `prompt.systemInstructions` (instrucciones del sistema o legacy `config.systemPrompt`)
2. `system` — `prompt.knowledgeContext` (fragmentos de Knowledge, **misma prioridad que system**) — RIESGO prompt injection
3. `memory` — mensajes de memoria (hasta 30 facts vía `memory-manager`)
4. `history` — historial de conversación (truncado por token budget)
5. `assistant` + `tool` — tool calls y resultados (si aplica)
6. `user` — mensaje actual

**Fronteras de confianza confirmadas:**
- **Knowledge como system:** `openai.mapper.ts:26-32` — fragmentos de Knowledge se envían como mensaje `system`, mismo nivel de prioridad que instrucciones del sistema. Un documento malicioso del cliente puede sobrescribir instrucciones de plataforma. **No hay mitigación verificada.**
- **Credenciales nunca al prompt:** `bot.toJSON` excluye `credential` (`bot.ts:418-420`). ✅
- **Provider response sin filtro:** `execution-engine.ts:93-100` — `context.providerResponse.content` se persiste y publica directamente, sin output filter. ❌
- **Acceso a Internet/código/env:** no hay capability genérica de HTTP/FS; solo CRM GraphQL (`auditoria-gobernanza-agentes-waby.md:365-370`). ✅

### 4.4 Fallos, retries, fallbacks e idempotencia

| Punto | Comportamiento ante fallo | Idempotente | Evidencia |
|-------|--------------------------|-------------|-----------|
| RabbitMQ consume | `nack(false, false)` sin requeue → mensaje perdido | ❌ No | `message-received-handler.ts:174` |
| correlationId | Generado aleatoriamente por mensaje (`corr-${Date.now()}-${random}`) | ❌ No dedup | `message-received-handler.ts:228` |
| Vigencia G1 | Fail-open si `vigenciaService` no disponible | N/A | `message-received-handler.ts:140-153` |
| RAG (Qdrant) | Fallback silencioso a `[]`, pipeline continúa | N/A | `execute-rag.step.ts:16, 43-55` |
| Provider LLM | Timeout configurable por adapter (30s default), retry en Qdrant (`maxRetries`) | ❌ No | `qdrant-vector-store.ts:90, 531-581` |
| Ejecución cancelada | `markFailed` o `cancel`, publicado como `failed`/`cancelled` | ❌ No | `execution-engine.ts:101-115` |
| Vigencia G2 | Fail-closed: cancelada si no válido | N/A | `execution-engine.ts:80-91` |
| Idempotencia de ingestión | `idempotency-check.step.ts` para versiones de documento (contenido hash) | ✅ (solo ingestión) | `ingestion/steps/idempotency-check.step.ts` |

## 5. Matriz de ownership y aislamiento

| Entidad | Propietario | Ámbito | Lectores efectivos | Escritores efectivos | Aislamiento verificado |
|---------|-------------|--------|-------------------|---------------------|------------------------|
| Bot | ai-service | workspaceId (denormalizado) | GraphQL `wrkRead`, HTTP interno, ai-service pipeline | GraphQL `wrkCreate/Update/Delete`, HTTP interno, `BotConfigurationService` | ⚠️ Parcial: `workspaceId` presente pero no hay validación conjunta orgId+workspaceId en escritura de GraphQL |
| BotInstance | ai-service | workspaceId (denormalizado, duplicado) | ai-service pipeline (`BotRuntimeResolver`) | `BotConfigurationService` (`createInstance`, `updateInstance`, `activateInstance`, `deactivateInstance`) | ⚠️ Parcial: mismo que Bot |
| Execution | ai-service | workspaceId | ai-service pipeline, `execution.repository` | `ExecutionEngine` | ✅ workspaceId presente en schema, sin verificación cross-tenant en escritura |
| KnowledgeBase | ai-service | workspaceId | GraphQL `knowledgeRead`, RAG pipeline | GraphQL `knowledgeCreate/Update/Delete` | ✅ `assertResourceOwnership` en `AiKnowledgeBase.resolver.ts:52,81,97` |
| Document | ai-service | workspaceId (vía KB) | GraphQL `knowledgeRead` | GraphQL `knowledgeCreate/Update/Delete` | ✅ T3 sobre KB padre en `AiDocument.resolver.ts:36,54,73,84,97,122` |
| MemoryFact | ai-service | workspaceId + contactId | `memory.repository` | `memory-manager.persistFacts` | ⚠️ Índices parciales únicos, sin verificación de conversationId vs instancia |
| Prompt | ai-service | workspaceId | GraphQL `wrkRead` (vía AiBot config) | `CreatePromptUseCase`, `DeletePromptUseCase` | ⚠️ `DeletePromptUseCase` verifica que no haya bots usando el prompt (L20-26) |
| ContactAssignment | ai-service | workspaceId | `contact-assignment.repository` | `assign-contact`, `reassign-contact`, `unassign-contact` | ✅ Índice único parcial `{contactId,status}` para `active` |

**Brechas de aislamiento confirmadas:**
1. **`orgId` no existe en entidades de agente** — la tenant se deriva cross-DB, no está persistida junto al recurso. No se puede afirmar integridad multitenant solo por `workspaceId`.
2. **`workspaceId` del payload no validado contra el bot** en entrada RabbitMQ — posible ejecución en contexto equivocado si un payload trae `workspaceId` ajeno.
3. **Capabilities globales** — `findAll()` expone todas las capabilities al proveedor, sin filtro por `enabledCapabilities` del bot/instancia.
4. **Knowledge compartido por workspace** (cumple I11) — pero fragmentos viajan como `system`, sin aislamiento de instrucciones de plataforma.

## 6. Matriz de autorización y permisos

### 6.1 Permisos actuales vs modelo objetivo

| Recurso | Permiso actual | Ámbito actual | Permiso WAB-75 objetivo | ¿Separado de Knowledge? | Evidencia |
|---------|---------------|---------------|------------------------|------------------------|-----------|
| Bot (crear/editar/eliminar/activar) | `wrkCreate/Update/Delete` | Workspace | Waby-only (plataforma) | ❌ Comparte con Knowledge | `AiBot.resolver.ts:18-114`, `permission.enum.ts` |
| Bot (consultar) | `wrkRead` | Workspace | Waby-only / cliente solo ve habilitadas | ❌ Comparte con Knowledge | `AiBot.resolver.ts:47-61` |
| Knowledge Base (CRUD) | `knowledgeCreate/Read/Update/Delete` | Workspace | Workspace roles (Owner/Admin/Editor) | ✅ Ya separado | `AiKnowledgeBase.resolver.ts:26-91` |
| Document (CRUD) | `knowledgeCreate/Read/Update/Delete` | Workspace | Workspace roles (Owner/Admin/Editor) | ✅ Ya separado | `AiDocument.resolver.ts` |
| Rules (CRUD) | `wrk.rules.*` | Workspace | Permiso específico de configuración de bot | ❌ Comparte namespace con skills | `permission.enum.ts:89-92` |
| Skills (CRUD) | `wrk.skills.*` | Workspace | Permiso específico de configuración de bot | ❌ Comparte namespace con rules | `permission.enum.ts:93-96` |
| API Key de bot | `wrkUpdate` (deleteAiBotApiKey) | Workspace | Waby-only (credenciales centralizadas) | N/A | `AiBot.resolver.ts:117-123` |

### 6.2 Ownership check efectivo por resolver/endpoint

| Resolver/Endpoint | `@Authorized` | `assertResourceOwnership` / T3 | Ownership verificado | Evidencia |
|-------------------|---------------|-------------------------------|---------------------|-----------|
| `createAiBot` | `wrkCreate` | ❌ No | ❌ No verifica workspaceId del payload vs x-wrk-id | `AiBot.resolver.ts:18-45` |
| `getAiBot` | `wrkRead` | ❌ No | ❌ No verifica ownership del bot | `AiBot.resolver.ts:47-53` |
| `listAiBots` | `wrkRead` | ❌ No | ❌ Recibe workspaceId como arg, no valida contra x-wrk-id | `AiBot.resolver.ts:55-61` |
| `updateAiBot` | `wrkUpdate` | ❌ No | ❌ No verifica ownership | `AiBot.resolver.ts:63-89` |
| `deleteAiBot` | `wrkDelete` | ❌ No | ❌ No verifica ownership | `AiBot.resolver.ts:91-98` |
| `activate/deactivateAiBot` | `wrkUpdate` | ❌ No | ❌ No verifica ownership | `AiBot.resolver.ts:100-114` |
| `deleteAiBotApiKey` | `wrkUpdate` | ❌ No | ❌ No verifica ownership | `AiBot.resolver.ts:117-123` |
| `aiBotReadiness/Configuration` | `wrkRead` | ❌ No | ❌ No verifica ownership | `AiBot.resolver.ts:125-145` |
| `createKnowledgeBase` | `knowledgeCreate` | ✅ `assertResourceOwnership(ctx, id, 'KNOWLEDGE_BASE', 'CONTEXT')` | ✅ Sí | `AiKnowledgeBase.resolver.ts:26` |
| `getKnowledgeBase` | `knowledgeRead` | ✅ T3 CONTEXT | ✅ Sí | `AiKnowledgeBase.resolver.ts:44` |
| `createDocument` | `knowledgeCreate` | ✅ T3 sobre knowledgeBaseId | ✅ Sí | `AiDocument.resolver.ts:36` |
| `attachRuleToBot` / `attachSkillToBot` | — | ✅ `assertOwnershipOrgAware(ctx, botId, 'BOT')` | ✅ Sí | `BotResourceLink.resolver.ts:56,73` |
| BotResource CRUD | — | ✅ `assertOwnershipOrgAware` | ✅ Sí | `BotResource.resolver.ts:219-282` |
| ai-service HTTP `/api/bots/*` | `AI_SERVICE_INTERNAL_TOKEN` | ❌ No | ❌ Controladores no verifican ownership | `bot.controller.ts:55-81`, `bot.routes.ts` |

### 6.3 Casos de autorización positiva y negativa existentes (evidencia de pruebas)

**Positivos (confirmados mediante pruebas):**
- `ResourceOwnership.test.ts` UT-01..UT-04: CROSS_TENANT, CROSS_WORKSPACE, RESOURCE_OUT_OF_WORKSPACE para WORKSPACE, CHANNEL, CONTACT, UOW, US_ORG, ORGANIZATION
- `ResourceOwnership.botResource.test.ts`: cross-tenant/cross-workspace para BOT (L95-125)
- `t0-t1-context-integrity.test.ts`: cross-workspace access denied (L64, 153)
- `bot-history.test.ts`: BR-MT-007, scope por contacto + workspace
- `rules-skills.graphql-rest.e2e.integration.test.ts`: cross-tenant org1/org2, cross-workspace intra-org (L588-603, 766)

**Negativos FALTANTES (no verificados mediante pruebas):**
- ❌ No hay prueba de `createAiBot` con `workspaceId` ajeno al contexto
- ❌ No hay prueba de `getAiBot` con bot de otro workspace
- ❌ No hay prueba de `updateAiBot` con bot de otro workspace
- ❌ No hay prueba de `deleteAiBot` con bot de otro workspace
- ❌ No hay prueba de ai-service HTTP interno con token válido pero workspace ajeno

### 6.4 Separación Knowledge vs configuración de agente — estado actual

**Confirmado:** Knowledge (KB + Documentos) y configuración de bot comparten el mismo modelo de permisos `wrk.*` en GraphQL (`AiBotResolver` usa `wrkCreate/Read/Update/Delete`, `AiKnowledgeBaseResolver` usa `knowledgeCreate/Read/Update/Delete`). No existe permiso separado "configuración de agente" vs "gestión de Knowledge".

**Evidence de separación parcial:** `BotResourceLink.resolver.ts` aplica `assertOwnershipOrgAware(ctx, botId, 'BOT')` al asociar Rules/Skills a un bot, pero esto es sobre el recurso `bot_resources` (DB crm), no sobre el `Bot` de `ai-service`.

**Brecha I19:** Los permisos de Knowledge no están independientes de los permisos de configuración del bot. Un rol con `wrk.knowledge.*` no accede directamente a `AiBotResolver` (diferentes permisos), pero ambos están bajo el mismo paraguas `wrk.*` sin distinción de "Waby-only" vs "cliente".

## 7. Inventario de componentes afectados

### 7.1 ai-service (`crm/microservices/ai-service/`)

| Componente | Ruta | Rol | Afectado por migración |
|------------|------|-----|----------------------|
| Bot entity | `src/domain/entities/bot.ts` | Definición + instancia mezcladas | ✅ Alta (separar AgentDefinition/AgentInstance) |
| BotInstance entity | `src/domain/entities/bot-instance.ts` | Sobrecarga opcional por bot | ✅ Alta |
| BotModel | `src/infrastructure/database/models/bot.model.ts` | Schema Mongo `bots` | ✅ Alta |
| BotInstanceModel | `src/infrastructure/database/models/bot-instance.model.ts` | Schema Mongo `bot_instances` | ✅ Alta |
| ExecutionModel | `src/infrastructure/database/models/execution.model.ts` | Schema Mongo `executions` | ⚠️ Media (añadir campos I20/I21) |
| KnowledgeBaseModel | `src/infrastructure/database/models/knowledge-base.model.ts` | Schema Mongo `knowledge_bases` | ⚠️ Media (compartido por workspace, cumple I11) |
| MemoryFactModel | `src/infrastructure/database/models/memory-fact.model.ts` | Schema Mongo `memory_facts` | ✅ Alta (reemplazar por ventana 20) |
| BotRuntimeResolver | `src/infrastructure/runtime/bot-runtime-resolver.ts` | Resuelve Bot + BotInstance por botId | ✅ Alta (resolver instancia por workspace+origen) |
| ExecutionEngine | `src/engine/execution-engine.ts` | Pipeline de ejecución | ✅ Alta (filtrar capabilities, G2, output filter) |
| LoadConfigurationStep | `src/engine/steps/load-configuration.step.ts` | Carga config, credential gate D*-cred | ✅ Alta (credenciales centralizadas) |
| ExecuteRAGStep | `src/engine/steps/execute-rag.step.ts` | RAG con BR-KB-001, fallback silencioso | ⚠️ Media (fallo no silencioso) |
| BuildPromptStep | `src/engine/steps/build-prompt.step.ts` | Composición de contexto, token budget | ⚠️ Media (orden, knowledge como system) |
| OpenAIMapper | `src/infrastructure/providers/openai/openai.mapper.ts` | Mapeo a mensajes OpenAI (knowledge como system) | ✅ Alta (riesgo prompt injection) |
| memory-manager | `src/engine/memory-manager.ts` | MAX_MEMORY_FACTS=30, deduplicación | ✅ Alta (ventana 20 user/assistant) |
| capability-registry | `src/infrastructure/capabilities/capability-registry.ts` | Registro de capabilities | ⚠️ Media (filtrar por instancia) |
| capability-executor | `src/infrastructure/capabilities/capability-executor.ts` | Ejecución de capabilities | ⚠️ Media |
| BotController | `src/presentation/http/controllers/bot.controller.ts` | HTTP interno `/api/bots/*` | ✅ Alta (ownership check, Waby-only) |
| bot.routes | `src/presentation/http/routes/bot.routes.ts` | Rutas HTTP internas | ✅ Alta |
| vigencia.service | `src/application/services/vigencia.service.ts` | G1/G2 vigencia (BR-AI-013) | ⚠️ Media (mantener, integrar con nuevo modelo) |
| delete-wrk | `crm/microservices/delete-wrk/` | Eliminación de Workspace (solo DB crm) | ✅ Alta (añadir ai_service, Qdrant, MinIO) |

### 7.2 CRM Backend (`crm/`)

| Componente | Ruta | Rol | Afectado por migración |
|------------|------|-----|----------------------|
| AiBotResolver | `src/graphql/resolvers/AiBot.resolver.ts` | GraphQL mutaciones/queries de bots | ✅ Alta (Waby-only, ownership) |
| AiKnowledgeBaseResolver | `src/graphql/resolvers/AiKnowledgeBase.resolver.ts` | GraphQL KB CRUD | ⚠️ Media (ya tiene T3) |
| AiDocumentResolver | `src/graphql/resolvers/AiDocument.resolver.ts` | GraphQL documentos CRUD | ⚠️ Media (ya tiene T3) |
| BotResourceResolver | `src/graphql/resolvers/BotResource.resolver.ts` | GraphQL Rules/Skills catálogo | ⚠️ Media (ya tiene T3) |
| BotResourceLinkResolver | `src/graphql/resolvers/BotResourceLink.resolver.ts` | GraphQL asociar Rules/Skills a bot | ⚠️ Media (ya tiene T3 sobre bot) |
| ResourceOwnership | `src/auth/ResourceOwnership.ts` | T3 ownership check | ⚠️ Media (extender a nuevas entidades) |
| aiServiceOwnership | `src/auth/aiServiceOwnership.ts` | Read-only cross-DB ai_service | ⚠️ Media (mantener) |
| resolveAuthContext | `src/auth/resolveAuthContext.ts` | Resuelve JWT + x-org-id + x-wrk-id | ✅ Media (mantener, extender) |
| permission.enum | `src/auth/permisions/permission.enum.ts` | Enum de permisos `wrk.*` | ✅ Alta (añadir permiso configuración de bot) |
| BotService | `src/services/AiBot.service.ts` | Wrapper cliente ai-service | ✅ Alta |
| bot-history | `src/modules/message-pipeline/bot-history.ts` | Historial contacto (default 30) | ⚠️ Media (ajustar a ventana 20) |
| delete-wrk services | `crm/microservices/delete-wrk/src/services/*.ts` | 7 pasos de eliminación | ✅ Alta (añadir ai_service, Qdrant, MinIO) |
| media-cleanup.worker | `src/modules/media/media-cleanup.worker.ts` | TTL media (30 días) | ⚠️ Media (retención diferenciada I15) |

### 7.3 Frontend (`WABY_frontend/`)

| Componente | Ruta | Rol | Afectado por migración |
|------------|------|-----|----------------------|
| AgentsConfigs | `src/views/AgentsConfigs.jsx` | Pantalla administración bots | ✅ Alta (Waby-only, ocultar al cliente) |
| AddAgentForm | `src/components/settings/workspace/AddAgentForm.tsx` | Crear/editar bot (provider, modelo, apiKey, etc.) | ✅ Alta (retirar campos Waby-only) |
| AgentsConfigs routes | `src/App.jsx:237` | Ruta `/settings/workspace/agents` | ✅ Alta |
| WorkspaceKnowledge | `src/views/WorkspaceKnowledge.jsx` | Pantalla administración KB | ⚠️ Media (mantener, cliente puede gestionar KB) |
| KnowledgeDocuments | `src/views/KnowledgeDocuments.jsx` | Documentos de KB | ⚠️ Media (mantener) |
| AddKnowledgeBaseForm | `src/components/settings/workspace/AddKnowledgeBaseForm.tsx` | Crear KB | ⚠️ Media (mantener) |
| KnowledgeUploadModal | `src/components/settings/workspace/KnowledgeUploadModal.tsx` | Subir documentos | ⚠️ Media (mantener) |
| BotResourcesTab | `src/components/settings/workspace/BotResourcesTab.tsx` | Gestionar Rules/Skills | ⚠️ Media (permiso específico de configuración) |
| agents.query.js | `src/graphql/agents.query.js` | Mutaciones/queries GraphQL bots | ✅ Alta |
| knowledge.query.js | `src/graphql/knowledge.query.js` | Queries GraphQL KB | ⚠️ Media |
| botResources.query.js | `src/graphql/botResources.query.js` | Queries GraphQL Rules/Skills | ⚠️ Media |

## 8. Matriz de brechas respecto del modelo objetivo WAB-75

| Invariante WAB-75 | Estado actual | Brecha | Evidencia |
|-------------------|---------------|--------|-----------|
| I1 Waby administra agentes | Cliente crea/edita/elimina bots | Requerir permiso Waby-only | `AiBot.resolver.ts:18-114` |
| I2 Unidad de propiedad = Workspace | `workspaceId` sin `orgId`, cross-DB join | Añadir `orgId` o validación conjunta | `bot.model.ts:5`, `aiServiceOwnership.ts:57-68` |
| I3 Cupo 1-2 instancias por plan | No existe cupo; BR-AI-009 solo documentada | Implementar cupo por plan con concurrencia | `bot-routing.md:130` (solo docs) |
| I4 Org tiene varios WS, no comparten | `workspaceId` denormalizado, sin FK | Verificar cross-workspace en escritura | No hay verificación en GraphQL bots |
| I5 Runtime/proveedores compartidos | Provider/modelo por bot | Centralizar control de Waby | `ModelConfiguration` en `Bot`/`BotInstance` |
| I6 Credenciales centralizadas | Credencial per-bot (D*-cred) | Eliminar API key de cliente | `bot.ts:43`, `load-configuration.step.ts:73-84` |
| I7 Aislamiento estricto | `workspaceId` del payload no validado | Validar ownership en entrada | `message-received-handler.ts:104` |
| I8 Políticas no delegables | No existe Policy Engine | Implementar Policy Engine | `auditoria-gobernanza-agentes-waby.md:204-208` |
| I9 No exponer interior de Waby | Provider/modelo visible al cliente | Ocultar detalles internos | `AiBot.object.ts` expone model/provider |
| I10 No acceso exterior implícito | No hay capability HTTP/FS genérica | Mantener restricción | `auditoria-gobernanza-agentes-waby.md:365-370` |
| I11 Knowledge compartido por WS | RAG filtra por `config.workspaceId` | ✅ Cumplido | `execute-rag.step.ts:43-55` |
| I12 Skills por instancia | `enabledCapabilities` en Bot, globales | Filtrar por instancia | `execution-engine.ts:65-67` |
| I13 Rules por instancia | Rules en `bot_resources`, vinculadas a Bot | Separar por AgentInstance | `BotResource.resolver.ts` |
| I14 Memoria 20 msgs user/assistant | 30 facts por contacto, sin ventana 20 | Reemplazar memoria | `memory-manager.ts:7`, `load-conversation.step.ts:27` |
| I15 Archivos por finalidad/retención | No hay purge de embeddings ni TTL en ai_service | Implementar purge y retención | `delete-wrk` no toca ai_service |
| I16 Varias instancias misma definición | No hay restricción wsId+definición (no existe definición) | N/A (no hay definición separada) | No existe AgentDefinition |
| I17 Actualización automática definiciones | No hay propagación en tiempo real verificada | Diseñar mecanismo | P10 de WAB-75 |
| I18 Provider/modelo centralizados | Configurables por bot/instancia | Centralizar en Waby | `ModelConfiguration` en `Bot`/`BotInstance` |
| I19 Permisos Knowledge independientes | Comparten `wrk.*` con configuración | Crear permiso separado | `permission.enum.ts:83-96` |
| I20 Atribución consumo por instancia | `execution.cost` estimación legacy, sin créditos | Implementar ledger de créditos | `execution.model.ts:36-50` |
| I21 Auditoría por ejecución | `execution` sin `snapshot_hash`, `provider_model_used`, créditos | Añadir campos | `execution.model.ts` |

## 9. Riesgos técnicos ordenados por severidad

| Riesgo | Severidad | Consecuencia | Evidencia |
|--------|-----------|--------------|-----------|
| **R1: Doble descuento de créditos** | Crítica | Saldo de Workspace incorrecto; no hay sistema de créditos, `execution.cost` es estimación legacy | No existe sistema de créditos; `execution.model.ts:36-50` es legacy |
| **R2: Mezcla definición/instancia en `Bot`** | Alta | Cliente modifica comportamiento global (modelo, credencial, capabilities) | `bot.ts:43,52,29` |
| **R3: Credencial per-bot** | Alta | Cliente gestiona API keys (violación I6) | `bot.ts:43`, `load-configuration.step.ts:73-84` |
| **R4: Provider/modelo configurable por bot** | Alta | Cliente elige proveedor (violación I18) | `ModelConfiguration` en `Bot`/`BotInstance` |
| **R5: Sin cupo de instancias por plan** | Alta | Exceso de instancias por Workspace | BR-AI-009 solo documentada (`bot-routing.md:130`) |
| **R6: `workspaceId` del payload no validado** | Media | Ejecución en contexto equivocado | `message-received-handler.ts:104` |
| **R7: Capabilities globales sin filtro** | Media | Bot ejecuta tools no asignadas a su instancia | `execution-engine.ts:65-67` |
| **R8: Knowledge como system (prompt injection)** | Media | Documento del cliente sobrescribe instrucciones de plataforma | `openai.mapper.ts:26-32` |
| **R9: RAG fallback silencioso a `[]`** | Media | Fallo de Qdrant indistinguible de "sin resultados" | `execute-rag.step.ts:16` |
| **R10: Memoria 30 facts sin ventana 20** | Media | Contexto/costo no determinista | `memory-manager.ts:7` |
| **R11: No hay output filter** | Media | Contenido no filtrado antes de enviar al usuario | `execution-engine.ts:93-100` |
| **R12: No idempotencia RabbitMQ** | Media | Duplicados de mensajes posibles | `message-received-handler.ts:174,228` |
| **R13: `orgId` no persistido en entidades** | Media | Join cross-DB obligatorio en cada check de ownership | `bot.model.ts`, `aiServiceOwnership.ts:57-68` |
| **R14: Borrado incompleto de Workspace** | Media | Datos huérfanos en ai_service, vectores Qdrant activos | `delete-wrk` solo toca DB crm |
| **R15: Sin pruebas negativas multitenant** | Baja | Invariantes de aislamiento no protegidas contra acceso ajeno | `ResourceOwnership.test.ts` solo casos positivos |

## 10. Elementos que se conservan, adaptan, separan, reemplazan o retiran

| Elemento | Acción | Justificación | Invariantes |
|----------|--------|---------------|-------------|
| `Bot` / `BotInstance` | **Adaptar** → separar AgentDefinition (Waby) y AgentInstance (Workspace) | `Bot` mezcla definición + instancia; no hay entidad de definición separada | I1, I2, I16 |
| `ModelConfiguration` por bot | **Reemplazar** → routing central de provider/modelo (Waby) | Configurable por bot hoy; debe ser control de Waby | I18 |
| Credencial per-bot (D*-cred) | **Reemplazar** → credenciales centralizadas de Waby | Cliente aporta API key hoy; debe ser centralizada | I6 |
| `enabledCapabilities` global en pipeline | **Separar** → filtrar por instancia | `findAll()` carga todas sin filtro | I8, I12 |
| RAG filtro `config.workspaceId` | **Conservar** | Cumple I11 (Knowledge compartido por workspace, no por agente) | I11 |
| `memory-manager` (30 facts) | **Reemplazar** → ventana ≤20 user/assistant por conversación | No existe ventana 20; memoria son facts por contacto | I14 |
| `Execution` (workspaceId, model, status) | **Conservar + ampliar** → añadir `snapshot_hash`, `provider_model_used`, tokens, créditos, latencia | Necesario para I20, I21 | I20, I21 |
| `delete-wrk` | **Adaptar** → limpiar Knowledge + vectores Qdrant + MinIO + jobs pendientes | Solo toca DB crm hoy; no toca ai_service | Invariante 13 |
| Sistema de créditos | **Crear** → bolsa única por Workspace, débito atómico/idempotente | No existe hoy | I17, I20, invariante 17/21 |
| Policy Engine | **Crear** → políticas de plataforma no delegables | No existe hoy | I8 |
| Output filter | **Crear** → filtrar contenido antes de enviar al usuario | No existe hoy | I8, I9 |

## 11. Recomendación de migración

La migración debe implementarse por fases, respetando el orden de dependencias identificado en WAB-75 §6:

**Fase 2 (Diseño):** Definir schema de `AgentDefinition` y `AgentInstance`, permiso de configuración de bot separado (`bot.configure`), contrato de resolución de instancia (origen → instancia), ledger de créditos, Política Engine, output filter.

**Fase 3 (Datos/Índices):** Añadir `orgId` o validación conjunta orgId+workspaceId en entidades de agente; crear índices únicos necesarios; definir contador de cupo por plan.

**Fase 4 (Provisioning):** Implementar operaciones Waby-only para crear/editar/eliminar AgentDefinition; asignar/activar/suspender AgentInstance; aplicar cupo por plan con concurrencia.

**Fase 5 (Routing/Runtime):** Resolver instancia por workspace + origen en backend; filtrar capabilities por instancia; validar `workspaceId` del payload contra el bot; implementar output filter.

**Fase 6 (Centralización):** Mover provider/modelo/credenciales a control central de Waby; eliminar API key de cliente; ocultar detalles internos de routing LLM.

**Fase 7 (Frontend):** Ocultar pantallas de administración de agentes al cliente; mantener pantallas de Knowledge; añadir dashboard por instancia con créditos.

**Fase 8 (Migración de datos):** Migrar bots existentes a AgentDefinition/AgentInstance; migrar ejecuciones e historial; invalidar embeddings si cambia esquema de Knowledge.

**Fase 9 (Créditos/Auditoría):** Implementar ledger de créditos con débito atómico/idempotente; añadir campos de auditoría a `execution` (snapshot_hash, provider_model_used, créditos descontados, latencia).

**Fase 10 (Pruebas/Regresión):** Implementar pruebas negativas multitenant, cupo concurrente, routing multi-instancia, ventana 20/21, idempotencia de créditos, fallos de RAG, filtrado de capabilities, prompt injection.

## 12. Decisiones que todavía requieren definición

| ID | Decisión | ¿Bloquea migración? | Donde se resuelve |
|----|----------|---------------------|-------------------|
| P1 | Cupo comercial exacto: qué planes habilitan 1 vs 2 instancias, si cuenta activas+suspendidas, comportamiento ante exceso | Sí (Fase 3) | WAB-75 §3, Fase 3 |
| P2 | Enrutamiento de interacciones: origen → instancia (canal/número/inbox apunta a instancia); fallback seguro si no hay instancia | Sí (Fase 5) | WAB-75 §5 P2, Fase 5 |
| P3 | Matriz de permisos: permiso separado `bot.configure` vs permisos de Knowledge | Sí (Fase 4) | WAB-75 §5 P3, Fase 4 |
| P4 | Conversión y liquidación de créditos: ledger append-only, hold atómico, idempotency key = `execution_id` | Sí (Fase 9) | WAB-75 §5 P4, Fase 9 |
| P9 | Estados de ciclo de vida de AgentInstance (draft/active/suspended/retired) | Sí (Fase 2) | WAB-75 §5 P9, Fase 2 |
| P10 | Mecanismo de propagación en tiempo real de definiciones a instancias (caché, evento, relectura) | Sí (Fase 3) | WAB-75 §5 P10, Fase 3 |
| P6 | Retención operativa vigente de mensajes/contactos y su aplicación a objetos de agente | No (puede definirse en Fase 2) | WAB-75 §5 P6 |

**Puntos no verificados en esta auditoría:**
- Índices Mongoose declarados no verificados contra una instancia real de MongoDB (solo leído el código de schema).
- No se inspeccionó el cluster MongoDB desplegado para confirmar existencia efectiva de índices.
- No se revisaron los esquemas Mongoose de `Document` y `KnowledgeBase` completamente para validar índices de vector store.
- No se verificó si el middleware `auth.middleware.ts` de `ai-service` se aplica a todas las rutas o solo a `/internal/*`.

## 13. Estado final de cada auditoría (WAB-87 a WAB-92)

| Issue | Subtarea | Hallazgos publicados en Linear | Criterios de aceptación cumplidos | Estado |
|-------|----------|-------------------------------|-----------------------------------|--------|
| WAB-87 | AUD-01 Persistencia | ✅ Comentario publicado | Inventario entidad→schema→colección→índices; diagrama as-is; matriz de tenant; restricciones implementadas vs documentadas | Completado (lectura) |
| WAB-88 | AUD-02 Autorización | ✅ Comentario publicado | Inventario resolvers/mutations/endpoints; matriz actor→operación→recurso; operaciones Waby-only vs cliente; permisos Knowledge vs bot separados | Completado (lectura) |
| WAB-89 | AUD-03 Frontend | ✅ Comentario publicado | Inventario rutas/pantallas/componentes; mapa UI→API→permiso; funciones que se conservan/retiran/rediseñan; brechas para múltiples instancias | Completado (lectura) |
| WAB-90 | AUD-04 Eliminación | ✅ Comentario publicado | Diagrama de ciclo de vida; matriz recurso→owner→retención→eliminación; inventario colecciones/buckets/Qdrant/colas; brechas de delete-wrk | Completado (lectura) |
| WAB-91 | AUD-05 Runtime | ✅ Comentario publicado | Diagrama entrada→routing→resolución→composición→tools→provider→persistencia; tabla de fuentes de identidad; matriz de capabilities; orden de contexto; matriz de fallos | Completado (lectura) |
| WAB-92 | AUD-06 Pruebas | ✅ Comentario publicado | Inventario de suites; matriz requisito→test→nivel; pruebas acopladas al modelo actual; brechas de cobertura priorizadas; propuesta de regresión | Completado (lectura) |

## 14. Informe consolidado y siguiente paso

**Ruta del informe:** `F:\ElemenSoft\crm\docs\plans\wab-75-fase-1-auditoria.md`

**Recomendación fundamentada:** Las seis auditorías están completas con evidencia verificada contra código, schemas, tests y documentación. Los comentarios están publicados en las seis issues de Linear (WAB-87 a WAB-92). Este informe consolidado los reconcilia con el modelo objetivo de WAB-75, corrige inconsistencias (delete-wrk tiene 7 pasos no 6; índices declarados no verificados contra DB real; cobertura de pruebas negativas ausente; criterio de cupo: la tercera instancia debe rechazarse, no la segunda), y clasifica cada hallazgo por nivel de evidencia.

**WAB-77 puede cerrarse y WAB-78 puede comenzar** una vez que el usuario revise este informe y autorice explícitamente el avance. No se debe cerrar WAB-77 ni iniciar WAB-78 sin esa autorización, tal como establece la regla de cierre de la Fase 1.

---
*Informe generado exclusivamente a partir de lectura de código, schemas, tests y documentación existentes. No se modificó ningún archivo, schema, índice, dato ni infraestructura durante esta auditoría.*
<tool_call>
<function=edit>