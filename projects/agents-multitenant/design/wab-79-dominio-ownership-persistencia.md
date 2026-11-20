# WAB-79 — Dominio, ownership y persistencia

> **Estado:** Documento de trabajo para revisión. No se ha iniciado formalmente la implementación.
> **Fase:** WAB-79 (Bloques B1, B2, B3).
> **Modo:** Revisión y reconciliación documental local de WAB-79 con el contrato canónico de Fase 2.
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
| WAB-82 | Operación funcional de Knowledge, Skills y Rules | Autorización, selección/resolución de Skills y versiones, composición y aplicación de Rules, integración con runtime, gestión funcional de bajas y coordinación de limpieza. Consume servicios de persistencia de A2 (WAB-79); no duplica modelos, schemas ni repositorios. |
| WAB-84 | Reemplazo y limpieza del modelo legacy | Retirar componentes Bot/BotInstance y documentación obsoleta; no requiere migración de datos ni compatibilidad hacia atrás porque la implementación de IA no llegó a producción |
| WAB-85 | Créditos y observabilidad | `OrganizationAICredit`, `CreditLedger`, presupuestos (P4) |

---

## 1.4 Frontera de responsabilidades A2 (WAB-79) / WAB-82 — Fuente principal

> **Esta sección es la fuente canónica de la frontera de responsabilidades entre A2 (incremento de persistencia de WAB-79) y WAB-82.** Los demás documentos del repositorio referencian esta sección; no se duplica la explicación en múltiples lugares.

### A2 — Persistencia autoritativa en CRM (WAB-79, incremento A2)

A2 es responsable **exclusivamente** de:

- Modelos y schemas de `SkillConfiguration` y `AgentRules`.
- Índices respaldados por el contrato (únicamente `{agentInstanceId: 1, source: 1, ownerId: 1, skillId: 1}` para `SkillConfiguration` y `{agentInstanceId: 1}` para `AgentRules`).
- Repositorios de persistencia y consulta (**Create, Read, Update**, listados, filtros por `agentInstanceId`). **Sin operación Delete en A2 durante este incremento.**
- Servicios de dominio necesarios para esas operaciones de persistencia.
- Validación de pertenencia de las entidades a una `AgentInstance` y respeto del aislamiento Workspace/Organization (cross-tenant en escritura).

**Fuera de A2 (no implementar en este incremento):**

- Autorización y operaciones funcionales de configuración (crear/habilitar/deshabilitar/borrar lógicamente Skills y Rules desde la UX/API).
- Selección y resolución de Skills y sus versiones (catálogo Waby, releases, rollout).
- Composición y aplicación de Rules (precedencia, combinación con políticas de plataforma).
- Integración con el runtime (inyección en prompt, ejecución).
- Gestión funcional de bajas y coordinación de limpieza de recursos asociados.
- Eliminación física de configuraciones y Rules (semántica por definir; excluida de A2 hasta que exista una semántica contractual definida).
- Definición del identificador concreto de `ownerId` para Skills de origen Waby (A2 usa valor genérico estable; no bloquea schemas genéricos).
- Defaults no respaldados por el contrato (incluido `enabled=true`).
- Índices secundarios en `SkillConfiguration` por anticipado.

### WAB-82 — Gestión funcional y runtime

WAB-82 es responsable de:

- Autorización y operaciones funcionales de configuración (API/UX para que usuarios autorizados gestionen Skills/Rules de su instancia).
- Selección y resolución de Skills y sus versiones (catálogo central Waby, releases inmutables, rollout controlado, Skills propias del Workspace).
- Composición y aplicación de Rules (precedencia determinista, combinación con políticas de plataforma, validación de contradicciones).
- Integración con el runtime (construcción de prompt efectivo, inyección de Rules/Skills, snapshot P10).
- Gestión funcional de bajas y coordinación de limpieza de recursos (elimina referencias, coordina con otros dominios). La baja funcional de una Skill o Rule, la eliminación física del registro persistido y la coordinación de limpieza de recursos asociados corresponden a WAB-82; A2 persiste `enabled` como deshabilitación de la configuración, sin implementar esa semántica completa.
- Consume los servicios de persistencia de A2. **No duplica modelos, schemas ni repositorios** para `SkillConfiguration` ni `AgentRules`.

### Correcciones contractuales obligatorias (resumen)

1. **Sin índice único `{agentInstanceId, ruleName}`** para `AgentRules` sin fundamento contractual explícito.
2. **Sin `contentRef` ni estructuras para almacenar contenido de Skills** dentro de A2.
3. **`enabled=false` es deshabilitación de la configuración, no eliminación física ni baja funcional.** La baja funcional de una Skill o Rule, la eliminación física del registro persistido y la coordinación de limpieza de recursos asociados corresponden a WAB-82. A2 persiste el atributo `enabled`, pero no implementa por sí mismo la semántica completa del ciclo de vida funcional.
4. **Eliminación física excluida de A2** — semántica por definir antes de implementar.
5. **Sin `ownerId` concreto para Skills Waby** — A2 define `ownerId` como valor genérico estable; no bloquea schemas genéricos.
6. **Sin defaults no contractuales** — `enabled` no es obligatorio ni tiene default `true`.
7. **Sin índices secundarios en `SkillConfiguration`** — solo el índice único contractual; cualquier otro requiere consulta runtime concreta.

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

### 2.1 Aclaración de dominios de credenciales (base de D-W79-08)

El modelo anterior permitía que cada cliente configurara sus propios agentes, eligiera libremente un proveedor LLM y proporcionara su API key. `Bot.model`, `Bot.aiCredential` y el mecanismo `Aes256GcmCredentialCipher` pertenecen a ese modelo legacy; no constituyen la arquitectura objetivo ni una base que deba trasladarse al nuevo dominio de credenciales empresariales.

En el modelo objetivo, Waby administra centralmente los proveedores LLM, sus cuentas y API keys, los modelos disponibles, el routing y los parámetros técnicos permitidos. `AgentDefinition` referencia una `LLMRoute` mediante `llmRouteId`; la ruta define proveedor/modelo/parámetros, pero no almacena los secretos del proveedor. El cliente no proporciona API keys LLM ni elige libremente el proveedor.

Las credenciales de integraciones empresariales son un dominio independiente: el Workspace administra sus valores mediante usuarios autorizados; la plataforma Waby opera su custodia segura y resolución durante la ejecución. La separación comprende ownership administrativo, custodia, autorización, almacenamiento, resolución y trazabilidad. El mecanismo legacy `Bot.aiCredential` no se reutiliza como fundamento de este dominio.

### 2.2 Sustitución del modelo legacy y tratamiento de la auditoría

La implementación actual de IA basada en `Bot`/`BotInstance` **nunca llegó a producción**. Por lo tanto, el modelo objetivo `AgentDefinition`/`AgentInstance` la reemplaza directamente: no se requiere migración de datos, backfill, cutover, rollback de datos ni compatibilidad funcional hacia atrás. WAB-84 no debe planificarse como una migración operativa del modelo legacy; su alcance será el reemplazo y la limpieza de los componentes obsoletos.

Las reglas, configuraciones y contratos actuales de Bot —incluidos Rules, Skills, Knowledge, asignaciones, ejecuciones, routing, credenciales y sus dependencias— no son la fuente normativa del modelo nuevo. Deberán sustituirse por los contratos aprobados de WABY. La documentación que describa ese comportamiento legacy deberá marcarse como obsoleta, reemplazarse o archivarse como antecedente histórico claramente no normativo; no debe seguir figurando como guía vigente.

La auditoría de la implementación existente se realizó para identificar activos técnicos reutilizables y evitar repetir análisis, no para preservar el dominio anterior. Se rescatan únicamente patrones o infraestructura neutral después de validarlos contra los contratos nuevos:

- Infraestructura Outbox/Change Streams/workers como punto de partida, condicionada a corregir atomicidad, adquisición exclusiva, concurrencia, idempotencia y recuperación.
- Patrones de transacción MongoDB y propagación de sesiones, junto con las pruebas existentes como referencia, no como demostración de que las nuevas operaciones ya cumplen el contrato.
- Conocimiento del almacenamiento MinIO, hashes, object keys inmutables, actualización por CAS, validación de ZIP y extracción de `SKILL.md`, sujeto al contrato nuevo de Skills y retención de snapshots.
- Cuotas y contabilidad por Workspace como referencia técnica, sin asumir que las reglas actuales satisfacen D-W79-17.

No se trasladan como base de dominio `Bot`, `BotInstance`, `Execution`, asignaciones autónomas de AI Service, contratos HTTP legacy de asignación, `BotResource`/`BotResourceLink` ni sus reglas de negocio. Tampoco se conserva automáticamente código solo porque exista: cada componente debe justificarse por una necesidad del modelo nuevo. La limpieza debe retirar referencias, endpoints, modelos, colecciones, configuraciones, pruebas y documentación que queden sin consumidor o contrato vigente.

---

## 3. B1 — Diagnóstico técnico y evidencias

### 3.1 Estado verificado de WAB-78 y WAB-79 (Linear)

| Issue | Estado | Tipo | Dependencia |
|-------|--------|------|-------------|
| WAB-78 | Done (completado 08/11/2026) | Diseño previo a implementación | Padre: WAB-75 |
| WAB-79 | Backlog (último estado documentado en Linear; reconciliación documental D-W79-01..16 completada, implementación no iniciada) | Diseño técnico previo a implementación | Padre: WAB-75; dependencia WAB-78 satisfecha |

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

`AgentDefinition`, `AgentInstance`, `SkillConfiguration`, `AgentRules`, `ConversationMemory`, `OrganizationAICredit`, `CreditLedger`, `ExecutionBudgetAuthorization`, `ExecutionBudgetAllocation` y `WorkspaceKnowledge` (como entidad nombrada). `AgentExecutionLog` queda expresamente diferido al diseño posterior de observabilidad; no forma parte del modelo objetivo actual (D-W79-13).

### 3.4 Brechas identificadas (G1â€“G12)

Brechas conocidas (ausencia esperada de entidades objetivo) y brechas reales (diferencias entre implementación y contrato). Ver sección 4 del informe de reconciliación R1â€“R10.

---

## 4. B2 — Modelo de dominio y ownership

### 4.1 Diagrama de entidades y relaciones

```
WABY (PLATAFORMA)
  LLMRoute (AI Service DB; Waby)
        | 1:N
        v
  AgentDefinition (AI Service DB; referencia llmRouteId)
        â”‚ 1:N
        â–¼
WORKSPACE
  AgentInstance (CRM DB)
        â”‚
        â”œâ”€â”€ 1:N SkillConfiguration (CRM DB)
        â”œâ”€â”€ 1:N AgentRules (CRM DB)
        â”œâ”€â”€ 1:N ConversationMemory (AI Service DB)
        â””â”€â”€ 1:N AgentExecution (AI Service DB)

WORKSPACE â”€â”€ 1:N â”€â”€ WorkspaceKnowledge (AI Service DB)
      â”‚
      â””â”€â”€ compartido por todas las AgentInstances del Workspace

ORGANIZACIÃ“N â”€â”€ 1:N â”€â”€ Workspace (CRM DB)

CRM — AUTORIDAD DE ASIGNACIONES
  Contacto (CRM DB)
        â”œâ”€â”€ assignedType (human | agent | unassigned)
        â”œâ”€â”€ ref_user_assigned (Ref<UsOrgsWorks>)
        â””â”€â”€ ref_agent_assigned (Ref<AgentInstance>)
        +-- ref_ev_assigned (Ref<Evento>; último EvAssigned confirmado, incluso tras liberar)
  Evento (CRM DB) â† EvAssigned es discriminator, no colección propia
```

## 5. B3 — Persistencia, colecciones, campos y referencias

### 5.1 Matriz de ownership y persistencia

