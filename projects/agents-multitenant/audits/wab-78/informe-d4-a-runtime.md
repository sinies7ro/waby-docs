> **Nota de vigencia — 07/11/2026:** este informe mantiene su alcance de auditoría runtime. ARQ-14 queda resuelto en D.1.9/B del diseño canónico. La fila D4A-C12 que remite a “contrato definido en D3/WAB-81” queda superada: D3 había identificado la brecha, pero no había documentado el contrato integral. ARQ-22 queda especificado en D.6; la implementación se difiere a WAB-81. Esta nota no afirma que las brechas estén corregidas en código.

---

# WAB-78 — Informe D4-A: Auditoría de Runtime y configuración efectiva

**Fecha:** 05/11/2026 · **Bloque:** D4-A (Runtime y configuración efectiva) · **Modo:** Solo lectura y diagnóstico
**Repositorio:** `F:\ElemenSoft\crm` · **Commit HEAD auditado:** `b9e0aee594bf7c4ffd1e88c84988b99e54b7d972` (origin/master)
**Carpeta de informe:** `C:\Users\sinies7ro\Dropbox\WABY\docs\revision-wab-78\informe-d4-a-runtime.md` (ruta Dropbox sincronizada verificada)

---

## 1. Resumen ejecutivo

Esta auditoría determina cómo funciona actualmente el runtime de agentes, cómo se resuelve y construye la configuración efectiva de una ejecución, y si la implementación permite garantizar el snapshot inmutable aprobado en P10. No se implementan soluciones; se produce evidencia para cerrar el diseño de WAB-78.

**Conclusión principal:** El runtime actual (ai-service + CRM) opera con el modelo **Bot / BotInstance**, donde el cliente configura proveedor, modelo, temperatura, capabilities y credencial por bot. No existe separación entre AgentDefinition (Waby) y AgentInstance (Workspace). La configuración efectiva se resuelve en tiempo de ejecución mediante `BotInstance.resolveEffectiveConfig(bot)` y se almacena en `EffectiveConfig` (objeto congelado en memoria). Existe `ModelConfiguration` y `EffectiveConfig` congelados tras su creación, por lo que una ejecución ya iniciada conserva su configuración en memoria; los cambios posteriores en Bot/BotInstance **no** alteran esa ejecución en curso. Sin embargo, no existe snapshot durable, ni hashes de configuración ni de prompt, ni trazabilidad histórica persistida: no hay forma de saber qué configuración usó una ejecución pasada, ni de reproducirla ni auditarla con la información persistida.

Se identificaron **13 entradas candidatas provisionales (D4A-C01 a D4A-C13)**. Algunas son observaciones propias de D4-A; otras son referencias a hallazgos existentes en ARQ (D4A-C10 → ARQ-07, D4A-C12 → ARQ-22, D4A-C13 → ARQ-19/20/21, D4A-C11 → D5-C). Ninguna duplica ARQ-01 a ARQ-22; se trata de una lista provisional de seguimiento, no de 13 hallazgos nuevos e independientes. Varios bloques WAB-78 dependen de decisiones pendientes (P2, P3, persistencia por entidad, contrato de vigencia contra CRM en G3/G4) que no pueden cerrarse sin evidencia adicional.

---

## 2. Alcance y exclusiones

### 2.1 Alcance (D4-A)

- Topología real del runtime: entrada, consumers, workers, colas, creación de Execution, resolución de Bot/BotInstance, construcción del pipeline, invocación del proveedor LLM, ejecución de capabilities, finalización y publicación.
- Resolución de configuración efectiva: Bot, BotInstance, parámetros de proveedor y modelo, ejecución y pipeline, prompts y overrides, Skills, Rules y capabilities, defaults, precedencias, versiones, persistencia de la configuración utilizada.
- Estado actual de AgentDefinition / AgentInstance frente al modelo objetivo (Waby + Workspace).
- Evaluación de P10 (snapshot inmutable por ejecución) y diferenciación entre configuración, prompt y contexto.
- Escenarios de cambios concurrentes de configuración.
- Contratos que deben quedar definidos en WAB-78.
- Hallazgos candidatos y clasificación.

### 2.2 Exclusiones expresas

No se auditan: D4-B (composición del contexto, PromptDocument, Knowledge/RAG, embeddings, Skills, Rules, memoria), D5-A (Policy Engine, output filter, prompt injection, ARQ-10), D5-B (restricciones de conocimiento externo y confidencialidad de Waby), D5-C (aislamiento multitenant y autorización de capabilities, herramientas, datos sensibles, logs y telemetría). No se repite D1, D2 ni D3. No se reabre ARQ-01 a ARQ-22 salvo evidencia nueva directamente relevante.

---

## 3. Fuentes consultadas y commit HEAD

### 3.1 Fuentes locales consultadas

| Fuente | Ruta | Uso |
|--------|------|-----|
| Documento canónico WAB-78 (Fase 2) | `crm/docs/plans/wab-75-fase-2-diseno.md` | Contrato objetivo: AgentDefinition/AgentInstance, E.1 flujo, C.7 snapshot P10, F.1 jerarquía, G.1-G.4 proveedores/créditos |
| Informe Fase 1 (WAB-77) | `crm/docs/plans/wab-75-fase-1-auditoria.md` | Estado as-is: entidades, flujo, pipeline, RAG, prompt, capabilities, fallos |
| Línea base revisión WAB-78 | `F:\ElemenSoft\docs\revision-wab-78\00-linea-base.md` | Estados de control, decisiones aprobadas, convención de IDs |
| Matriz acumulativa | `F:\ElemenSoft\docs\revision-wab-78\matriz-acumulativa.md` | Hallazgos ARQ-01 a ARQ-22, evitación de duplicación |
| Informe Bloque 1 (D1) | `F:\ElemenSoft\docs\revision-wab-78\informe-bloque-1.md` | Ownership, persistencia, índices, cardinalidades |
| Informe Bloque 2 (D2) | `F:\ElemenSoft\docs\revision-wab-78\informe-bloque-2.md` | Permisos, actores, identidad de servicio |
| Informe Bloque 3 (D3) | `F:\ElemenSoft\docs\revision-wab-78\informe-bloque-3.md` | EvAssigned, atomicidad, cancelación, G3/G4 |
| Complemento técnico WAB-78 | `F:\ElemenSoft\WAB-78-complemento-tecnico.md` | EvAssigned, cancelación, atomicidad, migración |
| Análisis previo D2-D4 | `F:\ElemenSoft\docs\revision-wab-78\analisis-previo-d2-d4.md` | Hallazgos preliminares ARQ-03 a ARQ-10 |
| Código fuente ai-service | `crm/microservices/ai-service/src/` | Runtime, pipeline, pasos, ejecución, configuración |
| Código fuente CRM | `crm/src/` | Contacto, Evento, Outbox, Rabbit consumers, routing |

### 3.2 Commit HEAD

`b9e0aee594bf7c4ffd1e88c84988b99e54b7d972` — estado limpio, sin cambios pendientes en la copia de trabajo. Toda la evidencia de código se extrajo de este commit.

### 3.3 Limitaciones de evidencia

- El informe se guardó en `C:\Users\sinies7ro\Dropbox\WABY\docs\revision-wab-78/informe-d4-a-runtime.md` (ruta Dropbox sincronizada verificada). Una copia de trabajo permanece en `F:\ElemenSoft\docs\revision-wab-78/informe-d4-a-runtime.md`.
- No se pudo consultar el contenido directo de Linear (plan de revisión `fca2514ac4ee`, registro de decisiones `345541575496`); se utilizó la documentación local como fuente de continuidad.
- No hay acceso a `.env` ni a producción; no se puede verificar si `CRM_API_KEY` es un JWT válido ni el comportamiento real del despliegue (ARQ-21).
- No se verificaron índices contra una instancia real de MongoDB Atlas (deferido a D6).

---

## 4. Topología real del runtime

### 4.1 Flujo de entrada y recepción

**Punto de entrada:** RabbitMQ, cola `ai-service.message-received` (durable, prefetch=1). Consumida por `MessageReceivedHandler` (`crm/microservices/ai-service/src/infrastructure/messaging/consumers/message-received-handler.ts`).

El payload validado por Zod (`message-received-handler.ts:21-36`) contiene: `workspaceId`, `conversationId`, `contactId`, `botId`, `botInstanceId` (opcional), `content`, `history` (opcional), `traceId`, `configId` (opcional), `source` (opcional). El `executionId` se genera aleatoriamente en el handler (`exec-${Date.now()}-${Math.random()...}`) sin deduplicación persistente por `originMessageId`.

