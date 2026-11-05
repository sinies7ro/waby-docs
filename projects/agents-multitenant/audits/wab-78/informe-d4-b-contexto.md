# WAB-78 — Informe D4-B: Auditoría de Construcción del Contexto

**Fecha:** 05/11/2026 · **Bloque:** D4-B (Construcción del contexto) · **Modo:** Solo lectura y diagnóstico
**Repositorio:** `F:\ElemenSoft\crm` · **Commit HEAD auditado:** `b9e0aee594bf7c4ffd1e88c84988b99e54b7d972` (origin/master) — estado limpio, sin cambios pendientes.
**Carpeta de informe:** `F:\ElemenSoft\docs\revision-wab-78\informe-d4-b-contexto.md`

---

## 1. Resumen ejecutivo

Esta auditoría determina cómo se construye el contexto efectivo que se entrega al proveedor LLM durante una ejecución, desde la resolución del prompt y la carga de recursos hasta la composición final de `PromptDocument`.

**Conclusión principal:** El flujo de construcción de contexto está implementado en ai-service como un pipeline de 8 pasos (`LoadConfiguration` → `LoadConversation` → `ResolvePrompt` → `LoadMemory` → `LoadRulesSkills` → `ExecuteRAG` → `BuildPrompt` → `InvokeProvider`). `BuildPromptStep` compone el `PromptDocument` final con `systemInstructions` (prompt renderizado o `config.systemPrompt` legacy), `rulesContext`, `skillsContext`, `memory`, `knowledgeContext`, `history`, `userMessage`, `toolCalls` y `toolResults`. Existe un presupuesto de tokens con prioridad (1. system prompt, 2. mensaje de usuario, 3. knowledge, 4. memoria, 5. historial).

**No existe separación AgentDefinition (Waby) / AgentInstance (Workspace):** el runtime usa `Bot` / `BotInstance` y no hay `AgentDefinition`, `AgentInstance`, `SkillConfiguration` ni `AgentRules` implementados. No hay `effectiveConfigSnapshot`, `effectiveConfigHash` ni `promptSnapshotHash` persistidos; P10 (snapshot inmutable por ejecución) no está implementado, solo definido como diseño.

Se identifican hallazgos provisionales D4B-C01 a D4B-C12. Ninguno duplica ARQ-01 a ARQ-22. Varios dependen de decisiones pendientes (P2, P3, propiedad de persistencia, versión inmutable de Skills) que no pueden cerrarse sin evidencia adicional.

---

## 2. Alcance y exclusiones

### 2.1 Alcance (D4-B)
- Resolución de `PromptVersion` y selección de versión activa/publicada.
- Relación entre Bot, BotInstance, system prompt y PromptVersion; overrides y precedencias.
- Construcción y transformación del prompt antes de enviarlo al proveedor.
- Variables, interpolaciones, placeholders y configuración dinámica.
- Persistencia, versionado y trazabilidad del prompt realmente utilizado.
- Origen y persistencia de Rules; modelos, schemas y entidades.
- Carga de Rules durante la ejecución; relación con Bot/Workspace/Organization; activación, desactivación, orden, combinación; transformación al formato incorporado al prompt.
- Versionado, revisiones y consistencia de Rules durante una ejecución.
- Origen y persistencia de Skills; `SkillConfiguration`; relación Skill ↔ AgentDefinition ↔ AgentInstance.
- Carga y habilitación de Skills; transformación en instrucciones o recursos del agente.
- Dependencias, parámetros y configuración específica por instancia; versionado e identificación inmutable de Skills.
- Gestión de Knowledge Base y documentos en CRM; persistencia y ownership; ingestión hacia ai-service.
- Extracción de texto y tratamiento por tipo de documento; chunking, metadatos y configuración de embeddings.
- Generación y persistencia de vectores; colecciones y estructura de payloads en Qdrant.
- Recuperación durante una ejecución; filtros aplicados; ranking, límites, umbrales, selección de fragmentos.
- Tratamiento de errores y fallos de Qdrant; actualización, reingesta, eliminación y obsolescencia de documentos.
- Versionado y trazabilidad del contenido recuperado.
- Modelo de memoria y persistencia; `LoadMemoryStep` y `MemoryPersister`; origen de los hechos y reglas de generación.
- Identidad y alcance de la memoria; asociación con contacto, conversación, Workspace, Organization.
- Recuperación y selección de hechos; orden y formato de incorporación al prompt; límites de cantidad, antigüedad y relevancia.
- `BuildPromptStep` y `PromptDocument`: componentes, orden de composición, precedencia, formato, límites de tokens, truncamiento, priorización.

### 2.2 Exclusiones (no auditar en D4-B)
- Auditorías D1, D2, D3 ni D4-A (no repetir).
- Policy Engine, output filter y prompt injection (D5-A).
- Restricciones de conocimiento externo y confidencialidad (D5-B).
- Aislamiento multitenant y autorización de capabilities, herramientas, datos y telemetría (D5-C).
- No modificar ni implementar soluciones (solo lectura).

---

## 3. Repositorio, commit y fuentes consultadas

### 3.1 Repositorio y commit
- Repositorio: `F:\ElemenSoft\crm` (backend principal).
- Commit HEAD auditado: `b9e0aee594bf7c4ffd1e88c84988b99e54b7d972` (origin/master).
- Estado: copia de trabajo limpia, sin cambios pendientes.
- Microservicio ai-service: `crm/microservices/ai-service/` con `src/` (domain, engine, infrastructure, application, presentation, observability), `dist/`, `tests/`, `docs/`, `package.json`, `jest.config.js`.

### 3.2 Fuentes consultadas (nivel primario salvo indicación)
| Fuente | Ruta / referencia | Uso |
|---|---|---|
| Documento canónico WAB-78 (Fase 2) | `crm/docs/plans/wab-75-fase-2-diseno.md` | Diseño objetivo: E.1 flujo, C.7 snapshot P10, F.1 jerarquía, G.1-G.4 proveedores/créditos, H.4-H.8 entidades |
| Informe Fase 1 (WAB-77) | `crm/docs/plans/wab-75-fase-1-auditoria.md` | Estado as-is: entidades, flujo, pipeline, RAG, prompt, capabilities |
| Línea base revisión WAB-78 | `docs/revision-wab-78/00-linea-base.md` | Estados de control, decisiones aprobadas, convención de IDs |
| Matriz acumulativa | `docs/revision-wab-78/matriz-acumulativa.md` | Hallazgos ARQ-01 a ARQ-22, evitación de duplicación |
| Informe Bloque 1 (D1) | `docs/revision-wab-78/informe-bloque-1.md` | Ownership, persistencia, índices, cardinalidades |
| Informe Bloque 2 (D2) | `docs/revision-wab-78/informe-bloque-2.md` | Permisos, actores, identidad de servicio |
| Informe Bloque 3 (D3) | `docs/revision-wab-78/informe-bloque-3.md` | EvAssigned, atomicidad, cancelación, G3/G4 |
| Informe D4-A | `docs/revision-wab-78/informe-d4-a-runtime.md` | Runtime, configuración efectiva, snapshot P10 |
| Complemento técnico WAB-78 | `F:\ElemenSoft\WAB-78-complemento-tecnico.md` | EvAssigned, cancelación, atomicidad, migración |
| Código fuente ai-service | `crm/microservices/ai-service/src/` | Runtime, pipeline, pasos, ejecución, configuración, RAG, memoria |
| Código fuente CRM | `crm/src/` | Contacto, Evento, Outbox, Rabbit consumers, routing |

### 3.3 Limitaciones de evidencia
- No se pudo consultar el contenido directo de Linear (plan de revisión `fca2514ac4ee`, registro de decisiones `345541575496`); se utilizó documentación local.
- No hay acceso a `.env` ni a producción (ARQ-21).
- No se verificaron índices contra MongoDB Atlas real (deferido a D6).
- `operationType` de EvAssigned es propuesta no aprobada (ARQ-09); `bot.configure` está prohibido por P3 hasta auditar (ARQ-03/ARQ-20).

---

## 4. Topología del flujo de construcción de contexto

El flujo de construcción de contexto se implementa en ai-service como un pipeline secuencial gestionado por `ExecutionEngine` (`crm/microservices/ai-service/src/engine/execution-engine.ts`). Los pasos están en `crm/microservices/ai-service/src/engine/steps/`.

```
Mensaje entrante (RabbitMQ: ai-service.message-received)
        │
        ▼
MessageReceivedHandler (G1: vigencia entrada)
        │ resuelve BotRuntime (Bot + BotInstance + EffectiveConfig)
        │ construye ExecutionContext (config + bot pre-seteados)
        ▼
ProcessMessageService.executeWithContext(context)
        │
        ▼
ExecutionEngine.process(context)
        │ crea Execution entity, markRunning, guarda
        │ carga capabilities (capabilityRegistry.findAll())
        ▼
runPipelineLoop → pipeline.execute(context)
        │ Paso 1: LoadConfigurationStep (config ya seteada, solo gate credencial)
        │ Paso 2: LoadConversationStep (historial desde DTO / [])
        │ Paso 3: ResolvePromptStep (PromptVersion: override activo o publicado)
        │ Paso 4: LoadMemoryStep (hechos por contacto vía MemoryManager)
        │ Paso 5: LoadRulesSkillsStep (Rules + Skills desde crm.bot_resources / bot_resource_links)
        │ Paso 6: ExecuteRAGStep (búsqueda Qdrant filtrada por config.workspaceId)
        │ Paso 7: BuildPromptStep (construye PromptDocument)
        │ Paso 8: InvokeProviderStep (invoca LLM con API key del bot)
        │       si toolCalls → capabilityExecutor.execute → vuelve al bucle (máx 5 iters)
        ▼
G2: vigenciaService.check() antes de markCompleted
        ▼
markCompleted / markFailed
        ▼
finally: publishResult (EventTypes.EXECUTION_COMPLETED, RabbitMQ)
        ▼
CRM consumer executionCompleted (G3): valida vigencia contra contact_assignments (proyección ai-service)
```

### 4.1 Puntos de decisión clave
- `MessageReceivedHandler.buildContext()` pre-setea `context.config` y `context.bot` desde `BotRuntime.resolve(botId)`. El resolver **no recibe** `botInstanceId` del payload; selecciona la primera `BotInstance` activa sin comprobar `isDefault`.
- `LoadConfigurationStep` omite resolución de Bot/BotInstance cuando `context.config` ya está seteado (camino Rabbit); solo aplica `assertCredentialGate`.
- `ResolvePromptStep` resuelve `PromptVersion` vía `IPromptRuntimeResolver.resolve(bot)`: 1) `Bot.activePromptVersionId` (runtime override), 2) `Bot.promptId` → publicado, 3) null → fallback legacy `config.systemPrompt`.
- `ExecuteRAGStep` aplica BR-KB-001: retrieval solo con `config.workspaceId` canónico del bot; si `context.workspaceId !== config.workspaceId` degrada a `[]`.
- `BuildPromptStep` compone el `PromptDocument` final con presupuesto de tokens priorizado.

### 4.2 Componentes del ExecutionContext
`crm/microservices/ai-service/src/domain/value-objects/execution/execution-context.ts` es el objeto mutable compartido por el pipeline. Campos relevantes para construcción de contexto: `config` (EffectiveConfig), `bot`, `history`, `memory` + `memoryContext`, `knowledgeFragments`, `rulesFragments`, `skillsFragments`, `promptVersion`, `renderedPrompt`, `promptDocument`, `providerResponse`, `toolCalls`, `toolResults`, `availableCapabilities`, `cancelled`, `iteration`.

---