| Entidad | Propietario de negocio | Servicio responsable | DB de persistencia | Colección | Autoritativa / proyección |
|---------|----------------------|---------------------|-------------------|-----------|--------------------------|
| Organización | Waby | CRM | CRM DB | `organizaciones` | Autoritativa |
| Workspace | Organización | CRM | CRM DB | `workspaces` | Autoritativa |
| AgentDefinition | Waby | ai-service | AI Service DB | `agent_definitions` | Autoritativa; identidad estable y configuración funcional vigente sin versiones publicadas (D-W79-01). Referencia un `LLMRoute` para la configuración técnica del proveedor/modelo (D-W79-03) |
| LLMRoute | Waby | ai-service | AI Service DB | `llm_routes` | Autoritativa; configuración técnica centralizada del proveedor, modelo y parámetros permitidos; sin secretos (D-W79-03) |
| AgentInstance | Workspace | CRM | CRM DB | `agent_instances` | Autoritativa |
| SkillConfiguration | AgentInstance | CRM | CRM DB | `skill_configurations` | Autoritativa |
| AgentRules | AgentInstance | CRM | CRM DB | `agent_rules` | Autoritativa |
| WorkspaceKnowledge (modelo físico `KnowledgeBase`) | Workspace | ai-service | AI Service DB | `knowledge_bases` (existente) | Autoritativa; 1:N con KnowledgeDocument |
| KnowledgeDocument (modelo físico `Document`) | Workspace | ai-service | AI Service DB | `documents` (existente) | Autoritativa; cada documento pertenece a una KnowledgeBase |
| AgentExecution | AgentInstance + Workspace | ai-service | AI Service DB | `agent_executions` | Autoritativa |
| Observabilidad de ejecuciones | Por ejecución | Diseño posterior | Por definir | Sin entidad ni colección aprobada en WAB-79; diferida a diseño de observabilidad (D-W79-13) |
| ConversationMemory | AgentInstance + conversación | ai-service | AI Service DB | `conversation_memory` | Operativa |
| Contacto / estado actual de ContactAssignment | CRM / Workspace | CRM | CRM DB | `contactos` | Autoritativa; conserva `assignedType`, `ref_user_assigned`, `ref_agent_assigned` y `ref_ev_assigned` |
| `contact_assignments` | Derivado de ContactAssignment CRM | ai-service | AI Service DB | `contact_assignments` | Proyección operativa reconstruible; usa `assignedType` (`human`/`agent`) y no conserva vocabulario legacy; no autoritativa (D-W79-04) |
| EvAssigned | CRM | CRM | CRM DB | `eventos` (discriminator) | Historial autoritativo de transiciones |
| Outbox de asignación y retiro | CRM | CRM | CRM DB | `outbox` | Propagación durable de transiciones y órdenes |
| RetirementOperation | CRM | CRM | CRM DB | Por definir | Autoritativa; coordina y verifica el retiro |
| Confirmaciones de retiro | CRM | CRM | CRM DB | `retirement_confirmations` | Registro durable e idempotente de aplicación de EvAssigned por AI Service |

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
| `isDefault` | Boolean | Sí (default false) | — | Índice único parcial por `{workspaceId: 1}` con filtro `isDefault=true` |
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
| `agentInstanceId` | String / UUID | Sí | AgentInstance | Parte del índice único compuesto |
| `source` | Enum (`waby`, `workspace`) | Sí | Origen del Skill | Parte del índice único compuesto |
| `ownerId` | String / UUID | Sí | Identidad estable del propietario: valor genérico para `source=waby` (no se define identificador concreto de Waby en A2); `workspaceId` para `source=workspace` | Parte del índice único compuesto |
| `skillId` | String / UUID | Sí | Identidad estable del Skill dentro de su propietario | Parte del índice único compuesto |
| `skillName` | String | Sí | — | — |
| `enabled` | Boolean | No | Deshabilitación de la configuración; no equivale a baja funcional ni eliminación física | — |
| `config` | JSON | No | — | — |
| `createdAt` | Date | Sí | — | — |
| `updatedAt` | Date | Sí | — | — |

**Índice único:** `{agentInstanceId: 1, source: 1, ownerId: 1, skillId: 1}`. No se agregan índices secundarios por anticipado; cualquier índice adicional requiere una consulta runtime concreta que lo justifique. Se respetan las restricciones contractuales sobre índices de `SkillConfiguration` (D-W79-09).

**Restricciones:**
- Pertenencia a AgentInstance (1:N). Una misma identidad lógica de Skill puede configurarse en distintas AgentInstances, pero no duplicarse dentro de la misma instancia con igual combinación de origen y propietario.
- `source`, `ownerId` y `skillId` son identidad de origen y no deben cambiarse mediante una actualización ordinaria de configuración. `source=waby` identifica un Skill del catálogo central; `ownerId` usa un valor genérico estable para Waby (no se inventa identificador concreto en A2). `source=workspace` identifica un Skill privado y `ownerId` debe ser exactamente el `workspaceId` de la instancia. Un Skill de Workspace no puede referenciarse desde otro Workspace.
- Sin herencia dinámica de `defaultSkills` de AgentDefinition.
- Los Skills estándar de Waby conservan releases inmutables administradas por Waby. Los Skills propios del Workspace mantienen un único contenido vigente, sin historial de versiones ni rollback (D-W79-10).
- La actualización de un Skill propio afecta únicamente a ejecuciones aceptadas durablemente después de la actualización. La aceptación y persistencia de `AgentExecution` en `createdAt` (antes del ack) fija el contenido aplicable; `startedAt` indica únicamente el primer procesamiento efectivo del worker. Una ejecución ya aceptada conserva su copia consistente aunque permanezca en cola y no debe mezclar contenido anterior y nuevo. Se registra el hash, pero la copia se descarta al terminar y no permite recuperar contenido histórico.
- `SkillConfiguration` no almacena secretos ni referencias directas a credenciales empresariales. La asociación entre una conexión y su credencial protegida pertenece a `AgentInstanceIntegration` (D-W79-08).
- La relación exacta entre una Skill y las operaciones/capacidades de integración que puede utilizar no se define mediante un campo de credenciales ni queda fijada por esta decisión; su contrato operativo se mantiene dentro del alcance posterior de Skills/integraciones.
- La entidad/ubicación física que conserva el contenido fuente de un Skill propio y su vínculo exacto con `SkillConfiguration` deben definirse en el contrato de persistencia correspondiente; esta definición **NO bloquea** el schema/modelo `SkillConfiguration`, el índice contractual ya definido, los repositorios de persistencia y consulta de configuraciones, los servicios de dominio limitados a esas operaciones, ni las validaciones de pertenencia y aislamiento. A2 no implementará almacenamiento, referencias ni resolución del contenido de Skills propias.
- **No se introduce `contentRef` ni ninguna estructura para almacenar contenido de Skills dentro de A2.** El contenido de Skills (tanto Waby como propias) reside fuera de esta entidad; A2 solo persiste la configuración de habilitación y parámetros por instancia.
- **`enabled=false` es deshabilitación de la configuración, no eliminación física ni baja funcional.** La baja funcional de una Skill, la eliminación física del registro persistido y la coordinación de limpieza de recursos asociados corresponden a WAB-82. A2 persiste el atributo `enabled`, pero no implementa por sí mismo la semántica completa del ciclo de vida funcional.
- **Eliminación física de configuraciones de Skill excluida de A2.** Su semántica deberá definirse antes de implementarla; no se implementa en este incremento.
- **No se introduce default `enabled=true` ni ningún otro default no respaldado por el contrato.** La obligatoriedad del campo se elimina; su valor lo determina la operación funcional de WAB-82 al crear/habilitar la configuración.
- **`ownerId` para `source=waby` usa un valor genérico estable.** No se define el identificador concreto de Waby en A2; los schemas y repositorios genéricos no quedan bloqueados por esta definición.

**Contrato:** WAB-78 A.2, D.1.9; D-W79-08, D-W79-09 y D-W79-10.

#### 5.2.3 `agent_rules` (CRM DB)

| Campo | Tipo | Obligatorio | Referencia | Índice |
|-------|------|-------------|------------|--------|
| `_id` | String / UUID | Sí | — | PK |
| `agentInstanceId` | String / UUID | Sí | AgentInstance | `{agentInstanceId: 1}` |
| `ruleName` | String | Sí | — | — |
| `content` | String | Sí | — | — |
| `priority` | Integer | No | — | — |
| `enabled` | Boolean | No | Deshabilitación de la configuración; no equivale a baja funcional ni eliminación física | — |
| `createdAt` | Date | Sí | — | — |
| `updatedAt` | Date | Sí | — | — |

**Restricciones (D-W79-12):**
- Pertenencia a AgentInstance (1:N).
- `AgentRules` tiene identidad estable y contenido vigente único; no declara `version`, historial de versiones ni rollback.
- Las modificaciones actualizan el recurso existente y afectan a nuevas ejecuciones aceptadas después del cambio.
- `AgentExecution.effectiveConfigSnapshot` conserva la identidad, contenido y atributos efectivos de las Rules aplicadas. Las ejecuciones aceptadas mantienen su snapshot inmutable.
- No confundir con `BotResource` (catálogo global por workspace).
- **No se introduce índice único `{agentInstanceId, ruleName}` sin fundamento contractual explícito.** El índice declarado es `{agentInstanceId: 1}`; cualquier índice adicional requiere justificación concreta.
- **`enabled=false` es deshabilitación de la configuración, no eliminación física ni baja funcional.** La baja funcional de una Rule, la eliminación física del registro persistido y la coordinación de limpieza de recursos asociados corresponden a WAB-82. A2 persiste el atributo `enabled`, pero no implementa por sí mismo la semántica completa del ciclo de vida funcional.
- **Eliminación física de Rules excluida de A2.** Su semántica deberá definirse antes de implementarla; no se implementa en este incremento.
- **No se introduce default `enabled=true` ni ningún otro default no respaldado por el contrato.** La obligatoriedad del campo se elimina; su valor lo determina la operación funcional de WAB-82 al crear/habilitar la regla.

**Contrato:** WAB-78 A.2, D.1.9; D-W79-12.

#### 5.2.4 `agent_definitions` (AI Service DB)

| Campo | Tipo | Obligatorio | Índice |
|-------|------|-------------|--------|
| `_id` | String / UUID | Sí | PK |
| `name` | String | Sí | — |
| `description` | String | No | — |
| `systemPrompt` | String | Sí | — |
| `llmRouteId` | String / UUID (Ref<LLMRoute>) | Sí | — |
| `defaultSkills` | JSON[] | No | — |
| `defaultRules` | JSON[] | No | — |
| `status` | Enum (`draft`, `active`, `suspended`, `retired`) | Sí | `{status: 1}` |
| `createdBy` | String | Sí | — |
| `createdAt` | Date | Sí | — |
| `updatedAt` | Date | Sí | — |
| `retiredAt` | Date | No | — |

**Restricciones:**
- `_id` es la identidad lógica estable de la definición; no se introduce una entidad de versiones ni un campo `version`.
- `llmRouteId` referencia exactamente una `LLMRoute` administrada por Waby. Una `LLMRoute` puede ser reutilizada por varias AgentDefinitions (N:1 desde AgentDefinition).
- AgentDefinition no almacena `provider`, `model` ni parámetros técnicos del proveedor; estos pertenecen exclusivamente a LLMRoute.
- Los cambios en la configuración funcional vigente de AgentDefinition se reflejan en todas las AgentInstance que la referencian, sin modificar `agentDefinitionId`. Los cambios en la LLMRoute referenciada se aplican a nuevas ejecuciones de todas esas instancias.
- `draft` no es ejecutable; `active` permite nuevas ejecuciones; `suspended` bloquea nuevas ejecuciones de todas las instancias vinculadas; `retired` es terminal y no ejecutable.
- El runtime debe validar el estado vigente de AgentDefinition antes de iniciar cada ejecución. Las ejecuciones ya iniciadas conservan su snapshot; por defecto no se cancelan retroactivamente.
- Las modificaciones de configuración deben propagarse a todas las instancias vinculadas; cualquier caché debe invalidarse o refrescarse para que ninguna ejecución nueva use indefinidamente una configuración obsoleta.
- `defaultSkills`/`defaultRules` son plantillas de inicialización, no herencia dinámica.
- D-W79-01 elimina el versionado publicado de AgentDefinition y queda reconciliada con DEC-CM-01/C9 en el documento canónico WAB-78. La configuración vigente se aplica a nuevas ejecuciones de todas las instancias vinculadas; el snapshot de cada ejecución preserva los valores efectivos utilizados.

**Contrato:** WAB-78 A.2, D.1.9; D-W79-01 reconciliada con DEC-CM-01/C9 y P10/C.7; D-W79-03.

#### 5.2.4.1 `llm_routes` (AI Service DB)

