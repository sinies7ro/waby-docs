# WAB-81 ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â ImplementaciÃƒÆ’Ã‚Â³n local del AgentInstance Runtime (2026-11-20)

**Estado:** implementaciÃƒÆ’Ã‚Â³n local en progreso; no integrada ni desplegada.  
**Repositorios:** `ElemeSoft/crm` (cÃƒÆ’Ã‚Â³digo) y `sinies7ro/waby-docs` (documentaciÃƒÆ’Ã‚Â³n).  
**No se realizaron commits ni pushes.**

## Incremento implementado

Se conectÃƒÆ’Ã‚Â³ el flujo de ejecuciÃƒÆ’Ã‚Â³n AgentInstance a travÃƒÆ’Ã‚Â©s de los siguientes componentes:

1. `AgentRuntimeContextComposer`: compone instrucciones con orden de autoridad, reglas/skills y fragmentos RAG explÃƒÆ’Ã‚Â­citamente marcados como datos no confiables; filtra historial a roles user/assistant y limita a 20 mensajes previos.
2. `AgentInstanceLLMInvoker`: resuelve `LLMRoute.providerAccountId` mediante `LLMCredentialResolver`, obtiene el adapter por `ProviderManager.resolve(provider, apiKey)`, invoca y normaliza resultado/mÃƒÆ’Ã‚Â©tricas. El request soporta temperature, maxTokens, topP, frequencyPenalty y presencePenalty.
3. `AgentInstanceExecutionRuntime`: valida identidad del snapshot contra la ejecuciÃƒÆ’Ã‚Â³n durable, valida input, compone contexto e invoca. Mapea resultado y mÃƒÆ’Ã‚Â©tricas al contrato de `AgentExecutionRuntime`.
4. Composition root: construye el runtime solo si estÃƒÆ’Ã‚Â¡ configurada la llave independiente del Credential Registry `AI_SERVICE_LLM_CREDENTIAL_ENCRYPTION_KEY`; si falta, el runtime no se habilita (fail-closed) y se emite warning.
5. `index.ts`: conecta `AgentExecutionInboxWorker` con repositorios Mongo y programa el procesamiento periÃƒÆ’Ã‚Â³dico de Inbox; el timer se cancela en graceful shutdown.

## ValidaciÃƒÆ’Ã‚Â³n

- `pnpm --dir microservices/ai-service test -- --runInBand` con 4 suites focalizadas: **4 suites, 14 tests aprobados**.
- `pnpm --dir microservices/ai-service type-check`: **exit code 0**.
- La advertencia existente de pnpm sobre `pnpm.onlyBuiltDependencies` no afecta TypeScript/tests.

## LÃƒÆ’Ã‚Â­mites conocidos / no declarar E2E cerrado

- No se ejecutÃƒÆ’Ã‚Â³ una prueba end-to-end contra MongoDB, RabbitMQ, CRM ni proveedor LLM real.
- El runtime compone snapshot e input durable; retrieval real de Knowledge/RAG aÃƒÆ’Ã‚Âºn no estÃƒÆ’Ã‚Â¡ conectado a esta nueva ruta. El composer admite fragmentos, pero runtime pasa el conjunto vacÃƒÆ’Ã‚Â­o actualmente.
- Skills se incluyen como configuraciÃƒÆ’Ã‚Â³n textual de contexto, no se ejecutan como capabilities/tools en este bloque.
- AbortSignal permite cancelar antes y despuÃƒÆ’Ã‚Â©s de la invocaciÃƒÆ’Ã‚Â³n; los adapters no implementan cancelaciÃƒÆ’Ã‚Â³n HTTP en-flight todavÃƒÆ’Ã‚Â­a.
- Se requiere configurar la llave de Credential Registry para habilitar el worker AgentExecution. No existe fallback a credenciales de environment ni Bot legacy.
- La fuente autoritativa y validaciÃƒÆ’Ã‚Â³n de policy/snapshot en Producer/CRM sigue siendo un punto contractual de WAB-81; este runtime valida identidad, shape y status ACTIVE, pero no sustituye la autorizaciÃƒÆ’Ã‚Â³n de ingreso.

## PrÃƒÆ’Ã‚Â³ximo bloque

- Verificar wiring y ciclo de vida del Inbox worker en una integraciÃƒÆ’Ã‚Â³n local.
- Conectar retrieval de Knowledge con filtros tenant/Workspace/AgentInstance y grounding estricto.
- Definir/implementar capabilities autorizadas para AgentInstance, sin reusar el executor legacy Bot.
- AÃƒÆ’Ã‚Â±adir cancelaciÃƒÆ’Ã‚Â³n HTTP real de adapters, si el contrato de proveedor/SDK lo permite.
- E2E Admission ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ Inbox ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ Runtime ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ Provider ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ durable result ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ Result Outbox.
- Reconciliar contratos CRM Producer/snapshot/policy y actualizar WAB-81 con evidencia.

### ValidaciÃƒÆ’Ã‚Â³n adicional del snapshot (20/11/2026)

El contrato AI Service recalcula SHA-256 sobre `JSON.stringify({ agentDefinition, agentInstance, policy })` y compara con `identity.effectiveConfigHash`; mismatch genera `AGENT_RUNTIME_SNAPSHOT_HASH_MISMATCH`. Se aÃƒÆ’Ã‚Â±adiÃƒÆ’Ã‚Â³ prueba especÃƒÆ’Ã‚Â­fica para hash alterado. Esta implementaciÃƒÆ’Ã‚Â³n sigue el hash producido por el endpoint de snapshots actual; canonicalizaciÃƒÆ’Ã‚Â³n/versionado entre CRM y AI Service permanece sujeto a reconciliaciÃƒÆ’Ã‚Â³n contractual.

## Incremento: retrieval de Knowledge conectado al AgentInstance Runtime (20/11/2026)

Se implementÃƒÆ’Ã‚Â³ `AgentInstanceKnowledgeRetriever` y se conectÃƒÆ’Ã‚Â³ al runtime antes de componer el contexto/invocar al proveedor.

### Comportamiento y controles

- La ejecuciÃƒÆ’Ã‚Â³n exige Knowledge Base IDs configurados; si no hay IDs, falla con `AGENT_KNOWLEDGE_NOT_CONFIGURED`.
- Cada KB se resuelve mediante `findById(id, workspaceId)`. No encontrada o de otro Workspace: `AGENT_KNOWLEDGE_SCOPE_MISMATCH`; inactiva: `AGENT_KNOWLEDGE_BASE_INACTIVE`.
- Verifica provider/model/dimension de embedding contra el proveedor activo para evitar vector spaces incompatibles.
- La bÃƒÆ’Ã‚Âºsqueda utiliza `workspaceId` y colecciÃƒÆ’Ã‚Â³n `knowledgeBaseId`; ademÃƒÆ’Ã‚Â¡s valida el payload retornado contra `workspaceId` y `knowledgeBaseId`, descartando resultados que no coincidan.
- Respeta threshold/topK de KB (fallback de config), filtra fragmentos sin contenido y aplica presupuesto de tokens.
- Si retrieval falla, no se oculta el error; si no hay fragmentos vÃƒÆ’Ã‚Â¡lidos tras filtros/presupuesto, falla con `AGENT_KNOWLEDGE_GROUNDING_UNAVAILABLE`. No hay fallback general/internet ni fail-open.
- Los fragmentos pasan al composer como `Workspace knowledge (untrusted reference data)` y no adquieren autoridad de instrucciones.

### Evidencia

- Test suite del runtime + retriever: **2 suites, 10 tests aprobados**.
- Type-check AI Service: `pnpm --dir microservices/ai-service type-check`, exit code 0.
- Cobertura retriever: scope Workspace/KB, KB ausente, KB inactiva, resultados vacÃƒÆ’Ã‚Â­os, error de vector provider, ausencia de Knowledge configurado.

### LÃƒÆ’Ã‚Â­mites contractuales

- El aislamiento fÃƒÆ’Ã‚Â­sico Qdrant es por Workspace y colecciÃƒÆ’Ã‚Â³n de Knowledge Base; se valida el Workspace/KB en payload. El esquema de KB/ingesta no expone organizationId en metadatos de chunks, por lo que la garantÃƒÆ’Ã‚Â­a explÃƒÆ’Ã‚Â­cita OrganizationÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢Workspace debe reconciliarse con el contrato CRM y tenancy de Workspace; no se inventa un filtro Organization que el corpus no posee.
- No es aÃƒÆ’Ã‚Âºn prueba E2E con Qdrant/Mongo/Rabbit/CRM/LLM real. No se validÃƒÆ’Ã‚Â³ que todas las ingestas histÃƒÆ’Ã‚Â³ricas incluyan los metadatos Workspace/KB esperados.
- El grounding estricto en esta fase significa fail-closed ante retrieval vacÃƒÆ’Ã‚Â­o/error y contexto limitado a fuentes del Workspace/KB; un verificador semÃƒÆ’Ã‚Â¡ntico determinista de que cada afirmaciÃƒÆ’Ã‚Â³n de salida estÃƒÆ’Ã‚Â© respaldada queda pendiente.
- Skills siguen siendo contexto textual, no herramientas/capabilities ejecutables.

### Siguiente bloque

1. Reconciliar metadata y tenant contract CRMÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬ÂAI Service para Organization/Workspace/AgentInstance Knowledge.
2. Validar corpus real e ingesta Qdrant, filtros, vector-space identity y fuente documental.
3. Definir polÃƒÆ’Ã‚Â­tica de respuesta cuando las fuentes recuperadas no respaldan la pregunta (verificador/abstenciÃƒÆ’Ã‚Â³n).
4. E2E AdmissionÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢InboxÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢RuntimeÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢KnowledgeÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ProviderÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢durable resultÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢Result Outbox.

### ValidaciÃƒÆ’Ã‚Â³n final ampliada

Se ejecutaron 5 suites focalizadas en runtime, retriever, invoker, composer y credential resolver: **5 suites / 20 tests aprobados**. Type-check posterior a la optimizaciÃƒÆ’Ã‚Â³n (embedding de query una vez por ejecuciÃƒÆ’Ã‚Â³n) terminÃƒÆ’Ã‚Â³ con **exit code 0**.

## ReconciliaciÃƒÆ’Ã‚Â³n del contrato de metadata y tenancy (incremento siguiente)

### Hallazgos comprobados en el cÃƒÆ’Ã‚Â³digo local

- `Works` en CRM guarda la Organization propietaria en `ref_organizacion`; `AgentRuntimeSnapshotService` resuelve el Workspace y verifica que `ref_organizacion` coincida con `organizationId` antes de obtener el AgentInstance activo y solicitar snapshot.
- `AgentInstance` tiene `workspaceId` y `knowledgeBaseIds`; no mantiene un `organizationId` propio, consistente con Workspace como ÃƒÆ’Ã‚Â¡mbito de recursos.
- AI Service `KnowledgeBase` y `Document` guardan `workspaceId` y `knowledgeBaseId`, sin `organizationId`.
- IngestionContext transporta `workspaceId`, `knowledgeBaseId` y `documentId`.
- `VectorPersistenceStep` persiste content, chunkIndex, totalChunks, tokenCount, documentId y knowledgeBaseId. `QdrantVectorStore.upsert` agrega workspaceId y collection. No se persiste organizationId.
- Qdrant collection name deriva `{workspaceId}_{knowledgeBaseId}`; los filtros de metadata se traducen a must. El aislamiento actual de retrieval se basa en namespace/colecciÃƒÆ’Ã‚Â³n + payload.

### Hardening aplicado

`AgentInstanceKnowledgeRetriever` ahora impone el filtro Qdrant de metadata `workspaceId == execution.workspaceId` y `knowledgeBaseId == kb.id` en todas las bÃƒÆ’Ã‚Âºsquedas. Los filtros configurados por KB se conservan, pero no pueden reemplazar las condiciones obligatorias de tenancy/KB. Se mantiene la validaciÃƒÆ’Ã‚Â³n posterior de payload. Test valida filtro obligatorio.

### DecisiÃƒÆ’Ã‚Â³n/contrato pendiente

No se aÃƒÆ’Ã‚Â±adiÃƒÆ’Ã‚Â³ organizationId arbitrario al payload de vector ni al IngestionContext: AI Service no tiene una fuente autoritativa local para resolver WorkspaceÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢Organization. La validaciÃƒÆ’Ã‚Â³n autoritativa actualmente reside en CRM. Si se decide persistir organizationId en metadata, deberÃƒÆ’Ã‚Â¡ derivarse de un contrato autenticado CRMÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢AI Service (no de un campo libre del request), y definir migraciÃƒÆ’Ã‚Â³n/reindexaciÃƒÆ’Ã‚Â³n de vectores existentes. No es necesario duplicar la relaciÃƒÆ’Ã‚Â³n Organization en AgentInstance si Workspace es el recurso dueÃƒÆ’Ã‚Â±o y CRM valida su ownership.

### Pruebas y alcance

No se ejecutan ni se solicitan E2E en este bloque. La validaciÃƒÆ’Ã‚Â³n se limita a unit tests focalizados y type-check. La integraciÃƒÆ’Ã‚Â³n real queda fuera del alcance actual, como se acordÃƒÆ’Ã‚Â³.

## PolÃƒÆ’Ã‚Â­tica de grounding y abstenciÃƒÆ’Ã‚Â³n (incremento local)

Se ampliÃƒÆ’Ã‚Â³ `AgentRuntimeContextComposer` con una instrucciÃƒÆ’Ã‚Â³n explÃƒÆ’Ã‚Â­cita para la ruta con Knowledge:

- Responder preguntas factuales del Workspace ÃƒÆ’Ã‚Âºnicamente con fragmentos recuperados.
- No utilizar conocimiento general del modelo, supuestos o informaciÃƒÆ’Ã‚Â³n externa para completar vacÃƒÆ’Ã‚Â­os.
- Si las fuentes no respaldan directamente una respuesta, abstenerse y utilizar: `No encuentro informaciÃƒÆ’Ã‚Â³n suficiente en la documentaciÃƒÆ’Ã‚Â³n disponible para responder con certeza.`
- No inventar, extrapolar ni afirmar hechos sin respaldo.
- Los fragmentos continÃƒÆ’Ã‚Âºan siendo datos no confiables, nunca instrucciones.

Esta es una polÃƒÆ’Ã‚Â­tica de prompt y no una verificaciÃƒÆ’Ã‚Â³n semÃƒÆ’Ã‚Â¡ntica determinista de la salida. No se declara garantÃƒÆ’Ã‚Â­a absoluta contra hallucination; la validaciÃƒÆ’Ã‚Â³n determinista de claims/sources permanece pendiente.

Test del composer valida la presencia de las instrucciones de grounding/abstenciÃƒÆ’Ã‚Â³n. No se ejecutan pruebas E2E, conforme a alcance acordado.

## AuditorÃƒÆ’Ã‚Â­a de lifecycle Knowledge/RAG y hardening (2026-11-20)

### Hallazgos

- DeleteDocument y DeleteKnowledgeBase ya purgaban Qdrant antes de borrar Mongo, pero las llamadas Qdrant no esperaban confirmaciÃƒÆ’Ã‚Â³n de aplicaciÃƒÆ’Ã‚Â³n (`wait=true` ausente). Una respuesta HTTP podÃƒÆ’Ã‚Â­a preceder la aplicaciÃƒÆ’Ã‚Â³n real de la purga.
- `VectorPersistenceStep` capturaba errores de upsert por chunk y devolvÃƒÆ’Ã‚Â­a un resultado con fallos; el pipeline podÃƒÆ’Ã‚Â­a concluir `success=true`, permitiendo que el documento pasara a `COMPLETED` con indexaciÃƒÆ’Ã‚Â³n parcial.
- El retriever no consultaba el estado de Document. Vectores de documentos `PROCESSING`, `FAILED` o eliminados (por purga parcial/consistencia eventual) podÃƒÆ’Ã‚Â­an entrar al contexto si seguÃƒÆ’Ã‚Â­an presentes.
- Reingesta V2 ya purga vectores anteriores antes de iniciar el pipeline. Si la nueva ingesta falla, el documento pasa a FAILED; la recuperaciÃƒÆ’Ã‚Â³n posterior requiere reingesta y vuelve a purgar por documentId.
- KB deletion purga colecciÃƒÆ’Ã‚Â³n antes del cascade Mongo; fallos Mongo pueden dejar KB/documentos parcialmente borrados, pero colecciÃƒÆ’Ã‚Â³n ya purgada. El retriever valida KB activa y documentos completados, mitigando exposiciÃƒÆ’Ã‚Â³n de vectores residuales.

### Correcciones locales

1. `QdrantVectorStore`: upsert, delete puntual, deleteByFilter y deleteCollection ahora solicitan `wait=true`, haciendo que la operaciÃƒÆ’Ã‚Â³n espere la aplicaciÃƒÆ’Ã‚Â³n de la mutaciÃƒÆ’Ã‚Â³n antes de devolver control.
2. `VectorPersistenceStep`: si cualquier chunk falla, arroja error agregado; el pipeline no puede reportar ÃƒÆ’Ã‚Â©xito parcial. Los vectores parcialmente escritos permanecen temporalmente, pero el documento no pasa a COMPLETED y el retriever los excluye; la siguiente reingesta realiza purga previa.
3. `AgentInstanceKnowledgeRetriever`: recibe `IDocumentRepository`, obtiene documentos de la KB/Workspace y acepta ÃƒÆ’Ã‚Âºnicamente chunks cuyo `documentId` corresponde a un documento `COMPLETED`. Payload sin documentId, de documentos desconocidos, eliminados o con status distinto se descarta. Si no hay documentos completados, fail-closed con `AGENT_KNOWLEDGE_GROUNDING_UNAVAILABLE`.
4. Composition root conecta `MongoDocumentRepository` al retriever.
5. Tests actualizados y agregados para status de documentos y ausencia de documentos completados.

### ValidaciÃƒÆ’Ã‚Â³n

- Type-check AI Service: `pnpm --dir microservices/ai-service type-check`, exit code 0.
- 5 suites focalizadas (runtime, retriever, composer, invoker, credential resolver): **5 suites / 22 tests aprobados**.
- No E2E; no commit, push ni deploy.

### Limitaciones residuales

- Los deletes de Qdrant ahora usan `wait=true`, pero no existe transacciÃƒÆ’Ã‚Â³n distribuida Mongo/Qdrant ni compensaciÃƒÆ’Ã‚Â³n durable. Si Mongo falla tras purgar vectors, puede quedar metadata sin corpus; la ejecuciÃƒÆ’Ã‚Â³n falla cerrada al no recuperar chunks.
- No se implementa un tombstone/soft-delete durable ni outbox de lifecycle para reintentar automÃƒÆ’Ã‚Â¡ticamente una purga si el provider queda inaccesible. DeleteUseCases rechazan el borrado Mongo si la purga vectorial falla, conservando el registro para retry manual.
- Reingesta no es atomic swap: purga corpus previo antes de generar el nuevo; durante el proceso el documento estÃƒÆ’Ã‚Â¡ PROCESSING y no es recuperable. Esto prioriza no servir corpus viejo sobre disponibilidad.
- Se requiere validar en integraciÃƒÆ’Ã‚Â³n la semÃƒÆ’Ã‚Â¡ntica `wait=true` contra la versiÃƒÆ’Ã‚Â³n real de Qdrant.
- Sin E2E por decisiÃƒÆ’Ã‚Â³n de alcance.

## AuditorÃƒÆ’Ã‚Â­a AgentInstance ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬Â Knowledge Base (20/11/2026)

### ReconciliaciÃƒÆ’Ã‚Â³n documental

Se revisaron `design/configuracion-efectiva-y-capacidades.md` y `design/fase-2-diseno-objetivo-y-contratos.md`, ademÃƒÆ’Ã‚Â¡s de WAB-81 description/comments en Linear. El contrato canÃƒÆ’Ã‚Â³nico ya define Knowledge como recurso propiedad del Workspace y selecciÃƒÆ’Ã‚Â³n/asignaciÃƒÆ’Ã‚Â³n como configuraciÃƒÆ’Ã‚Â³n permitida de AgentInstance por usuarios autorizados del Workspace. No se encontrÃƒÆ’Ã‚Â³ una decisiÃƒÆ’Ã‚Â³n que requiera nueva entidad o cambio de ownership; no se reabre diseÃƒÆ’Ã‚Â±o.

### Hallazgos fÃƒÆ’Ã‚Â­sicos

- CRM `AgentInstanceService.create()`/`updatePartial()` guardaban `knowledgeBaseIds` sin resolver existencia, Workspace propietario ni estado de KB.
- La validaciÃƒÆ’Ã‚Â³n de ownership GraphQL de KB existe para operaciones directas, pero no se aplica a referencias guardadas desde AgentInstance service.
- `AgentRuntimeSnapshotService` copia referencias del instance snapshot sin verificar existencia/status de KB. AI Service retriever las valida al ejecutar y falla cerrado, por lo que no se permite fallback cross-workspace, pero el snapshot/config podÃƒÆ’Ã‚Â­a persistir referencias invÃƒÆ’Ã‚Â¡lidas.
- `KnowledgeBaseService.delete()` ejecutaba cascada AI Service y limpieza de media, pero no retiraba la KB de `AgentInstance.knowledgeBaseIds` del Workspace. Referencias huÃƒÆ’Ã‚Â©rfanas permanecÃƒÆ’Ã‚Â­an en CRM y causaban fallo cerrado en runtime.

### Correcciones locales

1. `findAiServiceKnowledgeBase` ahora incluye status en la proyecciÃƒÆ’Ã‚Â³n de ownership.
2. `AgentInstanceService` valida IDs de KB al crear y actualizar: ID string no vacÃƒÆ’Ã‚Â­o, normalizaciÃƒÆ’Ã‚Â³n trim, deduplicaciÃƒÆ’Ã‚Â³n, existencia, Workspace propietario y estado active. Errors: `KNOWLEDGE_BASE_IDS_INVALID`, `KNOWLEDGE_BASE_SCOPE_MISMATCH`, `KNOWLEDGE_BASE_INACTIVE`.
3. `KnowledgeBaseService.delete()` elimina referencias a la KB en `AgentInstanceModel` del Workspace (`$pull`) solo despuÃƒÆ’Ã‚Â©s de que AI Service confirma delete/cascade. Fallo de cleanup se propaga y se registra como inconsistencia; runtime sigue fail-closed ante referencia residual.
4. Tests cubren referencia cross-Workspace, KB inactiva, normalizaciÃƒÆ’Ã‚Â³n/deduplicaciÃƒÆ’Ã‚Â³n, actualizaciÃƒÆ’Ã‚Â³n con referencia invÃƒÆ’Ã‚Â¡lida y cleanup de referencias posterior a delete.
5. Type-check CRM encontrÃƒÆ’Ã‚Â³ mismatch existente en `AgentInstanceAssignmentWriterService` (model `_id` typed string vs `toHexString`); corregido a `String(evAssigned._id)` como ajuste mÃƒÆ’Ã‚Â­nimo para permitir validaciÃƒÆ’Ã‚Â³n.

### ValidaciÃƒÆ’Ã‚Â³n

- Unit tests CRM AgentInstanceService + KnowledgeBaseService: 2 suites, 36 tests PASS en la ejecuciÃƒÆ’Ã‚Â³n previa a los 2 tests de update adicionales; se estÃƒÆ’Ã‚Â¡ revalidando el conjunto ampliado.
- CRM `pnpm type-check`: validaciÃƒÆ’Ã‚Â³n en curso tras correcciÃƒÆ’Ã‚Â³n del mismatch.
- AI Service 5 suites / 22 tests y type-check de bloque RAG/lifecycle previo siguen PASS.
- E2E excluido; no commit/push/deploy.

### LÃƒÆ’Ã‚Â­mites residuales

- Mongo CRM y AI Service comparten cluster pero delete KB y cleanup de referencias no estÃƒÆ’Ã‚Â¡n en una transacciÃƒÆ’Ã‚Â³n coordinada: el cleanup posterior puede fallar tras KB borrada; la ejecuciÃƒÆ’Ã‚Â³n se rechaza cerrada. No existe outbox durable de cleanup.
- Existe carrera entre validar una KB al guardar `knowledgeBaseIds` y eliminarla en AI Service. Runtime retriever revalida KB activa/Workspace antes de usarla, por lo que el riesgo es configuraciÃƒÆ’Ã‚Â³n stale/fallo, no acceso cross-Workspace. Eliminar esta carrera requerirÃƒÆ’Ã‚Â­a lifecycle contract/transacciÃƒÆ’Ã‚Â³n coordinada o tombstone.
- Desactivar KB deja referencias configuradas; runtime falla cerradamente si se ejecuta con KB inactiva. No se decidiÃƒÆ’Ã‚Â³ auto-desasignaciÃƒÆ’Ã‚Â³n al desactivar.
- No se modifica canonical design: las reglas existentes ya cubren ownership/selecciÃƒÆ’Ã‚Â³n. Se registra auditorÃƒÆ’Ã‚Â­a fÃƒÆ’Ã‚Â­sica en WAB-81.

**ValidaciÃƒÆ’Ã‚Â³n ampliada final (AgentInstance ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬Â KB):** CRM `pnpm type-check` PASS (exit 0); AgentInstanceService + KnowledgeBaseService **2 suites / 311 tests PASS**. Incluye create/update ownership y estado de KB, normalizaciÃƒÆ’Ã‚Â³n/deduplicaciÃƒÆ’Ã‚Â³n y limpieza de referencias en delete. Ajustar count de la secciÃƒÆ’Ã‚Â³n previa a 38 tests.

## AuditorÃƒÆ’Ã‚Â­a adicional: snapshot efectivo, admisiÃƒÆ’Ã‚Â³n, vigencia y retries

### Contrato observado

- CRM resuelve `AgentRuntimeSnapshotService` antes de cruzar a AI Service: valida Workspace/Organization, AgentInstance active, copia AgentDefinition prompt/revision y LLMRoute vigente, rules/skills enabled ordenadas, `knowledgeBaseIds` y policy suministrada por el resolver autorizado.
- AI Service `/internal/agent-runtime/snapshots` toma AgentDefinition y LLMRoute actuales de Mongo, captura valores (prompt, revision, route id/version/account/provider/model/params), instance config y policy, y calcula `effectiveConfigHash` sobre definition + instance + policy.
- Admission persiste `effectiveConfigSnapshot` en `AgentExecution` dentro de la transacciÃƒÆ’Ã‚Â³n que crea Inbox y asigna generation/sequence. Un duplicate accepted message exige identity, payloadHash y snapshot exactamente iguales; snapshot distinto para misma identidad produce conflict, no sobrescribe la ejecuciÃƒÆ’Ã‚Â³n.
- Worker lee el snapshot persistido; runtime valida identidad/hash y usa valores snapshot. No vuelve a consultar AgentDefinition, AgentInstance, Rules, Skills ni LLMRoute para reconstruir el prompt/config.
- Retries del Inbox reutilizan el mismo `AgentExecution`/snapshot; `markRetry` modifica Inbox status/error/lease, no el snapshot. Attempt 10 agota y terminaliza failed. ReclamaciÃƒÆ’Ã‚Â³n por lease expirado puede reintentar despuÃƒÆ’Ã‚Â©s de una ejecuciÃƒÆ’Ã‚Â³n parcial.

### QuÃƒÆ’Ã‚Â© queda congelado vs quÃƒÆ’Ã‚Â© se resuelve en runtime

**Congelado en el snapshot por ejecuciÃƒÆ’Ã‚Â³n:** system prompt y revision de AgentDefinition; route id/version, provider, model, params y providerAccountId (referencia, no secreto); additionalPrompt; KB IDs; rules/skills/config; policy; identity/snapshotId/capturedAt/hash. `input` y envelope de admisiÃƒÆ’Ã‚Â³n tambiÃƒÆ’Ã‚Â©n quedan almacenados en la ejecuciÃƒÆ’Ã‚Â³n/inbox.

**ResoluciÃƒÆ’Ã‚Â³n dinÃƒÆ’Ã‚Â¡mica en cada attempt:**
- Credential resolver consulta provider-account registry por providerAccountId y exige account `active`, provider coincidente, y resuelve el secreto actual. RevocaciÃƒÆ’Ã‚Â³n/rotaciÃƒÆ’Ã‚Â³n/estado de cuenta afecta un retry de la misma ejecuciÃƒÆ’Ã‚Â³n.
- Knowledge retriever consulta KB/documents en Mongo y Qdrant al momento del attempt. KB inexistente, inactive, provider/model/dimension incompatible, documentos no COMPLETED, vectors faltantes o sin resultados grounding hacen fallar cerrado. El contenido/vector index no estÃƒÆ’Ã‚Â¡ snapshotteado.
- Provider adapter/servicio externo/model runtime se ejecuta en cada attempt; no existe replay determinista del output del provider.

### Vigencia y lifecycle

- Snapshot endpoint exige AgentDefinition active/no retired y route existente, pero no valida el estado real de la AgentInstance: recibe `status: active` desde CRM y lo incluye. La autoridad de CRM valida active antes de resolver snapshot/admitir.
- DespuÃƒÆ’Ã‚Â©s de admission, el worker no revalida AgentInstance active ni assignment/contact ownership antes de ejecutar; consume el execution snapshot admitido. Esto es consistente con drain/retirement: admission se bloquea y executions pendientes se drenan, pero pause/reassignment posterior no cancela por sÃƒÆ’Ã‚Â­ solo una ejecuciÃƒÆ’Ã‚Â³n admitida.
- Snapshot no tiene expiry en AI Service. El TTL del assignment snapshot es independiente y no afecta a `AgentExecution.effectiveConfigSnapshot` ya persistido.
- LLMRoute `version` se captura/hash pero no se usa para recuperar versiÃƒÆ’Ã‚Â³n histÃƒÆ’Ã‚Â³rica; la ejecuciÃƒÆ’Ã‚Â³n utiliza los valores congelados. ProviderAccount credential/status sÃƒÆ’Ã‚Â­ se consulta live por referencia.

### Hallazgos / riesgos

1. **Snapshot config reproducible, ejecuciÃƒÆ’Ã‚Â³n no totalmente reproducible:** los valores de prompts/rules/skills/policy/route estÃƒÆ’Ã‚Â¡n congelados, pero KB contents/index y credential state/provider behavior son live. Reintento puede usar datos KB/credential distintos y generar respuesta diferente bajo mismo effectiveConfigHash. Hash identifica configuraciÃƒÆ’Ã‚Â³n declarativa, no todos los inputs de ejecuciÃƒÆ’Ã‚Â³n.
2. **KB deactivated/deleted despuÃƒÆ’Ã‚Â©s de admission:** retry/runtime falla cerrado (inactive/missing), no reemplaza snapshot ni desasigna automÃƒÆ’Ã‚Â¡ticamente. Un attempt que ya recuperÃƒÆ’Ã‚Â³ fragments puede continuar si lifecycle cambia despuÃƒÆ’Ã‚Â©s de retrieval; no hay lock/version fence entre KB lifecycle y provider invocation.
3. **AgentInstance pause despuÃƒÆ’Ã‚Â©s de admission:** la ejecuciÃƒÆ’Ã‚Â³n admitted sigue siendo eligible y puede correr; no existe cancellation/fencing por instance lifecycle aparte del retirement admission/drain. Debe considerarse polÃƒÆ’Ã‚Â­tica de pausa, no asumir que snapshot active significa status actual.
4. **Provider account revoke/rotate:** credential state live; retries fallan si revoked y usan nueva secret si rotation mantiene account active. Snapshot hash no cambia; no se incluye credential version/fingerprint (correcto no exponer secretos, pero falta identificador de versiÃƒÆ’Ã‚Â³n auditable).
5. **Route version semantics:** route version se captura, pero no hay resoluciÃƒÆ’Ã‚Â³n histÃƒÆ’Ã‚Â³rica; la semÃƒÆ’Ã‚Â¡ntica efectiva es snapshot de valores actuales, no pin/replay por route version.
6. **Hash contract:** CRM y AI Service calculan hash de `JSON.stringify` sobre definition/instance/policy. Orden estable proviene de orden explÃƒÆ’Ã‚Â­cito de arrays en CRM y construcciÃƒÆ’Ã‚Â³n del objeto en route; contrato no usa canonical JSON. Hash se valida al ejecutar, pero un producer distinto/serialization con diferente orden puede producir mismatch legÃƒÆ’Ã‚Â­timo.
7. **Admission retry:** duplicate admission con misma identidad exige snapshot byte-semantic-equivalent segÃƒÆ’Ã‚Âºn canonical JSON del repository; una nueva resoluciÃƒÆ’Ã‚Â³n despuÃƒÆ’Ã‚Â©s de cambio de config no actualiza la execution y genera conflict, fail closed. El caller debe conservar/reusar el snapshot original para retry idempotente.