## 5. Análisis de PromptVersion y prompts

### 5.1 Resolución de PromptVersion y selección de versión activa/publicada

**Entidades implicadas:**
- `Prompt` (`crm/microservices/ai-service/src/domain/entities/prompt.ts`): pertenece a un `workspaceId`, tiene `name`, `status` (active/deleted), `createdBy`.
- `PromptVersion` (`crm/microservices/ai-service/src/domain/entities/prompt-version.ts`): campos `id`, `promptId`, `version` (>= 1), `content` (PromptContent), `variables` (PromptVariable[]), `status` (PromptVersionStatusEnum: draft/published/archived), `createdBy`, `publishedAt`, `archivedAt`, `createdAt`, `updatedAt`. Métodos `isDraft()`, `isPublished()`, `isArchived()`, `publish()`, `archive()` con validación de transiciones de estado.
- `Bot` (`crm/microservices/ai-service/src/domain/entities/bot.ts`): campos `promptId` (referencia a Prompt._id) y `activePromptVersionId` (runtime override, no parte del ciclo de vida normal, usado exclusivamente por AI Runtime para pinchar a una PromptVersion específica).

**Resolución en ejecución (`ResolvePromptStep`, `crm/microservices/ai-service/src/engine/steps/resolve-prompt.step.ts:32-45`):**
1. Si `context.promptVersion` ya está seteado (one-time init), skip.
2. Requiere `context.bot` (establecido por `LoadConfigurationStep`).
3. Delegado a `IPromptRuntimeResolver.resolve(bot)` (`crm/microservices/ai-service/src/domain/ports/runtime/prompt-runtime-resolver.ts:16-18`).
4. Estrategia del resolver:
   - Paso 1: si `Bot.activePromptVersionId` está seteado (runtime override) → usar esa versión.
   - Paso 2: si no, si `Bot.promptId` está seteado → encontrar la `PromptVersion` publicada asociada.
   - Paso 3: si no → retornar `null` (sin prompt asignado; `BuildPromptStep` fallback a `EffectiveConfig.systemPrompt` legacy).
5. El resolver solo resuelve la entidad `PromptVersion`; no renderiza variables. El renderizado es responsabilidad de `BuildPromptStep` con el contexto completo de ejecución.

### 5.2 Relación Bot, BotInstance, system prompt y PromptVersion

- `BotInstance` tiene campo opcional `systemPrompt` (`crm/microservices/ai-service/src/domain/entities/bot-instance.ts:38`) que sobrescribe el del bot en `resolveEffectiveConfig()` (línea 264: `systemPrompt: this._systemPrompt || 'You are a helpful assistant'`).
- `resolveEffectiveConfig(bot)` requiere `bot.hasPromptVersion()` (línea 249-251), caso contrario lanza error. Esto acopla la resolución de configuración efectiva a la existencia de un prompt activo.
- `BuildPromptStep` (`build-prompt.step.ts:64-75`): si `context.promptVersion` existe, renderiza via `IPromptRenderer.render(context.promptVersion, renderContext)` y usa `renderedPrompt.systemInstructions`. Si no, fallback a `config.systemPrompt` con `promptVersionId = LEGACY_PROMPT_VERSION` ('legacy').
- Comentario TODO en `build-prompt.step.ts:37-39`: "Remove config.systemPrompt fallback after all bots are migrated to Prompt Management (Sprint 5.4+)".

### 5.3 Overrides y precedencias

Precedencia implícita en `BotInstance.resolveEffectiveConfig(bot)` (`bot-instance.ts:240-272`), documento en D4-A §5.2:
| Campo | Origen | Precedencia |
|---|---|---|
| model | `this._model` o `bot.model.withOverrides({temperature: this._temperature})` | Instance primero, fallback bot |
| temperatura | Incluida en `resolvedModel` via `withOverrides` | Sobrescribe solo si instance define temperatura |
| language | `this._language || bot.language` | Instance primero |
| timezone | `this._timezone || bot.timezone` | Instance primero |
| maxIterations | `this._maxIterations || bot.maxIterations` | Instance primero |
| enabledCapabilities | `this._enabledCapabilities || bot.enabledCapabilities` | Instance primero |
| knowledgeBaseIds | `bot.knowledgeBaseIds` | **Siempre del bot**, nunca del instance |
| systemPrompt | `this._systemPrompt || 'You are a helpful assistant'` | Instance primero; fallback genérico hardcodeado (no cae a `bot.systemPrompt`) |

**No hay precedencia documentada explícita entre definición e instancia más allá del fallback implícito en el código.** (Hallazgo D4B-C05, relación con C3/C4 del canónico).

### 5.4 Variables, interpolaciones, placeholders y configuración dinámica

- `PromptVersion.variables` es un arreglo de `PromptVariable` (entidad con nombre y valor, `prompt-version.ts:11`).
- `PromptContext` (`crm/microservices/ai-service/src/domain/ports/services/prompt-renderer.ts:4-19`): define los datos disponibles para renderización: `workspace`, `bot`, `contact`, `conversation` (history, lastMessage, messageCount), `knowledge` (fragments, count), `memory` (facts, summary, categories, factCount, highImportanceCount), `handoff`, `variables` (Record<string, unknown>).
- `IPromptRenderer.render(version, context)` devuelve `RenderedPrompt` con `systemInstructions` y `promptVersionId`.
- `BuildPromptStep.buildPromptContext()` (`build-prompt.step.ts:142-154`) construye el `PromptContext` pasando: `memory` (context.memoryContext), `knowledge` (fragments + count), `conversation` (history.join + lastMessage + messageCount). **No pasa `workspace`, `bot`, `contact`, `handoff`, ni `variables`** del `PromptContext` — esos campos están definidos en el puerto pero no se rellenan desde el contexto de ejecución actual.

### 5.5 Persistencia, versionado y trazabilidad del prompt realmente utilizado

- `PromptVersion.version` es un número entero >= 1 (incremental).
- `PromptVersion.status` rastrea draft/published/archived con transiciones validadas.
- `PromptVersion.publishedAt` y `archivedAt` rastrean cuándo se publicó/archivó.
- La ejecución referencia `promptVersionId` en `PromptDocument.promptVersion` (`build-prompt.step.ts:69`) y `Execution.promptVersionId` declarado en `execution.ts` pero **no asignado** en el flujo de escritura (`execution-engine.ts:153-164` solo asigna id, workspaceId, botId, botInstanceId, conversationId, contactId, channel, correlationData; los setters `setProvider`, `setModel`, `setPromptVersion` no se invocan).
- No existe hash del prompt efectivo (`promptSnapshotHash`) ni snapshot persistido del prompt usado (brecha P10, documentada en D4-A).

### 5.6 Distinción configuración efectiva, prompt efectivo y contexto dinámico

| Concepto | Origen | ¿Versionado? | ¿Snapshot? |
|---|---|---|---|
| Configuración efectiva | `EffectiveConfig` (provider, modelo, prompt, capabilities, knowledgeBaseIds) | No | No (P10 pendiente) |
| Prompt efectivo | `config.systemPrompt` o `PromptVersion` renderizado | `PromptVersion` tiene versión opcional | No |
| Contexto recuperado (RAG) | `knowledgeFragments` de Qdrant (BR-KB-001) | No | No; Qdrant falla silenciosamente a `[]` |
| Memoria | `context.memory` de `MemoryManager.loadForContact` | No | No |
| Rules/Skills | `context.rulesFragments` / `context.skillsFragments` | No (no hay versión en modelo actual) | No |

---

## 6. Análisis de Rules

### 6.1 Origen y persistencia de Rules

**Modelo as-is (ai-service):**
- `crm/bot_resources` y `crm/bot_resource_links` (colecciones de CRM, lectura cross-DB desde ai-service vía `IBotResourceReadRepository`).
- `LoadRulesSkillsStep` (`crm/microservices/ai-service/src/engine/steps/load-rules-skills.step.ts:96-376`) carga Rules y Skills desde estas colecciones (ADR-CD §9.3, lectura cross-DB read-only).
- No existe entidad `AgentRules` en código (grep 0 en `crm/src` y `crm/microservices/ai-service/src`).
- No existe `SkillConfiguration` como entidad separada en código.

**Entidades del modelo objetivo (canónico A.2):**
- `AgentRules` (por instancia, I13): `ruleName`, `content` (implícito), versión. Cardinalidad: 1:N por AgentInstance, N:1 exclusiva.
- `SkillConfiguration` (por instancia, I12): `skillId`, `skillName`, `enabled`, `config` (JSON), `credentials` (JSON). Cardinalidad: 1:N por AgentInstance, N:1.
- `AgentDefinition.defaultRules` y `AgentDefinition.defaultSkills` (JSON[]): reglas/skills por defecto de la definición.
- `AgentInstance.rules` y `AgentInstance.skills` (JSON[]): reglas/skills configuradas para la instancia.

### 6.2 Carga de Rules durante la ejecución

`LoadRulesSkillsStep` (`load-rules-skills.step.ts:107-265`):
1. One-time init: si `context.isRulesSkillsInitialized`, skip.
2. Requiere `context.config`; si no hay config, sets vacíos y return.
3. Lee `links = resourceRepo.findLinksByBot(workspaceId, botId)` (cross-DB, CRM). Si falla → throw `RulesSkillsAccessError` (no se puede determinar si el bot tiene Rules).
4. Si `links.length === 0` → sets vacíos, return (estado normal).
5. Resuelve `resourceIds` a catálogo vía `resourceRepo.findResourcesByIds(workspaceId, resourceIds)`. Si falla → throw `RulesSkillsAccessError`.
6. Separa `ruleLinks` (type === 'rule' && enabled) y `skillLinks` (type === 'skill' && enabled).
7. **Rules (P1, BR-RULES-010):** para cada link de regla, fetch contenido vía `contentFetcher.fetch({source: 'rules-skills/{objectKey}', sourceType: 'minio'})`. Si `content === null` (Redis → MinIO → copia verificable agotadas) → throw `RuleUnavailableError`. Verificación de integridad: hash SHA-256 del contenido debe coincidir con `res.hash` del catálogo; si no coincide, rechazar (retorna null). `ruleText = content.toString('utf-8')`.
8. **Skills (degradación controlada):** para cada link de skill, fetch contenido; si falla → log warn + omitir (nunca bloquea). Extrae `SKILL.md` de ZIP vía parser mínimo con `node:zlib` (`extractSkillMd`).
9. **Presupuesto:** `rulesTokens = sum(tokens de rules)`. Si `rulesTokens > availableBudget (maxTokens - 1024)` → throw controlado. Skills: completa u omitida; si una skill no cabe, omitir con warning (`SkillOmittedBudget`).
10. `context.setRulesSkills(rules, skillsThatFit)`.

### 6.3 Relación con Bot, Workspace y Organization

- Los links y recursos se filtran por `workspaceId` (del `EffectiveConfig`, canónico del bot).
- La lectura es cross-DB: ai-service lee de las colecciones CRM `bot_resources` / `bot_resource_links`.
- No hay relación directa con `Organization` en el modelo actual de rules (la隔离 es por `workspaceId`).

### 6.4 Activación, desactivación, orden y combinación

- **Activación:** solo recursos con `enabled === true` se incluyen (`ruleLinks` / `skillLinks` filtros en líneas 160-167).
- **Orden:** no hay orden explícito de reglas; se unen en el orden de los links. En `BuildPromptStep` (`build-prompt.step.ts:83-95`): Rules van primero, Skills después. Nunca se truncan (validados contra presupuesto en `LoadRulesSkillsStep`).
- **Combinación:** `systemInstructions = baseSystemInstructions + rulesBlock + skillsBlock`, donde cada regla se formatea como `## Rule: {name}\n\n{content}` y cada skill como `## Skill: {name}\n\n{content}`.