| Campo | Tipo | Obligatorio | Índice |
|-------|------|-------------|--------|
| `_id` | String / UUID | Sí | PK |
| `name` | String | Sí | — |
| `provider` | Enum (`openai`, `openrouter`, `nvidia`, `gemini`) | Sí | — |
| `model` | String | Sí | — |
| `version` | Integer | Sí | — |
| `temperature` | Decimal | No | — |
| `maxTokens` | Integer | No | — |
| `topP` | Decimal | No | — |
| `frequencyPenalty` | Decimal | No | — |
| `presencePenalty` | Decimal | No | — |
| `createdAt` | Date | Sí | — |
| `updatedAt` | Date | Sí | — |

**Restricciones:**
- Entidad y configuración administradas exclusivamente por Waby; AgentDefinition solo referencia `llmRouteId`.
- `version` es la revisión de configuración y aumenta ante cada cambio material de proveedor, modelo o parámetros; AgentDefinition referencia la identidad estable de LLMRoute y las nuevas ejecuciones resuelven la revisión vigente.
- Los parámetros son una allowlist controlada por Waby. El conjunto inicial contempla `temperature`, `maxTokens`, `topP`, `frequencyPenalty` y `presencePenalty`; cualquier ampliación requiere incorporar explícitamente el parámetro a la allowlist y validar su soporte en el adaptador del proveedor.
- No se acepta un objeto arbitrario con opciones nativas del proveedor. No se almacenan secretos ni credenciales; estas permanecen en el almacén seguro administrado por Waby.
- Una modificación de LLMRoute afecta nuevas ejecuciones que resuelvan la ruta vigente. Las ejecuciones iniciadas conservan proveedor, modelo y parámetros efectivos capturados en su `effectiveConfigSnapshot`.
- No se define fallback automático a otro proveedor/modelo. Un fallo de la ruta se trata conforme a la política de errores/reintentos del proveedor, sin cambiar de ruta implícitamente.

**Contrato:** D-W79-03; WAB-78 A.2, D.1.9 y P10.

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

**Restricciones (D-W79-06):**
- `KnowledgeBase` es el modelo físico de dominio `WorkspaceKnowledge`; no es una segunda entidad ni requiere una colección adicional.
- Workspace 1:N WorkspaceKnowledge/`KnowledgeBase`.
- Las bases de conocimiento pertenecen al Workspace y pueden ser compartidas/utilizadas por sus AgentInstances mediante sus referencias `knowledgeBaseIds`; no pertenecen a una AgentInstance.
- No duplicar autoridad ni crear una colección editable equivalente en CRM.

**Contrato:** WAB-78 A.2, D.1.9, I11; D-W79-06.

**Evidencia actual:** `KnowledgeBase` en ai-service (`knowledge-base.model.ts`) cumple esta semántica con `workspaceId`.

#### 5.2.6 `documents` (AI Service DB)

| Campo | Tipo | Obligatorio | Referencia | Índice |
|-------|------|-------------|------------|--------|
| `_id` | String / UUID | Sí | — | PK |
| `workspaceId` | String | Sí | Workspace | `{workspaceId: 1}` |
| `knowledgeBaseId` | String / UUID | Sí | `KnowledgeBase` (modelo físico de WorkspaceKnowledge) | `{knowledgeBaseId: 1}` |
| `name` | String | Sí | — | — |
| `mimeType` | String | Sí | — | — |
| `size` | Number | Sí | — | — |
| `status` | Enum (`pending`, `processing`, `completed`, `failed`) | Sí | `{knowledgeBaseId: 1, status: 1}` |
| `createdAt` | Date | Sí | — | — |
| `updatedAt` | Date | Sí | — | — |

**Restricciones (D-W79-06 y D-W79-07):**
- `Document` es el modelo físico de dominio `KnowledgeDocument`; no es una segunda entidad ni requiere una colección adicional.
- WorkspaceKnowledge/`KnowledgeBase` 1:N KnowledgeDocument/`Document`; cada documento pertenece a exactamente una KnowledgeBase mediante `knowledgeBaseId` obligatorio.
- `workspaceId` se conserva como atributo de alcance/filtrado tenant y debe coincidir con el `workspaceId` de la KnowledgeBase referenciada.
- Toda creación o actualización de Document debe validar la existencia de la KnowledgeBase y que su `workspaceId` coincide con el del documento; la entidad `Document` no realiza consultas de infraestructura ni resuelve referencias por sí misma.
- Los flujos de ingesta, recuperación, eliminación y reingesta aplican las invariantes de pertenencia definidas en 6.2.5. Una inconsistencia impide continuar con efectos sobre documentos o vectores.
- No crear una colección o modelo paralelo para `KnowledgeDocument`.

**Contrato:** WAB-78 D.1.9 (Knowledge en AI Service DB); D-W79-06 y D-W79-07.

**Evidencia actual:** `Document` en ai-service (`document.model.ts`).

#### 5.2.7 `agent_executions` (AI Service DB)

| Campo | Tipo | Obligatorio | Referencia | Índice |
|-------|------|-------------|------------|--------|
| `_id` | String / UUID | Sí | Identidad lógica `executionId`; idempotency key | PK |
| `workspaceId` | String | Sí | Workspace | `{workspaceId: 1, createdAt: -1}` |
| `organizationId` | String / UUID | Sí | Organización (derivada de Workspace) | `{organizationId: 1, createdAt: -1}` |
| `agentInstanceId` | String / UUID | Sí | AgentInstance | `{agentInstanceId: 1, createdAt: -1}` |
| `conversationId` | String | Sí | — | `{agentInstanceId: 1, conversationId: 1, createdAt: -1}` |
| `contactId` | String | Sí | Contacto | — |
| `evAssignedId` | String / UUID | Sí | Identidad del `EvAssigned` que autorizó esta ejecución (CRM; referencia lógica cross-service) | — |
| `channel` | String | Sí | Canal de entrada | — |
| `input` | String | Sí | Mensaje de entrada normalizado | — |
| `correlationData` | Objeto (traceId, executionId, correlationId) | Sí | `correlationData.executionId` coincide con `_id` | — |
| `agentDefinitionId` | String / UUID | Sí | Identidad estable de AgentDefinition | `{agentDefinitionId: 1}` |
| `promptSnapshotHash` | String | Sí | SHA-256 del prompt efectivo construido | — |
| `effectiveConfigHash` | String | Sí | SHA-256 de la configuración efectiva completa | — |
| `effectiveConfigSnapshot` | JSON | Sí | Snapshot inmutable capturado al aceptar duraderamente la ejecución, antes del ack de entrada | — |
| `providerModelUsed` | String \| null | Condicional; solo si hubo invocación LLM | Proveedor/modelo realmente invocado, no solo resuelto | — |
| `inputTokens` | Integer \| null | Condicional al recibir usage informado | No inventar cero cuando el proveedor no informa el valor | — |
| `outputTokens` | Integer \| null | Condicional al recibir usage informado | No inventar cero cuando el proveedor no informa el valor | — |
| `totalTokens` | Integer \| null | Condicional cuando inputTokens y outputTokens están disponibles | Suma de inputTokens y outputTokens | — |
| `totalCreditsDeducted` | Decimal \| null | Condicional según liquidación; semántica WAB-85 | Dato de atribución; no es autoridad financiera | — |
| `currency` | String \| null | Condicional junto al importe liquidado; semántica WAB-85 | Moneda/unidad aplicable al importe | — |
| `latencyMs` | Integer \| null | Condicional si existen startedAt y finishedAt | Duración de procesamiento efectivo | — |
| `status` | Enum (`pending`, `completed`, `failed`, `cancelled`) | Sí | Estado lógico de la ejecución; sin estado `running` | — |
| `error` | String \| null | Condicional si falla; sanitizado | No incluir secretos, credenciales ni datos sensibles innecesarios | — |
| `createdAt` | Date | Sí | Momento de aceptación y persistencia durable | — |
| `startedAt` | Date \| null | Condicional; null hasta el inicio efectivo del worker | Se establece una sola vez; una recuperación no crea otro inicio | — |
| `finishedAt` | Date \| null | Obligatorio cuando status es terminal | Instante de terminación para completed, failed o cancelled | — |

**Restricciones:**
- `_id` es la identidad lógica `executionId`; `correlationData.executionId` debe coincidir con ese valor. No se introduce otro ID de ejecución ni se duplica `originMessageId` en esta colección: su unicidad y asociación pertenecen a AgentExecutionInbox según D.6.4.
- `createdAt` registra la aceptación y persistencia durable de la ejecución lógica. `effectiveConfigSnapshot`, `effectiveConfigHash` y `promptSnapshotHash` se capturan en ese mismo punto, antes del ack del mensaje de entrada, para que una ejecución pendiente no adopte una configuración posterior.
- `startedAt` registra el primer inicio efectivo del procesamiento. Puede permanecer null si la ejecución termina antes de iniciar un worker; al recuperar/reintentar la misma ejecución lógica no se crea un nuevo `executionId` ni se reemplaza el instante del primer inicio.
- `finishedAt` es null mientras la ejecución no sea terminal y obligatorio para `completed`, `failed` o `cancelled`. Si existe `startedAt`, `finishedAt` debe ser posterior o igual. No se conserva `completedAt`, porque la terminación no siempre equivale a éxito.
- `status` representa únicamente el resultado/estado lógico de AgentExecution: `pending`, `completed`, `failed` o `cancelled`. No se agrega `running`; recepción durable, procesamiento, lease, recuperación y estados de Outbox/Inbox pertenecen a AgentExecutionInbox y al contrato operativo de WAB-81.
- `providerModelUsed` se registra solo cuando se invocó realmente un proveedor LLM; resolver una ruta sin invocarla no satisface este campo.
- `inputTokens` y `outputTokens` conservan el usage efectivamente informado por el proveedor. `totalTokens` se calcula cuando ambos valores están disponibles. Un valor desconocido permanece null y nunca se convierte artificialmente en cero.
- `latencyMs` se calcula solo cuando existen `startedAt` y `finishedAt`; mide el procesamiento efectivo, no el tiempo de espera durable anterior al worker.
- `totalCreditsDeducted` y `currency` se completan conforme a la liquidación/estado económico definido en WAB-85. Son datos de atribución de AgentExecution, no autoridad ni sustituto del ledger CRM.
- `error` contiene información sanitizada; no incluye secretos, credenciales ni datos sensibles innecesarios.
- `effectiveConfigSnapshot` es obligatorio, inmutable y sin secretos. Captura `agentDefinitionId`, `llmRouteId`, `LLMRoute.version`, provider/model y parámetros técnicos efectivos, además de las demás referencias de configuración/contexto exigidas por P10. No se sustituye por hashes.
- La ruta y sus valores capturados no se vuelven a resolver durante la ejecución ni se cambian en reintentos de esa misma ejecución.
- `organizationId` es obligatorio para atribución/ownership según WAB-78 A.2; se valida contra la Organización propietaria del Workspace y no se agrega a AgentInstance.

**Contrato:** WAB-78 P10, C.7.

#### 5.2.8 Observabilidad de ejecuciones — diferida (D-W79-13)

No se define una entidad `AgentExecutionLog`, una colección `agent_execution_logs` ni persistencia de eventos internos por ejecución dentro de WAB-79. `AgentExecution` es el único registro persistente de dominio por ejecución y conserva estado, resultado, configuración efectiva y métricas consolidadas. El diseño posterior de observabilidad decidirá si se requieren logs, trazas, eventos técnicos, su almacenamiento, retención y correlación.

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
| `ref_ev_assigned` | Ref<Evento> | No inicialmente; después de la primera transición, siempre presente | Último `EvAssigned` confirmado en `eventos` (CRM DB) | — |
| `assignedType` | Enum (`human`, `agent`, `unassigned`) | Sí | — | — |

**Invariantes:**
- Si `assignedType=agent`: `ref_agent_assigned` obligatorio, `ref_user_assigned=null`.
- Si `assignedType=human`: `ref_user_assigned` obligatorio, `ref_agent_assigned=null`.
- Si `assignedType=unassigned`: `ref_user_assigned` y `ref_agent_assigned` son null; `ref_ev_assigned` conserva el último `EvAssigned` confirmado.
- `ref_ev_assigned` puede ser null únicamente antes de la primera transición confirmada; una liberación también genera un `EvAssigned` y actualiza esta referencia, que no vuelve a null.
- Contacto y AgentInstance deben pertenecer al mismo Workspace.

**Contrato:** WAB-78 D.1.5, E.

