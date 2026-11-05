# WAB-78 — Informe del Bloque 3: Routing y asignaciones (D3)

**Fecha:** 04/11/2026 · **Estado del bloque:** Revisado (diagnóstico completado) — presente documento es una reconciliación posterior con evidencia adicional verificada; no se emite veredicto global WAB-78.
**Alcance:** Recepción, EvAssigned, Outbox, ABA, concurrencia, cancelación e invalidación de ejecuciones, validaciones G3/G4.
**Modo:** solo lectura y diagnóstico (plan §10). No se modificó código, modelos, schemas, tests, configuración ni documentos canónicos. No se creó ni cerró issues en Linear.

---

## 1. Alcance y fuentes

### 1.1 Alcance del diagnóstico

Este bloque corresponde a **D3 — Routing y asignaciones** del plan de revisión WAB-78 (§4, Bloque 3). Las preguntas obligatorias del plan §4 son:

1. **¿Cómo se obtiene la identidad de EvAssigned vigente?** (ARQ-05)
2. **¿Existe una operación atómica que incluya Contacto, EvAssigned y Outbox?** (ARQ-06)
3. **¿Existe cancelación física de ejecuciones LLM e invalidación de resultados?** (ARQ-07)
4. **¿Cómo se propaga `evAssignedId` y cómo validan G3/G4 contra CRM?** (ARQ-08)
5. **¿`operationType` de EvAssigned está aprobado o diferido?** (ARQ-09)

Además, se evalúan los 15 escenarios del apartado 6 del encargo y se construye la arquitectura real del flujo de asignación y ejecución.

### 1.2 Fuentes utilizadas (nivel primario salvo indicación)

| Fuente | Uso en este bloque |
|--------|-------------------|
| Plan de revisión WAB-78 (Linear `fca2514ac4ee`) | §1 estados, §2 línea base, §3 clasificación, §4 preguntas Bloque 3, §6 reutilización, §7 matriz, §8 formato hallazgos, §10 restricciones |
| Registro de decisiones 03/11/2026 (Linear `345541575496` / `3599e1e4`) | §1 P1, §2 P9, §4.1 P2, §6 P3, §8.1 AgentInstance en CRM, §8.2 |
| Canónico `crm/docs/plans/wab-75-fase-2-diseno.md` | D.1 (L694-798), D.1.5 (L710-743), D.1.6 (L744-766), D.1.7 (L768-783), D.1.8 (L785-797), E.4 (L1090-1198, E.4.8 takeover, E.4.11 cancelación, E.4.12 G3/G4). **No fue recuperable desde GitHub `master`; se utilizó la copia local existente en el workspace (sin trackear en git) y se señaló la diferencia entre el contenido local y el publicado.** |
| `analisis-previo-d2-d4.md` | §D.3 con ARQ-05…ARQ-09 (preliminar, sin validar) |
| `informe-bloque-1.md`, `informe-bloque-2.md` | Evidencias previas de D1 y D2 (no repetidas) |
| `matriz-acumulativa.md` | Estado de ARQ-03…ARQ-21, convenciones de ID |
| `crm/src/models/chat/Contacto.model.ts` | Modelo de Contacto: `ref_user_assigned` (L157); sin `assignedType`, `ref_agent_assigned` ni `evAssignedId` |
| `crm/src/models/events/Evento.model.ts` | Modelo base `Evento`, colección `eventos` (L16), sin discriminator `EvAssigned` |
| `crm/src/models/events/Outbox.model.ts` | Modelo Outbox: `type`, `payload`, `status` (L1-31) |
| `crm/src/services/Contacto.service.ts` | `newMessageReceived` (L511), `updateContactoByID` (L2826), `asUserToContact` (L3018), `sincronizarAsignacionPreCrm`, `sincronizarLiberacionPostCrm` |
| `crm/src/infrastructure/ai-service/assignmentSync.service.ts` | `sincronizarAsignacionPreCrm`, `sincronizarLiberacionPostCrm`, `ejecutarOutboxAssignmentSync`, `OUTBOX_ASSIGNMENT_SYNC` (L1-386) |
| `crm/src/graphql/resolvers/AiContactAssignment.resolver.ts` | `assignContact`, `reassignContact`, `unassignContact` (L1-74) |
| `crm/src/graphql/resolvers/Contacto.resolver.ts` | `sendMessage` (autoasignación, L558-610), `assignUserToContact` (L857-870) |
| `crm/src/stream/Outbox.stream.ts` | Change stream Outbox (L1-39) |
| `crm/src/Rabbit/workers/outbox.worker.ts` | `handleOutboxEvent` (L12-54) |
| `crm/src/Workers/outboxRetry.worker.ts` | Poller de reintentos Outbox (L1-124) |
| `crm/src/Rabbit/consumers/executionCompleted.consumer.ts` | Consumer `execution.completed`, G3 (L1-472) |
| `crm/src/infrastructure/ai-service/aiVigencia.repo.ts` | `evaluarVigenciaBot`, validación contra `contact_assignments` en ai-service (L1-124) |
| `crm/src/Rabbit/queues.enum.ts` | Colas Rabbit: `AI_EXECUTION_COMPLETED`, `AI_EXECUTION_COMPLETED_VIGENCIA_RETRY` |
| `crm/microservices/ai-service/src/domain/entities/execution.ts` | Modelo `Execution`: estados PENDING/RUNNING/COMPLETED/FAILED/CANCELLED, `cancel()` (L1-331) |
| `crm/microservices/ai-service/src/domain/entities/contact-assignment.ts` | `ContactAssignment`: `AssigneeType.USER/BOT`, `AssignmentStatus.ACTIVE/RELEASED` (L1-133) |
| `crm/microservices/ai-service/src/application/services/execution.service.ts` | `IExecutionService`, `createExecution`, `findById`, `findByConversationId` |
| `crm/microservices/ai-service/src/application/services/vigencia.service.ts` | `VigenciaService`: G1/G2 gate, `human_takeover`, `assignment_changed` |
| `crm/microservices/ai-service/src/application/services/process-message.service.ts` | `ProcessMessageService`, `executeWithContext` |
| `crm/microservices/ai-service/src/engine/execution-engine.ts` | `ExecutionEngine`: create execution, markRunning, pipeline loop, G2, publishResult (L1-235) |
| `crm/microservices/ai-service/src/engine/pipeline.ts` | `Pipeline`: cancelación entre pasos, ejecución secuencial |
| `crm/microservices/ai-service/src/engine/steps/` | LoadConfiguration, InvokeProvider, ExecuteRAG, etc. |
| `crm/microservices/ai-service/src/infrastructure/messaging/consumers/message-received-handler.ts` | Consumer Rabbit `ai-service.message-received`, G1 (L1-251) |
| `crm/microservices/ai-service/src/infrastructure/messaging/publishers/rabbitmq-event-publisher.ts` | Publicador Rabbit, never-throws (L1-88) |
| `crm/microservices/ai-service/src/infrastructure/crm/client/crm-graphql-client.ts` | Cliente ai-service→CRM: `Authorization: Bearer <CRM_API_KEY>`, sin `x-org-id`/`x-wrk-id` (L112-149) |
| `crm/microservices/ai-service/src/infrastructure/composition-root.ts` | Inyección de `vigenciaService`, engine, handler (L480-539) |
| `crm/microservices/Difusion/src/Worker/__tests__/vigencia-g4.test.ts` | G4 en worker interactivo (evidencia de cobertura) |
| `crm/microservices/ai-service/tests/unit/application/services/vigencia.service.test.ts` | Tests vigencia G1/G2 |
| `crm/microservices/ai-service/tests/unit/engine/execution-engine.test.ts` | Tests engine (G2) |
| `crm/src/services/__tests__/sendMessage.assign.test.ts` | Tests autoasignación y Outbox (BR-AI-015) |
| `crm/src/infrastructure/ai-service/__tests__/assignmentSync.test.ts` | Tests `sincronizarAsignacionPreCrm`, `ejecutarOutboxAssignmentSync` |
| `crm/src/Workers/__tests__/outboxRetry.worker.test.ts` | Tests poller Outbox |

### 1.3 Limitación de acceso a Linear

El plan de revisión y el registro de decisiones se citan como fuentes primarias (IDs `fca2514ac4ee` y `345541575496` / `3599e1e4`), pero **no se pudo consultar su contenido directo en Linear durante esta sesión** (no hay herramienta de lectura de Linear disponible en el entorno de ejecución). Se utiliza la documentación local como fuente de continuidad (per AGENTS.md raíz: "Si no tienes acceso a Linear, indícalo expresamente").

---

## 2. Conclusión ejecutiva

El diseño de routing y asignaciones de WAB-78 (D3) tiene **una autoridad de estado conceptual clara (CRM vía Contacto + EvAssigned + Outbox)** pero presenta **brechas técnicas significativas** que impiden considerar el bloque listo para aprobación sin decisiones pendientes y sin verificación de producción.

### 2.1 Fortalezas verificadas

- CRM es la autoridad única de asignación efectiva: `Contacto.ref_user_assigned` (humano) y el canónico prevé `ref_agent_assigned` (AgentInstance) y `assignedType` (D.1.5). `contact_assignments` en ai-service es explícitamente una proyección operativa, no autoridad (D.1, tabla D.1 fila "Asignación efectiva de Contacto").
- Existe un mecanismo Outbox (`contact.assignment.sync`) con change stream + poller de reintentos, idempotente y con guarda de supersede (archivos `assignmentSync.service.ts`, `Outbox.stream.ts`, `outboxRetry.worker.ts`).
- Existen gates de vigencia G1 (entrada ai-service), G2 (pre-completar engine), G3 (CRM consumer `execution.completed`) y G4 (worker interactivo Difusion), con vocabulario único (`human_takeover`, `assignment_changed`, `bot_deactivated`, `workspace_mismatch`). G1 es fail-open por diseño ante indisponibilidad de Mongo de ai-service; G2, G3 y G4 fallan cerrados ante indisponibilidad de la base consultada.
- Tests de regresión cubren escenarios clave: autoasignación, fallo HTTP → Outbox PENDING, self-heal vía Outbox, vigencia humana bloquea bot, G3/G4 descarte decisivo.

### 2.2 Brechas críticas (bloqueantes para aprobación)

| ID | Brecha | Severidad |
|----|--------|-----------|
| ARQ-05 | Ausencia de implementación verificada de identidad de `EvAssigned` vigente: `Contacto.model.ts` no tiene `assignedType`/`ref_agent_assigned`/`evAssignedId`; `Evento.model.ts` no tiene discriminator `EvAssigned`; no hay referencia directa al evento de asignación vigente desde Contacto. Ninguna de las alternativas (consulta del último evento, referencia directa en Contacto) está implementada. `ref_ev_assigned` es un nombre propuesto, no un campo aprobado; la elección entre alternativas es una decisión técnica de diseño, no de producto. | Alto / Ausencia verificada — Bloquea |
| ARQ-06 | No existe operación atómica Contacto + EvAssigned + Outbox. El mecanismo actual de `assignmentSync.service.ts` es una **recuperación compensatoria**, no un Transactional Outbox que registre cada transición: no hay sesión de transacción MongoDB compartida entre el write de Contacto y la creación de Outbox (`OutboxModel.create` fuera de transacción). La compatibilidad de transacciones en Atlas no está verificada (destino D6). La atomicidad lógica del diseño aprobado no está implementada. | Crítico / Ausencia verificada — Bloquea |
| ARQ-07 | Existe cancelación lógica (`Execution.cancel()` marca CANCELLED) pero no hay cancelación física de llamadas LLM (no hay `AbortController` ni abort signal en `IAIProvider.invoke` ni en `InvokeProviderStep`). La ausencia de aborto físico por sí sola no es incumplimiento fatal si el proveedor no lo permite; la obligación crítica es invalidar el resultado y rechazar efectos tardíos. Nueva brecha: `runPipelineLoop` no revalida `context.cancelled` ni la vigencia antes de ejecutar capabilities tras `pipeline.execute(context)`. `publishResult` publica `execution.completed` semánticamente mal etiquetado aunque la ejecución esté cancelada. | Alto / Ausencia parcial verificada — Bloquea |
| ARQ-08 | `evAssignedId` **no existe** como campo: no se propaga, no se persiste en contexto de ejecución, no se valida contra CRM. G1/G2/G3/G4 consultan `contact_assignments` en `AI_SERVICE_MONGO_DB` (proyección de ai-service), no la autoridad CRM directamente; la proyección eventual no es alternativa aceptable a la verificación contra CRM en G3/G4. No se acepta "validar contra CRM o contra `contact_assignments` convergente con CRM" como equivalente: CRM es la autoridad única y `contact_assignments` es solo una proyección operativa. G1 es fail-open por diseño; G2/G3/G4 fallan cerrados ante indisponibilidad de la base consultada pero siguen consultando la proyección. G3 y G4 permiten continuar si falta `botId` en mensajes legacy (bypass sin estrategia de compatibilidad/drenaje/rechazo). `AiContactAssignment.resolver.ts` (`assignContact`, `reassignContact`, `unassignContact`) llama directamente a ai-service vía `ContactAssignmentClient`; no actualiza `Contacto` CRM ni crea `EvAssigned`. | Medio / Ausencia verificada — Bloquea |
| ARQ-09 | `operationType` no está en `Evento.model.ts` ni aprobado: sigue siendo "propuesta pendiente" (D.1.6 L766). No se confunde con `AssignedType`. No se introduce en D3; diferido a WAB-79. | Medio / Propuesta no aprobada — No bloquea |
| ARQ-22 | Entrega durable e idempotencia del flujo asíncrono de ejecución no garantizadas: el publisher absorbe fallos sin publish confirms ni `drain`; el engine espera la publicación y el publisher puede absorber el fallo completando normalmente; el handler confirma (`ack`) el mensaje de entrada tras ejecución normal; no hay dedup persistente por `originMessageId`; una redelivery puede crear otra ejecución con `executionId` diferente; la deduplicación G3 por `executionId` no resuelve ese caso. | Alto / bloqueante — Brecha de confiabilidad verificada en código; impacto productivo no reproducido |

### 2.3 Veredicto parcial D3

**No se emite GO/NO-GO global** (corresponde al cierre de todos los bloques + integración). El bloque D3 tiene su diagnóstico completado, pero **no está listo para aprobación** tal como está: ARQ-05, ARQ-06, ARQ-07, ARQ-08 y ARQ-22 son bloqueantes con dependencias concretas (definición de modelo `EvAssigned` y campo de referencia en Contacto, implementación de transacción Contacto+EvAssigned+Outbox y verificación en Atlas, contrato de cancelación e invalidación de efectos tardíos, `evAssignedId` y validación contra autoridad CRM en G3/G4, y entrega durable e idempotencia del flujo asíncrono). Los escenarios de takeover humano y cancelación tardía no están garantizados por la implementación actual.