### 6.5 Transformación de Rules al formato incorporado al prompt

En `BuildPromptStep`:
```
rulesBlock = rulesFragments.map(r => `## Rule: ${r.name}\n\n${r.content}`).join('\n\n')
skillsBlock = skillsFragments.map(s => `## Skill: ${s.name}\n\n${s.content}`).join('\n\n')
systemInstructions = baseSystemInstructions + rulesBlock + skillsBlock
```
Los fragmentos incluyen `resourceId`, `name`, `content`. El `PromptDocument` expone `rulesContext` y `skillsContext` como arreglos de `{resourceId, name, content}`.

### 6.6 Versionado, revisiones y consistencia durante una ejecución

- **No existe versionado de Rules en el modelo actual.** `AgentRules.version` existe en el modelo objetivo (canónico A.2) pero no implementado.
- **Consistencia durante una ejecución:** `LoadRulesSkillsStep` es one-time init (`isRulesSkillsInitialized`). Las reglas cargadas al inicio permanecen inmutables durante la ejecución. No hay re-carga si cambian entre iteraciones del pipeline.
- **Brecha:** no hay hash ni versión de Rules/Skills incorporados al `PromptDocument` ni a la persistencia de `Execution`.

### 6.7 Hallazgos de Rules

- Reglas cargadas desde CRM cross-DB (`bot_resources` / `bot_resource_links`), no desde entidades `AgentRules` / `SkillConfiguration` (que no existen en código).
- `RuleUnavailableError` propaga como throw del pipeline si una regla habilitada no es recuperable (P1, BR-RULES-010) — impide responder con prompt incompleto.
- Skills se degradan con warning (opcional, nunca bloquean).
- No hay versión inmutable de Rules/Skills referenciada en la ejecución (relacionado con observación C8 de D4-A).

---

## 7. Análisis de Skills

### 7.1 Origen y persistencia de Skills

**Modelo as-is (ai-service):**
- Skills se almacenan en las mismas colecciones CRM `bot_resources` / `bot_resource_links` que las Rules, diferenciadas por `type === 'skill'`.
- `LoadRulesSkillsStep` carga Skills junto con Rules (ver §6.2).
- No existe entidad `SkillConfiguration` en código (grep 0).
- No existe catálogo de Skills independiente en el modelo actual.

**Entidades del modelo objetivo (canónico A.2):**
- `SkillConfiguration` (por instancia, I12): `skillId` (identificador de la skill en catálogo Waby), `skillName`, `enabled` (Boolean), `config` (JSON, configuración específica por instancia), `credentials` (JSON, credenciales propias gestionadas por Waby). Cardinalidad: 1:N por AgentInstance, N:1 exclusiva. Sin herencia ni compartición implícita.
- `AgentDefinition.defaultSkills` (JSON[]): skills por defecto de la definición.
- `AgentInstance.skills` (JSON[]): skills configuradas para la instancia.

### 7.2 `SkillConfiguration` y entidades relacionadas

En el modelo actual no existe `SkillConfiguration`. La funcionalidad equivalente está en `bot_resources`/`bot_resource_links` con campos `type`, `enabled`, `objectKey`, `name`, `hash`. La carga usa `IBotResourceReadRepository` y `ISourceAdapter` (content fetcher).

### 7.3 Relación entre Skill, AgentDefinition y AgentInstance

- **Modelo objetivo (canónico):** `AgentDefinition.defaultSkills` → skills por defecto de la definición (Waby). `AgentInstance.skills` / `SkillConfiguration` → skills por instancia (Workspace). Copiado desde la definición al crear la instancia (`C.5` L559: "Se crea AgentInstance con status draft, copiando Skills/Rules por defecto de la definición").
- **Modelo actual:** No existe separación AgentDefinition/AgentInstance. `Bot.enabledCapabilities` es un arreglo global de strings (capabilities, no skills estructuradas). `BotInstance.enabledCapabilities` sobrescribe o hereda de `Bot`.
- En `EffectiveConfig.enabledCapabilities` (`effective-config.ts:25`): `ReadonlyArray<string>` de nombres de capabilities. Usado por `isCapabilityEnabled()` y por `InvokeProviderStep` como `tools` en la petición al proveedor.

### 7.4 Carga y habilitación de Skills

- **Carga:** `LoadRulesSkillsStep` carga skills habilitadas desde `bot_resource_links` (type === 'skill' && enabled). Contenido extraído de ZIP (`SKILL.md` en raíz) o texto plano.
- **Habilitación:** campo `enabled` del recurso en el catálogo CRM. Solo los habilitados se incluyen.
- **Filtro por instancia:** `EffectiveConfig.enabledCapabilities` proviene de `this._enabledCapabilities || bot.enabledCapabilities` (instance primero, fallback bot). En `ExecutionEngine.runPipelineLoop` (`execution-engine.ts:65-67`), `capabilityRegistry.findAll()` carga **todas** las capabilities sin filtrar por `enabledCapabilities` del bot/instancia (brecha documentada en D4-A D4A-C06).

### 7.5 Transformación de Skills en instrucciones o recursos disponibles para el agente

- En `BuildPromptStep` (`build-prompt.step.ts:90-95`): skills se agregan al `systemInstructions` como bloques `## Skill: {name}\n\n{content}`.
- Las skills también se exponen en `PromptDocument.skillsContext` como `{resourceId, name, content}`.
- Las capabilities (no skills estructuradas) se pasan al proveedor LLM como `tools` vía `context.availableCapabilities` (`InvokeProviderStep`, `execution-engine.ts:194`).

### 7.6 Dependencias, parámetros y configuración específica por instancia

- `SkillConfiguration.config` (JSON) y `SkillConfiguration.credentials` (JSON) son campos del modelo objetivo para configuración específica por instancia. No implementados en el modelo actual.
- `SkillConfiguration.enabled` controla si la skill está activa para la instancia.
- En el modelo actual, `enabledCapabilities` es un arreglo de strings sin parámetros ni credenciales asociadas.

### 7.7 Versionado e identificación inmutable de Skills

- **No existe versionado de Skills en el modelo actual.** `SkillConfiguration` no declara campo `version` en A.2 del canónico; `AgentRules` sí declara `version`.
- La observación C8 de D4-A señala que el contrato de snapshot requiere identificar versiones de Skills, pero no hay entidad o catálogo que proporcione esa versión inmutable.
- En el modelo actual, `bot_resources` tiene campo `hash` (verificación de integridad del contenido en `LoadRulesSkillsStep.fetchContent`), pero no una versión semántica ni inmutable de la skill.

### 7.8 Hallazgos de Skills

- Skills y Rules comparten el mismo mecanismo de carga (`bot_resources` / `bot_resource_links` cross-DB).
- No existe `SkillConfiguration` ni catálogo de Skills como entidad separada.
- `enabledCapabilities` se carga globalmente sin filtro por instancia en el engine (D4A-C06).
- No hay versión inmutable de Skills referenciada en la ejecución (observación C8 de D4-A pendiente).

---

## 8. Análisis de Knowledge / RAG

### 8.1 Gestión de Knowledge Base y documentos en CRM

**Modelo actual (ai-service):**
- `crm/knowledge_bases` (colección en ai-service, persistencia actual según Fase 1 §2.1).
- `crm/documents` y `crm/chunks` (colecciones en ai-service).
- Casos de uso: `createKnowledgeBase`, `updateKnowledgeBase`, `deleteKnowledgeBase`, `listKnowledgeBases`, `createDocument`, `getDocument`, `listDocumentsByKnowledgeBase`, `deleteDocument`, `ingestDocument`, `ingestDocumentV2` (`crm/microservices/ai-service/src/application/use-cases/`).

**Modelo objetivo (canónico A.2):**
- `WorkspaceKnowledge` (compartido por workspace, I11): `name`, `embeddingConfig` (proveedor, modelo, dimensión), `vectorStoreConfig`. Cardinalidad: 1:N por Workspace, todas las AgentInstance del mismo workspace comparten los mismos WorkspaceKnowledge.
- `knowledgeBaseIds` en `AgentInstance` / `EffectiveConfig`: referencias a WorkspaceKnowledge del workspace.

**Brecha de propiedad de persistencia (ARQ-14):** D.1 asigna la gestión de Knowledge a CRM, pero la persistencia actual (`knowledge_bases`) está en ai-service. H.1 dice "conservar → WorkspaceKnowledge" sin indicar traslado.

### 8.2 Persistencia y ownership de los recursos

- **Ownership del conocimiento:** Workspace (compartido, I11). Restricción #4 L434: `WorkspaceKnowledge.workspaceId`.
- **Filtrado en búsqueda:** BR-KB-001 — el retrieval usa `config.workspaceId` canónico del bot, nunca `context.workspaceId` si difiere (`ExecuteRAGStep` L43-55).
- **Eliminación:** Al eliminar una WorkspaceKnowledge, invalidar embeddings asociados en Qdrant (purge automático, canónico H.4 L1718). No exigir `agentInstanceId` como filtro en Qdrant (I11).

### 8.3 Ingestión hacia ai-service

Casos de uso de ingestión (`crm/microservices/ai-service/src/application/use-cases/ingest-document.use-case.ts`, `ingest-document-v2.use-case.ts`):
- `createDocument.use-case.ts`, `get-document.use-case.ts`, `delete-document.use-case.ts`, `update-document-status.use-case.ts`.
- `ingestion-pipeline.factory.ts` y `ingestion-pipeline.ts` (`crm/microservices/ai-service/src/engine/ingestion/`): pipeline de ingestión con steps, hooks, contexto de ingestión (`ingestion-context.ts`).
- `ingestion-run.ts` (entidad), `ingestion-run-repository.ts`.

### 8.4 Extracción de texto y tratamiento por tipo de documento

Extractores en `crm/microservices/ai-service/src/infrastructure/rag/`:
- `plain-text-extractor.ts`, `markdown-extractor.ts`, `pdf-extractor.ts`, `docx-extractor.ts`.
- `text-extractor-registry.ts`: registro de extractores por tipo.
- `fixed-size-chunker.ts`, `fake-text-chunker.ts`: chunking.
- `simple-text-normalizer.ts`: normalización de texto.
- `fake-embedding-provider.ts`: proveedor de embeddings (fake, para tests).

Entidades de valor relacionadas:
- `ExtractedText`, `NormalizedText`, `ChunkSegment`, `ChunkMetadata`, `ChunkCollection`, `IndexedChunks`, `EmbeddedChunks`, `EmbeddingConfig`, `VectorStoreConfig`, `IngestionStatus`, `IngestionPipelineResult`.

### 8.5 Chunking, metadatos y configuración de embeddings

- `Chunk` entidad (`crm/microservices/ai-service/src/domain/entities/chunk.ts`).
- `DocumentVersion` entidad (`document-version.ts`).
- `EmbeddingConfig`: proveedor, modelo, dimensión de embeddings.
- `VectorStoreConfig`: configuración del vector store.
- Chunking de tamaño fijo (`fixed-size-chunker.ts`).

### 8.6 Generación y persistencia de vectores

- `IVectorStore` puerto (`crm/microservices/ai-service/src/domain/ports/providers/vector-store.ts`): operación `search({workspaceId, collection, queryEmbedding, topK})`.
- `IEmbeddingProvider` puerto (`.../providers/embedding-provider.ts`): operación `embed(query, {inputType})`.
- `VectorSearchResult` valor objeto: `{score, content, metadata}`.

### 8.7 Colecciones y estructura de payloads en Qdrant