**Evidencia actual:** la implementación legacy observada dispone de `ref_user_assigned`, pero todavía no incorpora `ref_agent_assigned` ni `ref_ev_assigned`; su incorporación pertenece al diseño objetivo, no acredita implementación.

#### 5.2.11 `contact_assignments` (AI Service DB — proyección)

| Campo | Tipo | Obligatorio | Índice |
|-------|------|-------------|--------|
| `id` | String | Sí | PK |
| `contactId` | String | Sí | `{contactId: 1, status: 1}` único parcial donde `status='active'` |
| `workspaceId` | String | Sí | `{workspaceId: 1, status: 1}` |
| `assignedType` | Enum (`human`, `agent`) | Sí | `{assignedType: 1, assigneeId: 1, status: 1}` |
| `assigneeId` | String | Sí | Identificador del usuario cuando `assignedType='human'` o de AgentInstance cuando `assignedType='agent'`; se conserva en filas liberadas para identificar la asignación cerrada | — |
| `evAssignedId` | String / UUID \| null | Sí (nullable) | Identificador del `EvAssigned` que estableció la asignación activa; null cuando `status='released'` | — |
| `status` | Enum (`active`, `released`) | Sí | — |
| `createdAt` | Date | Sí | `{contactId: 1, createdAt: -1}` |

**Restricciones:**
- Cero o una asignación activa por contacto.
- `assignedType` utiliza exclusivamente el vocabulario objetivo `human` o `agent`; `unassigned` se representa por la ausencia de una fila activa, no por una fila de asignación.
- Si `assignedType='human'`, `assigneeId` identifica al usuario asignado; si `assignedType='agent'`, identifica a la `AgentInstance` asignada.
- Si `status='active'`, `evAssignedId` contiene el identificador del `EvAssigned` que estableció la asignación proyectada.
- Al proyectar una liberación, `status='released'` y `evAssignedId=null`; la fila conserva `assignedType` y `assigneeId` para identificar la asignación que se cerró. El evento de liberación permanece únicamente en `Contacto.ref_ev_assigned` y en el historial CRM.
- Una nueva asignación genera una nueva fila y proyecta su propio `evAssignedId`; no reutiliza el identificador de una asignación anterior.
- Proyección derivada de CRM; no es autoridad editable ni conserva el historial completo de `EvAssigned`.
- Los valores legacy `user` y `bot` no forman parte del esquema objetivo. No se define mapping ni migración de registros: la implementación legacy no llegó a producción y sus contratos se reemplazan.

**Contrato:** WAB-78 D.1.9, DEC-CM-06; D-W79-04.

#### 5.2.12 `EvAssigned` (discriminator de `Evento`, CRM DB)

`EvAssigned` no tiene colección propia: se persiste en `eventos` mediante discriminator y conserva el historial autoritativo de transiciones de asignación.

**Campos heredados de `Evento` (no se redeclaran):**

| Campo | Significado |
|---|---|
| `_id` | Identidad única del evento/transición. |
| `contacto` | Contacto afectado. |
| `workspace` | Workspace propietario del Contacto. |
| `createdAt` | Momento de registro de la transición; sustituye a `assignedAt`. |
| `updatedAt` | Timestamp de última modificación heredado. |
| `searchText` | Campo auxiliar de búsqueda heredado, si corresponde. |

**Campos propios:**

| Campo | Tipo / nulabilidad | Significado |
|---|---|---|
| `prevAssignedType` | `AssignedType` | Responsable anterior: `human`, `agent` o `unassigned`. |
| `prevAssignedRef` | String / ID nullable | ID del usuario o AgentInstance anterior; `null` si no había responsable. |
| `newAssignedType` | `AssignedType` | Nuevo responsable: `human`, `agent` o `unassigned`. |
| `newAssignedRef` | String / ID nullable | ID del usuario o AgentInstance nuevo; `null` si queda sin asignar. |
| `actorType` | Enum (`user`, `agent`, `system`) | Tipo de actor que produjo la transición. |
| `actorRef` | String / ID obligatorio | Identidad del actor; nunca `null`, incluidas operaciones automáticas del sistema. |
| `operationType` | Enum | `automatic_assignment`, `self_assignment`, `manual_assignment`, `takeover`, `delegation`, `unassignment`. |
| `reason` | String nullable | Motivo opcional de la transición. |

**Restricciones e índices (D-W79-14):**
- Reutiliza `_id`, `contacto`, `workspace`, `createdAt`, `updatedAt` y `searchText` de `Evento`; no duplica `contacto` ni declara `assignedAt`.
- Reutiliza los índices heredados `{contacto: 1, __t: 1, createdAt: -1}` y `{contacto: 1, createdAt: -1}`. No se agregan índices equivalentes; cualquier índice adicional requiere una consulta runtime identificada.
- `originMessageId` y `delegationChainId` no son campos de `EvAssigned`. `originMessageId` mantiene su función independiente en Inbox/mensajería.
- Cada transición confirmada crea un evento nuevo. `Contacto.ref_ev_assigned` conserva el último evento confirmado, incluso tras una liberación.

**Contrato:** D-W79-02 y D-W79-14; campos heredados e índices observados en `Evento`.

---

## 5.3 Catálogos de recursos y almacenamiento de objetos (D-W79-17)

El modelo de recursos distingue dos ámbitos lógicos y físicos, con ownership y contabilidad separados:

| Ámbito | Propietario | Espacio físico MinIO | Cuota |
|---|---|---|---|
| Catálogo central de recursos predefinidos Waby (conceptual; sin entidad/colección física definida) | Waby | Espacio MinIO reservado a recursos centrales Waby, con políticas de acceso propias | No consume cuota de Organización ni de Workspace |
| Recursos personalizados | Workspace que los crea | Espacio MinIO reservado a recursos Workspace, con aislamiento y políticas de acceso por Workspace | Consume exclusivamente la cuota del Workspace propietario |

**Reglas normativas:**
- Cada Workspace conserva su propio espacio de almacenamiento lógico y su cuota independiente. El consumo de un Workspace no se suma al de otros Workspaces de la misma Organización ni reduce sus cuotas.
- La Organización determina el plan comercial del que pueden derivarse los límites aplicables, pero no constituye una bolsa compartida de almacenamiento.
- La separación Waby/Workspace debe mantenerse tanto en el catálogo/metadatos como en los espacios físicos MinIO y sus políticas de acceso. Esta decisión no exige despliegues MinIO independientes ni fija nombres de buckets, prefijos, credenciales o topología física concreta.
- Los metadatos del catálogo central Waby son independientes de las configuraciones y metadatos de recursos propios de cada Workspace.
- Asociar, habilitar o referenciar un recurso del catálogo Waby desde un Workspace no transfiere su propiedad ni implica por sí mismo copiar su binario. Cualquier materialización/copia al ámbito Workspace debe ser una operación explícita y definida por el contrato del recurso.
- D-W79-11 conserva su alcance específico para Knowledge: cuando el flujo de incorporación de Knowledge Waby materialice contenido como `KnowledgeBase`/`Document` del Workspace, debe tratarse como una operación explícita de materialización. No se generaliza ese comportamiento a Skills u otros recursos ni se presupone duplicación física automática por el mero hecho de referenciar el recurso central.
- La auditoría del CRM observó cuotas y pools asociados individualmente a cada Workspace. El diseño objetivo conserva esa independencia; no declara que la cuota actual ya se aplique de forma agregada a nivel Organización ni certifica capacidad física disponible en MinIO.

**Límite de evidencia:** esta decisión define ownership, separación de espacios y contabilidad. No fija límites numéricos, política de ampliación, lifecycle de objetos, estrategia de replicación, nombres de buckets, despliegues MinIO ni mecanismo de reconciliación del contador de uso. La capacidad real y el consumo de producción no se deducen de la configuración inspeccionada.

**Contrato:** D-W79-17; D-W79-11 para la materialización específica de Knowledge.

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
| `agent_executions` | `{organizationId: 1, createdAt: -1}` | Compuesto | Historial y métricas por Organización |
| `agent_executions` | `{workspaceId: 1, createdAt: -1}` | Compuesto | Historial y métricas por Workspace |
| `agent_executions` | `{agentInstanceId: 1, createdAt: -1}` | Compuesto | Historial y métricas por AgentInstance |
| `agent_executions` | `{agentInstanceId: 1, conversationId: 1, createdAt: -1}` | Compuesto | Historial por conversación dentro de AgentInstance |
| `agent_executions` | `{agentDefinitionId: 1}` | Simple | Consultas por definición; no sustituye índices por tenant |
| `conversation_memory` | `{agentInstanceId: 1, conversationId: 1}` | Ãšnico compuesto | Una memoria por conversación y AgentInstance |
| `contactos` | `{ref_agent_assigned: 1}` | Simple (parcial donde no null) | Consultas por agente asignado |
| `contactos` | `{ref_user_assigned: 1}` | Simple | Consultas por usuario asignado |
| `contact_assignments` | `{contactId: 1, status: 1}` | Ãšnico parcial (`status='active'`) | Cero o una activa por contacto |
| `contact_assignments` | `{assignedType: 1, assigneeId: 1, status: 1}` | Compuesto | Consultas por tipo e identificador del asignado según D-W79-04 |

**Nota:** Ningún índice declarado se asume aplicado en producción sin verificación. La existencia de un índice en esta matriz es una propuesta de schema, no evidencia de despliegue.

**Limitaciones de evidencia B1:** no se verificó la aplicación de índices en producción ni el soporte efectivo de transacciones multidocumento en MongoDB Atlas. La estrategia de atomicidad de `isDefault` queda pendiente de validación de infraestructura.

**Aclaración sobre ARQ-06 / transacciones MongoDB Atlas:** ARQ-06 se refiere a la atomicidad del flujo `Contacto + EvAssigned + Outbox` (D-W79-15). La verificación de transacciones Atlas sigue siendo necesaria para ese flujo y para cualquier otra operación que realmente requiera atomicidad multidocumento. No obstante, no constituye por sí sola un bloqueo para implementar la persistencia base de `SkillConfiguration` y `AgentRules`, siempre que sus operaciones no dependan de esa transacción.

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
- `ref_ev_assigned` apunta al último `EvAssigned` confirmado de Contacto; solo puede ser null antes de la primera transición. Las asignaciones, reasignaciones, takeovers humanos y liberaciones generan un nuevo `EvAssigned`.
- Al liberar, `assignedType='unassigned'`, `ref_user_assigned=null` y `ref_agent_assigned=null`, pero `ref_ev_assigned` conserva el identificador del evento de liberación.
- Contacto y AgentInstance deben pertenecer al mismo Workspace.
- CRM es autoridad única; `contact_assignments` en ai-service es proyección derivada vía Outbox. En la proyección, `evAssignedId` contiene el evento de la asignación activa y pasa a null al liberarse.
- Cada `AgentExecution.evAssignedId` es obligatorio e inmutable; CRM compara ese valor con `Contacto.ref_ev_assigned` antes de cualquier efecto externo.

**Contrato:** WAB-78 D.1.5, E, DEC-CM-06.

#### 6.2.4 Eliminación y referencias huérfanas

- `retired` representa retiro lógico terminal de `AgentInstance`; no equivale a purga física. La retención y eventual purga física de la instancia y sus datos asociados debe respetar las políticas de Waby y los contratos de retención; no eliminar registros de ejecución ni snapshots por cascada.
- **Eliminación física de `SkillConfiguration` y `AgentRules` excluida de A2 (WAB-79).** La deshabilitación de la configuración (`enabled=false`) no equivale a eliminación física ni a baja funcional. La semántica de eliminación física, coordinación de limpieza de recursos asociados y baja funcional corresponden a WAB-82, que consume los servicios de persistencia de A2.
- Eliminación de `WorkspaceKnowledge`: no afecta a `AgentInstance` que la referencien (referencias pueden volverse obsoletas; gestión en WAB-82).
- `BotResourceLink` con `botId` string plano sin Ref (cross-DB): riesgo de links huérfanos si se elimina un Bot en ai-service; gestión en GC (fuera de alcance WAB-79).

#### 6.2.5 Integridad de pertenencia Document ↔ KnowledgeBase ↔ Workspace (D-W79-07)

