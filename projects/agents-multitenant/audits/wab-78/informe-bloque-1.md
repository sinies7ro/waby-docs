# WAB-78 — Informe del Bloque 1: Dominio y ownership (D1)

**Fecha:** 04/11/2026 · **Sesión:** individual 1 · **Estado del bloque:** Revisado
**Alcance ejecutado:** AgentDefinition/AgentInstance; ownership Organización → Workspace → AgentInstance; cardinalidades, referencias, integridad e índices; relaciones con Contacto, conversaciones, canales, ejecuciones, Knowledge, Skills y Rules; límites de responsabilidad CRM / ai-service; entidades a conservar/adaptar/separar/retirar.
**Restricciones respetadas:** sin cambios de código, schemas, modelos, datos ni tests; sin edición de documentos canónicos ni del registro de decisiones; sin modificaciones en Linear (WAB-78 In Progress, WAB-79 Backlog verificados y intactos); sin reabrir P1/P2/P4/P9/P10; sin solicitar decisiones de producto; sin repetir AUD-01..06; sin avanzar análisis de D2–D7 (solo referenciados). Este informe es artefacto de trabajo, no documento canónico.

---

## 1. Fuentes utilizadas (nivel primario salvo indicación)

| Fuente | Uso en este bloque |
|---|---|
| `crm/docs/plans/wab-75-fase-2-diseno.md` (canónico) | §1 (L13/L17), A.1/A.2 (L43-330), B (L330-445), C.2-C.6 (L451-617), D.1-D.5 (L694-868), H.1/H.2 (L1675-1705), J ADRs (L1807-1917), I.1/L1792, L1995 |
| `crm/docs/plans/wab-75-fase-1-auditoria.md` (Fase 1) | §2.1 entidades as-is, §2.2, §5 matriz de ownership, §8 brechas I1-I21 (L334-344). **Reutilizado, no repetido** (AUD-01..06 intactos) |
| `WAB-78-complemento-tecnico.md` | Parte D (ownership), Parte E (`contactAssignmentHistory`), cabecera (premisa ai-service) |
| `crm/docs/reports/AUDITORIA_PERMISOS_WABY.md` | §2.4 (`bot.configure`), H9 — referencias para D2 |
| Registro de decisiones 03/11/2026 (Linear `345541575496`) | §1 P1, §2 P9, §4.1 P2, §6 P3, §8.1/§8.2, §9 |
| `crm/docs/plans/wab-75-agentes-administrados-por-waby.md` | I3 (L24, L70-76) — evidencia de contradicción con P1 |
| Código CRM (verificado en sesión) | `src/auth/ResourceOwnership.ts`, `src/models/chat/Contacto.model.ts`, `src/models/workspace/Works.model.ts`, `src/models/events/Evento.model.ts` y `Outbox.model.ts`, `src/graphql/resolvers/AiBot.resolver.ts`, `src/services/AiBot.service.ts` |
| `crm/microservices/ai-service/` (local) | Existencia del repo verificada (observación de fuente para D3) |

---

## 2. Respuestas a las preguntas del Bloque 1

### 2.1 ¿Cada recurso tiene un propietario inequívoco?

**Propietario de negocio: sí para todas las entidades del modelo objetivo.** La propiedad por capa del canónico B (L420-427) es unívoca y coherente con las decisiones aprobadas:

| Recurso | Propietario | Evidencia |
|---|---|---|
| AgentDefinition | **Waby** (plataforma, global, no visible al cliente) | A.2 L45-46; B L422; D.1 L699 (gestión ai-service Waby-only, CRM solo lectura) |
| AgentInstance | **Workspace** | A.2 L72-73; B L423; D.1 L700 (CRM, Waby-only); registro §8.1 (persistencia en CRM) |
| OrganizationAICredit / CreditLedger | **Organización** | B L424; ADR-002 aprobado (L1828-1847) |
| WorkspaceKnowledge | **Workspace** (compartido, I11) | B L423; restricción #4 L434 |
| SkillConfiguration / AgentRules | **AgentInstance** (I12/I13) | B L425; restricción #5 L435; cardinalidades L167/L184 |
| ConversationMemory | **AgentInstance + conversación** (I14) | B L426; L201 (N:1:N) |
| AgentExecution / AgentExecutionLog | **AgentInstance + Workspace** (I20/I21) | B L427; A.2 L99+ |
| Contacto (asignación efectiva) | **CRM** (autoridad única) | Registro §8.1; D.1 L704; `Contacto.model.ts` (ref_works) |
| EvAssigned (historial) | **CRM exclusivo** (colección `eventos`) | D.1.6 L744-766; `Evento.model.ts` L15-16 (`collection: "eventos"`) |
| `contact_assignments` (ai-service) | Proyección operativa, **no autoridad** | D.1 L704; complemento §8.1 |
| Definiciones/instancias (superficie) | Waby administra; cliente usa | D.4 L850-860 |