- En `RAGService.search()` (`crm/microservices/ai-service/src/infrastructure/services/rag-service.ts:35-80`): para cada `kbId` de `knowledgeBaseIds`, llama `vectorStore.search({workspaceId, collection: kbId, queryEmbedding, topK})`. La colección en Qdrant es el `kbId` (knowledge base id).
- Filtro por `workspaceId` enviado en cada búsqueda.
- Umbral: `result.score >= threshold` (default 0.25) para incluir un resultado.
- Ranking: `allResults.sort((a, b) => b.score - a.score)` descendente por score.
- Límite: `.slice(0, topK)` (default 5).
- `applyTokenBudget()`: si hay `defaultMaxTokens` y `tokenCounter`, ajusta fragmentos al presupuesto restante (evalúa en orden de score, uno entra si cabe).

### 8.8 Recuperación durante una ejecución

`ExecuteRAGStep` (`crm/microservices/ai-service/src/engine/steps/execute-rag.step.ts:28-64`):
1. One-time init: si `context.isKnowledgeFragmentsInitialized`, skip.
2. `knowledgeBaseIds = config?.knowledgeBaseIds ?? []`. Si vacío o sin config → `setKnowledgeFragments([])`, return.
3. BR-KB-001: si `context.workspaceId !== config.workspaceId` → log warn + `setKnowledgeFragments([])`, return (nunca usa `context.workspaceId` si difiere).
4. Llama `ragService.search({workspaceId: config.workspaceId, knowledgeBaseIds, query: context.message.content})`.
5. `context.setKnowledgeFragments([...fragments])`.

**Tratamiento de errores de Qdrant:** `RAGService.search` tiene try/catch alrededor de todo (línea 44-79) que retorna `[]` en caso de fallo. Dentro del loop por `kbId`, cada búsqueda tiene try/catch individual que hace `continue` en caso de error (línea 66-68). **Falla silenciosamente a `[]`** si Qdrant cae — documentado en Fase 1 y D4-A. El canónico E.1 paso 6 (L920-924) dice "Qdrant error → error visible, no [] silencioso" — **desacuerdo entre diseño objetivo e implementación actual**.

### 8.9 Filtros aplicados en la búsqueda

- `workspaceId` (canónico del bot, BR-KB-001).
- `collection` = `kbId` (cada knowledge base es una colección separada).
- `topK` (default 5).
- `threshold` (default 0.25, filtro por score).
- `maxTokens` / token budget (applyTokenBudget, opcional).

### 8.10 Ranking, límites, umbrales y selección de fragmentos

- Ranking descendente por score.
- Umbral mínimo 0.25.
- Top-K 5 por defecto.
- Token budget opcional (D7: `RETRIEVAL_MAX_TOKENS`).
- Selección: fragmentos que pasan umbral, ordenados por score, limitados por topK y luego por token budget.

### 8.11 Tratamiento de errores y fallos de Qdrant

- Error de embedding: catch externo retorna `[]`.
- Error por KB individual: catch interno `continue` (salta esa KB, continúa con las siguientes).
- Resultado vacío: `[]` válido (RAG ejecutado pero no encontró nada).
- No hay reintentos, no hay circuit breaker, no hay fallback a otra fuente.

### 8.12 Actualización, reingesta, eliminación y obsolescencia de documentos

- Casos de uso: `updateDocument`, `deleteDocument`, `updateDocumentStatus`.
- Reingesta: `ingestDocument`, `ingestDocumentV2`.
- Al eliminar WorkspaceKnowledge: invalidar embeddings en Qdrant (purge automático, canónico H.4).
- No hay evidencia de obsolescencia automática de chunks o versionado de documentos en el modelo actual (`DocumentVersion` existe como entidad pero su uso en la ruta de ejecución no está claro).

### 8.13 Versionado y trazabilidad del contenido recuperado

- **No hay registro de qué fragmentos se recuperaron en cada ejecución.** `ExecuteRAGStep` guarda `knowledgeFragments` (contenido de strings) en el contexto, pero no se persisten IDs de fragmentos, scores, ni versiones de documentos/chunks asociados a la ejecución.
- `PromptDocument.knowledgeContext` almacena los fragmentos de string (`build-prompt.step.ts:128`).
- El canónico C.7.3 (L658-670) distingue claramente: `knowledgeBaseIds` identifica bases habilitadas (configuración) pero **no** registra el contenido efectivamente recuperado. Para auditoría se deberían registrar IDs de fragmentos Qdrant, versiones de documentos/chunks y hashes de fragmentos utilizados — **no implementado**.

### 8.14 Distinción: bases habilitadas vs documentos indexados vs fragmentos recuperados vs contenido incorporado

| Concepto | Estado actual | ¿Persistido por ejecución? |
|---|---|---|
| Bases de conocimiento habilitadas | `knowledgeBaseIds` en `EffectiveConfig` (del Bot) | No (solo IDs en config) |
| Documentos indexados | Colecciones Qdrant por kbId | No |
| Fragmentos recuperados | `knowledgeFragments` (strings) en contexto | No (contenido solo, sin IDs/scores) |
| Contenido incorporado al prompt | `PromptDocument.knowledgeContext` | No (solo dentro del PromptDocument, no persistido) |

### 8.15 Hallazgos de Knowledge/RAG

- Qdrant falla silenciosamente a `[]` (implementación) vs "error visible" (canónico E.1 paso 6).
- No se registran IDs de fragmentos, scores ni versiones para trazabilidad RAG por ejecución (canónico C.7.3 Nivel 2 no implementado).
- Propiedad de persistencia de Knowledge sin declarar (CRM vs ai-service, ARQ-14).
- `knowledgeBaseIds` siempre proviene del Bot, nunca del AgentInstance (modelo actual).

---

## 9. Análisis de memoria

### 9.1 Modelo de memoria y persistencia

- `MemoryFact` entidad (`crm/microservices/ai-service/src/domain/entities/memory-fact.ts`): campos workspaceId, contactId, category, key, value, confidence, importance, source, sourceType, sourceId, extractionMethod, conversationId, visibility, extractedAt, expiración.
- `MemoryFactStatus` enum: ACTIVE y otros estados. `isExpired()` verifica antigüedad.
- `IMemoryFactRepository` puerto (`.../ports/repositories/memory-repository.ts`): `findActiveByContact(workspaceId, contactId, limit?)`, `save(fact)`.
- Implementación de repositorio en infraestructura (no inspeccionada en detalle).

### 9.2 `LoadMemoryStep` y `MemoryPersister`

**`LoadMemoryStep`** (`crm/microservices/ai-service/src/engine/steps/load-memory.step.ts:13-30`):
1. One-time init: si `context.isMemoryInitialized`, skip.
2. Llama `memoryManager.loadForContact(context.workspaceId, context.contactId)`.
3. Convierte hechos a `AbstractMessage.system(fact)` para cada fact.
4. `context.setMemory(messages)` + `context.setMemoryContext(memoryContext)`.

**`MemoryManager`** (`crm/microservices/ai-service/src/engine/memory-manager.ts:32-126`):
- `MAX_MEMORY_FACTS = 30`, `HIGH_IMPORTANCE_THRESHOLD = 7`.
- `loadForContact`: `memoryFactRepo.findActiveByContact(workspaceId, contactId, MAX_MEMORY_FACTS)`. Filtra `status === ACTIVE && !isExpired()`. Formatea hechos como `[Category] key: value [HIGH]`. Retorna `MemoryContext` con facts, summary (vacío ''), categories, factCount, highImportanceCount.
- `persistFacts`: carga hechos existentes, deduplica vía `MemoryDeduplicator`, guarda toUpdate, toContradict, toCreate.

**`MemoryPersister`** (`crm/microservices/ai-service/src/engine/memory-persister.ts:26-218`):
- Se invoca en `ExecutionEngine.process()` finally (`execution-engine.ts:123-125`): si `memoryPersister` y `!execution.isCancelled()`, `memoryPersister.persist(context)`.
- Extrae hechos de la respuesta del proveedor usando extractores (RuleMemoryExtractor + LLMMemoryExtractor en modo HYBRID por defecto).
- `buildExtractionInput`: workspaceId, contactId, conversationId, botInstanceId, userMessage, aiResponse, conversationHistory.
- `transformFindings`: convierte `MemoryFinding[]` a `MemoryFact[]` con `MemoryFact.create()`. Categoría derivada del prefijo de la key, importancia de un mapa fijo, visibility siempre `MemoryVisibility.SHARED`, source derivada de SourceType, estado siempre ACTIVE.
- Errores de persistencia se capturan y loguean (fail-open, no rompe el flujo de ejecución).

### 9.3 Origen de los hechos y reglas de generación

- Origen: extracción de la conversación (userMessage + aiResponse + conversationHistory) vía extractores.
- Modo de extracción: `context.bot?.extractionMode ?? ExtractionMode.HYBRID` (NONE, RULES, LLM, HYBRID).
- `MemoryKeyPolicyRegistry` (`IMemoryKeyPolicyRegistry`) para políticas de clave.
- `MemoryVisibility.SHARED` para todos los hechos (sin aislamiento por rol de usuario).

### 9.4 Identidad y alcance de la memoria

- Alcance: por `workspaceId` + `contactId` (findActiveByContact filtra por ambos).
- No hay aislamiento por conversación dentro del mismo contacto (hechos compartidos entre conversaciones del mismo contacto).
- Visibilidad siempre SHARED (sin distinción de quién puede ver qué hecho).

### 9.5 Asociación con contacto, conversación, Workspace y Organization

- Asociación directa: workspaceId, contactId, conversationId en cada MemoryFact.
- No hay relación directa con Organization (se infiere vía workspace).
- `conversationId` se almacena pero no se usa para filtrado en la carga (findActiveByContact usa workspaceId + contactId).

### 9.6 Recuperación y selección de hechos

- `findActiveByContact(workspaceId, contactId, MAX_MEMORY_FACTS=30)`: hasta 30 hechos activos y no expirados por contacto.
- Filtro: `status === ACTIVE && !isExpired()`.
- Orden: no especificado (depende del repositorio subyacente).
- No hay selección por relevancia semántica; solo por actividad y expiración.

### 9.7 Orden y formato de incorporación al prompt

- Formato: `[Category] key: value [HIGH]` (memory-manager.ts:99-104).
- Categoría prefix: Personal, Preference, Business, Goal, Constraint, Relationship, Fact, Memory (default).
- Indicador `[HIGH]` si `importance >= 7`.
- En `BuildPromptStep`: memoria se incluye en `PromptDocument.memory` y en `PromptContext.memory` (facts, summary, categories, factCount, highImportanceCount).
- En `build-prompt.step.ts:115`: `memoryTokens = countMemoryTokens([...context.memory])` con overhead fijo (CONVERSATION_OVERHEAD=2 + MESSAGE_OVERHEAD=4 por mensaje).
- Memoria no se trunca en `BuildPromptStep` (solo se cuenta en el presupuesto; el truncamiento aplica solo al historial). Ver §10.

### 9.8 Límites de cantidad, antigüedad y relevancia

- **Cantidad:** MAX_MEMORY_FACTS = 30 (límite de carga).
- **Antigüedad:** `isExpired()` en MemoryFact (hechos expirados se excluyen).
- **Relevancia:** no hay puntuación de relevancia; alta importancia (>=7) se marca con [HIGH] pero no se prioriza en carga.

### 9.9 Actualización, eliminación y consistencia

- `persistFacts` ejecuta deduplicación y guarda toUpdate/toContradict/toCreate por separado (3 loops secuenciales, no transacción única).
- No hay eliminación explícita de hechos obsoletos (solo expiración pasiva via `isExpired()`).
- No hay transacción entre persistencia de memoria y otros cambios de la ejecución.