- La identidad de dominio de un documento queda determinada por su `workspaceId` y `knowledgeBaseId`; ambos deben corresponder a una `KnowledgeBase` existente cuyo `_id` coincida con `knowledgeBaseId` y cuyo `workspaceId` sea idéntico al del documento.
- Toda creación o actualización de `Document` valida la relación contra la KnowledgeBase persistida, usando una consulta acotada por `_id` y `workspaceId` (o una comprobación equivalente que garantice ambos valores). No se delega integridad referencial a `Document.validate` ni se introducen consultas de repositorio en la entidad.
- Antes de ingerir un documento, tanto en v1/v2 como en consumers RabbitMQ, se recupera el documento dentro del Workspace esperado y se comprueba que el `knowledgeBaseId` del contexto/mensaje coincide con el del documento persistido; además se comprueba que la KnowledgeBase asociada pertenece a ese Workspace. Ante desajuste se rechaza el procesamiento antes de descargar contenido, generar chunks o escribir vectores.
- La recuperación RAG valida en una operación de repositorio agrupada que todos los `knowledgeBaseIds` solicitados existen y pertenecen al `workspaceId` del contexto. Si falta alguno, existe uno ajeno o la cantidad recuperada no coincide con la solicitada, se rechaza la consulta completa; no se descartan IDs silenciosamente ni se devuelve un resultado parcial que oculte la inconsistencia.
- El filtro de `workspaceId` de metadata es obligatorio en cada búsqueda vectorial y se deriva del contexto tenant validado. El caller no puede omitirlo ni sustituirlo por otro valor. La colección Qdrant namespaced por Workspace es defensa adicional, no reemplazo de la validación de pertenencia.
- Eliminación y reingesta recuperan el documento por `documentId + workspaceId` y verifican la coherencia de su `knowledgeBaseId` con la KnowledgeBase del mismo Workspace antes de purgar o escribir vectores. Si existe una inconsistencia, no se ejecutan efectos destructivos ni de persistencia vectorial y se informa un error controlado para su reconciliación.
- Las inconsistencias se rechazan de forma fail-closed; nunca se convierten silenciosamente en una búsqueda vacía ni permiten continuar con un contexto ambiguo.
- La validación pertenece a los casos de uso/servicios de aplicación y al límite del adaptador vectorial para el filtro tenant; no se incorporan dependencias de MongoDB o repositorios a entidades de dominio/value objects.
- D-W79-07 define estas invariantes, no su implementación concreta, índices adicionales, reparación de registros históricos ni migración de datos. Esos detalles requieren evidencia y planificación antes de su ejecución.

**Contrato:** D-W79-06 y D-W79-07; WAB-78 D.1.9, D.3 y H.4.

## 7. Correspondencia entre modelos legacy y objetivo

| Legacy (actual) | Objetivo (WAB-79) | Notas |
|-----------------|-------------------|-------|
| `Bot` (ai-service) | Reemplazado por `AgentDefinition` (AI Service DB) y `AgentInstance` (CRM DB) | Separar definición e instancia; implementar el modelo nuevo sin migrar registros ni conservar el contrato legacy |
| `BotInstance` (ai-service) | Reemplazado por `AgentInstance` (CRM DB) | Nueva persistencia y contrato; no trasladar registros ni mantener compatibilidad hacia atrás |
| `BotResource` / `BotResourceLink` (CRM) | Reemplazados por `SkillConfiguration` + `AgentRules` (CRM DB) | Crear recursos según el modelo nuevo; retirar catálogo, links y reglas legacy sin migrar datos |
| `KnowledgeBase` (ai-service) | `KnowledgeBase` (AI Service DB; concepto WorkspaceKnowledge) | El modelo/colección física puede reutilizarse si satisface D-W79-06/07; no implica migrar datos legacy |
| `Document` (ai-service) | `Document` (AI Service DB; concepto KnowledgeDocument) | El modelo/colección física puede reutilizarse si satisface D-W79-06/07; no implica migrar datos legacy |
| `Execution` (ai-service) | Nuevo `AgentExecution` (AI Service DB) | Diseñar según D-W79-05; no ampliar ni migrar la entidad o las ejecuciones legacy |
| `ExecutionEvent` (ai-service) | Observabilidad de ejecuciones (diseño posterior) | No se incorpora como entidad persistente de WAB-79; evaluar en el diseño futuro de observabilidad (D-W79-13) |
| `MemoryFact` / `MemoryRecord` (ai-service) | Nuevo `ConversationMemory` (AI Service DB) | Nuevo contrato por AgentInstance+conversación; no migrar datos ni adaptar el dominio legacy |
| `Contacto` (CRM) | `Contacto` (CRM) | Agregar `ref_agent_assigned`, `assignedType` y `ref_ev_assigned` (último `EvAssigned` confirmado) |
| `ContactAssignment` (ai-service proyección legacy) | Nueva proyección `contact_assignments` (AI Service DB) | Implementar el contrato objetivo con `assignedType` (`human`/`agent`) y `assigneeId`; ausencia de fila activa representa `unassigned`. Al liberar conserva tipo/identificador y establece `evAssignedId=null`. No se migran ni traducen registros legacy; CRM es autoridad (D-W79-02/04) |
| `EvAssigned` (CRM evento) | `EvAssigned` (discriminator de Evento en `eventos`) | Mantener como discriminator; el contrato requiere datos suficientes para identificar transición, actor, momento y contexto |

## 8. Cuestiones técnicas pendientes de resolución

Las siguientes cuestiones técnicas permanecen abiertas sin inventar decisiones. La identidad estable y ausencia de versiones publicadas de AgentDefinition ya están aprobadas mediante D-W79-01 y reconciliadas con DEC-CM-01/C9 en el documento canónico WAB-78; no deben reabrirse en WAB-79.

**Puntos ya resueltos y retirados de esta lista:**
- **Correspondencia conceptual/física — D-W79-06:** `WorkspaceKnowledge` → `KnowledgeBase` (`knowledge_bases`) y `KnowledgeDocument` → `Document` (`documents`), junto con las cardinalidades Workspace 1:N WorkspaceKnowledge y WorkspaceKnowledge 1:N KnowledgeDocument. No se crean modelos o colecciones duplicados.
- **Cuestión pendiente de credenciales — D-W79-08:** separación entre credenciales LLM Waby-owned y credenciales de integraciones empresariales administradas por Workspace; la referencia segura de credencial se sitúa en `AgentInstanceIntegration`, no en `SkillConfiguration`. Se elimina `SkillConfiguration.credentials` del modelo objetivo; no se decide aún el almacenamiento físico ni el campo/cardinalidad de asociación Skill–operación.
- **Cuestión #6 — D-W79-07:** integridad de pertenencia Document ↔ KnowledgeBase ↔ Workspace en escrituras, ingesta v1/v2 y RabbitMQ, recuperación RAG, eliminación y reingesta. Se valida en capas de aplicación y se exige un filtro vectorial de Workspace obligatorio; cualquier inconsistencia se rechaza fail-closed.
- El índice único compuesto de `ConversationMemory` por `{agentInstanceId: 1, conversationId: 1}` está definido en 5.2.9 y 6.1. La cardinalidad correspondiente también está fijada en WAB-78 A.2; no requiere una nueva decisión.
- D-W79-04 fija el vocabulario objetivo de la proyección `contact_assignments`: `assignedType` con valores `human`/`agent`; `unassigned` se representa por ausencia de una fila activa. Las filas liberadas conservan el tipo e identificador de la asignación cerrada, mientras `evAssignedId` queda null. No existe mapping/migración desde la proyección legacy; se reemplaza por el contrato nuevo.
- D-W79-05 fija el contrato de `AgentExecution`: identidad `executionId`, timestamps `createdAt`/`startedAt`/`finishedAt`, métricas y costes condicionales, estados lógicos de Fase 2, separación respecto de AgentExecutionInbox e índices cronológicos compuestos.
- **D-W79-09:** identidad estructurada e índice único compuesto de `SkillConfiguration` definidos en 5.2.2; la cuestión original #1 sobre cardinalidad e índice queda resuelta y no debe repetirse como pendiente.

2. **Semántica del versionado de AgentRules:** resuelta por D-W79-12; no existe campo `version`, historial ni rollback.
3. **Autoridad y función de AgentExecutionLog:** resuelto por D-W79-13; queda diferido al diseño de observabilidad y fuera del modelo objetivo actual.
4. **Unificación de los índices de asignación de Contacto.**