### ConclusiÃƒÆ’Ã‚Â³n

La implementaciÃƒÆ’Ã‚Â³n **sÃƒÆ’Ã‚Â­ congela el effective config en el punto de admission y reutiliza el snapshot en retries**, evitando adoptar cambios posteriores de prompt/rules/skills/policy/route config. La frontera de snapshot no incluye los recursos de Knowledge ni la credencial/provider runtime: ÃƒÆ’Ã‚Â©stos se resuelven live y pueden variar entre attempts. El contrato actual prioriza fail-closed y lifecycle drain, no replay determinista ni cancelaciÃƒÆ’Ã‚Â³n automÃƒÆ’Ã‚Â¡tica por pause/deactivate posterior a admission.

### Recomendaciones acotadas (no implementadas en este bloque)

- Definir explÃƒÆ’Ã‚Â­citamente si `pause`/deactivate debe bloquear solo nuevas admissions o tambiÃƒÆ’Ã‚Â©n cancelar executions admitted; no asumir equivalencia con retirement.
- Si se requiere replay/auditorÃƒÆ’Ã‚Â­a reproducible, registrar KB/document/vector revision o retrieval evidence IDs + hashes, credential version identifier (sin secreto), y provider/model response metadata; no congelar secretos.
- Mantener retry idempotente con el snapshot original; no volver a resolver config para una execution ya admitted.
- Considerar canonical JSON hash compartido como contrato explÃƒÆ’Ã‚Â­cito si productores adicionales construyen snapshots.
- No implementar transacciones/locking de KB lifecycle con provider call en este bloque: requiere decisiÃƒÆ’Ã‚Â³n de producto y contract separado.

**Alcance:** auditorÃƒÆ’Ã‚Â­a read-only; no cambios de cÃƒÆ’Ã‚Â³digo, E2E excluido, sin commit/push/deploy.

## AuditorÃƒÆ’Ã‚Â­a adicional: admission, ejecuciÃƒÆ’Ã‚Â³n durable, drain y runtime real

### Alcance cruzado

Se revisaron el contrato fÃƒÆ’Ã‚Â­sico WAB-79 RetirementOperation v3.10, WAB-78 diseÃƒÆ’Ã‚Â±o objetivo/contratos, WAB-81 runtime, Linear WAB-81/WAB-79/WAB-95, admission repository, execution/inbox models, worker, runtime AgentInstance, invoker, ExecutionEngine legacy, assignment inbox y result outbox.

### Admission ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â componentes fÃƒÆ’Ã‚Â­sicos vs flujo integrado

**Componentes existentes:** `MongoAgentExecutionAdmissionRepository`, `AgentExecution`, `AgentExecutionInbox`, `AgentInstanceExecutionControl`, `AgentExecutionInboxWorker` y `ExecutionResultOutbox`.

`admit()` realiza en una transacciÃƒÆ’Ã‚Â³n Mongo:
- duplicate check por originMessageId/executionId;
- verifica identity + payloadHash + snapshot en duplicados;
- CAS al control de instancia `admissionState=open` y `$inc admissionSequence`;
- captura `admissionGeneration`/sequence;
- crea `AgentExecution` y `AgentExecutionInbox`.

**Gap de integraciÃƒÆ’Ã‚Â³n:** no se encontrÃƒÆ’Ã‚Â³ call-site funcional de `MongoAgentExecutionAdmissionRepository.admit()` en AI Service. El `AgentAssignmentInboxWorker` procesa assignment projection y mensajes retirement (block/drain/finalize), no admission de mensajes de ejecuciÃƒÆ’Ã‚Â³n. El worker durable de AgentExecution estÃƒÆ’Ã‚Â¡ bootstrappeado/polling, pero no hay entrada integrada que cree los registros Inbox/Execution desde la solicitud real. La admisiÃƒÆ’Ã‚Â³n fÃƒÆ’Ã‚Â­sica es scaffold disponible, no un flujo end-to-end funcional conectado.

### Admission lock y drain

- `admit()` y `block()` escriben el mismo `AgentInstanceExecutionControl` dentro de transacciones; bloqueado/missing fail-closed. La secuencia de admission y el generation token se capturan en Execution e Inbox.
- `block()` es idempotente para el mismo retirementOperationId y confirma timestamp/generation.
- `countNonTerminal()` valida que el control estÃƒÆ’Ã‚Â¡ bloqueado por esa operaciÃƒÆ’Ã‚Â³n y el `blockedAt` exacto; cuenta `AgentExecution.status=pending`.
- El worker no cambia execution status al iniciar; `startedAt` se setea, status sigue pending hasta terminal. Por eso el count incluye activas y queued y permite drain.
- Terminal execution + Outbox + Inbox processed se escribe en transacciÃƒÆ’Ã‚Â³n, con fencing del lease Inbox.

### Hallazgos de lifecycle / fencing

1. **Dead-letter puede dejar execution no terminal:** las ramas de inbox/execution missing e identity mismatch invocan `markRetry()` y no terminalizan AgentExecution. Al attempt 10, Inbox puede pasar a `dead_letter`, mientras execution permanece `pending`; `countNonTerminal()` continÃƒÆ’Ã‚Âºa contÃƒÆ’Ã‚Â¡ndola y RetirementOperation drain no puede confirmar cero. En error de runtime normal, el worker sÃƒÆ’Ã‚Â­ intenta `executions.fail()` al agotar 10.
2. **`markRetry()` no verifica resultado CAS:** si lease owner/generation/expiry ya no coincide, `updateOne` no se valida; la rama puede asumir retry/dead-letter sin confirmar que actualizÃƒÆ’Ã‚Â³. Fencing evita la escritura stale, pero el caller no obtiene conflicto explÃƒÆ’Ã‚Â­cito.
3. **`markStarted()` no estÃƒÆ’Ã‚Â¡ lease-fenced:** la escritura de `startedAt` usa solo executionId/status/startedAt; no recibe token de lease Inbox. La terminalizaciÃƒÆ’Ã‚Â³n sÃƒÆ’Ã‚Â­ se protege mediante transacciÃƒÆ’Ã‚Â³n + lease generation/owner/expiry en Inbox. Riesgo de metadata de start en worker stale, no de terminalizar stale.
4. **Lease perdido abort cooperativo pero no provider request:** worker renueva lease y aborta `AbortController` si pierde lease. Runtime/invoker checks signal antes/despuÃƒÆ’Ã‚Â©s, pero `provider.invoke(AIProviderRequest)` no recibe `AbortSignal`; no hay cancelaciÃƒÆ’Ã‚Â³n fÃƒÆ’Ã‚Â­sica de HTTP/provider en esta ruta. Una llamada externa puede seguir generando costo y luego ser descartada por fencing.
5. **Retry puede ejecutar provider mÃƒÆ’Ã‚Â¡s de una vez:** lease expiry/recovery vuelve a ejecutar la misma AgentExecution; no existe idempotencia del proveedor. Resultado stale no se persiste si lease fencing falla, pero efectos externos/costo no son reversibles.
6. **AdmisiÃƒÆ’Ã‚Â³n duplicate post-block:** la rama duplicate busca inbox existente antes de comprobar control open. ReenvÃƒÆ’Ã‚Â­o de un mensaje ya admitido conserva la ejecuciÃƒÆ’Ã‚Â³n previa incluso cuando admission estÃƒÆ’Ã‚Â¡ bloqueado; no crea nueva sequence/execution. Comportamiento idempotente razonable, requiere preservar snapshot original.
7. **Missing runtime config:** composition root omite AgentExecution worker cuando no estÃƒÆ’Ã‚Â¡ configurada credential-registry encryption key. El servicio loguea warning; no hay readiness gate ni admission caller que garantice rechazo previo. Debe cerrar fail-closed en el punto de admission y readiness, no dejar ejecuciones acumuladas.

### ConexiÃƒÆ’Ã‚Â³n con ExecutionEngine existente

`AgentInstanceExecutionRuntime` estÃƒÆ’Ã‚Â¡ registrado en CompositionRoot e inyectado a `AgentExecutionInboxWorker`, pero **no ejecuta el `ExecutionEngine` existente**. La ruta nueva compone contexto, recupera Knowledge y llama directamente `AgentInstanceLLMInvoker` ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ provider una sola vez. El `ExecutionEngine` legacy sigue en el pipeline de Bot/messageReceivedHandler y su loop de tool/capability calls.

Consecuencias verificadas:
- No hay loop de `tool_calls` en AgentInstanceExecutionRuntime.
- `allowedCapabilities` se conserva en el context, pero invoker no utiliza tool definitions ni capability executor.
- Skills se serializan como instrucciones/config en prompt; no se ejecutan como herramientas.
- No hay puente que adapte el snapshot AgentInstance a `ExecutionContext`/pipeline/ExecutionEngine legacy.

Esto confirma que runtime AgentInstance real estÃƒÆ’Ã‚Â¡ conectado al worker durable y provider, pero no al ExecutionEngine existente. Es gap de integraciÃƒÆ’Ã‚Â³n pendiente del bloque de implementaciÃƒÆ’Ã‚Â³n solicitado; no se intenta adaptar legacy aquÃƒÆ’Ã‚Â­ porque requiere cerrar contratos WAB-81/WAB-78.

### Controles positivos

- `AgentExecutionInbox` unique indexes en originMessageId/executionId.
- claim usa atomic `findOneAndUpdate`, leaseOwner, leaseUntil y leaseGeneration.
- heartbeat renueva solo lease vigente; loss aborta signal.
- `completeWithResultOutbox` valida lease dentro de transaction; aplica terminal effect, crea result outbox y marca inbox processed atÃƒÆ’Ã‚Â³micamente.
- `AgentExecution` terminal update exige status pending y modifiedCount=1.
- result outbox usa lease fencing y messageId estable `executionId:result`; entrega RabbitMQ puede ser at-least-once, no exactly-once.
- admission lock/control usa documento compartido con block CAS; no se encontrÃƒÆ’Ã‚Â³ un path de ingress que lo evada mediante el admission repository (el problema actual es que la ruta de admission no estÃƒÆ’Ã‚Â¡ conectada).

### Prioridad de correcciÃƒÆ’Ã‚Â³n para siguiente implementaciÃƒÆ’Ã‚Â³n

**P0 ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â integraciÃƒÆ’Ã‚Â³n funcional:** conectar request/admission ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ snapshot ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ `admit()` ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ durable Inbox ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ AgentInstance runtime ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ terminal persistence/outbox ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ CRM result handling. El punto de entrada debe fallar cerrado si snapshot, ownership, runtime readiness, assignment/evAssigned vigencia o admission control no validan.

**P1 ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â lifecycle drain:** dead-letter y missing/identity corruption deben tener una polÃƒÆ’Ã‚Â­tica terminal/reconciliation explÃƒÆ’Ã‚Â­cita para que no queden `AgentExecution.pending` huÃƒÆ’Ã‚Â©rfanas bloqueando retirement. No marcar processed falsamente; definir failed/cancelled + audit/reconciliation, y verificar CAS.

**P1 ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â provider cancellation:** pasar AbortSignal a adapter/request y validar soporte en proveedores. Distinguir cancellation efectiva vs invalidaciÃƒÆ’Ã‚Â³n del resultado; stale result no se persiste aunque HTTP no pueda abortarse.

**P1 ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â execution engine integration:** decidir/adaptar AgentInstance snapshot y policy/capabilities al Engine/pipeline existente, sin reutilizar Bot legacy ni restaurar autoridades antiguas. Actualmente el runtime directo no ejecuta tools.

**P2 ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â fencing observability:** comprobar matchedCount de markRetry, hacer markStarted lease-aware, registrar attempt IDs/lease generation y recovery para diagnÃƒÆ’Ã‚Â³sticos.

### Estado

AuditorÃƒÆ’Ã‚Â­a read-only. No se modificÃƒÆ’Ã‚Â³ cÃƒÆ’Ã‚Â³digo ni canonical design. DocumentaciÃƒÆ’Ã‚Â³n local y Linear WAB-81/WAB-95 registran evidencia y gaps. E2E excluido; sin commit/push/deploy.

## AuditorÃƒÆ’Ã‚Â­a adicional: frontera CRM ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ AI Service ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ resultado ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ CRM (read-only)

### Alcance

Se inspeccionÃƒÆ’Ã‚Â³ el ingreso funcional de ejecuciones AgentInstance, la admisiÃƒÆ’Ã‚Â³n durable, el publisher de `execution_result_outbox`, el contrato de envelope producido por `AgentExecutionInboxWorker`, y el consumer CRM `executionCompleted.consumer.ts` / su binding RabbitMQ. Se contrastÃƒÆ’Ã‚Â³ con WAB-81, WAB-79 y WAB-95. Sin cambios de cÃƒÆ’Ã‚Â³digo.

### Hallazgos

**P0 ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â No existe un ingress funcional conectado a `admit()`**. El worker `AgentAssignmentInboxWorker` procesa `agent.assignment.transition`, bloqueos, drain y finalize; no admite ejecuciones. No se encontrÃƒÆ’Ã‚Â³ call-site funcional de `MongoAgentExecutionAdmissionRepository.admit()` fuera de su implementaciÃƒÆ’Ã‚Â³n/tests. El durable execution worker estÃƒÆ’Ã‚Â¡ registrado/polling, pero no hay productor conectado que cree AgentExecution+Inbox desde un evento de canal/CRM. La ejecuciÃƒÆ’Ã‚Â³n nueva sigue siendo scaffolding aislado del camino de mensajes.

**P0 ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â Result Outbox no es compatible con el consumer CRM actual.** El publisher usa `envelope.messageType` como routing key. El envelope del worker genera `agent.execution.completed|failed|cancelled`, mientras CRM sÃƒÆ’Ã‚Â³lo binding `crm.execution.completed` a routing key `execution.completed`; no hay binding para `agent.execution.*`. Por tanto, incluso si se publica, no llega al consumer existente.

**P0 ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â Envelope incompatible con el contrato del consumer legacy.** El consumer espera `message.type === 'execution.completed'`, `payload.status === 'completed'`, `payload.result` string, `payload.botId`, `payload.contactId`, `payload.workspaceId`, `payload.channel`. El envelope durable usa `messageType`, `workspaceId`, `agentInstanceId`, `agentDefinitionId`, `executionId`, `contactId`, `conversationId`, `evAssignedId`; `payload.result` es objeto y no incluye botId/channel. El consumer legacy lo ACKea por tipo/status/result invÃƒÆ’Ã‚Â¡lido, por lo que un adapter superficial de routing no cerrarÃƒÆ’Ã‚Â­a el flujo.

**P0 ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â No hay consumer CRM de resultados AgentInstance.** El consumer actual es el loop legacy BotÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢persistOutgoingÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢dispatchOutgoing y aplica vigencia via `botId`; no valida ownership de AgentInstance/Workspace/Contact/Assignment ni `evAssignedId` contra asignaciÃƒÆ’Ã‚Â³n vigente para este nuevo contrato. No hay adaptador que transforme resultado durable a salida CRM de AgentInstance.

**P1 ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â Outbox garantiza confirmaciÃƒÆ’Ã‚Â³n Rabbit, pero at-least-once.** AI Service usa ConfirmChannel; callback de publish confirma aceptaciÃƒÆ’Ã‚Â³n por broker y `markSent` fenced por lease. Si Rabbit acepta y el worker pierde confirmaciÃƒÆ’Ã‚Â³n o falla `markSent`, el envelope puede republicarse; el messageId estable (`executionId:result`) permite dedup, pero CRM durable consumer no lo implementa para el nuevo contrato. Consumer legacy usa Redis dedup fail-open (TTL 24h), distinto de una dedup persistente vinculada al executionId/assignment.

**P1 ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â Dead-letter del Outbox no tiene reconciliaciÃƒÆ’Ã‚Â³n/alerta visible.** Tras 10 intentos pasa a `dead_letter`; `listPending()` excluye dead_letter. No se observÃƒÆ’Ã‚Â³ un mecanismo operativo que lo reabra, alerte o reconcilie con AgentExecution terminal. Esto puede dejar resultado no entregado de forma permanente.

### Flujo fÃƒÆ’Ã‚Â­sico observado

```mermaid
flowchart TD
  A[CRM / canal] -->|Ingress AgentInstance no conectado| B[admit()]
  B --> C[AgentExecution + Inbox]
  C --> D[Durable worker]
  D --> E[AgentInstanceExecutionRuntime]
  E --> F[Terminal transaction + Result Outbox]
  F --> G[Publisher ai-service.events]
  G -->|agent.execution.completed| H[No binding CRM actual]
  I[Legacy CRM consumer] -->|execution.completed| J[PersistOutgoing + dispatch]
  H -. no conecta .-> I
```

### Compatibilidad de contrato

| Elemento | Result envelope durable | CRM consumer legacy | Compatibilidad |
|---|---|---|---|
| Routing key | `agent.execution.completed` | `execution.completed` | No |
| discriminator | `messageType` | `type` | No |
| status | `payload.status` | `payload.status` | Parcial |
| result | objeto | string | No |
| instance | `agentInstanceId` | `botId` | No |
| channel | no presente | `channel` | No |
| assignment | `evAssignedId` | no contrato | No |
| contact/workspace | presente | presente con nombres legacy | Parcial |

### Prioridad recomendada

1. Definir contrato de resultado AgentInstance y routing key propio, sin reutilizar el envelope legacy Bot.
2. Implementar ingress confiable CRM/canal ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ resolver workspace/contact/assignment/instance ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ snapshot ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ admission; validar que la admisiÃƒÆ’Ã‚Â³n persista identidad causal (`evAssignedId`, originMessageId/correlation) y que pause/retire policy estÃƒÆ’Ã‚Â© definida.
3. Implementar consumer CRM durable para `agent.execution.completed|failed|cancelled`: validar organizaciÃƒÆ’Ã‚Â³n/workspace/instance/contact/conversation y assignment vigente (incluida `evAssignedId`), definir polÃƒÆ’Ã‚Â­tica de resultados stale, dedup persistente por messageId/executionId y ACK sÃƒÆ’Ã‚Â³lo despuÃƒÆ’Ã‚Â©s de persistencia durable del mensaje/result.
4. Integrar con `ContactoSvc.persistOutgoing` y sender ÃƒÆ’Ã‚Âºnico sin fallback de canal/config que permita enviar en otro canal/instance. Revalidar ownership/vigencia antes de dispatch, ademÃƒÆ’Ã‚Â¡s de persistencia.
5. Operabilidad: DLQ/outbox dead_letter alerta, replay/reconciliation y observabilidad por executionId.

### ConclusiÃƒÆ’Ã‚Â³n

El Outbox y la ejecuciÃƒÆ’Ã‚Â³n durable poseen una base fÃƒÆ’Ã‚Â­sica sÃƒÆ’Ã‚Â³lida (transacciÃƒÆ’Ã‚Â³n terminal + outbox, lease fencing, Rabbit publisher confirms), pero la cadena no estÃƒÆ’Ã‚Â¡ integrada funcionalmente: falta ingress de admission y un consumer CRM compatible con el nuevo envelope. No es un fallo aislado de routing; requiere contrato y adaptador de resultado AgentInstance con controles de ownership/assignment. No realizar E2E segÃƒÆ’Ã‚Âºn alcance actual.

## ReconciliaciÃƒÆ’Ã‚Â³n con diseÃƒÆ’Ã‚Â±o existente ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â antes de definir contratos nuevos

RevisiÃƒÆ’Ã‚Â³n posterior a la auditorÃƒÆ’Ã‚Â­a de frontera. Se confirma que **la arquitectura y varios contratos ya estaban diseÃƒÆ’Ã‚Â±ados**; la siguiente implementaciÃƒÆ’Ã‚Â³n debe partir de ellos, no crear un diseÃƒÆ’Ã‚Â±o paralelo.

### Fuentes canÃƒÆ’Ã‚Â³nicas/locales revisadas
- `projects/agents-multitenant/design/fase-2-diseno-objetivo-y-contratos.md`: D.1.9 (ownership, AgentExecution/snapshot, Outboxes/Inboxes); D.6/ARQ-22 (at-least-once, durable-before-ACK, stable identities, dedup, recovery/DLQ); E runtime/routing, assignment precedence y cancellation/result invalidation.
- `projects/agents-multitenant/design/wab-81-agent-execution-envelope-and-snapshot-proposal.md`: Ãƒâ€šÃ‚Â§16 reemplazo de Bot legacy; Ãƒâ€šÃ‚Â§17 Producer ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ Outbox ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ Admission, envelope propuesto, Admission/control, snapshot, secuencia; Ãƒâ€šÃ‚Â§18ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Å“20 inspecciÃƒÆ’Ã‚Â³n fÃƒÆ’Ã‚Â­sica y reconciliaciones.
- `projects/agents-multitenant/audits/wab-78/complemento-tecnico-asignaciones-evassigned-cancelacion.md`: causalidad EvAssigned, G3/G4, takeover, ABA y stale-result invalidation.
- Linear WAB-81 comments: C3 admission; D.1.9 Outbox ownership; snapshot reconciliation; WAB-79/WAB-95 coordination.

### Contratos que ya existen y deben reutilizarse
1. CRM es autoridad de Organization/Workspace, Contacto, asignaciÃƒÆ’Ã‚Â³n y `EvAssigned`; Producer deriva IDs del inbound persistido, nunca del payload externo.
2. El Producer AgentInstance reemplaza routing Bot; no hay fallback legacy.
3. `Contacto.ref_ev_assigned` es pointer canÃƒÆ’Ã‚Â³nico a transiciÃƒÆ’Ã‚Â³n confirmada. Producer resuelve ese evento exacto y valida tuple/contact/workspace/target; no timestamp-latest ni `assignmentVersion`. La escritura Outbox requiere checkpoint race-safe/transaction/CAS.
4. Unassigned sigue DEC-CM-06/08: asignar default sÃƒÆ’Ã‚Â³lo si ACTIVE y transiciÃƒÆ’Ã‚Â³n confirmada; human no ejecuta.
5. MensajerÃƒÆ’Ã‚Â­a separada: CRM `AgentMessageOutbox` ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ AI Service `AgentExecutionInbox` ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ AI Service `ExecutionResultOutbox` ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ CRM `ExecutionResultInbox`. At-least-once; durable-before-ACK; dedup durable, Redis sÃƒÆ’Ã‚Â³lo cache; stable `originMessageId` y `executionId`; transport no exactly-once.
6. Admission transaccional OPEN-only sobre `AgentInstanceExecutionControl` compartido con Retirement BLOCK; crea AgentExecution+Inbox y conserva admission generation/sequence. Duplicate identity/hash devuelve original; conflict no crea otra.
7. Runtime reutiliza ExecutionEngine/pipeline neutral adaptado a AgentDefinition/AgentInstance; retirar Bot routing, no mantener paralelo.
8. Resultado debe revalidar asignaciÃƒÆ’Ã‚Â³n vigente y `evAssignedId` antes de efecto externo; cancelar/invalidate pendientes y resultados obsoletos por takeover/reassignment, incluyendo ABA.

### Correcciones a la propuesta Ãƒâ€šÃ‚Â§17 (no asumirla ÃƒÆ’Ã‚Â­ntegra)
- Ãƒâ€šÃ‚Â§17 envelope/snapshot es propuesta y tiene correcciones en Ãƒâ€šÃ‚Â§19/Ãƒâ€šÃ‚Â§20; no usar `snapshotVersion` como versiÃƒÆ’Ã‚Â³n AgentInstance ni `configHash` legacy.
- Contrato canÃƒÆ’Ã‚Â³nico utiliza `effectiveConfigHash`, `effectiveConfigSnapshot` y `promptSnapshotHash`, y captura snapshot en durable acceptance; `LLMRoute.version` sÃƒÆ’Ã‚Â³lo es route revision.
- Snapshot: resolver fuentes autoritativas (CRM AgentInstance/Rules/SkillConfiguration; AI Service AgentDefinition/Prompt/LLMRoute/catalog; policy/capabilities/authorization/entitlement) sin confiar policy caller ni defaults. MaterializaciÃƒÆ’Ã‚Â³n request-stable, persistida antes del Outbox; retry reutiliza IDs y snapshot.
- Outbox CRM genÃƒÆ’Ã‚Â©rico existente no prueba que execution-request tenga routing/retry/lease/confirm/uncertain-publish correctos. Requiere handler/poller especÃƒÆ’Ã‚Â­fico; no reutilizar AssignmentSync/Retirement handlers.
- Inbound durable identity: `EvMensaje._id` como originMessageId/crmEventId; `wamid` metadata provider, no identidad global. Conversation ref proviene de registro CRM persistido.
- La implementaciÃƒÆ’Ã‚Â³n de ResultInbox CRM debe ser nueva/contractual, no adaptar el consumer legacy `execution.completed`/Bot de forma superficial.

### ReconciliaciÃƒÆ’Ã‚Â³n de hallazgos de auditorÃƒÆ’Ã‚Â­a

La auditorÃƒÆ’Ã‚Â­a anterior detectÃƒÆ’Ã‚Â³ falta de ingress, routing y consumer compatible. Se mantiene como evidencia fÃƒÆ’Ã‚Â­sica; **no es una nueva decisiÃƒÆ’Ã‚Â³n de diseÃƒÆ’Ã‚Â±o**. Las soluciones contractuales ya estÃƒÆ’Ã‚Â¡n en D.1.9/D.6 y WAB-81 Ãƒâ€šÃ‚Â§17, reconciliadas por Ãƒâ€šÃ‚Â§19ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Å“20. La prÃƒÆ’Ã‚Â³xima fase debe hacer checkpoint fÃƒÆ’Ã‚Â­sico contra esas especificaciones, identificar fuentes/handlers existentes y definir el incremento mÃƒÆ’Ã‚Â­nimo de wiring, sin reabrir contratos ni crear envelope paralelo.

## Checkpoint fÃƒÆ’Ã‚Â­sico: CRM inbound y routing legado (2026-11-20)

### Punto de entrada confirmado
`crm/src/modules/message-pipeline/messageProcessor.service.ts` procesa inbound Meta/Evolution. Tras `ContactoSvc.newMessageReceived`, si no estÃƒÆ’Ã‚Â¡ bloqueado, el mensaje ya persistido (`mensaje._id`) y el contacto se publican en realtime; luego se llama `forwardToAIService(...)`.

### Persistencia e identidad
`Contacto.service.ts` (`procesarEntrante`) resuelve/crea contacto por Workspace, abre/reutiliza conversaciÃƒÆ’Ã‚Â³n, asigna `ref_conversation` al `MensajeInput`, guarda el mensaje mediante `saveMsj`, actualiza Conversation y Contacto. La identidad durable CRM disponible para el nuevo contrato es el `_id` del evento/mensaje persistido; `wamid`/JID son identidad de proveedor y no deben reemplazar `originMessageId`.

### Routing legado todavÃƒÆ’Ã‚Â­a conectado
`forwardToAIService` usa `DefaultBotRoutingPolicy`, `contactAssignmentClient.getActive`, `botClient` indirectamente, historial `buildBotHistory` y `inboundQueuePublisher.publishToAIService`, con `botId`, `source`, `traceId`, `configId`. Tiene guards para `fromMe`, usuario humano, conversaciÃƒÆ’Ã‚Â³n y errores. La consulta a asignaciÃƒÆ’Ã‚Â³n externa y routing tiene degradaciÃƒÆ’Ã‚Â³n fail-open en errores de defensas secundarias; no reutilizar este comportamiento para AgentInstance admission.

### No existe todavÃƒÆ’Ã‚Â­a wiring del contrato nuevo
No se observa en MessageProcessor un call-site para AgentMessageOutbox, AgentRuntimeSnapshotService ni AgentExecution admission. La transiciÃƒÆ’Ã‚Â³n debe reemplazar el forwarding legacy de mensajes inbound; no adaptar superficialmente el payload legacy.

### PrÃƒÆ’Ã‚Â³ximos puntos del checkpoint
1. Inspeccionar `saveMsj` y garantÃƒÆ’Ã‚Â­as de idempotencia/identidad ante duplicados, para establecer el checkpoint exacto de persistencia.
2. Resolver la asignaciÃƒÆ’Ã‚Â³n causal desde `Contacto.ref_ev_assigned` y validar el EvAssigned exacto (contact/workspace/target), incluyendo el caso sin asignaciÃƒÆ’Ã‚Â³n y default ACTIVE confirmado; no usar routingPolicy ni timestamp-latest.
3. Inspeccionar AgentRuntimeSnapshotService, la fuente autoritativa de policy, y cÃƒÆ’Ã‚Â³mo se obtiene organization/workspace antes de formar el snapshot.
4. Determinar si Outbox genÃƒÆ’Ã‚Â©rico/stream/poller puede reutilizarse sin interferir con assignment-sync/retirement; el handler actual sÃƒÆ’Ã‚Â³lo cubre workspace-delete, assignment-sync y coordinaciÃƒÆ’Ã‚Â³n assignment/retirement, dejando otros tipos PENDING.
5. Mapear el contrato fÃƒÆ’Ã‚Â­sico mÃƒÆ’Ã‚Â­nimo de AgentMessageOutbox ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ Admission y luego ExecutionResultInbox ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ persistOutgoing, contra WAB-81 Ãƒâ€šÃ‚Â§17ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Å“20 y D.1.9/D.6/E; mantener las correcciones de snapshot ya reconciliadas.

**Alcance:** inspecciÃƒÆ’Ã‚Â³n read-only. No se modificÃƒÆ’Ã‚Â³ cÃƒÆ’Ã‚Â³digo; no E2E; no commit/push/deploy.

## Checkpoint fÃƒÆ’Ã‚Â­sico: identidad del mensaje, asignaciÃƒÆ’Ã‚Â³n causal y Outbox (2026-11-20)

### Persistencia de mensaje
`Contacto.service.ts` `saveMsj` crea `MensajeModel` con `workspace`, `contacto`, `ref_conversation`, `wamid` y contenido; el `save()` ocurre dentro de `enFila` por contacto, tras calcular `orderAt`. El `ObjectId` de Mongo del evento queda disponible como `mensaje._id` despuÃƒÆ’Ã‚Â©s de persistir. El ÃƒÆ’Ã‚Â­ndice compuesto `{wamid, contacto}` es ÃƒÆ’Ã‚Âºnico cuando `wamid` es string, lo que protege duplicados de proveedor, pero el forwarding actual no usa una clave durable de Outbox ni prueba que persistencia y request de ejecuciÃƒÆ’Ã‚Â³n sean atÃƒÆ’Ã‚Â³micos.

### AsignaciÃƒÆ’Ã‚Â³n causal
CRM ya tiene `AgentInstanceAssignmentWriterService.transition`: crea EvAssigned, actualiza Contacto mediante CAS de `ref_ev_assigned`/tipo/ref, crea ContactAssignmentRecord y Outbox `agent.assignment.transition` en la misma sesiÃƒÆ’Ã‚Â³n/transaction. `AgentAssignmentSnapshotService` usa `Contacto.ref_ev_assigned` y busca EvAssigned por ID + Workspace; no selecciona por timestamp-latest. Este patrÃƒÆ’Ã‚Â³n es evidencia reutilizable.

No obstante, inbound `MessageProcessor` sigue llamando a `DefaultBotRoutingPolicy`: asignaciÃƒÆ’Ã‚Â³n externa legacy y fallback al primer Bot activo del Workspace. No convierte ese fallback en una transiciÃƒÆ’Ã‚Â³n durable AgentInstance ni exige evidencia `EvAssigned`. El nuevo ingress debe leer el estado causal actual, resolver el EvAssigned exacto y validar Contacto/Workspace/tipo/destino; para default, seguir el contrato de designaciÃƒÆ’Ã‚Â³n/confirmaciÃƒÆ’Ã‚Â³n de AgentInstance, no seleccionar arbitrariamente el primer instance ACTIVE.

### Snapshot y policy
`AgentRuntimeSnapshotService.resolve` valida OrganizationÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬ÂWorkspace y AgentInstance ACTIVE/Workspace, carga Definition id desde instancia, Rules y Skills enabled, AdditionalPrompt y KnowledgeBaseIds; envÃƒÆ’Ã‚Â­a snapshot request al AI Service. La policy es obligatoria como input (`allowedCapabilities`, `limits`) y deliberadamente no se inventan defaults. InspecciÃƒÆ’Ã‚Â³n no encontrÃƒÆ’Ã‚Â³ call-sites productivos del service ni resolver autoritativo de policy. Por tanto, wiring ingress aÃƒÆ’Ã‚Âºn debe identificar/integrar la fuente de policy Waby y obtener OrganizationId de Workspace antes de snapshot; no debe introducir defaults.