### 9.10 Riesgos de mezclar información entre contextos de ejecución

- Hechos cargados por `workspaceId + contactId`: si el mismo contacto tiene múltiples conversaciones, los hechos se comparten entre todas (no hay aislamiento por conversación).
- `MemoryVisibility.SHARED` para todos los hechos: sin control de visibilidad por rol.
- `MemoryPersister` persiste hechos del contacto actual; si hay ejecución concurrente para el mismo contacto, pueden escribirse hechos duplicados o contradictorios (la deduplicación carga existentes antes de guardar, pero hay ventana de carrera entre la lectura y la escritura).

### 9.11 Hallazgos de memoria

- Ventana de 30 hechos (canónico E.1 paso 8 especifica ventana de 20 mensajes user/assistant — desacuerdo: implementación usa 30 MemoryFact, no 20 mensajes).
- `summary` siempre vacío ('') en MemoryContext (no hay resumen automático, coincide con canónico).
- Memoria cargada antes que knowledge en el pipeline (paso 4 vs paso 6), pero en `BuildPromptStep` el presupuesto prioriza knowledge antes que memory antes que history.
- No hay aislamiento de hechos por conversación ni por visibilidad diferenciada.

---

## 10. Análisis de BuildPromptStep y PromptDocument

### 10.1 Reconstrucción exacta de `BuildPromptStep`

Archivo: `crm/microservices/ai-service/src/engine/steps/build-prompt.step.ts`. Clase `BuildPromptStep implements PipelineStep`.

**Flujo de `execute(context)` (líneas 51-140):**

1. Requiere `context.config` (EffectiveConfig), caso contrario throw.
2. `totalBudget = config.model.maxTokens`; `availableBudget = totalBudget - DEFAULT_RESPONSE_RESERVE` (DEFAULT_RESPONSE_RESERVE = 1024).
3. Resuelve instrucciones base del sistema y ID de versión de prompt:
   - Si `context.promptVersion` existe: renderiza via `this.promptRenderer.render(context.promptVersion, renderContext)` donde `renderContext = buildPromptContext(context)`. Obtiene `baseSystemInstructions = renderedPrompt.systemInstructions` y `promptVersionId = renderedPrompt.promptVersionId`. Guarda `context.setRenderedPrompt(renderedPrompt)`.
   - Si no: fallback legacy a `config.systemPrompt`, `promptVersionId = LEGACY_PROMPT_VERSION` ('legacy').
4. Obtiene `rulesFragments = context.rulesFragments` y `skillsFragments = context.skillsFragments`.
5. **Append Rules y Skills a system instructions:** Rules primero, Skills después. Formato: `## Rule: {name}\n\n{content}` y `## Skill: {name}\n\n{content}`, unidos con '\n\n'. Nunca se truncan (validados contra presupuesto en LoadRulesSkillsStep).
6. `systemTokens = tokenCounter.count(systemInstructions)`. Si `systemTokens > availableBudget` → throw.
7. `toolResultItems = buildToolResults(context)`; `userMessage = buildUserMessage(context)`.
8. `messageTokens = tokenCounter.count(userMessage.content)`. Si `messageTokens > availableBudget - systemTokens` → throw.
9. `knowledgeTokens = countKnowledgeTokens(context.knowledgeFragments)`; `memoryTokens = countMemoryTokens([...context.memory])`.
10. `historyBudget = availableBudget - systemTokens - messageTokens - knowledgeTokens - memoryTokens`.
11. `truncatedHistory = truncateHistory([...context.history], Math.max(0, historyBudget))` — trunca desde el más antiguo hacia atrás hasta ajustar al presupuesto.
12. `historyTokens = tokenCounter.countMessages(truncatedHistory)`; `totalTokens = suma de todos`.
13. Construye `new PromptDocument({...})` con: systemInstructions, promptVersion, history (truncado), userMessage, memory, knowledgeContext, rulesContext, skillsContext, toolCalls, toolResults, metadata (PromptMetadata con totalTokens y TokenBudget).
14. `context.setPromptDocument(promptDocument)`.

**Métodos privados:**
- `buildPromptContext(context)`: construye `PromptContext` con `memory` (context.memoryContext), `knowledge` (fragments + count si hay), `conversation` (history.join('\n'), lastMessage, messageCount). **No rellena workspace, bot, contact, handoff, variables.**
- `buildToolResults(context)`: mapea toolResults a `{toolCallId, content}` (JSON.stringify de data o `Error: {error}`).
- `buildToolCalls(context)`: si no hay toolResults, retorna []; si los hay, mapea toolCalls a `{id, name, parameters}`.
- `buildUserMessage(context)`: si hay toolCalls, anexa `[Tool Call: name(params)]` al contenido del mensaje del usuario como `[Tool Context]`.
- `countKnowledgeTokens(fragments)`: suma de tokens de cada fragmento.
- `countMemoryTokens(memory)`: CONVERSATION_OVERHEAD (2) + MESSAGE_OVERHEAD (4) por mensaje + tokens de contenido.
- `truncateHistory(messages, budget)`: itera desde el último mensaje hacia atrás, acumula tokens + overhead, detiene al exceder budget. Retorna mensajes en orden original (unshift).

### 10.2 Estructura de `PromptDocument`

Archivo: `crm/microservices/ai-service/src/domain/value-objects/prompt/prompt-document.ts`. Clase `PromptDocument` (congelada con `Object.freeze`).

**Campos (PromptDocumentProps):**
- `systemInstructions: string` (requerido, no vacío)
- `promptVersion: string` (requerido, no vacío)
- `history: AbstractMessage[]` (requerido)
- `userMessage: AbstractMessage` (requerido)
- `memory?: AbstractMessage[]`
- `knowledgeContext?: string[]`
- `rulesContext?: Array<{resourceId, name, content}>`
- `skillsContext?: Array<{resourceId, name, content}>`
- `toolCalls?: ToolCallItem[]` (`{id, name, parameters}`)
- `toolResults?: ToolResultItem[]` (`{toolCallId, content}`)
- `metadata: PromptMetadata` (requerido: totalTokens, tokenBudget)

**Garantías:**
- Todos los arreglos se clonan y congelan (`Object.freeze`).
- Objeto raíz congelado (`Object.freeze(this)`).
- `validate()` en constructor verifica campos requeridos.
- `toJSON()` retorna PromptDocumentProps (no congelado, copia superficial).
- `equals(other)` compara systemInstructions, promptVersion, userMessage, metadata, history (por longitud e igualdad elemento a elemento).
- `withAdditionalToolResult(result)` retorna nueva PromptDocument con el toolResult añadido.

**`totalMessages()`**: retorna `history.length + 1 + memory.length` (historial + mensaje de usuario + memoria).

### 10.3 Componentes incluidos y orden de composición

**Orden de composición del systemInstructions:**
1. `baseSystemInstructions` (PromptVersion renderizado o `config.systemPrompt` legacy).
2. Bloque de Rules (`## Rule: {name}\n\n{content}` cada una).
3. Bloque de Skills (`## Skill: {name}\n\n{content}` cada una).

**Componentes del PromptDocument final:**
| Componente | Origen | Orden en mensaje al proveedor |
|---|---|---|
| systemInstructions | PromptVersion renderizado / config.systemPrompt + Rules + Skills | Primero (rol system) |
| history | contexto de conversación (últimos N truncados por presupuesto) | Después del system |
| memory | hechos de memoria por contacto | Dentro de history/memoria |
| knowledgeContext | fragmentos de Qdrant (RAG) | Incorporados en systemInstructions o como contexto |
| userMessage | mensaje actual + tool context si aplica | Último (rol user) |
| toolCalls / toolResults | resultados de tool calling de iteraciones previas | En userMessage o como mensajes de tool |

### 10.4 Precedencia de instrucciones

1. **System prompt base:** PromptVersion.rendered.systemInstructions (si hay PromptVersion) o `config.systemPrompt` (legacy).
2. **Rules:** append después del system prompt base (nunca truncadas).
3. **Skills:** append después de Rules (nunca truncadas).
4. **Mensaje de usuario:** contenido actual + tool context.
5. **Historial:** truncado desde el más antiguo para ajustar al presupuesto restante.
6. **Memoria:** contada en presupuesto pero no truncada explícitamente en BuildPromptStep (se cuenta completa).

**Precedencia efectiva:** system > rules > skills > knowledge > memoria > historial > usuario (en términos de presupuesto: system y usuario son "required — must fit or error"; knowledge, memoria e historial son opcionales y se ajustan al presupuesto restante).

### 10.5 Formato y estructura de mensajes

- System: `systemInstructions` como un solo bloque de texto (rol system implícito en InvokeProviderStep).
- Historial: arreglo de `AbstractMessage` con rol (user/assistant) y contenido.
- Memoria: arreglo de `AbstractMessage.system(fact)` (rol system, formato `[Category] key: value [HIGH]`).
- Usuario: `AbstractMessage` con rol user, contenido actual opcionalmente anexado con `[Tool Context]\n...`.
- Tool calls/results: anexados al contenido del usuario como texto, no como mensajes de tool separados del proveedor.

### 10.6 Límites de tokens, truncamiento y priorización

**Presupuesto:** `availableBudget = maxTokens - 1024` (reserva para respuesta).

**Prioridad de asignación:**
1. System prompt (required — debe caber o error).
2. Mensaje de usuario (required — debe caber o error).
3. Knowledge fragments (opcional — incluido si cabe; contado completo).
4. Memory (opcional — contado completo, no truncado en BuildPromptStep).
5. History (opcional — truncado desde el más antiguo para ajustar al presupuesto restante).

**Comportamiento cuando excede:**
- System prompt excede: throw `System prompt exceeds available token budget`.
- Mensaje de usuario excede: throw `User message exceeds available token budget`.
- Knowledge + memoria + history juntos: se reduce history (truncado). Knowledge y memoria se cuentan completos; si juntos con system y usuario exceden el budget, history se trunca a 0 y el total excede (no hay error, solo se usa menos history).

**Default response reserve:** 1024 tokens. `CONVERSATION_OVERHEAD = 2`, `MESSAGE_OVERHEAD = 4`.

### 10.7 Transformaciones o normalizaciones

- Rules y Skills se formatean con markdown `## Rule:` / `## Skill:`.
- Tool calls se serializan como `[Tool Call: name(params)]` en el mensaje de usuario.
- Tool results se serializan como `JSON.stringify(data)` o `Error: {error}`.
- Knowledge fragments se usan como strings puros (sin formato especial).
- Memoria se formatea como `[Category] key: value [HIGH]`.
- Todo el PromptDocument se congela (inmutabilidad post-construcción).

### 10.8 Diferencias entre contexto cargado y contexto enviado al proveedor

**Contexto cargado (ExecutionContext):** config, bot, history, memory + memoryContext, knowledgeFragments, rulesFragments, skillsFragments, promptVersion, renderedPrompt, availableCapabilities, toolCalls, toolResults, providerResponse, cancelled, iteration.

**Contexto enviado al proveedor (a través de PromptDocument → InvokeProviderStep):**
- `systemInstructions` (texto completo: base + rules + skills).
- `userMessage` (texto: contenido + tool context).
- `history` (truncado).
- `knowledgeContext` (fragmentos de string).
- `memory` (mensajes system).
- `rulesContext` / `skillsContext` (metadatos de recursos).
- `toolCalls` / `toolResults`.
- `promptVersion` (ID).
- `metadata.totalTokens` y `metadata.tokenBudget`.