**Resolución de instancia (ruta RabbitMQ productiva):** `MessageReceivedHandler` llama a `BotRuntimeResolver.resolve(botId)` (`message-received-handler.ts:104`). El resolver **no recibe** el `botInstanceId` del payload. Selecciona la primera `BotInstance` activa, sin comprobar `isDefault`. Si no encuentra una instancia activa o falla la consulta, utiliza una configuración sintética `default-{bot.id}`. Posteriormente, `buildContext` (`message-received-handler.ts:194-250`) preestablece `config` y `bot` en el `ExecutionContext`, por lo que `LoadConfigurationStep` omite la resolución en esta ruta productiva (solo aplica el gate de credencial). Este comportamiento se relaciona con **ARQ-13** (`isDefault` sin regla de unicidad ni mecanismo atómico de selección), sin crear un ARQ duplicado.

### 4.2 Consumers, workers y colas

| Componente | Archivo | Responsabilidad |
|------------|---------|-----------------|
| MessageReceivedHandler | `message-received-handler.ts` | Consume RabbitMQ, valida payload, ejecuta G1 (vigencia), construye ExecutionContext, llama a ProcessMessageService |
| ExecutionEngine | `execution-engine.ts` | Crea Execution, ejecuta pipeline en bucle, G2 (revalidación), publica resultado |
| Pipeline | `pipeline.ts` | Ejecuta pasos secuencialmente con cancelación cooperativa entre pasos |
| CapabilityExecutor | `execution-engine.ts:194` | Ejecuta tool calls devueltas por el proveedor LLM |
| eventPublisher | `rabbitmq-event-publisher.ts` | Publica `EventTypes.EXECUTION_COMPLETED` en exchange `ai-service.events` |

**Límites entre servicios:** ai-service es propietario de la ejecución (pipeline, LLM, capabilities). CRM consume `execution.completed` (G3, `executionCompleted.consumer.ts`) para validar vigencia y despachar por WhatsApp. Difusion ejecuta G4 (outbound interactivo) antes del POST final.

### 4.3 Creación y carga de Execution

`ExecutionEngine.process()` (`execution-engine.ts:57-129`):
1. `createExecution(context)` → entidad `Execution` (estado `pending`) con `id`, `workspaceId`, `botId`, `botInstanceId`, `conversationId`, `contactId`, `channel`, `correlationData`.
2. `execution.markRunning()` + `executionRepo.save(execution)`.
3. Carga capabilities globalmente (`capabilityRegistry.findAll()`, sin filtro por instancia).
4. `runPipelineLoop()` ejecuta el pipeline.
5. G2: `vigenciaService.check()` antes de `markCompleted`; si inválido → `context.cancel()` + `execution.cancel()`.
6. `markCompleted(response.content)` o `markFailed(error)`.
7. `finally`: `accumulateTokens`, `accumulateCost`, `save`, `memoryPersister.persist` (si no cancelada), `publishResult`.

### 4.4 Resolución de Bot/BotInstance y configuración asociada