### Outbox existente
`OutboxModel` genÃƒÆ’Ã‚Â©rico soporta `messageId/type/payload/status/attempts/lastError`, ÃƒÆ’Ã‚Â­ndice ÃƒÆ’Ã‚Âºnico sparse de messageId y PENDING/SENT/FAILED. `Outbox.stream.ts` y poller/retry worker sÃƒÆ’Ã‚Â³lo procesan tipos allowlisted actuales: workspace delete, assignment sync, agent assignment transition y retirement coordination. `handleOutboxEvent` deja tipos desconocidos PENDING. El publisher de assignment/retirement usa ConfirmChannel; no es evidencia de publisher para AgentMessageOutbox. ReutilizaciÃƒÆ’Ã‚Â³n del modelo genÃƒÆ’Ã‚Â©rico requiere handler dedicado, routing/confirm/retry explÃƒÆ’Ã‚Â­citos, y evitar mezclar las colas de coordinaciÃƒÆ’Ã‚Â³n.

### Riesgos y decisiones pendientes para implementaciÃƒÆ’Ã‚Â³n mÃƒÆ’Ã‚Â­nima
- Persistir message + AgentMessageOutbox con idempotency key derivada de `originMessageId` (y Workspace/instance segÃƒÆ’Ã‚Âºn contrato) dentro de transacciÃƒÆ’Ã‚Â³n; no publicar directamente despuÃƒÆ’Ã‚Â©s del save.
- Definir elegibilidad de mensaje (incoming, no blocked, not fromMe, conversation vigente) y assignment causal; resolver snapshot antes o en el producer checkpoint con persistencia estable para retry.
- Resolver policy autoritativa y OrganizationId.
- Producer Outbox no debe quedar atado a GraphQL PubSub.
- No fallback a BotRoutingPolicy ni Bot runtime.

**Alcance:** inspecciÃƒÆ’Ã‚Â³n read-only; sin cambios de cÃƒÆ’Ã‚Â³digo, sin E2E, sin commit/push/deploy.

## Checkpoint fÃƒÆ’Ã‚Â­sico: respuesta AI y sender ÃƒÆ’Ã‚Âºnico CRM (2026-11-20)

### Camino legacy reutilizable como servicio de salida, no como consumer nuevo
`executionCompleted.consumer.ts` escucha `execution.completed`, filtra `payload.status=completed` y result string no vacÃƒÆ’Ã‚Â­o, valida vigencia legacy cuando hay botId, dedup Redis `bot:exec:<executionId>` (fail-open), resuelve channel con fallback a `Works.refs_configs[0]`, llama `ContactoSvc.persistOutgoing` y `dispatchOutgoing`. Esto no implementa ExecutionResultInbox ni valida AgentInstance/Assignment causal/evAssignedId; el contrato nuevo no debe adaptar este consumer por campos superficiales.

### Sender ÃƒÆ’Ã‚Âºnico y persistencia
`ContactoSvc.persistOutgoing` persiste un mensaje outbound pending, reabre/crea conversaciÃƒÆ’Ã‚Â³n y prepara el contexto del sender; `dispatchOutgoing` encola el envÃƒÆ’Ã‚Â­o y `applyOutboundResult` cierra el resultado/status/realtime. Este camino es un candidato para reutilizar tras la validaciÃƒÆ’Ã‚Â³n durable del resultado en CRM, no para invocarlo directamente desde un consumer Rabbit legacy.

### Compatibilidad y brechas
- El consumer legacy espera `type=execution.completed`, `payload.botId`, `payload.channel`, `payload.contactId`, `payload.workspaceId`, `payload.result` string.
- El resultado AgentInstance durable usa `agent.execution.completed/failed/cancelled`, `agentInstanceId`, `evAssignedId`, y `payload.result` objeto; no es compatible.
- Dedup actual Redis TTL no sustituye el durable dedup de ExecutionResultInbox.
- Falta consumer nuevo: persistir/validar envelope y causalidad en Mongo antes de ACK, aplicar dedup durable, validar asignaciÃƒÆ’Ã‚Â³n vigente y `evAssignedId` (incluyendo stale/takeover/ABA), y sÃƒÆ’Ã‚Â³lo despuÃƒÆ’Ã‚Â©s invocar el sender ÃƒÆ’Ã‚Âºnico `persistOutgoing`/`dispatchOutgoing`.
- No se debe fallback a `refs_configs[0]` para el nuevo contrato: channel/config debe provenir de la evidencia/snapshot del execution envelope conforme al contrato.

**Alcance:** inspecciÃƒÆ’Ã‚Â³n read-only; no se modificÃƒÆ’Ã‚Â³ cÃƒÆ’Ã‚Â³digo, no E2E, no commit/push/deploy.

## ImplementaciÃƒÆ’Ã‚Â³n Bloque A ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â subbloque 1: resolver de asignaciÃƒÆ’Ã‚Â³n causal (2026-11-20)

### CÃƒÆ’Ã‚Â³digo creado
- `crm/src/services/InboundAgentAssignmentResolver.service.ts`
- `crm/src/services/__tests__/InboundAgentAssignmentResolver.service.test.ts`

### Contrato del resolver
- Lee Contacto por `_id + ref_works`.
- Deriva `assignedType` usando campo persistido; sÃƒÆ’Ã‚Â³lo compatibilidad para histÃƒÆ’Ã‚Â³ricos sin tipo: `ref_user_assigned` ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ human, `ref_agent_assigned` ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ agent, de otro modo unassigned.
- Human/unassigned son resultados legÃƒÆ’Ã‚Â­timos sin ejecuciÃƒÆ’Ã‚Â³n IA y nunca eligen un default arbitrario.
- Agent exige `ref_agent_assigned` y `ref_ev_assigned`.
- Busca el EvAssigned exacto por `_id + contacto + workspace` y verifica `newAssignedType=agent` y `newAssignedRef` igual al AgentInstance actual.
- Valida AgentInstance ACTIVE y Workspace propietario, y requiere `agentDefinitionId`.
- Errores de evidencia/ownership/estado retornan `ineligible` fail-closed.

### ValidaciÃƒÆ’Ã‚Â³n
- CRM type-check: PASS.
- Jest enfocado: 1 suite / 5 tests PASS.
- Casos: human; unassigned sin fallback; pointer causal faltante; target EvAssigned inconsistente; AgentInstance ACTIVE y scoped.

### PrÃƒÆ’Ã‚Â³ximo bloqueo del Producer
Antes de crear AgentMessageOutbox/envelope y conectar inbound, falta cerrar fuente autoritativa de policy/capabilities/limits y canonical snapshot/hash/version semantics, identificadas en WAB-81 Ãƒâ€šÃ‚Â§17. No se inventan defaults ni caller policy. El resolver queda aislado y aÃƒÆ’Ã‚Âºn no conectado al MessageProcessor.

**Alcance:** sÃƒÆ’Ã‚Â³lo se creÃƒÆ’Ã‚Â³ resolver + tests; no E2E; sin commit/push/deploy.

## ImplementaciÃƒÆ’Ã‚Â³n Bloque A ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â subbloque 2: verificaciÃƒÆ’Ã‚Â³n de integridad del snapshot (2026-11-20)

Se fortaleciÃƒÆ’Ã‚Â³ `crm/src/infrastructure/ai-service/agent-runtime-snapshot.client.ts`: despuÃƒÆ’Ã‚Â©s de validar success e identidad Organization/Workspace/AgentInstance/AgentDefinition, CRM ahora exige los componentes del snapshot y recalcula SHA-256 sobre `agentDefinition`, `agentInstance` y `policy` con la serializaciÃƒÆ’Ã‚Â³n JSON actual del contrato. Si el hash recibido no coincide, falla con `AGENT_RUNTIME_SNAPSHOT_HASH_MISMATCH` antes de devolver el snapshot.

Tests nuevos `src/infrastructure/ai-service/__tests__/agent-runtime-snapshot.client.test.ts`: hash vÃƒÆ’Ã‚Â¡lido, contenido alterado y mismatch de identidad (3/3 PASS). CRM `pnpm type-check` PASS.

Esto valida integridad del contenido recibido, **no** autorÃƒÆ’Ã‚Â­a/autoridad de policy: la fuente de policy/allowedCapabilities/limits sigue sin estar implementada y el endpoint continÃƒÆ’Ã‚Âºa recibiendo esos valores desde CRM. No desbloquea producer/outbox ni sustituye el resolver confiable requerido. No E2E, commit, push ni deploy.

## Cierre del componente Policy / Capabilities / Limits ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â investigaciÃƒÆ’Ã‚Â³n fÃƒÆ’Ã‚Â­sica (2026-11-20)

La inspecciÃƒÆ’Ã‚Â³n focalizada confirma que el contrato conceptual estÃƒÆ’Ã‚Â¡ definido en `configuracion-efectiva-y-capacidades.md` y WAB-81 Ãƒâ€šÃ‚Â§Ãƒâ€šÃ‚Â§20.3ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Å“20.7; no falta diseÃƒÆ’Ã‚Â±o de la intersecciÃƒÆ’Ã‚Â³n de capabilities. La investigaciÃƒÆ’Ã‚Â³n de fuentes fÃƒÆ’Ã‚Â­sicas queda cerrada (CRM, AI Service, infraestructura y frontend previamente inspeccionados).

**Contrato conocido:** `CapabilitiesEffective = PlatformAllowed ÃƒÂ¢Ã‹â€ Ã‚Â© DefinitionAllowed ÃƒÂ¢Ã‹â€ Ã‚Â© InstanceEnabled ÃƒÂ¢Ã‹â€ Ã‚Â© CurrentAuthorization`. La autoridad de plataforma, AgentDefinition, configuraciÃƒÆ’Ã‚Â³n AgentInstance y autorizaciÃƒÆ’Ã‚Â³n contextual son independientes. Skills/Rules/prompts no conceden autorizaciÃƒÆ’Ã‚Â³n. Entitlements AI/limits deben ser autoritativos y no inferirse de suscripciones genÃƒÆ’Ã‚Â©ricas; P4 crÃƒÆ’Ã‚Â©ditos es un contrato separado.

**Fuentes fÃƒÆ’Ã‚Â­sicas disponibles y no equivalentes:** AI Service `CapabilityRegistry` registra capabilities implementadas, no catÃƒÆ’Ã‚Â¡logo Waby de autorizaciÃƒÆ’Ã‚Â³n. AgentDefinition no tiene allowlist de capabilities. CRM AgentInstance tiene recursos de Workspace, no policy runtime completa. CRM subscription plan tiene lÃƒÆ’Ã‚Â­mites genÃƒÆ’Ã‚Â©ricos, sin mapping AI. Los permisos humanos de Workspace no son CurrentAuthorization de ejecuciÃƒÆ’Ã‚Â³n. No existe fuente fÃƒÆ’Ã‚Â­sica de mandatory platform policy, DefinitionAllowed, CurrentAuthorization o AI runtime limits/entitlements.

**ConclusiÃƒÆ’Ã‚Â³n:** No se implementa resolver de producciÃƒÆ’Ã‚Â³n, merge de limits ni default. Cualquier algoritmo que invente el significado de lÃƒÆ’Ã‚Â­mites ausentes o derive autorizaciones de fuentes no autoritativas romperÃƒÆ’Ã‚Â­a el contrato. El snapshot debe permanecer fail-closed y no aceptar policy caller-supplied como autoridad.

**MÃƒÆ’Ã‚Â­nimo bloqueo de implementaciÃƒÆ’Ã‚Â³n/decisiÃƒÆ’Ã‚Â³n:** identificar/crear (por sus owners) las fuentes autoritativas para: (1) catÃƒÆ’Ã‚Â¡logo + policy obligatoria y mapping a CapabilityRegistry; (2) allowlist de AgentDefinition; (3) configuraciÃƒÆ’Ã‚Â³n habilitada por AgentInstance; (4) decisiÃƒÆ’Ã‚Â³n autenticada de autorizaciÃƒÆ’Ã‚Â³n runtime con semÃƒÆ’Ã‚Â¡ntica de revocaciÃƒÆ’Ã‚Â³n; (5) entitlement/limits AI y relaciÃƒÆ’Ã‚Â³n explÃƒÆ’Ã‚Â­cita con Organization/Workspace y P4. TambiÃƒÆ’Ã‚Â©n debe especificarse cÃƒÆ’Ã‚Â³mo combinar lÃƒÆ’Ã‚Â­mites de diferentes autoridades (upper-bound/minimum, claves y semÃƒÆ’Ã‚Â¡ntica de ausencia) antes de implementar.

La investigaciÃƒÆ’Ã‚Â³n queda cerrada; el componente **no puede considerarse implementado** por falta de autoridades fÃƒÆ’Ã‚Â­sicas. WAB-81 continÃƒÆ’Ã‚Âºa In Progress. No se creÃƒÆ’Ã‚Â³ catÃƒÆ’Ã‚Â¡logo paralelo ni se reutilizÃƒÆ’Ã‚Â³ Bot legacy. Sin E2E, commit, push o deploy.

### Guard fail-closed de policy en CRM (20/11/2026)

Se retirÃƒÆ’Ã‚Â³ la policy caller-supplied del input de `AgentRuntimeSnapshotService`. El servicio solicita policy a `IAgentRuntimePolicyResolver` usando ÃƒÆ’Ã‚Âºnicamente IDs validados de Organization, Workspace, AgentInstance y AgentDefinition. El resolver default `FailClosedAgentRuntimePolicyResolver` falla con `AGENT_RUNTIME_POLICY_RESOLVER_NOT_CONFIGURED`; no devuelve capabilities vacÃƒÆ’Ã‚Â­as ni lÃƒÆ’Ã‚Â­mites inventados ni acepta policy del llamador. Se agregÃƒÆ’Ã‚Â³ test unitario del rechazo (1/1 PASS); CRM type-check PASS.

Esto es un guard seguro/inyecciÃƒÆ’Ã‚Â³n de autoridad, **no una resoluciÃƒÆ’Ã‚Â³n funcional**: aÃƒÆ’Ã‚Âºn no existe implementaciÃƒÆ’Ã‚Â³n que consulte las fuentes autoritativas. El servicio no debe conectarse al Producer hasta que un resolver real sea implementado y validado.

## ReconciliaciÃƒÆ’Ã‚Â³n y avance del Policy Resolver ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â 20/11/2026

La revisiÃƒÆ’Ã‚Â³n de `configuracion-efectiva-y-capacidades.md`, WAB-78 (CTX-P3/CTX-P4, ARQ-20) y WAB-80 confirma que permisos, capacidades, ownership y lÃƒÆ’Ã‚Â­mites funcionales **ya estÃƒÆ’Ã‚Â¡n definidos conceptualmente**. La implementaciÃƒÆ’Ã‚Â³n no debe crear un nuevo modelo de autoridad ni redefinir decisiones aprobadas.

Contrato reconciliado:
- `PlatformAllowed`: catÃƒÆ’Ã‚Â¡logo y polÃƒÆ’Ã‚Â­ticas obligatorias Waby.
- `DefinitionAllowed`: capacidades mÃƒÆ’Ã‚Â¡ximas publicadas por AgentDefinition.
- `InstanceEnabled`: selecciÃƒÆ’Ã‚Â³n/habilitaciÃƒÆ’Ã‚Â³n de AgentInstance, dentro de DefinitionAllowed.
- `CurrentAuthorization`: autorizaciÃƒÆ’Ã‚Â³n vigente, tenant/ownership y restricciones de entitlement aplicables.
- `CapabilitiesEffective`: intersecciÃƒÆ’Ã‚Â³n estricta de las cuatro capas. Skills/Rules/Knowledge/prompt no conceden capabilities.
- Los lÃƒÆ’Ã‚Â­mites de runtime son mÃƒÆ’Ã‚Â¡ximos/upper bounds. Cuando varias autoridades definen el mismo lÃƒÆ’Ã‚Â­mite numÃƒÆ’Ã‚Â©rico, la composiciÃƒÆ’Ã‚Â³n usa el mÃƒÆ’Ã‚Â­nimo; una capa que no declara ese lÃƒÆ’Ã‚Â­mite no agrega un valor ni un default. Valores negativos/no finitos y autoridad invÃƒÆ’Ã‚Â¡lida son errores fail-closed.

ImplementaciÃƒÆ’Ã‚Â³n local adicional:
- `IntersectingAgentRuntimePolicyResolver` implementa la composiciÃƒÆ’Ã‚Â³n de las cuatro capas y valida los valores de lÃƒÆ’Ã‚Â­mites.
- `IAgentRuntimePolicyAuthoritySource` delimita la fuente autoritativa que debe resolverlas; no introduce nuevas entidades ni inventa fuentes.
- `FailClosedAgentRuntimePolicyResolver` sigue siendo el default en producciÃƒÆ’Ã‚Â³n mientras el source real no estÃƒÆ’Ã‚Â© conectado. El resolver de composiciÃƒÆ’Ã‚Â³n es testeable, pero no se declara wired/production-ready.
- La fuente de authorities aÃƒÆ’Ã‚Âºn no estÃƒÆ’Ã‚Â¡ implementada: las polÃƒÆ’Ã‚Â­ticas/catalogo de plataforma, capabilities/versiones de AgentDefinition, enabled capabilities de AgentInstance y entitlement/autorizaciÃƒÆ’Ã‚Â³n efectiva requieren adaptaciÃƒÆ’Ã‚Â³n a sus owners reales. El plan/suscripciÃƒÆ’Ã‚Â³n CRM actual no contiene entitlement de AI-agent/capabilities equivalente; no se reutilizan lÃƒÆ’Ã‚Â­mites comerciales genÃƒÆ’Ã‚Â©ricos ni crÃƒÆ’Ã‚Â©ditos P4 como lÃƒÆ’Ã‚Â­mites tÃƒÆ’Ã‚Â©cnicos.

ValidaciÃƒÆ’Ã‚Â³n: test focalizado 5/5 PASS; `pnpm type-check` PASS. Sin E2E, commit, push ni deploy.

## Contrato de envelope CRM ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ AgentExecutionInbox ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â avance local

### Implementado

- Nuevo contrato tipado `crm/src/infrastructure/ai-service/agent-execution-envelope.ts` para `AgentExecutionEnvelopeV1` conforme al Ãƒâ€šÃ‚Â§17.2 de WAB-81.
- Validador fail-closed de metadata, identidad estable `messageId = crmEventId`, provider/providerMessageId, asignaciÃƒÆ’Ã‚Â³n causal `assignedType=agent`, timestamps, y correspondencia de tenancy/AgentInstance/AgentDefinition con `runtimeSnapshot.identity`.
- Tests unitarios: envelope vÃƒÆ’Ã‚Â¡lido, conflicto de identidad del evento, snapshot de otra AgentInstance y asignaciÃƒÆ’Ã‚Â³n invÃƒÆ’Ã‚Â¡lida (4/4 PASS).
- `pnpm type-check` de CRM PASS.

### DelimitaciÃƒÆ’Ã‚Â³n

Este contrato aÃƒÆ’Ã‚Âºn no se conecta al inbound `MessageProcessor`, al CRM Outbox ni a un endpoint de admission. El inbound actual todavÃƒÆ’Ã‚Â­a usa `BotRoutingPolicy` + publisher legacy y no se debe reutilizar ni activar para AgentInstance. El prÃƒÆ’Ã‚Â³ximo bloque debe crear la persistencia durable de AgentMessageOutbox desde el mensaje CRM ya persistido, resolver ÃƒÆ’Ã‚Âºnicamente la asignaciÃƒÆ’Ã‚Â³n causal con `InboundAgentAssignmentResolver`, y publicar por publisher confirm/retry. No se puede admitir ejecuciÃƒÆ’Ã‚Â³n hasta que el resolver de policy tenga fuentes autoritativas conectadas; el comportamiento actual es fail-closed.

### Archivos

- `crm/src/infrastructure/ai-service/agent-execution-envelope.ts`
- `crm/src/infrastructure/ai-service/__tests__/agent-execution-envelope.test.ts`

Sin commit, push, deploy ni E2E.

## AgentMessageOutbox durable intent ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â avance local

### Implementado

- `crm/src/infrastructure/ai-service/AgentMessageOutbox.service.ts` introduce `OUTBOX_AGENT_MESSAGE_RECEIVED` y `AgentMessageOutboxService.enqueue()`.
- La intenciÃƒÆ’Ã‚Â³n durable conserva el ID CRM del evento como `messageId`, el ID del provider, provider (`meta|evo`), Organization/Workspace/Contact/Conversation, canal, contenido y `receivedAt`.
- Enqueue valida campos y provider; inserta Outbox `PENDING`; deduplica por `messageId` consultando y usando el ÃƒÆ’Ã‚Â­ndice unique sparse existente. Maneja carrera `E11000` y devuelve el evento ya creado.
- Tests 3/3 PASS; `pnpm type-check` PASS.

### DelimitaciÃƒÆ’Ã‚Â³n crÃƒÆ’Ã‚Â­tica

Este servicio aÃƒÆ’Ã‚Âºn no estÃƒÆ’Ã‚Â¡ invocado desde `MessageProcessor`, ni el Outbox worker/change stream/poller estÃƒÆ’Ã‚Â¡ habilitado para este type. Se mantiene aislado para evitar activar un evento que todavÃƒÆ’Ã‚Â­a no tiene publisher/admission completos. La siguiente etapa implementarÃƒÆ’Ã‚Â¡ resolver causal + snapshot fail-closed + publisher confirm hacia la queue/ingress acordada, y luego wiring atÃƒÆ’Ã‚Â³mico o recuperable desde el mensaje persistido. La creaciÃƒÆ’Ã‚Â³n del Outbox y el mensaje CRM aÃƒÆ’Ã‚Âºn no comparten transacciÃƒÆ’Ã‚Â³n en este avance.

Archivos:
- `crm/src/infrastructure/ai-service/AgentMessageOutbox.service.ts`
- `crm/src/infrastructure/ai-service/__tests__/AgentMessageOutbox.service.test.ts`

Sin commit, push, deploy ni E2E.

## ReconciliaciÃƒÆ’Ã‚Â³n adicional: CRM Outbox vs AI Service publisher (20-11-2026)

### Hallazgo
`AgentMessageOutboxService` persiste un payload de evento `agent.message.received` en el CRM `OutboxModel`, pero no estÃƒÆ’Ã‚Â¡ conectado al publisher de AI Service. El publisher existente `DurableMessageOutboxPublisher` publica envelopes provenientes de `MessageOutboxModel` (colecciÃƒÆ’Ã‚Â³n `retirement_coordination_outbox`) y estÃƒÆ’Ã‚Â¡ dedicado al flujo de Retirement; no consume el CRM Outbox ni constituye el publisher del nuevo evento.

El CRM payload actual tampoco contiene el envelope completo que espera la mensajerÃƒÆ’Ã‚Â­a de AI Service (`schemaVersion`, `messageId`, `messageType`, `occurredAt`, `source`, `workspaceId`, `payload`). No se debe adaptar/reutilizar el publisher de Retirement.

### DecisiÃƒÆ’Ã‚Â³n tÃƒÆ’Ã‚Â©cnica pendiente de implementaciÃƒÆ’Ã‚Â³n
Crear publisher dedicado del CRM Outbox para `agent.message.received`, que:
- consuma el Outbox CRM de manera durable con lease/retry y publisher confirms;
- construya envelope versionado compatible con el contrato AgentExecution;
- publique en exchange/routing key acordados y dedicados;
- preserve `messageId` estable y detecte conflictos de identidad/payload;
- confirme el Outbox sÃƒÆ’Ã‚Â³lo despuÃƒÆ’Ã‚Â©s de publisher confirm;
- maneje fallos/reintentos sin perder el evento.

Luego implementar el consumidor dedicado de AI Service, validar el envelope y llamar `MongoAgentExecutionAdmissionRepository.admit()`. ACK sÃƒÆ’Ã‚Â³lo despuÃƒÆ’Ã‚Â©s de admisiÃƒÆ’Ã‚Â³n durable. El consumidor legacy `ai-service.message-received` permanece sin cambios.

### Estado
InspecciÃƒÆ’Ã‚Â³n completada; no se activÃƒÆ’Ã‚Â³ el consumidor ni se conectÃƒÆ’Ã‚Â³ el producer. No commit, push, deploy ni E2E.

## Publisher CRM + ingress AI Service (implementaciÃƒÆ’Ã‚Â³n local, 20-11-2026)

### CRM Outbox publisher
- `agent.message.received` agregado al Outbox change stream y poller de respaldo.
- `handleOutboxEvent` invoca publisher dedicado `src/Rabbit/producer/agentMessageReceived.producer.ts`.
- Publisher valida payload v1, resuelve asignaciÃƒÆ’Ã‚Â³n causal persistida mediante `InboundAgentAssignmentResolver`; HUMAN/UNASSIGNED se consideran no elegibles y se marca evento atendido sin publicar. Evidencia de asignaciÃƒÆ’Ã‚Â³n invÃƒÆ’Ã‚Â¡lida provoca retry, no descarte.
- Para asignaciÃƒÆ’Ã‚Â³n Agent resuelve snapshot mediante `AgentRuntimeSnapshotService`, crea envelope `AgentExecutionEnvelopeV1` con IDs estables (`messageId`/`executionId` derivado de EvMensaje._id), y publica en `ai-service.events` routing `agent.message.received`, persistente y con publisher confirm antes de marcar Outbox SENT.
- Fallos del publisher/resolver/snapshot se conservan PENDING con retry/backoff existente.

### AI Service ingress
- Consumidor dedicado `AgentExecutionMessageConsumer`, queue durable `ai-service.agent-execution`, bind al routing key dedicado.
- Envelope Zod + validaciÃƒÆ’Ã‚Â³n cruzada de identidad tenancy/assignment/snapshot.
- AdmisiÃƒÆ’Ã‚Â³n mediante `MongoAgentExecutionAdmissionRepository.admit()`; ACK sÃƒÆ’Ã‚Â³lo luego de ÃƒÆ’Ã‚Â©xito durable (transaction AgentExecution + AgentExecutionInbox).
- Mensajes malformed/identity mismatch/conflict dead-letter a `ai-service.agent-execution.dead-letter`; fallos transitorios (control admission bloqueado/no disponible) requeue.
- Registrado en bootstrap RabbitMQ y graceful shutdown. No toca el consumer legacy `ai-service.message-received`.

### ValidaciÃƒÆ’Ã‚Â³n
- AI Service type-check: PASS.
- CRM type-check: PASS.
- Consumer tests: 4/4 PASS (admit+ACK, malformedÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢DLQ, transientÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢requeue, conflictÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢DLQ).

### Limitaciones y no activaciÃƒÆ’Ã‚Â³n E2E
- `AgentMessageOutboxService.enqueue()` sigue sin invocarse desde el persist/MessageProcessor del mensaje CRM; producer aÃƒÆ’Ã‚Âºn aislado del ingreso real.
- `AgentRuntimePolicyResolver` default fail-closed sigue sin fuentes de autoridad configuradas. El publisher Agent asignado no puede resolver snapshot mientras esto no estÃƒÆ’Ã‚Â© conectado y mantiene Outbox en retry (no fail-open).
- El runtime worker puede estar deshabilitado por falta de credential registry key; la admisiÃƒÆ’Ã‚Â³n durable puede quedar pendiente de runtime.
- No se ejecutÃƒÆ’Ã‚Â³ E2E, no se desplegÃƒÆ’Ã‚Â³, no commit/push.

### RevisiÃƒÆ’Ã‚Â³n del punto de activaciÃƒÆ’Ã‚Â³n del producer

Se inspeccionÃƒÆ’Ã‚Â³ `Contacto.service.ts` y el pipeline inbound Meta/Evolution para determinar dÃƒÆ’Ã‚Â³nde invocar `AgentMessageOutboxService.enqueue()`. La persistencia de EvMensaje (`saveMsj`) y Outbox actualmente no comparten transacciÃƒÆ’Ã‚Â³n; invocar enqueue despuÃƒÆ’Ã‚Â©s de guardar puede perder el evento si Outbox falla y el handler captura el error, mientras que enqueue antes puede publicar una ejecuciÃƒÆ’Ã‚Â³n sin EvMensaje persistido. No se conectÃƒÆ’Ã‚Â³ el enqueue a `newMessageReceived` hasta diseÃƒÆ’Ã‚Â±ar la atomicidad/reconciliaciÃƒÆ’Ã‚Â³n y preservar idempotencia del ingreso. El producer y consumer estÃƒÆ’Ã‚Â¡n implementados, pero el Outbox inicial continÃƒÆ’Ã‚Âºa sin ser alimentado por el flujo real de mensajes.


## Inbound real: persistencia CRM + AgentMessageOutbox (avance local, 20-11-2026)

### Implementado

- `Contacto.service.saveMsj()` ahora detecta mensajes entrantes (`my=false`) con `wamid` y conversaci?n/contacto persistidos.
- El guardado de `EvMensaje` y la creaci?n del Outbox `agent.message.received` se ejecutan en una misma transacci?n MongoDB.
- El evento conserva `EvMensaje._id` como `messageId`, el wamid como `providerMessageId`, Organization derivada del Workspace CRM, Workspace/Contact/Conversation, provider Meta/Evolution, canal y contenido textual disponible.
- La transacci?n no ejecuta RabbitMQ ni LLM. El Outbox Worker contin?a siendo el publisher posterior al commit.
- `AgentMessageOutboxService.enqueue()` acepta `ClientSession` para consultar/insertar Outbox dentro de la transacci?n; el comportamiento previo sin sesi?n se conserva.
- Se verific? conectividad MongoDB local de configuraci?n sin exponer URI/credenciales; `hello` confirm? Replica Set, sesiones y soporte de transacciones.

### Fail-closed y alcance

- Mensajes propios (`my=true`) no generan Outbox de Agent.
- Mensajes sin wamid, conversaci?n/contacto no generan evento Agent.
- Workspace sin `ref_organizacion` provoca rollback y no confirma mensaje/evento parcialmente.
- La transacci?n s?lo cubre EvMensaje + Outbox; media, contacto/conversaci?n, RabbitMQ y ejecuci?n quedan fuera.
- El handler inbound existente captura errores de persistencia; por lo tanto, el comportamiento de reintento del webhook ante rollback debe validarse antes de considerar esta integraci?n operativa. No se declara E2E ni garant?a de recuperaci?n por reentrega del provider.
- El payload para media/no-texto usa `searchText` o marcador de tipo cuando no existe texto; la ingesti?n multimodal del Agent Runtime queda fuera de este bloque.

### Validaci?n

- CRM `pnpm type-check`: ejecutado; no se observaron errores TypeScript en salida.
- Test focalizado `AgentMessageOutbox.service.test.ts`: ejecutado; la salida Jest fue silenciosa y no expuso resumen/exit code en el canal remoto, por lo que se registra como resultado no verificable, no como PASS.
- `git diff --check`: ejecutado; resultado no expuesto de forma verificable.
- Sin E2E, commit, push ni deploy.

### Archivos

- `crm/src/services/Contacto.service.ts`
- `crm/src/infrastructure/ai-service/AgentMessageOutbox.service.ts`

## 21. Webhook ingress: ACK posterior a publisher-confirm (20/11/2026)

### Hallazgo
El webhook respondÃƒÆ’Ã‚Â­a HTTP 202 antes del `setImmediate` que publicaba en RabbitMQ. El publisher utilizaba un `ConfirmChannel`, pero `publishInbound/publishUnrouted` no esperaban `waitForConfirms`; `WebhookIngress` ademÃƒÆ’Ã‚Â¡s trataba los errores de enqueue como ÃƒÆ’Ã‚Â©xito por defecto. Esto abrÃƒÆ’Ã‚Â­a una ventana de pÃƒÆ’Ã‚Â©rdida anterior a la persistencia CRM y al `AgentMessageOutbox`.

### ImplementaciÃƒÆ’Ã‚Â³n local
- `InboundQueuePublisher.publishInbound` y `publishUnrouted` ahora comprueban backpressure (`sendToQueue=false`) y esperan `waitForConfirms()` antes de devolver aceptaciÃƒÆ’Ã‚Â³n.
- `WebhookIngress` trata false/exception de publicaciÃƒÆ’Ã‚Â³n como fallo de aceptaciÃƒÆ’Ã‚Â³n; delivery estricto es fail-closed y no depende de `STRICT_DELIVERY=false`.
- `POST /webhook/:channel/:workspace` espera el resultado del ingress. Devuelve HTTP 202 sÃƒÆ’Ã‚Â³lo tras confirmaciÃƒÆ’Ã‚Â³n de RabbitMQ; ante confirmaciÃƒÆ’Ã‚Â³n negativa, backpressure o error devuelve 503 para permitir retry del proveedor. Se eliminÃƒÆ’Ã‚Â³ el ACK previo y el `setImmediate` que desacoplaba la publicaciÃƒÆ’Ã‚Â³n sin garantÃƒÆ’Ã‚Â­a durable.
- La validaciÃƒÆ’Ã‚Â³n de autenticidad Meta/Evolution permanece antes del enqueue.