**Qué NO se envía al proveedor (perdido en la transición):**
- `context.bot` (solo se usa para resolución, no se serializa en prompt).
- `context.config` completo (solo `maxTokens` para presupuesto y `systemPrompt` si legacy).
- `context.memoryContext` (solo `memory` mensajes van al prompt; summary/categories/factCount no se incluyen en el texto del prompt).
- `context.availableCapabilities` (se pasa como `tools` al proveedor, no como texto del prompt).
- `context.correlationData`, `context.traceId`, `context.executionId` (metadatos de traza, no van al prompt).
- Versiones de Rules/Skills (no hay versión en el modelo actual).
- IDs de fragmentos de knowledge (solo el contenido string va al prompt).

---

## 11. Comparación as-is vs. diseño objetivo

| Dimensión | Estado actual (as-is) | Diseño objetivo (canónico) | ¿Coincide? | Hallazgo relacionado |
|---|---|---|---|---|
| Modelo de agente | `Bot` / `BotInstance` (mezcla definición e instancia) | `AgentDefinition` (Waby) + `AgentInstance` (Workspace) | No | D4B-C01, D4A-C01 |
| System prompt | `BotInstance.systemPrompt || 'You are a helpful assistant'` (fallback hardcodeado, no cae a `bot.systemPrompt`) | `AgentDefinition.systemPrompt` + `AgentInstance.systemPromptOverride` | Parcial | D4B-C02 |
| Prompt versioning | `PromptVersion` con versión numérica, status draft/published/archived; `Bot.activePromptVersionId` override; `Bot.promptId` | PromptVersion activo/publicado por AgentDefinition | Parcial | — |
| Rules | `bot_resources` / `bot_resource_links` (cross-DB CRM), type 'rule', enabled, hash; sin versionado | `AgentRules` por instancia, con versión, `defaultRules` en definición | No | D4B-C03, D4A-C08 |
| Skills | `bot_resources` / `bot_resource_links`, type 'skill', enabled; `enabledCapabilities` como strings globales | `SkillConfiguration` por instancia, `config`/`credentials` JSON, `defaultSkills` en definición | No | D4B-C04, D4A-C08 |
| Versión inmutable de Skills | No existe (solo `hash` de integridad de contenido) | `Skill.id + version` en snapshot; catálogo que provea versión inmutable | No | D4B-C05 (observación C8 de D4-A) |
| Knowledge | `knowledgeBaseIds` del Bot; Qdrant por kbId; fallback silencioso a `[]` | `WorkspaceKnowledge` compartido por workspace; error visible, no `[]` silencioso; trazabilidad RAG Nivel 2 | Parcial | D4B-C06, D4B-C07 |
| Memoria | `MemoryFact` hasta 30 por contacto, sin expiración automática activa en carga, `summary` vacío | `ConversationMemory` ventana 20 mensajes user/assistant | No | D4B-C08 |
| Snapshot inmutable (P10) | No existe (`effectiveConfigSnapshot`, `effectiveConfigHash`, `promptSnapshotHash` no persistidos) | Snapshot al iniciar ejecución; hashes SHA-256; `effectiveConfigSnapshot` JSON sin secretos | No | D4B-C09, D4A-C02/C03/C04 |
| BuildPrompt / PromptDocument | Implementado con presupuesto de tokens priorizado; Rules+Skills append; historial truncado | [System Prompt] + [RAG Workspace] + [20 msgs]; Policy Engine prevalece | Parcial | D4B-C10 |
| Contexto RAG incorporado | Solo contenido string de fragmentos (sin IDs, scores, versiones) | IDs de fragmentos, versiones de documentos/chunks, hashes de fragmentos | No | D4B-C07 |
| Precedencia definición vs instancia | Fallback implícito (`this._x || bot.x`), sin reglas documentadas | Reglas de combinación documentadas (C3, C4) | No | D4B-C11 |
| Cancelación física LLM | No existe (sin AbortController) | Intentar cancelar físicamente si es posible | No | ARQ-07 (ya registrado) |
| Validación vigencia G3/G4 | Contra `contact_assignments` (proyección ai-service), no contra CRM | Contra CRM (autoridad única) | No | ARQ-08 (ya registrado) |

**Desacuerdos principales:**
1. **Qdrant error handling:** implementación falla silenciosamente a `[]`; canónico E.1 paso 6 exige "error visible, no [] silencioso".
2. **Ventana de memoria:** implementación usa 30 MemoryFact; canónico E.1 paso 8 especifica ventana de 20 mensajes user/assistant.
3. **System prompt fallback:** implementación cae a 'You are a helpful assistant' (hardcodeado); canónico prevé `AgentInstance.systemPromptOverride` sobre `AgentDefinition.systemPrompt`.
4. **Conocimiento como rol:** implementación no distingue rol de knowledge en el prompt; canónico ADR-003 / F.3 exige rol `user` o `knowledge` dedicado, nunca `system`.
5. **Propiedad de persistencia Knowledge:** CRM (diseño) vs ai-service (as-is) — ARQ-14.

---

## 12. Contratos definidos y pendientes

### 12.1 Contratos ya definidos (en documentación, no necesariamente implementados)

| # | Contrato | Estado | Evidencia |
|---|----------|--------|-----------|
| CTX-01 | Separación AgentDefinition (Waby) + AgentInstance (Workspace) | Aprobado (diseño) | Canónico A.2, B; P10 aprobado |
| CTX-02 | AgentInstance persiste en CRM | Aprobado | Registro §8.1 |
| CTX-03 | Cadena AgentInstance → Workspace → Organización | Aprobado | Registro §8.1; D.1.5 L734 |
| CTX-04 | Snapshot inmutable de configuración efectiva por ejecución (P10) | Aprobado | Registro §2 (P10); Canónico C.7, E.1 paso 5 |
| CTX-05 | Dos hashes: `promptSnapshotHash` + `effectiveConfigHash` | Aprobado (diseño) | Canónico C.7.1 |
| CTX-06 | `effectiveConfigSnapshot` como JSON sin secretos | Aprobado (diseño) | Canónico C.7.2 |
| CTX-07 | `knowledgeBaseIds` = bases habilitadas (no contenido recuperado) | Aprobado | Canónico C.7.3 |
| CTX-08 | Provider/modelo/credenciales centralizados en Waby | Aprobado | Canónico G.1, G.2 |
| CTX-09 | Knowledge como rol no-sistema (user o knowledge dedicado) | Aprobado (ADR-003) | Canónico L1882-1890 |
| CTX-10 | Inmutabilidad de configuración durante la ejecución | Aprobado | Canónico C.7.2, E.1 paso 5 |
| CTX-11 | G3/G4 validan vigencia antes de despacho | Aprobado | Canónico D.1.8, E.4.12 |
| CTX-12 | `knowledgeBaseIds` filtra por `config.workspaceId` (BR-KB-001) | Aprobado | Canónico E.1 paso 6; ExecuteRAGStep L43-55 |

**Total contratos ya definidos:** 12

### 12.2 Contratos pendientes de definición en WAB-78

| # | Contrato pendiente | ¿Bloquea WAB-78? | Justificación | Destino |
|---|-------------------|-----------------|---------------|---------|
| CTX-P1 | Resolución de AgentDefinition: quién la crea, versiona, publica, retira | No | Contrato de diseño; falta de implementación no bloquea (regla de línea base) | WAB-78 → WAB-79 |
| CTX-P2 | Resolución de AgentInstance: quién crea, activa, suspende, retira, designa predeterminada | No | Referencia a ARQ-19 (actor sin definir, bloqueo D2 preexistente) | WAB-78 (P3) → WAB-80 |
| CTX-P3 | Composición determinista de configuración efectiva (precedencia definición → instancia → overrides) | **Sí** | Sin reglas de combinación documentadas no se puede definir configuración efectiva determinista | WAB-78 |
| CTX-P4 | Precedencia entre valores de plataforma y configuración de instancia | **Sí** | Sin distinción entre valores de plataforma y de instancia no se puede definir contrato de configuración efectiva | WAB-78 |
| CTX-P5 | Momento exacto de creación del snapshot | No | P10, C.7.1, ADR-005 ya definen el contrato; momento exacto es detalle de implementación | WAB-79/WAB-80 |
| CTX-P6 | Persistencia y asociación del snapshot con Execution | No | C.7.2 ya define effectiveConfigSnapshot; campos específicos son detalle de implementación | WAB-79/WAB-80 |
| CTX-P7 | Identificación de versiones/revisiones usadas (AgentDefinition.version, Rule.version, Skill.version) | No | C.7.1 ya define contrato de versiones; **queda por definir qué entidad o catálogo proporciona la versión inmutable de Skills** (observación C8 de D4-A) | WAB-78/WAB-79 |
| CTX-P8 | Comportamiento ante configuración inexistente/inválida/inconsistente | No | Puede definirse como contrato sin bloquear diseño | WAB-78 |
| CTX-P9 | Exclusión de secretos del snapshot y de los logs | No | Credenciales excluidas de toJSON; política de logs es contrato separado (D5-C) | WAB-78 / D5-C |
| CTX-P10 | Rol de conocimiento en el prompt (user vs knowledge dedicado) | No | ADR-003 ya aprobó; implementación pendiente | WAB-79 |
| CTX-P11 | Trazabilidad RAG Nivel 2 (IDs de fragmentos, versiones, hashes por ejecución) | No | C.7.3 Nivel 2 es opcional en AgentExecutionLog | WAB-79/WAB-80 |
| CTX-P12 | Ventana de memoria (20 mensajes vs 30 MemoryFact) | No | Desacuerdo as-is vs diseño; detalle de implementación | WAB-79 |
| CTX-P13 | Manejo de errores de Qdrant ([] silencioso vs error visible) | No | E.1 paso 6 vs implementación actual; detalle de implementación | WAB-79 |

**Contratos bloqueantes de diseño:** CTX-P3, CTX-P4 (precedencia y combinación de configuración efectiva).
**Contrato pendiente crítico de observación C8:** CTX-P7 (qué entidad o catálogo proporciona versión inmutable de Skills — no asumir solución sin evidencia).

---

## 13. Hallazgos candidatos y clasificación

Cada hallazgo usa identificador provisional `D4B-Cxx`. Ninguno duplica ARQ-01 a ARQ-22 ni D4A-C01 a D4A-C13.

### D4B-C01 — No existe separación AgentDefinition / AgentInstance en el runtime

| Campo | Contenido |
|---|---|
| **ID** | D4B-C01 |
| **Dominio** | Modelo de dominio / Construcción de contexto |
| **Estado** | Ausencia de implementación |
| **Evidencia** | `crm/src` y `crm/microservices/ai-service/src`: grep `AgentDefinition` = 0; `AgentInstance` = 0 como entidad; `ResourceOwnership.ts:33-45` sin `AGENT_INSTANCE`; `permission.enum.ts` sin `platform.*` |
| **Comportamiento observado** | El runtime resuelve todo a partir de `Bot` + `BotInstance` en ai-service. `Bot` mezcla definición (provider, modelo, credencial, capabilities) e instancia. No hay entidad separada de definición propiedad de Waby ni de instancia propiedad del Workspace. |
| **Contrato objetivo afectado** | CTX-P1 (resolución AgentDefinition), CTX-P2 (resolución AgentInstance) |
| **Impacto arquitectónico** | Alto: el modelo objetivo no tiene contraparte implementada; WAB-79 debe definir el modelo desde cero |
| **Bloquea WAB-78** | No (ausencia de implementación esperada antes de WAB-79; no impide definir contratos de resolución) |
| **Fase de destino** | WAB-78 (contrato) → WAB-79 (modelo) |
| **Criterio de cierre** | Entidades AgentDefinition y AgentInstance definidas en el modelo objetivo con propiedad y cardinalidad claras |