**Dos reservas que impiden dar la respuesta por completa (ambas con hallazgo propio):**

1. **Ownership de persistencia no declarado** para casi todas las entidades: D.1 declara quién *gestiona* (API) pero no quién *persiste*, salvo AgentInstance (CRM, aprobado). → **ARQ-14 (bloquea)**.
2. **Integridad del "receptor predeterminado" sin regla:** `isDefault` es un booleano opcional sin unicidad ni mecanismo atómico, pese al acuerdo P2 §4.1. → **ARQ-13 (bloquea)**.

*Distinción de evidencia:* la ausencia de `AgentInstance`, `assignedType`, `ref_agent_assigned` o `AGENT_INSTANCE` **en código** (`grep` en `crm/src` sin matches; `ResourceOwnership.ts` L33-45 sin `AGENT_INSTANCE`; `Contacto.model.ts` L110-157 solo `ref_user_assigned`) es **ausencia de implementación esperada** antes de WAB-79/WAB-80, no ausencia de diseño ni deficiencia confirmada (plan §3).

### 2.2 ¿Puede demostrarse el ownership hasta Organización?

**Sí, con la decisión aprobada y un eslabón verificado en código:**

- Cadena aprobada: `AgentInstance → Workspace → Organización` (registro §8.1; D.1.5 L734: "Ownership: AgentInstance → Workspace → Organización").
- Eslabón Workspace → Organización verificado en código actual: `crm/src/models/workspace/Works.model.ts:63` → `ref_organizacion: Ref<Organizacion>`.
- Como **AgentInstance persiste en CRM** (aprobado), toda la cadena queda en una sola base de datos: `AgentInstance.workspaceId` → `Works.ref_organizacion` → `Organizacion`, con validación cross-tenant en escritura (restricción #3, L433).
- `OrganizationAICredit` tiene ownership directo a Organización (`organizationId`, B L424; ADR-002 L1841).
- `AgentDefinition` **no encadena a Organización**: es global de Waby (B L422) — coherente y esperado; su ownership es de plataforma, no tenant.

**Salvedades verificadas (no son deficiencias de diseño, son estado actual):**

- El modelo as-is (`Bot`/`BotInstance` en ai-service) tiene `workspaceId` **sin `orgId`** → join cross-DB hacia CRM (Fase 1 §2.1, brecha I2, L339). La cadena actual hasta Organización depende de resolver `workspaceId` contra CRM; WAB-79 debe cerrarlo al definir índices/validación (**ARQ-15**).
- Para el resto de entidades, demostrar la cadena exige completar el ownership de persistencia (**ARQ-14**).

### 2.3 ¿Las cardinalidades soportan varias instancias sin unicidades o cupos no aprobados?

**Sí en lo esencial; con cuatro excepciones registradas:**

- **Coherente con P1/I16:** sin unicidad `workspaceId`+`agentDefinitionId` (restricción #1, L431; C.4 L564-566) → múltiples instancias de una misma definición por Workspace, sin tope. AgentDefinition → 0..N AgentInstance (L70). Esto ya soporta el mundo sin cupos.
- **Coherente con P4:** OrganizationAICredit 1:1 por Organización (L226).
- **Coherente con I12/I13/I14:** SkillConfiguration/AgentRules 1:N por instancia (L167/L184); ConversationMemory N:1:N (L201).

**Excepciones:**

1. Cardinalidad de AgentInstance aún dice "Un Workspace puede tener **1 o 2** AgentInstance según su plan (I3, P1)" (L97) → obsoleta con P1 (**ARQ-11**).
2. Todo el bloque de concurrencia/creación/ciclo de vida sigue condicionado a cupo y gracia: C.3 paso 3 (L558), C.5 índice/contador de cupo (L570-571), C.6 "ocupa cupo/si hay cupo/gracia 5 días" (L574-616) → obsoleto con P1 (**ARQ-11**).
3. `isDefault` sin regla de cardinalidad/unicidad (**ARQ-13**, bloquea).
4. Enum de estados incompleto frente a P9 (**ARQ-12**) e índices sin definir, diferidos a WAB-79 de forma explícita (**ARQ-15**, no bloquea).

### 2.4 ¿Hay responsabilidades duplicadas entre servicios?

**No hay dos autoridades rivales para la misma entidad con decisión aprobada.** Donde ambos servicios aparecen en D.1, los roles son cooperativos y están definidos:

| Área | CRM | ai-service | Evaluación |
|---|---|---|---|
| Créditos (L706) | Ledger, autoridad del saldo, presupuesto | Reporta consumo, valida saldo, autoriza | Cooperación explícita (P4/ADR-002/006), no duplicación |
| Auditoría por ejecución (L707) | Log técnico | Campos I21 | Cooperación definida |
| Asignación de Contacto (L704) | Autoridad única | Proyección operativa `contact_assignments` | Frontera clara y aprobada |
| Eliminación de Workspace (L708) | delete-wrk extendido | Limpieza de ai_service | Coordinación definida |

**Tres puntos donde la línea divisoria no está cerrada** (registrados, no resueltos):

1. **API vs persistencia:** D.1 asigna "gestión de AgentDefinition" a ai-service y "Gestión de Knowledge" a CRM, sin declarar dónde persiste cada una; el CRUD de bots hoy entra por CRM como proxy HTTP hacia ai-service (`AiBot.resolver.ts` L18+ con permisos `wrk*`; `AiBot.service.ts` L1-50). Sin la columna de persistencia, "gestión" es ambigua → **ARQ-14**.
2. **Knowledge:** D.1 da la gestión a CRM (L701), pero la persistencia actual (`knowledge_bases`) está en ai-service (Fase 1 §2.1) y H.1 dice "conservar → WorkspaceKnowledge" (L1684) sin indicar el traslado → **ARQ-14**.
3. **Skills/Rules:** D.1 asigna la edición a CRM bajo `bot.configure` (L702, D.5 L862-868), permiso que P3/§6 prohíbe introducir hasta auditar y que no existe en código (AUDITORIA §2.4 L137; `ResourceOwnership.ts` L33-45) → evidencia adicional a **ARQ-03** (sin nuevo ID).

### 2.5 Entidades existentes: conservar / adaptar / separar / reemplazar / retirar

Base: A.1 (L27-41), H.1 (L1677-1690), H.2-H.8, complemento §8.1. Reutiliza AUD-01 y entregables de Fase 1.

| Acción | Elementos | Evidencia |
|---|---|---|
| **Separar** | `Bot` → `AgentDefinition` (Waby) + `AgentInstance` (Workspace); `enabledCapabilities` global → filtrar por instancia | H.1 L1679, L1683 |
| **Adaptar** | `BotInstance` → `AgentInstance`; `delete-wrk` → limpiar ai_service/Qdrant/MinIO; `Contacto` → modelo unificado `assignedType`/`ref_user_assigned`/`ref_agent_assigned` (campos opcionales primero, según orden de migración aprobado) | H.1 L1680, L1687; D.1.5; registro §8.1 |
| **Conservar** | `Execution` → `AgentExecution` + campos I20/I21; RAG filtro `workspaceId` → `WorkspaceKnowledge`; colección `eventos` (para discriminator `EvAssigned`) y Outbox (propagación post-commit); `contactAssignmentHistory` (esta fase); `contact_assignments` como proyección; `Prompt`/`PromptVersion` como base de AgentDefinition | H.1 L1684, L1686; D.1.6/D.1.8; complemento §8.1; A.1 L29-30; `Evento.model.ts`/`Outbox.model.ts` existentes |
| **Reemplazar** | `ModelConfiguration` per-bot → routing central Waby; credencial per-bot → credenciales centralizadas; `memory-manager`/`MemoryFact` → `ConversationMemory` (ventana 20) | H.1 L1681-1682, L1685; A.1 L31-32; H.6/H.7 |
| **Retirar** | `BotCredential`; cupos por plan, gracia de 5 días, índices/contadores de cupo e I3 (consecuencia documental de P1) | A.1 L32; registro §1 (**ARQ-11, ARQ-16**) |
| **Crear** | `AgentDefinition`, `AgentInstance`, `SkillConfiguration`, `AgentRules`, `ConversationMemory`, `AgentExecutionLog`, `OrganizationAICredit` + `CreditLedger`, `EvAssigned` (discriminator); Policy Engine y output filter (contratos en WAB-78, implementación WAB-81) | A.1 L34-41; H.1 L1688-1690; D.1.6; ARQ-10 |

---

## 3. Hallazgos nuevos del Bloque 1 (formato obligatorio, plan §8)

### ARQ-11 — Cupos por plan y gracia siguen en el canónico pese a P1 aprobada

| Campo | Contenido |
|---|---|
| **ID** | ARQ-11 |
| **Dominio** | D1 |
| **Estado** | Inconsistencia |
| **Evidencia** | `crm/docs/plans/wab-75-fase-2-diseno.md`: L13 ("cupo por plan"), L17 (P1 listada como pendiente), L97 (cardinalidad "1 o 2… (I3, P1)"), L432 (restricción de integridad #2), §C.2 L451-460 (tabla Básico 1 / Profesional 2), §C.2.1 L464-471 (gracia de 5 días), §C.3 L558 ("verifica cupo disponible"), §C.5 L570-571 (índice/contador de cupo), §C.6 L574-616 ("ocupa cupo", "si hay cupo", gracia, selección de instancias), L1792 (Fase 3: "Índices de cupo"). vs Registro decisiones 03/11/2026 §1 (P1 + "consecuencia documental: eliminar I3 y apartados de cupos/gracia; no mantener gracia de reducción") |
| **Hallazgo** | El documento canónico no integra la consecuencia documental de P1: mantiene reglas de cupo, gracia, índices de cupo y transiciones condicionadas a cupo, y el §1 incluso todavía lista P1 como decisión pendiente de resolver en la fase |
| **Impacto** | Alto |
| **Bloquea WAB-78** | No |
| **Justificación** | P1 está aprobada: no falta decisión de negocio ni contrato (el contrato es "sin cupo"), falta integración documental. Aprobar sin corregir deja material obsoleto que WAB-79/Fase 3 podría implementar (L1792) |
| **Destino** | WAB-78 (actualización del canónico, ya listada en registro §9.5) — efecto directo en WAB-79 |
| **Recomendación** | Actualizar/eliminar §C.2, C.2.1, C.3(3), C.5, dependencias de cupo en C.6, restricción #2, L97, L13, L17 y Fase 3 (L1792). **Criterio verificable:** búsqueda de "cupo/gracia/1 o 2 instancias" sin resultados vigentes en el canónico y coherencia textual con registro §1. No editar durante el diagnóstico |

### ARQ-12 — Enum de estados de AgentInstance incompleto frente a P9 (4 vs 5)

| Campo | Contenido |
|---|---|
| **ID** | ARQ-12 |
| **Dominio** | D1 |
| **Estado** | Inconsistencia |
| **Evidencia** | Canónico A.2 L82: `status` = `draft`/`active`/`suspended`/`retired` etiquetado "(P9)" — 4 estados. Canónico §C.6 L581-611: diagrama y reglas de transición con `retirement_pending` — 5 estados. Registro 03/11 §2 (P9): tabla de 5 estados aprobados, incluido `retirement_pending` |
| **Hallazgo** | La definición de campo de la tabla de entidades (fuente natural para WAB-79) omite `retirement_pending`, mientras el mismo documento y P9 aprobado lo incluyen |
| **Impacto** | Medio |
| **Bloquea WAB-78** | No |
| **Justificación** | La decisión está aprobada (P9 de 5 estados); es una corrección de integración, no una decisión abierta. El pendiente de reversibilidad de `retirement_pending` ya está registrado en el registro §2 y no se reabre aquí |
| **Destino** | WAB-78 (actualización del canónico) |
| **Recomendación** | Sincronizar el enum de A.2 con la tabla de estados de registro §2 y con C.6. **Criterio verificable:** enum de A.2 = 5 estados de P9, sin menciones a 4 estados |

### ARQ-13 — `isDefault` sin regla de unicidad ni mecanismo atómico (P2 pendiente)

| Campo | Contenido |
|---|---|
| **ID** | ARQ-13 |
| **Dominio** | D1 |
| **Estado** | Decisión pendiente (P2 §4.1) + pendiente técnico de integridad |
| **Evidencia** | Canónico A.2 L84: `isDefault` Boolean **No** (opcional), sin restricción asociada en B L429-441; §C.6 L614: única regla existente ("no puede modificarse durante exceso temporal") depende de la gracia de cupo obsoleta con P1; ADR-004 L1898-1911 (fallback a instancia por defecto, estado "Propuesto"). vs Registro 03/11 §4.1: acordado "al menos una AgentInstance predeterminada por Workspace, checkbox, primera instancia garantizada, integridad atómica"; **pendientes abiertos:** ¿más de una?, ¿cómo se selecciona con varias?, ¿cómo se reemplaza sin dejar al Workspace sin receptor? |
| **Hallazgo** | El requisito aprobado "siempre hay receptor predeterminado con integridad atómica" no tiene regla de cardinalidad/unicidad ni mecanismo en el modelo; solo existe un booleano opcional, y el único mecanismo escrito (C.6) era el congelado durante gracia de cupo, ahora obsoleto |
| **Impacto** | Alto |
| **Bloquea WAB-78** | **Sí** |
| **Justificación** | Sin decidir unicidad y reemplazo, WAB-79 no puede definir el índice/regla de integridad de `isDefault` ni WAB-80 el provisioning con garantía de receptor; ADR-004 (fallback) quedaría apoyado en una selección sin regla. Es una dependencia concreta para la fase siguiente: modelo e índices de AgentInstance |
| **Destino** | WAB-78 (resolución del pendiente P2 §4.1 — **registrada, no solicitada en esta sesión**) → detalle en WAB-79/WAB-80 |
| **Recomendación** | Mantener visible en la matriz para la resolución de P2 por el usuario; una vez decidida, añadir restricción de unicidad en B + operación atómica de designación/reemplazo en WAB-79. **Criterio verificable:** regla de unicidad explícita coherente con la opción aprobada, prueba de reemplazo sin ventana sin receptor y transición de creación de primera instancia garantizada |

### ARQ-14 — Ownership de persistencia no declarado en D.1 (API vs persistencia)

| Campo | Contenido |
|---|---|
| **ID** | ARQ-14 |
| **Dominio** | D1 |
| **Estado** | Pendiente técnico |
| **Evidencia** | Canónico D.1 L696-708 (matriz "Responsabilidades": AgentDefinition → ai-service; AgentInstance → CRM "detalle diferido WAB-79"; Knowledge → CRM; Skills/Rules → CRM con `bot.configure`; créditos/auditoría → ambos) sin columna de persistencia; B L420-427 (propiedad por capa, sin servicio); H.1 L1679-1690. vs Registro §8.1: único ownership de persistencia aprobado es AgentInstance en CRM; ADR-002 L1847 ("CreditLedger y OrganizationAICredit residen en el mismo deployment", sin nombrarlo). vs Fase 1 §2.1: persistencia actual en ai-service (`bots`, `bot_instances`, `executions`, `knowledge_bases`, con `workspaceId` sin `orgId`). D.1.5 L737 confirma que el modelo completo de AgentInstance va a WAB-79 |
| **Hallazgo** | La matriz de responsabilidades mezcla superficie de API con ownership de persistencia y no declara en qué servicio persiste cada entidad del modelo objetivo (AgentDefinition, WorkspaceKnowledge, SkillConfiguration, AgentRules, ConversationMemory, AgentExecution/Log); Knowledge aparece como responsabilidad CRM mientras su persistencia actual está en ai-service y H.1 solo dice "conservar" |
| **Impacto** | Alto |
| **Bloquea WAB-78** | **Sí** |
| **Justificación** | La pregunta de aprobación del plan exige habilitar WAB-79 "sin trasladar decisiones arquitectónicas críticas sin resolver"; la frontera de persistencia por entidad es exactamente una de ellas. Aprobar D.1 ambiguo trasladaría a WAB-79 una decisión crítica (podría definirse persistencia en dos servicios contradictoriamente) |
| **Destino** | WAB-78 (completar la frontera en D.1) → detalle de esquemas/índices en WAB-79 |
| **Recomendación** | Completar D.1 con declaración explícita de "servicio que persiste" por entidad, separando API de persistencia, apoyándose en lo aprobado (AgentInstance→CRM; créditos→mismo deployment) y en el estado as-is de Fase 1. **Criterio verificable:** cada entidad de A.2/B tiene propietario de negocio + servicio persistente declarados, sin filas donde "gestión" no distinga API de persistencia |

### ARQ-15 — Índices y unicidad de AgentInstance sin definir (diferido explícito a WAB-79)

| Campo | Contenido |
|---|---|
| **ID** | ARQ-15 |
| **Dominio** | D1 |
| **Estado** | Pendiente técnico (diferido explícitamente) |
| **Evidencia** | Canónico A.2 L72-97 (AgentInstance sin índices declarados); B L429-441 (para AgentInstance solo #2 — cupo, obsoleto — y #3 workspaceId denormalizado); §C.5 L568-572 (unicidad ligada a cupo y P2); L737 ("el modelo completo de AgentInstance —campos, índices, ciclo de vida— corresponde a WAB-79"); L1995 ("índices específicos de MongoDB… se resuelven en fases de implementación; no bloquean el diseño conceptual") |
| **Hallazgo** | Los índices y restricciones de unicidad de AgentInstance no están definidos salvo los ligados al cupo (obsoletos con P1); ownership/validación tenant (p. ej. papel de `orgId`) e unicidad operativa quedan sin especificar |
| **Impacto** | Medio |
| **Bloquea WAB-78** | No |
| **Justificación** | El canónico difiere explícitamente el detalle a Fase 3/WAB-79 y el plan prohíbe bloquear por ausencia de detalle diferido; la frontera y el ownership sí están definidos (con ARQ-14 como salvedad) |
| **Destino** | WAB-79 |
| **Recomendación** | En WAB-79: tabla de índices por entidad coherente con P1 (sin índices de cupo), P2 (unicidad de `isDefault` según ARQ-13), P9 y cadena `workspaceId → ref_organizacion`. **Criterio verificable:** índices listados por entidad sin restos de I3, con validación cross-tenant declarada |

### ARQ-16 — I3 y referencias de cupo en documentación histórica sin marcar como superseded

| Campo | Contenido |
|---|---|
| **ID** | ARQ-16 |
| **Dominio** | D1 |
| **Estado** | Inconsistencia |
| **Evidencia** | `crm/docs/plans/wab-75-agentes-administrados-por-waby.md` L24 (I3: "una o dos instancias por Workspace según plan"), L70-76 (cupo, pendiente/recomendación P1); Fase 1 §8 L340 (fila I3, con BR-AI-009 documentada en `docs/business/bot-routing.md:130`); canónico H.1 L1680 (fila `BotInstance` con invariantes "I2, I3, I16"). vs Registro 03/11 §1 (consecuencia documental: eliminar I3 y apartados de cupos) |
| **Hallazgo** | La invariante I3 y sus referencias en documentación de negocio/histórica siguen vigentes sin marcarse como superseded o eliminadas por P1 (complemento de ARQ-11, que cubre el canónico) |
| **Impacto** | Medio |
| **Bloquea WAB-78** | No |
| **Justificación** | Documentación histórica con decisión aprobada y unívoca; el riesgo es de reintroducción (lectores futuros o reglas de negocio reinstaurando el cupo), no de contrato sin definir |
| **Destino** | WAB-78 (actualización documental, registro §9.5) |
| **Recomendación** | Marcar I3, BR-AI-009 y la fila de H.1 como superseded/eliminados por P1, referenciando la decisión. **Criterio verificable:** ninguna regla vigente de "1 o 2 instancias" sin marcado. Referenciar ARQ-11 (no duplicar) |

### ARQ-17 — Estados de los ADRs desincronizados de las decisiones aprobadas

| Campo | Contenido |
|---|---|
| **ID** | ARQ-17 |
| **Dominio** | D1 |
| **Estado** | Inconsistencia |
| **Evidencia** | Canónico §J: ADR-001 "Propuesto" (L1811), ADR-003 "Propuesto" (L1884), ADR-004 "Propuesto" (L1900), ADR-005 "Propuesto" (L1917); ADR-002 y ADR-006 "Aprobado por el usuario el 03/11/2026" (L1828, L1857). vs Plan de revisión §2 línea base: separación AgentDefinition/AgentInstance (ADR-001) y P10 snapshot inmutable (ADR-005) figuran como decisiones aprobadas |
| **Hallazgo** | Cuatro ADRs siguen en estado "Propuesto" pese a que dos de sus decisiones de fondo (ADR-001, ADR-005) están en la línea base aprobada; ADR-004 depende de pendientes de P2 y ADR-003 no tiene aprobación registrada |
| **Impacto** | Medio |
| **Bloquea WAB-78** | No |
| **Justificación** | Los estados de ADR son metadatos del documento; las decisiones de fondo ya están aprobadas en el registro. No cambia frontera, ownership ni contrato |
| **Destino** | WAB-78 (sincronización de ADRs, registro §9.5) |
| **Recomendación** | Sincronizar ADR-001 y ADR-005 con sus decisiones aprobadas; mantener/revisar ADR-003 y ADR-004 según sus pendientes. **Criterio verificable:** cada ADR con estado coherente con el registro de decisiones |

### ARQ-18 — Inventario de entidades (A.2/B) omite `EvAssigned` y `contactAssignmentHistory`

| Campo | Contenido |
|---|---|
| **ID** | ARQ-18 |
| **Dominio** | D1 |
| **Estado** | Inconsistencia |
| **Evidencia** | Canónico A.2 (L43-330) y diagrama B (L330-416): **sin** `EvAssigned` (primer match de "EvAssigned" en el documento: L704, ya en D.1); `contactAssignmentHistory`: sin aparición en el canónico (grep sin matches). vs D.1.6 L744-766 (EvAssigned aprobado como discriminator de `Evento` en colección `eventos`) y D.1.7 (clave para ABA); Complemento §8.1 (`contactAssignmentHistory`: "conservar en esta fase"). Soporte en código: `crm/src/models/events/Evento.model.ts` L15-16 (`collection: "eventos"`), `Outbox.model.ts` existentes |
| **Hallazgo** | El inventario de dominio del canónico no incluye dos elementos que las decisiones aprobadas sí definen: `EvAssigned` (histócial funcional, clave de ABA) y `contactAssignmentHistory` (a conservar en esta fase) |
| **Impacto** | Medio |
| **Bloquea WAB-78** | No |
| **Justificación** | El ownership de ambos está aprobado (EvAssigned: CRM exclusivo; contactAssignmentHistory: conservar) y el modelo receptor (`eventos`) existe; la omisión es de inventario documental, no de contrato de ownership |
| **Destino** | WAB-78 (completar A.2/B) |
| **Recomendación** | Añadir `EvAssigned` (y nota de `contactAssignmentHistory` conservado) al inventario A.2 y al diagrama B, con referencia a D.1.6/D.1.8 y al complemento. **Criterio verificable:** A.2/B cubren todas las entidades referenciadas por decisiones aprobadas |

---

## 4. Actualización de hallazgos existentes (sin nuevo ID, sin renumerar)

### ARQ-03 (D2 — preliminar) — evidencia adicional de D1

- **Evidencia añadida:** el canónico **asume** `bot.configure` como parte del diseño vigente: D.1 L702 (Gestión de Skills/Rules "✅ CRM (permiso `bot.configure`)"), D.4 L859 (edición de Skills/Rules "⚠️ `bot.configure`"), D.5 L862-868 (tabla del permiso nuevo). El registro §6 (P3) establece lo contrario: **no introducir** `bot.configure` hasta concluir la auditoría de permisos. AUDITORIA_PERMISOS §2.4 (L137) confirma que no existe en código; `ResourceOwnership.ts` L33-45 no tiene tipo `AGENT_INSTANCE`.
- **Efecto:** la contradicción canónico↔registro se registra aquí como evidencia para la validación de ARQ-03 en D2; **clasificación original conservada** (Crítico, bloquea — preliminar, sin validar). No se crea un ID nuevo.

### ARQ-07 (D3 — preliminar) — observación de fuente de D1

- **Observación:** ai-service **sí está disponible localmente** (`crm/microservices/ai-service/`, con `src/` completo y `dist/`; Fase 1 §2.1 leyó sus entidades desde ahí), contradiciendo la premisa "repositorio no disponible" de ARQ-07, del complemento (cabecera) y del registro §8.2.
- **Efecto:** D3 **podrá** revisitar la verificación de cancelación con evidencia primaria. **Hasta entonces se conserva la clasificación original** (Alto, bloquea / No verificado). Detalle en `analisis-previo-d2-d4.md`.

### ARQ-04 (D2 — preliminar)

- Sin cambios en esta sesión (fuera de alcance de D1). Su validación (H1/H4/H5/H7 con las aclaraciones previas) corresponde a D2 — ver §6.

---

## 5. Contradicciones detectadas (registradas, no resueltas unilateralmente)

| # | Contradicción | Evidencia | Impacto | Recomendación (sin ejecutar) |
|---|---|---|---|---|
| 1 | Canónico mantiene cupos/gracia/I3 vs P1 aprobada | ARQ-11 | Alto | Actualizar canónico a P1 (WAB-78) |
| 2 | I3/BR-AI-009 vigentes sin marcado superseded | ARQ-16 | Medio | Marcar como superseded por P1 |
| 3 | Enum de 4 estados vs P9 de 5 | ARQ-12 | Medio | Sincronizar enum con P9 |
| 4 | `isDefault` opcional vs "al menos una, atómica" (P2 §4.1) | ARQ-13 | Alto (bloquea) | Resolver pendiente P2 y añadir integridad |
| 5 | D.1 API vs persistencia sin declarar; Knowledge CRM vs persistencia actual ai-service | ARQ-14 | Alto (bloquea) | Completar frontera de persistencia en D.1 |
| 6 | D.1/D.4/D.5 asumen `bot.configure` vs P3/§6 que lo prohíbe | ARQ-03 (evidencia añadida) | Crítico (preliminar) | Auditoría D2 → decisión P3 |
| 7 | ADRs "Propuesto" vs decisiones aprobadas | ARQ-17 | Medio | Sincronizar estados de ADRs |
| 8 | A.2/B sin `EvAssigned`/`contactAssignmentHistory` vs decisiones aprobadas | ARQ-18 | Medio | Completar inventario |
| 9 | Premisa "ai-service no disponible" (complemento L4, registro §8.2) vs repo local existente | §1 de fuentes; `analisis-previo-d2-d4.md` | Medio (para D3) | Revisitar ARQ-07 con evidencia del repo local |

Ninguna se resolvió aquí: la regla es registrar evidencia, impacto y recomendación (plan §3).

---

## 6. Lista de cuestiones y fuentes para la sesión D2 (Bloque 2 — Provisioning y permisos)

### 6.1 Fuentes a leer (en este orden)

1. **Plan de revisión** (Linear `fca2514ac4ee`): §4 Bloque 2, §6 (reutilización del análisis previo), §8 (formato), restricciones §10.
2. **`analisis-previo-d2-d4.md` §D.2** — material preliminar con ARQ-03 y ARQ-04 (conservar IDs y clasificaciones).
3. **Registro de decisiones 03/11**: §6 (P3 sin decidir; no introducir `bot.configure`), §1 (administración centralizada por Waby), §4.3/§4.4 (pendientes `wrk.teams.*`), §8.1.
4. **`crm/docs/reports/AUDITORIA_PERMISOS_WABY.md` completo**: §2.1 (flujo T0→T5), §2.2-§2.3 (componentes y modelo `org.*`/`wrk.*`), §2.4 (`bot.configure`), §3 (catálogo de roles/permisos y seeds), §4 (guards/middleware), H1/H4/H5/H7 (evaluación con aclaraciones previas; no asumir que todos bloquean).
5. **Código CRM**: `crm/src/auth/` (`ResourceOwnership.ts` L33-45 sin `AGENT_INSTANCE`; `permission.enum.ts`; guards/middleware), seeds `tipAccessOrg.load.ts` / `tipAccessWorks.roles.ts`, `crm/src/auth/__tests__/ResourceOwnership.botResource.test.ts`, `crm/src/graphql/resolvers/AiBot.resolver.ts` (permisos `wrk*` actuales).
6. **Canónico D.1/D.2/D.4/D.5** (L694-868) — matriz de operaciones y separación Waby/cliente.
7. **Documentación de negocio de permisos** (`docs/business/permisos-roles.md`, `org-user-hierarchy.md`, `wrk-user-hierarchy.md`) — solo si hace falta para H1/BR-MT-001.
8. **`matriz-acumulativa.md`** — estado de ARQ-03/04 y de los bloqueantes de D1.

### 6.2 Cuestiones a resolver en D2

1. Validar o ajustar **ARQ-03**: ¿qué contrato de permisos/ownership necesita WAB-79/WAB-80 y cuál es su estado real? (ausencia de código ≠ ausencia de decisión; el diseño de permisos existe: roles/permisos/scopes).
2. Validar **ARQ-04** (H1, H4, H5, H7) con las aclaraciones ya aprobadas; determinar cuáles siguen vigentes y cuáles diferibles.
3. Construir la **matriz de actores × operación** de AgentInstance (crear, consultar, configurar, activar, suspender, retirar, asignar) con recurso, ownership y permiso por celda, distinguiendo Waby / admin Organización / admin Workspace / miembros / runtime.
4. Decidir **como propuesta pendiente** (no como aprobada) si hace falta un tipo `AGENT_INSTANCE` en `ResourceOwnership` y cómo reutilizar el sistema actual sin crear un segundo sistema (P3/§6).
5. Resolver la **contradicción 6** de §5 (canónico asume `bot.configure` vs P3): registrar opciones, no decidir.
6. Verificar la separación Waby/cliente (D.4): qué impide hoy que el cliente administre definiciones/instancias en backend.
7. Revisar el **estado de ARQ-13/ARQ-14** desde la óptica de permisos/ownership (no duplicar hallazgos: referenciarlos).

### 6.3 Restricciones para D2 (heredadas)

No inventar `bot.configure`, `AGENT_INSTANCE` ni nuevos scopes sin proponerlos como decisión pendiente; no reabrir P1/P2/P4/P9/P10; no repetir AUD-01..06; conservar ARQ-03/ARQ-04; siguiente ID nuevo disponible: **ARQ-19**.

---

## 7. Cierre de la sesión

- **Qué se cambió:** solo se crearon los 4 archivos de `F:\ElemenSoft\docs\revision-wab-78\` (línea base, análisis previo, matriz acumulativa, este informe). Ningún archivo fuera de esa carpeta fue modificado.
- **Qué se verificó:** estados de control en Linear (WAB-78 In Progress, WAB-79 Backlog — intactos); evidencias de código CRM citadas por línea; existencia del repo ai-service local; contenido íntegro del plan de revisión y del registro de decisiones.
- **Scope:** D1 completo; D2-D4 solo referenciados (material preservado, sin analizar).
- **Bloqueantes D1 para la aprobación de WAB-78:** ARQ-13 y ARQ-14. Condiciones de actualización documental: ARQ-11, ARQ-12, ARQ-16, ARQ-17, ARQ-18. **No se emite veredicto** (solo al final de bloques + integración, plan §9).
- **Pendiente humano (no solicitado en sesión):** resolución de los pendientes P2 §4.1 y de los demás pendientes ya registrados en el registro de decisiones.