### ValidaciÃƒÆ’Ã‚Â³n
- Jest focalizado: `publisher.test.ts` + `hook.route.autenticidad.test.ts`: **2 suites, 20 tests PASS**.
- CRM `pnpm type-check`: **PASS**.
- Diff-check focalizado: **PASS**.
- No E2E, no commit/push/deploy.

### Limite restante
Este checkpoint describio el estado anterior del consumer. La recuperacion parcial de inbound se implemento y valido en la seccion 22; permanece pendiente la idempotencia ante commit ambiguo/duplicado y la validacion de side-effects bajo reentrega.
## 22. Recovery durable del consumer inbound (20/11/2026)

### PropagaciÃƒÆ’Ã‚Â³n de fallos CRM
- `Contacto.procesarEntrante` ahora devuelve `processingFailed`, `messagePersisted` y `agentOutboxPersisted` para distinguir rollback/fallo previo de efectos que ya quedaron durables.
- `MessageProcessor` propaga `InboundMessagePersistenceError` cuando el mensaje no quedÃƒÆ’Ã‚Â³ durable; si `EvMensaje + AgentMessageOutbox` ya confirmÃƒÆ’Ã‚Â³ su transacciÃƒÆ’Ã‚Â³n, evita retry del procesamiento por fallos posteriores de UI/pubsub.

### Retry/DLQ RabbitMQ
- Nueva cola `inbound.messages.retry.v1`: durable, TTL 5 s, DLX/routing-key hacia `inbound.messages`.
- MÃƒÆ’Ã‚Â¡ximo 5 retries adicionales (6 intentos contando el original); header `x-waby-inbound-retry-count` y metadata `x-waby-inbound-last-error`.
- Nueva DLQ durable `inbound.messages.dead-letter.v1`; DLQ separadas para `inbound.status` y `inbound.unrouted`.
- Ante error, el consumer publica el original persistent y espera publisher-confirm antes de ACK. Si backpressure/confirm falla, NACK con requeue del original.
- El mensaje original no se ACK hasta que retry o DLQ quedÃƒÆ’Ã‚Â³ confirmado.

### ValidaciÃƒÆ’Ã‚Â³n
- Tests focalizados de recovery, MessageProcessor, publisher y webhook: **4 suites / 52 tests PASS**.
- CRM `pnpm type-check`: **PASS**.
- No E2E ni conexiÃƒÆ’Ã‚Â³n real con RabbitMQ; no commit/push/deploy.

### LÃƒÆ’Ã‚Â­mite restante
- Los retries son at-least-once. Falta cerrar idempotencia explÃƒÆ’Ã‚Â­cita para el caso de commit ambiguo/duplicado `EvMensaje` (ÃƒÆ’Ã‚Â­ndice compuesto `wamid+contacto`) y comprobar que los side-effects CRM no se duplican bajo reentrega. El `AgentMessageOutbox` es transaccional con el mensaje, por lo que el evento durable puede existir aunque el procesamiento CRM posterior haya terminado en DLQ.
- Esto no desbloquea la fuente fÃƒÆ’Ã‚Â­sica de policy authorities del snapshot; permanece fail-closed.

## 23. AgentInstanceExecutionRuntime integrado al ExecutionEngine (20/11/2026)

### Integracion arquitectonica
- Se ampliÃƒÆ’Ã‚Â³ `ExecutionEngine` con `executeAgentRuntime(executionId, signal, steps)`, una frontera de orquestaciÃƒÆ’Ã‚Â³n para etapas del runtime AgentInstance. Valida ID, pipeline no vacÃƒÆ’Ã‚Â­o, nombres ÃƒÆ’Ã‚Âºnicos y callbacks; ejecuta en orden; chequea cancelaciÃƒÆ’Ã‚Â³n antes/despuÃƒÆ’Ã‚Â©s de cada etapa; devuelve executionId, etapas completadas y duraciÃƒÆ’Ã‚Â³n.
- `AgentInstanceExecutionRuntime` ya no ejecuta retrieval/composiciÃƒÆ’Ã‚Â³n/invocaciÃƒÆ’Ã‚Â³n directamente en secuencia propia. Define tres etapas y delega su orquestaciÃƒÆ’Ã‚Â³n al Engine:
  1. `knowledge-retrieval`
  2. `context-composition`
  3. `provider-invocation`
- Se conserva separado el lifecycle durable de `AgentExecutionInboxWorker`/`MongoAgentExecutionRepository`/`ExecutionResultOutbox`; no se fuerza el modelo AgentInstance dentro de legacy `ExecutionContext`, BotExecution ni legacy memory persister.
- Composition root ahora construye `ExecutionEngine` primero e inyecta el mismo Engine en `AgentInstanceExecutionRuntime`; el worker durable continÃƒÆ’Ã‚Âºa invocando el runtime, que pasa por el Engine.
- `AgentInstanceExecutionRuntime.parseExecutionInput` admite `message` y `content`, resolviendo la discrepancia actual del Admission Consumer que persistÃƒÆ’Ã‚Â­a `input.content`.

### Validaciones
- Runtime tests: **5 PASS**.
- ExecutionEngine AgentRuntime tests: **3 PASS** (orden secuencial, cancellation, step validation).
- AgentExecution MessageConsumer tests: **4 PASS**.
- Total bloque: **3 suites / 12 tests PASS**.
- AI Service `pnpm type-check`: **PASS**.
- Jest aÃƒÆ’Ã‚Âºn reporta warning de handles abiertos y pnpm warning de configuraciÃƒÆ’Ã‚Â³n deprecated; no bloquean estos tests.

### LÃƒÆ’Ã‚Â­mites funcionales actuales (no declarar agente completo)
- La integraciÃƒÆ’Ã‚Â³n conecta y ejecuta un agente conversacional bÃƒÆ’Ã‚Â¡sico (snapshot + Knowledge + context + proveedor + resultado durable), pero el runtime nuevo todavÃƒÆ’Ã‚Â­a no usa el capability/tool-call loop del ExecutionEngine legacy. La invocaciÃƒÆ’Ã‚Â³n actual es una sola llamada LLM; `AgentInstanceLLMInvoker` no transporta AbortSignal al provider request, por lo que la cancelaciÃƒÆ’Ã‚Â³n es cooperativa entre etapas y posterior al provider response.
- La inbound envelope actual incluye mensaje, no historial conversacional; runtime ejecuta sin history hasta que se conecte el gateway de historial CRM.
- El snapshot authority source permanece fail-closed/no configurado segÃƒÆ’Ã‚Âºn checkpoint anterior.
- No se ejecutÃƒÆ’Ã‚Â³ E2E, provider real, RabbitMQ real ni prueba de webhook; no commit/push/deploy.

## 24. ContinuaciÃƒÆ’Ã‚Â³n acordada: Agent Runtime capability loop (20/11/2026)

Se acuerda continuar localmente con el orden:
1. Inspeccionar ExecutionEngine, CapabilityRegistry/Executor y provider tool-call contracts.
2. Reconciliar con WAB-78/WAB-81: `CapabilitiesEffective = PlatformAllowed ÃƒÂ¢Ã‹â€ Ã‚Â© DefinitionAllowed ÃƒÂ¢Ã‹â€ Ã‚Â© InstanceEnabled ÃƒÂ¢Ã‹â€ Ã‚Â© CurrentAuthorization`. No caller policy ni defaults; fuentes autoritativas desconectadas permanecen fail-closed.
3. Implementar Tool Calls loop en Engine/runtime sin forzar Bot `ExecutionContext`.
4. Autorizar cada capability antes de ejecutar y usar contexto AgentInstance/tenant validado.
5. Integrar historial CRM (ÃƒÂ¢Ã¢â‚¬Â°Ã‚Â¤20 mensajes user/assistant, fuente CRM; envelope actual no lo transporta).
6. Integrar Rules/Skills respetando autoridad; Skill no concede permiso.
7. Propagar AbortSignal y timeout a provider HTTP y capabilities.
8. Ejecutar la suite automatizada al final del bloque completo, segÃƒÆ’Ã‚Âºn instrucciÃƒÆ’Ã‚Â³n del usuario.
9. DespuÃƒÆ’Ã‚Â©s integrar Drain/Retirement.

Hallazgo fÃƒÆ’Ã‚Â­sico: ExecutionEngine legacy contiene tool-call loop pero estÃƒÆ’Ã‚Â¡ ligado a Bot `ExecutionContext`; CapabilityExecutor construye `CapabilityContext` con `botId`. Provider adapters soportan DTO de ToolCall y AIProviderRequest.tools; el AgentInstanceLLMInvoker actual no envÃƒÆ’Ã‚Â­a tools ni devuelve toolCalls. No se reutilizarÃƒÆ’Ã‚Â¡ el executor legacy directamente ni se habilitarÃƒÆ’Ã‚Â¡n capabilities sin policy autoritativa.

Estado: reconciliaciÃƒÆ’Ã‚Â³n inicial completada; implementaciÃƒÆ’Ã‚Â³n en curso. Local-only; no tests intermedios, commit, push ni deploy.

## 25. Runtime Tool Calls, conversation history y cancellation (20/11/2026)

### 25.1 Engine-owned Tool Call loop

- `ExecutionEngine.executeAgentRuntimeToolLoop<TCall,TResult,TInvocation>` orquesta invocation ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ calls ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ results ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ nueva invocation; lÃƒÆ’Ã‚Â­mite plataforma 5 rounds, cancelaciÃƒÆ’Ã‚Â³n y validaciÃƒÆ’Ã‚Â³n de resultado por llamada.
- AgentInstance permanece fuera de legacy Bot `ExecutionContext`.
- Runtime expone definitions sÃƒÆ’Ã‚Â³lo por intersecciÃƒÆ’Ã‚Â³n del registry runtime y `snapshot.policy.allowedCapabilities`.
- Registry in-memory y executor AgentRuntime separados del CapabilityRegistry legacy. Composition root registra registry vacÃƒÆ’Ã‚Â­o y executor; no se habilitan capabilities Bot automÃƒÆ’Ã‚Â¡ticamente.
- Executor valida esquema JSON bÃƒÆ’Ã‚Â¡sico, autorizaciÃƒÆ’Ã‚Â³n snapshot y ejecuta con AbortSignal/timeout 30s. Implementaciones futuras deben respetar signal y definir idempotencia para side-effects.

### 25.2 Provider tool-call contract

- AgentInstanceLLMInvoker transporta tool definitions, call DTOs y resultados al `AIProviderRequest`.
- Tool calls/responses se serializan causalmente despuÃƒÆ’Ã‚Â©s del turno user para OpenAI-compatible mapper y Gemini.
- JSON arguments invÃƒÆ’Ã‚Â¡lidos fail-closed en mapper OpenAI-compatible.
- Resultado final registra toolCalls/toolResults y agrega usage/cost de las invocations; latency total y providerLatency separados.
- No hay capabilities registradas en AgentRuntime registry todavÃƒÆ’Ã‚Â­a; el loop es infraestructura/orquestaciÃƒÆ’Ã‚Â³n y no habilita acciones de producciÃƒÆ’Ã‚Â³n.

### 25.3 Conversation history causal y scoped

- CRM GraphQL `PaginateEventsInput` aÃƒÆ’Ã‚Â±ade `refConversation` y `beforeMessageId`.
- CRM `Contacto.service.getMessages` aÃƒÆ’Ã‚Â±ade query anterior al cursor, filtrada por contacto + `ref_conversation`, evitando recuperar mensajes de otra conversaciÃƒÆ’Ã‚Â³n y mensajes posteriores al evento causal.
- AI Service CRMConversationGateway aÃƒÆ’Ã‚Â±ade `getMessagesBeforeConversation`; HistoryProvider toma la fuente CRM, mÃƒÆ’Ã‚Â¡ximo 20 mensajes user/assistant, orden cronolÃƒÆ’Ã‚Â³gico.
- Runtime usa `AgentExecution._id` como cursor estable del mensaje que originÃƒÆ’Ã‚Â³ la ejecuciÃƒÆ’Ã‚Â³n; caller-supplied history se rechaza. No hay fallback contact-wide.

### 25.4 Cancellation/timeout

- Provider upper bound plataforma 60s; capability upper bound 30s.
- AbortSignal se propaga al OpenAI-compatible SDK y Gemini `config.abortSignal`.
- Engine revisa signal entre stages/rounds y executor entre calls.
- Timeout de capabilities es cooperativo: la implementaciÃƒÆ’Ã‚Â³n debe honrar signal. No se hace Promise.race sobre side-effects no cancelables.

### 25.5 Rules / Skills / Policy

- Rules existentes se compone ordenadas determinÃƒÆ’Ã‚Â­sticamente, subordinadas a plataforma/AgentDefinition.
- Skill snapshot actual sÃƒÆ’Ã‚Â³lo contiene identidad/config metadata; se presenta como metadata declarativa, no ejecutable ni autorizaciÃƒÆ’Ã‚Â³n.
- Fuentes reales de las cuatro authorities no conectadas; snapshot policy resolver sigue fail-closed y registry runtime vacÃƒÆ’Ã‚Â­o.
- Grounding/output validation determinista y Policy Engine/output filtering aÃƒÆ’Ã‚Âºn pendientes; RAG abstention continÃƒÆ’Ã‚Âºa prompt-level.

### 25.6 ValidaciÃƒÆ’Ã‚Â³n

No se ejecutaron tests ni type-check en este checkpoint por instrucciÃƒÆ’Ã‚Â³n de reservar las pruebas al final de la implementaciÃƒÆ’Ã‚Â³n completa. No E2E, commit, push o deploy. Los cambios CRM/AI Service/docs permanecen locales.

## 26. Workspace Skill content y Drain primitives (20/11/2026)

### 26.1 Workspace Skill snapshot content

- CRM snapshot resolver, para `source=workspace`, exige `ownerId === workspaceId`, consulta `BotResource` por `{workspaceId, skillId}`, `type=skill`, `enabled=true`, descarga desde bucket privado rules-skills y valida el ZIP con parser existente (path/duplicate/zip-bomb/CRC).
- Se extrae `SKILL.md` raÃƒÆ’Ã‚Â­z UTF-8; lÃƒÆ’Ã‚Â­mite platform snapshot 128 KiB. Content se fija en snapshot y participa en hash.
- AI Service HTTP schema + snapshot contract aceptan content acotado. ContextComposer incorpora procedimiento como instrucciones subordinadas; Skill no concede capability.
- `source=waby` rechaza snapshot con `AGENT_RUNTIME_WABY_SKILL_CONTENT_SOURCE_NOT_CONFIGURED`: catÃƒÆ’Ã‚Â¡logo/release inmutable Waby no tiene source fÃƒÆ’Ã‚Â­sico conectado. No fallback metadata-only.

### 26.2 Drain primitives

`AgentExecutionDrainCoordinator` local:
- `blockAdmission(agentInstanceId, workspaceId, retirementOperationId)` delega al repository `block`, retorna `admissionBlockedAt` + generation.
- `confirmDrain(...)` valida identidad/timestamp y verifica bloqueo exacto + count de `AgentExecution.status=pending`; confirma ÃƒÆ’Ã‚Âºnicamente cuando count=0.
- No cancela ejecuciones ya admitidas; permite completion durable. AgentExecution terminal y result Outbox se escriben en transacciÃƒÆ’Ã‚Â³n existente.
- Primitive no estÃƒÆ’Ã‚Â¡ conectado a RetirementMessageConsumer/Outbox confirmation porque WAB-79 C2 (transport/cursor/idempotency) y C3 (physical admission location/atomicity) siguen bloqueantes en contract v3.10. No se inventan routing keys/envelopes.

### 26.3 Limitaciones pendientes

- Policy authorities no conectadas; capability registry vacÃƒÆ’Ã‚Â­o. Tool loop/executor es framework safe, no action capabilities de producciÃƒÆ’Ã‚Â³n.
- Waby standard Skill release source pendiente.
- Grounding/output validation determinista pendiente; abstention es prompt-level.
- Retirement coordination transport + causal Inbox + finalization sigue bloqueado por C2/C3 y C1 topology/transactions segÃƒÆ’Ã‚Âºn WAB-79 physical contract. El coordinator local no equivale a E2E Drain/Retirement.
- No tests/type-check ejecutados; se reservan al final del bloque completo.

## 27. Static validation checkpoint (20/11/2026)

- CRM `pnpm type-check`: PASS after snapshot Workspace Skill resolver, causal conversation-history GraphQL/service and Skill-package extraction.
- AI Service `pnpm --dir microservices/ai-service type-check`: PASS after Engine Tool Call loop, runtime context/history, capability registry/executor, provider Tool Call mapping and AbortSignal wiring.
- Scoped `git diff --check`: PASS for touched files.
- Unit/integration/E2E tests: NOT RUN (user requested defer all tests until complete implementation).
- No commit, push, deploy.

Remaining blockers: authoritative Policy sources and capability registrations; Waby Skill release content source; deterministic grounding/output validator; WAB-79 C2/C3 transport and physical admission integration.


## 28. Handoff a nueva sesiÃƒÆ’Ã‚Â³n ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â siguiente bloque de implementaciÃƒÆ’Ã‚Â³n

**Checkpoint:** 20/11/2026. La sesiÃƒÆ’Ã‚Â³n siguiente debe continuar desde el working tree local; no recrear, revertir ni duplicar los cambios anteriores.

### Estado preservado
- CRM y AI Service mantienen cambios locales no committeados de WAB-81 y otros trabajos existentes. No hacer reset/checkout masivo, no descartar cambios, no stage/commit/push/deploy.
- Runtime AgentInstance conectado al ExecutionEngine con etapas de Knowledge, Context y capability/tool loop.
- Conversation history causal/scoped; Workspace Skill content en snapshot; provider Tool Calls y AbortSignal; Drain primitives locales.
- Type-check CRM y AI Service PASS en el checkpoint anterior. Tests intencionalmente diferidos hasta que la implementaciÃƒÆ’Ã‚Â³n completa estÃƒÆ’Ã‚Â© terminada.

### Siguiente bloque aprobado
Cerrar **Policy Authority Sources + Capability Registry/Execution** antes de grounding y Drain/Retirement integration.

1. Releer WAB-78, WAB-81 y WAB-79 physical contract v3.10; verificar decisiones y no asumir contratos nuevos.
2. Inspeccionar las fuentes fÃƒÆ’Ã‚Â­sicas existentes para platform allowed capabilities, AgentDefinition allowed capabilities, AgentInstance enabled capabilities y autorizaciÃƒÆ’Ã‚Â³n vigente del Workspace/user/operation.
3. Resolver e inyectar esas cuatro authorities al snapshot/resolver sin defaults permisivos; cualquier fuente faltante, inconsistencia o error debe fail-closed.
4. Definir capability contracts/registry y registrations exclusivamente para operaciones de AgentInstance aprobadas por WAB-78/81. No reutilizar capabilities legacy Bot automÃƒÆ’Ã‚Â¡ticamente; no agregar HTTP arbitrario, shell, filesystem ni acciones sin contrato.
5. Implementar el executor/adapters reales para capabilities aprobadas, con validaciÃƒÆ’Ã‚Â³n de inputs, autorizaciÃƒÆ’Ã‚Â³n previa a cada call, tenant/workspace scoping, timeout/cancelaciÃƒÆ’Ã‚Â³n cooperativa, idempotencia para side-effects y resultados serializables.
6. Integrar Rules/Skills/Policy de forma subordinada al authority model. Skill nunca concede capability. Policy no puede elevar permisos.
7. Documentar el bloque y actualizar Linear WAB-81 con checkpoint/bloqueos.

### Fuera de este bloque
- Grounding/output filter determinista.
- Waby Skill release source.
- IntegraciÃƒÆ’Ã‚Â³n fÃƒÆ’Ã‚Â­sica completa de Drain/Retirement; seguir bloqueada hasta resolver WAB-79 C2/C3 (y validar C1 topology/transaction boundaries) sin inventar envelopes/routing.
- Suite de pruebas completa: ejecutar sÃƒÆ’Ã‚Â³lo al final de la implementaciÃƒÆ’Ã‚Â³n completa, segÃƒÆ’Ã‚Âºn instrucciÃƒÆ’Ã‚Â³n del usuario. No obstante, no afirmar que el cÃƒÆ’Ã‚Â³digo estÃƒÆ’Ã‚Â¡ validado por tests; type-check y diff-check pueden usarse como validaciÃƒÆ’Ã‚Â³n estÃƒÆ’Ã‚Â¡tica local si se informa.

### MÃƒÆ’Ã‚Â©todo de trabajo obligatorio en la sesiÃƒÆ’Ã‚Â³n nueva
- Trabajar directamente en la PC/repos locales autorizados mediante Desktop Commander, PowerShell y los repos existentes.
- Inspeccionar antes de modificar; cambios acotados, conservar todas las modificaciones preexistentes.
- No crear archivos descargables ni branches/commits salvo peticiÃƒÆ’Ã‚Â³n expresa.
- Reconciliar contratos/documentaciÃƒÆ’Ã‚Â³n antes de implementar; no reabrir decisiones aprobadas ni duplicar auditorÃƒÆ’Ã‚Â­as.
- Informar checkpoints ÃƒÆ’Ã‚Âºtiles, documentar en este archivo y Linear.
- Pruebas automatizadas completas al final del bloque de implementaciÃƒÆ’Ã‚Â³n, no en cada etapa.



## 29. Checkpoint de sesiÃƒÆ’Ã‚Â³n ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â Policy Authorities + Capability Registry (20/11/2026)

### VerificaciÃƒÆ’Ã‚Â³n inicial
- PC autorizada DESKTOP-G1PG73N conectada; repos locales accesibles.
- Se leyÃƒÆ’Ã‚Â³ Ãƒâ€šÃ‚Â§28 y se confirmÃƒÆ’Ã‚Â³ que coincide con el bloque aprobado.
- Linear WAB-81 continÃƒÆ’Ã‚Âºa In Progress; checkpoint previo actualizado recientemente.
- CRM y waby-docs contienen cambios locales preexistentes extensos. Se preservan sin reset/checkout/staging/commit/push.
- Branch local observada en CRM y ai-service: `waby/reconcile-master-2026-11-19`; no se creÃƒÆ’Ã‚Â³ ni cambiÃƒÆ’Ã‚Â³ branch.

### Contratos reconciliados
- WAB-78 `configuracion-efectiva-y-capacidades.md`: `CapabilitiesEffective = PlatformAllowed ÃƒÂ¢Ã‹â€ Ã‚Â© DefinitionAllowed ÃƒÂ¢Ã‹â€ Ã‚Â© InstanceEnabled ÃƒÂ¢Ã‹â€ Ã‚Â© CurrentAuthorization`. Skills/Rules/prompts/Knowledge no conceden permisos. Ownership y autorizaciÃƒÆ’Ã‚Â³n contextual son independientes.
- WAB-81 snapshot: configuraciÃƒÆ’Ã‚Â³n materializada/hash verificado; la autorizaciÃƒÆ’Ã‚Â³n vigente debe validarse antes de cada ejecuciÃƒÆ’Ã‚Â³n de capability, no confiarse indefinidamente al snapshot.
- WAB-79 RetirementOperation physical contract v3.10: permanece referencia fÃƒÆ’Ã‚Â­sica; C1/C2/C3 siguen bloqueando coordinaciÃƒÆ’Ã‚Â³n/drain/retirement completa. No se tocÃƒÆ’Ã‚Â³ ese bloque.

### InspecciÃƒÆ’Ã‚Â³n fÃƒÆ’Ã‚Â­sica
**Fuentes presentes:** CRM AgentInstance (Workspace, Definition, status, additionalPrompt, Knowledge IDs); AgentRules (enabled opcional); SkillConfiguration (source/owner/config; contenido Workspace resoluble desde almacenamiento privado); AI Service AgentDefinition (prompt, route, defaultSkills/defaultRules) y LLMRoute.

**Fuentes autoritativas faltantes/no verificadas:** Waby Platform capability catalog/mandatory policy; AgentDefinition allowed-capability policy; AgentInstance enabled-capability configuration; current authorization para operaciÃƒÆ’Ã‚Â³n/actor/runtime; AI-specific entitlement/limits source. Generic organization subscription limits y Workspace role permissions no son equivalentes ni se pueden mapear por inferencia. P4 credit authorization es contrato separado.

**Infraestructura existente:** CRM `AgentRuntimePolicyResolver.service.ts` define authority set de cuatro capas, intersecciÃƒÆ’Ã‚Â³n y limits=min, y default fail-closed. `AgentRuntimeSnapshotService` usa `FailClosedAgentRuntimePolicyResolver` por defecto. AI Service tiene contratos/registry/executor de AgentInstance separados de legacy Bot; registry actualmente no establece catÃƒÆ’Ã‚Â¡logo autorizado ni registrations aprobadas. Executor verifica snapshot allowedCapabilities, schema y timeout/cancel, pero no implementa un resolver de autorizaciÃƒÆ’Ã‚Â³n vigente independiente por call.

### Resultado / lÃƒÆ’Ã‚Â­mite de implementaciÃƒÆ’Ã‚Â³n
No se modificÃƒÆ’Ã‚Â³ cÃƒÆ’Ã‚Â³digo productivo durante esta inspecciÃƒÆ’Ã‚Â³n. No se puede conectar ni implementar registrations reales sin fuentes autoritativas y operaciones concretas aprobadas. No reutilizar capabilities legacy Bot, inventar catÃƒÆ’Ã‚Â¡logo paralelo, agregar defaults permisivos, ni aceptar policy del caller. Mantener fail-closed.

### Siguiente acciÃƒÆ’Ã‚Â³n
Concretar el mÃƒÆ’Ã‚Â­nimo contrato/owner necesario para materializar las cuatro authorities y current authorization; verificar si los recursos de integraciones/capabilities ya tienen fuente fÃƒÆ’Ã‚Â­sica en CRM u otro repositorio autorizado. Si no existen, documentar la brecha como decisiÃƒÆ’Ã‚Â³n/implementaciÃƒÆ’Ã‚Â³n bloqueante en WAB-81 y solicitar definiciÃƒÆ’Ã‚Â³n mÃƒÆ’Ã‚Â­nima antes de introducir modelos. Registry/executor especÃƒÆ’Ã‚Â­ficos de AgentInstance permanecen como scaffold seguro hasta que haya capabilities aprobadas.

**Verificaciones:** inspecciÃƒÆ’Ã‚Â³n y lectura documental solamente. No tests, build, type-check, lint ni cambios de cÃƒÆ’Ã‚Â³digo en este checkpoint.



## 30. ReconciliaciÃƒÆ’Ã‚Â³n documental ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â cuatro authorities de Capability (20/11/2026)

### Resultado de bÃƒÆ’Ã‚Âºsqueda en waby-docs y Linear
Se revisaron: configuracion-efectiva-y-capacidades.md, fase-2-diseno-objetivo-y-contratos.md, plan-administracion-catalogo-waby.md, informes de auditorÃƒÆ’Ã‚Â­a WAB-78, continuidad WAB-81; Linear WAB-78, WAB-80, WAB-22, WAB-88 y WAB-91.

### Contratos ya existentes
1. **Platform Authority:** Waby controla polÃƒÆ’Ã‚Â­ticas obligatorias, catÃƒÆ’Ã‚Â¡logo global, conectores/operaciones soportados, lÃƒÆ’Ã‚Â­mites y revocaciÃƒÆ’Ã‚Â³n de seguridad. Waby puede revocar capabilities en el punto de autorizaciÃƒÆ’Ã‚Â³n, aun con snapshot previo.
2. **Definition Authority:** AgentDefinition administrada por Waby define el mÃƒÆ’Ã‚Â¡ximo conjunto de capabilities/funcionalidades compatibles. No inferir grants desde defaultSkills, Skills, Rules, prompts o Knowledge.
3. **Instance Authority:** configuraciÃƒÆ’Ã‚Â³n privada de AgentInstance/AgentInstanceIntegration define conexiones y operaciones habilitadas. Usuario autorizado del Workspace configura/revoca; credenciales son referencias protegidas; SkillConfiguration no almacena secretos. Contrato conceptual AgentInstanceIntegration existe en WAB-79, pero su implementaciÃƒÆ’Ã‚Â³n fÃƒÆ’Ã‚Â­sica sigue diferida. No asumir nombres de campos/cardinalidad.
4. **CurrentAuthorization:** documentaciÃƒÆ’Ã‚Â³n define autorizaciÃƒÆ’Ã‚Â³n contextual independiente y valida OrganizationÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢WorkspaceÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢AgentInstance, conexiÃƒÆ’Ã‚Â³n, operaciÃƒÆ’Ã‚Â³n, recurso y revocaciÃƒÆ’Ã‚Â³n. ARQ-20 wrk.agents.* / wrk.agents.read autoriza administraciÃƒÆ’Ã‚Â³n de AgentInstances, no execution grants. JWT RS256 ARQ-21 autentica servicios, no autoriza capability. P4 budget auth no es capability auth.

### Contrato tÃƒÆ’Ã‚Â©cnico propuesto para WAB-81
- Effective capability eligibility = intersection PlatformAllowed ÃƒÂ¢Ã‹â€ Ã‚Â© DefinitionAllowed ÃƒÂ¢Ã‹â€ Ã‚Â© InstanceEnabled ÃƒÂ¢Ã‹â€ Ã‚Â© CurrentAuthorization.
- Platform/Definition/Instance config pueden materializarse y quedar referenciadas en snapshot. CurrentAuthorization es un gate vigente que se consulta antes de cada operaciÃƒÆ’Ã‚Â³n y puede revocar/denegar una operaciÃƒÆ’Ã‚Â³n durante ejecuciÃƒÆ’Ã‚Â³n.
- Capability Authorization Gate dedicado al runtime, sin nuevo permission enum por inferencia. Debe verificar service identity, validated org/workspace/instance ownership, instance status, platform revocation, definition compatibility, operation enabled in instance integration, connection/credential status, resource scoping y authorization vigente.
- Las credenciales nunca se exponen al LLM, snapshot, tool arguments ni logs. No HTTP arbitrario, shell, filesystem ni capability genÃƒÆ’Ã‚Â©rica.
- WAB-80 Workspace permissions administran configuraciÃƒÆ’Ã‚Â³n; no se delega al AgentInstance identidad de usuario humano.

### Referencias Linear
- WAB-78 (Done): contrato de configuraciÃƒÆ’Ã‚Â³n efectiva/capabilities y autoridad Waby/Workspace.
- WAB-80 (Backlog): provisioning y permisos wrk.agents.*; distingue administraciÃƒÆ’Ã‚Â³n de instancia y operaciÃƒÆ’Ã‚Â³n.
- WAB-22 (Done): permisos de Workspace declarados para whoAmI.
- WAB-88 (Done): auditorÃƒÆ’Ã‚Â­a de autorizaciÃƒÆ’Ã‚Â³n GraphQL/interna; evidencia de controles efectivos.
- WAB-91 (auditorÃƒÆ’Ã‚Â­a): routing, aislamiento y seguridad runtime; exige verificar filtering y tenant context.

### PrÃƒÆ’Ã‚Â³ximo paso
Inspeccionar los repositorios locales/organizaciÃƒÆ’Ã‚Â³n autorizados para localizar fÃƒÆ’Ã‚Â­sicamente Waby catalog/policy y AgentInstanceIntegration/credential authorization. Mapear las autoridades existentes al contrato anterior sin duplicarlas. Si las fuentes no existen, registrar implementaciÃƒÆ’Ã‚Â³n mÃƒÆ’Ã‚Â­nima bloqueante en WAB-81. Esta reconciliaciÃƒÆ’Ã‚Â³n no modifica cÃƒÆ’Ã‚Â³digo ni declara las authorities fÃƒÆ’Ã‚Â­sicamente implementadas.

**ValidaciÃƒÆ’Ã‚Â³n:** bÃƒÆ’Ã‚Âºsqueda documental/Linear e inspecciÃƒÆ’Ã‚Â³n; no tests, build, type-check ni lint.



## 31. InvestigaciÃƒÆ’Ã‚Â³n fÃƒÆ’Ã‚Â­sica y plan de conexiÃƒÆ’Ã‚Â³n de authorities (20/11/2026)

### InspecciÃƒÆ’Ã‚Â³n adicional de repositorios locales
Se inspeccionaron CRM, ai-service, infraestructura y WABY_frontend dentro de F:\\ElemenSoft; bÃƒÆ’Ã‚Âºsquedas de cÃƒÆ’Ã‚Â³digo en CRM/ai-service y revisiÃƒÆ’Ã‚Â³n de archivos de runtime, modelos y middleware.