---

## 3. Respuesta a las preguntas obligatorias del plan §4, Bloque 3

### 3.1 Pregunta 1 — Identidad de EvAssigned vigente (ARQ-05)

**Estado: Ausencia de implementación verificada.**

**¿Cómo se obtiene la identidad de EvAssigned vigente?**

El código actual **no implementa ninguna de las alternativas**:

| Alternativa | ¿Está en el código? | Evidencia |
|-------------|---------------------|-----------|
| Consultar el último evento de asignación | No | `Evento.model.ts` (L1-61) no tiene discriminator `__t` para `EvAssigned`; no hay campo `operationType` ni `assignedType` en el modelo base |
| Referencia directa al evento de asignación vigente desde Contacto | No | `Contacto.model.ts` (L1-330) tiene `ref_user_assigned` (L157) pero **no** `assignedType`, `ref_agent_assigned`, `evAssignedId` ni referencia a `Evento._id` de asignación |
| `contactAssignmentHistory` | No existe en código | No aparece en `crm/src` (grep sin matches); el canónico A.2/B lo omite (ARQ-18) |

**Evaluación:**

- **Consistencia entre Contacto y Evento:** No existe. Contacto solo tiene `ref_user_assigned` (humano). No hay forma de saber qué transición `EvAssigned` autorizó una ejecución.
- **Atomicidad de la actualización:** No aplica (no hay escritura de EvAssigned).
- **Carreras entre transiciones concurrentes:** Sin identidad de transición no hay forma de resolver carreras.
- **Identificación inequívoca de la transición que autorizó una ejecución:** Imposible con el modelo actual. El contrato aprobado exige una identidad única para cada transición confirmada de asignación; esa identidad permite saber qué transición autorizó una ejecución y resolver ABA.
- **Resistencia frente a ABA:** No hay mecanismo. Una secuencia A → B → A no es detectable sin identidad de transición.
- **Coste y fiabilidad de consultar el historial:** No hay historial de asignaciones persistido como `EvAssigned`.
- **`originMessageId` vs. identidad de transición:** `originMessageId` sirve para trazabilidad/idempotencia del mensaje de origen y **no sustituye** la identidad de transición `EvAssigned`.

**Alternativas (decisión técnica de diseño, no de producto):**

1. **Referencia directa en Contacto:** agregar un campo (nombre propuesto `ref_ev_assigned`, **no aprobado**) que apunte al `Evento._id` de la transición vigente. Requeriría escritura atómica con Contacto y Outbox.
2. **Consulta del último evento de asignación:** requeriría indexar `eventos` por `contacto` + `__t` = `EvAssigned` + `createdAt -1`. Existe índice `{contacto: 1, __t: 1, createdAt: -1}` (L22) pero no hay documentos `EvAssigned` que consultar.

El informe puede comparar ambas alternativas y recomendar una, siempre que justifique atomicidad, concurrencia, ABA, índices y coste de lectura. La elección entre ambas es una **decisión técnica de diseño**, no una decisión de producto que deba solicitarse al usuario. `ref_ev_assigned` es un nombre propuesto, no un campo aprobado.

**Conclusión:** Ninguna alternativa está implementada. La ausencia de implementación está verificada: no hay discriminator `EvAssigned`, no hay referencia directa en Contacto, no hay `evAssignedId`. No hay forma de obtener la identidad de EvAssigned vigente. Queda pendiente la definición del modelo `EvAssigned` y el campo de referencia en Contacto (decisión técnica de diseño). No introducir `assignmentVersion`.


### 3.2 Pregunta 2 — Atomicidad Contacto + EvAssigned + Outbox (ARQ-06)

**Estado: Ausencia de transacción verificada (recuperación compensatoria, no Transactional Outbox).**

**¿Existe una operación atómica que incluya Contacto, EvAssigned y Outbox?**

No. El código verificado muestra escrituras **desacopladas** y un mecanismo de **recuperación compensatoria**, no un Transactional Outbox que registre cada transición:

1. `updateContactoByID` (Contacto.service.ts L2826) escribe `ref_user_assigned` en CRM vía `findByIdAndUpdate` (L2882).
2. **Camino ASIGNAR (`sincronizarAsignacionPreCrm`):** llama HTTP a ai-service **antes** del write de CRM. Si el HTTP tuvo éxito, CRM se escribe después. Si falla HTTP, se crea un Outbox de compensación y se aborta la escritura de CRM.
3. **Camino LIBERAR (`sincronizarLiberacionPostCrm`):** CRM se escribe primero; luego se llama HTTP a ai-service. Si falla HTTP, se crea un Outbox de compensación.
4. `ejecutarOutboxAssignmentSync` (assignmentSync.service.ts L310) hace un `updateOne` con CAS sobre `ref_user_assigned` (L375-378) — operación separada del write original.

**¿Los documentos pertenecen a la misma base de datos/sesión de transacción?**

- Contacto y Outbox están en la misma base de datos CRM (colecciones `contactos` y `eventos`/`outbox`).
- **No hay uso de `mongoose.Session` ni `session.startTransaction()`** en ninguno de los archivos revisados. No hay código que comparta una sesión de transacción entre el write de Contacto y la creación de Outbox.
- El Outbox se crea en `encolarOutboxPendiente` (L155-199) como `OutboxModel.create(...)` **fuera de cualquier transacción**.
- **El camino normal no genera necesariamente un documento Outbox:** solo se crea Outbox cuando falla el HTTP (compensación).

**Ausencia de transacción en el código: verificada.** No hay sesión de transacción compartida. Cada operación es independiente.

**¿Qué ocurre ante errores parciales, rollback y reintentos?**

- En el camino de asignar: si el HTTP falla, se encaja Outbox y se relanza; el CRM no se escribe (fail-closed para la asignación). El Outbox luego converge vía `ejecutarOutboxAssignmentSync` con guarda de supersede (L343-349).
- En el camino de liberar: si falla HTTP, CRM ya escribió y se crea Outbox de compensación; no se relanza.
- No hay rollback atómico porque no hay transacción. Cada operación es independiente.

**¿Hay idempotencia frente a mensajes duplicados y publicaciones repetidas?**

- Sí, parcial: `convergerHttp` (L89-143) es idempotente (no-op si ya converge; 409 → reassign; 404 en unassign → éxito). `ejecutarOutboxAssignmentSync` tiene guarda de supersede (L343-349) y CAS (L375-384).
- Pero la **publicación en Outbox** (`OutboxModel.create`) no es condicional a una transacción previa; si el proceso cae entre el write de CRM y el create de Outbox, queda divergencia sin recuperación automática hasta que el poller la detecte.

**¿Qué garantías aporta Outbox ante fallos del proceso publicador?**

- El change stream (`Outbox.stream.ts`) es el primer intento rápido.
- El poller (`outboxRetry.worker.ts`) reintenta con backoff exponencial (`min(600s, 10s·2^attempts)`, L47-50) hasta `OUTBOX_MAX_ATTEMPTS` (50 por omisión, L44).
- Agotado → `FAILED` + métrica + intervención manual (L72-83). No hay reconciliación automática después del agotamiento.

**Respecto de MongoDB Atlas:**

- No hay código que use sesiones de transacción. No hay evidencia de que el deployment real de Atlas soporte transacciones en las colecciones involucradas.
- **No declaramos compatibilidad con Atlas sin evidencia del deployment real.** Queda como **No verificado** y se coordina con D6.

**Conclusión:** No existe operación atómica Contacto + EvAssigned + Outbox. No hay EvAssigned en el modelo. No hay sesión de transacción compartida. El mecanismo actual es recuperación compensatoria, no un Transactional Outbox que registre cada transición, y no proporciona la misma garantía que el diseño transaccional aprobado. La atomicidad lógica del diseño (D.1.8) no está implementada.

**Ventana de fallo (ARQ-06):** ai-service recibe y aplica correctamente una asignación; la llamada remota termina correctamente; posteriormente falla la persistencia de la asignación en CRM. No existe garantía de que se genere un Outbox compensatorio, porque el mecanismo actual lo utiliza principalmente ante errores de la llamada HTTP. Esto puede dejar la proyección de ai-service adelantada respecto de la autoridad CRM.

La solución técnica objetivo queda pendiente de implementación y aceptación.

### 3.3 Pregunta 3 — Cancelación e invalidación de ejecuciones en ai-service (ARQ-07)

**Estado: Ausencia parcial verificada (cancelación lógica sí; cancelación física de LLM no; invalidación de efectos tardíos incompleta).**

**Cancelación física — clasificación ajustada conforme a decisiones aprobadas:**

- No existe `AbortSignal` en `IAIProvider.invoke` ni en `InvokeProviderStep` (ai-service `src/engine/steps/invoke-provider.step.ts` L81: `provider.invoke(request)` sin signal de aborto). No existe evidencia de aborto físico de llamadas LLM en curso.
- El diseño aprobado exige intentar cancelar físicamente cuando sea posible, pero si el proveedor no lo permite, la obligación crítica es **invalidar el resultado y rechazar todo efecto tardío**.
- Por tanto, la ausencia de aborto físico por sí sola **no debe considerarse incumplimiento fatal ni requisito absoluto**. La garantía obligatoria es la invalidación y el rechazo de efectos tardíos, incluida la ejecución de capabilities.

**Nueva brecha en `ExecutionEngine.runPipelineLoop` (ai-service `src/engine/execution-engine.ts` L166-200):**

1. Espera a `pipeline.execute(context)`.
2. Lee `context.providerResponse`.
3. Si hay `toolCalls`, invoca `capabilityExecutor.execute(response.toolCalls, context)` (L194).
4. **No vuelve a comprobar `context.cancelled` ni revalida vigencia antes de ejecutar las capabilities.**

No se encontró mecanismo CRM → ai-service que solicite `Execution.cancel()` de inmediato ante un takeover o una reasignación (detallado abajo); en el flujo inspeccionado `cancel()` aparece solo en G2 o en el catch del pipeline. Por tanto, el riesgo demostrado no es una cancelación durante la invocación, sino una carrera por cambio de asignación: la asignación puede cambiar en CRM mientras el proveedor procesa y, al regresar la respuesta, `runPipelineLoop` puede ejecutar capabilities antes de que G2 vuelva a comprobar la vigencia. Además, `runPipelineLoop` no comprueba `context.cancelled` antes de ejecutar capabilities: deficiencia local independiente, sin afirmar que el takeover actual invoque esa ruta.

**Inspección de la superficie de cancelación:**

- **Mecanismo externo que solicite `Execution.cancel()` en takeover humano:** no se encontró evidencia de un comando/evento de cancelación desde CRM hacia ai-service que invoque `execution.cancel()` al producirse un takeover.
- **`ExecutionService` expone cancelación:** su interfaz actual solo contempla `createExecution`, `findById`, `findByConversationId` (ai-service `src/application/services/execution.service.ts`). No expone cancelación.
- **Cuándo se ejecuta `execution.cancel()` en el flujo actual:** solo en G2 (`execution-engine.ts` L87-88), cuando la vigencia falla antes de `markCompleted`, y en el catch del pipeline si `context.cancelled` y la ejecución sigue RUNNING (L104-105).

**`publishResult` (execution-engine.ts L202-234):**

- El engine publica `EventTypes.EXECUTION_COMPLETED` también para una ejecución cancelada.
- El payload contiene `status: cancelled`, y el consumer G3 actual descarta cualquier payload cuyo estado no sea `completed` antes de persistir o enviar.
- Por tanto, la publicación está **semánticamente mal etiquetada** y transporta eventos cancelados.
- No se afirma que ese evento por sí mismo produzca un envío externo en el flujo G3 actualmente inspeccionado (G3 lo descarta si el estado no es `completed`).
- No se asume la existencia de otros consumidores que produzcan efectos sin evidencia.

**Modelo y ciclo de vida de Execution:**

- `Execution` (ai-service `domain/entities/execution.ts` L1-331): estados `PENDING`, `RUNNING`, `COMPLETED`, `FAILED`, `CANCELLED`.
- `markRunning()` (L164): solo desde PENDING.
- `markCompleted(result)` (L182): solo desde RUNNING.
- `markFailed(error)` (L206): solo desde RUNNING.
- `cancel()` (L230): si es terminal, retorna (no-op). De lo contrario, marca CANCELLED. Es **cancelación lógica** (marca de estado), no aborto físico.

**¿Qué ocurre si una ejecución termina después de perder su asignación?**

- G2 (execution-engine.ts L80-91): revalida vigencia antes de `markCompleted`. Si `!vigencia.valido`, llama `context.cancel()` y `execution.cancel()`, luego si `execution.isRunning()` marca completado — pero tras `cancel()` el estado es CANCELLED, no RUNNING, así que **no marca completado**. El resultado no se publica como completed. Sin embargo, `publishResult` **siempre publica `execution.completed`** (EventTypes.EXECUTION_COMPLETED) incluso si la ejecución fue cancelada por vigencia.
- G3 (CRM `executionCompleted.consumer.ts` L218-249): valida vigencia antes de persistir `EvMensaje`. Si `!vigencia.valido`, descarta (ack por S4). Esto protege el efecto externo en el camino CRM→WhatsApp.
- G4 (worker interactivo Difusion, `vigencia-g4.test.ts`): valida antes del POST al proveedor.

**¿Es posible impedir que una ejecución obsoleta publique o entregue resultados?**

- Parcial: G2 cancela la ejecución en el engine si la vigencia falla antes de completar. Pero:
  1. `publishResult` publica `execution.completed` sin importar el estado cancelado (siempre publica el envelope).
  2. No hay cancelación física del LLM: si el proveedor ya respondió, el resultado existe en memoria de la ejecución aunque esté cancelada.
  3. Si el proceso de ai-service se reinicia, las ejecuciones en memoria se pierden; solo persiste lo que el repo guarde.

**Idempotencia de cancelaciones y finalizaciones:**

- `cancel()` es idempotente (retorna si terminal, L231-233).
- `markCompleted`/`markFailed` lanzan error si no es RUNNING (no son idempotentes: segunda llamada falla).

**Recuperación tras reinicios del proceso:**

- No hay evidencia de persistencia de estado de cancelación tras reinicio que impida republicar. `publishResult` no es idempotente frente al evento Rabbit (el consumer de CRM tiene dedup por executionId, L280-303, pero eso es del lado CRM).