La cuestión sobre campos heredados de `Evento`, campos propios e índices de `EvAssigned` queda resuelta por D-W79-14 y se documenta en 5.2.12; no permanece como pendiente.
6. **Estrategia técnica de atomicidad de `isDefault`.**
7. **Inventario y limpieza de `BotResourceLink` y demás referencias legacy:** determinar qué código, colecciones, endpoints, configuraciones y documentación quedan sin consumidor bajo el modelo nuevo y retirarlos. Es una tarea de reemplazo/limpieza, no de migración de datos.

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
| D-W79-02 | Identidad de transición de asignaciones: cada asignación, reasignación, takeover humano y liberación confirmados generan `EvAssigned`; `Contacto.ref_ev_assigned` conserva el último evento confirmado incluso tras liberar; la proyección `contact_assignments.evAssignedId` contiene el identificador de la asignación activa y pasa a null al liberar; `AgentExecution.evAssignedId` es obligatorio e inmutable y se propaga en el envelope. CRM valida la igualdad con `Contacto.ref_ev_assigned` antes de cualquier efecto externo. | Aprobada por el usuario | Revisión WAB-79, 09/11/2026 |
| D-W79-03 | Separar la configuración técnica LLM en `LLMRoute`, entidad Waby-owned y persistida en AI Service DB. `AgentDefinition` referencia una ruta lógica mediante `llmRouteId`, sin almacenar provider/model/parámetros; Waby controla una allowlist explícita y la revisión propia `LLMRoute.version`. `AgentInstance` no puede sobrescribir ruta ni parámetros. El snapshot de cada ejecución conserva identidad, revisión y valores efectivos de la ruta. No hay fallback automático entre proveedores/modelos. | Aprobada por el usuario | Revisión WAB-79, 09/11/2026 |
| D-W79-04 | La proyección `contact_assignments` de AI Service utiliza `assignedType` con valores `human`/`agent`; `assigneeId` identifica al usuario o AgentInstance correspondiente. `unassigned` se representa por ausencia de una fila activa. Las filas `released` conservan tipo e identificador de la asignación cerrada y llevan `evAssignedId=null`. CRM sigue siendo la única autoridad y el historial de `EvAssigned` no se replica. No se realiza mapping ni migración de filas legacy: la implementación de IA anterior no llegó a producción y la proyección se reemplaza por el contrato objetivo. | Aprobada por el usuario; aclaración de reemplazo sin migración el 10/11/2026 | Revisión WAB-79, 09/11/2026; aclaración 10/11/2026 |
| D-W79-05 | `AgentExecution` se identifica por `_id=executionId` y distingue aceptación durable (`createdAt`), primer inicio efectivo (`startedAt`, nullable) y terminación (`finishedAt`, obligatorio en estados terminales). Mantiene los estados lógicos `pending`, `completed`, `failed` y `cancelled`; los estados de procesamiento/recuperación pertenecen a AgentExecutionInbox. `channel` e `input` son obligatorios; proveedor, usage, latencia y costes son condicionales a la información realmente disponible. Los costes siguen sujetos a WAB-85 y al ledger CRM. Se adoptan índices cronológicos compuestos por Organización, Workspace, AgentInstance y conversación, además del índice por AgentDefinition. | Aprobada por el usuario | Revisión WAB-79, 09/11/2026 |
| D-W79-06 | `WorkspaceKnowledge` se persiste mediante el modelo físico `KnowledgeBase` (`knowledge_bases`) y `KnowledgeDocument` mediante `Document` (`documents`), ambos en AI Service DB y sin modelos/colecciones duplicados. Cardinalidades: Workspace 1:N WorkspaceKnowledge y WorkspaceKnowledge 1:N KnowledgeDocument; cada KnowledgeDocument pertenece a exactamente una WorkspaceKnowledge mediante `knowledgeBaseId`. `Document.workspaceId` se conserva como atributo de alcance y debe corresponder al workspace de su KnowledgeBase; la validación técnica de esa coherencia se define en D-W79-07. | Aprobada por el usuario | Revisión WAB-79, 09/11/2026 |
| D-W79-07 | Integridad de pertenencia Document ↔ KnowledgeBase ↔ Workspace: escrituras validan la KnowledgeBase persistida por ID y Workspace; ingesta v1/v2 y RabbitMQ verifican la coincidencia entre el documento persistido, su KnowledgeBase y el contexto/mensaje; recuperación RAG valida en lote que todas las KnowledgeBases solicitadas pertenecen al Workspace y aplica siempre filtro vectorial obligatorio derivado del contexto; eliminación y reingesta validan antes de efectos sobre documentos/vectores. Las inconsistencias se rechazan fail-closed. Validaciones en capas de aplicación y límite vectorial, sin consultas de infraestructura en entidades de dominio. No se decide implementación de índices, reparación ni migración de registros históricos. | Aprobada por el usuario | Revisión WAB-79, 09/11/2026 |
| D-W79-08 | Separación de dominios de credenciales: Waby administra por separado las credenciales LLM; `AgentDefinition` referencia una `LLMRoute` que identifica proveedor/modelo y parámetros técnicos permitidos, sin contener secretos. El modelo legacy `Bot.aiCredential`/API key aportada por cliente queda fuera del objetivo. Las credenciales empresariales son administradas por usuarios autorizados del Workspace; su referencia protegida pertenece a `AgentInstanceIntegration`, conexión privada por defecto de una AgentInstance. Waby opera la custodia segura y resolución. Se elimina `SkillConfiguration.credentials`; no se decide el almacenamiento físico, tecnología de cifrado/Vault/KMS ni el vínculo persistido Skill–operación. Los secretos nunca entran en snapshots, prompts, logs ni ledger; las referencias/versiones no secretas pueden conservarse para trazabilidad. | Aprobada por el usuario | Revisión WAB-79, 09/11/2026 |
| D-W79-09 | Identidad y cardinalidad de `SkillConfiguration`: la identidad lógica de cada configuración se compone de `agentInstanceId`, `source` (`waby`/`workspace`), `ownerId` y `skillId`. La combinación es inmutable y única por AgentInstance mediante el índice compuesto `{agentInstanceId: 1, source: 1, ownerId: 1, skillId: 1}`. Para Skills Waby, `ownerId` identifica a Waby; para Skills propios, `ownerId` es el `workspaceId`. No se agregan índices secundarios sin evidencia de una consulta runtime que los requiera. | Aprobada por el usuario | Revisión WAB-79, 09/11/2026 |
| D-W79-10 | Ciclo de vida de Skills propios del Workspace: existe un único contenido vigente, sin historial de versiones ni rollback. Una actualización afecta únicamente a ejecuciones aceptadas durablemente después de ella. Para mantener la consistencia con P10, el contenido queda fijado en la aceptación durable de la ejecución (`createdAt`, antes del ack); `startedAt` sigue representando el primer procesamiento efectivo del worker. La copia de trabajo se mantiene disponible solo durante la vida de esa ejecución; se registra su hash, pero no se conserva el contenido histórico al finalizar ni se promete su recuperación/reproducción. Las releases inmutables de Skills estándar Waby permanecen bajo el régimen separado del catálogo central. | Aprobada por el usuario | Revisión WAB-79, 09/11/2026 |
| D-W79-11 | Incorporación de Knowledge publicado por Waby: el contenido que Waby publique para uso de un Workspace se materializa como recursos Knowledge del propio Workspace, en sus `KnowledgeBase`/`Document` de AI Service DB, manteniendo separadas la publicación central y la copia incorporada. La política de sincronización, actualización, obsolescencia y retirada de copias materializadas queda pendiente de diseño. | Aprobada por el usuario | Revisión WAB-79, 09/11/2026 |
| D-W79-12 | `AgentRules` mantiene identidad estable y contenido vigente único, sin campo `version`, historial ni rollback. Las modificaciones actualizan el recurso existente y afectan a nuevas ejecuciones aceptadas después del cambio. `AgentExecution.effectiveConfigSnapshot` conserva identidad, contenido y atributos efectivos de las Rules aplicadas; los snapshots aceptados son inmutables. | Aprobada por el usuario | Revisión WAB-79, 10/11/2026 |
| D-W79-13 | `AgentExecutionLog` se difiere al diseño del módulo de observabilidad WABY y queda fuera del modelo objetivo actual de WAB-79. No se crea colección ni persistencia de eventos internos por ejecución. `AgentExecution` es el único registro persistente de dominio por ejecución y conserva estado, resultado, configuración y métricas consolidadas. El diseño futuro decidirá logs, trazas y su mecanismo. | Aprobada por el usuario | Revisión WAB-79, 10/11/2026 |
| D-W79-14 | `EvAssigned` reutiliza los campos heredados de `Evento` (`_id`, `contacto`, `workspace`, `createdAt`, `updatedAt`, `searchText`) y sus índices existentes `{contacto, __t, createdAt}` y `{contacto, createdAt}`, sin duplicar `contacto` ni `assignedAt`. Sus campos propios son `prevAssignedType`, `prevAssignedRef`, `newAssignedType`, `newAssignedRef`, `actorType`, `actorRef`, `operationType` y `reason`. `actorRef` es obligatorio/no nullable; `reason` es nullable. `operationType`: `automatic_assignment`, `self_assignment`, `manual_assignment`, `takeover`, `delegation`, `unassignment`. Se excluyen `originMessageId` y `delegationChainId`. | Aprobada por el usuario | Revisión WAB-79, 10/11/2026 |
| D-W79-15 | Cada transición de asignación persiste `Contacto`, `EvAssigned` y el registro Outbox correspondiente dentro de una única transacción MongoDB. El ID de `EvAssigned` se genera antes del CAS y se reutiliza en las escrituras relacionadas. El CAS valida el estado anterior completo relevante de `Contacto` (`assignedType`, `ref_user_assigned`, `ref_agent_assigned` y `ref_ev_assigned`), usando esta última referencia como token de concurrencia; si no coincide o falla cualquier escritura, se aborta la transacción completa. No se permiten escrituras secuenciales como degradación silenciosa. Las comunicaciones externas y la actualización de proyecciones ocurren después del commit mediante procesamiento asíncrono al menos una vez; los consumidores deben ser idempotentes por `evAssignedId`. Las tres colecciones pertenecen a la misma base de datos MongoDB Atlas (confirmación del usuario). Antes de implementar debe verificarse que el clúster Atlas utilizado tenga una topología compatible con transacciones multidocumento (replica set o clúster sharded) y que todas las operaciones propaguen la misma sesión MongoDB. | Aprobada por el usuario; viabilidad de infraestructura pendiente de verificación | Revisión WAB-79, 10/11/2026 |
| D-W79-16 | Un Workspace puede tener cero o un `AgentInstance` con `isDefault=true`, independientemente del estado del ciclo de vida (`draft`, `active`, `suspended`, `retirement_pending`, `retired`). El cambio de predeterminado es transaccional y se protege con un índice único parcial por `workspaceId` para documentos con `isDefault=true`. Las solicitudes concurrentes que compiten por establecer el predeterminado se resuelven mediante conflicto, sin estado parcial. Cambiar `isDefault` no altera asignaciones existentes. Suspender la instancia predeterminada conserva `isDefault`; no se selecciona otra instancia automáticamente. | Aprobada por el usuario | Revisión WAB-79, 10/11/2026 |
| D-W79-17 | Separación de catálogos y almacenamiento de recursos: Waby mantiene un catálogo central de recursos predefinidos y los Workspaces mantienen sus recursos personalizados. Ambos ámbitos utilizan espacios físicos MinIO separados y políticas de acceso diferenciadas. Los recursos Waby no consumen cuotas de clientes; cada Workspace conserva su cuota independiente y solo contabiliza sus recursos propios. La Organización determina el plan comercial, pero no comparte una bolsa de almacenamiento entre sus Workspaces. Referenciar un recurso Waby no transfiere propiedad ni implica copia física automática; cualquier materialización debe ser explícita. La materialización de Knowledge definida en D-W79-11 se limita a su flujo específico y no se generaliza a otros recursos. | Aprobada por el usuario | Revisión WAB-79, 10/11/2026 |

**Trazabilidad:** D-W79-01 fue aprobada para el diseño WAB-79 y reconciliada en el documento canónico WAB-78 (`fase-2-diseno-objetivo-y-contratos.md`), en DEC-CM-01/C9 y el contrato de snapshot P10/C.7. La referencia estable es `AgentDefinition._id`; no se mantiene `agentDefinitionVersion`.

D-W79-02 se incorpora en las secciones 5.2.7, 5.2.10, 5.2.11 y 6.2.3 de este documento, y se reconcilia con D.1.5–D.1.8, D.6.2–D.6.4 y E.4.12 de Fase 2. No introduce `assignmentVersion` ni sincroniza el historial completo de `EvAssigned` a AI Service.

D-W79-03 se incorpora en la matriz 5.1, el diagrama 4.1 y las definiciones 5.2.4/5.2.4.1. La reconciliación en Fase 2 actualiza A.2, B, D.1.9, E.1, H.2/H.3 y ADR-005. No se introduce configuración técnica editable por Workspace ni fallback implícito.

D-W79-04 actualiza el contrato objetivo de `contact_assignments` en 5.1, 5.2.11 y 6.1, y su relación conceptual con la proyección anterior en la sección 7. La reconciliación puntual de Fase 2 explicita el vocabulario `assignedType` (`human`/`agent`), la semántica de `unassigned` como ausencia de asignación activa y la conservación de `assignedType`/`assigneeId` en filas liberadas. No hay conversión ni migración de valores legacy; la proyección anterior se reemplaza.

D-W79-05 actualiza el esquema de `agent_executions` en 5.2.7 y sus índices en 6.1. La reconciliación de Fase 2 actualiza A.2. Se diferencia el momento de aceptación durable (`createdAt`), el inicio efectivo (`startedAt`) y la terminación de cualquier resultado (`finishedAt`). Las métricas de proveedor/tokens/latencia y la liquidación quedan condicionales a datos reales; la semántica financiera detallada permanece en WAB-85. La identidad y los estados operativos de recepción/recuperación permanecen en AgentExecutionInbox según D.6 y WAB-81.

D-W79-06 fija la correspondencia conceptual/física y la cardinalidad del dominio Knowledge: `WorkspaceKnowledge` corresponde al modelo físico `KnowledgeBase`/`knowledge_bases`, y `KnowledgeDocument` al modelo físico `Document`/`documents`, ambos bajo autoridad de AI Service DB. Se documenta Workspace 1:N WorkspaceKnowledge y WorkspaceKnowledge 1:N KnowledgeDocument, sin crear entidades persistidas duplicadas.

D-W79-07 cierra la integridad de pertenencia Document ↔ KnowledgeBase ↔ Workspace: valida las referencias en escrituras, ingesta y mensajería asíncrona; valida en lote las KnowledgeBases habilitadas para recuperación RAG y exige el filtro vectorial de Workspace; y protege eliminación/reingesta antes de efectos sobre vectores. Los rechazos son fail-closed y las comprobaciones permanecen en capas de aplicación/infraestructura apropiadas, sin hacer impuras las entidades de dominio.

D-W79-08 separa el modelo legacy de API keys LLM aportadas por clientes del modelo objetivo: Waby administra por separado las credenciales LLM; `AgentDefinition` referencia la configuración técnica `LLMRoute`, que no almacena secretos. El Workspace administra credenciales de integraciones empresariales, cuya referencia segura se conserva en `AgentInstanceIntegration`. `SkillConfiguration` no contiene credenciales ni referencias directas a ellas. La decisión establece ownership, custodia y límites de exposición, pero no selecciona almacenamiento físico, mecanismo criptográfico ni proveedor Vault/KMS, y no define el campo/cardinalidad de asociación Skill–operación. La reconciliación se aplica en conjunto a WAB-79, Fase 2 y el diseño de Integraciones API externas.

D-W79-09 fija la identidad estructurada y la unicidad de `SkillConfiguration` por instancia, separando explícitamente el origen Waby del origen Workspace y evitando colisiones entre propietarios. D-W79-10 limita la ausencia de historial/rollback exclusivamente a Skills propios del Workspace; no altera las releases inmutables del catálogo Waby. Para ejecuciones en curso se exige consistencia del contenido cargado al inicio, con hash persistido y sin retención histórica del cuerpo del Skill propio. La entidad fuente/ubicación física del contenido propio queda pendiente de especificación.

D-W79-11 establece que el Knowledge publicado por Waby se incorpora mediante materialización dentro del ámbito de Knowledge del Workspace. No define sincronización automática ni política de retirada/obsolescencia; esos comportamientos permanecen pendientes.