**Evidencia fÃƒÆ’Ã‚Â­sica:**
- CRM `src/services/AgentRuntimePolicyResolver.service.ts`: existen `IAgentRuntimePolicyAuthoritySource`, `AgentRuntimePolicyAuthoritySet`, resolver de intersecciÃƒÆ’Ã‚Â³n de cuatro conjuntos y min de lÃƒÆ’Ã‚Â­mites; resolver default fail-closed.
- CRM `src/services/agent-runtime-snapshot.service.ts`: resuelve ownership OrganizationÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢Workspace, AgentInstance activa, Rules y Workspace Skills. Invoca policyResolver; default es fail-closed. No existe wiring fÃƒÆ’Ã‚Â­sico de fuentes de authorities en esta clase.
- AI Service `src/infrastructure/database/models/agent-definition.model.ts`: AgentDefinition tiene prompt, LLM route, defaultSkills/defaultRules, status y lifecycle; no existe allowed-capability policy explÃƒÆ’Ã‚Â­cita.
- AI Service `src/presentation/http/routes/agent-runtime.routes.ts`: snapshot endpoint protegido por service JWT scope `agent.runtime.snapshot.resolve`; valida Definition activa y LLMRoute. El service JWT autentica CRM, no constituye CurrentAuthorization de ejecuciÃƒÆ’Ã‚Â³n de capability.
- AI Service `src/presentation/http/middleware/service-jwt-auth.middleware.ts`: RS256, issuer/audience, scope y replay guard; esto es autenticaciÃƒÆ’Ã‚Â³n/autorizaciÃƒÆ’Ã‚Â³n de servicio para endpoint, no grants de herramientas.
- AI Service `src/application/services/agent-runtime-capabilities.ts`, `agent-runtime-capability-registry.ts`, `agent-runtime-capability-executor.ts`: contrato y registry/executor especÃƒÆ’Ã‚Â­ficos de AgentInstance, separados de legacy Bot. Bootstrap vacÃƒÆ’Ã‚Â­o y fail-closed; executor valida allowedCapabilities del snapshot, schema y timeout/cancel, pero no resuelve autorizaciÃƒÆ’Ã‚Â³n live independiente.
- Legacy AI Service `src/infrastructure/capabilities/*` y composiciÃƒÆ’Ã‚Â³n de ExecutionEngine: capabilities count/get/search contacts, get-last-messages y assign-contact se registran para Bot legacy. No se deben registrar, exponer ni adaptar automÃƒÆ’Ã‚Â¡ticamente a AgentInstance. Son evidencia de servicios/gateways potencialmente reutilizables sÃƒÆ’Ã‚Â³lo tras revisiÃƒÆ’Ã‚Â³n de dominio, tenancy, permiso, resource scoping y contrato de operaciÃƒÆ’Ã‚Â³n.
- No se localizÃƒÆ’Ã‚Â³ en el alcance inspeccionado una entidad/modelo fÃƒÆ’Ã‚Â­sico de AgentInstanceIntegration, conexiÃƒÆ’Ã‚Â³n, credential reference o enabled operation que sirva como Instance Authority. Tampoco se verificÃƒÆ’Ã‚Â³ un catÃƒÆ’Ã‚Â¡logo/policy central Waby ni un grant de capability por AgentDefinition. No se concluye que no existan fuera del workspace/repositories inspeccionados.

### Mapa de estado de las cuatro authorities
| Authority | Contrato | Fuente fÃƒÆ’Ã‚Â­sica verificada | Estado |
|---|---|---|---|
| PlatformAllowed | Waby catÃƒÆ’Ã‚Â¡logo/polÃƒÆ’Ã‚Â­tica, lÃƒÆ’Ã‚Â­mites, revocaciÃƒÆ’Ã‚Â³n | No localizada en alcance | Bloqueada |
| DefinitionAllowed | MÃƒÆ’Ã‚Â¡ximo permitido por AgentDefinition | Modelo AgentDefinition sin capability policy | Bloqueada |
| InstanceEnabled | ConfiguraciÃƒÆ’Ã‚Â³n integration/connection/operation por instancia | No localizada; AgentInstanceIntegration conceptual | Bloqueada |
| CurrentAuthorization | Gate live contextual, operaciÃƒÆ’Ã‚Â³n y recurso | Service JWT y auth CRM existentes, pero no grant de capability; no hay gate runtime conectado | Bloqueada |

### Plan de implementaciÃƒÆ’Ã‚Â³n ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â no iniciar registrations hasta resolver fuentes
**Fase 0 ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â Source inventory y ownership (actual):** ampliar inspecciÃƒÆ’Ã‚Â³n de repositorios ElemeSoft disponibles y docs/Linear para identificar owner fÃƒÆ’Ã‚Â­sico, API/modelo y lifecycle de catÃƒÆ’Ã‚Â¡logo, Definition grants, integration connections y live authorization. Confirmar si alguna fuente estÃƒÆ’Ã‚Â¡ en repositorios no clonados o servicios Waby fuera del workspace local.

**Fase 1 ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â Contratos y decisiones bloqueantes:** registrar fuente/owner para cada autoridad, IDs estables, estados/revocaciÃƒÆ’Ã‚Â³n, lÃƒÆ’Ã‚Â­mites, scopes de recursos, conexiÃƒÆ’Ã‚Â³n y credenciales protegidas. Si alguna fuente no existe, proponer el modelo mÃƒÆ’Ã‚Â­nimo al owner y obtener aprobaciÃƒÆ’Ã‚Â³n; no crear campos/modelos especulativos ni aceptar policy enviada por caller.

**Fase 2 ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â Adapter de Authorities:** implementar un ÃƒÆ’Ã‚Âºnico `IAgentRuntimePolicyAuthoritySource` con adapters a las fuentes verificadas. Cada capa debe distinguir unavailable/invalid/denied y fallar cerrada. Resolver intersecciÃƒÆ’Ã‚Â³n de capabilities y lÃƒÆ’Ã‚Â­mites min con el resolver existente; eliminar cualquier default permisivo. Snapshot sÃƒÆ’Ã‚Â³lo materializa authorities estables y sus revisiones/identidad; current authorization debe consultarse live por operaciÃƒÆ’Ã‚Â³n.

**Fase 3 ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â Capability Authorization Gate:** gate previo a cada call, que recibe identidad validada del execution envelope/context y capability + parÃƒÆ’Ã‚Â¡metros/resource scope. Revalidar Organization/Workspace/AgentInstance ownership, status, platform revocation, Definition compatibility, instance operation enablement, connection status y autorizaciÃƒÆ’Ã‚Â³n contextual vigente. No confiar sÃƒÆ’Ã‚Â³lo en snapshot, nombre de tool o service JWT. Credential secrets no llegan al LLM, snapshot, tool arguments ni logs.

**Fase 4 ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â Capability catalog/registry AgentInstance:** catÃƒÆ’Ã‚Â¡logo explÃƒÆ’Ã‚Â­cito de operaciones aprobadas con ID estable, schema de parÃƒÆ’Ã‚Â¡metros/results, resource scope, side-effect classification, lÃƒÆ’Ã‚Â­mites, timeout/cancel, idempotency requirements y adapter owner. Registry sÃƒÆ’Ã‚Â³lo registra capabilities de AgentInstance. Definition/instance config no puede ampliar Platform catalog; Skills/Rules/prompts/Knowledge no otorgan permisos. No HTTP arbitrario, shell, filesystem ni capability genÃƒÆ’Ã‚Â©rica.

**Fase 5 ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â Primeras capabilities reales:** elegir operaciones read-only y de bajo riesgo que tengan contratos y gateways CRM existentes verificados, despuÃƒÆ’Ã‚Â©s de revisiÃƒÆ’Ã‚Â³n individual. Implementar adapters nuevos del runtime AgentInstance hacia servicios/domain ports autorizados; no copiar ni registrar legacy Bot capabilities. Cada capability debe tener autorizaciÃƒÆ’Ã‚Â³n live, workspace/resource scoping, validaciÃƒÆ’Ã‚Â³n estricta, resultado serializable, redacciÃƒÆ’Ã‚Â³n de errores, timeout y observabilidad. Side-effects sÃƒÆ’Ã‚Â³lo en una fase posterior con idempotencia y auditorÃƒÆ’Ã‚Â­a contractual.

**Fase 6 ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â Snapshot/runtime wiring:** conectar source ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ resolver ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ snapshot policy; registrar sÃƒÆ’Ã‚Â³lo capabilities compatibles con snapshot y catÃƒÆ’Ã‚Â¡logo. Antes de cada tool call ejecutar gate live. Rechazo/no source/no connection/no resource authorization produce denial, sin fallback. Confirmar que el LLM sÃƒÆ’Ã‚Â³lo ve tools elegibles.

**Fase 7 ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â ValidaciÃƒÆ’Ã‚Â³n y cierre:** al terminar implementaciÃƒÆ’Ã‚Â³n completa, correr suite automatizada segÃƒÆ’Ã‚Âºn instrucciÃƒÆ’Ã‚Â³n del usuario; antes, static type-check/diff-check sÃƒÆ’Ã‚Â³lo si necesario y reportar explÃƒÆ’Ã‚Â­citamente. Auditar aislamiento multi-tenant, revocation, concurrent changes, failure modes y logs. Actualizar WAB-81 y docs con evidencia; no commit/push/deploy sin autorizaciÃƒÆ’Ã‚Â³n.

### Criterios de salida
- Cada authority tiene fuente fÃƒÆ’Ã‚Â­sica y owner documentados; no hay source caller-supplied ni default grant.
- Resolver integrado al snapshot con intersecciÃƒÆ’Ã‚Â³n/min limits y fail-closed.
- CurrentAuthorization se revalida por operaciÃƒÆ’Ã‚Â³n.
- Registry AgentInstance sÃƒÆ’Ã‚Â³lo contiene registrations explÃƒÆ’Ã‚Â­citamente aprobadas; legacy Bot permanece aislado.
- Primeras capabilities reales cuentan con contratos, adapter y autorizaciÃƒÆ’Ã‚Â³n/tenancy verificadas.
- No se expone credential secret; no se acepta capability arbitraria.

### Estado de este checkpoint
InvestigaciÃƒÆ’Ã‚Â³n e inventario; no se modificÃƒÆ’Ã‚Â³ cÃƒÆ’Ã‚Â³digo productivo, no se ejecutaron tests/build/type-check/lint, no commit/push/deploy. El plan identifica dependencias bloqueantes y no declara las authorities conectadas.


## 32. AmpliaciÃƒÆ’Ã‚Â³n del inventario fÃƒÆ’Ã‚Â­sico ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â workspace y repositorios ElemeSoft (20/11/2026)

### Workspace local
Se enumeraron los directorios de primer nivel en `F:\\ElemenSoft`. Repositorios/proyectos presentes: `crm`, `WABY_frontend`, `infrastructure`, `waby-docs` y `wabyland`. TambiÃƒÆ’Ã‚Â©n existen artefactos y documentaciÃƒÆ’Ã‚Â³n de revisiÃƒÆ’Ã‚Â³n WAB-78/WAB-79 en la raÃƒÆ’Ã‚Â­z, que no se consideraron fuentes de runtime por sÃƒÆ’Ã‚Â­ solos.

### Repositorios visibles mediante GitHub connector
Se consultÃƒÆ’Ã‚Â³ el inventario de repositorios accesibles de la organizaciÃƒÆ’Ã‚Â³n ElemeSoft. AdemÃƒÆ’Ã‚Â¡s de los repositorios locales, estÃƒÆ’Ã‚Â¡n visibles: `eleme-web`, `Tareas`, `MIX`, `AG_TickeraMIX_frontend`, `waby-bot`, `waby-Baileys`, `waby-evolution`. Esta enumeraciÃƒÆ’Ã‚Â³n no implica que todos sean parte del runtime WABY ni que hayan sido inspeccionados. `waby-bot` y `waby-evolution` requieren investigaciÃƒÆ’Ã‚Â³n de su responsabilidad/contratos antes de considerarlos fuentes.

### BÃƒÆ’Ã‚Âºsqueda fÃƒÆ’Ã‚Â­sica adicional
- CRM `src`: bÃƒÆ’Ã‚Âºsqueda de `AgentInstanceIntegration`, `ConnectorDefinition`, `IntegrationInvocation`, `credentialRef`, `externalIntegration` y `integrationConnection`: sin coincidencias.
- CRM `src`: bÃƒÆ’Ã‚Âºsqueda de archivos con nombres de capability/integration/credential/connector: sin resultados relevantes.
- AI Service: coincidencias de credenciales pertenecen a `LLMProviderAccount`/`LLMRoute` y al resolver de credenciales LLM. Es un dominio Waby-owned de proveedor LLM, no credenciales empresariales del Workspace; no reutilizar como fuente de Instance Authority.
- WABY_frontend: bÃƒÆ’Ã‚Âºsqueda no encontrÃƒÆ’Ã‚Â³ modelos/flows de AgentInstanceIntegration o capability. Las coincidencias de `credential` fueron tests de contratos REST y un mensaje de AddAgentForm, sin evidencia de configuraciÃƒÆ’Ã‚Â³n de integraciones empresariales.
- infrastructure: bÃƒÆ’Ã‚Âºsqueda de connector/integration/credential vault/secret manager en archivos de configuraciÃƒÆ’Ã‚Â³n/documentaciÃƒÆ’Ã‚Â³n: sin coincidencias. Esto no descarta infraestructura externa no declarada en el repositorio.
- waby-docs contiene el diseÃƒÆ’Ã‚Â±o conceptual `design/integraciones-api-externas.md`: ConnectorDefinition (Waby), AgentInstanceIntegration (Workspace+AgentInstance), AgentInstanceOperation e IntegrationInvocation. Ownership, referencia protegida, no HTTP arbitrario, secreto fuera de prompt/snapshot/logs y revocaciÃƒÆ’Ã‚Â³n urgente estÃƒÆ’Ã‚Â¡n documentados; el documento declara explÃƒÆ’Ã‚Â­citamente que no define modelo fÃƒÆ’Ã‚Â­sico final ni autoriza implementaciÃƒÆ’Ã‚Â³n.

### Inventario de autoridad actualizado
| Authority | Evidencia fÃƒÆ’Ã‚Â­sica/documental adicional | Estado |
|---|---|---|
| PlatformAllowed | DiseÃƒÆ’Ã‚Â±o conceptual en waby-docs; catÃƒÆ’Ã‚Â¡logo fÃƒÆ’Ã‚Â­sico no localizado en repositorios locales revisados | Bloqueada |
| DefinitionAllowed | Modelo AgentDefinition fÃƒÆ’Ã‚Â­sico no incluye capability policy explÃƒÆ’Ã‚Â­cita; contrato WAB-78 documenta autoridad de Definition | Bloqueada |
| InstanceEnabled | DiseÃƒÆ’Ã‚Â±o conceptual AgentInstanceIntegration/Operation; sin modelo, API o UI fÃƒÆ’Ã‚Â­sica encontrada en CRM/frontend/AI Service local | Bloqueada |
| CurrentAuthorization | Auth CRM/Service JWT existentes, pero no capability grant gate live; sin source fÃƒÆ’Ã‚Â­sico identificado | Bloqueada |

### ConclusiÃƒÆ’Ã‚Â³n y siguiente investigaciÃƒÆ’Ã‚Â³n
La bÃƒÆ’Ã‚Âºsqueda local ampliada no encontrÃƒÆ’Ã‚Â³ una fuente fÃƒÆ’Ã‚Â­sica de integraciones/capabilities. La documentaciÃƒÆ’Ã‚Â³n conceptual de WAB-78 sÃƒÆ’Ã‚Â­ define ownership y lÃƒÆ’Ã‚Â­mites, pero no debe convertirse directamente en modelos de persistencia sin contrato tÃƒÆ’Ã‚Â©cnico aprobado. El inventario GitHub muestra repositorios adicionales no clonados localmente, especialmente `waby-evolution` y `waby-bot`; deben inspeccionarse por responsabilidad y cÃƒÆ’Ã‚Â³digo relevante antes de concluir que las fuentes estÃƒÆ’Ã‚Â¡n ausentes en toda la organizaciÃƒÆ’Ã‚Â³n. No se encontrÃƒÆ’Ã‚Â³ evidencia que autorice reutilizar legacy Bot capability registry.

**Checkpoint:** investigaciÃƒÆ’Ã‚Â³n de workspace y lista de repositorios accesibles; no se modificÃƒÆ’Ã‚Â³ cÃƒÆ’Ã‚Â³digo productivo ni se ejecutaron tests/build/type-check/lint; no commit/push/deploy.


## 33. CorrecciÃƒÆ’Ã‚Â³n de alcance de repositorios (20/11/2026)

El usuario aclarÃƒÆ’Ã‚Â³ que:
- `ElemeSoft/waby-evolution` es un fork de Evolution API, no parte del runtime WABY Agents ni fuente de capabilities para WAB-81.
- `ElemeSoft/waby-bot` es un proyecto abandonado/no utilizado y no forma parte de la arquitectura en desarrollo.

Se retiran ambos repositorios del inventario de candidatos para authorities/capabilities y no se investigarÃƒÆ’Ã‚Â¡n ni reutilizarÃƒÆ’Ã‚Â¡n para este trabajo. La investigaciÃƒÆ’Ã‚Â³n debe centrarse en los repositorios y servicios que sÃƒÆ’Ã‚Â­ forman parte de WABY: CRM/AI Service, WABY_frontend, infrastructure y documentaciÃƒÆ’Ã‚Â³n canÃƒÆ’Ã‚Â³nica waby-docs, ademÃƒÆ’Ã‚Â¡s de cualquier otro repositorio que el usuario confirme como vigente y relevante.

La conclusiÃƒÆ’Ã‚Â³n de Ãƒâ€šÃ‚Â§32 sobre estos dos repositorios queda corregida por esta secciÃƒÆ’Ã‚Â³n. No cambia el estado bloqueado de las cuatro authorities ni autoriza implementaciÃƒÆ’Ã‚Â³n especulativa.


## 34. InvestigaciÃƒÆ’Ã‚Â³n dirigida de autorizaciÃƒÆ’Ã‚Â³n CRM y modelos AgentInstance (20/11/2026)

### Evidencia fÃƒÆ’Ã‚Â­sica confirmada
- `crm/src/models/agent/AgentInstance.model.ts`: AgentInstance tiene `agentDefinitionId`, `workspaceId`, status, prompt suplementario, knowledgeBaseIds y lifecycle; no contiene integrations, enabled operations, credential refs ni capability grants.
- `crm/src/auth/permisions/permission.enum.ts`: existen `wrk.agents.read` y `wrk.agents.*` (ARQ-20).
- `crm/src/utils/autoloader/tipAccessWorks.roles.ts`: el rol Manager incluye `wrk.agents.*`.
- Estas permissions son administraciÃƒÆ’Ã‚Â³n/configuraciÃƒÆ’Ã‚Â³n de AgentInstances en Workspace. No equivalen a autorizaciÃƒÆ’Ã‚Â³n live para que una capability del runtime acceda a un recurso externo y no deben usarse como CurrentAuthorization.
- CRM contiene servicios y middleware de autorizaciÃƒÆ’Ã‚Â³n de Workspace, pero la bÃƒÆ’Ã‚Âºsqueda dirigida no identificÃƒÆ’Ã‚Â³ un grant model/authorization service especÃƒÆ’Ã‚Â­fico de capability runtime. No se concluye que el mecanismo general sea insuficiente para adaptarse; debe evaluarse su contract/context y ownership antes de diseÃƒÆ’Ã‚Â±ar otro.

### ImplicaciÃƒÆ’Ã‚Â³n
La fuente de InstanceEnabled sigue sin identificarse. AgentInstance model confirma que no existe en ese agregado. CurrentAuthorization debe tener un gate dedicado o adapter a un mecanismo existente, tras definir contexto de ejecuciÃƒÆ’Ã‚Â³n, scopes y owner. No se crea permiso nuevo ni se usa `wrk.agents.*` como grant de tool execution.

### Estado
InspecciÃƒÆ’Ã‚Â³n de modelos/permisos y bÃƒÆ’Ã‚Âºsqueda; sin cÃƒÆ’Ã‚Â³digo modificado, tests/build/type-check/lint, commit/push/deploy.


## 35. EvaluaciÃƒÆ’Ã‚Â³n del mecanismo general de autorizaciÃƒÆ’Ã‚Â³n CRM (20/11/2026)

### Componentes inspeccionados
- `src/graphql/Auth/AuthChecker.ts`: evalÃƒÆ’Ã‚Âºa permisos declarados en resolvers; requiere usuario autenticado, roles/contexto de organizaciÃƒÆ’Ã‚Â³n y workspace; para permisos `wrk.*` llama a `assertWrkAccess`; evalÃƒÆ’Ã‚Âºa restricciones de membresÃƒÆ’Ã‚Â­a antes de conceder.
- `src/graphql/Auth/WrkGuard.ts`: `assertWrkAccess` valida `x-wrk-id`, usuario/roles y membresÃƒÆ’Ã‚Â­as activas `uowActive`/`uoActive`, consulta `UsOrgsWorksModel`, compara workspace de sesiÃƒÆ’Ã‚Â³n y organizaciÃƒÆ’Ã‚Â³n activa. Es un guard de sesiÃƒÆ’Ã‚Â³n humana/Workspace.
- `src/auth/hasPermission.ts`: semÃƒÆ’Ã‚Â¡ntica pura de permisos de organizaciÃƒÆ’Ã‚Â³n y Workspace, wildcards y restricciones por namespace.
- `src/auth/resolveAuthContext.ts`: resuelve JWT de usuario, sesiÃƒÆ’Ã‚Â³n vigente, membresÃƒÆ’Ã‚Â­a de organizaciÃƒÆ’Ã‚Â³n y Workspace activa; agrega roles, restricciones y scope. No se cachea deliberadamente para que revocaciones de roles no permanezcan vigentes por cache.
- `src/graphql/context/basicContext.ts`: contexto basado en user id, orgRole, wrkRole, membresÃƒÆ’Ã‚Â­a activa, restricciones y alcance de contactos.
- `src/graphql/Auth/OrgGuard.ts`: valida organizaciÃƒÆ’Ã‚Â³n activa/contexto, pero explÃƒÆ’Ã‚Â­citamente no valida ownership del recurso; ownership se comprueba por operaciÃƒÆ’Ã‚Â³n.

### ReutilizaciÃƒÆ’Ã‚Â³n: quÃƒÆ’Ã‚Â© sÃƒÆ’Ã‚Â­ y quÃƒÆ’Ã‚Â© no
**No es un gate reutilizable directamente para CurrentAuthorization.** Es un mecanismo de autorizaciÃƒÆ’Ã‚Â³n de usuario/membresÃƒÆ’Ã‚Â­a con contexto GraphQL/HTTP, headers `x-org-id`/`x-wrk-id`, roles humanos y restricciones de sesiÃƒÆ’Ã‚Â³n. El runtime AI Service ejecuta con contexto service-to-service y envelope; no debe simular usuario humano ni depender de headers activos.

**Posible adaptaciÃƒÆ’Ã‚Â³n limitada:** el resolver de membership/Workspace y comprobaciones de tenant/ownership pueden servir como primitives o adapter si una capability requiere actuar en nombre de un usuario autorizado. Esto exige contrato explÃƒÆ’Ã‚Â­cito de principal/actor, orgId, workspaceId, AgentInstance, executionId, capability/operation, resource/connection scope y revocation semantics. No se ha identificado todavÃƒÆ’Ã‚Â­a una fuente persistente de grants para capabilities ni se confirma que los roles actuales concedan acceso a integraciones.

### Hallazgos de seguridad relevantes
- `assertWrkAccess` verifica sesiÃƒÆ’Ã‚Â³n/membresÃƒÆ’Ã‚Â­a activa y cruce de org/workspace, pero no autoriza una operaciÃƒÆ’Ã‚Â³n de capability ni un recurso externo especÃƒÆ’Ã‚Â­fico.
- `hasPermission` es evaluaciÃƒÆ’Ã‚Â³n de permisos administrativos `org.*`/`wrk.*`; `wrk.agents.*` no debe reinterpretarse como capability grant.
- `resolveAuthContext` no cachea roles/membresÃƒÆ’Ã‚Â­as y realiza agregaciÃƒÆ’Ã‚Â³n por request, una propiedad favorable a revocaciÃƒÆ’Ã‚Â³n humana; el runtime necesitarÃƒÆ’Ã‚Â¡ una polÃƒÆ’Ã‚Â­tica equivalente de freshness/revocaciÃƒÆ’Ã‚Â³n, pero su mecanismo queda por definir.
- `OrgGuard` documenta que no prueba ownership del recurso. El gate runtime debe comprobar tenant, Instance, integraciÃƒÆ’Ã‚Â³n/conexiÃƒÆ’Ã‚Â³n, operation y resource scope, no asumir ownership por el guard.
- No se debe pasar el contexto humano GraphQL al AI Service ni convertir el JWT de servicio en autoridad de capability.

### DecisiÃƒÆ’Ã‚Â³n de investigaciÃƒÆ’Ã‚Â³n
No adaptar ni duplicar todavÃƒÆ’Ã‚Â­a `AuthChecker`, `WrkGuard` o `hasPermission`. Son candidatos a primitives de tenant/membership en un futuro adapter, pero CurrentAuthorization requiere contrato runtime explÃƒÆ’Ã‚Â­cito y fuente de autorizaciÃƒÆ’Ã‚Â³n live. La falta de fuente persistente/grants especÃƒÆ’Ã‚Â­ficos mantiene bloqueadas PlatformAllowed, DefinitionAllowed, InstanceEnabled y CurrentAuthorization.

### Estado
InspecciÃƒÆ’Ã‚Â³n de autorizaciÃƒÆ’Ã‚Â³n CRM y contexto. Sin cambios de cÃƒÆ’Ã‚Â³digo productivo ni ejecuciÃƒÆ’Ã‚Â³n de tests/build/type-check/lint; sin commit/push/deploy.


## 36. Flujo fÃƒÆ’Ã‚Â­sico CRM ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ AI Service ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ ExecutionEngine y lÃƒÆ’Ã‚Â­mites de autorizaciÃƒÆ’Ã‚Â³n (20/11/2026)

### Flujo reconstruido
1. `AgentMessageOutboxService.enqueue()` persiste evento durable `agent.message.received`; no resuelve assignment ni policy.
2. `publishAgentMessageReceived()` valida payload, `InboundAgentAssignmentResolver.resolve()` verifica contacto/Workspace, asignaciÃƒÆ’Ã‚Â³n causal `EvAssigned` y AgentInstance activa/propia.
3. `AgentRuntimeSnapshotService.resolve()` valida ownership orgÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬ÂWorkspace, instancia activa/Workspace, Rules y Skills; consulta `IAgentRuntimePolicyResolver`. Default es `FailClosedAgentRuntimePolicyResolver`, que falla si no se configura autoridad.
4. Snapshot CRMÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢AI Service vÃƒÆ’Ã‚Â­a Service JWT scope `agent.runtime.snapshot.resolve`. AI Service valida AgentDefinition activa y LLMRoute y devuelve snapshot/hash. No verifica autorizaciÃƒÆ’Ã‚Â³n de capability live.
5. CRM publica envelope a RabbitMQ exchange `ai-service.events`, routing `agent.message.received`; contiene tenant, assignment, mensaje y snapshot.
6. Consumer AI Service valida schema e identidades tenant/instance/definition; admission transaccional crea execution+inbox y toma secuencia del `AgentInstanceExecutionControl` open. Persist snapshot/envelope. Duplicate exige identidad y payload hash consistentes.
7. Inbox worker toma lease, marca startedAt y llama `AgentInstanceExecutionRuntime.execute()`.
8. Runtime valida snapshot identidad/hash, carga historial causal por workspace/contact/conversation/execution; retrieve Knowledge; compone contexto; expone registry capabilities cuya name estÃƒÆ’Ã‚Â¡ en `snapshot.policy.allowedCapabilities`; ejecuta Tool Loop dentro de `ExecutionEngine`.
9. Por cada tool call, runtime valida que la capability estÃƒÆ’Ã‚Â© en registry y snapshot allowlist, luego executor valida schema/timeout/cancel y llama `capability.execute()`. No existe CurrentAuthorization live gate entre selecciÃƒÆ’Ã‚Â³n y ejecuciÃƒÆ’Ã‚Â³n ni se deriva actor/connection/resource scope de un contrato autorizado.
10. Worker persiste resultado y publica durable execution-result outbox.

### Hallazgos crÃƒÆ’Ã‚Â­ticos de autorizaciÃƒÆ’Ã‚Â³n
- El snapshot policy es capturado en CRM antes de publicar el evento y queda persistido en execution. Su hash protege integridad/config coherente, no frescura ni autorizaciÃƒÆ’Ã‚Â³n actual.
- `allowedCapabilities` sÃƒÆ’Ã‚Â³lo limita tool exposure y una segunda comprobaciÃƒÆ’Ã‚Â³n local. No representa las cuatro authorities de manera verificable porque resolver default fail-closed y no existe fuente fÃƒÆ’Ã‚Â­sica conectada.
- Registry runtime estÃƒÆ’Ã‚Â¡ vacÃƒÆ’Ã‚Â­o por default; executor default fail-closed. Esto evita ejecuciones legacy implÃƒÆ’Ã‚Â­citas, pero al registrar capability futura no provee autorizaciÃƒÆ’Ã‚Â³n live.
- Capability executor recibe `AgentRuntimeContext` + ToolCall; el contexto actual no acredita principal/actor, integration connection, operation-specific grant, resource scope ni revocation version.
- Envelope/Execution model identifican org, Workspace, Instance, Definition, contact/conversation y causal assignment; no transportan principal humano ni grant token/authorization evidence. No asumir que assignment equivale a permiso de herramienta.
- Consumer verifica identidad del snapshot con envelope, pero no resuelve autoridades ni consulta CRM en cada tool call. Admission controla lifecycle (open/blocked), no capability access.
- Service JWT protege endpoint snapshot y service-to-service identity; no es CurrentAuthorization.

### Posible punto de integraciÃƒÆ’Ã‚Â³n ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â no aprobado aÃƒÆ’Ã‚Âºn
El gate live debe estar dentro del path de ejecuciÃƒÆ’Ã‚Â³n de cada capability, justo antes de invocar `capability.execute`, no sÃƒÆ’Ã‚Â³lo en snapshot/admission/tool exposure. Debe consumir un contract autoritativo para capability+operation+connection+resource+tenant+Instance+execution y fallar cerrado. Resolver autoridad y owner de grants antes de escribir adapter/modelos; mantener ExecutionEngine y admission/retirement separados de authorization.

### SeparaciÃƒÆ’Ã‚Â³n de concerns confirmada
- AssignmentResolver: causalidad y elegibilidad de AgentInstance asignada.
- SnapshotService: configuraciÃƒÆ’Ã‚Â³n efectiva y policy snapshot.
- AdmissionRepository/ExecutionControl: admisiÃƒÆ’Ã‚Â³n lifecycle y fencing.
- ExecutionEngine: stages, tool loop y orquestaciÃƒÆ’Ã‚Â³n.
- CapabilityExecutor: validaciÃƒÆ’Ã‚Â³n de llamada/timeout/cancel y despacho actual; candidato al gate live.
- CurrentAuthorization: autoridad ausente; no puede derivarse de roles `wrk.agents.*`, Service JWT, snapshot allowlist ni assignment.

### Estado
InvestigaciÃƒÆ’Ã‚Â³n de flujo fÃƒÆ’Ã‚Â­sico completada en este bloque; continuidad local actualizada. Sin cÃƒÆ’Ã‚Â³digo productivo modificado, sin tests/build/type-check/lint, commit/push/deploy.


## 37. ReconciliaciÃƒÆ’Ã‚Â³n corregida: Integraciones y autorizaciÃƒÆ’Ã‚Â³n runtime (20/11/2026)

### CorrecciÃƒÆ’Ã‚Â³n de continuidad
La investigaciÃƒÆ’Ã‚Â³n previa no debe tratar el ownership/contrato de integraciones ni la separaciÃƒÆ’Ã‚Â³n de autorizaciÃƒÆ’Ã‚Â³n como diseÃƒÆ’Ã‚Â±o pendiente. Estos acuerdos ya estÃƒÆ’Ã‚Â¡n definidos conceptualmente en WAB-78 y reconciliados con D-W79-08. Se conserva la ausencia de implementaciÃƒÆ’Ã‚Â³n fÃƒÆ’Ã‚Â­sica como gap, pero no se reabre la arquitectura.