**Diferenciación obligatoria:**

1. **Solicitud de cancelación:** `context.cancel()` / `execution.cancel()` existen (lógica).
2. **Cancelación física del trabajo en curso:** NO existe (no hay AbortController ni abort signal al provider).
3. **Invalidación del derecho a producir efectos:** G2/G3/G4 invalidan (no despachan si vigencia falla), pero `publishResult` aún publica el evento.
4. **Descarte de resultados obsoletos:** G3 descarta en CRM; G4 descarta en worker; pero no hay mecanismo en ai-service para impedir que un resultado completado se use internamente.
5. **Persistencia e historial de ejecución:** `Execution` se persiste vía `IExecutionRepository`; `ExecutionEvent` registra transiciones.

**Conclusión:** La cancelación lógica existe; la cancelación física de LLM no existe (y no se exige como absoluto si el proveedor no lo permite). La invalidación de efectos externos está incompleta porque `runPipelineLoop` no revalida `context.cancelled` ni la vigencia antes de ejecutar capabilities tras `pipeline.execute(context)`, y `publishResult` publica `execution.completed` indistintamente incluso cuando la ejecución está cancelada. Distinción obligatoria: `publishResult` puede publicar `execution.completed` aunque la ejecución esté cancelada, pero G3 descarta los eventos cuyo estado no sea `COMPLETED`; por tanto, el evento cancelado no demuestra por sí solo que se produzca un envío externo. El riesgo crítico es una carrera por cambio de asignación: la asignación puede cambiar en CRM mientras el proveedor procesa la solicitud, y, al regresar la respuesta, `runPipelineLoop` puede ejecutar `toolCalls` y capabilities antes de que G2 vuelva a comprobar la vigencia (no se encontró mecanismo CRM → ai-service que solicite `Execution.cancel()` de inmediato; `cancel()` aparece solo en G2 o en el catch del pipeline), lo que permite efectos posteriores al cambio de asignación; G2 puede detectar después la pérdida de vigencia y cancelar lógicamente la ejecución. A ello se suma una deficiencia local independiente: `runPipelineLoop` no comprueba `context.cancelled` antes de ejecutar capabilities (sin afirmar que el takeover actual invoque esa ruta), y no hay garantía suficiente de invalidación de esos efectos tardíos. La clasificación bloqueante de ARQ-07 se fundamenta en el riesgo de ejecutar capabilities después de un takeover o una reasignación, no en afirmar que el evento cancelado causa automáticamente un envío de WhatsApp. ARQ-07 sigue como **bloqueante**, centrado en la garantía obligatoria de invalidación y rechazo de efectos tardíos, incluida la ejecución de capabilities.

### 3.4 Pregunta 4 — Propagación de evAssignedId y validaciones G3/G4 (ARQ-08)

**Estado: Ausencia verificada — `evAssignedId` no existe; las validaciones G3/G4 consultan la proyección, no la autoridad CRM.**

**`evAssignedId` no existe ni se propaga:**

- No existe en ningún archivo revisado. Grep por `evAssignedId`, `ev_assigned_id`, `EvAssignedId` en `crm/` y en `crm/microservices/ai-service/` devolvió **0 matches**.
- `Evento.model.ts` no tiene discriminator `EvAssigned` ni campo de identificación de transición.
- `Contacto.model.ts` no tiene `evAssignedId`.
- `ExecutionContext` (ai-service `execution-context.ts`) no tiene `evAssignedId`.
- `Execution` (ai-service `execution.ts`) no tiene `evAssignedId`.
- El mensaje de Rabbit `ai-service.message-received` (schema zod, L21-36) no tiene `evAssignedId`.
- El envelope de RabbitMQ (`rabbitmq.mapper.ts`) no tiene campo de EvAssigned.
- `Outbox.model.ts` payload es `Record<string, any>` genérico; el payload de `assignmentSync.service.ts` tiene `contactId`, `workspaceId`, `target`, `actorUserId`, `prevCrmAssigned`, `reason` — sin `evAssignedId`.

**Cómo se transporta:** No se transporta (no existe). No se persiste en el contexto de ejecución.

**G1/G2/G3/G4 consultan la proyección, no la autoridad CRM:**

- G1/G2/G3/G4 consultan `contact_assignments` en `AI_SERVICE_MONGO_DB` (misma cluster, distinta DB de CRM), que es una **proyección operativa** sincronizada vía Outbox desde CRM. Ninguna consulta `Contacto` de CRM directamente para validar vigencia.
- La proyección eventual **no es una alternativa aceptable** a la verificación contra CRM en G3/G4. El contrato aprobado exige consultar CRM antes del efecto externo. La convergencia eventual no garantiza la validación de la autoridad vigente; no se acepta como solución equivalente la frase "CRM o proyección convergente".
- Si el Outbox está retrasado o falló, `contact_assignments` puede no reflejar la asignación vigente de CRM, y una ejecución obsoleta podría pasar la validación.

**Comportamiento ante indisponibilidad:**

- G1 es fail-open por diseño (continúa si Mongo de ai-service no responde; defensa en capas G2/G3/G4).
- G2/G3/G4 fallan cerrados ante indisponibilidad de la base consultada, pero **siguen consultando la proyección** `contact_assignments`, no CRM.

**Bypass de `botId` en mensajes legacy:**

- G3 y G4 permiten continuar si falta `botId` en mensajes legacy. Este bypass debe registrarse y exigirse una estrategia de compatibilidad, drenaje o rechazo de eventos antiguos sin identidad de bot.

**Superficie de separación de autoridad incumplida (evidencia):**

- `AiContactAssignment.resolver.ts` expone `assignContact`, `reassignContact` y `unassignContact`. Estas operaciones llaman directamente a ai-service mediante `ContactAssignmentClient`; **no actualizan `Contacto` CRM ni crean `EvAssigned`**.
- Esta superficie documenta la separación de autoridad actualmente incumplida. Se reutilizan ARQ-05 y ARQ-08 para este mismo problema; no se crea otro ID.

**Reevaluación de severidad y bloqueo (criterios plan §3/§8):**

- Sin `evAssignedId` no hay forma de comparar la ejecución contra la asignación vigente de CRM por identidad de transición.
- Las validaciones G3/G4 usan la proyección `contact_assignments` de ai-service, no la autoridad CRM, y no usan identidad de transición EvAssigned.
- Es requisito de G3/G4 y de resolución ABA.
- **ARQ-08 se mantiene como bloqueante** (impacto Medio; Bloquea WAB-78: Sí).

**Destino:** WAB-78 (contrato G3/G4).

**Recomendación:** Definir `evAssignedId` como metadata que viaje con la ejecución; persistirlo en `ExecutionContext`; validar contra CRM antes de cada efecto externo. No dar por válida la garantía porque el campo exista en un DTO. No usar la ausencia de EvAssigned como explicación única para la deduplicación del mismo mensaje entrante (ese rol corresponde a `originMessageId`).

### 3.5 Pregunta 5 — operationType de EvAssigned (ARQ-09)

**Estado: Propuesta pendiente, no aprobada.**

**¿Qué contiene actualmente el modelo `Evento`?**

- `Evento.model.ts` (L1-61): campos `_id`, `searchText`, `workspace`, `createdAt`, `updatedAt`, `contacto`. No hay discriminator `__t` configurado para subtipos (aunque `resolveType` en L7-12 lee `value.__t`). No hay `EvAssigned` como discriminator. No hay `operationType`, `assignedType`, `originMessageId`, `prevAssigned`, `newAssigned`.

**¿Existen eventos de asignación existentes?**

- No hay discriminator `EvAssigned` ni eventos de asignación persistidos como tales en la colección `eventos`.

**¿Es necesario `operationType` y dónde correspondería definirlo?**

- El canónico D.1.6 (L754) menciona `operationType` como "propuesta pendiente — separado de `AssignedType`".
- D.1.6 (L766): "`operationType` es una propuesta pendiente de aprobación, no una decisión aprobada por el usuario."
- Correspondería definirlo en el modelo `Evento` como campo del discriminator `EvAssigned`, si se aprueba.

**Restricciones respetadas:**

- No se asume que está aprobado.
- No se confunde con `AssignedType` (que sí está aprobado: `human`/`agent`/`unassigned`).
- No se modificó el modelo.
- Continúa siendo una decisión de implementación diferible a WAB-79, justificada por la evidencia de que no está aprobado y no está en el modelo.

**Conclusión:** `operationType` es una propuesta no aprobada, diferible a WAB-79.

---

## 4. Estado de ARQ-05 a ARQ-09, individualmente

### ARQ-05 — Identidad de EvAssigned vigente

| Campo | Valor |
|-------|-------|
| ID | ARQ-05 |
| Dominio | D3 |
| Estado | **Ausencia de implementación verificada** (ausencia de `EvAssigned`/`evAssignedId` comprobada en código) |
| Evidencia | `crm/src/models/chat/Contacto.model.ts` (L157: solo `ref_user_assigned`; sin `assignedType`, `ref_agent_assigned`, `evAssignedId`); `crm/src/models/events/Evento.model.ts` (L1-61: sin discriminator `EvAssigned`, sin `operationType`, sin referencia a asignación vigente); grep `evAssignedId`/`EvAssigned` en `crm/src` = 0 matches en código (solo en docs) |
| Hallazgo | No existe ninguna de las tres alternativas (consulta último evento, referencia directa en Contacto, alternativa presente en código). El contrato aprobado exige una identidad única para cada transición confirmada de asignación; esa identidad permite saber qué transición autorizó una ejecución y resolver ABA. `originMessageId` sirve para trazabilidad/idempotencia del mensaje de origen y no sustituye la identidad de transición. `ref_ev_assigned` es un nombre propuesto, no un campo aprobado. La elección entre alternativas (referencia directa en Contacto vs. consulta del último evento) es una decisión técnica de diseño, no de producto. |
| Impacto | Alto |
| Bloquea WAB-78 | Sí |
| Justificación | Sin identidad de transición no se puede resolver ABA ni identificar qué transición autorizó una ejecución. Es requisito previo para G3/G4 y para la atomicidad Contacto+EvAssigned+Outbox. No introducir `assignmentVersion`. |
| Destino | WAB-78 (contrato D3) → definición del modelo `EvAssigned` y campo en Contacto |
| Recomendación | Comparar ambas alternativas y recomendar una justificando atomicidad, concurrencia, ABA, índices y coste de lectura. No usar `assignmentVersion`. No reabrir decisiones P1/P2/P4/P9/P10. |

### ARQ-06 — Atomicidad Contacto + EvAssigned + Outbox

| Campo | Valor |
|-------|-------|
| ID | ARQ-06 |
| Dominio | D3 |
| Estado | **Ausencia de transacción verificada** (el mecanismo actual es recuperación compensatoria, no Transactional Outbox) |
| Evidencia | `crm/src/services/Contacto.service.ts` (L2826-2900: `findByIdAndUpdate` sin sesión); `crm/src/infrastructure/ai-service/assignmentSync.service.ts` (L155-199: `OutboxModel.create` fuera de transacción; L310-386: `ejecutarOutboxAssignmentSync` con `updateOne` CAS separado); grep `startTransaction`/`mongoose.Session` en `crm/src` = 0 matches |
| Hallazgo | No existe operación atómica que incluya Contacto, EvAssigned y Outbox. No hay sesión de transacción MongoDB compartida. El camino normal no genera necesariamente un documento Outbox (solo se crea cuando falla HTTP). El mecanismo actual es recuperación compensatoria, no un Transactional Outbox que registre cada transición, y no proporciona la misma garantía que el diseño transaccional aprobado. Ventana de fallo: ai-service recibe y aplica correctamente una asignación; la llamada remota termina correctamente; posteriormente falla la persistencia de la asignación en CRM; no existe garantía de que se genere un Outbox compensatorio porque el mecanismo actual lo utiliza principalmente ante errores de la llamada HTTP, lo que puede dejar la proyección de ai-service adelantada respecto de la autoridad CRM. |
| Impacto | Crítico |
| Bloquea WAB-78 | Sí |
| Justificación | La atomicidad es requisito funcional aprobado (D.1.8 L787-789). Sin ella, pueden quedar estados parciales (asignación efectiva sin historial correspondiente). Ausencia de transacción en el código: verificada. Soporte de transacciones en Atlas: no verificado (destino D6). Solución técnica objetivo: pendiente de implementación y aceptación. |
| Destino | WAB-78/D6 (verificación Atlas) → WAB-83/WAB-84 |
| Recomendación | Implementar transacción MongoDB para Contacto + EvAssigned + Outbox; verificar soporte en Atlas (pendiente, no afirmar compatibilidad sin evidencia del deployment real). Si no es viable, diseñar alternativa recuperable aprobada. |

### ARQ-07 — Cancelación e invalidación de ejecuciones en ai-service