### D4B-C02 — System prompt fallback hardcodeado sin caer al prompt del bot

| Campo | Contenido |
|---|---|
| **ID** | D4B-C02 |
| **Dominio** | Prompt y PromptVersion |
| **Estado** | Implementación parcial (desacuerdo con diseño objetivo) |
| **Evidencia** | `bot-instance.ts:264`: `systemPrompt: this._systemPrompt || 'You are a helpful assistant'`; fallback genérico hardcodeado, no cae a `bot.systemPrompt` |
| **Comportamiento observado** | Cuando la instancia no tiene `systemPrompt`, se usa un texto genérico hardcodeado en lugar del `systemPrompt` del bot asociado. No existe `systemPromptOverride` por instancia (campo del modelo objetivo, no implementado). |
| **Contrato objetivo afectado** | CTX-09 (precedencia system prompt), diseño objetivo `AgentDefinition.systemPrompt` + `AgentInstance.systemPromptOverride` |
| **Impacto arquitectónico** | Medio: el prompt efectivo puede no reflejar la intención del bot definido |
| **Bloquea WAB-78** | No (detalle de combinación de prompts, definible en WAB-78) |
| **Fase de destino** | WAB-78 (contrato CTX-P3/CTX-P4) → WAB-79 (implementación) |
| **Criterio de cierre** | Reglas de precedencia de system prompt documentadas: `AgentInstance.systemPromptOverride` sobre `AgentDefinition.systemPrompt` |

### D4B-C03 — Rules sin versión inmutable ni entidad AgentRules implementada

| Campo | Contenido |
|---|---|
| **ID** | D4B-C03 |
| **Dominio** | Rules |
| **Estado** | Ausencia de implementación |
| **Evidencia** | `LoadRulesSkillsStep` carga desde `crm.bot_resources` / `crm.bot_resource_links` (cross-DB CRM); no existe entidad `AgentRules` en código (grep 0); `AgentRules.version` existe en canónico A.2 pero no implementado |
| **Comportamiento observado** | Las Rules se cargan como recursos con `type === 'rule'` y `enabled`, con verificación de hash SHA-256 de integridad, pero sin versión semántica ni inmutable. No hay forma de identificar qué versión de una Rule se usó en una ejecución. |
| **Contrato objetivo afectado** | CTX-P7 (versiones de Rule en snapshot), canónico C.7.1 |
| **Impacto arquitectónico** | Medio: sin versión de Rule no hay trazabilidad completa de qué configuración de reglas recibió el LLM |
| **Bloquea WAB-78** | No (contrato de versión ya definido en C.7.1; implementación en fases posteriores) |
| **Fase de destino** | WAB-78 (contrato CTX-P7) → WAB-79 (modelo AgentRules con versión) |
| **Criterio de cierre** | `AgentRules.version` implementado; la ejecución referencia la versión de Rule usada |

### D4B-C04 — Skills sin versión inmutable ni entidad SkillConfiguration implementada (observación C8 de D4-A)

| Campo | Contenido |
|---|---|
| **ID** | D4B-C04 |
| **Dominio** | Skills |
| **Estado** | Ausencia de implementación |
| **Evidencia** | No existe entidad `SkillConfiguration` en código (grep 0); `bot_resources` tiene campo `hash` (integridad de contenido) pero no versión semántica; canónico A.2 `SkillConfiguration` no declara campo `version`; `AgentRules` sí declara `version` |
| **Comportamiento observado** | Las Skills se cargan como recursos con `type === 'skill'` y `enabled`. No hay identificación inmutable de versión de Skill. La observación C8 de D4-A requiere identificar versiones de Skills en el contrato de snapshot, pero no hay entidad o catálogo que proporcione esa versión. |
| **Contrato objetivo afectado** | CTX-P7 (qué entidad o catálogo proporciona versión inmutable de Skills — observación C8 de D4-A) |
| **Impacto arquitectónico** | Medio: sin versión inmutable de Skills no se puede garantizar trazabilidad de qué skill recibió el LLM; requiere definir entidad o catálogo |
| **Bloquea WAB-78** | No (falta de implementación no bloquea por regla de línea base; pero CTX-P7 es un contrato de diseño pendiente) |
| **Fase de destino** | WAB-78 (contrato CTX-P7, sin asumir solución) → WAB-79 (modelo/catálogo) |
| **Criterio de cierre** | Definida la entidad o catálogo que provee versión inmutable de Skills; la ejecución referencia la versión usada |

### D4B-C05 — No hay precedencia documentada entre definición e instancia (C3/C4)

| Campo | Contenido |
|---|---|
| **ID** | D4B-C05 |
| **Dominio** | Configuración efectiva |
| **Estado** | Ausencia de documentación / implementación |
| **Evidencia** | `bot-instance.resolveEffectiveConfig(bot)` (`bot-instance.ts:240-272`): fallback implícito `this._x || bot.x` sin reglas documentadas. Canónico A.2 define `defaultSkills`/`defaultRules` en definición y `skills`/`rules`/`systemPromptOverride` en instancia, pero sin semántica de copia/combinación/sustitución definida |
| **Comportamiento observado** | Campos de `BotInstance` sobrescriben a `Bot` con fallback implícito. `knowledgeBaseIds` siempre del Bot. `systemPrompt` cae a hardcodeado, no a `bot.systemPrompt`. No hay documento explícito de reglas de combinación entre definición e instancia. |
| **Contrato objetivo afectado** | CTX-P3 (composición determinista), CTX-P4 (precedencia plataforma vs instancia) |
| **Impacto arquitectónico** | Alto: sin reglas de combinación no se puede definir la configuración efectiva determinista; `defaultSkills`/`defaultRules` vs `skills`/`rules` y `systemPromptOverride` quedan sin semántica definida |
| **Bloquea WAB-78** | **Sí** (CTX-P3 y CTX-P4 son contratos fundamentales de WAB-78) |
| **Fase de destino** | WAB-78 |
| **Criterio de cierre** | Reglas de precedencia y combinación documentadas y aprobadas, incluyendo si `defaultSkills`/`defaultRules` se copian/combinan/sustituyen en la instancia, y si `systemPromptOverride` reemplaza o complementa el prompt base |

### D4B-C06 — Qdrant falla silenciosamente a `[]` (desacuerdo con canónico E.1)

| Campo | Contenido |
|---|---|
| **ID** | D4B-C06 |
| **Dominio** | Knowledge / RAG |
| **Estado** | Implementación parcial (desacuerdo con diseño objetivo) |
| **Evidencia** | `rag-service.ts:77-79`: catch externo retorna `[]`; `rag-service.ts:66-68`: catch interno por KB hace `continue`; Fase 1 y D4-A documentan fallo silencioso. Canónico E.1 paso 6 (L920-921): "Qdrant error → error visible, no [] silencioso" |
| **Comportamiento observado** | Si Qdrant cae o una KB falla, el paso devuelve `[]` sin error visible al canal. El pipeline continúa sin knowledge context. |
| **Contrato objetivo afectado** | CTX-P13 (manejo de errores de Qdrant), canónico E.1 paso 6 |
| **Impacto arquitectónico** | Medio: el agente responde sin conocimiento sin que el sistema lo señale; posible degradación silenciosa de calidad |
| **Bloquea WAB-78** | No (detalle de implementación de RAG; no bloquea definición de contratos) |
| **Fase de destino** | WAB-79 (implementación de manejo de errores RAG) |
| **Criterio de cierre** | Qdrant errors produce error visible (no silencioso) o se documenta explícitamente el comportamiento de degradación |

### D4B-C07 — No se registra trazabilidad RAG por ejecución (IDs de fragmentos, scores, versiones)

| Campo | Contenido |
|---|---|
| **ID** | D4B-C07 |
| **Dominio** | Knowledge / RAG |
| **Estado** | Ausencia de implementación |
| **Evidencia** | `ExecuteRAGStep` guarda `knowledgeFragments` (strings) en contexto pero no IDs, scores ni versiones de documentos/chunks. `PromptDocument.knowledgeContext` almacena solo strings. Canónico C.7.3 (L658-670): `knowledgeBaseIds` identifica bases habilitadas pero **no** registra contenido efectivamente recuperado; Nivel 2 requiere IDs de fragmentos Qdrant, versiones de documentos/chunks, hashes de fragmentos |
| **Comportamiento observado** | No hay registro de qué fragmentos específicos se recuperaron en cada ejecución, ni de sus scores ni versiones. `knowledgeBaseIds` solo indica qué bases estaban disponibles. |
| **Contrato objetivo afectado** | CTX-P11 (trazabilidad RAG Nivel 2), canónico C.7.3 |
| **Impacto arquitectónico** | Medio: no se puede auditar qué contexto RAG recibió exactamente una ejecución pasada |
| **Bloquea WAB-78** | No (Nivel 2 es opcional en AgentExecutionLog) |
| **Fase de destino** | WAB-79/WAB-80 |
| **Criterio de cierre** | Registrados opcionalmente en AgentExecutionLog: IDs de fragmentos Qdrant, versiones de documentos/chunks, hashes de fragmentos utilizados |

### D4B-C08 — Ventana de memoria: 30 MemoryFact vs 20 mensajes (desacuerdo con canónico E.1)

| Campo | Contenido |
|---|---|
| **ID** | D4B-C08 |
| **Dominio** | Memoria |
| **Estado** | Desacuerdo as-is vs diseño objetivo |
| **Evidencia** | `memory-manager.ts:7`: `MAX_MEMORY_FACTS = 30`. Canónico E.1 paso 8 (L935-936): "ConversationMemory: últimos 20 mensajes user/assistant, sin resumen automático" |
| **Comportamiento observado** | Se cargan hasta 30 MemoryFact activos y no expirados por contacto. El canónico diseña ventana de 20 mensajes user/assistant. |
| **Contrato objetivo afectado** | CTX-P12 (ventana de memoria), canónico E.1 paso 8 |
| **Impacto arquitectónico** | Bajo-medio: diferencia de cantidad de contexto de memoria incorporado al prompt |
| **Bloquea WAB-78** | No (detalle de implementación de memoria) |
| **Fase de destino** | WAB-79 (implementación de ConversationMemory ventana 20) |
| **Criterio de cierre** | Ventana de memoria alineada con el diseño objetivo (20 mensajes o 30 hechos, según decisión de producto) |

### D4B-C09 — No existe snapshot inmutable de configuración efectiva por ejecución (P10 no implementado)

| Campo | Contenido |
|---|---|
| **ID** | D4B-C09 |
| **Dominio** | Runtime / P10 / Trazabilidad |
| **Estado** | Ausencia de implementación |
| **Evidencia** | `execution.ts`: sin `effectiveConfigSnapshot`, `effectiveConfigHash`, `promptSnapshotHash`; `execution-context.ts`: sin snapshot; `effective-config.ts`: objeto congelado pero no persistido; `createExecution` (`execution-engine.ts:153-164`) no asigna provider, model, promptVersionId |
| **Comportamiento observado** | La configuración se resuelve al inicio y se congela en memoria, pero no se captura copia inmutable ni hash para persistencia. No hay forma de saber qué configuración usó una ejecución pasada. |
| **Contrato objetivo afectado** | CTX-04, CTX-05, CTX-06 (P10, C.7.1, C.7.2, ADR-005) |
| **Impacto arquitectónico** | Medio: P10 ya define el contrato de snapshot y hashes; la brecha es de implementación, no de definición de diseño |
| **Bloquea WAB-78** | No (contrato de P10 aprobado; implementación en fases posteriores) |
| **Fase de destino** | WAB-78 (contrato ya definido) → WAB-79/WAB-80 (implementación) |
| **Criterio de cierre** | Execution persiste `effectiveConfigSnapshot` (JSON sin secretos) y `effectiveConfigHash` al iniciar cada ejecución; `promptSnapshotHash` calculado y persistido |