### Contratos canÃƒÆ’Ã‚Â³nicos recuperados
- IntegraciÃƒÆ’Ã‚Â³n empresarial pertenece a Workspace + AgentInstance; conexiÃƒÆ’Ã‚Â³n privada por instancia por defecto.
- ConnectorDefinition (Waby), AgentInstanceIntegration (conexiÃƒÆ’Ã‚Â³n/credential reference protegida), AgentInstanceOperation (operaciÃƒÆ’Ã‚Â³n declarada) e IntegrationInvocation (auditorÃƒÆ’Ã‚Â­a) son nombres conceptuales; no autorizan crear schemas fÃƒÆ’Ã‚Â­sicos sin el incremento tÃƒÆ’Ã‚Â©cnico correspondiente.
- Workspace-authorized user administra connection/credential/operations bajo ARQ-20 (wrk.agents.*; lectura wrk.agents.read). Organization administra plan/entitlement, no la configuraciÃƒÆ’Ã‚Â³n de conexiÃƒÆ’Ã‚Â³n por instancia.
- Waby custodia y resuelve secretos. Secrets no entran en prompt/context/logs. No arbitrary HTTP, URLs, shell ni filesystem. Executor construye request a partir de operation registrada y argumentos validados.
- Antes de cada invocation se revalidan ownership, instance/integration state, operation grant/version, connection/credential state, resource scope y current authorization. RevocaciÃƒÆ’Ã‚Â³n de seguridad bloquea nuevas llamadas aun con execution snapshot previo.
- Capabilities efectivas: PlatformAllowed ÃƒÂ¢Ã‹â€ Ã‚Â© DefinitionAllowed ÃƒÂ¢Ã‹â€ Ã‚Â© InstanceEnabled ÃƒÂ¢Ã‹â€ Ã‚Â© CurrentAuthorization. Skills/Rules/Prompt/Knowledge no conceden autoridad.
- CurrentAuthorization es contextual/live y distinta de administraciÃƒÆ’Ã‚Â³n wrk.agents.*, JWT de servicio RS256, assignment, snapshot allowlist o P4 budget.

### ReconciliaciÃƒÆ’Ã‚Â³n con WAB-79 y WAB-81
- WAB-79 define ownership/persistencia y lifecycle; AgentInstanceIntegration/Operation aÃƒÆ’Ã‚Âºn no tienen fuente fÃƒÆ’Ã‚Â­sica verificada. Retirement/Admission/Drain/fencing permanecen independientes del capability authorization gate.
- WAB-81 ya tiene AgentInstance registry/executor y snapshot allowlist fail-closed; el gate live no estÃƒÆ’Ã‚Â¡ conectado. Punto de integraciÃƒÆ’Ã‚Â³n candidato sigue siendo el executor path, inmediatamente antes de capability.execute, condicionado a contrato autoritativo.
- La investigaciÃƒÆ’Ã‚Â³n de repositorios vigentes (CRM, AI Service, frontend, infrastructure) cerrÃƒÆ’Ã‚Â³ sin identificar catalog/grant/instance-operation sources fÃƒÆ’Ã‚Â­sicas. No repetir bÃƒÆ’Ã‚Âºsqueda amplia ni reabrir diseÃƒÆ’Ã‚Â±o. La brecha es implementaciÃƒÆ’Ã‚Â³n/source mapping concreta.

### PrÃƒÆ’Ã‚Â³ximo trabajo acotado
1. Revisar la implementaciÃƒÆ’Ã‚Â³n actual de AgentInstanceCapabilityExecutor y registry para determinar interfaz exacta del gate, sin modificar aÃƒÆ’Ã‚Âºn.
2. Mapear el contract conceptual de WAB-78 a un mÃƒÆ’Ã‚Â­nimo de interfaces/DTOs necesarios, distinguiendo: capability catalog/policy (Platform), definition allowlist, instance operation/config, live authorization y connection/credential resolver.
3. Confirmar owners/DB/service boundaries contra WAB-79 y contratos existentes de CRMÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬ÂAI Service; no crear modelo duplicado ni caller-supplied policy.
4. Si los authority sources continÃƒÆ’Ã‚Âºan ausentes (evidencia ya documentada), registrar un blocker/decision request mÃƒÆ’Ã‚Â­nimo en WAB-81; mantener fail-closed. No implementar adapters que fabrican grants/defaults.
5. Solo despuÃƒÆ’Ã‚Â©s de source/contract approval, conectar gate per-call y capabilities concretas aprobadas.

### Control
Este checkpoint es reconciliaciÃƒÆ’Ã‚Â³n documental/Linear, no cambio de cÃƒÆ’Ã‚Â³digo. No se ejecutan tests/build/type-check/lint. No commit/push/deploy.


## 38. InspecciÃƒÆ’Ã‚Â³n fÃƒÆ’Ã‚Â­sica del CapabilityExecutor AgentInstance target (20/11/2026)

### Componentes inspeccionados
- application/services/agent-runtime-capability-executor.ts
- application/services/agent-runtime-capability-registry.ts
- application/services/agent-runtime-capabilities.ts
- application/services/agent-instance-execution-runtime.ts
- application/services/agent-runtime-context-composer.ts

### Flujo fÃƒÆ’Ã‚Â­sico confirmado
AgentInstanceExecutionRuntime filtra capabilities registradas por snapshot.policy.allowedCapabilities y pasa definitions al Tool Loop. Para cada tool call, valida que el nombre estÃƒÆ’Ã‚Â© en exposed/allowlist y delega en RegistryAgentRuntimeCapabilityExecutor. El executor vuelve a buscar en registry, comprueba allowedCapabilities, valida argumentos contra schema, aplica timeout/cancel y luego invoca capability.execute(). Registry target es explÃƒÆ’Ã‚Â­cito/in-memory y Empty por default; executor fail-closed por default. No reutiliza legacy Bot registry.

### Punto de gate confirmado
El punto candidato es inmediatamente antes de capability.execute() dentro de RegistryAgentRuntimeCapabilityExecutor. No requiere crear otro ExecutionEngine ni modificar lifecycle admission/drain. La comprobaciÃƒÆ’Ã‚Â³n actual del registry + snapshot allowlist es local y no representa CurrentAuthorization live.

### Contexto disponible y faltante
AgentRuntimeContext incluye organizationId, workspaceId, agentInstanceId, agentDefinitionId, contactId, conversationId, snapshotId, effectiveConfigHash, allowedCapabilities y limits. No contiene executionId explÃƒÆ’Ã‚Â­cito en el context de capability, actor/principal, integrationId/connectionId, operationId/version, resource scope ni authorization evidence/revocation revision. ToolCall identifica capability name + arguments; no existe binding contractual a operation/connection.

### ImplicaciÃƒÆ’Ã‚Â³n
El executor es el lugar correcto para insertar un gate, pero el contrato de request/decision no puede fijarse sÃƒÆ’Ã‚Â³lo con el contexto actual. Antes de implementar: reconciliar identity/executionId con AgentExecution durable; mapear capability name a una operaciÃƒÆ’Ã‚Â³n/connector registrada (no inferir desde nombre/tool args); definir quiÃƒÆ’Ã‚Â©n proporciona actor y grants live; y resolver source de conexiÃƒÆ’Ã‚Â³n/credencial y revocation. Gate debe denegar por default ante source ausente, stale, error o scope mismatch.

### Estado
InspecciÃƒÆ’Ã‚Â³n de cÃƒÆ’Ã‚Â³digo ÃƒÆ’Ã‚Âºnicamente. No cambios productivos, tests/build/type-check/lint, commit/push/deploy.


## 39. Cierre de identidad durable y propagaciÃƒÆ’Ã‚Â³n hacia capability gate (20/11/2026)

### Identidad autoritativa verificada
- `AgentExecution._id` es la identidad durable de ejecuciÃƒÆ’Ã‚Â³n.
- `AgentExecutionInbox.executionId` referencia la ejecuciÃƒÆ’Ã‚Â³n admitida.
- `AgentExecutionInboxWorker` carga el `AgentExecutionDocument`, valida coincidencia de identity entre Inbox y Execution, marca started y entrega el documento durable al `AgentExecutionRuntime.execute(execution, signal)`.
- `AgentInstanceExecutionRuntime` utiliza `execution._id` en `ExecutionEngine.executeAgentRuntime` y `executeAgentRuntimeToolLoop`.
- `AgentRuntimeContextComposer` construye `AgentRuntimeContext` con tenancy, instance/definition, contact/conversation, snapshot/hash, policy allowlist y limits; actualmente no recibe ni expone `executionId`.
- `AgentRuntimeCapabilityExecutor.execute(context, call, signal)` recibe context y ToolCall. `RegistryAgentRuntimeCapabilityExecutor` valida registry + snapshot allowlist, schema, timeout/cancel y llama `capability.execute()`. El executor no tiene source live de authorization.

### Campos durable disponibles para gate/audit
`executionId = AgentExecution._id`; organizationId, workspaceId, agentInstanceId, agentDefinitionId, contactId, conversationId y evAssignedId se obtienen del durable AgentExecution. No generar identity paralela ni aceptar identity desde tool arguments/LLM.

### Gap contractual exacto
La propagation de `executionId` al `AgentRuntimeContext` es un cambio local y determinista, pero por sÃƒÆ’Ã‚Â­ sola no constituye autorizaciÃƒÆ’Ã‚Â³n. No se modifica aÃƒÆ’Ã‚Âºn porque el contrato request/decision del CurrentAuthorization gate, mapping capabilityÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢registered operation y authority live no estÃƒÆ’Ã‚Â¡n definidos fÃƒÆ’Ã‚Â­sicamente. Tampoco hay capabilities target registradas en composition root: `InMemoryAgentRuntimeCapabilityRegistry` se crea vacÃƒÆ’Ã‚Â­o y `RegistryAgentRuntimeCapabilityExecutor` usa ese registry; no hay binding fÃƒÆ’Ã‚Â­sico operation/connection/grant.

### DecisiÃƒÆ’Ã‚Â³n de checkpoint
- Se confirma la fuente de execution identity y la cadena de propagation durable.
- No crear nuevo ExecutionContext/ExecutionId, modelo de grants, actor ficticio, connection/operation selector ni adapter que autorice.
- El gate debe usar execution identity derivada de AgentExecution durable, tenant/instance identity y operation binding de una capability registrada; no debe confiar en nombres/args del LLM.
- La investigaciÃƒÆ’Ã‚Â³n de fuentes authoritative de capability/current authorization/AI limits queda cerrada segÃƒÆ’Ã‚Âºn Ãƒâ€šÃ‚Â§20.7; no repetir bÃƒÆ’Ã‚Âºsquedas generales.
- PrÃƒÆ’Ã‚Â³ximo bloque: especificar el mÃƒÆ’Ã‚Â­nimo blocker/decision request para el source contractual del gate y operation binding; revisar si WAB-81/WAB-78 contienen un contrato operativo que autorice la fuente. Hasta resolverlo, runtime registry vacÃƒÆ’Ã‚Â­o y executor default fail-closed se mantienen.

InspecciÃƒÆ’Ã‚Â³n solamente; no cambios de cÃƒÆ’Ã‚Â³digo, tests/build/type-check/lint, commit/push/deploy.


## 40. Bloqueo contractual mÃƒÆ’Ã‚Â­nimo para el gate de autorizaciÃƒÆ’Ã‚Â³n de capacidades

La inspecciÃƒÆ’Ã‚Â³n fÃƒÆ’Ã‚Â­sica cerrÃƒÆ’Ã‚Â³ identidad durable y ubicaciÃƒÆ’Ã‚Â³n del gate; no existe aÃƒÆ’Ã‚Âºn contrato fÃƒÆ’Ã‚Â­sico aprobado que permita implementarlo correctamente. No se reabre el diseÃƒÆ’Ã‚Â±o conceptual de WAB-78 ni se realiza otra auditorÃƒÆ’Ã‚Â­a general.

### Decisiones/contratos que faltan para desbloquear implementaciÃƒÆ’Ã‚Â³n

1. **Fuente autoritativa de CurrentAuthorization:** identificar componente/servicio y contrato existente que decide, en tiempo real, si AgentInstance puede ejecutar una operaciÃƒÆ’Ã‚Â³n. Debe cubrir revocaciÃƒÆ’Ã‚Â³n y estados actuales; snapshot, JWT de servicio y permisos administrativos no sustituyen esta decisiÃƒÆ’Ã‚Â³n.
2. **Binding confiable capability ÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢ operaciÃƒÆ’Ã‚Â³n:** establecer cÃƒÆ’Ã‚Â³mo una capacidad registrada resuelve una AgentInstanceOperation concreta, su versiÃƒÆ’Ã‚Â³n, integraciÃƒÆ’Ã‚Â³n/conexiÃƒÆ’Ã‚Â³n y scope. Este binding debe ser del registro/configuraciÃƒÆ’Ã‚Â³n confiable, nunca seleccionado por argumentos del LLM.
3. **Identidad y contexto de la decisiÃƒÆ’Ã‚Â³n:** derivar desde AgentExecution y configuraciÃƒÆ’Ã‚Â³n persistida executionId, organizaciÃƒÆ’Ã‚Â³n, Workspace, AgentInstance, AgentDefinition, capability, operation/version, integraciÃƒÆ’Ã‚Â³n/conexiÃƒÆ’Ã‚Â³n y scope requerido. No aceptar estos valores como autoridad desde tool arguments.
4. **SemÃƒÆ’Ã‚Â¡ntica de freshness/revocaciÃƒÆ’Ã‚Â³n:** confirmar que el gate consulta autoridad vigente por invocation (o mecanismo equivalente con garantÃƒÆ’Ã‚Â­as explÃƒÆ’Ã‚Â­citas de revocaciÃƒÆ’Ã‚Â³n); errores, ausencia, mismatch o estado stale deben denegar.
5. **Resultado mÃƒÆ’Ã‚Â­nimo:** decisiÃƒÆ’Ã‚Â³n allow/deny, cÃƒÆ’Ã‚Â³digo/razÃƒÆ’Ã‚Â³n no sensible, revision/version de autoridad y metadata de freshness suficiente para auditorÃƒÆ’Ã‚Â­a. No inventar un modelo fÃƒÆ’Ã‚Â­sico ni una API hasta identificar el contrato existente.
6. **ResoluciÃƒÆ’Ã‚Â³n de credenciales:** separar autorizaciÃƒÆ’Ã‚Â³n de secret resolution. Resolver credencial protegida solo tras autorizaciÃƒÆ’Ã‚Â³n; nunca exponer secretos al prompt, argumentos de tool ni logs.

### Punto de implementaciÃƒÆ’Ã‚Â³n confirmado
RegistryAgentRuntimeCapabilityExecutor, inmediatamente antes de capability.execute(). El registro actual estÃƒÆ’Ã‚Â¡ vacÃƒÆ’Ã‚Â­o y el executor default falla cerrado; no agregar capacidades reales ni scaffolding de autorizaciÃƒÆ’Ã‚Â³n hasta resolver los contratos anteriores.

### PrÃƒÆ’Ã‚Â³xima acciÃƒÆ’Ã‚Â³n
Solicitar/identificar la fuente y contrato fÃƒÆ’Ã‚Â­sicos existentes para los puntos 1ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Å“4 y su owner. Una vez confirmados, documentar el contrato mÃƒÆ’Ã‚Â­nimo, implementar gate + binding en el executor/registry existente y aÃƒÆ’Ã‚Â±adir pruebas; tests/build/typecheck/lint quedan pospuestos hasta terminar la implementaciÃƒÆ’Ã‚Â³n, segÃƒÆ’Ã‚Âºn el orden de trabajo acordado. Sin cambios de cÃƒÆ’Ã‚Â³digo, tests, build, typecheck, lint, commit, push o deploy en este checkpoint.


## 41. PrecisiÃƒÆ’Ã‚Â³n sobre el resolver de polÃƒÆ’Ã‚Â­tica CRM existente

La bÃƒÆ’Ã‚Âºsqueda focalizada encontrÃƒÆ’Ã‚Â³ `crm/src/services/AgentRuntimePolicyResolver.service.ts`. El contrato `AgentRuntimePolicyAuthoritySet` incluye `currentAuthorization` como una de cuatro capas, y `IntersectingAgentRuntimePolicyResolver` intersecta sus `allowedCapabilities` y reduce limits. Sin embargo:

- `IAgentRuntimePolicyAuthoritySource.resolveAuthorities(context)` es una abstracciÃƒÆ’Ã‚Â³n; no se identificÃƒÆ’Ã‚Â³ una implementaciÃƒÆ’Ã‚Â³n fÃƒÆ’Ã‚Â­sica autoritativa conectada.
- El contexto solo contiene organizationId, workspaceId, agentInstanceId y agentDefinitionId.
- `currentAuthorization` en este resolver es una capa de policy de capacidades/limits durante la resoluciÃƒÆ’Ã‚Â³n de polÃƒÆ’Ã‚Â­tica del runtime; no constituye un gate por invocaciÃƒÆ’Ã‚Â³n de `AgentInstanceOperation`, ni entrega binding de conexiÃƒÆ’Ã‚Â³n, resource scope, revision/freshness o revocaciÃƒÆ’Ã‚Â³n inmediata.
- El resolver fail-closed existente confirma que no debe inventarse un source ni wiring en esta etapa.

**Resultado:** contrato conceptual de cuatro capas y scaffold del resolver ya existen. El bloqueo se estrecha al source fÃƒÆ’Ã‚Â­sico de autoridades y, separadamente, al contrato de autorizaciÃƒÆ’Ã‚Â³n de operaciÃƒÆ’Ã‚Â³n por invocation para integraciones externas. No duplicar el resolver ni tratar `currentAuthorization.allowedCapabilities` como autorizaciÃƒÆ’Ã‚Â³n suficiente para ejecutar una operaciÃƒÆ’Ã‚Â³n externa.


## 42. VerificaciÃƒÆ’Ã‚Â³n de implementaciÃƒÆ’Ã‚Â³n del AuthoritySource

BÃƒÆ’Ã‚Âºsqueda exacta de `IAgentRuntimePolicyAuthoritySource` dentro de `crm/src` devuelve ÃƒÆ’Ã‚Âºnicamente su declaraciÃƒÆ’Ã‚Â³n y uso en `AgentRuntimePolicyResolver.service.ts`; no se encontrÃƒÆ’Ã‚Â³ implementaciÃƒÆ’Ã‚Â³n concreta. El contrato permanece sin wiring a autoridades fÃƒÆ’Ã‚Â­sicas. Se cierra esta bÃƒÆ’Ã‚Âºsqueda focalizada: no ampliar a una auditorÃƒÆ’Ã‚Â­a de permisos general.


## 43. PropagaciÃƒÆ’Ã‚Â³n de executionId al AgentRuntimeContext

Se implementÃƒÆ’Ã‚Â³ un cambio local acotado, independiente del bloqueo de autoridad:

- `AgentRuntimeInput` ahora requiere `executionId`.
- `AgentRuntimeContext` expone `executionId` como identidad durable.
- `AgentRuntimeContextComposer` valida que executionId no estÃƒÆ’Ã‚Â© vacÃƒÆ’Ã‚Â­o junto a contactId/conversationId.
- `AgentInstanceExecutionRuntime` deriva executionId exclusivamente de `AgentExecutionDocument._id` y lo pasa al Composer.
- Tests del Composer fueron ajustados para aportar executionId y comprobar su propagaciÃƒÆ’Ã‚Â³n. No se ejecutaron.

No se deriva identidad desde input del usuario/LLM, no se cambia el contrato del ExecutionEngine y no se agregan autorizaciones. El campo queda disponible para el futuro gate/binding de capacidades.

**ValidaciÃƒÆ’Ã‚Â³n:** revisiÃƒÆ’Ã‚Â³n de cambios por lectura; tests/build/typecheck/lint no ejecutados, segÃƒÆ’Ã‚Âºn orden acordado. No commit/push/deploy.


## 44. Cierre del siguiente paso: contrato fÃƒÆ’Ã‚Â­sico del gate CurrentAuthorization (20/11/2026)

### Punto de integraciÃƒÆ’Ã‚Â³n
La inspecciÃƒÆ’Ã‚Â³n de `RegistryAgentRuntimeCapabilityExecutor.execute()` confirma que el gate live debe ejecutarse inmediatamente antes de `capability.execute(context, call, executionSignal)`. Las verificaciones existentes de registry, snapshot allowlist, schema, timeout y cancelaciÃƒÆ’Ã‚Â³n se conservan; ninguna equivale a CurrentAuthorization por invocation.

### Contrato fÃƒÆ’Ã‚Â­sico que debe existir antes de conectar operaciones externas
El gate debe recibir un request derivado exclusivamente de datos confiables del runtime y binding registrado:
- executionId durable (AgentExecution._id), organizationId, workspaceId, agentInstanceId y agentDefinitionId;
- capability registrada y binding confiable a AgentInstanceOperation, operation/version, integraciÃƒÆ’Ã‚Â³n/conexiÃƒÆ’Ã‚Â³n y resource scope;
- actor/principal y contexto de autorizaciÃƒÆ’Ã‚Â³n segÃƒÆ’Ã‚Âºn la fuente autoritativa existente (no inventar actor a partir del mensaje o JWT de servicio);
- evidencia/revisiÃƒÆ’Ã‚Â³n de autoridad y freshness suficiente para garantizar revocaciÃƒÆ’Ã‚Â³n segÃƒÆ’Ã‚Âºn el contrato;

La fuente debe responder allow/deny. Ausencia, error, identidad mismatch, operation/connection inconsistente o autoridad stale deniega. La credencial protegida se resuelve ÃƒÆ’Ã‚Âºnicamente despuÃƒÆ’Ã‚Â©s del allow; nunca se expone a prompt, tool arguments o logs.

### Estado de fuentes
El resolver CRM `AgentRuntimePolicyResolver` conserva el contrato conceptual de cuatro autoridades e intersecciÃƒÆ’Ã‚Â³n de capabilities/limits, pero `IAgentRuntimePolicyAuthoritySource` no tiene implementaciÃƒÆ’Ã‚Â³n fÃƒÆ’Ã‚Â­sica encontrada. Tampoco se identificÃƒÆ’Ã‚Â³ source fÃƒÆ’Ã‚Â­sico para operaciÃƒÆ’Ã‚Â³n registrada, connection binding, actor/grant live o revocation revision. El contrato de WAB-78 es conceptual y no autoriza inventar schemas/adapters.

### Resultado / bloqueo preciso
No se implementa todavÃƒÆ’Ã‚Â­a un gate que pueda fabricar o inferir grants. El runtime permanece fail-closed y el registry target vacÃƒÆ’Ã‚Â­o. El siguiente desbloqueo requiere identificar el owner y contrato fÃƒÆ’Ã‚Â­sico de CurrentAuthorization por invocation y el binding persistido capabilityÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢operationÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢integration/connection. Una vez confirmados, implementar gate y binding en el executor/registry existente, agregar tests especÃƒÆ’Ã‚Â­ficos y luego ejecutar la baterÃƒÆ’Ã‚Â­a de validaciÃƒÆ’Ã‚Â³n acordada.

No se modificÃƒÆ’Ã‚Â³ cÃƒÆ’Ã‚Â³digo en este checkpoint. Tests/build/typecheck/lint continÃƒÆ’Ã‚Âºan diferidos hasta completar implementaciÃƒÆ’Ã‚Â³n. Sin commit/push/deploy.


## 45. ReconciliaciÃƒÆ’Ã‚Â³n con contratos WAB-78/WAB-79 y Linear ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â evidencia que estrecha el gate (20/11/2026)

### Fuentes revisadas
- WAB-81 en Linear: checkpoint de cuatro authorities y capability gate.
- WAB-80 en Linear: aclaraciones de ARQ-20 y ownership de conexiones/operaciones por Workspace + AgentInstance.
- WAB-78 en Linear y documentaciÃƒÆ’Ã‚Â³n canÃƒÆ’Ã‚Â³nica: integraciones-api-externas.md, configuracion-efectiva-y-capacidades.md, fase-2-diseno-objetivo-y-contratos.md.
- WAB-79 dominio/ownership/persistencia y D-W79-08.
- AUD-02 WAB-88 y AUD-05 WAB-91 como lÃƒÆ’Ã‚Â­mites de autorizaciÃƒÆ’Ã‚Â³n administrativa/legacy, no autoridad runtime.

### Hallazgo importante: el contrato ya ayuda y no debe volver a tratarse como diseÃƒÆ’Ã‚Â±o pendiente
La documentaciÃƒÆ’Ã‚Â³n canÃƒÆ’Ã‚Â³nica ya define el gate operacional con bastante precisiÃƒÆ’Ã‚Â³n:
- IntegraciÃƒÆ’Ã‚Â³n privada por defecto a Workspace + AgentInstance.
- Waby-owned ConnectorDefinition y executor.
- AgentInstanceIntegration como ubicaciÃƒÆ’Ã‚Â³n conceptual de conexiÃƒÆ’Ã‚Â³n y referencia protegida a credencial.
- AgentInstanceOperation como operaciÃƒÆ’Ã‚Â³n registrada, versionada, con method/ruta relativa, schemas, efecto y controles.
- IntegrationInvocation como auditorÃƒÆ’Ã‚Â­a de la invocaciÃƒÆ’Ã‚Â³n vinculada a Organization, Workspace, AgentInstance y Execution.
- LLM solo selecciona operaciÃƒÆ’Ã‚Â³n registrada y entrega argumentos; nunca URL/host/HTTP arbitrario.
- Waby revalida ownership, autorizaciÃƒÆ’Ã‚Â³n, estado de conexiÃƒÆ’Ã‚Â³n, operaciÃƒÆ’Ã‚Â³n/version, credencial y scope antes de invocar; revocaciÃƒÆ’Ã‚Â³n de seguridad bloquea llamadas nuevas aun con snapshot previo.
- Credential resolution ocurre en el executor despuÃƒÆ’Ã‚Â©s de autorizar; secretos fuera de prompt/context/logs.
- CapabilityEffective = PlatformAllowed ÃƒÂ¢Ã‹â€ Ã‚Â© DefinitionAllowed ÃƒÂ¢Ã‹â€ Ã‚Â© InstanceEnabled ÃƒÂ¢Ã‹â€ Ã‚Â© CurrentAuthorization.
- CurrentAuthorization es contextual/live, no wrk.agents.*, JWT service, assignment, snapshot allowlist o P4.

### Contratos existentes que reutilizar (no duplicar)
1. **AdministraciÃƒÆ’Ã‚Â³n/configuraciÃƒÆ’Ã‚Â³n:** ARQ-20 wrk.agents.* y wrk.agents.read, WAB-80. Sirve para que usuarios Workspace administren instancia, integraciones y operaciones; no es permiso de ejecuciÃƒÆ’Ã‚Â³n.
2. **AutenticaciÃƒÆ’Ã‚Â³n servicio:** ARQ-21 RS256, claims acordados iss/sub/aud/iat/nbf/exp/jti/scope/orgId/workspaceId/agentInstanceId/initiatedBy; identidad tÃƒÆ’Ã‚Â©cnica diferenciada por direcciÃƒÆ’Ã‚Â³n y token por peticiÃƒÆ’Ã‚Â³n/scope mÃƒÆ’Ã‚Â­nimo. Autentica CRM/AI Service, no concede capability. Operaciones/scopes exactos e implementaciÃƒÆ’Ã‚Â³n siguen en WAB-80/81.
3. **Ownership/persistencia:** WAB-79/D-W79-08 define boundaries conceptuales de conexiÃƒÆ’Ã‚Â³n y credential reference; no se aprobÃƒÆ’Ã‚Â³ schema fÃƒÆ’Ã‚Â­sico, storage/KMS ni modelos AgentInstanceIntegration/Operation.
4. **Snapshot/policy:** WAB-81 resolver de cuatro authorities/intersection y fail-closed; snapshot materializa authorities estÃƒÆ’Ã‚Â¡ticas, CurrentAuthorization se verifica live. AuthoritySource physical wiring sigue ausente.
5. **Security audits:** WAB-88 identifica auth administrativa GraphQL/ownership legacy; WAB-91 identifica capabilities globales Bot legacy y aislamiento deficiente. Son evidencia de riesgos legacy, no sources target de grants.

### QuÃƒÆ’Ã‚Â© se puede cerrar ahora
- Cerrar la ambigÃƒÆ’Ã‚Â¼edad conceptual del request/decision del gate: el gate valida execution identity + ownership chain + capability/operation binding + instance enablement/connection state + CurrentAuthorization vigente + platform revocation + resource scope + credential status. Allow/deny; deny/fail closed. Auditable invocation.
- No exigir un actor humano por cada tool call como supuesto. El runtime es identidad tÃƒÆ’Ã‚Â©cnica Waby acotada; initiatedBy conserva actor iniciador cuando exista, pero no autentica ni autoriza. CurrentAuthorization debe derivarse del contrato contextual/runtime aprobado, no de un actor inventado.
- No usar AgentRuntimePolicyResolver.currentAuthorization.allowedCapabilities como substitute: resolver composition no contiene operation/connection/resource scope ni freshness/revocation por invocation.

### Bloqueo fÃƒÆ’Ã‚Â­sico restante (ya no documental)
No se encontraron schemas/implementaciones aprobados de AgentInstanceIntegration/Operation/Invocation, catalog/operation grants, live authorization/revocation source ni concrete AuthoritySource wiring. El storage de secret refs/resolver tampoco estÃƒÆ’Ã‚Â¡ seleccionado. No crear esos modelos/adapters unilateralmente en WAB-81.

### PrÃƒÆ’Ã‚Â³ximo paso recomendado
Revisar si WAB-80/WAB-79 tiene un owner/issue explÃƒÆ’Ã‚Â­cito para implementar estos contratos (no encontrado en los issue descriptions revisados). Si no lo tiene, registrar en Linear como decisiÃƒÆ’Ã‚Â³n/bloqueo concreto de implementaciÃƒÆ’Ã‚Â³n: owner de source live authorization + binding operation/connection + physical integration/credential resolver, con scope mÃƒÆ’Ã‚Â­nimo compatible con WAB-78. No abrir auditorÃƒÆ’Ã‚Â­a general ni reabrir decisiones ARQ-19/20/21.

InspecciÃƒÆ’Ã‚Â³n/reconciliaciÃƒÆ’Ã‚Â³n documental; no code, tests/build/typecheck/lint, commit/push/deploy.


### 45.1 VerificaciÃƒÆ’Ã‚Â³n de ownership en WAB-79 y distinciÃƒÆ’Ã‚Â³n Credential Registry

La revisiÃƒÆ’Ã‚Â³n de WAB-79 confirma que su issue padre solo delimita incrementos aceptados por separado; A1 es AgentInstance CRM. No asigna actualmente un incremento a la persistencia fÃƒÆ’Ã‚Â­sica de AgentInstanceIntegration/AgentInstanceOperation/IntegrationInvocation ni al live authorization source.

El avance reciente de Credential Registry en WAB-79/CRM/AI Service es especÃƒÆ’Ã‚Â­ficamente para credenciales de proveedores LLM y binding LLMRoute.providerAccountId. No debe reutilizarse como registry/resolver de credenciales empresariales de integrations; WAB-78/D-W79-08 mantiene ambos dominios separados.

Por tanto, los docs/Linear ayudan a cerrar el contrato funcional del gate, pero no existe aÃƒÆ’Ã‚Âºn owner/incremento aprobado que implemente las entidades y autoridad live de integraciones. La brecha queda confirmada como decisiÃƒÆ’Ã‚Â³n de alcance/ownership, no como falta de documentaciÃƒÆ’Ã‚Â³n de arquitectura.

## CurrentAuthorization ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â inspecciÃƒÆ’Ã‚Â³n fÃƒÆ’Ã‚Â­sica y gate fail-closed (20/11/2026)

### Evidencia del CRM

- `src/services/AgentRuntimePolicyResolver.service.ts` declara `IAgentRuntimePolicyAuthoritySource` y `currentAuthorization` como una de las cuatro capas, pero no se encontrÃƒÆ’Ã‚Â³ implementaciÃƒÆ’Ã‚Â³n concreta de `resolveAuthorities` en CRM.
- `AgentRuntimeSnapshotService` usa por defecto `FailClosedAgentRuntimePolicyResolver`; resuelve tenancy WorkspaceÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢Organization, AgentInstance activa, Rules/Skills y policy antes de solicitar el snapshot.
- El resolver intersecta capacidades y lÃƒÆ’Ã‚Â­mites de las cuatro autoridades. Esa composiciÃƒÆ’Ã‚Â³n sirve para policy/snapshot, pero no constituye autorizaciÃƒÆ’Ã‚Â³n contextual por invocaciÃƒÆ’Ã‚Â³n: no vincula operaciÃƒÆ’Ã‚Â³n registrada, integraciÃƒÆ’Ã‚Â³n/conexiÃƒÆ’Ã‚Â³n, scope del recurso, estado de credencial ni revocaciÃƒÆ’Ã‚Â³n vigente justo antes del efecto externo.
- `hasPermission` / AuthChecker resuelven permisos administrativos de usuarios Organization/Workspace. No son un mecanismo de grants de ejecuciÃƒÆ’Ã‚Â³n de AgentInstance y no deben reutilizarse como CurrentAuthorization.
- No se identificÃƒÆ’Ã‚Â³ contrato fÃƒÆ’Ã‚Â­sico de `AgentInstanceIntegration`, `AgentInstanceOperation`, `IntegrationInvocation`, ni una fuente live de grants/revocaciÃƒÆ’Ã‚Â³n en el CRM. La documentaciÃƒÆ’Ã‚Â³n WAB-78 define su ownership y comportamiento conceptual, no el schema fÃƒÆ’Ã‚Â­sico.