D-W79-12 fija `AgentRules` como recurso de identidad estable y contenido vigente único, sin versionado, historial ni rollback. Las modificaciones afectan a nuevas ejecuciones aceptadas; `effectiveConfigSnapshot` conserva identidad, contenido y atributos efectivos de las Rules aplicadas.

D-W79-13 excluye `AgentExecutionLog` del modelo objetivo actual: no se crea colección ni persistencia de eventos internos por ejecución. `AgentExecution` conserva el registro de dominio y sus métricas consolidadas; logs/trazas y su retención se decidirán en el diseño futuro de observabilidad.

D-W79-14 define el contrato de campos de `EvAssigned`, reutiliza los campos e índices heredados de `Evento` y descarta `originMessageId` y `delegationChainId` como campos del evento. No duplica `assignedAt`; `createdAt` representa el momento de registro de la transición.

D-W79-15 fija la unidad atómica de cada transición de asignación en CRM: CAS de `Contacto`, inserción de `EvAssigned` e inserción de Outbox dentro de una transacción MongoDB. `ref_ev_assigned` participa en la condición CAS y evita aceptar una transición basada en un estado anterior obsoleto. AI Service y los efectos externos quedan fuera de la transacción CRM y se actualizan después del commit mediante entrega asíncrona idempotente. La compatibilidad real de producción con transacciones y la conexión compartida de las colecciones son una condición técnica de implementación todavía no verificada; no se autoriza degradar silenciosamente a escrituras secuenciales.

D-W79-16 define la unicidad y concurrencia de `isDefault`: máximo uno por Workspace en todos los estados del ciclo de vida, operación transaccional e índice único parcial por `workspaceId`. No se reasignan contactos ni se elige un reemplazo automático si la instancia predeterminada se suspende.

D-W79-17 separa el catálogo central Waby de los recursos personalizados de Workspace en ownership, metadatos y espacios físicos MinIO. Los recursos Waby quedan fuera de cuotas de clientes; cada Workspace mantiene una cuota independiente, sin bolsa organizacional compartida. La mera referencia a un recurso Waby no cambia su propiedad ni fuerza una copia física. D-W79-11 se interpreta como el flujo explícito y específico de materialización de Knowledge, no como una regla genérica para todo recurso Waby.

## 10. Criterios de aceptación para autorizar implementación

Antes de iniciar la implementación de WAB-79, debe verificarse:

1. **Esquemas `AgentDefinition` y `LLMRoute` definidos** en AI Service DB: AgentDefinition conserva identidad estable y referencia `llmRouteId`; LLMRoute centraliza proveedor/modelo/parámetros allowlisted bajo control Waby, sin secretos ni fallback automático (D-W79-01 y D-W79-03).
2. **Esquema `AgentInstance` definido** en CRM DB con `workspaceId`, `agentDefinitionId`, `isDefault`, estados P9, sin `orgId`, sin unicidad `workspaceId + agentDefinitionId` (contratos C2, C3, C4, C5).
3. **Unicidad de `isDefault` (C5 y D-W79-16):** índice único parcial por `{workspaceId: 1}` con filtro `isDefault=true` (equivalente funcional a `{workspaceId: 1, isDefault: 1}` parcial), aplicable sin filtrar por estado; operación de cambio transaccional y conflicto controlado ante concurrencia. Suspender la instancia no limpia `isDefault` ni selecciona otra.
4. **Esquemas `SkillConfiguration` y `AgentRules`** en CRM DB con `agentInstanceId` (contrato C6). `SkillConfiguration` debe incorporar `source`, `ownerId` y `skillId`, además del índice único compuesto `{agentInstanceId: 1, source: 1, ownerId: 1, skillId: 1}`; no se agregan índices secundarios sin justificación de consultas reales (D-W79-09). Debe respetar la política de Skills propios sin historial/rollback y snapshot con hash (D-W79-10).
5. **Persistencia e integridad de Knowledge** en AI Service DB mediante `KnowledgeBase`/`knowledge_bases` (`WorkspaceKnowledge`) y `Document`/`documents` (`KnowledgeDocument`), sin duplicados CRM ni colecciones paralelas. Deben cumplirse las cardinalidades de D-W79-06 y los controles de pertenencia en escritura, ingesta v1/v2, RabbitMQ, recuperación RAG y eliminación/reingesta, con filtro vectorial de Workspace obligatorio y rechazo fail-closed ante inconsistencias (D-W79-06/07). El Knowledge publicado por Waby que un Workspace incorpore se materializa dentro de sus recursos Workspace; sincronización/actualización/retirada siguen pendientes (D-W79-11).
6. **Esquema `AgentExecution`** en AI Service DB con `_id=executionId`, `evAssignedId` obligatorio e inmutable, snapshot y hashes obligatorios, timestamps diferenciados `createdAt`/`startedAt`/`finishedAt`, estados lógicos de Fase 2 y métricas/costes condicionales. Deben definirse los índices cronológicos compuestos por Organización, Workspace, AgentInstance y conversación (P10, D.1.7, D.6 y D-W79-05).
7. **Campos, invariantes y atomicidad de asignación:** `Contacto.ref_agent_assigned` y `ref_ev_assigned` en CRM, junto con la proyección `contact_assignments.assignedType` (`human`/`agent`), `assigneeId` y la transición a `status='released'` con `evAssignedId=null` (contratos C12, D.1.6–D.1.9 y D-W79-04). Las escrituras de `Contacto`, `EvAssigned` y Outbox deben compartir una transacción MongoDB con CAS y rollback completo ante cualquier fallo (D-W79-15). Las tres colecciones están en la misma base MongoDB Atlas; antes de implementar debe verificarse la topología compatible con transacciones multidocumento y la propagación de una misma sesión en todas las operaciones. **Esta verificación es requerida para el flujo de asignación (ARQ-06 / D-W79-15) y cualquier operación que realmente requiera atomicidad multidocumento; no bloquea la persistencia base de `SkillConfiguration` y `AgentRules` en A2.**
8. **Validación cross-tenant** `AgentInstance.workspaceId → Workspace → Organización` documentada (contrato C3, D.3).
9. **Matriz de ownership y persistencia** revisada y aprobada (sección 5.1).
10. **Cuestiones pendientes de la sección 8** clasificadas como: decisión requerida, propuesta técnica o evidencia no verificada.
11. **Separación de credenciales D-W79-08**: `AgentDefinition` referencia `LLMRoute` sin almacenar secretos; la credencial empresarial se referencia desde `AgentInstanceIntegration`; `SkillConfiguration` no contiene `credentials`; los secretos no se incorporan a snapshots, prompts, logs ni ledger. El mecanismo físico de custodia y cifrado queda para diseño técnico posterior.
12. **Fuente del contenido de Skills propias**: la definición de la entidad/ubicación física del contenido fuente de Skills propias sigue pendiente y es necesaria antes de implementar su almacenamiento, resolución y ciclo de vida integral. **Sin embargo, esa definición NO bloquea**: el schema/modelo `SkillConfiguration`, el índice contractual ya definido, los repositorios de persistencia y consulta de configuraciones, los servicios de dominio limitados a esas operaciones, ni las validaciones de pertenencia y aislamiento. A2 no implementará almacenamiento, referencias ni resolución del contenido de Skills propias.
13. **Separación de catálogos y almacenamiento (D-W79-17):** catálogo/metadatos centrales Waby independientes de recursos Workspace; espacios físicos MinIO separados con políticas diferenciadas; recursos Waby fuera de cuota de clientes; cuota independiente por Workspace, sin bolsa organizacional; referenciar un recurso Waby no transfiere ownership ni implica copia física automática. Cualquier materialización debe ser explícita y respetar la regla específica de Knowledge D-W79-11.

## 11. Historial de cambios

| Versión | Fecha | Cambio | Autor |
|---------|-------|--------|-------|
| 1.0 | 08/11/2026 | Creación del documento consolidado B1/B2/B3 | Agente OpenCode |
| 1.1 | 08/11/2026 | Pasada correctiva: identidad/estado global de AgentDefinition, eliminación propuesta de versionado, consistencia de persistencia/índices, snapshot, ownership y limpieza de duplicados | ChatGPT (revisión) |
| 1.2 | 09/11/2026 | Reconciliación cruzada D-W79-01 con WAB-78 DEC-CM-01/C9 y P10/C.7; actualización de referencias pendientes, alcance WAB-81 y criterios de aceptación | ChatGPT (revisión) |
| 1.3 | 09/11/2026 | Ajuste de alcance y trazabilidad operativa tras la reconciliación cruzada con el documento canónico WAB-78 | ChatGPT (revisión) |
| 1.4 | 09/11/2026 | Contrato de identidad de transición: `Contacto.ref_ev_assigned`, `contact_assignments.evAssignedId` nullable según estado y `AgentExecution.evAssignedId` obligatorio e inmutable; alineación del envelope y validación ABA con Fase 2 | ChatGPT (revisión con aprobación del usuario) |
| 1.5 | 09/11/2026 | Separación de configuración técnica mediante `LLMRoute`, allowlist Waby, eliminación de overrides técnicos en AgentInstance, snapshot de ruta efectiva y cierre de la primera cuestión pendiente | ChatGPT (revisión con aprobación del usuario) |
| 1.6 | 09/11/2026 | Depuración de cuestiones pendientes: retiro del índice único de ConversationMemory ya definido en 5.2.9/6.1 y reconciliado con WAB-78 A.2 | ChatGPT (verificación documental) |
| 1.7 | 09/11/2026 | Aprobación D-W79-04: vocabulario objetivo `assignedType` (`human`/`agent`) para `contact_assignments` y semántica de liberación; el tratamiento de la implementación legacy se aclaró posteriormente como reemplazo sin migración | ChatGPT (revisión con aprobación del usuario) |
| 1.8 | 09/11/2026 | Aprobación D-W79-05: contrato de AgentExecution, timestamps de aceptación/inicio/terminación, métricas condicionales, separación de AgentExecutionInbox e índices cronológicos compuestos | ChatGPT (revisión con aprobación del usuario) |
| 1.9 | 09/11/2026 | Aprobación D-W79-06: correspondencia WorkspaceKnowledge/KnowledgeBase y KnowledgeDocument/Document, cardinalidades y separación de la validación de workspaceId entre documento y KnowledgeBase | ChatGPT (revisión con aprobación del usuario) |
| 1.10 | 09/11/2026 | Aprobación D-W79-07: integridad Document/KnowledgeBase/Workspace en escritura, ingesta, RabbitMQ, recuperación RAG, eliminación y reingesta; filtro vectorial tenant obligatorio y rechazo fail-closed | ChatGPT (revisión con aprobación del usuario) |
| 1.11 | 09/11/2026 | Aprobación D-W79-08: separación entre credenciales LLM Waby-owned y credenciales empresariales Workspace-owned; `AgentInstanceIntegration` como ubicación de referencia protegida; eliminación de `SkillConfiguration.credentials`; tecnología física de custodia diferida | ChatGPT (revisión con aprobación del usuario) |
| 1.12 | 09/11/2026 | Aprobación D-W79-09/10/11: identidad estructurada y unicidad de SkillConfiguration; Skills propios sin historial/rollback con contenido consistente por ejecución y hash; Knowledge Waby materializado en ámbito Workspace | ChatGPT (revisión con aprobación del usuario) |
| 1.13 | 10/11/2026 | Aprobación D-W79-15/16: atomicidad transaccional de Contacto + EvAssigned + Outbox con CAS y entrega asíncrona idempotente; unicidad transaccional de isDefault para todos los estados, sin reasignación automática | ChatGPT (revisión con aprobación del usuario) |
| 1.14 | 10/11/2026 | Aprobación D-W79-17: separación de catálogo central Waby y recursos Workspace, espacios MinIO diferenciados, cuota independiente por Workspace y recursos Waby fuera de cuotas de clientes; materialización física solo explícita | ChatGPT (revisión con aprobación del usuario) |
| 1.15 | 10/11/2026 | Aclaración aprobada: la implementación de IA Bot/BotInstance nunca llegó a producción; se reemplaza sin migración ni compatibilidad hacia atrás. Se ajusta WAB-84 a limpieza/retiro de legacy y se identifican patrones técnicos neutrales rescatables de la auditoría. | ChatGPT (revisión con aprobación del usuario) |
| 1.16 | 16/11/2026 | Correcciones documentales WAB-79/A2: (1) semántica precisa de `enabled` distinguiendo deshabilitación, baja funcional, eliminación física y limpieza; (2) alcance de repositorios A2: Create, Read, Update (sin Delete); (3) desacoplamiento de contenido de Skills propias de la persistencia de configuración; (4) ARQ-06/transacciones Atlas reclasificado como dependencia externa al flujo Contacto+EvAssigned+Outbox, no bloqueo general de A2. | Agente OpenCode |