### D4B-C10 — PromptDocument no incorpora versiones de Rules/Skills ni IDs de fragmentos knowledge

| Campo | Contenido |
|---|---|
| **ID** | D4B-C10 |
| **Dominio** | BuildPromptStep / PromptDocument |
| **Estado** | Ausencia de implementación |
| **Evidencia** | `prompt-document.ts`: `rulesContext` y `skillsContext` tienen `{resourceId, name, content}` sin versión; `knowledgeContext` es `string[]` sin IDs ni scores. `PromptDocument.toJSON()` no incluye versiones de recursos ni metadata RAG. |
| **Comportamiento observado** | El PromptDocument contiene el contenido de Rules/Skills/knowledge pero no sus versiones ni identificadores de fragmentos recuperados. No hay trazabilidad de qué versión de cada recurso se incorporó. |
| **Contrato objetivo afectado** | CTX-P7 (versiones de Rule/Skill), CTX-P11 (trazabilidad RAG Nivel 2), canónico C.7.1/C.7.3 |
| **Impacto arquitectónico** | Medio: sin versiones de recursos en el PromptDocument no se puede reproducir exactamente qué vio el modelo |
| **Bloquea WAB-78** | No (definible como parte del contrato de snapshot en WAB-78) |
| **Fase de destino** | WAB-78 (contrato) → WAB-79 (implementación de versionado en PromptDocument) |
| **Criterio de cierre** | PromptDocument incluye versión de cada Rule/Skill y, opcionalmente, IDs de fragmentos knowledge para trazabilidad RAG |

### D4B-C11 — Memoria sin aislamiento por conversación ni visibilidad diferenciada

| Campo | Contenido |
|---|---|
| **ID** | D4B-C11 |
| **Dominio** | Memoria |
| **Estado** | Implementación actual (posible riesgo de mezclado de contextos) |
| **Evidencia** | `memory-manager.ts:42-46`: `findActiveByContact(workspaceId, contactId, MAX_MEMORY_FACTS)` filtra solo por workspaceId + contactId. `MemoryVisibility.SHARED` para todos los hechos (`memory-persister.ts:149`). `conversationId` se almacena pero no se usa para filtrado en carga. |
| **Comportamiento observado** | Si el mismo contacto tiene múltiples conversaciones, los hechos se comparten entre todas (no hay aislamiento por conversación). Visibilidad siempre SHARED. |
| **Contrato objetivo afectado** | Aislamiento de contexto de ejecución (riesgo diferido a D5-C, registrar como observación) |
| **Impacto arquitectónico** | Medio: hechos de una conversación pueden influir en otra conversación del mismo contacto |
| **Bloquea WAB-78** | No (diferido a D5-C; observar sin ampliar alcance) |
| **Fase de destino** | Observación diferida (D5-C) |
| **Criterio de cierre** | Decisión de producto sobre aislamiento de memoria por conversación/visibilidad |

---

## 14. Riesgos, dependencias y aspectos no verificados

### 14.1 Riesgos identificados

| Riesgo | Descripción | Severidad | Mitigación / nota |
|---|---|---|---|
| R1 | Sin separación AgentDefinition/AgentInstance: el runtime no distingue valores de plataforma de valores de instancia | Alto | Bloquea definición de configuración efectiva determinista (CTX-P3/P4) |
| R2 | Sin versión inmutable de Skills: no se puede identificar qué versión de Skill recibió el LLM (observación C8 de D4-A) | Medio | Requiere definir entidad o catálogo que provea versión inmutable |
| R3 | Qdrant falla silenciosamente a `[]`: degradación de calidad sin señal visible | Medio | Desacuerdo con canónico E.1 paso 6 |
| R4 | Sin snapshot persistido (P10): no hay trazabilidad de configuración efectiva por ejecución | Medio | Contrato definido, implementación pendiente |
| R5 | Memoria sin aislamiento por conversación: hechos compartidos entre conversaciones del mismo contacto | Medio | Diferido a D5-C |
| R6 | Reglas de combinación definición→instancia no documentadas: comportamiento ambiguo de overrides | Alto | Bloqueante de diseño (CTX-P3/P4) |

### 14.2 Dependencias de D4-B con otros bloques/fases

- **D1 (dominio/ownership):** ARQ-14 (propiedad de persistencia Knowledge sin declarar) afecta a dónde residen WorkspaceKnowledge y los vectores.
- **D2 (permisos):** ARQ-19/ARQ-20/ARQ-21 (actor de ciclo de vida, permisos, identidad de servicio) afectan a quién puede editar Rules/Skills/Knowledge.
- **D3 (routing/asignaciones):** ARQ-05/ARQ-06/ARQ-07/ARQ-08 (EvAssigned, atomicidad, cancelación, validación contra CRM) afectan a la vigencia del contexto durante la ejecución.
- **D4-A (runtime):** D4A-C01 a D4A-C13 (separación, snapshot, hashes, versionado) son directamente relevantes; D4B no los duplica.
- **D5-A (Policy Engine/output filter):** no in scope; observación de que el canónico prevé Policy Engine prevaleciendo sobre Rules/Skills/Knowledge (canónico E.1 paso 9).
- **D5-C (aislamiento multitenant):** D4B-C11 (aislamiento de memoria) se observa como diferido.
- **WAB-79:** definición de modelos AgentDefinition, AgentInstance, SkillConfiguration, AgentRules, ConversationMemory, WorkspaceKnowledge; índices; versión inmutable de Skills (CTX-P7).
- **WAB-80:** implementación de provisioning, permisos, identidad de servicio.

### 14.3 Aspectos no verificados (limitaciones de evidencia)

- No se verificó el comportamiento real de Qdrant en producción (fallos, latencia, umbrales reales).
- No se verificó el contenido de `.env` ni valores de `CRM_API_KEY` / `AI_SERVICE_INTERNAL_TOKEN` en producción (ARQ-21).
- No se verificaron índices de MongoDB Atlas para las colecciones involucradas (deferido a D6).
- No se pudo consultar el contenido directo de Linear (plan de revisión, registro de decisiones).
- No se inspeccionó en detalle la implementación del repositorio de memoria (`IMemoryFactRepository` implementación).
- No se inspeccionaron los extractores de documentos (PDF, DOCX) en detalle de funcionamiento.

---

## 15. Conclusión y veredicto de D4-B

### 15.1 Resumen de lo verificado

El flujo de construcción de contexto está implementado en ai-service como un pipeline de 8 pasos bien estructurado, con `BuildPromptStep` componiendo un `PromptDocument` congelado e inmutable tras su construcción. Existe presupuesto de tokens con prioridad clara (system y usuario son requeridos; knowledge, memoria e historial son opcionales ajustados al presupuesto). La inmutabilidad en memoria durante la ejecución está garantizada por `Object.freeze` en `EffectiveConfig`, `ModelConfiguration` y `PromptDocument`.

### 15.2 Brechas principales frente al diseño objetivo

1. **No existe separación AgentDefinition (Waby) / AgentInstance (Workspace)** — el runtime usa `Bot`/`BotInstance` mixtos.
2. **No existe snapshot inmutable persistido** — P10 aprobado pero no implementado (no hay `effectiveConfigSnapshot`, `effectiveConfigHash`, `promptSnapshotHash`).
3. **No existe versión inmutable de Skills** — observación C8 de D4-A sin entidad o catálogo que la proporcione (CTX-P7, no asumir solución).
4. **Reglas de combinación definición→instancia no documentadas** — CTX-P3/P4 bloqueantes de diseño.
5. **Qdrant falla silenciosamente a `[]`** — desacuerdo con canónico E.1 paso 6.
6. **No hay trazabilidad RAG Nivel 2** — no se registran IDs de fragmentos, scores ni versiones por ejecución.

### 15.3 Veredicto de D4-B

**D4-B no está listo para aprobación.** El bloque tiene su diagnóstico completado, pero presenta brechas de implementación significativas y dos contratos de diseño bloqueantes (CTX-P3, CTX-P4) que requieren definición en WAB-78 antes de que WAB-79 pueda definir el modelo de AgentInstance con reglas de combinación claras.

La ausencia de implementación (snapshot, separación AgentDefinition/AgentInstance, versión de Skills) **no bloquea** WAB-78 por regla de la línea base, siempre que los contratos objetivo estén definidos. Los contratos CTX-P3 y CTX-P4 (precedencia y combinación de configuración efectiva) sí requieren definición explícita en WAB-78 para no trasladar decisiones críticas a WAB-79.

### 15.4 Recomendaciones de cierre (sin implementar)

- Definir CTX-P3 y CTX-P4 (reglas de precedencia y combinación definición → instancia) en WAB-78.
- Definir CTX-P7 (entidad o catálogo que provee versión inmutable de Skills) sin asumir solución específica.
- Documentar desacuerdos as-is vs diseño: Qdrant error handling, ventana de memoria, system prompt fallback, rol de conocimiento.
- No duplicar ARQ-01 a ARQ-22 ni D4A-C01 a D4A-C13 en los hallazgos D4B-Cxx.

**D4-B queda sujeto a verificación independiente. No se declara cerrado.**

---

## Informe de finalización

1. **Secciones creadas:** 15 secciones (resumen ejecutivo, alcance, fuentes, topología, PromptVersion, Rules, Skills, Knowledge/RAG, memoria, BuildPromptStep/PromptDocument, comparación as-is vs objetivo, contratos, hallazgos, riesgos, conclusión/veredicto).
2. **Fuentes y commit utilizados:** Repositorio `F:\ElemenSoft\crm`, commit HEAD `b9e0aee594bf7c4ffd1e88c84988b99e54b7d972` (origin/master), estado limpio. Fuentes: canónico `wab-75-fase-2-diseno.md`, informe Fase 1, línea base, matriz acumulativa, informes D1-D4-A, complemento técnico, código fuente ai-service y CRM.
3. **Hallazgos principales:** D4B-C01 a D4B-C11 (ausencia de separación AgentDefinition/AgentInstance, system prompt fallback hardcodeado, Rules/Skills sin versión inmutable, precedencia no documentada, Qdrant fallo silencioso, falta trazabilidad RAG, ventana de memoria discrepante, falta snapshot P10, PromptDocument sin versiones de recursos, memoria sin aislamiento por conversación).
4. **Contratos realmente pendientes:** CTX-P1 a CTX-P13; bloqueantes de diseño CTX-P3 y CTX-P4; observación C8 crítica CTX-P7 (versión inmutable de Skills).
5. **Bloqueos de diseño vs. brechas de implementación:** Bloqueos de diseño: CTX-P3, CTX-P4 (precedencia/combinación). Brechas de implementación (no bloquean por regla de línea base): snapshot P10, separación AgentDefinition/AgentInstance, versión de Skills, trazabilidad RAG, manejo de errores Qdrant.
6. **Limitaciones de evidencia:** sin acceso a Linear directo, sin `.env`/producción, sin verificación Atlas, repositorio de memoria y extractores no inspeccionados en detalle.
7. **Ruta, tamaño y fecha del informe:** `F:\ElemenSoft\docs\revision-wab-78\informe-d4-b-contexto.md`, creado 05/11/2026.
8. **Confirmación:** No se modificó ningún otro artefacto ni estado (solo lectura; no se modificó código, schemas, modelos, tests, configuración, documento canónico, línea base, matriz acumulativa, ni Linear).