| Campo | Valor |
|-------|-------|
| ID | ARQ-07 |
| Dominio | D3 |
| Estado | **Ausencia parcial verificada** (cancelación lógica sí; cancelación física de LLM no; invalidación de efectos tardíos incompleta) |
| Evidencia | `crm/microservices/ai-service/src/domain/entities/execution.ts` (L230-248: `cancel()` marca CANCELLED, es no-op si terminal); `crm/microservices/ai-service/src/engine/steps/invoke-provider.step.ts` (L81: `provider.invoke(request)` sin AbortController/signal); `crm/microservices/ai-service/src/engine/execution-engine.ts` (L80-91: G2 cancela; L166-200: `runPipelineLoop` no revalida `context.cancelled` antes de ejecutar capabilities; L202-234: `publishResult` publica `execution.completed` sin importar estado cancelado); `crm/microservices/ai-service/src/application/services/vigencia.service.ts` (G1/G2 gate); `crm/src/Rabbit/consumers/executionCompleted.consumer.ts` (G3); `crm/microservices/Difusion/src/Worker/__tests__/vigencia-g4.test.ts` (G4) |
| Hallazgo | Existe cancelación lógica (`cancel()`), tests de vigencia G1-G4, pero no hay cancelación física de llamadas LLM (no hay AbortController ni abort signal en `IAIProvider.invoke` ni en `InvokeProviderStep`). La ausencia de aborto físico por sí sola no es incumplimiento fatal si el proveedor no lo permite; la obligación crítica es invalidar el resultado y rechazar efectos tardíos. Nueva brecha: `runPipelineLoop` no revalida `context.cancelled` ni vigencia antes de ejecutar capabilities tras `pipeline.execute(context)`. `publishResult` publica `execution.completed` semánticamente mal etiquetado aunque la ejecución esté cancelada. Distinción obligatoria: `publishResult` puede publicar `execution.completed` aunque la ejecución esté cancelada, pero G3 descarta los eventos cuyo estado no sea `COMPLETED`; por tanto, el evento cancelado no demuestra por sí solo que se produzca un envío externo. El riesgo crítico es una carrera por cambio de asignación: la asignación puede cambiar en CRM mientras el proveedor procesa la solicitud y, al regresar la respuesta, `runPipelineLoop` puede ejecutar `toolCalls` y capabilities antes de que G2 vuelva a comprobar la vigencia (no se encontró mecanismo CRM → ai-service que solicite `Execution.cancel()` de inmediato; `cancel()` aparece solo en G2 o en el catch del pipeline); G2 puede detectar después la pérdida de vigencia y cancelar lógicamente la ejecución. Deficiencia local independiente: `runPipelineLoop` no comprueba `context.cancelled` antes de ejecutar capabilities, sin afirmar que el takeover actual invoque esa ruta. La clasificación bloqueante se fundamenta en el riesgo de ejecutar capabilities después de un takeover o una reasignación, no en afirmar que el evento cancelado causa automáticamente un envío de WhatsApp. |
| Impacto | Alto |
| Bloquea WAB-78 | Sí |
| Justificación | Sin invalidación garantizada de efectos tardíos (incluida ejecución de capabilities), una ejecución obsoleta puede producir efectos tras takeover o reasignación. Es requisito de E.4.11. No se exige aborto físico absoluto si el proveedor no lo permite. |
| Destino | WAB-78 (contrato) → WAB-81 |
| Recomendación | Definir contrato de cancelación e invalidación: asegurar que `runPipelineLoop` revalide `context.cancelled` antes de capabilities; asegurar que `publishResult` no publique eventos para ejecuciones canceladas; diferenciar cancelación de invalidación. No eliminar RabbitMQ ni workers. |

### ARQ-08 — Propagación de evAssignedId y validaciones G3/G4

| Campo | Valor |
|-------|-------|
| ID | ARQ-08 |
| Dominio | D3 |
| Estado | **Ausencia verificada — `evAssignedId` no existe; las validaciones G3/G4 consultan la proyección, no la autoridad CRM** |
| Evidencia | grep `evAssignedId`/`ev_assigned_id`/`EvAssignedId` en `crm/` y `crm/microservices/ai-service/` = 0 matches; `crm/microservices/ai-service/src/domain/value-objects/execution/execution-context.ts` (sin `evAssignedId`); `crm/microservices/ai-service/src/domain/entities/execution.ts` (sin `evAssignedId`); `crm/microservices/ai-service/src/infrastructure/messaging/consumers/message-received-handler.ts` (schema Rabbit sin `evAssignedId`); `crm/src/infrastructure/ai-service/aiVigencia.repo.ts` (valida contra `contact_assignments` en ai-service, no contra CRM ni contra EvAssigned) |
| Hallazgo | `evAssignedId` no existe como campo, no se propaga, no se persiste en contexto de ejecución, no se valida contra CRM. G1/G2/G3/G4 consultan `contact_assignments` en `AI_SERVICE_MONGO_DB` (proyección de ai-service), no la autoridad CRM directamente; la proyección eventual no es alternativa aceptable a la verificación contra CRM en G3/G4. G1 es fail-open por diseño; G2/G3/G4 fallan cerrados ante indisponibilidad de la base consultada pero siguen consultando la proyección. G3 y G4 permiten continuar si falta `botId` en mensajes legacy (bypass sin estrategia de compatibilidad/drenaje/rechazo). `AiContactAssignment.resolver.ts` (`assignContact`, `reassignContact`, `unassignContact`) llama directamente a ai-service vía `ContactAssignmentClient`; no actualiza `Contacto` CRM ni crea `EvAssigned`. La convergencia eventual no garantiza la validación de la autoridad vigente. |
| Impacto | Medio |
| Bloquea WAB-78 | Sí |
| Justificación | Sin `evAssignedId` no hay forma de comparar la ejecución contra la asignación vigente de CRM por identidad de transición. Es requisito de G3/G4 y de resolución ABA. Reevaluado con criterios del plan §3/§8: se mantiene como bloqueante. |
| Destino | WAB-78 (contrato G3/G4) |
| Recomendación | Definir `evAssignedId` como metadata que viaje con la ejecución; persistirlo en `ExecutionContext`; validar contra CRM antes de cada efecto externo. No dar por válida la garantía porque el campo exista en un DTO. Reutilizar ARQ-05/ARQ-08 para la superficie de separación de autoridad (`AiContactAssignment.resolver.ts`); no crear otro ID por el mismo problema. |

### ARQ-09 — operationType de EvAssigned