**Resolución de instancia (ruta RabbitMQ productiva):** `BotRuntimeResolver.resolve(botId)` carga `Bot` + `BotInstance` desde la DB de ai-service. El resolver **no recibe** `botInstanceId` del payload; selecciona la primera `BotInstance` activa sin comprobar `isDefault`. Si no encuentra una instancia activa o falla la consulta, utiliza una configuración sintética `default-{bot.id}`. No valida `workspaceId` del payload contra el bot (Fase 1 §4.1, hallazgo #4). Este comportamiento se relaciona con **ARQ-13** (`isDefault` sin regla de unicidad ni mecanismo atómico de selección), sin crear un ARQ duplicado.

**Carga de configuración:** `LoadConfigurationStep` (`load-configuration.step.ts:36-66`):
- Si `context.config` ya está seteado (camino Rabbit, preestablecido por `buildContext`), solo aplica el gate de credencial y omite la resolución de configuración.
- Si no (ruta no-Rabbit), carga `Bot` por `botId` y `BotInstance` por `botInstanceId`.
- Resuelve `EffectiveConfig` mediante `botInstance.resolveEffectiveConfig(bot)` (combinación de campos propios con fallback a Bot).
- Valida que el provider exista en `ProviderManager`.
- `assertCredentialGate`: el bot debe tener su propia API key cifrada para el proveedor (modo estricto, D*-cred); sin ella se aborta con `BotCredentialMissingError`.

### 4.5 Construcción del pipeline y ejecución de steps

Orden del pipeline (`execution-engine.ts:69`, Fase 1 §4.2):
1. `LoadConfigurationStep` — configura EffectiveConfig, gate de credencial.
2. `LoadConversationStep` — historial vacío o hasta 30 facts (sin límite de 20 mensajes).
3. `ResolvePromptStep` — resuelve `PromptVersion` (runtime override `activePromptVersionId` o publicado).
4. `LoadMemoryStep` — memoria de hechos por contacto.
5. `LoadRulesSkillsStep` — carga Rules y Skills desde `crm.bot_resources` / `crm.bot_resource_links` (cross-DB read-only).
6. `ExecuteRAGStep` — búsqueda Qdrant filtrada por `config.workspaceId` (BR-KB-001); falla silenciosamente a `[]` si Qdrant cae.
7. `BuildPromptStep` — construye `PromptDocument` con system prompt + Rules + Skills + knowledge + memory + history + user message.
8. `InvokeProviderStep` — invoca al proveedor LLM con API key del bot descifrada.

**Capabilities:** `LoadConfigurationStep` carga `capabilityRegistry.findAll()` → todas las capabilities, sin filtrar por `enabledCapabilities` del bot/instancia (Fase 1 §4.1, hallazgo #5). `InvokeProviderStep` usa `context.availableCapabilities` como `tools` en la petición al proveedor.

### 4.6 Invocación del proveedor LLM

`InvokeProviderStep` (`invoke-provider.step.ts:39-121`):
- Lee `config.model.provider`, `config.model.name`, `config.model.temperature`, `config.model.maxTokens`.
- Descipta la API key del bot (`bot.credential`, AES-256-GCM) mediante `ICredentialCipher`.
- Resuelve el proveedor vía `providerManager.resolve(providerName, apiKey)`.
- Construye `AIProviderRequest` con `promptDocument`, `model`, `temperature`, `maxTokens`, `tools` (availableCapabilities).
- Invoca `provider.invoke(request)` **sin AbortController ni abort signal** (no hay cancelación física de llamadas LLM).
- Acumula tokens y costo desde `response.usage` / `response.cost`.

### 4.7 Ejecución de capabilities en el runtime

En `runPipelineLoop` (`execution-engine.ts:166-200`), tras recibir `toolCalls` del proveedor:
1. `context.addToolCalls(response.toolCalls)`.
2. `capabilityExecutor.execute(response.toolCalls, context)` → ejecuta capabilities.
3. `context.addToolResults(toolResults)`.
4. Vuelve al inicio del bucle (máximo 5 iteraciones).

**Brecha:** `runPipelineLoop` no revalida `context.cancelled` ni la vigencia antes de ejecutar capabilities tras `pipeline.execute(context)` (Bloque 3, ARQ-07).

### 4.8 Finalización, persistencia y publicación del resultado

- `markCompleted(result)` o `markFailed(error)` (solo desde RUNNING).
- `publishResult` (`execution-engine.ts:202-234`): publica `EventTypes.EXECUTION_COMPLETED` con `execution.toJSON()` en RabbitMQ **independientemente del estado cancelado** (publicación semánticamente mal etiquetada).
- CRM G3 (`executionCompleted.consumer.ts`): valida vigencia contra `contact_assignments` de ai-service; descarta si no válido; si válido, persiste `EvMensaje` y despacha por WhatsApp.
- `memoryPersister.persist(context)` si la ejecución no está cancelada.

---

## 5. Resolución y ensamblado de configuración efectiva

### 5.1 Modelo actual: EffectiveConfig

`EffectiveConfig` (`crm/microservices/ai-service/src/domain/value-objects/configuration/effective-config.ts:16-102`):
- Campos: `botId`, `botInstanceId`, `workspaceId`, `model` (ModelConfiguration), `systemPrompt`, `language`, `timezone`, `maxIterations`, `enabledCapabilities`, `knowledgeBaseIds`.
- `ModelConfiguration`: provider, modelo, temperatura, maxTokens, etc.
- Es un objeto congelado (`Object.freeze(this)`) creado una vez por ejecución.
- No hay hash de configuración, no hay versión, no hay snapshot persistido.

### 5.2 Resolución efectiva: `BotInstance.resolveEffectiveConfig(bot)`

La combinación se realiza en `bot-instance.ts:240-272`. `resolveEffectiveConfig(bot)` requiere que `bot` tenga un `PromptVersion` activo (`bot.hasPromptVersion()`), caso contrario lanza error. La precedencia por campo es:

| Campo | Origen en `resolveEffectiveConfig` | Precedencia |
|-------|-----------------------------------|-------------|
| `model` | `this._model || (this._temperature !== null ? bot.model.withOverrides({ temperature: this._temperature }) : bot.model)` | Modelo propio del instance si existe; si solo hay temperatura, se aplica override al modelo del bot; si no hay ninguno, modelo del bot tal cual |
| `temperatura` | Incluida en `resolvedModel` vía `withOverrides` | Sobreescribe la temperatura del modelo del bot solo cuando el instance define temperatura |
| `language` | `this._language || bot.language` | Instance primero, fallback al bot |
| `timezone` | `this._timezone || bot.timezone` | Instance primero, fallback al bot |
| `maxIterations` | `this._maxIterations || bot.maxIterations` | Instance primero, fallback al bot |
| `enabledCapabilities` | `this._enabledCapabilities || bot.enabledCapabilities` | Instance primero, fallback al bot |
| `knowledgeBaseIds` | `bot.knowledgeBaseIds` | **Siempre del bot**, nunca del instance |
| `systemPrompt` | `this._systemPrompt || 'You are a helpful assistant'` | Instance primero; fallback genérico hardcodeado (no cae al `bot.systemPrompt`) |

No hay precedencia documentada explícita entre definición e instancia más allá de este fallback implícito en el código.

### 5.3 Elementos de la configuración efectiva actual

| Elemento | Origen actual | ¿Versionado? | ¿Snapshot? | Notas |
|----------|--------------|-------------|-----------|-------|
| Proveedor | `Bot.modelConfiguration.provider` | No | No | Configurable por bot (cliente) |
| Modelo | `Bot.modelConfiguration.model` (con override de temperatura del instance) | No | No | Configurable por bot (cliente) |
| Temperatura | `BotInstance.temperature` → `bot.model.withOverrides` | No | No | Sobreescribe solo si el instance define temperatura |
| System prompt | `BotInstance.systemPrompt || 'You are a helpful assistant'` | No | No | Fallback genérico hardcodeado, no cae al `bot.systemPrompt` |
| Capabilities | `BotInstance.enabledCapabilities || Bot.enabledCapabilities` | No | No | Cargadas todas vía `capabilityRegistry.findAll()` sin filtro por instancia en el engine |
| Credencial | `Bot.aiCredential` (per-bot, cifrada) | No | No | Excluida de `toJSON` |
| KnowledgeBaseIds | `Bot.knowledgeBaseIds` (siempre del bot) | No | No | El instance no aporta knowledgeBaseIds |
| RAG filtrado | Por `config.workspaceId` (canonical del bot) | No | No | BR-KB-001 |

### 5.4 Prompts y overrides

- `Bot.activePromptVersionId`: runtime override de la versión publicada (Fase 1 §2.1).
- `PromptVersion.promptId`: referencia a `Prompt._id`, índice único parcial para `status='published'`.
- `resolveEffectiveConfig` requiere `bot.hasPromptVersion()`; sin él lanza error.
- `ResolvePromptStep` (`resolve-prompt.step.ts`) resuelve el `PromptVersion` posteriormente en el pipeline (runtime override `activePromptVersionId` o publicado).
- `BuildPromptStep` usa `config.systemPrompt` como fallback legacy si no hay `PromptVersion` resuelta.
- No existe `systemPromptOverride` por instancia (ese campo pertenece al modelo objetivo `AgentInstance`, no al actual).

### 5.5 Defaults, precedencias y reglas de combinación

- Precedencia implícita: campos de `BotInstance` sobrescriben a `Bot` (fallback), sin documento explícito de reglas de combinación.
- Defaults: `language='es'`, `timezone='America/Mexico_City'`, `maxIterations=5` (en `EffectiveConfig` constructor).
- No hay defaults a nivel de plataforma Waby; el proveedor y modelo son elegidos por el cliente por bot.
- No hay identificación de versiones o revisiones de la configuración efectiva utilizada (salvo `PromptVersion` opcional).

### 5.6 Persistencia de la configuración utilizada

`Execution` entity (`execution.ts:19-331`):
- **Campos declarados y serializados en `toJSON`** (`execution.ts:306-329`): `provider`, `model`, `promptVersionId`, entre otros.
- **Campos efectivamente asignados por el flujo de escritura**: `createExecution` en `execution-engine.ts:153-164` solo asigna `id`, `workspaceId`, `botId`, `botInstanceId`, `conversationId`, `contactId`, `channel`, `correlationData`. Los setters `setProvider` (`execution.ts:270-275`), `setModel` (`execution.ts:277-282`) y `setPromptVersion` (`execution.ts:284-286`) existen pero **no se invocan** en el flujo de `ExecutionEngine.process()` ni en los pasos del pipeline inspeccionados. Por tanto, `provider`, `model` y `promptVersionId` permanecen `null` en la persistencia actual.
- **No almacena** `effectiveConfigSnapshot`, `effectiveConfigHash`, ni `promptSnapshotHash`. La configuración utilizada no se persiste de forma inmutable; solo se pueden reconstruir parcialmente desde `botId`/`botInstanceId` si esas entidades no han cambiado.

### 5.7 Comparación con el modelo objetivo AgentDefinition + AgentInstance

| Aspecto | Modelo actual (Bot/BotInstance) | Modelo objetivo (AgentDefinition/AgentInstance) |
|---------|--------------------------------|------------------------------------------------|
| Separación definición/instancia | No existe (Bot mezcla ambas) | Sí: AgentDefinition (Waby) + AgentInstance (Workspace) |
| Proveedor/modelo | Configurable por cliente por bot | Waby central, no configurable por cliente |
| Credencial | Per-bot, cifrada, en Bot | Centralizadas en Waby, nunca expuestas |
| System prompt | Por bot, con PromptVersion opcional | AgentDefinition.systemPrompt (Waby) + AgentInstance.systemPromptOverride (opcional) |
| Capabilities | Globales, todas cargadas | Por instancia (SkillConfiguration) |
| Rules | Por bot (bot_resources) | Por instancia (AgentRules) |
| Knowledge | knowledgeBaseIds en Bot | WorkspaceKnowledge compartido por workspace |
| Versión de configuración | No existe | AgentDefinition.version + AgentInstance |
| Snapshot inmutable | No existe | effectiveConfigSnapshot + effectiveConfigHash |

---

## 6. Estado real de AgentDefinition / AgentInstance frente al modelo objetivo

### 6.1 Existencia en código

- `AgentDefinition`: **no existe** en código (grep 0 en `crm/src` y en `crm/microservices/ai-service/src`).
- `AgentInstance`: **no existe** como entidad en CRM ni en ai-service (confirmado en D1 y D2).
- `ResourceOwnership.ts:33-45`: 12 tipos de recurso, sin `AGENT_INSTANCE`.
- `permission.enum.ts`: sin `platform.*`, `bot.configure`, `wrk.agent.*`, `wrk.credits.*`.

### 6.2 Modelo as-is vs objetivo

El modelo actual `Bot` mezcla definición (provider, modelo, credencial, capabilities globales) e instancia (estado activo/inactivo). `BotInstance` es una sobrecarga opcional con campos sobrescribibles. No hay entidad separada de definición propiedad de Waby ni entidad de instancia propiedad del Workspace con `agentDefinitionId`, `workspaceId`, `status` (draft/active/suspended/retirement_pending/retired), `skills`, `rules`, `knowledgeBaseIds`.

**Inconsistencia documental (sin modificar el canónico):** En la tabla A.2 del documento canónico, `AgentExecution.effectiveConfigSnapshot` se declara como campo opcional (`JSON | No`), mientras que C.7.2 y ADR-005 definen `effectiveConfigSnapshot` como el mecanismo de reproducibilidad técnica de P10. Esta diferencia de tratamiento (opcional vs. requerido por el diseño aprobado) no se resuelve aquí; se registra como observación para la consolidación central.

### 6.3 Impacto en el runtime

El runtime actual resuelve todo a partir de `Bot` + `BotInstance` en ai-service. No hay resolución de `AgentDefinition` (Waby) ni de `AgentInstance` (Workspace) como entidades separadas. El contrato de routing por instancia (`channelAssignment`, `isDefault`, `priority`) no existe en código. El runtime no distingue entre valores de plataforma y valores de instancia porque no existe esa separación.

---

## 7. Evaluación de P10 y snapshot por ejecución

### 7.1 Momento de resolución de la configuración

**Flujo RabbitMQ (ruta productiva):** `MessageReceivedHandler` llama a `BotRuntimeResolver.resolve(botId)` (`message-received-handler.ts:104`), que resuelve `Bot` + `BotInstance` y computa la configuración efectiva *antes* de construir el contexto. `buildContext` (`message-received-handler.ts:194-250`) preestablece `config` y `bot` en el `ExecutionContext`. `LoadConfigurationStep` (`load-configuration.step.ts:37-41`) recibe la configuración ya establecida, omite la resolución de `Bot`/`BotInstance` y solo ejecuta el control de credenciales (`assertCredentialGate`).

**Flujo alternativo (ruta no-Rabbit):** si `context.config` no está seteado, `LoadConfigurationStep` carga `Bot` y `BotInstance` desde la DB (`load-configuration.step.ts:43-51`) y resuelve `EffectiveConfig` mediante `botInstance.resolveEffectiveConfig(bot)` (`load-configuration.step.ts:53`).

En ambos flujos, `EffectiveConfig` se almacena en `ExecutionContext._config` una sola vez (one-time init; `setConfig` lanza si ya está seteado).

### 7.2 ¿Existe snapshot o copia inmutable?

**No existe snapshot durable.** No existe `effectiveConfigSnapshot`, `effectiveConfigHash`, ni `promptSnapshotHash` en el modelo `Execution` ni en `ExecutionContext`. La ejecución no captura una copia duradera de la configuración efectiva para persistencia.

**Inmutabilidad en memoria durante la ejecución:** `EffectiveConfig` y `ModelConfiguration` son objetos congelados (`Object.freeze(this)`) creados una vez por ejecución. Los pasos posteriores del pipeline leen `context.config` (la `EffectiveConfig` resuelta al inicio; en la ruta RabbitMQ, ya preestablecida por `buildContext`); dado que el objeto es inmutable, no puede mutarse durante la ejecución. Por tanto, **los cambios posteriores en `Bot` o `BotInstance` en la DB no alteran una ejecución ya iniciada** que ya tiene su configuración congelada en memoria.

**Brecha:** La inmutabilidad solo existe en memoria durante la vida de la ejecución. No hay snapshot persistido, ni hash, ni trazabilidad histórica que permita saber qué configuración usó una ejecución pasada ni reproducirla. El diseño canónico (P10, C.7.1, C.7.2, ADR-005) ya define el contrato de snapshot y hashes, pero la implementación no lo ha materializado.

### 7.3 Información asociada a la ejecución

`Execution` declara y serializa en `toJSON` los campos `provider`, `model` y `promptVersionId`, pero **no se les asigna valor** en el flujo de escritura actual (`ExecutionEngine.process()` solo asigna `id`, `workspaceId`, `botId`, `botInstanceId`, `conversationId`, `contactId`, `channel`, `correlationData`; los setters `setProvider`, `setModel`, `setPromptVersion` no son invocados). Por tanto, esos campos permanecen `null` en la persistencia.

Se persisten: `tokens`, `cost`, `status`, fechas, `iteration`, `result`, `error`. **No persiste** la configuración efectiva completa, ni hash de configuración, ni hash de prompt, ni versiones de Skills/Rules utilizadas.

### 7.4 Hashes de configuración efectiva y de prompt

**No existen.** No hay cálculo de SHA-256 ni de ningún hash de configuración efectiva ni de prompt. El modelo objetivo define `promptSnapshotHash` y `effectiveConfigHash` (canónico C.7.1), pero no están implementados.

### 7.5 Referencias versionadas vs mutables

La ejecución referencia `botId` y `botInstanceId` (entidades mutables). No almacena versiones inmutables de AgentDefinition, Rules o Skills. Si `Bot` o `BotInstance` se modifican, no hay forma de saber qué configuración usó una ejecución pasada.

### 7.6 Cambios concurrentes de configuración

`EffectiveConfig` y `ModelConfiguration` están congelados en memoria al inicio de la ejecución. Un cambio en `Bot`/`BotInstance` (proveedor, modelo, temperatura, systemPrompt, capabilities, knowledgeBaseIds) **entre el inicio y el final de una ejecución no afecta a esa ejecución en curso**, porque su configuración ya está cargada e inmutable en memoria. Una ejecución posterior resolvería la configuración nueva desde la DB.

La brecha no es de corrupción de ejecución en curso, sino de **auditoría y trazabilidad**: no hay snapshot persistido, ni hash, ni versión que permita saber qué configuración usó una ejecución pasada, ni reproducirla. No hay mecanismo que impida ni que detecte cambios concurrentes en las entidades fuente.

### 7.7 Reproducibilidad y auditoría

**No reproducible.** Sin snapshot, sin hash y sin versiones persistidas, una ejecución no puede reproducirse ni auditarse con la información persistida. Solo se sabe qué `botId`/`botInstanceId` usó, no qué configuración efectiva tuvo en ese momento.

### 7.8 Secretos en snapshot y logs

Las credenciales del bot (`Bot.aiCredential`) están excluidas de `toJSON` (`bot.ts:418-420`, Fase 1 §4.3). Esta exclusión solo cubre la serialización del modelo `Bot`; **no demuestra por sí sola** que no existan secretos en los logs de traza. La política de logs (exclusión de secretos de registros de traza) es un contrato separado aún no definido. Dado que no existe snapshot, no hay secreto en snapshot.

**Metadata de Execution:** `Execution` declara y serializa en `toJSON` los campos `provider`, `model` y `promptVersionId` (`execution.ts:306-329`), pero en el recorrido verificado del flujo normal esos campos permanecen nulos: `createExecution` en `execution-engine.ts:153-164` solo asigna `id`, `workspaceId`, `botId`, `botInstanceId`, `conversationId`, `contactId`, `channel`, `correlationData`; los setters `setProvider`, `setModel` y `setPromptVersion` (`execution.ts:270-286`) existen pero no se invocan en `ExecutionEngine.process()` ni en los pasos del pipeline inspeccionados. No afirmar que proveedor y modelo queden efectivamente persistidos en el flujo normal. Mantener coherencia con §5.6 y §7.3.

### 7.9 Diferencias entre configuración inicial y steps posteriores

`EffectiveConfig` es inmutable (congelada) tras su creación (ya sea en `BotRuntimeResolver.resolve` + `buildContext` en la ruta RabbitMQ, o en `LoadConfigurationStep` en la ruta alternativa). Los steps posteriores no modifican la configuración; solo acumulan resultados (tokens, cost, tool calls). No hay diferencia entre la configuración cargada inicialmente y la usada en steps posteriores porque el objeto es el mismo y es inmutable. El problema es que no hay persistencia de esa configuración para auditoría.

### 7.10 Diferenciación configuración / prompt / contexto

| Concepto | Estado actual | ¿Snapshot? | ¿Versionado? |
|----------|--------------|-----------|-------------|
| Configuración efectiva | `EffectiveConfig` (provider, modelo, prompt, capabilities, knowledgeBaseIds) | No | No |
| Prompt efectivo | `config.systemPrompt` o `PromptVersion` renderizado | No | `PromptVersion` tiene versión opcional |
| Contexto recuperado (RAG) | `knowledgeFragments` de Qdrant (BR-KB-001) | No | No; Qdrant falla silenciosamente a `[]` |

---

## 8. Escenarios de cambios concurrentes de configuración

### 8.1 Cambio de Bot/BotInstance durante una ejecución en curso

La `EffectiveConfig` se carga en memoria al inicio y es inmutable (congelada). Un cambio posterior en `Bot` o `BotInstance` no afecta a la ejecución en curso porque el objeto ya está cargado. Sin embargo:
- No hay evidencia persistida de qué configuración usó la ejecución.
- No hay hash que permita detectar si la configuración cambió entre el inicio y el final.
- Una ejecución posterior resolverá la configuración nueva automáticamente.

### 8.2 Cambio de proveedor, modelo o credencial

El cliente puede cambiar `Bot.modelConfiguration` (provider, modelo) y `Bot.aiCredential` en cualquier momento. Una ejecución en curso conserva su `EffectiveConfig` en memoria, pero la próxima ejecución usará el nuevo proveedor/modelo. No hay mecanismo que impida cambios concurrentes ni que los detecte.

### 8.3 Cambio de capabilities o Rules

`LoadRulesSkillsStep` carga Rules y Skills al inicio del pipeline. Si cambian durante la ejecución, la ejecución en curso conserva los fragmentos cargados. No hay inmutabilidad garantizada ni auditoría de qué versión se usó.

### 8.4 Cambio de Knowledge (knowledgeBaseIds)

`ExecuteRAGStep` lee `config.knowledgeBaseIds` al inicio. Si cambian durante la ejecución, la ejecución en curso conserva los fragmentos recuperados. Qdrant filtra por `config.workspaceId` (canonical del bot). No hay registro de qué documentos se recuperaron en cada ejecución.

### 8.5 Cambio de system prompt / PromptVersion

`ResolvePromptStep` resuelve el `PromptVersion` al inicio. Si se publica una nueva versión durante la ejecución, la ejecución en curso conserva la versión resuelta. No hay hash del prompt efectivo.

### 8.6 Resumen de riesgo de concurrencia

| Escenario | ¿Afecta ejecución en curso? | ¿Persistido para auditoría? | ¿Detectable? |
|-----------|---------------------------|---------------------------|-------------|
| Cambio de proveedor/modelo | No (config en memoria) | No | No |
| Cambio de temperatura/maxTokens | No | No | No |
| Cambio de system prompt | No | No | No |
| Cambio de capabilities | No | No | No |
| Cambio de Rules/Skills | No | No | No |
| Cambio de knowledgeBaseIds | No | No | No |
| Cambio de credencial | No | No | No |

**Conclusión:** Los cambios concurrentes no alteran ejecuciones en curso (por inmutabilidad en memoria), pero no hay forma de auditar ni reproducir qué configuración usó cada ejecución. P10 no está implementado.

---

## 9. Contratos que deben quedar definidos en WAB-78

### 9.1 Contratos ya definidos (en documentación)

| Contrato | Estado | Evidencia |
|----------|--------|-----------|
| Separación AgentDefinition (Waby) + AgentInstance (Workspace) | Aprobado (diseño) | Canónico A.2, B; P10 aprobado |
| AgentInstance persiste en CRM | Aprobado | Registro §8.1 |
| Cadena AgentInstance → Workspace → Organización | Aprobado | Registro §8.1; D.1.5 L734 |
| Snapshot inmutable de configuración efectiva por ejecución (P10) | Aprobado | Registro §2 (P10); Canónico C.7, E.1 paso 5 |
| Dos hashes: promptSnapshotHash + effectiveConfigHash | Aprobado (diseño) | Canónico C.7.1 |
| effectiveConfigSnapshot como JSON sin secretos | Aprobado (diseño) | Canónico C.7.2 |
| knowledgeBaseIds = bases habilitadas (no contenido recuperado) | Aprobado | Canónico C.7.3 |
| Provider/modelo/credenciales centralizados en Waby | Aprobado | Canónico G.1, G.2 |
| AssignmentType unificado en Contacto | Aprobado | Canónico D.1.5 |
| EvAssigned como discriminator de Evento en CRM | Aprobado | Canónico D.1.6 |
| Sin assignmentVersion | Aprobado | Canónico D.1.7 |
| G3/G4 validan vigencia antes de despacho | Aprobado | Canónico D.1.8, E.4.12 |
**Total contratos ya definidos:** 12

### 9.2 Contratos pendientes de definición en WAB-78

| # | Contrato pendiente | ¿Bloquea WAB-78? | Justificación | Destino |
|---|-------------------|-----------------|---------------|---------|
| C1 | Resolución de AgentDefinition: quién la crea, versiona, publica, retira | No | El contrato de resolución de AgentDefinition se define en WAB-78; la falta de implementación de la entidad por sí sola no bloquea WAB-78 (regla de la línea base) | WAB-78 (contrato) → WAB-79 (modelo) |
| C2 | Resolución de AgentInstance: quién crea, activa, suspende, retira, designa predeterminada | No | Referencia a ARQ-19 (actor sin definir), bloqueo preexistente de D2; no es un bloqueo nuevo de D4-A | WAB-78 (contrato P3) → WAB-80 (implementación) |
| C3 | Composición determinista de configuración efectiva (precedencia definición → instancia → overrides) | Sí | Sin reglas de combinación documentadas no se puede definir la configuración efectiva determinista; es un contrato de diseño que debe aprobarse en WAB-78 | WAB-78 |
| C4 | Precedencia entre valores de plataforma y configuración de instancia | Sí | Sin distinción entre valores de plataforma y valores de instancia no se puede definir el contrato de configuración efectiva; es un contrato de diseño que debe aprobarse en WAB-78 | WAB-78 |
| C5 | Momento exacto de creación del snapshot (antes de G1, después de resolución, antes de pipeline) | No | P10, C.7.1 y ADR-005 ya definen el contrato de snapshot inmutable por ejecución; el momento exacto de creación es un detalle de implementación para fases posteriores, no un bloqueo de diseño de WAB-78 | WAB-79/WAB-80 |
| C6 | Persistencia y asociación del snapshot con Execution (campos, colección) | No | C.7.2 ya define `effectiveConfigSnapshot` como JSON sin secretos; los campos específicos de la colección `Execution` son detalle de implementación pendiente, no bloqueo de diseño | WAB-79/WAB-80 |
| C7 | Inmutabilidad durante la ejecución (garantía de que no se consulta config mutable durante el procesamiento) | No | P10 ya define la inmutabilidad por ejecución; la garantía persistida es trabajo de implementación para fases posteriores, no bloqueo de diseño de WAB-78 | WAB-79/WAB-80 |
| C8 | Identificación de versiones o revisiones utilizadas (AgentDefinition.version, Rule.version, Skill.version) | No | C.7.1 ya define el contrato de versiones en el snapshot; la falta de implementación de los campos de versión por sí sola no bloquea WAB-78 (regla de la línea base). Queda por definir qué entidad o catálogo proporciona la versión inmutable de Skills | WAB-78/WAB-79 |
| C9 | Comportamiento ante configuración inexistente, inválida o inconsistente | No | `BotRuntimeResolver.resolve(botId)` no recibe `botInstanceId` del payload; si no encuentra instancia activa o falla la consulta, usa configuración sintética `default-{bot.id}`; `resolveEffectiveConfig` lanza error si `bot` no tiene `PromptVersion` activo; no hay contrato de fallback definido para estos casos. La falta de definición de este contrato por sí sola no bloquea WAB-78; se define en WAB-78 sin requerir implementación previa | WAB-78 |
| C10 | Exclusión de secretos del snapshot y de los logs | No | Credenciales excluidas de `toJSON` (`bot.ts:418-420`) para el snapshot; la política de logs (exclusión de secretos de registros de traza) es un contrato separado que aún no está definido. La exclusión de `toJSON` no demuestra por sí sola ausencia de secretos en logs | WAB-78 (snapshot) / D5-C (logs) |

### 9.3 Contratos que NO pertenecen a WAB-78 (excluidos)

- Composición del contexto, PromptDocument, Knowledge/RAG → D4-B.
- Policy Engine, output filter, prompt injection → D5-A (ARQ-10 revisado allí).
- Restricciones de conocimiento externo y confidencialidad → D5-B.
- Aislamiento multitenant y autorización de capabilities → D5-C.

---

## 10. Hallazgos candidatos y clasificación

Los hallazgos usan identificadores provisionales `D4A-C01` a `D4A-C13`. No duplican ARQ-01 a ARQ-22.

### D4A-C01 — No existe separación AgentDefinition / AgentInstance en el runtime

| Campo | Contenido |
|-------|-----------|
| Dominio | Runtime / Modelo de dominio |
| Estado | Ausencia de implementación |
| Evidencia | `crm/src` y `crm/microservices/ai-service/src`: grep `AgentDefinition` = 0; `AgentInstance` = 0 como entidad; `ResourceOwnership.ts:33-45` sin `AGENT_INSTANCE`; `permission.enum.ts` sin `platform.*` |
| Comportamiento observado | El runtime resuelve todo a partir de `Bot` + `BotInstance` en ai-service. `Bot` mezcla definición (provider, modelo, credencial, capabilities) e instancia. |
| Contrato objetivo afectado | Resolución de AgentDefinition (C1); Resolución de AgentInstance (C2) |
| Impacto arquitectónico | Alto: el modelo objetivo no tiene contraparte implementada; WAB-79 debe definir el modelo desde cero |
| Bloquea WAB-78 | No (ausencia de implementación esperada antes de WAB-79; no impide definir el contrato de resolución) |
| Justificación | La ausencia de las entidades es estado as-is esperado; el contrato de resolución se define en WAB-78 sin requerir su implementación previa |
| Fase | WAB-78 (contrato) → WAB-79 (modelo) |
| Criterio de cierre | Entidades AgentDefinition y AgentInstance definidas en el modelo objetivo con propiedad y cardinalidad claras |

### D4A-C02 — No existe snapshot inmutable de configuración efectiva por ejecución (brecha de implementación, no bloqueo de diseño)

| Campo | Contenido |
|-------|-----------|
| Dominio | Runtime / P10 / Trazabilidad |
| Estado | Ausencia de implementación |
| Evidencia | `execution.ts`: sin `effectiveConfigSnapshot`, `effectiveConfigHash`, `promptSnapshotHash`; `execution-context.ts`: sin snapshot; `effective-config.ts`: objeto congelado pero no persistido |
| Comportamiento observado | La configuración se resuelve al inicio y se carga en memoria, pero no se captura copia inmutable ni hash para persistencia |
| Contrato objetivo afectado | Momento de creación del snapshot (C5); Persistencia del snapshot (C6); Inmutabilidad (C7) |
| Impacto arquitectónico | Medio: P10, C.7.1, C.7.2 y ADR-005 ya definen el contrato de snapshot y hashes; la brecha es de implementación, no de definición de diseño |
| Bloquea WAB-78 | No (el contrato de P10 está aprobado; la implementación corresponde a fases posteriores) |
| Justificación | El contrato de snapshot inmutable está definido en P10, C.7.1, C.7.2 y ADR-005; la ausencia de implementación no impide cerrar el diseño de WAB-78 |
| Fase | WAB-78 (contrato ya definido) → WAB-79/WAB-80 (implementación) |
| Criterio de cierre | Execution persiste `effectiveConfigSnapshot` (JSON sin secretos) y `effectiveConfigHash` al iniciar cada ejecución |

### D4A-C03 — No existen hashes de configuración efectiva ni de prompt (brecha de implementación, no bloqueo de diseño)

| Campo | Contenido |
|-------|-----------|
| Dominio | Runtime / P10 / Trazabilidad |
| Estado | Ausencia de implementación |
| Evidencia | No hay cálculo SHA-256 ni hash de ningún tipo en `execution.ts`, `execution-context.ts`, `effective-config.ts` |
| Comportamiento observado | No hay forma de identificar qué configuración o prompt usó una ejecución |
| Contrato objetivo afectado | Identificación de versiones/revisiones (C8) |
| Impacto arquitectónico | Medio: C.7.1 y ADR-005 ya definen el contrato de `promptSnapshotHash` y `effectiveConfigHash`; la brecha es de implementación, no de definición de diseño |
| Bloquea WAB-78 | No (el contrato de hashes está definido; la implementación corresponde a fases posteriores) |
| Justificación | Los contratos de hash están definidos en C.7.1 y ADR-005; la ausencia de implementación no impide cerrar el diseño de WAB-78 |
| Fase | WAB-78 (contrato ya definido) → WAB-79 (implementación) |
| Criterio de cierre | `promptSnapshotHash` y `effectiveConfigHash` calculados y persistidos por ejecución |

### D4A-C04 — Inmutabilidad solo en memoria, no hay snapshot persistido

| Campo | Contenido |
|-------|-----------|
| Dominio | Runtime / Inmutabilidad |
| Estado | Implementación parcial (inmutable en memoria, mutable en DB) |
| Evidencia | `effective-config.ts:28-41`: `Object.freeze(this)` en `EffectiveConfig`; `model-configuration.ts`: `Object.freeze` en `ModelConfiguration`; `load-configuration.step.ts:43-53`: carga Bot y BotInstance desde DB al inicio |
| Comportamiento observado | `EffectiveConfig` y `ModelConfiguration` se congelan al crearse; los steps posteriores no mutan la configuración. Los cambios en Bot/BotInstance en la DB **no** afectan a una ejecución ya iniciada que tiene su configuración congelada en memoria. No hay snapshot persistido ni hash para auditoría histórica |
| Contrato objetivo afectado | Inmutabilidad durante la ejecución (C7) |
| Impacto arquitectónico | Medio: la inmutabilidad en memoria protege la ejecución en curso, pero la ausencia de snapshot persistido impide auditoría y reproducibilidad |
| Bloquea WAB-78 | No (la ejecución en curso no se ve afectada por cambios concurrentes) |
| Justificación | El riesgo es de auditoría (no hay trazabilidad histórica), no de corrupción de ejecución en curso |
| Fase | WAB-78 (contrato) |
| Criterio de cierre | Contrato C7 define que la ejecución no consulta configuraciones mutables durante el procesamiento; snapshot persistido definido en C6 |

### D4A-C05 — No hay precedencia documentada entre definición e instancia; defaultSkills/defaultRules y systemPromptOverride sin definir

| Campo | Contenido |
|-------|-----------|
| Dominio | Configuración efectiva |
| Estado | Ausencia de implementación / documentación |
| Evidencia | `bot-instance.resolveEffectiveConfig(bot)` (`bot-instance.ts:240-272`): fallback implícito sin reglas documentadas; no hay modelo `AgentDefinition`/`AgentInstance` en código (grep 0). Canónico A.2 define `defaultSkills`/`defaultRules` en `AgentDefinition` y `skills`/`rules`/`systemPromptOverride` en `AgentInstance`, pero no existen como entidades ni campos implementados |
| Comportamiento observado | Campos de `BotInstance` sobrescriben a `Bot` con fallback implícito (`this._x || bot.x`), sin precedencia explícita ni reglas de combinación. `systemPrompt` cae a `'You are a helpful assistant'` (hardcodeado), no a `bot.systemPrompt`. `knowledgeBaseIds` siempre proviene del `Bot`, nunca del instance |
| Contrato objetivo afectado | Composición determinista (C3); Precedencia plataforma vs instancia (C4) |
| Impacto arquitectónico | Alto: sin reglas de combinación no se puede definir la configuración efectiva determinista; `defaultSkills`/`defaultRules` (definición) vs `skills`/`rules` (instancia) y `systemPromptOverride` quedan sin semántica de copia/combinación/sustitución definida |
| Bloquea WAB-78 | Sí |
| Justificación | C3 y C4 son contratos fundamentales de WAB-78; la relación entre `defaultSkills`/`defaultRules` y `SkillConfiguration`/`AgentRules`, y el comportamiento de `systemPromptOverride`, deben precisarse |
| Fase | WAB-78 |
| Criterio de cierre | Reglas de precedencia y combinación documentadas y aprobadas, incluyendo si `defaultSkills`/`defaultRules` se copian, combinan o sustituyen en la instancia, y si `systemPromptOverride` reemplaza o complementa el prompt base |

### D4A-C06 — Capabilities cargadas globalmente, sin filtro por instancia

| Campo | Contenido |
|-------|-----------|
| Dominio | Runtime / Capabilities |
| Estado | Ausencia de implementación (filtro por instancia) |
| Evidencia | `execution-engine.ts:65-67`: `capabilityRegistry.findAll()` carga todas; Fase 1 §4.1 hallazgo #5 |
| Comportamiento observado | Todas las capabilities se cargan para cada ejecución, sin filtrar por `enabledCapabilities` del bot/instancia |
| Contrato objetivo afectado | Composición determinista (C3) — las capabilities deben ser por instancia en el modelo objetivo |
| Impacto arquitectónico | Medio: en el modelo objetivo, skills/capabilities son por instancia (SkillConfiguration) |
| Bloquea WAB-78 | No (diferido a WAB-79 como parte del modelo de Skills/Rules) |
| Justificación | Es una consecuencia de no existir AgentInstance; se resuelve con el modelo objetivo |
| Fase | WAB-79 |
| Criterio de cierre | Capabilities filtradas por AgentInstance/SkillConfiguration en el modelo objetivo |

### D4A-C07 — Credenciales per-bot, no centralizadas

| Campo | Contenido |
|-------|-----------|
| Dominio | Runtime / Seguridad |
| Estado | Implementación actual (per-bot) vs objetivo (centralizadas) |
| Evidencia | `bot.ts:43` (`aiCredential`), `bot.ts:418-420` (excluida de toJSON); `invoke-provider.step.ts:67-69` (descipta key del bot) |
| Comportamiento observado | Cada bot tiene su propia API key cifrada; el cliente la configura |
| Contrato objetivo afectado | Credenciales centralizadas en Waby (G.2, I6) |
| Impacto arquitectónico | Alto: cambio de arquitectura de credenciales |
| Bloquea WAB-78 | No (pertenece a G.2/I6, ya aprobado como dirección; implementación en WAB-79/WAB-80) |
| Justificación | Es una decisión de producto ya aprobada; la implementación es posterior |
| Fase | WAB-79/WAB-80 |
| Criterio de cierre | Credenciales gestionadas centralmente por Waby, nunca expuestas al cliente |

### D4A-C08 — No hay versionado de configuración (salvo PromptVersion opcional); SkillConfiguration no declara versión (brecha de implementación, no bloqueo de diseño)

| Campo | Contenido |
|-------|-----------|
| Dominio | Configuración / Versionado |
| Estado | Ausencia de implementación |
| Evidencia | `effective-config.ts`: sin campo de versión; `Bot` sin versión de configuración; `PromptVersion` tiene versión opcional. Canónico C.7.1 requiere versiones de Skills en el snapshot, pero `SkillConfiguration` (modelo objetivo) no declara campo `version` en A.2; `AgentRules` sí declara `version` |
| Comportamiento observado | No hay forma de identificar qué versión de configuración, ni de Skills, usó una ejecución. `AgentDefinition.version` y `AgentRules.version` existen en el modelo objetivo pero no están implementados. `SkillConfiguration` no declara versión pese a que C.7.1 la requiere |
| Contrato objetivo afectado | Identificación de versiones/revisiones (C8) |
| Impacto arquitectónico | Medio: C.7.1 y ADR-005 ya definen el contrato de versiones; la ausencia de implementación no impide cerrar el diseño de WAB-78 |
| Bloquea WAB-78 | No (el contrato de versiones está definido en C.7.1; la implementación corresponde a fases posteriores) |
| Justificación | El contrato de versión de configuración está definido en C.7.1; la ausencia de implementación de los campos de versión no bloquea el diseño de WAB-78 |
| Fase | WAB-78 (contrato ya definido) → WAB-79 (modelo) |
| Criterio de cierre | `AgentDefinition.version` y `AgentRules.version` implementadas; `SkillConfiguration` con campo `version` o catálogo que provea versión inmutable de Skills; la ejecución referencia la versión usada |

### D4A-C09 — No hay contrato de comportamiento ante configuración inexistente o inválida (evidencia de BotRuntimeResolver)

| Campo | Contenido |
|-------|-----------|
| Dominio | Configuración efectiva |
| Estado | Ausencia de implementación |
| Evidencia | `message-received-handler.ts:104`: `BotRuntimeResolver.resolve(botId)` no recibe `botInstanceId` del payload; selecciona la primera `BotInstance` activa sin comprobar `isDefault`; si no encuentra instancia activa o falla la consulta, usa configuración sintética `default-{bot.id}`. `bot-instance.ts:249-251`: `resolveEffectiveConfig` lanza error si `bot` no tiene `PromptVersion` activo. `load-configuration.step.ts:44-51`: lanza error si Bot o BotInstance no existen en la ruta no-Rabbit |
| Comportamiento observado | Ausencia de instancia activa → fallback sintético `default-{bot.id}`. Fallo de consulta → configuración sintética. `PromptVersion` ausente → error. No hay contrato de fallback definido para estos casos; el comportamiento objetivo está pendiente |
| Contrato objetivo afectado | Configuración inexistente/inválida/inconsistente (C9) |
| Impacto arquitectónico | Medio: sin contrato definido, el comportamiento ante ausencia de instancia o de PromptVersion es implícito |
| Bloquea WAB-78 | No (es un detalle de contrato a definir) |
| Justificación | C9 puede definirse como parte del contrato sin bloquear el diseño |
| Fase | WAB-78 |
| Criterio de cierre | Contrato C9 define comportamiento ante configuración inexistente, inválida o inconsistente, incluyendo ausencia de instancia activa, fallo de consulta y ausencia de PromptVersion |

### D4A-C10 — Publicación de `execution.completed` sin importar estado cancelado → ARQ-07

| Campo | Contenido |
|-------|-----------|
| Dominio | Runtime / Cancelación |
| Estado | Implementación parcial (anomalía semántica) |
| Evidencia | `execution-engine.ts:202-234`: `publishResult` publica `EventTypes.EXECUTION_COMPLETED` incluso si la ejecución está cancelada |
| Comportamiento observado | El evento viaja por RabbitMQ aunque la ejecución esté cancelada; G3 lo descarta por estado, pero el evento ya viajó |
| Contrato objetivo afectado | Invalicación de efectos tardíos (relacionado con ARQ-07) |
| Impacto arquitectónico | Medio: posible envío de resultados obsoletos si otro consumidor procesa el evento |
| Bloquea WAB-78 | No (duplicado de ARQ-07 en D3) |
| Justificación | ARQ-07 cubre esta brecha; no crear duplicado. Ver D3, informe-bloque-3.md §3.3 y §8 |
| Fase | D3 (ya resuelto como bloqueante) |
| Criterio de cierre | `publishResult` no publica eventos para ejecuciones canceladas (contrato definido en D3/WAB-81) |

### D4A-C11 — No hay validación de workspaceId del payload contra el bot en entrada → D5-C

| Campo | Contenido |
|-------|-----------|
| Dominio | Runtime / Multitenant |
| Estado | Ausencia de implementación |
| Evidencia | `message-received-handler.ts:104`: `BotRuntimeResolver.resolve(botId)` sin validar `workspaceId` del payload contra `bot.workspaceId`; Fase 1 §4.1 hallazgo #4 |
| Comportamiento observado | El `workspaceId` del payload no se valida contra el bot resuelto |
| Contrato objetivo afectado | Aislamiento multitenant (D5-C, excluido de este informe) |
| Impacto arquitectónico | Alto: posible cruzamiento de tenants |
| Bloquea WAB-78 | No (pertenece a D5-C, excluido expresamente del alcance de D4-A) |
| Justificación | Excluido del alcance de D4-A por plan §4 |
| Fase | D5-C |
| Criterio de cierre | Validación de workspaceId contra el bot/instancia resuelta (dentro de D5-C) |

### D4A-C12 — No hay deduplicación persistente por originMessageId en la entrada → ARQ-22

| Campo | Contenido |
|-------|-----------|
| Dominio | Runtime / Idempotencia |
| Estado | Ausencia de implementación |
| Evidencia | `message-received-handler.ts:226-229`: `executionId` y `correlationId` generados aleatoriamente; sin dedup por `originMessageId` |
| Comportamiento observado | Una redelivery de RabbitMQ puede crear otra ejecución con `executionId` diferente |
| Contrato objetivo afectado | Relacionado con ARQ-22 (entrega durable e idempotencia) |
| Impacto arquitectónico | Alto: posible duplicación de ejecuciones |
| Bloquea WAB-78 | No (duplicado de ARQ-22 en D3) |
| Justificación | ARQ-22 ya es bloqueante en D3; no crear duplicado. Ver D3, informe-bloque-3.md §2.2 y §3.2 |
| Fase | D3 (ya resuelto) |
| Criterio de cierre | Dedup persistente por `originMessageId` (contrato definido en D3/WAB-81) |

### D4A-C13 — No hay separación de superficie de administración Waby vs cliente (referencia a ARQ-19/20/21 preexistentes)

| Campo | Contenido |
|-------|-----------|
| Dominio | Runtime / Autorización |
| Estado | Ausencia de mecanismo |
| Evidencia | `AiBot.resolver.ts`: cliente puede crear/actualizar bots con `apiKey` y `model`; sin actor Waby; `AuthChecker.ts:27-29` exige rol de organización; no existe `platform.*`, `bot.configure`, `wrk.agent.*`, `wrk.credits.*` en `permission.enum.ts` |
| Comportamiento observado | El cliente administra proveedor, modelo y credencial por bot (lo que el modelo objetivo reserva a Waby). No hay actor de ciclo de vida definido, ni permisos representables, ni identidad de servicio para operaciones internas |
| Contrato objetivo afectado | Separación Waby/cliente (G.1, G.2); Resolución de AgentDefinition (C1) |
| Impacto arquitectónico | Alto: la superficie actual no respeta la separación aprobada |
| Bloquea WAB-78 | No como bloqueo independiente de D4-A (referencia los bloqueos preexistentes ARQ-19, ARQ-20 y ARQ-21 ya identificados en bloques anteriores) |
| Justificación | D4A-C13 no es un hallazgo nuevo de D4-A; apunta a los bloqueos de actor, autorización e identidad de servicio ya registrados como ARQ-19, ARQ-20 y ARQ-21 |
| Fase | WAB-78 (contrato P3) → WAB-80 (implementación) |
| Criterio de cierre | Matriz de permisos P3 aprobada; actor de ciclo de vida definido (ARQ-19, ARQ-20, ARQ-21) |

---

## 11. Riesgos, dependencias y aspectos no verificados

### 11.1 Riesgos para WAB-78

1. **P10 sin implementación:** Sin snapshot inmutable ni hash, no se puede garantizar la inmutabilidad por ejecución aprobada. El contrato de P10, C.7.1, C.7.2 y ADR-005 está definido; la implementación es trabajo para fases posteriores (WAB-79/WAB-80), no un bloqueo de diseño de WAB-78.
2. **Modelo objetivo sin contraparte implementada:** AgentDefinition y AgentInstance no existen en código; WAB-79 debe definirlos desde cero sin una base de migración verificada.
3. **Contrato de permisos sin definir (P3):** Sin matriz de permisos ni actor de ciclo de vida, WAB-80 no puede implementar provisioning ni autorización (ARQ-19, ARQ-20, ARQ-21).
4. **Identidad de servicio no definida:** Las operaciones internas de D.2 (resolveAgentInstance, executionCompleted, deductCredits, deleteWorkspaceResources) no tienen mecanismo de autenticación en CRM (ARQ-21).
5. **Superficie actual no respeta la separación Waby/cliente:** El cliente puede administrar proveedor, modelo y credencial por bot hoy; WAB-80 debe retirar/restringir esta superficie explícitamente.

### 11.2 Dependencias

| Dependencia | Origen | Estado | Impacto en WAB-78 |
|-------------|--------|--------|-------------------|
| Definición contractual de AgentDefinition/AgentInstance | WAB-78 | Pendiente (contrato) | Los contratos de resolución pueden definirse en WAB-78; su implementación de dominio y persistencia corresponde a fases posteriores (WAB-79/WAB-80) |
| Implementación de AgentDefinition/AgentInstance | WAB-79 | Pendiente | No bloquea el diseño contractual de WAB-78 |
| Matriz de permisos P3 | WAB-78 (decisión de producto) | Pendiente | Bloquea C2; referencia ARQ-19, ARQ-20 |
| Identidad de servicio para operaciones internas | WAB-78 (contrato) | Pendiente | Referencia ARQ-21 (brecha de identidad de servicio ya identificada) |
| Verificación de transacciones en MongoDB Atlas | D6 | Pendiente (no verificable localmente) | Relacionado con atomicidad (ARQ-06) |
| Cancelación e invalidación de efectos tardíos | D3 (ARQ-07) | Bloqueante | Relacionado con invalicación de ejecución |
| EvAssigned y identidad de transición | D3 (ARQ-05, ARQ-08) | Bloqueante | Relacionado con ABA y vigencia |

### 11.3 Aspectos no verificados

- Compatibilidad de transacciones MongoDB en Atlas (no se puede afirmar sin evidencia del despliegue).
- Valor real de `CRM_API_KEY` en producción (no es un JWT verificable localmente).
- Comportamiento real del evento `execution.completed` con consumidores adicionales no inspectados.
- Índices declarados en schemas Mongoose no verificados contra una instancia real de MongoDB.

---

## 12. Conclusión del bloque D4-A

### 12.1 ¿Está trazado el flujo real del runtime?

Sí. El flujo está documentado desde la entrada por RabbitMQ (`message-received-handler.ts`) hasta la publicación del resultado (`publishResult`), pasando por `ExecutionEngine`, el pipeline de 8 pasos, la invocación del proveedor LLM (`InvokeProviderStep`), la ejecución de capabilities (`runPipelineLoop`) y el consumo en CRM G3 (`executionCompleted.consumer.ts`).

### 12.2 ¿Está documentada la resolución de configuración efectiva con evidencia?

Sí. En la ruta RabbitMQ (productiva), la resolución ocurre anticipadamente en `BotRuntimeResolver.resolve(botId)` (`message-received-handler.ts:104`), antes de construir el contexto; `buildContext` (`message-received-handler.ts:194-250`) preestablece `config` y `bot`, y `LoadConfigurationStep` omite la resolución, solo aplicando el control de credenciales. En la ruta alternativa (no-Rabbit), `LoadConfigurationStep` carga `Bot` y `BotInstance` y resuelve `EffectiveConfig` mediante `botInstance.resolveEffectiveConfig(bot)` (`load-configuration.step.ts:43-53`). En ambos casos se almacena en `EffectiveConfig` (objeto congelado). No hay snapshot, ni hash, ni versionado, ni persistencia inmutable.

### 12.3 ¿Existe un snapshot real e inmutable por ejecución?

No. No existe `effectiveConfigSnapshot`, `effectiveConfigHash` ni `promptSnapshotHash` en el modelo `Execution` ni en `ExecutionContext`. P10 (aprobado) no tiene implementación.

### 12.4 ¿Se distinguen configuración, prompt y contexto?

Sí. El informe diferencia expresamente: configuración efectiva (`EffectiveConfig`: provider, modelo, capabilities, knowledgeBaseIds), prompt efectivo (`systemPrompt` o `PromptVersion` renderizado), y contexto recuperado (`knowledgeFragments` de Qdrant, sin registro de qué fragmentos se usaron).

### 12.5 ¿Se identifican diferencias entre el modelo actual y AgentDefinition/AgentInstance?

Sí. El modelo actual `Bot` mezcla definición e instancia; no existe separación propiedad Waby/Workspace; no hay versionado; las credenciales son per-bot; las capabilities son globales; no hay snapshot.

### 12.6 ¿Están enumerados los contratos para WAB-78?

Sí. En §9.1 se enumeran 12 contratos ya definidos (en documentación/canónico/registro de decisiones). En §9.2 se enumeran 10 contratos pendientes de definición (C1 a C10). Los identificadores C1-C10 se usan exclusivamente para los contratos pendientes de §9.2; no se mezclan con los contratos ya definidos de §9.1. Los bloques excluidos son D4-B, D5-A, D5-B y D5-C.

### 12.7 ¿Los candidatos no duplican ARQ-01 a ARQ-22?

Las 13 entradas (D4A-C01 a D4A-C13) son candidatas provisionales de seguimiento, no 13 hallazgos nuevos e independientes. Algunas son observaciones propias de D4-A; otras son referencias a hallazgos existentes: D4A-C10 → ARQ-07, D4A-C12 → ARQ-22, D4A-C13 → ARQ-19/20/21, D4A-C11 → D5-C. Ninguna duplica ARQ-01 a ARQ-22.

### 12.8 ¿Están declaradas las limitaciones de evidencia?

Sí: informe guardado en `C:\Users\sinies7ro\Dropbox\WABY\docs\revision-wab-78/informe-d4-a-runtime.md` (ruta Dropbox verificada); sin acceso a Linear directo, sin acceso a `.env`/producción, índices no verificados contra Atlas.

### Veredicto del bloque D4-A

El diagnóstico D4-A tiene suficiente evidencia para cerrar el bloque y está listo para revisión independiente. WAB-78 no está listo para aprobación global porque faltan por definir contratos de diseño (C3: composición determinista; C4: precedencia plataforma vs instancia) y existen brechas de implementación (D4A-C02: snapshot; D4A-C03: hashes; D4A-C08: versionado) cuyos contratos ya están aprobados (P10, C.7.1, C.7.2, ADR-005). Los hallazgos D4A-C10 y D4A-C12 están deduplicados contra ARQ-07 y ARQ-22 respectivamente; D4A-C11 pertenece a D5-C; D4A-C13 referencia los bloqueos preexistentes ARQ-19, ARQ-20 y ARQ-21 sin presentar un bloqueo nuevo de D4-A. D4A-C01 y D4A-C04 no bloquean por ausencia de implementación o inmutabilidad en memoria. No se inicia D4-B ni D5.

---