### Cambio local acotado

Se aÃƒÆ’Ã‚Â±adiÃƒÆ’Ã‚Â³ `application/services/agent-runtime-current-authorization.ts`:
- Request de gate derivado del contexto runtime durable: executionId, Organization/Workspace/AgentInstance/AgentDefinition, snapshotId/hash, capabilityName y toolCallId.
- Los argumentos de ToolCall se pasan como datos no confiables; no son evidencia de autorizaciÃƒÆ’Ã‚Â³n ni deben decidir tenancy.
- Interface `AgentRuntimeCurrentAuthorizationGate`.
- ImplementaciÃƒÆ’Ã‚Â³n default `FailClosedAgentRuntimeCurrentAuthorizationGate`, que deniega con `AGENT_RUNTIME_CURRENT_AUTHORIZATION_NOT_CONFIGURED`.

Se modificÃƒÆ’Ã‚Â³ `RegistryAgentRuntimeCapabilityExecutor`:
- Gate opcional inyectable, default fail-closed.
- La comprobaciÃƒÆ’Ã‚Â³n ocurre despuÃƒÆ’Ã‚Â©s de lookup, allowlist snapshot y validaciÃƒÆ’Ã‚Â³n de schema, pero inmediatamente antes de `capability.execute()`.
- DenegaciÃƒÆ’Ã‚Â³n devuelve el motivo de autorizaciÃƒÆ’Ã‚Â³n; fallo/indisponibilidad del gate devuelve `AGENT_RUNTIME_CURRENT_AUTHORIZATION_UNAVAILABLE`.
- No se permite ejecutar si el gate no estÃƒÆ’Ã‚Â¡ configurado o no autoriza.

### LÃƒÆ’Ã‚Â­mites deliberados

Este incremento materializa el punto de enforcement y un contrato mÃƒÆ’Ã‚Â­nimo de identidad, pero NO implementa una fuente autoritativa live. El contrato de integraciÃƒÆ’Ã‚Â³n/operaciÃƒÆ’Ã‚Â³n/scope aÃƒÆ’Ã‚Âºn no existe fÃƒÆ’Ã‚Â­sicamente; por eso no se declara CurrentAuthorization funcional ni se habilitan capacidades externas. No se inventÃƒÆ’Ã‚Â³ un endpoint CRM, modelo grant, credential resolver ni esquema de operaciÃƒÆ’Ã‚Â³n.

### ValidaciÃƒÆ’Ã‚Â³n y estado

- No se ejecutaron tests, type-check, build ni lint (segÃƒÆ’Ã‚Âºn el alcance acordado de completar implementaciÃƒÆ’Ã‚Â³n antes de validar).
- No se hicieron commits, pushes ni deploys.
- PrÃƒÆ’Ã‚Â³ximo paso bloqueante: delimitar e implementar el contrato fÃƒÆ’Ã‚Â­sico mÃƒÆ’Ã‚Â­nimo CRM-authoritative para autorizaciÃƒÆ’Ã‚Â³n contextual de invocaciones; reconciliarlo con WAB-78/WAB-79 antes de conectar un gate real. El gate fail-closed ya estÃƒÆ’Ã‚Â¡ preparado para recibir esa implementaciÃƒÆ’Ã‚Â³n.

## Contrato fÃƒÆ’Ã‚Â­sico mÃƒÆ’Ã‚Â­nimo propuesto para CurrentAuthorization (incremento de diseÃƒÆ’Ã‚Â±o, 20/11/2026)

### 1. DecisiÃƒÆ’Ã‚Â³n de ownership recomendada

Separar la autoridad de configuraciÃƒÆ’Ã‚Â³n/ownership de la autoridad de ejecuciÃƒÆ’Ã‚Â³n:

- **CRM-authoritative para configuraciÃƒÆ’Ã‚Â³n y ownership vigente:** OrganizationÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢WorkspaceÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢AgentInstance, estado actual de instancia, integraciÃƒÆ’Ã‚Â³n configurada/activa, operaciÃƒÆ’Ã‚Â³n registrada/habilitada/versionada, binding de operaciÃƒÆ’Ã‚Â³n a conexiÃƒÆ’Ã‚Â³n y revocaciones administrativas. No se crea una segunda DB de grants en AI Service.
- **Waby-authoritative para ejecuciÃƒÆ’Ã‚Â³n:** polÃƒÆ’Ã‚Â­ticas globales PlatformAllowed, compatibilidad DefinitionAllowed, executor soportado, custodia/resoluciÃƒÆ’Ã‚Â³n de credenciales empresariales y estado del credential resolver. CRM no ejecuta APIs externas ni recibe secretos en claro.
- **AI Service Runtime:** solicita una decisiÃƒÆ’Ã‚Â³n live con identidad durable y binding confiable; no infiere autorizaciÃƒÆ’Ã‚Â³n de prompt, ToolCall, snapshot ni JWT de servicio.
- **Integration Executor de Waby:** debe revalidar/consumir autorizaciÃƒÆ’Ã‚Â³n en su lÃƒÆ’Ã‚Â­mite de efecto, resolver secreto solo despuÃƒÆ’Ã‚Â©s del allow y ejecutar operaciÃƒÆ’Ã‚Â³n registrada. El gate del Runtime es una barrera previa, no sustituye el control en el executor que produce el efecto externo.

Esta separaciÃƒÆ’Ã‚Â³n respeta WAB-78 y D-W79-08; Credential Registry LLM no se reutiliza para integraciones empresariales.

### 2. No introducir un Grant duplicado por ahora

La configuraciÃƒÆ’Ã‚Â³n explÃƒÆ’Ã‚Â­cita y vigente de AgentInstanceOperation es el grant de Instance Authority, siempre que la entidad fÃƒÆ’Ã‚Â­sica defina su habilitaciÃƒÆ’Ã‚Â³n, binding, versiÃƒÆ’Ã‚Â³n y scope. CurrentAuthorization revalida ese estado en tiempo real y las autoridades de seguridad adicionales; no mantiene un segundo catÃƒÆ’Ã‚Â¡logo de permisos que pueda divergir.

La entidad fÃƒÆ’Ã‚Â­sica no existe todavÃƒÆ’Ã‚Â­a. Esta propuesta no autoriza crear schema unilateralmente en WAB-81. Su owner/incremento debe confirmarse antes de implementar modelos.

### 3. Request de autorizaciÃƒÆ’Ã‚Â³n

El request debe construirse a partir de contexto durable y metadata registrada del capability executor, nunca de campos de tenancy o binding proporcionados por el LLM.

Identidad requerida:
- executionId (AgentExecution durable)
- organizationId, workspaceId
- agentInstanceId, agentDefinitionId
- snapshotId, effectiveConfigHash

Binding requerido para operaciones externas:
- capabilityId/nombre estable registrado
- connectorDefinitionId
- integrationId
- operationId
- operationVersion
- connectionRevision (o revision estable del binding)
- resourceScope resuelto por cÃƒÆ’Ã‚Â³digo confiable de la capability/adapter

toolCallId es correlaciÃƒÆ’Ã‚Â³n de la invocaciÃƒÆ’Ã‚Â³n; no autoridad. Los argumentos son input no confiable y nunca determinan tenancy, connectionId, credentialRef ni grant. El scope debe ser derivado y validado por una capability registrada, con reglas explÃƒÆ’Ã‚Â­citas por operaciÃƒÆ’Ã‚Â³n.

### 4. Decision response

Contrato lÃƒÆ’Ã‚Â³gico mÃƒÆ’Ã‚Â­nimo:
- decision: allow/deny
- reasonCode: cÃƒÆ’Ã‚Â³digo estable no sensible para denial
- decisionId/invocationId: correlaciÃƒÆ’Ã‚Â³n auditable
- checkedAt
- authorityRevision/configurationRevision para identificar el estado verificado
- binding confirmado (IDs/versiones) para evitar que la respuesta se use con otra operaciÃƒÆ’Ã‚Â³n/conexiÃƒÆ’Ã‚Â³n
- expiresAt corto, si se emite ticket; no se autoriza cachear grants generales

El AI Service no debe interpretar ausencia de respuesta, respuesta malformed, mismatch de identidad/binding, timeout o autoridad unavailable como allow. Todos son fail-closed.

### 5. Fuente CRM y transporte

RecomendaciÃƒÆ’Ã‚Â³n: endpoint interno de CRM para autorizaciÃƒÆ’Ã‚Â³n de capability/integration invocation, invocado con autenticaciÃƒÆ’Ã‚Â³n servicio-a-servicio RS256 y scope mÃƒÆ’Ã‚Â­nimo dedicado. El endpoint re-resuelve la configuraciÃƒÆ’Ã‚Â³n desde persistencia CRM; no acepta el estado de instancia/grant enviado como prueba por AI Service.

ARQ-21 JWT existente fue diseÃƒÆ’Ã‚Â±ado y verificado para CRMÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢AI Service; no asumir que ese token/client sirve automÃƒÆ’Ã‚Â¡ticamente para AI ServiceÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢CRM. La direcciÃƒÆ’Ã‚Â³n inversa requiere issuer/audience, principal, scope, key/verificaciÃƒÆ’Ã‚Â³n y middleware explÃƒÆ’Ã‚Â­citamente compatibles. No reutilizar token de otra direcciÃƒÆ’Ã‚Â³n ni bypass administrativo.

Ruta/nombre, DTO exacto, scopes, claims y middleware deben reconciliarse con WAB-80/WAB-81 antes de codificar; el endpoint aquÃƒÆ’Ã‚Â­ es una propuesta de contrato, no uno existente.

### 6. RevalidaciÃƒÆ’Ã‚Â³n y revocaciÃƒÆ’Ã‚Â³n

La decisiÃƒÆ’Ã‚Â³n live debe comprobar:
1. Tenant/Workspace/AgentInstance ownership y status operativo.
2. AgentDefinition/capability compatibilidad, ademÃƒÆ’Ã‚Â¡s de policy global.
3. Integration activa y perteneciente al AgentInstance.
4. Operation habilitada, version/binding vigente y schema.
5. Resource scope permitido por la operaciÃƒÆ’Ã‚Â³n.
6. RevocaciÃƒÆ’Ã‚Â³n/estado de conexiÃƒÆ’Ã‚Â³n vigente.
7. Credential reference/state usable por resolver de Waby (sin revelar secreto a CRM/AI Service).
8. lÃƒÆ’Ã‚Â­mites/entitlements y restricciones de seguridad relevantes.

El gate Runtime autoriza la admisiÃƒÆ’Ã‚Â³n de una invocaciÃƒÆ’Ã‚Â³n en el instante de decisiÃƒÆ’Ã‚Â³n. RevocaciÃƒÆ’Ã‚Â³n bloquea nuevas admisiones; no se afirma cancelaciÃƒÆ’Ã‚Â³n retroactiva de un request externo ya iniciado. El executor Waby debe revalidar en el lÃƒÆ’Ã‚Â­mite del efecto externo para minimizar la ventana de carrera y evitar que una autorizaciÃƒÆ’Ã‚Â³n genÃƒÆ’Ã‚Â©rica se reutilice.

### 7. AuditorÃƒÆ’Ã‚Â­a

IntegrationInvocation es el concepto de auditorÃƒÆ’Ã‚Â­a WAB-78. Debe correlacionar Organization, Workspace, AgentInstance, Execution, integration/operation/version, decisionId, scope permitido, timestamps y outcome. Se registra allow/deny y errores de autorizaciÃƒÆ’Ã‚Â³n sin secretos, tokens ni payload sensible por defecto. La persistencia/atomicidad/retenciÃƒÆ’Ã‚Â³n fÃƒÆ’Ã‚Â­sica no se inventa en este incremento.

### 8. ReutilizaciÃƒÆ’Ã‚Â³n / no reutilizaciÃƒÆ’Ã‚Â³n

- Reutilizar: CRM tenancy/ownership y AgentInstance status; service-to-service JWT como patrÃƒÆ’Ã‚Â³n, no token direccional sin reconciliaciÃƒÆ’Ã‚Â³n; AgentRuntimeContext executionId/identity; cuatro-authority snapshot resolver para policy estÃƒÆ’Ã‚Â¡tica; fail-closed executor.
- No reutilizar: GraphQL wrk.agents.* como runtime grant; snapshot allowlist como autorizaciÃƒÆ’Ã‚Â³n live; Credential Registry de proveedores LLM como credential registry de integraciones; legacy Bot CapabilityRegistry/CapabilityContext.
- No duplicar: grants/catalog/connection config en AI Service DB.

### 9. Bloqueo de implementaciÃƒÆ’Ã‚Â³n fÃƒÆ’Ã‚Â­sica

No se encontrÃƒÆ’Ã‚Â³ schema existente de Integration/Operation/Invocation ni source live. Antes de implementar endpoint/modelos se requiere asignar ownership/incremento y reconciliar con WAB-78, WAB-79 y WAB-80. El gate fail-closed existente se mantiene; ninguna capability externa se habilita.

Este contrato es una recomendaciÃƒÆ’Ã‚Â³n de diseÃƒÆ’Ã‚Â±o local para revisiÃƒÆ’Ã‚Â³n; no constituye aprobaciÃƒÆ’Ã‚Â³n de endpoint/schema ni implementaciÃƒÆ’Ã‚Â³n fÃƒÆ’Ã‚Â­sica. No se ejecutaron tests/type-check/build/lint; no commit/push/deploy.

### 10. DTO lÃƒÆ’Ã‚Â³gico candidato y secuencia (sin crear endpoint)

Request candidate:
- executionId, organizationId, workspaceId, agentInstanceId, agentDefinitionId, snapshotId, effectiveConfigHash.
- capabilityId y toolCallId.
- Binding server-registered: connectorDefinitionId, integrationId, operationId, operationVersion, connectionRevision.
- resourceScope tipado, derivado por el adapter autorizado (por ejemplo { resourceType, resourceId }); no se acepta scope libre que el LLM use para ampliar acceso.

Response candidate:
- decision (allow | deny), reasonCode estable, decisionId, checkedAt, authorityRevision.
- binding echo con IDs/version/revision verificados; mismatch invalida la respuesta.
- expiresAt opcional para ticket de una sola invocaciÃƒÆ’Ã‚Â³n; si existe, se vincula al execution/toolCall/binding y no es reutilizable. No cachear decisiones allow generales.

Secuencia candidata:
1. Registry entrega capability + binding confiable al executor; ToolCall solo selecciona el nombre registrado y argumentos.
2. AI Service gate llama a CRM con service identity direccional y scope de autorizaciÃƒÆ’Ã‚Â³n mÃƒÆ’Ã‚Â­nimo.
3. CRM valida tenancy/status/config/operation/connection y revocaciÃƒÆ’Ã‚Â³n contra estado autoritativo actual; responde deny o evidencia versionada.
4. Runtime valida identidad/binding de la respuesta; cualquier mismatch/timeout/unavailable deniega.
5. El executor Waby revalida controles globales/credential state y, en el boundary del efecto externo, consume autorizaciÃƒÆ’Ã‚Â³n vinculada a esa invocaciÃƒÆ’Ã‚Â³n. Credential resolution ocurre solo despuÃƒÆ’Ã‚Â©s del allow.
6. Se registra IntegrationInvocation/audit correlacionado; no secretos ni payload sensible por defecto.

**Punto de linearizaciÃƒÆ’Ã‚Â³n:** el allow del gate admite una invocaciÃƒÆ’Ã‚Â³n en el instante de check. Una revocaciÃƒÆ’Ã‚Â³n posterior impide nuevas admisiones, pero no puede garantizar cancelar una llamada externa ya iniciada. Si la integraciÃƒÆ’Ã‚Â³n soporta cancelaciÃƒÆ’Ã‚Â³n, se aplicarÃƒÆ’Ã‚Â¡ separadamente; no prometer rollback del efecto remoto.

### 11. ReconciliaciÃƒÆ’Ã‚Â³n de alcance Linear (20/11/2026)

VerificaciÃƒÆ’Ã‚Â³n directa de WAB-80, WAB-79 y WAB-94:
- WAB-80 (Provisioning y autorizaciÃƒÆ’Ã‚Â³n) define administraciÃƒÆ’Ã‚Â³n de AgentInstance, Knowledge y subrecursos Skills/Rules/integraciones bajo wrk.agents.*; no especifica live authorization por capability/tool call ni schemas Integration/Operation/Invocation.
- WAB-79 padre solo define incrementos delimitados despuÃƒÆ’Ã‚Â©s de A1; no autoriza ampliar A1 ni anticipar contratos fÃƒÆ’Ã‚Â­sicos de integrations.
- WAB-94 A1 explÃƒÆ’Ã‚Â­citamente excluye provisioning/WAB-80, permisos wrk.agents.*, GraphQL, endpoints/controllers y cambios a AI Service. Su estado actual Backlog.
- Por tanto, el endpoint/modelos propuestos no se implementan dentro de WAB-81 ni se aÃƒÆ’Ã‚Â±aden unilateralmente a A1/WAB-94. El ownership/incremento debe autorizarse explÃƒÆ’Ã‚Â­citamente en Linear antes de la implementaciÃƒÆ’Ã‚Â³n CRM.

Resultado: contrato propuesto y gate fail-closed quedan documentados. El siguiente paso de cÃƒÆ’Ã‚Â³digo requiere scope task autorizado para los contratos fÃƒÆ’Ã‚Â­sicos Integration/Operation/Invocation + CurrentAuthorization source, con dependencias y AC definidos; no se reabre WAB-78 ni se modifica el alcance aprobado de A1.


## Ãƒâ€šÃ‚Â§49 ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â DelimitaciÃƒÆ’Ã‚Â³n y registro del incremento A2 (20/11/2026)

Tras reconciliar los contratos conceptuales de WAB-78 y las restricciones de WAB-79/WAB-80/WAB-94, se creÃƒÆ’Ã‚Â³ en Linear el incremento explÃƒÆ’Ã‚Â­cito **WAB-96 ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â A2: Integraciones de AgentInstance y CurrentAuthorization runtime**: https://linear.app/waby/issue/WAB-96/a2-integraciones-de-agentinstance-y-currentauthorization-runtime. Estado Backlog; prioridad High; relacionado con WAB-78, WAB-79, WAB-80, WAB-81 y WAB-94; parent WAB-75.

Scope: modelos/contratos fÃƒÆ’Ã‚Â­sicos mÃƒÆ’Ã‚Â­nimos de integraciÃƒÆ’Ã‚Â³n, operaciÃƒÆ’Ã‚Â³n registrada/versionada e invocaciÃƒÆ’Ã‚Â³n/auditorÃƒÆ’Ã‚Â­a; autoridad CRM CurrentAuthorization basada en ownership, estado, configuraciÃƒÆ’Ã‚Â³n enabled, binding/revisiÃƒÆ’Ã‚Â³n, revocaciÃƒÆ’Ã‚Â³n y scope; autenticaciÃƒÆ’Ã‚Â³n direccional AI ServiceÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢CRM; DTO request/decision; conectar gate fail-closed; revalidaciÃƒÆ’Ã‚Â³n en boundary de Integration Executor; resoluciÃƒÆ’Ã‚Â³n de credential tras allow; auditorÃƒÆ’Ã‚Â­a y tests de allow/deny, revocaciÃƒÆ’Ã‚Â³n, tenant isolation, mismatch, timeout/unavailable y service auth.

Ownership: CRM autoritativo de tenant/AgentInstance y configuraciÃƒÆ’Ã‚Â³n/revocaciÃƒÆ’Ã‚Â³n; Waby dueÃƒÆ’Ã‚Â±o del catÃƒÆ’Ã‚Â¡logo global, executor, polÃƒÆ’Ã‚Â­tica global, compatibilidad de Definition y credential custody/resolution; AI Service solicita decisiÃƒÆ’Ã‚Â³n usando identity/binding confiables. No se crea registry paralelo de grants si AgentInstanceOperation enabled representa Instance Authority. wrk.agents.* sigue siendo administraciÃƒÆ’Ã‚Â³n, no runtime grant. No reutilizar LLM provider Credential Registry como credenciales empresariales.

Restricciones: WAB-94/A1 no se amplÃƒÆ’Ã‚Â­a; no asumir endpoints/schemas existentes sin inspecciÃƒÆ’Ã‚Â³n; no fail-open/default grants; no arbitrary URL/HTTP/shell/filesystem; no cambios frontend/infrastructure/repositorios no autorizados; local-first, sin commit/push/deploy.

Linear WAB-81 actualizado con el scope y link a WAB-96. Esta delimitaciÃƒÆ’Ã‚Â³n no implica implementaciÃƒÆ’Ã‚Â³n fÃƒÆ’Ã‚Â­sica ni valida el contrato final. WAB-96 queda Backlog y debe comenzar con inspecciÃƒÆ’Ã‚Â³n de convenciones/evidencia en CRM, AI Service y executor, identificando dependencias reales antes de modificar cÃƒÆ’Ã‚Â³digo.

No se modificÃƒÆ’Ã‚Â³ cÃƒÆ’Ã‚Â³digo en este checkpoint. No se ejecutaron tests/typecheck/build/lint.


## Ãƒâ€šÃ‚Â§50 ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â InspecciÃƒÆ’Ã‚Â³n tÃƒÆ’Ã‚Â©cnica focalizada de WAB-96 (20/11/2026)

InspecciÃƒÆ’Ã‚Â³n directa de F:\ElemenSoft. No se modificÃƒÆ’Ã‚Â³ cÃƒÆ’Ã‚Â³digo ni documentaciÃƒÆ’Ã‚Â³n canÃƒÆ’Ã‚Â³nica; se actualizaron ÃƒÆ’Ã‚Âºnicamente comentarios de continuidad/Linear. No se ejecutaron tests/typecheck/build/lint.

### Estado de repositorios locales
- CRM: F:\ElemenSoft\crm; working tree muy sucio con cambios y untracked de WAB-79/WAB-81 y otros trabajos. No limpiar, revertir, stagear ni asumir autorÃƒÆ’Ã‚Â­a.
- WABY_frontend: master behind origin/master por 4 commits, limpio en inspecciÃƒÆ’Ã‚Â³n.
- infrastructure: limpio.
- waby-docs: numerosas modificaciones y untracked, incluidos audits locales. No commit/push.
- wabyland: repo limpio y sincronizado; frontend React/Vite de marketing, no backend ni Integration Executor.
- No se encontrÃƒÆ’Ã‚Â³ otro repo local propietario del Integration Executor/credential custody. Esto no prueba inexistencia remota o en otro workspace.

### Evidencia CRM
- AgentInstance fÃƒÆ’Ã‚Â­sica y AgentInstanceService existen. Modelo base contiene Workspace, Definition, status y configuraciÃƒÆ’Ã‚Â³n; no integration/connection/operation/credential binding.
- Service valida OrganizationÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢Workspace y ownership de Workspace, administra instancia/Knowledge, no integraciones empresariales.
- No localizados modelos/repositorios/servicios/rutas CRM para AgentInstanceIntegration, AgentInstanceOperation, IntegrationInvocation ni CurrentAuthorization runtime.
- Ruta interna AI ServiceÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢CRM de AssignmentSnapshot implementa RS256 direccional, issuer/audience/scope, workspace claim y replay protection. Es especÃƒÆ’Ã‚Â­fica de snapshots; no se reutiliza como authorization endpoint sin nuevo contrato.
- CRMÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢AI Service service JWT es distinto issuer/audience/subject. La direcciÃƒÆ’Ã‚Â³n AI ServiceÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢CRM necesita contrato explÃƒÆ’Ã‚Â­cito.
- AgentRuntimePolicyResolver expone cuatro authorities/intersecciÃƒÆ’Ã‚Â³n, pero fuente concreta no estÃƒÆ’Ã‚Â¡ implementada/verificada. AgentRuntimeSnapshotService usa FailClosed resolver default.

### Evidencia AI Service
- AgentInstance capability registry/executor separados del legacy Bot.
- CurrentAuthorization gate se ejecuta antes de capability.execute; default FailClosed deniega. Request incluye durable execution, tenant, instance/definition, snapshot/hash, capability/toolCall/parameters; aÃƒÆ’Ã‚Âºn no tiene trusted connector/integration/operation/version/connection binding.
- Composition root instancia registry in-memory vacÃƒÆ’Ã‚Â­o y executor fail-closed; no authority adapter real.
- agent-runtime.routes.ts solo snapshot resolve (Definition/LLMRoute), no CurrentAuthorization ni integration routes.
- Service JWT middleware AI ServiceÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢CRM no implementa una llamada saliente ni route de autorizaciÃƒÆ’Ã‚Â³n inversa.
- Legacy capabilities search_contacts/get_contact/count_contacts/assign_contact/get_last_messages son CRM/Bot internas mediante gateways/use cases; no son external ConnectorDefinition ni deben migrarse automÃƒÆ’Ã‚Â¡ticamente a AgentInstance.
- LLM provider-account credential registry presente en cambios locales de AI Service (Mongo/AES-GCM) es dominio diferente de credenciales empresariales de integraciones; no reutilizar.

### Evidencia Waby executor
Wabyland local es frontend marketing, sin backend, executor, connector catalog o credential custody. No se identificÃƒÆ’Ã‚Â³ localmente el repo propietario de Integration Executor. A2.4 queda bloqueado hasta identificar repo/owner y autorizaciÃƒÆ’Ã‚Â³n de cambios.

### Plan tÃƒÆ’Ã‚Â©cnico recomendado ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â dividir A2
1. **A2.1 CRM integration domain/persistence:** AgentInstanceIntegration, operaciÃƒÆ’Ã‚Â³n enabled/version binding, referencia protegida (sin secret), invocation/audit, tenant indexes/ownership. Reconciliar contrato canÃƒÆ’Ã‚Â³nico y no ampliar A1.
2. **A2.2 CRM CurrentAuthorization authority + endpoint:** revalidaciÃƒÆ’Ã‚Â³n ownership/status/config/revocation/scope; DTO, reason codes, binding echo; service identity AI ServiceÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢CRM RS256 issuer/audience/sub/scope/key rotation/replay.
3. **A2.3 AI Service adapter/gate:** trusted binding metadata por capability; solicitud derivada de execution context durable; validaciÃƒÆ’Ã‚Â³n de response/decision/binding, fail-closed, timeout/cancel; no authority desde parÃƒÆ’Ã‚Â¡metros de ToolCall.
4. **A2.4 Waby Integration Executor/credential custody:** global policy, definition compatibility, live instance authority y credential health en effect boundary; secret resolver solo tras allow; outbound/SSRF controls; invocation audit. Requiere repo/owner desconocido y issue/dependency separado.
5. Tests cross-cutting tras implementaciÃƒÆ’Ã‚Â³n completa: tenant isolation, status/revocation, operation/version/connection mismatch, JWT invalid/replay/unavailable, timeout, redaction/no-secret leakage.

A2 completo es demasiado amplio para un solo pase. Recomendado dividir en A2.1 CRM, A2.2 transporte/endpoint, A2.3 AI Service gate y A2.4 executor/custody como issue separado una vez identificado el repo. WAB-81 permanece fail-closed; ninguna capability se habilita hasta existir enforcement real en gate y executor.

Linear WAB-96 y WAB-81 recibieron comentarios con esta evidencia y secuencia. InspecciÃƒÆ’Ã‚Â³n limitada a los repos locales existentes; no se auditÃƒÆ’Ã‚Â³ org GitHub remota ni workspace no conectado.

## Ãƒâ€šÃ‚Â§51 ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Â DescomposiciÃƒÆ’Ã‚Â³n A2 y revisiÃƒÆ’Ã‚Â³n documental (20/11/2026)

### RevisiÃƒÆ’Ã‚Â³n: Ãƒâ€šÃ‚Â¿ya existÃƒÆ’Ã‚Â­a en Linear y documentaciÃƒÆ’Ã‚Â³n?

SÃƒÆ’Ã‚Â­, los contratos conceptuales ya existÃƒÆ’Ã‚Â­an; no se debe reabrir diseÃƒÆ’Ã‚Â±o. WAB-78 y `design/integraciones-api-externas.md` describen AgentInstanceIntegration, operaciones, IntegrationInvocation/auditorÃƒÆ’Ã‚Â­a, credencial protegida, executor/catÃƒÆ’Ã‚Â¡logo Waby, ownership y autorizaciÃƒÆ’Ã‚Â³n contextual. Sin embargo, faltaba una descomposiciÃƒÆ’Ã‚Â³n de implementaciÃƒÆ’Ã‚Â³n y la verificaciÃƒÆ’Ã‚Â³n de fuentes fÃƒÆ’Ã‚Â­sicas. Linear WAB-96 estaba como tarea monolÃƒÆ’Ã‚Â­tica y no enumeraba subbloques ni dependencias.

### Linear actualizado

- WAB-96 actualizado con descomposiciÃƒÆ’Ã‚Â³n A2.1ÃƒÂ¢Ã¢â€šÂ¬Ã¢â‚¬Å“A2.4.
- WAB-97 A2.1 CRM: persistencia de integraciones, operaciones y binding.
- WAB-98 A2.2 CRM: CurrentAuthorization y endpoint direccional AI ServiceÃƒÂ¢Ã¢â‚¬Â Ã¢â‚¬â„¢CRM; bloqueado por WAB-97.
- WAB-99 A2.3 AI Service adapter/gate fail-closed; bloqueado por WAB-98.
- WAB-100 A2.4 discovery del owner de Integration Executor y boundary de efecto; discovery paralelo, implementaciÃƒÆ’Ã‚Â³n dependiente de repo confirmado y contratos.
- Comentario de evidencia aÃƒÆ’Ã‚Â±adido a WAB-96.

### Repositorios y actualizaciÃƒÆ’Ã‚Â³n

- `waby-docs`: `git fetch origin` ejecutado. Rama master local estaba sincronizada con origin/master (0 ahead, 0 behind); working tree continÃƒÆ’Ã‚Âºa dirty con cambios locales previos. No se integraron ni publicaron cambios.
- `crm`: `git fetch origin` ejecutado. Rama `waby/reconcile-master-2026-11-19` diverge: 2 commits locales ahead y 24 commits de origin/master behind. Working tree contiene extensos cambios locales y untracked de WABY. No se hizo merge/rebase/stash para evitar conflicto, pÃƒÆ’Ã‚Â©rdida o alterar el estado de trabajo; no se subiÃƒÆ’Ã‚Â³ ningÃƒÆ’Ã‚Âºn cambio.
- `wabyland` es frontend marketing, no se asumirÃƒÆ’Ã‚Â¡ backend executor. No se confirmÃƒÆ’Ã‚Â³ aÃƒÆ’Ã‚Âºn repo remoto/otro workspace de Integration Executor.

### Secuencia para acelerar

Prioridad: comenzar WAB-97 inmediatamente; en paralelo, discovery WAB-100. Luego WAB-98 y WAB-99 en secuencia contractual. A2.4 implementaciÃƒÆ’Ã‚Â³n se habilita tras identificar el owner y reconciliar contratos. No ejecutar tests/build/typecheck/lint en este checkpoint.

## 52 - WAB-97 A2.1 CRM integration persistence (20/11/2026)

### Added local CRM files

- `src/models/agent/integrations/AgentInstanceIntegration.model.ts`: Organization/Workspace/AgentInstance binding, connector ID/version, lifecycle status, non-secret connectionConfig, protected credential reference with select=false, revision, actor/timestamps/revocation; tenant and instance indexes.
- `src/models/agent/integrations/AgentInstanceOperation.model.ts`: catalog operation binding by ID/version, read/write/sensitive effect, enabled default false, non-secret operationConfig; unique integration/operation/version index.
- `src/models/agent/integrations/IntegrationInvocation.model.ts`: execution-correlated audit metadata; status/timing/HTTP/error/correlation/attempt/actor. No request/response, headers, credential or secret fields.
- Integration-specific Typegoose model registry, three repositories, AgentInstanceIntegrationService and config validator.
- Service operations: create/list/get/update integration; create/list/update operation; record/list invocation. Organization-to-Workspace-to-AgentInstance ownership validated. Credential reference excluded from default query and create response. Config recursively rejects sensitive keys. Revocation terminal and removes credential reference. Active status requires reference. Clearing credential from active connection disables it. Audit metadata validates dates, attempt, errorCode and correlationId.
- Tests for schema/index contracts and secret-key validation.

### Validation

- Jest targeted: 2 suites, 12 tests passed.
- ESLint targeted: 0 errors, 0 warnings.
- `tsc --noEmit`: blocked by pre-existing missing dependency `@aws-sdk/s3-request-presigner` in `src/modules/media/urlFirmadaEvolution.ts`; no WAB-97 errors were reported.
- `git diff --check`: clean.