| Campo | Valor |
|-------|-------|
| ID | ARQ-09 |
| Dominio | D3 |
| Estado | **Propuesta pendiente, no aprobada** (diferible a WAB-79) |
| Evidencia | `crm/src/models/events/Evento.model.ts` (sin `operationType`); Canónico D.1.6 L754 (menciona `operationType` como "propuesta pendiente — separado de `AssignedType`); D.1.6 L766 ("`operationType` es una propuesta pendiente de aprobación, no una decisión aprobada por el usuario") |
| Hallazgo | `operationType` no está en el modelo `Evento` ni aprobado. Está claramente separado de `AssignedType` (que sí está aprobado: `human`/`agent`/`unassigned`). |
| Impacto | Medio |
| Bloquea WAB-78 | No |
| Justificación | Es una decisión de implementación que puede diferirse a WAB-79 sin bloquear D3. No confundir con `AssignedType`. |
| Destino | WAB-79 (propuesta y aprobación) |
| Recomendación | No introducir `operationType` en D3. Mantener como pendiente para WAB-79. |

---

## 5. Arquitectura real del flujo de asignación y ejecución

### 5.1 Flujo de recepción de mensaje (CRM)

```
1. Webhook llega a Contacto.resolver.newMessageReceived (L460-486)
2. ContactoService.newMessageReceived (L511): busca contacto por phone/wid/jid en ref_works
3. Si contacto existe y no está bloqueado: guarda mensaje, abre/actualiza conversación
4. Devuelve { contacto, mensaje }
5. Resolver publica CHANGE_CONTACTO, NEW_MSJ_OF_CONTACT (pubSub)
```

**Autoasignación (solo humano):** En `Contacto.resolver.sendMessage` (L558-610), si el contacto estaba sin asignar y hay `ctx.user.uowActive`, llama `updateContactoByID` con `ref_user_assigned` en segundo plano. Esto dispara `sincronizarAsignacionPreCrm` (HTTP a ai-service ANTES del write de CRM) y luego escribe CRM. Si HTTP falla, CRM no se escribe y queda Outbox PENDING.

**Asignación a agente IA (bot):** El canónico D.1.5 prevé `ref_agent_assigned` (referencia a `AgentInstance._id`) pero **no existe en el código actual** (`Contacto.model.ts` no tiene `ref_agent_assigned` ni `assignedType`). El modelo de ai-service `ContactAssignment` sí soporta `AssigneeType.BOT` (L3-18), pero no hay camino de escritura desde CRM hacia ai-service para asignaciones a bot vía `sincronizarAsignacionPreCrm` (que solo maneja `assigneeType: "user"` — L43-45).

### 5.2 Flujo de routing y ejecución (ai-service)

```
1. CRM publica execution.completed? No — el flujo entrante va por Rabbit:
2. Webhook/CRM → RabbitMQ cola 'ai-service.message-received' (contenido: workspaceId, conversationId, contactId, botId, content, traceId, source)
3. MessageReceivedHandler (ai-service) consume (prefetch=1)
4. G1: VigenciaService.check() — descarta si human_takeover/assignment_changed/bot_deactivated (fail-open si Mongo no responde)
5. IBotRuntimeResolver.resolve(botId): carga Bot, busca BotInstance activo (primero que encuentre), computa EffectiveConfig (o synthetic `default-{botId}`)
6. Builds ExecutionContext con CorrelationData (executionId, traceId, correlationId) — SIN evAssignedId
7. ProcessMessageService.executeWithContext → ExecutionEngine.process
8. ExecutionEngine: crea Execution entity, markRunning, guarda en repo
9. Pipeline: LoadConfiguration → LoadConversation → ResolvePrompt → LoadMemory → LoadRulesSkills → ExecuteRAG → BuildPrompt → InvokeProvider
10. InvokeProvider: invoca LLM con API key del bot (sin abort signal)
11. G2: revalida vigencia antes de markCompleted; si inválido, context.cancel() + execution.cancel()
12. markCompleted o markFailed
13. publishResult: publica `EventTypes.EXECUTION_COMPLETED` en Rabbit exchange 'ai-service.events' (siempre, incluso si cancelado — publicación semánticamente mal etiquetada que transporta eventos cancelados; G3 los descarta por estado, pero el evento viajó por Rabbit)
14. CRM consumer executionCompleted (G3): valida vigencia contra contact_assignments en ai-service; si válido, persistOutgoing → dispatchOutgoing → WhatsApp
```

### 5.3 Flujo de cancelación / takeover

```
Takeover humano (CRM):
1. Usuario asigna contacto a humano (updateContactoByID con ref_user_assigned)
2. sincronizarAsignacionPreCrm: HTTP a ai-service (assign/unassign en contact_assignments) ANTES de write CRM
3. Write CRM: ref_user_assigned = UOW humano
4. Si HTTP falló: Outbox PENDING, converge eventualmente

Invalidación en ai-service:
5. Las ejecuciones activas del contacto no se cancelan físicamente (no hay AbortController)
6. G2 revalida antes de completar: si human_takeover, cancela (estado CANCELLED)
7. Pero publishResult publica execution.completed aunque esté cancelado
8. G3 en CRM descarta si human_takeover (no persiste EvMensaje)
```

**Anomalía semántica/operativa:** `publishResult` puede publicar `execution.completed` aunque la ejecución haya quedado cancelada. G3 descarta los eventos cuyo estado no sea `COMPLETED`. No se encontró evidencia de otro consumidor que permita afirmar que ese evento produce por sí mismo un efecto externo; por tanto, la publicación del evento constituye una anomalía semántica/operativa, no un efecto externo obsoleto demostrado. El riesgo principal y comprobado sigue siendo que `runPipelineLoop` ejecute `toolCalls` o capabilities después de recibir la respuesta del proveedor y antes de que G2 vuelva a validar la vigencia de la asignación.

---

## 6. Análisis de EvAssigned e identidad de transición

### 6.1 ¿Qué existe hoy?

- **No existe `EvAssigned`** como discriminator de `Evento`. `Evento.model.ts` es una clase base con `searchText`, `workspace`, `createdAt`, `updatedAt`, `contacto`. El `resolveType` (L7-12) lee `value.__t` pero no hay subtipos definidos.
- **No existe `evAssignedId`** en ningún modelo, DTO, contexto de ejecución o mensaje de Rabbit.
- **No existe `contactAssignmentHistory`** como colección o campo en código (grep 0 matches en `crm/src`).
- `Contacto` solo tiene `ref_user_assigned` (referencia a `UsOrgsWorks`), sin `assignedType` ni `ref_agent_assigned`.

### 6.2 ¿Qué dice el diseño aprobado?

- D.1.5: `assignedType` = `human`/`agent`/`unassigned`; `ref_user_assigned` = UOW; `ref_agent_assigned` = referencia a `AgentInstance._id`. Sin `assignmentVersion`.
- D.1.6: `EvAssigned` es discriminator de `Evento` en colección `eventos`, exclusivamente CRM. Cada transición confirmada tiene identificador único. `originMessageId` es trazabilidad/idempotencia del mensaje de origen, no sustituye la identidad de EvAssigned.
- D.1.7: La identidad única de EvAssigned identifica qué transición autorizó cada ejecución (resolución ABA). Ejemplo: M1→Agent A (E1), takeover→Human (E2), desasignación→Unassigned (E3), M2→Agent A (E4). La ejecución de M1 conserva E1 aunque Agent A vuelva por E4.

### 6.3 Brecha entre diseño y código

| Aspecto | Diseño aprobado | Código actual | Brecha |
|---------|----------------|---------------|--------|
| Modelo Contacto | `assignedType`, `ref_user_assigned`, `ref_agent_assigned` | Solo `ref_user_assigned` | Faltan `assignedType` y `ref_agent_assigned` |
| EvAssigned | Discriminator de Evento, identidad única | No existe | No hay modelo ni discriminator |
| Identidad de transición en Contacto | Referencia al evento vigente (o consulta último evento) | No existe | No hay forma de saber qué EvAssigned autorizó una ejecución |
| `evAssignedId` en ejecución | Metadata que viaja con la ejecución | No existe | ExecutionContext y Execution no lo tienen |
| `contactAssignmentHistory` | Conservar en esta fase | No existe | No hay colección ni campo |

### 6.4 Alternativas para identidad de EvAssigned vigente (pendiente de decisión)

1. **Consultar el último evento de asignación:** requeriría indexar `eventos` por `contacto` + `__t` = `EvAssigned` + `createdAt -1`. Existe índice `{contacto: 1, __t: 1, createdAt: -1}` (L22) pero no hay documentos `EvAssigned` que consultar.
2. **Referencia directa en Contacto:** agregar campo `ref_ev_assigned` (o similar) que apunte al `Evento._id` de la transición vigente. Requeriría escritura atómica con Contacto.
3. **Alternativa presente en código:** ninguna.

**Determinación:** Ninguna alternativa está implementada. La referencia directa en Contacto es la más coherente con el diseño aprobado (identidad única de transición, resolución ABA, atomicidad con Outbox), pero requiere una **decisión técnica de diseño** (no de producto) sobre el nombre del campo (`ref_ev_assigned` es propuesto, no aprobado) y la semántica de actualización. Queda como **pendiente de decisión técnica de diseño**.

### 6.5 Resistencia frente a ABA

Sin identidad de transición, una secuencia A → B → A no es detectable:
- Ejecución originada bajo transición A (E1) se ejecuta.
- Contacto se reasigna a B (E2), luego vuelve a A (E3).
- La ejecución antigua de M1 no tiene forma de saber que fue autorizada por E1 y no por E3.
- Con identidad de EvAssigned, la ejecución lleva E1 y se compara contra el EvAssigned vigente (E3); como difieren, se descarta.

**Conclusión:** El diseño actual no resiste ABA. Es un bloqueante para ARQ-05.

---

## 7. Análisis de atomicidad, transacciones y Outbox

### 7.1 Estado actual de las escrituras

El camino de asignación humana desde CRM se divide en dos caminos:

**Camino ASIGNAR (`sincronizarAsignacionPreCrm`):**
1. HTTP a ai-service (`contactAssignmentClient.assign`) — efecto remoto primero
2. Si HTTP OK: escribe CRM (`ref_user_assigned`)
3. Si HTTP falla: encaja Outbox `contact.assignment.sync`, relanza, CRM no se escribe
4. Outbox se procesa vía change stream (rápido) o poller (reintentos con backoff)

**Camino LIBERAR (`sincronizarLiberacionPostCrm`):**
1. Escribe CRM (`ref_user_assigned = null`) — el caller ya escribió
2. HTTP a ai-service (`contactAssignmentClient.unassign`)
3. Si HTTP falla: encaja Outbox, NO relanza, CRM ya escribió (estado conservador: bot bloqueado por contact_assignments viejo)

**`ejecutarOutboxAssignmentSync` (recuperación):**
1. Lee CRM actual (`ref_user_assigned`)
2. Guarda de supersede: si CRM se movió a algo distinto de `prev` y de `target`, se deja de lado
3. HTTP converge (no-op si ya coincide)
4. CAS write a CRM: `updateOne({_id, ref_user_assigned: crmActual}, {$set: {ref_user_assigned: destino}})`

### 7.2 ¿Hay transacción?

**No.** Verificación exhaustiva:
- `mongoose.Session` o `startTransaction()`: 0 matches en `crm/src`.
- `OutboxModel.create` se llama fuera de cualquier sesión de transacción.
- `ContactoModel.findByIdAndUpdate` se llama fuera de cualquier sesión.
- No hay código que comparta una sesión entre Contacto y Outbox.

### 7.3 ¿Misma base de datos?

Contacto y Outbox están en la misma base de datos CRM (mismo `mongoose.connection`). Pero sin sesión, no hay atomicidad transaccional.

### 7.4 ¿Qué ocurre ante errores parciales?

| Escenario | Comportamiento actual |
|-----------|----------------------|
| HTTP asignación falla | Outbox PENDING, CRM no se escribe, relanza al usuario. Convergencia eventual vía Outbox. |
| Outbox create falla (asignar) | Log de error, divergia sin recuperación automática. El caller aborta igual (throw). |
| HTTP liberación falla | Outbox PENDING, CRM ya escribió, no relanza. contact_assignments viejo sigue bloqueando al bot (conservador). |
| Outbox agotado (50 intentos) | FAILED + métrica + intervención manual. No hay reconciliación automática. |
| Proceso cae entre write CRM y create Outbox | Divergencia sin recuperación hasta que el poller detecte (si el Outbox se creó) o para siempre (si no se creó). |
| Llamada remota OK, persistencia CRM falla después | ai-service recibe y aplica correctamente la asignación; la llamada remota termina correctamente; posteriormente falla la persistencia de la asignación en CRM. No existe garantía de que se genere un Outbox compensatorio, porque el mecanismo actual lo utiliza principalmente ante errores de la llamada HTTP. Puede dejar la proyección de ai-service adelantada respecto de la autoridad CRM. |

### 7.5 Idempotencia

- `convergerHttp`: idempotente (no-op si ya converge; 409 → reassign; 404 en unassign → éxito).
- `ejecutarOutboxAssignmentSync`: guarda de supersede + CAS. Doble ejecución (stream + poller) es inofensiva.
- `OutboxModel.create`: no es condicional; puede crear duplicados si se reintenta sin control, pero el procesamiento es idempotente.

### 7.6 Outbox ante fallos del publicador

- Change stream: primer intento rápido. Si el cursor cae, el poller cubre.
- Poller: backoff exponencial, 50 intentos máximos, luego FAILED + intervención manual.
- RabbitMQ (para `workspace.delete.requested`): el canal se adquiere solo para publicar; si Rabbit cae, la operación Outbox de assignment sync no toca Rabbit (solo HTTP).

### 7.7 Veredicto ARQ-06

**No existe operación atómica Contacto + EvAssigned + Outbox.** No hay sesión de transacción. No hay EvAssigned en el modelo. El mecanismo actual de `assignmentSync.service.ts` es una **recuperación compensatoria**, no un Transactional Outbox que registre cada transición: el camino normal no genera necesariamente un documento Outbox (solo se crea cuando falla HTTP) y `OutboxModel.create` está fuera de cualquier transacción compartida con el write de Contacto. No proporciona la misma garantía que el diseño transaccional aprobado. La atomicidad lógica del diseño (D.1.8) no está implementada. La compatibilidad con Atlas no se puede afirmar sin evidencia del deployment real.

**Ventana de fallo documentada:** ai-service recibe y aplica correctamente una asignación; la llamada remota termina correctamente; posteriormente falla la persistencia de la asignación en CRM. No existe garantía de que se genere un Outbox compensatorio, porque el mecanismo actual lo utiliza principalmente ante errores de la llamada HTTP. Esto puede dejar la proyección de ai-service adelantada respecto de la autoridad CRM.

**Dependencia con D6:** ARQ-06 se coordina con D6 (verificación de transacciones en Atlas). No se afirma compatibilidad Atlas sin evidencia. Solución técnica objetivo pendiente de implementación y aceptación.

---

## 8. Análisis de cancelación e invalidación

### 8.1 Las cinco diferencias obligatorias

1. **Solicitud de cancelación:** `context.cancel()` y `execution.cancel()` existen (execution.ts L230-248, execution-context.ts L441-443). Marcan estado CANCELLED.

2. **Cancelación física del trabajo en curso:** **NO existe.** `InvokeProviderStep` llama `provider.invoke(request)` (L81) sin AbortController ni abort signal. No se encontró mecanismo de aborto de llamadas LLM en curso. El diseño aprobado exige intentar cancelar físicamente cuando sea posible, pero si el proveedor no lo permite, la obligación crítica es invalidar el resultado y rechazar todo efecto tardío; por tanto, la ausencia de aborto físico por sí sola **no es incumplimiento fatal ni requisito absoluto**.

3. **Invalidación del derecho a producir efectos:** G2 (execution-engine.ts L80-91) cancela si vigencia falla antes de completar. G3 (CRM consumer) descarta si inválido. G4 (Difusion worker) descarta pre-POST. Pero `publishResult` (execution-engine.ts L202-234) publica `execution.completed` **independientemente del estado cancelado** — publicación semánticamente mal etiquetada que transporta eventos cancelados. No se afirma que ese evento por sí mismo produzca un envío externo en el flujo G3 actualmente inspeccionado (G3 descarta payloads cuyo estado no es `completed`). No se asume la existencia de otros consumidores que produzcan efectos sin evidencia.

**Nueva brecha en `runPipelineLoop` (execution-engine.ts L166-200):** el loop espera `pipeline.execute(context)`, lee `context.providerResponse`, y si hay `toolCalls` invoca `capabilityExecutor.execute(...)`, pero **no vuelve a comprobar `context.cancelled` ni revalida vigencia antes de ejecutar las capabilities**. No se encontró mecanismo CRM → ai-service que solicite `Execution.cancel()` de inmediato (§3.3); en el flujo inspeccionado `cancel()` aparece solo en G2 o en el catch del pipeline. El riesgo demostrado es una carrera por cambio de asignación: la asignación puede cambiar en CRM mientras el proveedor procesa y, al regresar la respuesta, `runPipelineLoop` puede ejecutar capabilities antes de que G2 vuelva a comprobar la vigencia. Además, la ausencia de comprobación de `context.cancelled` es una deficiencia local independiente, sin afirmar que el takeover actual invoque esa ruta.

4. **Descarte de resultados obsoletos:** G3 descarta en CRM (no persiste EvMensaje). G4 descarta en worker (no POST). Pero no hay mecanismo en ai-service para impedir que un resultado completado se use internamente ni para impedir la publicación del evento Rabbit.

5. **Persistencia e historial de ejecución:** `Execution` se persiste vía `IExecutionRepository`. `ExecutionEvent` registra transiciones (ExecutionStarted, ExecutionFinished, ExecutionFailed, ExecutionCancelled). No hay persistencia de cancelación física ni de invalidez de resultado.

### 8.2 Escenarios de cancelación

| Escenario | Comportamiento actual |
|-----------|----------------------|
| LLM en curso, se solicita cancelación | No hay cancelación física. El LLM continúa. `cancel()` marca CANCELLED pero el proveedor sigue generando. |
| LLM terminó, resultado no aplicado | G2 revalida antes de `markCompleted`. Si inválido, cancela y no marca completado. Pero `publishResult` publica evento `execution.completed`. |
| Resultado antiguo llega después de nueva asignación | G2/G3 detectan `assignment_changed` o `human_takeover` y descartan. Depende de que `contact_assignments` esté convergente. |
| Takeover humano, LLM procesando | No se encontró mecanismo de cancelación inmediata desde CRM hacia ai-service (§3.3): el LLM continúa generando. Al retornar el proveedor, `runPipelineLoop` puede ejecutar capabilities antes de que G2 revalide la vigencia y cancele lógicamente. `publishResult` publica evento `execution.completed` aunque la ejecución esté cancelada. |
| Reinicio de ai-service | Ejecuciones en memoria se pierden. No hay evidencia de persistencia de estado de cancelación que impida republicar tras reinicio. |

### 8.3 Anomalía semántica/operativa: `publishResult` publica evento aunque cancelado

En `execution-engine.ts` L202-234, `publishResult` se ejecuta en el `finally` y publica `EventTypes.EXECUTION_COMPLETED` con `execution.toJSON()`. No verifica `execution.isCancelled()` antes de publicar. Esto significa que aunque G2 cancele la ejecución, el evento viaja por Rabbit. El consumer de CRM (G3) lo descarta si la vigencia falla (payload con `status: cancelled` no es `completed`), pero:
- El evento ya consumió ancho de banda de Rabbit.
- No se afirma que ese evento por sí mismo produzca un envío externo en el flujo G3 actualmente inspeccionado.
- No se asume la existencia de otros consumidores que produzcan efectos sin evidencia.

**Conclusión ARQ-07:** La cancelación lógica existe; la cancelación física de LLM no existe (y no se exige como absoluto si el proveedor no lo permite). La invalidación de efectos externos está incompleta porque `runPipelineLoop` no revalida `context.cancelled` ni la vigencia antes de ejecutar capabilities tras `pipeline.execute(context)`, y `publishResult` publica `execution.completed` indistintamente incluso cuando la ejecución está cancelada. Distinción obligatoria: `publishResult` puede publicar `execution.completed` aunque la ejecución esté cancelada, pero G3 descarta los eventos cuyo estado no sea `COMPLETED`; por tanto, el evento cancelado no demuestra por sí solo que se produzca un envío externo. El riesgo crítico es una carrera por cambio de asignación: la asignación puede cambiar en CRM mientras el proveedor procesa la solicitud, y, al regresar la respuesta, `runPipelineLoop` puede ejecutar `toolCalls` y capabilities antes de que G2 vuelva a comprobar la vigencia (no se encontró mecanismo CRM → ai-service que solicite `Execution.cancel()` de inmediato; `cancel()` aparece solo en G2 o en el catch del pipeline), lo que permite efectos posteriores al cambio de asignación; G2 puede detectar después la pérdida de vigencia y cancelar lógicamente la ejecución. A ello se suma una deficiencia local independiente: `runPipelineLoop` no comprueba `context.cancelled` antes de ejecutar capabilities (sin afirmar que el takeover actual invoque esa ruta), y no hay garantía suficiente de invalidación de esos efectos tardíos. La clasificación bloqueante de ARQ-07 se fundamenta en el riesgo de ejecutar capabilities después de un takeover o una reasignación, no en afirmar que el evento cancelado causa automáticamente un envío de WhatsApp. ARQ-07 sigue como **bloqueante**, centrado en la garantía obligatoria de invalidación y rechazo de efectos tardíos, incluida la ejecución de capabilities.

---

## 9. Análisis de propagación y validaciones G3/G4

### 9.1 Recorrido completo del `evAssignedId`

**`evAssignedId` no existe.** No hay rastro del campo en ningún archivo de CRM ni de ai-service:

| Punto del recorrido | ¿Existe `evAssignedId`? | Evidencia |
|---------------------|------------------------|-----------|
| CRM (Contacto) | No | `Contacto.model.ts` sin `evAssignedId` |
| CRM (Evento/EvAssigned) | No | `Evento.model.ts` sin discriminator `EvAssigned` |
| CRM (Outbox payload) | No | `Outbox.model.ts` payload es `Record<string, any>` genérico; `assignmentSync.service.ts` payload tiene `contactId`, `workspaceId`, `target`, `actorUserId`, `prevCrmAssigned`, `reason` — sin `evAssignedId` |
| Rabbit CRM→ai-service | No | No hay publicación de asignación a ai-service vía Rabbit con `evAssignedId`; la sincronización es HTTP directa (`contact-assignment.client.ts`) |
| ai-service (ContactAssignment) | No | `contact-assignment.ts` no tiene `evAssignedId` |
| ai-service (Execution) | No | `execution.ts` no tiene `evAssignedId` |
| ai-service (ExecutionContext) | No | `execution-context.ts` no tiene `evAssignedId` |
| Rabbit ai-service→CRM (execution.completed) | No | `rabbitmq.mapper.ts` metadata: traceId, correlationId, executionId, workspaceId, conversationId, timestamp, source — sin `evAssignedId` |
| CRM (executionCompleted consumer / G3) | No | `aiVigencia.repo.ts` valida contra `contact_assignments`, no contra EvAssigned |
| G3/G4 validación | No | Valida contra `contact_assignments` (proyección ai-service), no contra CRM, sin identidad de transición |

### 9.2 Existencia del identificador vs. propagación técnica vs. integridad

| Dimensión | Estado | Evidencia |
|-----------|--------|-----------|
| Existencia del identificador | **No existe** | `evAssignedId` no está definido en ningún modelo, DTO o schema |
| Propagación técnica | **No aplica** | No hay campo que propagar |
| Integridad del identificador | **No aplica** | Sin campo, no hay integridad que proteger |
| Validación de vigencia contra CRM | **No existe** | G1/G2/G4 validan contra `contact_assignments` en ai-service (misma cluster, distinta DB). G3 también usa `aiVigencia.repo.ts` que lee `contact_assignments` de ai-service. Ninguna consulta CRM directamente para vigencia. |
| Validación antes de cada efecto externo | **Parcial** | G1 (entrada), G2 (pre-completar), G3 (pre-persistir EvMensaje), G4 (pre-POST). Pero `publishResult` publica evento Rabbit aunque cancelado, y no hay validación en el momento del propio publish. |
| Comparación contra asignación vigente de CRM | **No** | Se compara contra `contact_assignments` de ai-service, que es una proyección eventual (sincronizada vía Outbox), no contra CRM directamente. |
| Ejecuciones antiguas después de reasignación | **Detectadas si `contact_assignments` convergió** | G2/G3 detectan `assignment_changed` o `human_takeover`. Pero si `contact_assignments` no está convergente (retraso de Outbox), una ejecución antigua puede pasar la validación. |
| CRM no disponible durante validación | **G1: fail-open; G2/G3/G4: fail-closed** | G1 continúa si Mongo no responde (message-received-handler.ts L140-153). G2 lanza `VigenciaNoDisponibleError` → `markFailed` (fail-closed). G3 re-republica diferido (no envía con estado desconocido). G4 descarta (fail-closed). |
| Resultado anterior después de takeover humano | **G2/G3 descartan si convergente** | Pero `publishResult` ya publicó el evento Rabbit. Depende de que G3 lo descarte. |
| Falla cerrado | **G2/G3/G4: sí; G1: no (fail-open documentado)** | G1 es fail-open por diseño, con defensa en capas (G2/G3/G4). |

### 9.3 Brecha: validación contra proyección vs. autoridad CRM

El plan §3.5 y el canónico D.1.12 establecen que **CRM es la autoridad de vigencia** y que G3/G4 deben validar contra CRM. Sin embargo:

- G1/G2/G4 (ai-service) validan contra `contact_assignments` en la DB de ai-service (`AI_SERVICE_MONGO_DB`), que es una proyección operativa sincronizada vía Outbox desde CRM.
- G3 (CRM consumer) también valida contra `contact_assignments` de ai-service vía `aiVigencia.repo.ts` (misma cluster, distinta DB), no contra `Contacto` de CRM directamente.

Esto significa que la validación **no es contra la autoridad CRM** sino contra una proyección eventual. Si el Outbox está retrasado o falló, `contact_assignments` puede no reflejar la asignación vigente de CRM, y una ejecución obsoleta podría pasar la validación.

**Conclusión ARQ-08:** `evAssignedId` no existe. Las validaciones G3/G4 existen pero validan contra la proyección `contact_assignments` de ai-service, no contra CRM directamente, y no usan identidad de transición EvAssigned. ARQ-08 sigue como **bloqueante**.

---

## 10. Evaluación de concurrencia, idempotencia y ABA

### 10.1 Concurrencia en asignación

**Dos mensajes concurrentes para el mismo contacto:**

- `newMessageReceived` (Contacto.service.ts L511) no tiene mecanismo de bloqueo por contacto. Dos mensajes concurrentes para un contacto sin asignación podrían ambos pasar el filtro de "sin asignar" y desencadenar dos autoasignaciones.
- La autoasignación en `sendMessage` (Contacto.resolver.ts L601-610) corre en segundo plano (`.catch(log)`), sin sincronización entre llamadas concurrentes.
- `updateContactoByID` lee `before = ContactoModel.findById(id)` (L2849) y compara, pero no usa `findOneAndUpdate` con condición en el campo de asignación, por lo que hay ventana de carrera entre la lectura y el write.

**Dos procesos intentan asignar simultáneamente el mismo contacto:**

- `sincronizarAsignacionPreCrm` → `convergerHttp` hace `getActive` luego `assign`/`reassign`. Hay ventana de carrera entre el GET y el POST.
- `convergerHttp` maneja 409 (conflicto) con reassign (last-write-wins), lo que mitiga parcialmente pero no elimina la carrera.
- `ejecutarOutboxAssignmentSync` tiene guarda de supersede (L343-349) y CAS (L375-384), que mitigan carreras en la recuperación.
- Pero no hay mecanismo de bloqueo pesimista ni transacción que serialice las asignaciones.

**Idempotencia:**

- `convergerHttp`: idempotente (no-op si ya converge).
- `ejecutarOutboxAssignmentSync`: idempotente (guarda de supersede + CAS).
- Outbox retry: puede ejecutarse dos veces (stream + poller), es inofensiva.
- Pero `OutboxModel.create` no tiene deduplicación previa: si el mismo intento se reenvía, puede crear múltiples documentos Outbox PENDING (aunque el procesamiento sea idempotente).

### 10.2 ABA

**Secuencia A → B → A:**

- Sin identidad de `EvAssigned`, no hay forma de detectar que una ejecución fue autorizada por la transición A (E1) y no por la transición A posterior (E3).
- El contacto tendría `ref_user_assigned` = A después de E3, pero una ejecución antigua originada bajo E1 no tiene forma de saber que fue autorizada por E1 y no por E3.
- Con `evAssignedId`, la ejecución llevaría E1 y se compararía contra el vigente (E3): como difieren, se descarta.

**Estado actual:** No hay resistencia a ABA. Es un bloqueante de ARQ-05.

### 10.3 Mensajes duplicados / reentregados por Rabbit

- `executionCompleted.consumer.ts` tiene dedup por `executionId` con Redis SET NX EX 86400 (L280-303). Fail-open si Redis no disponible.
- `message-received-handler.ts` (ai-service) usa prefetch=1 y `msg.properties.messageId` para trazabilidad, pero no hay dedup explícito de mensajes de entrada duplicados.
- El Outbox retry es idempotente.
- `EvAssigned` identifica la **transición de asignación**; `originMessageId` identifica el **mensaje de origen** para trazabilidad/idempotencia. La ausencia de `EvAssigned` no es explicación única para la deduplicación del mismo mensaje entrante (ese rol corresponde a `originMessageId`, que tampoco tiene dedup persistente).

**Conclusión:** La idempotencia está parcialmente cubierta (Outbox, dedup de execution.completed), pero no hay dedup persistente por identidad del mensaje de origen; un delivery unacked reentregado puede crear otra ejecución con otro `executionId`. La concurrencia en asignación no está resuelta sin transacción ni bloqueo por contacto. ABA no está resistido.

---

## 11. Evaluación de los escenarios del apartado 6

Para cada escenario: estado inicial, evento disparador, secuencia, fuente de verdad, carreras, resultado esperado, comportamiento verificable, brecha/riesgo.

| # | Escenario | Estado inicial | Disparador | Fuente de verdad | Carreras | Resultado esperado | Comportamiento actual | Brecha/Riesgo |
|---|-----------|---------------|------------|------------------|----------|-------------------|----------------------|---------------|
| 1 | Mensaje para contacto sin asignación, existe AgentInstance predeterminada | Contacto sin `ref_user_assigned`; AgentInstance activa en workspace | `newMessageReceived` | CRM (Contacto) | Ninguna en recepción; routing usa `contact_assignments` de ai-service (proyección) | Si existe AgentInstance predeterminada activa y disponible, se confirma la asignación antes de enviar al runtime; si no existe, contacto queda para atención humana | `BotRouting.policy.ts` usa `contactAssignmentClient.getActive` (proyección ai-service) y `botClient.listByWorkspace`; selecciona primer bot activo sin consultar `isDefault` ni AgentInstance predeterminada en CRM; no hay `ref_agent_assigned` en CRM | El resultado esperado del informe era incorrecto: conforme a P2, si hay AgentInstance predeterminada activa/disponible se confirma asignación antes de enviar al runtime; si no, atención humana. No existe `ref_agent_assigned` ni `assignedType` en `Contacto.model.ts`. El routing a bot por defecto usa ai-service, no CRM. |
| 2 | Mensaje cuando no existe AgentInstance predeterminada activa | Contacto sin asignación; sin bot activo | `newMessageReceived` | CRM | Ninguna | Contacto queda para atención humana | `BotRouting.policy.ts` paso 3: retorna null (sin fallback) | Coherente con el diseño, pero no hay `AgentInstance` en CRM para validar. |
| 3 | Dos mensajes concurrentes, mismo contacto | Contacto sin asignación | Dos `newMessageReceived` concurrentes | CRM | Lectura de `ref_user_assigned` sin bloqueo; ambos pueden ver "sin asignar" | Solo uno asigna; el otro no roba | No hay bloqueo por contacto ni transacción | **Riesgo de doble asignación** (sin mecanismo de serialización) |
| 4 | Dos procesos asignan simultáneamente mismo contacto | Contacto con asignación previa | Dos `assignContact` concurrentes | CRM + ai-service (`contact_assignments`) | Carrera entre GET y POST en `convergerHttp` | Last-write-wins; uno gana | `convergerHttp` maneja 409 con reassign; `ejecutarOutboxAssignmentSync` tiene guarda de supersede | Mitigado parcialmente (409 + supersede), pero no hay bloqueo pesimista ni transacción |
| 5 | Reasignación mientras ejecución activa | Contacto asignado a Bot A; ejecución de Bot A en curso | Reasignación a Bot B o humano | CRM + ai-service | La ejecución de Bot A continúa generando | Se cancela ejecución de Bot A; se invalida resultado | No se encontró mecanismo que cancele de inmediato la ejecución en ai-service al reasignar: `sincronizarAsignacionPreCrm` solo sincroniza `contact_assignments` vía HTTP, sin invocar `Execution.cancel()`. Al retornar el proveedor, `runPipelineLoop` procesa `toolCalls` y ejecuta capabilities sin revalidar `context.cancelled` ni la vigencia; después, G2 puede detectar que la ejecución perdió vigencia y cancelarla lógicamente. `publishResult` publica `execution.completed` aunque la ejecución esté cancelada | **Brecha crítica:** tras la reasignación no hay cancelación inmediata CRM → ai-service; al recibir la respuesta del proveedor, `runPipelineLoop` puede ejecutar capabilities antes de que G2 revalide la vigencia (posibles efectos tras la reasignación) y solo después G2 puede cancelar lógicamente la ejecución. La cancelación física sigue siendo best-effort. Anomalía semántica/operativa verificada: `publishResult` publica `execution.completed` aunque cancelado; G3 descarta los eventos cuyo estado no sea `COMPLETED`, por lo que la publicación no demuestra por sí sola un efecto externo |
| 6 | Takeover humano mientras LLM procesa | Contacto asignado a bot; LLM en curso | Usuario asigna contacto a humano | CRM (`ref_user_assigned`) + ai-service (`contact_assignments`) | Carrera entre write CRM y sync HTTP a ai-service | IA detiene intervención; ejecuciones pendientes canceladas; resultados inválidos | `sincronizarAsignacionPreCrm` (HTTP antes de CRM) solo sincroniza `contact_assignments`; no se encontró un mecanismo que envíe una orden de cancelación inmediata desde CRM hacia ai-service al producirse el takeover. Al retornar el proveedor, `runPipelineLoop` ejecuta capabilities sin revalidar `context.cancelled` ni la vigencia; después, G2 puede detectar `human_takeover` y cancelar lógicamente. `publishResult` publica evento aunque cancelado | **Brecha crítica:** durante el takeover la ejecución no se cancela de inmediato (no se encontró mecanismo CRM → ai-service); al recibir la respuesta del proveedor, `runPipelineLoop` puede ejecutar capabilities antes de que G2 revalide la vigencia, con posibles efectos posteriores al takeover; solo después G2 puede cancelar lógicamente. La cancelación física sigue siendo best-effort. `publishResult` publica `execution.completed` aunque cancelado (anomalía semántica/operativa); G3 descarta los eventos cuyo estado no sea `COMPLETED`, por lo que la publicación no demuestra por sí sola un envío externo |
| 7 | Cancelación solicitada cuando LLM terminó, resultado no aplicado | Ejecución COMPLETED en ai-service, resultado en memoria | Cancelación / takeover | ai-service (`contact_assignments`) | Ninguna en el camino CRM→ai-service | Resultado no se aplica; no hay efecto externo | G2 revalida antes de `markCompleted` (ya completado, no aplica). G3 descarta si `human_takeover` antes de persistir EvMensaje | G2 no se ejecuta si ya está COMPLETED. Depende de G3 para descartar. `publishResult` ya publicó el evento Rabbit. |
| 8 | Resultado de ejecución antigua después de nueva asignación | Ejecución antigua completada; contacto reasignado | `execution.completed` Rabbit → CRM | CRM (Contacto) + ai-service (`contact_assignments`) | Retraso de Outbox: `contact_assignments` puede no reflejar la reasignación | Resultado descartado (G3 detecta `assignment_changed`/`human_takeover`) | G3 valida contra `contact_assignments` de ai-service. Si no convergente, puede pasar | **Riesgo:** si Outbox retrasado, `contact_assignments` viejo permite el resultado obsoleto |
| 9 | ABA: A → B → A | Contacto asignado a A; reasignado a B; reasignado a A | Ejecución antigua de A (bajo E1) llega después de E3 | CRM + ai-service | Ninguna protección sin identidad de transición | Ejecución antigua rechazada (E1 ≠ E3 vigente) | No hay `evAssignedId` ni `EvAssigned`. No hay forma de distinguir E1 de E3 | **Brecha crítica:** ABA no resistido. Bloqueante ARQ-05 |
| 10 | Indisponibilidad de MongoDB de ai-service/proyección durante validación G3/G4 | Ejecución completada en ai-service | `execution.completed` → CRM consumer | ai-service (`contact_assignments`) | G3 falla si Mongo de ai-service no responde | No enviar con estado desconocido; reintento diferido | G3: `VigenciaNoDisponibleError` → re-republicación diferida con backoff. Nunca envía con estado desconocido | No es una caída de CRM durante la validación: los gates no consultan CRM directamente, sino la proyección `contact_assignments` en `AI_SERVICE_MONGO_DB`. Coherente con fail-closed. Pero `contact_assignments` es proyección de ai-service, no CRM. |
| 11 | Ausencia de transacción entre write de CRM y Outbox compensatorio | Operación de asignación | Write CRM + create Outbox (compensatorio) | CRM | Proceso cae entre write CRM y create Outbox | Diferenciar write de CRM y creación eventual de Outbox compensatorio; no hay transacción confirmada | Asignar: HTTP a ai-service primero, CRM después si HTTP OK; si falla HTTP, Outbox de compensación y CRM no se escribe. Liberar: CRM primero, HTTP después; si falla HTTP, Outbox de compensación. Si proceso cae entre write CRM y create Outbox, divergencia sin recuperación hasta que poller detecte | **Riesgo:** no hay transacción; si Outbox no se crea, no hay recuperación automática. No es un fallo posterior a transacción confirmada (dicha transacción no existe). |
| 12 | Mensaje duplicado / reentregado por Rabbit | Mensaje ya procesado | Rabbit redelivery | ai-service (execution.completed dedup) | Ninguna si dedup funciona | Mensaje procesado una vez | Dedup por `executionId` con Redis SET NX (L280-303). Fail-open si Redis caído | Coherente para execution.completed. No hay dedup explícito para mensajes de entrada (`message-received`) |
| 13 | Rechazo de mensaje sin requeue en `MessageReceivedHandler` | Mensaje en cola `ai-service.message-received` | Consumer falla mid-procesamiento | RabbitMQ (cola durable) | Ninguna | Mensaje se rechaza sin requeue explícito | `channel.nack(msg, false, false)` (L174) es un rechazo explícito sin requeue. Cola durable, pero código inspeccionado no declara argumentos DLX; no se afirma que exista una DLX configurada porque no fue verificada. Distinguir rechazo explícito sin requeue de caída del proceso/conexión con mensaje todavía sin ACK: si el proceso o la conexión cae mientras la entrega sigue sin ACK, RabbitMQ normalmente puede reencolarla para entregarla nuevamente. La redelivery se relaciona con el riesgo de crear otra ejecución con un `executionId` diferente cuando no hay deduplicación persistente por `originMessageId`; no se presenta la redelivery como una pérdida automática del mensaje. |
| 14 | Transferencias A → B → C → A | Contacto asignado a A; cadena de transferencia | Transferencias sucesivas | CRM (Contacto) | Ninguna protección de ciclo | Contacto queda `unassigned`; cadena termina | No hay detección de ciclos de transferencia en código revisado | **Riesgo:** no hay mecanismo de detección de ciclos A→B→C→A |
| 15 | Reintento del mismo mensaje entrante después de cadena terminada | Contacto `unassigned`; cadena terminada | Nuevo mensaje o reintento | CRM | Ninguna | Nuevo mensaje inicia evaluación nueva; no se reinicia cadena terminada | `newMessageReceived` evalúa contacto actual. No hay deduplicación persistente por identidad del mensaje de origen | **Riesgo:** `EvAssigned` identifica una transición confirmada de asignación; su identidad permite verificar qué transición autorizó una ejecución y resolver ABA. `originMessageId` identifica el mensaje de origen y sirve para trazabilidad e idempotencia de ese mensaje. No son identificadores intercambiables. El problema de deduplicación de mensajes de origen es la falta de una clave persistente adecuada basada en `originMessageId`; la ausencia de `evAssignedId` no es, por sí sola, explicación para distinguir reintentos de nuevos mensajes. |

### 11.1 Escenarios críticos sin cobertura

- **Escenario 5, 6 (riesgo crítico ARQ-07):** se produce un takeover o una reasignación en CRM mientras el proveedor LLM todavía procesa la solicitud y no se encontró un mecanismo que envíe inmediatamente una orden de cancelación desde CRM hacia ai-service; cuando el proveedor responde, `runPipelineLoop` puede procesar `toolCalls` y ejecutar capabilities antes de que G2 vuelva a comprobar la vigencia de la asignación; solo después, G2 puede detectar la pérdida de vigencia y cancelar la ejecución lógicamente. Esto puede permitir efectos posteriores a un takeover o reasignación. La cancelación física de una solicitud ya enviada al proveedor sigue siendo best-effort. Distinguir: `publishResult` publica `execution.completed` incluso cuando la ejecución está cancelada (anomalía semántica/operativa verificada); G3 descarta los eventos cuyo estado no sea `COMPLETED`; por tanto, la publicación del evento cancelado no demuestra por sí sola que se produzca un envío externo.
- **Escenario 7:** `publishResult` publica el evento Rabbit aunque la ejecución esté cancelada, pero G3 descarta los eventos cuyo estado no sea `COMPLETED` antes de persistir; la publicación no demuestra por sí sola un efecto externo.
- **Escenario 8:** Validación contra proyección `contact_assignments` (eventual) no contra CRM. Si Outbox retrasado, resultado obsoleto pasa.
- **Escenario 9:** ABA no resistido (sin `evAssignedId`).
- **Escenario 11:** No hay transacción; si Outbox compensatorio no se crea entre el write de CRM y el create, divergencia permanente hasta que el poller detecte (o para siempre si no se creó). No es un fallo posterior a transacción confirmada.
- **Escenario 14:** No hay detección de ciclos de transferencia.
- **Escenario 15:** `EvAssigned` identifica la transición de asignación; `originMessageId` identifica el mensaje de origen para trazabilidad e idempotencia. No son intercambiables. El problema de deduplicación de mensajes de origen es la falta de una clave persistente basada en `originMessageId`; no se afirma que la ausencia de `evAssignedId` impida distinguir reintentos de nuevos mensajes.

---

## 12. Contradicciones detectadas

| # | Contradicción | Evidencia | Impacto | ARQ relacionado |
|---|--------------|-----------|---------|-----------------|
| 1 | Canónico D.1.5 prevé `assignedType`/`ref_agent_assigned` en Contacto; código solo tiene `ref_user_assigned` | `Contacto.model.ts` L157 vs canónico D.1.5 L712-736 | Alto | ARQ-05 |
| 2 | Canónico D.1.6 prevé `EvAssigned` discriminator de Evento; código no tiene discriminador | `Evento.model.ts` L1-61 vs canónico D.1.6 L744-766 | Alto | ARQ-05 |
| 3 | Canónico D.1.8 prevé transacción Contacto+EvAssigned+Outbox; código no tiene sesión de transacción | `assignmentSync.service.ts` (Outbox.create fuera de transacción) vs canónico D.1.8 L789 | Crítico | ARQ-06 |
| 4 | Canónico E.4.11 prevé intentar la cancelación física de LLM cuando sea posible (aborto físico best-effort, según las capacidades del proveedor) e invalidar los resultados obsoletos rechazando todos sus efectos tardíos, incluida la ejecución de capabilities; el código solo tiene `cancel()` lógico y `runPipelineLoop` no revalida `context.cancelled` ni la vigencia antes de ejecutar capabilities (`publishResult` publica aunque cancelado) | `execution.ts` L230 vs `execution-engine.ts` L166-234 vs canónico E.4.11 L1177-1198 | Alto | ARQ-07 |
| 5 | Canónico D.1.12/G3-G4 prevé validación contra CRM (autoridad); código valida contra `contact_assignments` de ai-service (proyección) | `aiVigencia.repo.ts` vs canónico D.1.12 L1202-1211 | Medio | ARQ-08 |
| 6 | Canónico D.1.6 menciona `operationType` como propuesta pendiente; no está en el modelo ni aprobado | `Evento.model.ts` vs canónico D.1.6 L754, L766 | Medio | ARQ-09 |
| 7 | `publishResult` publica `execution.completed` aunque ejecución cancelada por vigencia; el evento viaja por Rabbit aunque no deba tener efecto | `execution-engine.ts` L202-234 | Alto | ARQ-07 (brecha de invalidación) |

---

## 13. Bloqueantes y diferibles, con dependencias concretas

### 13.1 Bloqueantes para aprobación de WAB-78 (D3)

| ID | Por qué bloquea | Dependencia concreta |
|----|----------------|---------------------|
| ARQ-05 | Sin identidad de EvAssigned no hay resolución ABA ni identificación de transición que autorizó ejecución | Definición de modelo `EvAssigned` (discriminator de Evento) + campo de referencia en Contacto. Decisión técnica de diseño (no de producto): nombre del campo (`ref_ev_assigned` es propuesto, no aprobado) y semántica de actualización. No introducir `assignmentVersion`. |
| ARQ-06 | Sin transacción Contacto+EvAssigned+Outbox pueden quedar estados parciales. El mecanismo actual es recuperación compensatoria, no Transactional Outbox | Implementar transacción MongoDB. Verificar soporte en Atlas (pendiente, D6). No afirmar compatibilidad sin evidencia del deployment real. Solución técnica objetivo pendiente de implementación y aceptación. |
| ARQ-07 | Sin invalidación garantizada de efectos tardíos (incluida ejecución de capabilities en `runPipelineLoop`), ejecución obsoleta puede producir efectos. La cancelación física de LLM no se exige como absoluto si el proveedor no lo permite | Definir contrato de cancelación e invalidación: `runPipelineLoop` debe revalidar `context.cancelled` antes de capabilities; `publishResult` no debe publicar para ejecuciones canceladas. WAB-81. |
| ARQ-08 | Sin `evAssignedId` no hay validación contra identidad de transición de EvAssigned; G3/G4 validan contra proyección `contact_assignments`, no contra CRM directamente | Definir `evAssignedId` como metadata de ejecución. Propagarlo a través del flujo. Validar contra CRM antes de cada efecto externo. `contact_assignments` se conserva únicamente para describir su función operativa y el problema actual; no se acepta como alternativa a CRM. |
| ARQ-22 | Entrega durable e idempotencia del flujo asíncrono de ejecución no garantizadas: el publisher absorbe fallos sin publish confirms ni `drain`; el engine ack del input sin garantía de entrega del evento de resultado; no hay dedup persistente por identidad del mensaje de origen (`originMessageId`); reentrega puede generar ejecución duplicada con otro `executionId` | Implementar publish confirms / entrega durable; dedup persistente por `originMessageId`; no ack del input sin garantía de entrega del resultado. Verificar relación con decisión aprobada de que reintentos no generen respuestas duplicadas. |
| ARQ-13 (D1, referenciado) | `isDefault` sin unicidad ni mecanismo de reemplazo | Resolución de P2 §4.1 (pendiente de producto). Prerrequisito para routing por defecto. |
| ARQ-14 (D1, referenciado) | Ownership de persistencia no declarado para entidades del modelo objetivo | Completar frontera de persistencia en D.1 (CRM vs ai-service). |

### 13.2 Diferibles (no bloquean D3, pero requieren atención)

| ID | Motivo | Destino |
|----|--------|---------|
| ARQ-09 | `operationType` es propuesta no aprobada | WAB-79 |
| ARQ-15 | Índices de AgentInstance sin definir | WAB-79 |
| ARQ-10 | Policy Engine y output filter sin implementar | WAB-81 |
| ARQ-04 (H1/H4/H5/H7) | Hallazgos de permisos diferibles | WAB-80 |
| Detección de ciclos de transferencia (escenario 14) | No hay mecanismo en código | WAB-79/WAB-80 |
| Cancelación física de LLM (parte de ARQ-07) | No hay AbortController ni abort signal | WAB-81 |

**Siguiente ID disponible: ARQ-23** (ARQ-22 consumido por el nuevo hallazgo de entrega e idempotencia asíncrona).

---

## 14. Decisiones que continúan pendientes del usuario

**Decisiones de producto pendientes (requieren definición del usuario, registradas en la línea base o en la matriz):**

- **P2 §4.1 (`isDefault`):** unicidad y mecanismo de reemplazo de la AgentInstance predeterminada (ARQ-13). Ya registrado en el registro de decisiones; no se reabre.
- **P3 (matriz de permisos / actor de ciclo de vida de AgentInstance):** quién ejecuta cada operación de ciclo de vida (ARQ-19, ARQ-20). Ya registrado en el registro de decisiones; no se reabre.

**Decisiones técnicas (no son decisiones de producto pendientes del usuario):**

- **Identidad de EvAssigned vigente (ARQ-05):** referencia directa en Contacto vs. consulta del último evento; nombre del campo (`ref_ev_assigned` es propuesto, no aprobado) y semántica de actualización. Decisión técnica de diseño, no de producto. No introducir `assignmentVersion`.
- **Transacción MongoDB (ARQ-06):** viabilidad de transacción para Contacto+EvAssigned+Outbox en el deployment real de Atlas. Verificación de infraestructura pendiente de D6; no se afirma compatibilidad sin evidencia del deployment real.
- **Cancelación e invalidación de efectos tardíos (ARQ-07):** contrato de invalidación; `runPipelineLoop` revalida `context.cancelled` antes de capabilities; `publishResult` no publica para ejecuciones canceladas. Contrato técnico pendiente de especificación. La cancelación física de LLM no se exige como absoluto si el proveedor no lo permite.
- **`evAssignedId` (ARQ-08):** definición como campo en `ExecutionContext` y `Execution`; propagación por Rabbit. Campo y metadata pendientes de especificación técnica.
- **`operationType` (ARQ-09):** se deferra a WAB-79 (no aprobado en D3). No aprobar `operationType`.

**Pendientes técnicos de arquitectura/documentación (no son decisiones de producto pendientes del usuario):**

- **Frontera de persistencia por entidad (ARQ-14):** clasificado en la matriz como **Pendiente técnico**. Consiste en completar la frontera arquitectónica y documental indicando qué servicio persiste cada entidad, diferenciando superficie de API, ownership de negocio y persistencia. No se pide al usuario decidir individualmente el servicio persistente de cada entidad durante esta revisión; su clasificación en la matriz no cambia.

Estos ítems técnicos quedan como recomendaciones técnicas, contratos técnicos pendientes de especificación, verificaciones de infraestructura y tareas de diseño/implementación asignables a los bloques correspondientes. No se pide al usuario que decida individualmente cada campo o mecanismo interno.

---

## 15. Riesgos para la aprobación de WAB-78

1. **Aprobar D3 sin EvAssigned implementado** dejaría ABA sin resolver y sin forma de identificar qué transición autorizó una ejecución.
2. **Aprobar D3 sin transacción Contacto+EvAssigned+Outbox** permitiría estados parciales (asignación sin historial, o sin mecanismo de propagación recuperable). El mecanismo actual es recuperación compensatoria, no Transactional Outbox.
3. **Aprobar D3 sin invalidación garantizada de efectos tardíos** (incluida ejecución de capabilities en `runPipelineLoop`) permitiría que una ejecución obsoleta genere resultados tras takeover o reasignación. La cancelación física de LLM no se exige como absoluto si el proveedor no lo permite.
4. **Validar G3/G4 contra `contact_assignments` (proyección) en lugar de CRM** introduce una ventana de inconsistencia donde el Outbox está retrasado.
5. **Ejecutar capabilities antes de revalidar la vigencia tras un takeover o reasignación:** se produce un takeover o una reasignación en CRM mientras el proveedor LLM todavía procesa la solicitud y no se encontró un mecanismo que envíe inmediatamente una orden de cancelación desde CRM hacia ai-service; cuando el proveedor responde, `runPipelineLoop` puede procesar `toolCalls` y ejecutar capabilities antes de que G2 vuelva a comprobar la vigencia de la asignación, con posibles efectos posteriores al takeover o reasignación; solo después, G2 puede detectar la pérdida de vigencia y cancelar la ejecución lógicamente. La cancelación física de una solicitud ya enviada al proveedor sigue siendo best-effort. Anomalía semántica/operativa asociada y verificada: `publishResult` publica `execution.completed` aunque la ejecución esté cancelada; G3 descarta los eventos cuyo estado no sea `COMPLETED`, por lo que no se ha demostrado un efecto externo derivado exclusivamente de esa publicación.
6. **No afirmar compatibilidad con Atlas sin evidencia del deployment real:** ARQ-06 queda como No verificado hasta D6.
7. **ARQ-22:** entrega durable e idempotencia del flujo asíncrono no garantizadas (publisher sin publish confirms ni `drain`, engine ack sin garantía de entrega del evento de resultado, sin dedup por `originMessageId`, reentrega puede crear ejecución duplicada con otro `executionId`). Brecha de confiabilidad verificada en código; impacto productivo no reproducido.

---

## 16. Qué debe leer la siguiente sesión

1. Este informe (`informe-bloque-3.md`) completo.
2. `matriz-acumulativa.md` actualizada (ver sección 17).
3. `informe-bloque-2.md` (D2) y `informe-bloque-1.md` (D1) para contexto de hallazgos previos.
4. Canónico `crm/docs/plans/wab-75-fase-2-diseno.md` D.1.5-D.1.8 y E.4.8-E.4.12 (referencia de decisiones aprobadas).
5. Código clave: `Contacto.model.ts`, `Evento.model.ts`, `Outbox.model.ts`, `assignmentSync.service.ts`, `execution-engine.ts`, `execution.ts`, `executionCompleted.consumer.ts`, `aiVigencia.repo.ts`.
6. Pruebas clave: `sendMessage.assign.test.ts`, `assignmentSync.test.ts`, `outboxRetry.worker.test.ts`, `vigencia.service.test.ts`, `vigencia-g4.test.ts`, `execution-engine.test.ts`.
7. Plan de revisión §4 (preguntas Bloque 3), §6 (reutilización), §8 (formato hallazgos), §10 (restricciones).
8. Registro de decisiones 03/11/2026 (línea base de decisiones P1, P2, P4, P9, P10).

---

## 17. Archivos revisados, creados y modificados

### Archivos revisados (solo lectura, sin modificaciones)

- `crm/src/models/chat/Contacto.model.ts`
- `crm/src/models/events/Evento.model.ts`
- `crm/src/models/events/Outbox.model.ts`
- `crm/src/services/Contacto.service.ts` (partes: newMessageReceived, updateContactoByID, asUserToContact)
- `crm/src/services/AiContactAssignment.service.ts`
- `crm/src/graphql/resolvers/AiContactAssignment.resolver.ts`
- `crm/src/graphql/resolvers/Contacto.resolver.ts` (partes: sendMessage, assignUserToContact)
- `crm/src/infrastructure/ai-service/assignmentSync.service.ts`
- `crm/src/infrastructure/ai-service/clients/contact-assignment.client.ts`
- `crm/src/infrastructure/ai-service/clients/bot.client.ts`
- `crm/src/infrastructure/ai-service/dto/ai-service-requests.ts`
- `crm/src/infrastructure/ai-service/aiVigencia.repo.ts`
- `crm/src/stream/Outbox.stream.ts`
- `crm/src/Rabbit/workers/outbox.worker.ts`
- `crm/src/Workers/outboxRetry.worker.ts`
- `crm/src/Rabbit/consumers/executionCompleted.consumer.ts`
- `crm/src/Rabbit/queues.enum.ts`
- `crm/src/services/BotRouting/BotRouting.policy.ts`
- `crm/microservices/ai-service/src/domain/entities/execution.ts`
- `crm/microservices/ai-service/src/domain/entities/contact-assignment.ts`
- `crm/microservices/ai-service/src/domain/value-objects/execution/execution-context.ts`
- `crm/microservices/ai-service/src/domain/ports/services/bot-vigencia.ts`
- `crm/microservices/ai-service/src/application/services/execution.service.ts`
- `crm/microservices/ai-service/src/application/services/vigencia.service.ts`
- `crm/microservices/ai-service/src/application/services/process-message.service.ts`
- `crm/microservices/ai-service/src/engine/execution-engine.ts`
- `crm/microservices/ai-service/src/engine/pipeline.ts`
- `crm/microservices/ai-service/src/engine/steps/load-configuration.step.ts`
- `crm/microservices/ai-service/src/engine/steps/invoke-provider.step.ts`
- `crm/microservices/ai-service/src/engine/steps/execute-rag.step.ts`
- `crm/microservices/ai-service/src/infrastructure/messaging/consumers/message-received-handler.ts`
- `crm/microservices/ai-service/src/infrastructure/messaging/publishers/rabbitmq-event-publisher.ts`
- `crm/microservices/ai-service/src/infrastructure/messaging/publishers/rabbitmq.mapper.ts`
- `crm/microservices/ai-service/src/infrastructure/messaging/rabbitmq.ts`
- `crm/microservices/ai-service/src/infrastructure/crm/client/crm-graphql-client.ts`
- `crm/microservices/ai-service/src/infrastructure/composition-root.ts`
- `crm/microservices/ai-service/src/domain/events/event-types.ts`
- `crm/microservices/ai-service/src/domain/runtime/bot-runtime.ts`
- `crm/microservices/ai-service/src/infrastructure/runtime/bot-runtime-resolver.ts`
- `crm/microservices/ai-service/src/domain/entities/bot-instance.ts`
- `crm/docs/plans/wab-75-fase-2-diseno.md` (secciones D.1, D.1.5-D.1.8, E.4.8-E.4.12)
- `crm/docs/plans/wab-75-agentes-administrados-por-waby.md` (referencia I3)
- `crm/docs/reports/AUDITORIA_PERMISOS_WABY.md` (referencia)

### Pruebas inspeccionadas (sin modificar)

- `crm/src/services/__tests__/sendMessage.assign.test.ts`
- `crm/src/infrastructure/ai-service/__tests__/assignmentSync.test.ts`
- `crm/src/Workers/__tests__/outboxRetry.worker.test.ts`
- `crm/microservices/ai-service/tests/unit/application/services/vigencia.service.test.ts`
- `crm/microservices/ai-service/tests/unit/engine/execution-engine.test.ts`
- `crm/microservices/ai-service/tests/unit/domain/entities/execution.test.ts`
- `crm/microservices/ai-service/tests/unit/domain/entities/contact-assignment.test.ts`
- `crm/microservices/ai-service/tests/unit/application/use-cases/contact-assignment.use-cases.test.ts`
- `crm/microservices/Difusion/src/Worker/__tests__/vigencia-g4.test.ts`

### Archivos creados

- `F:\ElemenSoft\docs\revision-wab-78\informe-bloque-3.md` (este informe)

### Archivos modificados

- `F:\ElemenSoft\docs\revision-wab-78\matriz-acumulativa.md` (ver siguiente sección)

### Símbolos, clases, métodos inspeccionados

`Contacto.ref_user_assigned`, `ContactoModel.findByIdAndUpdate`, `Evento`, `EventoModel`, `OutboxModel`, `Outbox.status` (PENDING/SENT/FAILED), `sincronizarAsignacionPreCrm`, `sincronizarLiberacionPostCrm`, `ejecutarOutboxAssignmentSync`, `convergerHttp`, `encolarOutboxPendiente`, `handleOutboxEvent`, `outboxRetryTick`, `backoffMsPorIntentos`, `initOutboxStream`, `executionCompletedConsumer`, `evaluarVigenciaBot`, `VigenciaService.check`, `Execution.markRunning`, `Execution.markCompleted`, `Execution.markFailed`, `Execution.cancel`, `ExecutionEngine.process`, `ExecutionEngine.publishResult`, `ProcessMessageService.executeWithContext`, `MessageReceivedHandler`, `VigenciaNoDisponibleError`, `ContactAssignment.AssigneeType.USER/BOT`, `ContactAssignment.release()`, `AssignContactUseCase`, `ReassignContactUseCase`, `UnassignContactUseCase`, `BotRuntimeResolver.resolve`, `LoadConfigurationStep`, `InvokeProviderStep`, `ExecuteRAGStep`, `rabbitmq.mapper.ts`, `crm-graphql-client.ts`.

---

## 18. Hallazgos nuevos

Se confirma un problema distinto de ARQ-05 a ARQ-09, verificado en el código local (HEAD `b9e0aee`, coincidente con el commit consultado de GitHub `master`):

### ARQ-22 — Entrega durable e idempotencia del flujo asíncrono de ejecución no garantizadas.

**Severidad:** Alto. **Bloquea WAB-78:** Sí. Brecha de confiabilidad verificada en código; impacto productivo no reproducido.

**Evidencia verificada en `crm/microservices/ai-service/`:**

- `RabbitMQEventPublisher` (`src/infrastructure/messaging/publishers/rabbitmq-event-publisher.ts` L21-87):
  - Si no hay canal (`getChannel()` retorna null), registra advertencia y retorna **sin publicar**.
  - Si publicar falla, captura la excepción en `catch` y **no la propaga** (never-throws).
  - No utiliza publish confirms ni mecanismo durable de reintento.
  - Si `channel.publish` retorna `false`, registra backpressure pero **no espera `drain`**.
- `ExecutionEngine` (`src/engine/execution-engine.ts` L57-129):
  - Espera `eventPublisher.publish(envelope)` en `publishResult` (L222).
  - El publisher absorbe fallos, de modo que el engine puede completar normalmente aunque el evento no se haya entregado.
- `MessageReceivedHandler` (`src/infrastructure/messaging/consumers/message-received-handler.ts` L87-175):
  - Hace `channel.ack(msg)` cuando `executeWithContext` resuelve (L164).
  - Puede confirmarse el input aunque no exista garantía de entrega del evento de resultado.
- Idempotencia del mensaje entrante:
  - Usa `msg.properties.messageId` para trazabilidad (L92), pero **no hay deduplicación persistente por identidad del mensaje de origen**.
  - La ejecución crea un `executionId` nuevo mediante `Date.now()` + aleatoriedad (`buildContext`, L226-229).
  - Si Rabbit redeliveriza un delivery que quedó unacked por caída del proceso/conexión, puede crearse **otra ejecución con otro `executionId`**.
  - La deduplicación G3 por `executionId` (`executionCompleted.consumer.ts` L280-303) **no detecta** que ambas ejecuciones proceden del mismo mensaje original.
  - El control Redis de G3 es fail-open si Redis falla.

**Relación con la decisión aprobada:** evaluar la relación con la decisión de que los reintentos no generen respuestas duplicadas.

**Diferenciación con ARQ-06:**
- ARQ-06: atomicidad de la transición CRM Contacto + EvAssigned + Outbox (recuperación compensatoria, sin transacción).
- ARQ-22: confiabilidad e idempotencia del transporte de ejecución/resultados ai-service ↔ CRM (publisher, ack, dedup por mensaje de origen).
No mezclar ambos hallazgos.

**Clasificación:** Alto / bloqueante. Brecha de confiabilidad verificada en código; impacto productivo no reproducido. La justificación indica que el diseño no garantiza la entrega del resultado antes de confirmar el mensaje de entrada ni evita duplicaciones ante reentregas. Que `channel.publish()` devuelva `false` es una señal de backpressure, no prueba por sí sola de que el mensaje se haya perdido; el problema es la combinación de ausencia de confirmación de publicación, absorción de errores, confirmación del mensaje de entrada y falta de idempotencia persistente por mensaje de origen.

**Siguiente ID disponible: ARQ-23.**

---

## 19. Confirmaciones de restricciones

- **No se modificó código:** Ningún archivo de `crm/src`, `crm/microservices` o documentos canónicos fue modificado.
- **No se modificaron tests:** Ningún archivo de prueba fue creado, modificado o eliminado.
- **No se modificó configuración ni variables de entorno.**
- **No se modificaron documentos canónicos de WAB-75.**
- **No se modificó el registro de decisiones de producto.**
- **No se crearon, modificaron ni cerraron issues de Linear.**
- **No se iniciaron WAB-79, D4, D5, D6 o D7.**
- **No se cambiaron decisiones aprobadas de P1/P2/P4/P9/P10.**
- **No se eliminó RabbitMQ, workers, Outbox ni procesamiento asíncrono.**
- **No se introdujo `assignmentVersion`.**
- **No se aprobó `operationType`.**
- **No se convirtió `contact_assignments` en autoridad de asignación.**
- **No se afirmó compatibilidad con Atlas sin evidencia del deployment real.**
- **No se emitieron operaciones que alteren datos persistentes o el estado del sistema.**
- **No se emitió un GO/NO-GO global** (corresponde al cierre de todos los bloques + integración).

---

## 20. Cuestiones no verificadas

1. **Compatibilidad de transacciones MongoDB con Atlas:** No se pudo verificar. Requiere acceso al deployment real (D6).
2. **Cancelación física de llamadas LLM:** No se confirmó si los providers de ai-service soportan AbortController/signal. No se inspeccionó `src/infrastructure/providers/` en profundidad.
3. **Comportamiento real de producción de Outbox bajo carga:** No se verificó empíricamente.
4. **Valor real de `CRM_API_KEY` en producción:** No se conoce si es un JWT válido para el contexto GraphQL de CRM (ARQ-21 de D2, referenciado).
5. **Comportamiento del consumer de Rabbit ante fallos intermitentes bajo carga:** No se verificó empíricamente.

---

## 21. Dependencias que quedan para D6 u otros bloques

| Dependencia | Bloque | Descripción |
|-------------|--------|-------------|
| Verificación de transacciones MongoDB en Atlas | D6 | ARQ-06: confirmar si el deployment real soporta transacciones en las colecciones involucradas |
| Contrato de cancelación física de LLM | WAB-81 | ARQ-07: definir AbortController/signal al provider, comportamiento si no soporta aborto |
| Definición de modelo `EvAssigned` y campo en Contacto | WAB-79/WAB-80 | ARQ-05: definir `EvAssigned` como discriminator, campo de referencia en Contacto |
| Definición de `evAssignedId` y propagación | WAB-79/WAB-80 | ARQ-08: campo en ExecutionContext, Execution, mensajes Rabbit |
| Índices de AgentInstance | WAB-79 | ARQ-15: tabla de índices por entidad |
| Resolución P2 §4.1 (`isDefault`) | Usuario/producto | ARQ-13: unicidad y mecanismo de reemplazo |
| Frontera de persistencia por entidad | WAB-78/WAB-79 | ARQ-14: completar D.1 con servicio que persiste por entidad |