---

## Alcance y restricciones operativas de esta actualización

- Esta actualización local incorpora D-W79-15 (atomicidad de transiciones de asignación), D-W79-16 (atomicidad y unicidad de isDefault) y D-W79-17 (catálogos, espacios MinIO y cuota independiente por Workspace) en WAB-79 y reconcilia sus contratos en Fase 2. El usuario confirmó que `Contacto`, `EvAssigned` y Outbox pertenecen a la misma base de datos MongoDB Atlas. Permanecen pendientes la verificación de la topología del clúster para transacciones multidocumento y la propagación de la misma sesión en todas las operaciones. **Esta verificación es requerida para el flujo de asignación (ARQ-06 / D-W79-15) y cualquier operación que realmente requiera atomicidad multidocumento; no bloquea la persistencia base de `SkillConfiguration` y `AgentRules` en A2.**
- D-W79-06 resolvió la cuestión original #1 sobre correspondencia conceptual/física y cardinalidades; D-W79-07 resolvió la cuestión original #6 sobre integridad de pertenencia y coherencia de `Document.workspaceId`/`knowledgeBaseId` con su KnowledgeBase.
- Se retiraron de la lista de cuestiones pendientes el índice compuesto de ConversationMemory, ya definido en el esquema e índices propuestos; el vocabulario de la proyección ContactAssignment, resuelto mediante D-W79-04; y los campos/índices de AgentExecution, resueltos mediante D-W79-05.
- No se modificó código, modelos, schemas, tests ni configuraciones.
- No se modificó Linear (issues, estados ni dependencias); WAB-79 conserva su estado Backlog.
- No se repitió la auditoría B1 ni se declararon índices aplicados en producción sin evidencia.
- Se mantuvo separada la regla de dominio de su mecanismo técnico de implementación.
- La ubicación física de la fuente de contenido de Skills propios y la política de sincronización/retirada de Knowledge materializado desde Waby permanecen explícitamente pendientes; no se inventaron colecciones, sincronización ni mecanismos técnicos. **La definición pendiente del contenido de Skills propias NO bloquea el schema/modelo `SkillConfiguration`, el índice contractual, los repositorios de persistencia y consulta, los servicios de dominio ni las validaciones de pertenencia y aislamiento en A2.**



## 16. Retiro coordinado de AgentInstance

> **Estado:** Contrato de diseño aprobado y reconciliado en las sesiones WAB-79 del 11 y 12/11/2026. No implica implementación. Este apartado concreta P9 y prevalece sobre formulaciones anteriores incompatibles con el retiro coordinado.

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
| `executionDrainConfirmation` | Confirmación inmutable del drenaje, ligada al `retirementOperationId` y `admissionBlockedAt` |
| `finalizationConfirmation` | Confirmación terminal durable de AI Service para la orden de finalización |
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

Cada transición valida mediante CAS la asignación actual a A y que `ref_ev_assigned` coincida con el evento previo esperado. Cada contacto se procesa en transacción MongoDB independiente con Contacto + EvAssigned + Outbox + actualización de `RetirementOperation.eventSetRevision` atómicos. Si otra transición válida ya movió o liberó el contacto, el retiro respeta el resultado, no duplica evento ni revierte la decisión.

La regla aprobada es como máximo un `EvAssigned` correlacionado con un retiro por combinación de operación y contacto. Se utiliza el campo heredado `Evento.contacto`, sin duplicarlo como `contactId`. El índice único parcial conceptual es `{retirementOperationId: 1, contacto: 1}`, aplicado sólo a documentos `EvAssigned` que tengan `retirementOperationId`; los eventos no vinculados a retiros quedan fuera. La transición externa válida que saca al contacto de A usa la misma correlación y clave, sin alterar su actor ni `operationType`.

### 16.9 Correlación, causalidad y confirmación durable

Toda transición válida que saque un contacto de A mientras el retiro de A esté activo queda correlacionada con ese `retirementOperationId`, aunque la inicie otro administrador/proceso. La correlación no altera `actorType`, `actorRef` ni `operationType`: reflejan al actor y operación reales. El conjunto esperado incluye tanto transiciones del worker como transiciones concurrentes externas que sacaron contactos de A.

CRM obtiene los eventos esperados de los EvAssigned correlacionados, no de un preconteo ni de contadores. `SENT` en Outbox no significa aplicado.

AI Service usa Inbox durable y cursor causal: aplica sólo cuando el cursor coincide con `previousEvAssignedId`; eventos futuros esperan al predecesor y eventos obsoletos no revierten la proyección. `lastProcessedEvAssignedId` conserva la última transición aplicada incluso tras liberar; la asignación activa queda nula al liberar.

En una transacción local de AI Service, al aplicar cada evento: actualiza `contact_assignments`, avanza `lastProcessedEvAssignedId`, marca Inbox aplicado y crea confirmación durable en su Outbox. La confirmación significa aplicación real en la proyección, no recepción ni publicación. AI Service reintenta hasta que CRM la persista.

CRM mantiene `retirement_confirmations` como colección independiente. Cada documento contiene `evAssignedId`, `retirementOperationId` y `confirmedAt`; índice único `{evAssignedId: 1}` e índice de consulta `{retirementOperationId: 1, evAssignedId: 1}`. La recepción valida que el evento pertenece al conjunto esperado y a la misma operación. Un duplicado de la misma operación es idempotente; una confirmación asociada a otra operación o a un evento no esperado se rechaza y trata como inconsistencia. La finalización compara la correspondencia individual y completa entre eventos esperados y confirmaciones; la igualdad de contadores no basta.

### 16.10 Cierre del conjunto esperado

Mientras la operación esté en `releasing`, toda transición que saque un contacto de A escribe el mismo documento RetirementOperation en la misma transacción que Contacto, EvAssigned y Outbox, incrementando `eventSetRevision`. Esto incluye transiciones concurrentes externas.

Para pasar a `finalizing`, una transacción valida estado `releasing`, lease/fencing vigente y cero contactos asignados a A; cierra el conjunto y registra `eventSetClosedAt`. Tanto el alta de eventos como el cierre escriben el mismo documento RetirementOperation. Una colisión debe abortar/reintentar y volver a validar. En `finalizing` el conjunto es inmutable; sólo pueden seguir llegando confirmaciones pendientes.

### 16.11 Condiciones de finalización

CRM sólo puede finalizar cuando verifica: (1) cero contactos actualmente asignados a A; (2) ninguna AgentExecution aceptada pendiente o en procesamiento y confirmación durable de drenaje; (3) todos los EvAssigned del conjunto cerrado aplicados y confirmados durablemente por AI Service; (4) ningún evento causal pendiente de la operación en Inbox; y (5) correspondencia exacta entre los eventos esperados y las confirmaciones persistidas por CRM. Se consulta el estado real, no contadores.

CRM es propietario de `AgentInstance` y de su lifecycle (`agent_instances` en CRM DB). Por tanto, AI Service nunca modifica directamente `AgentInstance.status`. Tras verificar las condiciones anteriores, CRM registra en su Outbox una orden durable de finalización identificada por `retirementOperationId`. AI Service valida que el bloqueo de admisión y el drenaje corresponden a esa operación y registra una confirmación terminal durable en su Outbox; la recepción de la orden y la creación de esa confirmación se procesan idempotentemente mediante su Inbox. CRM persiste la confirmación terminal en una transacción local que también cambia `AgentInstance.status` a `retired`, `RetirementOperation.status` a `completed` y establece `completedAt`.

Si AI Service no está disponible, no confirma el bloqueo/drenaje o no ha aplicado el último evento, la operación permanece en `finalizing` o `blocked`, conserva bloqueos y reintenta. No se marca `retired` por timeout, respuesta HTTP exitosa ni Outbox `SENT`. La confirmación de AI Service certifica que el plano de ejecución está detenido; CRM realiza el cambio autoritativo de lifecycle.

### 16.12 Idempotencia, resultados inciertos y drenaje de ejecuciones

La identidad de la operación es `retirementOperationId`; la identidad de la transición por contacto es `{retirementOperationId, contacto}`; la identidad de la confirmación de aplicación es `evAssignedId`. No se usan `workerId`, número de intento ni `leaseGeneration` como claves de idempotencia.

Antes de ejecutar una transición, el proceso consulta si ya existe el `EvAssigned` para la combinación operación/contacto. Si existe, verifica su correspondencia y la existencia del Outbox atómico, reutiliza el evento y no vuelve a mutar Contacto, crear Outbox ni incrementar `eventSetRevision`. Si no existe, revalida el estado actual mediante CAS y ejecuta la transacción completa. El índice único es la protección definitiva ante carreras; la consulta previa no la sustituye.

La transacción CRM por cada salida de contacto durante `releasing` incluye Contacto, EvAssigned, Outbox y actualización/incremento de `RetirementOperation.eventSetRevision`. Una falla revierte el conjunto completo. La transición y el cierre del conjunto escriben el mismo documento RetirementOperation. Si MongoDB devuelve `UnknownTransactionCommitResult` y la sesión sigue disponible, se reintenta el commit de esa misma transacción según el driver. Si el resultado sigue siendo incierto o la sesión ya no puede recuperarse, se consulta por la clave operación/contacto y se verifica el Outbox; si existe, se reutiliza; si no existe, se relee Contacto y sólo entonces se decide reintentar. La ausencia de respuesta nunca se interpreta por sí sola como aborto confirmado. Un CAS o índice único perdido obliga a releer y respetar la transición ganadora.

AI Service confirma el drenaje de ejecuciones de forma separada del bloqueo de admisión. Después de confirmar que `executionAdmission` continúa bloqueado por el mismo `retirementOperationId`, verifica que no existen `AgentExecution` de esa instancia en `pending` ni `processing` y registra una confirmación durable en su Outbox. CRM la persiste directamente en `RetirementOperation.executionDrainConfirmation`, con `retirementOperationId`, `confirmedAt` y `admissionBlockedAt`. Es inmutable; duplicados de la misma operación/bloqueo son idempotentes y confirmaciones de otra operación o bloqueo se rechazan. La aceptación de ejecuciones y el bloqueo ya están coordinados atómicamente, por lo que después del bloqueo confirmado no pueden admitirse nuevas ejecuciones.

### 16.13 Orden y confirmación de finalización

Una vez satisfechas todas las condiciones de finalización, CRM crea una orden durable en su Outbox identificada por `retirementOperationId`. AI Service la recibe mediante Inbox durable y procesa idempotentemente: valida que el bloqueo y `executionDrainConfirmation` correspondan a la operación; si la orden ya fue procesada, recupera/republica la confirmación existente. No cambia `AgentInstance.status`, porque esa entidad pertenece a CRM. AI Service registra en su Outbox una confirmación terminal durable dentro de la misma transacción local que marca la orden de Inbox como procesada.

CRM valida la confirmación terminal y, en una única transacción local, la persiste en `RetirementOperation.finalizationConfirmation`, cambia `AgentInstance.status` a `retired`, cambia `RetirementOperation.status` a `completed` y establece `completedAt`. La orden y su confirmación usan `retirementOperationId` como identidad idempotente. Una confirmación de otra operación, bloqueo o estado no se acepta. La indisponibilidad o pérdida de respuesta mantiene la operación en `finalizing`; los reintentos no duplican la transición.

### 16.14 Detalles pendientes de concreción técnica

Sin reabrir las reglas aprobadas, quedan por definir: schema final y ubicación/topología transaccional del control de admisión; nombres definitivos, payloads y contratos de transporte Inbox/Outbox; propagación de sesión y topología MongoDB para atomicidad entre Contacto, EvAssigned, Outbox y RetirementOperation; mecanismo físico de índice parcial/discriminador compatible con el schema Mongoose real; y política independiente de retención/purga física. La existencia, semántica e índices conceptuales de `retirement_confirmations` y las claves de idempotencia ya están decididas. No se autoriza implementación ni cambios en Linear.

---

**Fin del documento.**