### Boundaries

No GraphQL/REST endpoints, CurrentAuthorization, credential secret custody/resolution or external executor. Connector catalog/schema enforcement remains Waby-owned and requires A2.4. WAB-97 marked Done in Linear. Changes are local only; no commit/push.
## 53 - WAB-98 A2.2 CRM CurrentAuthorization (20/11/2026)

**Estado:** implementacion local, sin commit/push/deploy.

### Implementado

- `src/services/CurrentAuthorization.service.ts`: autoridad CRM live; request bounded/validated y decision allow/deny con reason codes estables, evaluatedAt y binding echo.
- Verifica Organization->Workspace, Workspace activo, AgentInstance dentro del Workspace y estado `active`; integration pertenece exactamente a org/workspace/instance, estado `active`, connector/version y revision coinciden; credential reference existe; operation/version binding exacto y enabled.
- La referencia de credential solo se verifica como presencia y nunca se retorna. No se resuelve secreto ni se crea grant registry.
- `src/routes/current-authorization.route.ts`: `POST /api/internal/current-authorization`, AI Service->CRM JWT direction-specific RS256, issuer/audience/sub/scope/kid/exp/iat/jti, key rotation current/previous, workspaceId+executionId binding, replay-store Mongo con fail-closed.
- Scope requerido: `agent.integration.authorization.current`. JWT claims requeridos: `sub=ai-service`, `workspaceId`, `executionId`, `scope`, `jti`, `iat`, `exp`. Config keys: `AI_SERVICE_TO_CRM_JWT_PUBLIC_KEY_KID`, `AI_SERVICE_TO_CRM_JWT_PUBLIC_KEY`, optional previous key/kid, issuer/audience vars.
- Request fields: organizationId, workspaceId, agentInstanceId, agentDefinitionId, snapshotId, effectiveConfigHash, executionId, integrationId, connectorId, connectorVersion, integrationRevision, operationId, operationVersion, capabilityId.
- Test `CurrentAuthorization.service.test.ts`: 8 tests allow/deny, inactive instance, org mismatch, revision mismatch, missing credential ref, disabled op, authority unavailable, request validation.

### Validacion

- Jest focalizado: 1 suite / 11 tests PASS.
- ESLint focalizado: 0 errors; inicialmente 2 warnings `no-explicit-any` corregidos; revalidacion pendiente.
- `tsc --noEmit`: falla solo por dependencia preexistente no instalada `@aws-sdk/s3-request-presigner` en `src/modules/media/urlFirmadaEvolution.ts`; no reporto error de WAB-98 en la salida.
- `git diff --check`: PASS.
- No E2E; no commit/push/deploy.

### Limites / contrato a completar en A2.3/A2.4

- El CRM no mantiene agentDefinitionId en AgentInstance persistence actual; definition/snapshot/hash son correlation binding echoed, no authority source. A2.3 debe validar el binding con el trusted execution snapshot y no inferir capability authorization del snapshot.
- `capabilityId` debe coincidir con operationId persistido; la identidad del connector/operation catalog y credential resolution pertenece a Waby executor A2.4.
- La revision de integration funciona como fencing/config binding; cualquier cambio de connection config/status/credential ref debe incrementar revision (A2.1 update ya incrementa revision).
- La autoridad evalua estado actual al momento de la consulta; A2.4 debe revalidar justo antes del efecto externo, no tratar este allow como permiso durable.



## 54 - WAB-99 A2.3 AI Service CurrentAuthorization adapter/gate (20/11/2026)

**Estado:** implementacion local parcial/completada de adapter y gate; WAB-99 permanece In Progress por fuente trusted capability-binding no disponible.

### Implementado

- `agent-runtime-current-authorization-binding.ts`: contrato trusted binding resolver; binding incluye integrationId, connectorId/version, integrationRevision, operationId/version y capabilityId. Default resolver es fail-closed (null). No se obtiene binding desde ToolCall.parameters ni LLM output.
- `agent-runtime-current-authorization-http-gate.ts`: adapter HTTP AI Service->CRM, request `POST {CRM_BASE_URL}/api/internal/current-authorization`; firma service JWT RS256 (issuer/audience/sub/scope/jti/iat/exp + workspaceId/executionId), usa kid, key privada y timeout/cancel. Valida envelope, decision/reason/evaluatedAt y binding echo exacto; sólo acepta allow + AUTHORIZED. Deny, mismatch, invalid response, HTTP, unavailable, timeout, cancel y missing key/binding fallan closed.
- Composición AgentInstance Runtime conecta Registry executor con adapter HTTP cuando private key + kid existen; de otro modo usa FailClosed gate.
- Config/env añadidos: `AI_SERVICE_TO_CRM_JWT_PRIVATE_KEY`, `AI_SERVICE_TO_CRM_JWT_KID`, issuer/audience y timeout. Variables añadidas a `microservices/ai-service/.env.example`. Private key no se comparte ni loguea.
- Tests gate + executor: 12 casos PASS; incluyen que capability no ejecuta ante live deny, sí ejecuta con allow mock, JWT claims direccionales, binding echo, missing resolver/key, HTTP/malformed/network/timeout/cancel.

### Validacion

- AI Service `tsc --noEmit`: PASS.
- Jest focalizado: 2 suites / 12 tests PASS.
- ESLint focalizado: 0 errors; composition-root muestra 19 warnings preexistentes de unused imports/vars, no introducidos por este cambio.
- `git diff --check`: PASS.
- No E2E contra CRM real; no commit/push/deploy.

### Bloqueo contractual

No existe en los snapshots runtime actuales un trusted binding capability -> AgentInstanceIntegration/connector/operation/version/revision. El resolver default permanece fail-closed y no se registra ninguna capability. WAB-97 persiste integraciones/operaciones, pero no se ha implementado ni verificado un endpoint/snapshot trusted para resolver bindings por AgentInstance/Execution; tampoco se puede inferir ese binding de ToolCall, parameters, skills, prompt o policy snapshot. Para habilitar capacidades reales falta acordar/conectar esa fuente trusted (CRM A2.2/A2.4 owner), y luego registrar capabilities explícitas dentro del catálogo aprobado.

WAB-99 se mantiene In Progress, no Done, hasta resolver ese binding fuente y demostrar una operación autorizada end-to-end. El adapter/gate no abre capabilities por sí solo.
## WAB-100 A2.4 discovery — 20/11/2026

**Resultado: owner/repository del Waby Integration Executor no identificado. WAB-100 continúa In Progress / bloqueado para implementación.** No se creó un executor ni se añadieron capacidades externas a CRM o AI Service.

### Revisión documental y Linear
- Revisado el contrato existente en `design/integraciones-api-externas.md`, `design/fase-2-diseno-objetivo-y-contratos.md` y el contexto de WAB-96/97/98/99.
- El diseño mantiene el executor/catálogo global y custodia de secretos como responsabilidad Waby; CRM persiste el binding de AgentInstanceIntegration/Operation y CurrentAuthorization; AI Service realiza pre-gate fail-closed.
- WAB-100 conserva alcance de discovery + implementación solo tras confirmar el repo/owner. No se reabren los contratos A2.1–A2.3.

### Discovery repositorios locales
Inspeccionados los repositorios existentes bajo `F:\ElemenSoft`: `crm`, `infrastructure`, `wabyland`, `WABY_frontend`. Búsqueda local textual de IntegrationExecutor, ConnectorDefinition, CredentialCustody y credential resolver no encontró implementación (no se encontraron coincidencias).

- `wabyland`: frontend marketing React/Vite; no es backend executor.
- `infrastructure`: repo DevOps/shared infra; no se encontró código de executor/catálogo/custodia.
- `crm`: contiene los incrementos locales WAB-97/98 y AI Service WAB-99; no debe asumir ownership del efecto externo.
- `WABY_frontend`: frontend de plataforma, no executor backend.

### Discovery GitHub remoto
La integración GitHub permitió listar los repositorios visibles de ElemeSoft: `crm`, `WABY_frontend`, `eleme-web`, `Tareas`, `MIX`, `AG_TickeraMIX_frontend`, `infrastructure`, `waby-bot`, `wabyland`, `waby-Baileys`, `waby-evolution`. No hay repo cuyo nombre identifique Integration Executor/Connector Registry.

Inspección de metadatos:
- `ElemeSoft/waby-bot`: servicio Node/TS con AI SDK, Express, RabbitMQ, Redis; no se identificó contractualmente como executor global de integraciones.
- `ElemeSoft/waby-evolution`: fork de Evolution API/WhatsApp transport, no es el owner presumible del executor empresarial genérico.
- `ElemeSoft/wabyland`: frontend marketing.
- Búsquedas de código GitHub para Integration Executor, ConnectorDefinition, connector y credential resolver no produjeron coincidencias; la indexación de código de repositorios es parcial, por lo que esto no prueba ausencia de código.
- No se pudo verificar repositorio de otra organización, workspace GitHub privado no listado, repo sin indexación, ni servicio externo.

### Conclusión y bloqueo
No existe evidencia suficiente para atribuir Waby Integration Executor a `waby-bot`, `crm`, `waby-evolution` o `infrastructure`. No se implementa en un repo supuesto ni se duplica catálogo/custodia.

Para destrabar A2.4 se necesita identificar el repositorio/servicio que actualmente posee (o será designado para poseer) el catálogo de conectores globales y la ejecución/custodia de credenciales empresariales. Una vez confirmado, revisar su arquitectura y reconciliar la boundary de efecto:
1. Revalidar CurrentAuthorization live inmediatamente antes del efecto, vinculando binding/version/revision.
2. Resolver secretos solo después de allow, nunca retornar a AI Service/LLM/logs.
3. Connector/operation allowlist y schemas versionados.
4. Controles de destino/SSRF/DNS/redirecciones/network egress.
5. Límites/timeouts, respuesta redactada, audit/correlation e idempotencia/retry.
6. Fail-closed para deny/stale/mismatch/secret/network/policy.

WAB-99 sigue fail-closed y no registra capabilities. No E2E. No commits, pushes, deploys ni cambios de repositorio realizados durante este discovery.


## 56 - WAB-100 discovery corregido: arquitectura real de CRM e infrastructure (20/11/2026)

### Correccion de alcance

La inspeccion previa de WAB-100 (§55) asumio incorrectamente que el owner del executor podia estar en un repositorio/backend distinto de CRM. El usuario aclaro la estructura real: `crm/` es el backend principal y contiene todos los microservicios en `crm/microservices/`; `WABY_frontend/` es el frontend de plataforma; `infrastructure/` es un repositorio separado con servicios compartidos de soporte al backend. Se excluye `waby-bot` de cualquier analisis (proyecto deprecated/pending deletion). `waby-evolution` es un fork de Evolution API para WhatsApp/Baileys; `wabyland` es marketing frontend.

### Evidencia inspeccionada localmente

Repositorios en `F:\\ElemenSoft`:
- `crm/`: backend principal, con `src/` (models, repositories, services, routes, modules, Workers, Rabbit, infrastructure, GraphQL, etc.) y `microservices/`.
- Microservicios presentes: `ai-service`, `delete-contacts`, `delete-wrk`, `Difusion`, `insert-contacts`, `observability`, `sync-evo-contacts`.
- `crm/microservices/ai-service/`: capas `application`, `domain`, `engine`, `infrastructure`, `observability`, `presentation`; transportes RabbitMQ para runtime y Hono HTTP para management/internal APIs.
- `infrastructure/`: RabbitMQ, Redis, Evolution API + PostgreSQL, MinIO, Qdrant y stack opcional observability (OTel Collector, Prometheus, Grafana, Tempo, Loki, Alertmanager, redis-exporter). No contiene catalogo de conectores empresariales ni executor de operaciones AgentInstance en la configuracion inspeccionada.

### Hallazgos relevantes CRM/AI Service

- `ai-service/src/engine/execution-engine.ts` implementa `executeAgentRuntime` y `executeAgentRuntimeToolLoop`; es el motor de orquestacion compartido existente.
- `application/services/agent-instance-execution-runtime.ts` compone knowledge, context, invocacion LLM y tool loop; el limite de iteraciones es 5. Capability registry/executor se inyectan.
- `application/services/agent-runtime-capabilities.ts` define los contratos y defaults vacios/fail-closed; no expone legacy Bot capabilities automaticamente.
- `application/services/agent-runtime-capability-executor.ts` valida schema, revisa capability permitida, llama CurrentAuthorization gate y luego ejecuta capability registrada. No implementa connector HTTP ni credential custody.
- Composition root crea `InMemoryAgentRuntimeCapabilityRegistry` vacio. El gate HTTP de WAB-99 esta cableado si hay keys, pero el binding resolver sigue fail-closed/no configurado; no hay capability de integracion registrada.
- CRM tiene persistencia WAB-97: `AgentInstanceIntegration`, `AgentInstanceOperation`, `IntegrationInvocation`; `AgentInstanceIntegrationService` CRUD/validacion/registro de auditoria. `protectedCredentialRef` es opaco/select=false; configs rechazan claves sensibles; invocation no persiste payload/headers/secrets.
- CRM `CurrentAuthorization.service.ts` (WAB-98) revalida workspace/org, AgentInstance activa, integration activa y binding/revision, referencia credential presente, operation version/enabled/capability binding. No resuelve ni entrega el secreto.
- CRM route `POST /api/internal/current-authorization` autentica service JWT RS256 direccional AI Service->CRM, claims/scope/context, replay record fail-closed.
- No se encontro resolver/custodia real para `protectedCredentialRef`, ConnectorDefinition catalog global ni executor HTTP empresarial en `crm/src`, `crm/microservices/ai-service/src` o `infrastructure` (busqueda textual excluyendo node_modules/dist/.git).
- CRM `docker-compose.yml` incluye `microservice-ai-service` y lo conecta a `backend_network`, `rabbit_net`, `redis_net`; infrastructure define dichas redes externas y servicios de soporte. No hay microservicio Integration Executor separado identificado.

### Conclusion arquitectonica (evidencia + propuesta, no contrato aprobado)

La ubicacion candidata mas coherente para el runtime executor/connector dispatch es el existente `crm/microservices/ai-service`, dentro del boundary de AgentInstance Runtime/capabilities, porque recibe ToolCalls y ya tiene execution loop/capability registry. CRM core mantiene autoridad/persistencia/bindings y CurrentAuthorization; `infrastructure/` provee dependencias operativas (red/colas/cache/storage/observability), no logica de negocio de connectors. No se propone un nuevo repo o microservicio.

Esto NO resuelve todavia credential custody: el executor no debe leer directamente un protectedCredentialRef como si fuera secreto ni enviar secrets al LLM. Falta identificar/diseñar un mecanismo autorizado de resolver secreto (posiblemente boundary interno CRM/AI Service o vault/secret service existente; la inspeccion no encontro uno). El mecanismo requiere decision contractual antes de implementacion.

### Estado de trabajo y limites

- Discovery corregido a repositorios reales locales, sin modificar codigo de CRM ni infrastructure.
- `crm` working tree tiene cambios locales WAB-97/98/99 y otros cambios WABY; preservados, sin checkout/reset/stash.
- `infrastructure` working tree limpio en la inspeccion.
- `waby-docs` working tree ya tenia cambios/untracked locales; el presente informe se append-only, no se alteraron otros archivos.
- No tests/build/lint se ejecutaron para discovery.
- No commit/push/deploy.

### Siguiente paso WAB-100

Reconciliar con los contratos A2.1-A2.3 y el diseno `integraciones-api-externas.md` la ubicacion del dispatch/catalog executor dentro del AI Service existente, y definir explicitamente credential resolution/custody boundary. Antes de implementar, inspeccionar si CRM/infrastructure disponen de secret-management runtime no identificado en la primera busqueda (config/deploy/env/volumes/service endpoints) y precisar network policy/SSRF. Luego cerrar un contrato minimo para trusted binding resolver, catalog operation/schema, effect execution, response redaction, audit e idempotency/retry; implementar solo dentro del repo existente confirmado.


### 56.1 - Wiring refinement

A follow-up checked CRM `src/utils/autoloadRoutes.ts` convention and `src/server.ts`: route files under `src/routes/` are auto-mounted at `/api`; therefore `current-authorization.route.ts` is not missing merely because no explicit import appears in server/index. The route is expected to be autoloaded by this convention. Conversely, `agentInstanceIntegrationService` is only referenced at its declaration in current CRM source (excluding tests); its management operations have no GraphQL/REST/controller wiring identified in this inspection. Treat WAB-97 persistence/service as local scaffolding until the admin API wiring is separately verified/implemented, not as available UI/API functionality.


## 57 - WAB-100 implementación local del Integration Executor (20/11/2026)

### Decisión/alcance implementado

- Boundary confirmado en CRM core para tenancy, configuración, CurrentAuthorization y custodia; AI Service existente para AgentInstance runtime, capabilities y tool loop. Sin repo/microservicio nuevo.
- Connector catalog Waby allowlisted local CRM: `http-v1@1.0.0`, HTTPS public origin, auth none/bearer/api-key, GET-only, redirects false.
- Integration admin REST auto-mounted en CRM: create/list/patch integration, create/list/patch operation; guards `wrk.agents.read` / `wrk.agents.*` y `resolveAuthContext` tenant ownership.
- Config no-secret connection; credential input sólo en management API, encrypted AES-256-GCM en colección `integration_credential_secrets`, binding exacto Organization+Workspace+AgentInstance+Integration, ref opaca. Requiere `CRM_AGENT_INTEGRATION_CREDENTIAL_ENCRYPTION_KEY` base64 32-byte key.
- Create authMode none genera secret marker cifrado para mantener ref requerida por CurrentAuthorization; bearer/api-key requieren secret/ref. Rotation almacena nueva ref, incrementa integration revision y deshabilita hasta reactivación. Revocation terminal revoca secret store.
- Operation config immutable por version, schema object `additionalProperties=false`, path relativo con placeholders declarados, GET-only. Enabled operationId único por AgentInstance (partial unique index).
- Snapshot CRM agrega sólo read/enabled operations de integration ACTIVE `http-v1`; metadata/schema no incluye baseUrl/secret; hash efectivo incluye operaciones/version/revision/schema. Duplicated enabled operation IDs fail snapshot.
- AI Service Runtime crea capability por operationId del snapshot; herramienta sólo se expone si policy snapshot autoriza exactamente operationId. Resolver binding AI Service→CRM scope `agent.integration.binding.resolve` deriva integration/connector/version/revision/operation desde DB, no del ToolCall args. Capability pasa por Registry executor/CurrentAuthorization gate.
- Effect boundary CRM scope `agent.integration.execute`: revalidación tenant/binding/status/operation/revision y CurrentAuthorization live antes de resolver credential y realizar request. Secret no viaja a AI Service/LLM.
- Executor HTTPS: origin port 443, rejects userinfo/path/query/hash, literal IP/local host/private/reserved DNS, validates all DNS records and pins public address, no redirects, GET only, parameter/schema/path/query validation, timeout 10s, response cap 64 KiB, credential echo redaction. No arbitrary host/URL/headers.
- IntegrationInvocation audit per attempt; no request/response/secrets persistidos. Write/Sensitive denegados hasta contrato human confirmation/idempotency.

### Archivos principales CRM

- `src/services/AgentIntegrationConnectorCatalog.service.ts`
- `src/services/AgentIntegrationHttpExecutor.service.ts`
- `src/services/IntegrationCredentialSecret.service.ts`
- `src/models/agent/integrations/IntegrationCredentialSecret.model.ts`
- `src/services/agent-integration-config-validator.ts`
- `src/services/AgentInstanceIntegration.service.ts`
- `src/services/agent-runtime-snapshot.service.ts`
- `src/infrastructure/ai-service/agent-runtime-snapshot.client.ts`
- `src/routes/agent-integrations.route.ts`
- `src/routes/agent-integration-runtime.route.ts`
- `src/models/agent/integrations/AgentInstanceOperation.model.ts` (unique enabled operation index)
- `.env.example`, `.env.production.example`

### Archivos principales AI Service

- `src/application/services/agent-runtime-integration-capability.ts`
- `src/application/services/agent-runtime-capability-executor.ts` (executeResolved capability + gate)
- `src/application/services/agent-runtime-capabilities.ts`
- `src/application/services/agent-runtime-context-composer.ts`
- `src/application/services/agent-instance-execution-runtime.ts` (dynamic capability tools)
- `src/application/contracts/agent-runtime-snapshot.contract.ts`
- `src/presentation/http/routes/agent-runtime.routes.ts`
- `src/infrastructure/composition-root.ts` (binding resolver wired into CurrentAuthorization gate)
- `.env.example`

### Tests

- CRM focused suites: 6 suites, 37 tests passed (config validator, service config safety, CurrentAuthorization, encrypted credential custody, Connector Catalog, HTTP executor SSRF/params/pinning/redaction).
- AI Service focused suites: 7 suites, 29 tests passed (runtime, snapshot identity/hash, context composer, dynamic capability, gate, ExecutionEngine).
- AI Service `tsc --noEmit`: pass.
- CRM `tsc --noEmit`: still exits with one preexisting unrelated error `src/modules/media/urlFirmadaEvolution.ts:23` missing module `@aws-sdk/s3-request-presigner`; no errors reported in WAB-100 files.
- CRM ESLint changed source files: pass (0 errors/warnings).
- AI Service ESLint changed source files: 0 errors; 19 preexisting warnings in composition-root prompt legacy imports/variables.
- Tests are focused suites; no full monorepo test run, no E2E against real provider.

### Config/operational blockers before deployment or end-to-end live use

1. Provision production `CRM_AGENT_INTEGRATION_CREDENTIAL_ENCRYPTION_KEY` (base64 32 bytes), secure backup and key rotation policy; local dev key generated in ignored CRM `.env` only and not shown.
2. Configure authoritative Policy Resolver/A2 capabilities. FailClosed policy is unchanged; registered operation is NOT auto-granted.
3. WABY_frontend management UI not implemented; REST backend API exists.
4. WAB-100 executor is read-only GET; write/sensitive operations denied pending human-confirmation/idempotency/retry contracts.
5. Connector catalog currently contains only `http-v1@1.0.0`; no OpenAPI importer, OAuth, private network connectors or real-provider E2E.
6. Production deployment/env wiring and external network policy not validated; no deployment performed.

### Local state

All code/docs changes remain local and uncommitted. CRM and AI Service existing working changes preserved. `infrastructure/` unchanged. `waby-docs` preexisting local changes/untracked artifacts preserved; only the two docs append-only updated. No commit, push or deploy.


### 57.1 - Audit preflight refinado

CRM effect endpoint ahora intenta persistir `IntegrationInvocationStatus.REQUESTED` con correlationId antes de CurrentAuthorization live/credential resolution/effect. Si preflight audit falla, el flujo se corta antes del efecto. Después se agrega evento final (SUCCEEDED/FAILED/DENIED) append-only con mismo correlationId. Post-audit failure se loguea (el efecto remoto puede haber ocurrido); requiere alerting operacional. Payloads/secrets no se guardan.


### 57.2 - Cancellation

AI Service HTTP executor honors tool-loop `AbortSignal`; CRM internal route binds inbound request abort/response close to an AbortController, and outbound HTTPS request uses the same signal. The outbound request also has a 10-second timeout.


## 58 — Cierre de validación del P0 Runtime + CurrentAuthorization (local, 20/11/2026)

### Restricción de entorno

Trabajo exclusivamente en `F:\\ElemenSoft` local. No se accedió a production ni staging; no hubo deploy, migración, commit o push. Se comprobó Docker local (`docker ps`) y no había contenedores activos, por lo que no se usó ningún servicio containerizado. Los working trees preexistentes permanecen preservados.

### P0 Runtime — implementación y evidencia

- `AgentInstanceExecutionRuntime` usa el `ExecutionEngine` compartido para las etapas retrieval/context/tool-loop; no entra al legacy Bot ExecutionContext.
- Identidad durable viene de `AgentExecutionDocument`; snapshot valida organization/workspace/instance/definition y hash efectivo; input externo no puede inyectar history.
- CRM canonical history provider limita a últimos 20 mensajes user/assistant de la conversación y los compone con el mensaje actual.
- Tool definitions salen del registry runtime + operations del snapshot, filtradas por policy `allowedCapabilities`; Rules/Skills/Knowledge se componen como contexto subordinado/no confiable y no conceden permisos.
- Tool Loop del Engine ejecuta invocaciones sucesivas, pasa llamadas/resultados causalmente, limita rounds a 5, valida cantidad de resultados y chequea AbortSignal entre provider/tool calls.
- Capability executor valida allowlist/contexto/schema, aplica timeout y CurrentAuthorization gate inmediatamente antes de executor. Integration capability obtiene binding trusted desde CRM; binding no se deriva de ToolCall params.
- Cancellation se propaga desde worker lease signal al retrieval/provider/capability HTTP; invalidación durable/assignment live y cancelación de ejecución física después de iniciar un efecto externo no se afirman como garantías.

### Prueba adicional

Se añadió `crm/microservices/ai-service/src/engine/__tests__/execution-engine.agent-runtime-tool-loop.test.ts` con 4 casos: round Tool Call causal + final invocation, límite de iteraciones, pre-cancel, identidad/iteration/result-count inválidos. Resultado: **1 suite / 4 tests PASS**.

### Validación local acumulada de este checkpoint

- AI Service TypeScript `tsc --noEmit`: **PASS** antes de añadir la nueva suite; rerun final pendiente.
- AI Service focused tests: **7 suites / 29 tests PASS**, más Tool Loop **1 suite / 4 tests PASS**.
- CRM focused tests: **7 suites / 42 tests PASS** (policy resolver, CurrentAuthorization, integrations, catalog, HTTP executor, secret custody, config validator).
- AI Service ESLint scoped: **0 errors, 19 warnings preexistentes** en `composition-root.ts` (unused imports/vars).
- CRM ESLint scoped: **PASS**.
- CRM `tsc --noEmit`: falla únicamente en `src/modules/media/urlFirmadaEvolution.ts:23`, módulo preexistente ausente `@aws-sdk/s3-request-presigner`; no se atribuye a este P0.
- `git diff --check`: PASS (solo warnings de conversión LF→CRLF).
- No full monorepo tests ni E2E; E2E no solicitado y no requerido para este checkpoint.

### CurrentAuthorization / WAB-99

Adapter AI Service→CRM y gate están conectados al registry executor. El binding trusted se resuelve por endpoint interno CRM desde integración/operación persistidas; CRM CurrentAuthorization revalida ownership, estado, revisión y operación enabled; el effect boundary de WAB-100 revalida antes de secret resolution/external effect. El gate deniega ante deny, timeout, unavailable, malformed response, missing binding, identity/binding mismatch o configuración ausente. Tests focalizados cubren allow/deny, mismatch, timeout, missing config/binding y no ejecución tras deny. WAB-99 puede cerrarse por su alcance A2.3; no implica que las capabilities se encuentren habilitadas por policy.

### Bloqueo objetivo para cerrar todo el P0 de authorities

`IntersectingAgentRuntimePolicyResolver` implementa correctamente la intersección `PlatformAllowed ∩ DefinitionAllowed ∩ InstanceEnabled ∩ CurrentAuthorization` y mínimos de límites. Pero CRM `AgentRuntimeSnapshotService` conserva `FailClosedAgentRuntimePolicyResolver` como default: no hay wiring de un `IAgentRuntimePolicyAuthoritySource` concreto. La inspección local no encontró fuente física autoritativa de Platform capability grants/mandatory restrictions, AgentDefinition allowed capabilities ni Instance enabled capability set general. Las operations de integración enabled y CurrentAuthorization operacional sí tienen fuente CRM; no equivalen a las tres policy authorities globales/definition/instance. No inventar grants, reutilizar permisos `wrk.agents.*`, usar legacy Bot capabilities ni habilitar por default. Consecuencia: el runtime/tool loop está ejecutable, pero el snapshot policy fail-closed no ofrece capabilities externas hasta que las tres fuentes faltantes se implementen y conecten. Por seguridad, este bloqueo impide declarar cerrado el P0 completo de authorization.

### Estado

- Runtime nucleus / Engine / Tool Loop / identity / context / capability executor: implementación local completada y validación focalizada PASS.
- A2.3 CurrentAuthorization adapter/gate + trusted binding: completado localmente; Linear WAB-99 pasa a Done.
- P0 de authorities: **incompleto de manera explícita y fail-closed**; WAB-81 continúa In Progress. No cerrar WAB-81 ni WAB-100 hasta fuente autoritativa de policy, wiring y validación final.
- No production/staging, Docker no disponible activo, commit/push/deploy.


### 58.1 — Revalidación final

- AI Service `tsc --noEmit` rerun after adding Tool Loop test: **PASS**.
- AI Service combined focused tests: **8 suites / 33 tests PASS**.
- New test file Jest PASS. A scoped ESLint attempt on that test file is not supported by current type-aware ESLint project config because tests are excluded from tsconfig (`was not found by the project service`); source-file scoped ESLint remains PASS with 0 errors and 19 existing warnings. No lint config was changed to accommodate this test-only file.
- CRM focused tests and scoped ESLint results remain **7 suites / 42 PASS** and **PASS**, respectively.


### 58.2 — Tracking del bloqueo

Se creó en Linear `WAB-101 — P0.1 Fuentes autoritativas y wiring de Policy Resolver para capabilities`, hijo de WAB-81. Delimita implementación de Platform catalog/restrictions, DefinitionAllowed persistido en AgentDefinition, InstanceEnabled source, concrete `IAgentRuntimePolicyAuthoritySource`, snapshot wiring, live CurrentAuthorization gate y tests fail-closed. No se cambia la arquitectura ni se inventan grants. WAB-81 permanece In Progress hasta aceptar WAB-101.


## 59. Consolidación del incremento P0 Runtime + CurrentAuthorization + Integration Executor (20/11/2026)

### Alcance completado localmente

- AgentInstance Runtime conectado al ExecutionEngine existente, con identidad durable/tenant validada y snapshot/hash checks.
- Context Composer con orden de autoridad, Rules/Skills y Knowledge como datos no confiables, historial limitado a 20 mensajes user/assistant y grounding fail-closed.
- Knowledge retriever conectado: scope Workspace/KB, filtro obligatorio, documentos COMPLETED únicamente y error ante retrieval no disponible/vacío.
- Tool Loop multi-round con correlación causal de tool calls/results, límite de iteraciones, validación de resultados y AbortSignal.
- Capability Registry/Executor AgentInstance separado de legacy Bot, schema/timeout/cancel y gate CurrentAuthorization.
- CRM Integration Executor vertical local: catalog allowlisted http-v1@1.0.0, GET-only, credential custody AES-256-GCM, trusted binding, snapshot sin secretos, live authorization en effect boundary, SSRF controls, timeout/response cap, audit redacted y cancellation propagation.
- AgentInstance Knowledge references validadas al crear/actualizar; delete de KB retira referencias de instancias después del cascade confirmado.

### Validación local reportada

- AI Service focused tests: 8 suites / 33 tests PASS.
- CRM focused tests: 7 suites / 42 tests PASS.
- AI Service TypeScript noEmit: PASS.
- CRM scoped ESLint: PASS.
- AI Service scoped ESLint: 0 errors, 19 warnings preexistentes de composition-root.
- git diff check: PASS (warnings LF/CRLF).
- CRM full type-check bloqueado por dependencia preexistente faltante @aws-sdk/s3-request-presigner en src/modules/media/urlFirmadaEvolution.ts:23; no atribuible al incremento.
- No full monorepo suite ni E2E; expresamente fuera de alcance acordado.

### Estado y blockers

El runtime base, Knowledge y vertical HTTP read-only están implementados localmente, pero WAB-81 no se cierra. El Policy Resolver real permanece fail-closed porque no se verificaron fuentes autoritativas físicas completas para PlatformAllowed, DefinitionAllowed y capability-level InstanceEnabled/general CurrentAuthorization. La integración HTTP enabled no equivale a las cuatro authorities generales.

WAB-101 queda como siguiente bloque P0 para resolver fuentes, ownership y wiring real sin grants/defaults inventados. WAB-100 permanece In Progress por authority wiring, provisioning/rotación de key en entornos, frontend y validación de integración real; no se declara production-ready.

### Restricciones de trabajo

Todo realizado localmente en F:\\ElemenSoft; no production/staging, deploy, migration live, commit ni push. Se preservó el trabajo local existente. No se ejecutó E2E.

### Linear sincronizado

- WAB-81: progreso y blockers P0 documentados; permanece In Progress.
- WAB-96: A2.1/A2.2/A2.3 completados, A2.4 WAB-100 en progreso.
- WAB-99: Done con evidencia de adapter/gate.
- WAB-100: implementación y validaciones locales, blockers restantes; In Progress.
- WAB-101: creado como P0.1, Backlog, hijo de WAB-81.

