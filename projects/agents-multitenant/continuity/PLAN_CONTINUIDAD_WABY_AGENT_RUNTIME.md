# WABY â€” Plan maestro de implementaciÃ³n y continuidad

**Fecha del checkpoint:** 20/11/2026  
**Workspace de desarrollo:** repositorios WABY clonados localmente; las rutas locales no son parte del contrato de continuidad.  
**PropÃ³sito:** permitir que otra sesiÃ³n/PC/agent (Claude) continÃºe el trabajo sin perder contexto, repetir auditorÃ­as, reabrir decisiones aprobadas ni mezclar repositorios.  
**Estado:** documento operativo local. No implica commit, push, deploy ni cierre de Linear.

---

## 0. Instrucciones de lectura y autoridad de fuentes

Este documento es una guÃ­a de continuidad y plan consolidado, no reemplaza las fuentes originales.

### Orden de autoridad

1. **CÃ³digo y estado fÃ­sico del repositorio:** determina quÃ© estÃ¡ implementado realmente. Inspeccionar el working tree antes de editar.
2. **Linear:** estado, alcance autorizado, dependencias, aceptaciÃ³n, responsables y bloqueos.
3. **`waby-docs`:** contratos canÃ³nicos, arquitectura, decisiones reconciliadas y evidencia tÃ©cnica.
4. **Este documento:** mapa de continuidad y plan de ejecuciÃ³n. Si contradice una fuente anterior, detenerse y reconciliar; no asumir que este resumen prevalece.

### Repositorios y responsabilidades

| Ruta | Responsabilidad | Uso en este plan |
|---|---|---|
| `crm/` | Backend CRM principal, dominio, GraphQL, workers, integraciÃ³n AI Service | ImplementaciÃ³n backend y autoridad de dominio/tenant |
| `WABY_frontend/` | Frontend plataforma | Solo cuando exista tarea/contrato explÃ­cito |
| `infrastructure/` | Infraestructura compartida/DevOps | Solo si una dependencia aprobada requiere cambios |
| `waby-docs/` | Repositorio privado documental canÃ³nico | Contratos, arquitectura, auditorÃ­as, decisiones, planes y evidencia |
| `ElemeSoft` (organizaciÃ³n GitHub) | Workspace/organizaciÃ³n que agrupa proyectos independientes | No es un repositorio Ãºnico ni tiene una responsabilidad tÃ©cnica propia |
| `wabyland` | Marketing/web | Fuera del alcance de este plan |

**Regla:** no modificar frontend, infrastructure ni otro repositorio por conveniencia. La dependencia debe estar identificada y autorizada.

### Herramientas y roles

- **Linear:** seguimiento, tareas, dependencias, estados y bloqueos. No copiar todo el diseÃ±o en descripciones de issues.
- **GitHub / repositorios:** fuente del cÃ³digo y documentaciÃ³n versionada.
- **OpenCode / Claude:** inspecciÃ³n, implementaciÃ³n, pruebas y revisiÃ³n tÃ©cnica en el repositorio autorizado.
- **ChatGPT:** anÃ¡lisis crÃ­tico, reconciliaciÃ³n de decisiones, generaciÃ³n de prompts y revisiÃ³n de entregables.
- **waby-docs:** documento contractual canÃ³nico. No crear documentos paralelos sin motivo.
- Trabajo **local-first**. No commit, push, PR, deploy, migraciones live ni cambios en staging/production salvo autorizaciÃ³n explÃ­cita.

---

## 1. Objetivo global

Completar la transiciÃ³n al sistema multitenant de agentes administrados por Waby:

- `AgentDefinition`: definiciÃ³n central administrada por Waby, identidad estable.
- `AgentInstance`: instancia perteneciente a un Workspace y unidad runtime ejecutable.
- CRM conserva la autoridad sobre OrganizaciÃ³n, Workspace, AgentInstance, ownership, estado, configuraciÃ³n autorizada, revocaciÃ³n y contexto empresarial.
- AI Service ejecuta el runtime compartido; no se crea un microservicio executor nuevo.
- Las capabilities externas solo pueden ejecutarse mediante catÃ¡logo allowlisted y autorizaciÃ³n vigente, no por prompt ni por ToolCall.
- Knowledge es propiedad/Ã¡mbito del Workspace y se configura como referencia de AgentInstance.
- Skills y Rules son configuraciÃ³n contextual subordinada; no conceden permisos.
- La ejecuciÃ³n debe ser durable, idempotente, tenant-safe, cancelable cuando el contrato lo permita, auditable y fail-closed.

---

## 2. Decisiones arquitectÃ³nicas que no deben reabrirse sin evidencia

### 2.1 Identidad y ownership

- `AgentDefinition` y `AgentInstance` reemplazan el vocabulario ambiguo Bot/BotInstance.
- AgentInstance es el runtime executable unit; referencia AgentDefinition estable.
- No se fija una versiÃ³n publicada permanentemente en la instancia si el contrato aprobado establece adopciÃ³n automÃ¡tica de la configuraciÃ³n vigente.
- AgentInstance pertenece a Workspace; CRM valida el ownership Organizationâ€“Workspace.
- ContactAssignment y su identidad de transiciÃ³n `evAssignedId` no se sustituyen por snapshot/runtime identity.

### 2.2 Autoridad y seguridad

La elegibilidad efectiva de una capability es:

`PlatformAllowed âˆ© DefinitionAllowed âˆ© InstanceEnabled âˆ© CurrentAuthorization`

- PlatformAllowed: catÃ¡logo y restricciones obligatorias administradas por Waby.
- DefinitionAllowed: capabilities compatibles/permitidas por AgentDefinition.
- InstanceEnabled: configuraciÃ³n de AgentInstance/integraciÃ³n/operaciÃ³n.
- CurrentAuthorization: autorizaciÃ³n contextual vigente; debe verificarse live antes de cada operaciÃ³n/efecto.

No son grants:
- Prompt, Rules, Skills, Knowledge o ToolCall args.
- `wrk.agents.*` (permisos de administraciÃ³n).
- JWT de servicio (autentica transporte; no concede capabilities).
- CrÃ©ditos/budget authorization (autoriza gasto; no autoriza operaciones externas).
- Snapshot antiguo como sustituto de autorizaciÃ³n vigente.

**Fail-closed:** fuente ausente, timeout, error, mismatch de tenant/binding, revocaciÃ³n o decisiÃ³n indeterminada deniega. No inventar grants, defaults, catÃ¡logos ni mappings. No reutilizar capabilities legacy de Bot.

### 2.3 Runtime/contexto

- PolÃ­tica de seguridad obligatoria > definiciÃ³n Waby > Rules/Skills de instancia > Knowledge recuperado (datos no confiables) > Ãºltimos 20 mensajes user/assistant.
- No incluir mensajes internos de workflow ni memoria/resumen automÃ¡tico fuera del contrato.
- Knowledge/RAG debe limitarse a fuentes del Workspace/KB autorizadas, validar ownership/status, y fallar cerrado.
- No fallback silencioso a conocimiento general/internet.
- Secretos nunca en prompt/context/logs.
- Provider/model/route y parÃ¡metros tÃ©cnicos sujetos a polÃ­tica central Waby.

### 2.4 Durabilidad

- At-least-once transport + consumidores idempotentes; no prometer exactly-once entre Mongo, RabbitMQ, LLM y efectos externos.
- originMessageId, executionId, eventId y evAssignedId tienen identidades distintas y estables.
- Outbox/Inbox durable, ack despuÃ©s de persistencia durable, redelivery deduplicada, DLQ y recovery.
- CancelaciÃ³n/revocaciÃ³n no garantiza revertir un efecto externo ya iniciado.

### 2.5 Credits/capacity

- Pool de AI Credits perteneciente a Organization, compartido por Workspaces, con attribution por Workspace.
- Authorization/commit/ledger append-only, idempotencia y fencing conforme a P4.
- No sustituir CurrentAuthorization con budget authorization.
- Retirement/grace semantics y capacidad deben respetar contratos WAB-78/WAB-79/WAB-81.

---

## 3. Estado fÃ­sico consolidado al checkpoint

**Importante:** todo el trabajo descrito como local se considera no integrado/no desplegado. Antes de cualquier cambio, comprobar el working tree real; este documento no autoriza a sobrescribir cambios.

### 3.1 CRM / AI Service â€” Runtime y contexto

Incrementos locales previamente implementados:
- AgentInstance Runtime conectado al ExecutionEngine existente.
- Identidad durable de ejecuciÃ³n y validaciÃ³n tenant/snapshot/hash.
- Canonical CRM conversation history provider: Ãºltimos 20 mensajes user/assistant.
- Context Composer con prioridad de seguridad y contexto empresarial subordinado/no confiable.
- Knowledge retriever conectado a runtime, validaciÃ³n Workspace/KB, vector-space y scope.
- Grounding fail-closed, documentos COMPLETED Ãºnicamente, sin fallback general/internet.
- Engine Tool Loop multi-round, provider tool calls, correlaciÃ³n causal, lÃ­mite de rounds, validaciÃ³n de resultados y AbortSignal.
- Capability Registry/Executor con schema validation, timeout y CurrentAuthorization gate.
- Integraciones expuestas dinÃ¡micamente segÃºn snapshot/policy.
- Binding confiable resuelto por CRM, no por argumentos de ToolCall.
- Boundary CRM revalida tenant, binding, integraciÃ³n/operaciÃ³n, estado/revisiÃ³n y autorizaciÃ³n antes de resolver credential/efecto.
- Credential custody CRM AES-256-GCM, referencias opacas tenant/instance-bound; no secrets en snapshot.
- HTTP Integration Executor restringido: HTTPS pÃºblico, allowlist/config registrada, GET read-only, auth limitada, no redirects, DNS/SSRF protections, timeout y response cap.
- Operaciones WRITE/SENSITIVE no habilitadas sin confirmaciÃ³n humana, idempotency y retry contract.
- Knowledge Base references validadas al crear/actualizar AgentInstance y limpiadas durante delete/cascade conforme al incremento local.

**ValidaciÃ³n registrada antes de los Ãºltimos cambios de dependencia:**
- AI Service focused tests: suites focalizadas PASS (la cifra exacta debe reconciliarse con la Ãºltima ejecuciÃ³n).
- AI Service type-check: PASS.
- CRM focused tests: PASS.
- CRM scoped ESLint: PASS.
- AI Service scoped ESLint: 0 errors, warnings preexistentes.
- `git diff --check`: PASS, con warnings de line endings.
- Full CRM type-check habÃ­a estado bloqueado por `@aws-sdk/s3-request-presigner`; el usuario informÃ³ que ya lo solucionÃ³. **No reinstalar ni alterar dependencias.** Ejecutar la verificaciÃ³n actual para confirmar.
- No E2E real con Mongo/Rabbit/CRM/Qdrant/provider. No commit/push/deploy.

### 3.2 Integrations / CurrentAuthorization

Trabajo local asociado a WAB-96/A2:
- WAB-98 (A2.2 CRM authority CurrentAuthorization + endpoint): completado localmente segÃºn checkpoint.
- WAB-99 (A2.3 adapter/gate AI Service): completado localmente segÃºn checkpoint.
- WAB-100 (A2.4 Integration Executor): implementaciÃ³n local en progreso; executor read-only HTTP registrado, custody de secrets, audit e invocation boundary.
- WAB-97 (A2.1 persistencia de integraciones/operaciones/binding): verificar estado actual en Linear y repo antes de declarar completo.
- WAB-96 parent: no asumir cierre.

### 3.3 Bloqueo estructural de capabilities generales

Fuentes fÃ­sicas no confirmadas/encontradas en el checkpoint:
- Platform capability catalog/mandatory policy.
- DefinitionAllowed en AgentDefinition.
- InstanceEnabled general para capabilities (no confundir con IntegrationOperation.enabled).
- CurrentAuthorization general para capabilities/runtime.
- Fuente AI-specific entitlement/limits.

Existe resolver/intersecciÃ³n de policy, pero snapshot wiring usa fail-closed resolver por defecto; no se identificÃ³ una autoridad fÃ­sica concreta completamente wired. No se inventan grants. Este es un blocker real para habilitar tools/capabilities generales.

### 3.4 WAB-79 â€” RetirementOperation / lifecycle durable

WAB-79 estÃ¡ In Progress. Se han realizado mÃºltiples revisiones/reconciliaciones del contrato fÃ­sico RetirementOperation, ownership/persistencia, Outbox y viabilidad.

Antes de continuar:
1. Inspeccionar working tree y archivos locales relacionados.
2. Leer la Ãºltima versiÃ³n del contrato RetirementOperation; confirmar versionado (no asumir que un reporte antiguo v3.8 es el vigente).
3. Revisar las tareas hijas de WAB-79 y sus estados en Linear.
4. Reconciliar implementaciÃ³n fÃ­sica con contrato aceptado.
5. Evitar reconstruir/repetir auditorÃ­as ya realizadas.

El objetivo no es solamente scaffolding: RetirementOperation debe conectarse al runtime real, drain de ejecuciones, leases/fencing, estado terminal y recuperaciÃ³n segÃºn el contrato.

### 3.5 DocumentaciÃ³n ya actualizada

Se aÃ±adiÃ³ checkpoint al documento:
`waby-docs/projects/agents-multitenant/audits/wab-81/implementacion-runtime-local-2026-11-20.md`

El checkpoint consolidÃ³ Runtime, Knowledge, CurrentAuthorization, Integration Executor, pruebas, bloqueos y restricciones local-first. Revisar la secciÃ³n mÃ¡s reciente antes de volver a documentar.

---

## 4. Fuente canÃ³nica documental â€” mapa

Repositorio:
`projects/agents-multitenant`

### DiseÃ±o/contratos

| Documento | Rol | Consultar para |
|---|---|---|
| `design/fase-2-diseno-objetivo-y-contratos.md` | DiseÃ±o integral canÃ³nico de WAB-78/Fase 2 | Domain, contracts, ARQ-19/20/21, DEC, P4, runtime, durability, lifecycle |
| `design/configuracion-efectiva-y-capacidades.md` | ConfiguraciÃ³n efectiva/capabilities | Precedencia, grants, capability contract |
| `design/decisiones-operativas-agent-instance.md` | DEC-CM-01..08 | Decisiones operativas aprobadas |
| `design/reconciliacion-vigencia-fase-2-wab-78.md` | ReconciliaciÃ³n documental | Precedencia y contradicciones corregidas |
| `design/wab-79-dominio-ownership-persistencia.md` | Ownership/persistencia | Domain physical model |
| `design/wab-79-retirementoperation-physical-contract.md` | Contrato fÃ­sico RetirementOperation | Lifecycle, drain, durable operation, recovery |
| `design/wab-79-informe-reconciliacion-outbox-seccion-16-14.md` | ReconciliaciÃ³n Outbox | Outbox semantics y revisiÃ³n de contrato |
| `design/wab-79-viabilidad-tecnica-cierre-revision-*.md` | Historial tÃ©cnico WAB-79 | Usar solo la revisiÃ³n mÃ¡s reciente y evidencia relevante; no leer todas si no es necesario |
| `design/wab-81-agent-execution-envelope-and-snapshot-proposal.md` | Envelope/snapshot proposal | Execution identity, snapshot contract, transport |
| `design/integraciones-api-externas.md` | Integraciones externas | Connector/operation scope, API integration |
| `design/plan-administracion-catalogo-waby.md` | CatÃ¡logo administrado por Waby | Platform catalog and administration |
| `design/catalogo-agentes-y-skills-estandar.md` | Agent catalog / Skills | Definitions and standard skills |

### AuditorÃ­as y continuidad

| Documento | Rol |
|---|---|
| `audits/wab-78/README.md` | Ãndice y material de revisiÃ³n D1â€“D5 |
| `audits/wab-78/matriz-acumulativa.md` | Matriz acumulativa de hallazgos |
| `audits/wab-78/informe-d4-a-runtime.md` | Runtime findings/analysis |
| `audits/wab-78/informe-d4-b-contexto.md` | Context findings/analysis |
| `audits/wab-78/informe-bloque-5b.md` | Bloque 5B |
| `audits/wab-77/README.md` | Fase 1 consolidada; histÃ³rica/aceptada |
| `audits/wab-81/implementacion-runtime-local-2026-11-20.md` | Checkpoint fÃ­sico mÃ¡s reciente de WAB-81/runtime local |

### Regla documental

- Canonical design: actualizar solo cuando un contrato o implementaciÃ³n haya sido reconciliado.
- Audit reports: evidencia histÃ³rica, no reabrir sin contradicciÃ³n/evidencia nueva.
- Implementation checkpoint: registra hechos del repo, tests, limitaciones y estado local.
- Linear: seguimiento y tareas; no convertirlo en duplicado de diseÃ±o completo.

---

## 5. Linear â€” mapa de issues y seguimiento

Linear workspace/team: Waby. Project: Agentes de IA.

### Issues de fases principales

| Issue | Rol | Estado conocido / acciÃ³n |
|---|---|---|
| WAB-75 | Plan de migraciÃ³n arquitectÃ³nica / padre | Consultar estado actual y dependencias |
| WAB-77 | Fase 1 audit | Done; no reabrir sin evidencia |
| WAB-78 | Fase 2 diseÃ±o/contratos | In Progress segÃºn documentaciÃ³n histÃ³rica; consultar estado actual. Canonical contract |
| WAB-79 | Fase 3 dominio/ownership/persistencia | In Progress; lifecycle, durable operations y persistencia |
| WAB-80 | Fase 4 provisioning/autorizaciÃ³n | Consultar estado, criterios y dependencias |
| WAB-81 | Fase 5 runtime/context composition | In Progress; actual foco de runtime |
| WAB-94 | A1 AgentInstance persistence base | Consultar estado; no ampliar su scope |

### Integraciones/CurrentAuthorization â€” WAB-96

| Issue | Incremento | Estado al checkpoint |
|---|---|---|
| WAB-96 | A2 Integrations + CurrentAuthorization parent | Backlog en Ãºltimo fetch; reconciliar Linear |
| WAB-97 | A2.1 CRM persistence integrations/operations/binding | Estado por verificar |
| WAB-98 | A2.2 CRM CurrentAuthorization authority/endpoint | Completado localmente; confirmar Linear |
| WAB-99 | A2.3 AI Service adapter/gate | Completado localmente; confirmar Linear |
| WAB-100 | A2.4 Integration Executor/owner boundary | In Progress; trabajo local |
| WAB-101 | Authority discovery/wiring follow-up | Backlog; scope a reconciliar |

Links:
- WAB-78: https://linear.app/waby/issue/WAB-78/fase-2-diseno-objetivo-y-contratos
- WAB-79: https://linear.app/waby/issue/WAB-79/fase-3-dominio-ownership-y-persistencia
- WAB-80: https://linear.app/waby/issue/WAB-80/fase-4-provisioning-y-autorizacion
- WAB-81: https://linear.app/waby/issue/WAB-81/fase-5-runtime-y-composicion-de-contexto
- WAB-96: https://linear.app/waby/issue/WAB-96/a2-integraciones-de-agentinstance-y-currentauthorization-runtime
- WAB-97: https://linear.app/waby/issue/WAB-97
- WAB-98: https://linear.app/waby/issue/WAB-98
- WAB-99: https://linear.app/waby/issue/WAB-99
- WAB-100: https://linear.app/waby/issue/WAB-100
- WAB-101: https://linear.app/waby/issue/WAB-101

**Antes de trabajar:** recuperar descripciÃ³n, comments, subtasks y estado actual del issue concreto. No asumir que los estados escritos en este checkpoint siguen vigentes.

---

## 6. Plan de implementaciÃ³n â€” secuencia recomendada

Esta secuencia es un plan tÃ©cnico de continuidad, no una autorizaciÃ³n automÃ¡tica para ejecutar todos los bloques. Cada bloque debe respetar su issue, alcance y dependencias.

### Bloque 0 â€” Recuperar estado y validar workspace (obligatorio al iniciar nueva sesiÃ³n)

1. Confirmar la ruta real y repositorios presentes.
2. Inspeccionar `git status` de crm, waby-docs y repositorio que se vaya a tocar.
3. No limpiar, resetear, stash, checkout ni sobrescribir cambios locales.
4. Leer este documento y el checkpoint WAB-81.
5. Recuperar issues relevantes de Linear y comentarios recientes.
6. Identificar el issue que autoriza el bloque actual.
7. Confirmar el branch y archivos dirty antes de editar.
8. No repetir auditorÃ­as histÃ³ricas.

**Entrega:** resumen de estado real + plan acotado del siguiente incremento.

### Bloque 1 â€” Cerrar/validar el incremento local Runtime + Integration Executor

Issues principales: WAB-81, WAB-96, WAB-97/98/99/100, WAB-101.

Orden:
1. Confirmar soluciÃ³n del usuario a `@aws-sdk/s3-request-presigner`; no reinstalar paquetes.
2. Ejecutar CRM full type-check actual.
3. Ejecutar tests focalizados CRM/AI Service relevantes.
4. Confirmar lint y diff review.
5. Inspeccionar WAB-97 physical state (integration, operation, binding).
6. Confirmar CurrentAuthorization CRM authority/endpoint WAB-98.
7. Confirmar AI Service adapter/gate WAB-99.
8. Confirmar executor WAB-100: registered allowlisted operation, no arbitrary HTTP, credential resolution after allow, boundary revalidation, audit, cancellation and read-only restrictions.
9. Confirmar tests para allow/deny, revocation, scope mismatch, timeout, binding mismatch, secrets, SSRF.
10. Reconciliar documentaciÃ³n checkpoint y comments Linear con evidencia actual.

**No cerrar WAB-81/WAB-96 por el mero hecho de que type-check pase.** Deben cumplirse los criterios de aceptaciÃ³n y blockers de authority/runtime.

### Bloque 2 â€” Completar Runtime de AgentInstance sobre ExecutionEngine

Issue: WAB-81. Es el siguiente bloque central de implementaciÃ³n.

#### 2.1 ReconstrucciÃ³n del flujo fÃ­sico

Inspeccionar el ExecutionEngine existente y documentar:
- Entrada y validaciÃ³n.
- Execution identity/durable record.
- Snapshot/AgentRuntimeSnapshot.
- Worker/lease/fencing.
- Runtime invocation.
- Provider routing y credential resolver.
- Tool Loop y Capability Registry.
- Knowledge retrieval/context composer.
- Result persistence.
- Result Outbox/CRM consumer.
- Cancellation y graceful shutdown.
- Estados terminales/recovery.

No crear otro engine ni executor paralelo.

#### 2.2 Reconciliar contratos

Comparar el flujo fÃ­sico con:
- WAB-78 canonical runtime contracts.
- WAB-81 issue acceptance criteria.
- Agent execution envelope/snapshot proposal.
- WAB-79 ownership, lifecycle y durable operation.
- CurrentAuthorization contract.
- Integration executor boundary.
- Outbox/Inbox ARQ-22.

Identificar divergencias concretas, no reauditar el diseÃ±o completo.

#### 2.3 IntegraciÃ³n de runtime

Cerrar gaps entre:
- CRM authoritative snapshot y AI Service runtime.
- Durable Execution identity y snapshot identity/hash.
- AgentInstance lifecycle status.
- CurrentAuthorization gate y executor effect boundary.
- Tool Loop multi-round con limits, validation y causality.
- Knowledge retrieval + grounding/abstention.
- Provider invoker + metrics/usage.
- Result durable persistence + Outbox.

#### 2.4 CancelaciÃ³n y lifecycle

Verificar propagaciÃ³n:
- Admission â†’ execution/worker lease.
- Retrieval/vector provider.
- LLM invocation.
- Capability calls.
- CRM boundary y outbound executor.
- Result persistence y terminal state.

Distinguir cancelaciÃ³n antes del efecto de efecto externo ya iniciado. No prometer rollback inexistente.

#### 2.5 Tests y validaciÃ³n

Unit/integration focalizados para:
- Tenant/snapshot identity mismatch.
- Config hash mismatch.
- Instance/Definition inactive.
- CurrentAuthorization denied/unavailable/revoked.
- Knowledge scope/KB/document state.
- Empty/error retrieval fail-closed.
- Tool loop rounds/correlation/schema.
- Integration binding/operation revision.
- Cancellation.
- Durable execution recovery.
- Result Outbox/idempotency.

E2E con dependencias reales solo si el scope/issue lo autoriza; no asumir staging/production.

### Bloque 3 â€” Resolver Authorities de capabilities generales (blocker de seguridad)

Relacionado: WAB-81/WAB-101 y reconciliar WAB-78/WAB-80/WAB-96.

Discovery e implementaciÃ³n mÃ­nima:
1. Platform catalog/mandatory policy (Waby owner).
2. DefinitionAllowed source (AgentDefinition).
3. InstanceEnabled source (AgentInstance/configuration).
4. CurrentAuthorization live source.
5. Capability identifier/version contract.
6. Concrete `IAgentRuntimePolicyAuthoritySource`.
7. Wire resolver/intersection to snapshot/runtime.
8. Live gate before every capability/effect.
9. Revocation semantics.
10. Tenant/resource scope validation.
11. Tests of intersection, missing source, unavailable source, revocation and mismatches.

No grants/defaults invented. No legacy Bot executor reuse. Si falta una decisiÃ³n de producto, registrar decision request/blocker en Linear y documentar en waby-docs antes de habilitar.

### Bloque 4 â€” RetirementOperation y drain funcional (WAB-79)

Objetivo: cerrar lifecycle fÃ­sico, no solo scaffolding.

1. Recuperar el contrato mÃ¡s reciente de RetirementOperation y subtasks.
2. Verificar modelo/Ã­ndices/ownership/state transitions.
3. Conectar retirement al runtime real AgentInstance/ExecutionEngine.
4. Admission fencing: impedir nuevas ejecuciones de una instancia retirada/retirement-pending segÃºn contrato.
5. Drain de ejecuciones activas: identificar, esperar, cancelar o reconciliar segÃºn reglas aprobadas.
6. Lease/fencing y concurrency safety.
7. Durable operation state, retry/recovery, terminal outcomes.
8. Outbox/event semantics si aplica.
9. Idempotencia de requests/retries.
10. Garantizar que delete/retire no invalide recursos antes de que las ejecuciones y efectos requeridos se hayan reconciliado.
11. Tests de race, duplicate request, process restart, lease expiry, stale worker, drain timeout y recovery.
12. Reconciliar WAB-79 docs/Linear con evidencia.

No crear un lifecycle paralelo en AI Service; CRM es la autoridad del recurso.

### Bloque 5 â€” Durable messaging / Outbox-Inbox / Recovery

Reconciliar ARQ-22 y WAB-81 con implementation.

CRM â†’ AI Service:
- Durable source message + intent/outbox.
- Persistent RabbitMQ publish.
- Positive confirm + routing validation.
- Stable originMessageId/eventId.
- Backpressure handling.
- Retry/backoff, DLQ.
- AI Service inbox/receipt + unique logical execution.
- Ack after durable receipt.
- Redelivery dedupe/conflicting payload quarantine.

AI Service â†’ CRM:
- Durable terminal execution + result Outbox atomically/equivalent recoverable guarantee.
- Stable executionId/eventId.
- Confirmed publication.
- CRM durable Inbox receipt/dedupe.
- Durable intent for subsequent delivery.
- Ack only after durable acceptance.
- External channel effect has separate idempotency/reconciliation.

Tests: publisher failure, uncertain confirm, duplicate physical messages, redelivery, process restart, DLQ replay, conflicting payload, stale execution, cancellation/failure event semantics.

### Bloque 6 â€” Provisioning y autorizaciÃ³n administrativa (WAB-80)

Confirmar physical state e issue scope before work.

- Provisioning AgentInstance from AgentDefinition.
- Workspace ownership/authorization.
- Admin permissions ARQ-20 `wrk.agents.*` and `wrk.agents.read`.
- Permission enum/roles, Manager/Owner semantics.
- Authorized lifecycle/configuration.
- Snapshot correctness.
- Integration connection/binding lifecycle.
- AgentDefinition compatibility and platform restrictions.
- No admin permission is a runtime capability grant.
- Tests GraphQL/REST/service authorization and tenant isolation.

No frontend unless separate approved task.

### Bloque 7 â€” P4 credits/capacity and execution economics

Use canonical WAB-78 P4 contract and existing tasks. Confirm actual implementation before starting.

- Organization-owned shared AI Credit pool.
- Workspace attribution.
- ExecutionBudgetAuthorization per execution.
- Fencing generation/committed budget.
- Append-only ledger and explicit refund records.
- Atomic/idempotent reserve/commit/reconcile.
- Provider usage/uncertain invocation reconciliation.
- Avoid duplicate charges across retries.
- Capacity/retirement interactions and grace periods.
- On-Demand Waby-owned.
- Limits source distinct from CurrentAuthorization.
- 90% suspend idea remains unapproved unless Linear/docs show decision.

Tests for concurrent reservations, retries, stale fencing, duplicate usage, refund, uncertain LLM result and organization/workspace accounting.

### Bloque 8 â€” AgentInstance configuration resources

Implement only through specific approved issues and canonical contracts:
- Knowledge references, lifecycle and deletion cascade.
- Skills/Rules instance scope.
- Additional prompt bounded/validated.
- Integration connections/credentials.
- Configuration effective precedence.
- ContactAssignment behavior on instance suspension/reassignment.
- Immediate memory: last 20 user+assistant messages only.
- Durable Knowledge retention/deletion.
- Skill media storage and lifecycle.
- No accidental cross-Workspace resource references.

### Bloque 9 â€” Cross-repo integration validation

When local contracts and focused tests are stable:
- CRM â†” AI Service service auth/directional JWT scopes.
- Snapshot request/response.
- RabbitMQ envelopes and durable messaging.
- Mongo repositories/indexes.
- Qdrant scope/metadata and real corpus.
- Provider adapter compatibility.
- Credential resolver.
- Integration Executor.
- Cancellation and shutdown.
- Observability/correlation/audit.

Run only local test environments explicitly approved. No production/staging access or deployment.

### Bloque 10 â€” Final reconciliation and release readiness

For each issue:
- Acceptance criteria evidenced.
- Tests and commands recorded.
- Diff reviewed.
- Documentation updated with implementation evidence, not design speculation.
- Linear comments/status synchronized.
- Blockers/deviations recorded.
- Local dirty state preserved.
- Commit/push/PR only on explicit authorization.

No issue is marked Done solely because code exists or a type-check passes.

---

## 7. PrÃ³xima acciÃ³n recomendada â€” concreta

**Retomar desde WAB-81: conectar y completar el AgentInstance Runtime real sobre el ExecutionEngine existente, despuÃ©s de validar el estado local actual y los blockers de CurrentAuthorization/Integration Executor.**

Secuencia inicial para la siguiente sesiÃ³n:

1. Leer este plan.
2. Leer el checkpoint WAB-81, especialmente las Ãºltimas secciones.
3. Consultar Linear WAB-81, WAB-96, WAB-100, WAB-101 y WAB-79; leer comments recientes.
4. Inspeccionar working tree de CRM y waby-docs.
5. Ejecutar/confirmar CRM type-check tras la reparaciÃ³n que informÃ³ el usuario; no tocar dependencias.
6. Inspeccionar el ExecutionEngine/runtime existente y mapear el flujo real.
7. Formular un incremento acotado y verificable antes de editar.
8. Implementar solo ese incremento.
9. Tests/type-check/lint segÃºn alcance.
10. Revisar diff, actualizar checkpoint y Linear.

No empezar con auditorÃ­a completa de WAB-78 ni repetir D1â€“D5. Consultar solo los contratos relevantes para la divergencia fÃ­sica detectada.

---

## 8. Prompt de continuidad para Claude en otra PC

Copiar este prompt y adjuntar/indicar este documento:

> Estamos continuando el proyecto WABY desde otra PC. Lee primero `waby-docs/projects/agents-multitenant/continuity/PLAN_CONTINUIDAD_WABY_AGENT_RUNTIME.md`.
>
> El documento es un mapa de continuidad; las fuentes de verdad siguen siendo:
> - Linear para scope, estado, dependencias, aceptaciÃ³n y blockers.
> - `waby-docs/` para arquitectura y contratos canÃ³nicos.
> - CÃ³digo/working tree para estado fÃ­sico.
>
> Antes de cambiar nada:
> 1. Inspecciona los repositorios y sus working trees; no limpies, resetees, hagas stash ni sobrescribas cambios.
> 2. Consulta los issues Linear mencionados y sus comments recientes.
> 3. Lee el checkpoint WAB-81 y los contratos directamente relacionados con el siguiente incremento.
> 4. Confirma el estado actual de CRM type-check; el usuario ya solucionÃ³ `@aws-sdk/s3-request-presigner`. No reinstales ni modifiques dependencias.
> 5. Identifica quÃ© estÃ¡ implementado realmente, quÃ© falta y quÃ© estÃ¡ bloqueado.
>
> Nuestro siguiente foco es WAB-81: completar el Runtime de AgentInstance conectado al ExecutionEngine existente, y reconciliarlo con CurrentAuthorization, Integration Executor, lifecycle, durable execution y result delivery. No crear otro engine ni executor paralelo.
>
> MantÃ©n el trabajo local-first. No commit, push, PR, deploy, staging/production ni migraciones live. No modifiques frontend/infrastructure salvo dependencia aprobada. No reabras decisiones aceptadas ni repitas auditorÃ­as sin evidencia nueva.
>
> Usa incrementos pequeÃ±os, tests pertinentes, type-check/lint y revisiÃ³n del diff. Antes de implementar, presenta un plan acotado del incremento y sus criterios de aceptaciÃ³n. Al finalizar, actualiza el checkpoint documental y Linear con evidencia, archivos, pruebas, bloqueos y pendientes. No declares cerrado WAB-81/WAB-79 por scaffolding o validaciones parciales.
>
> Comienza recuperando el estado real y proponiendo el siguiente incremento concreto; no empieces a editar hasta que el scope quede reconciliado.

---

## 9. Reglas para evitar pÃ©rdida de contexto y trabajo duplicado

- Todo checkpoint debe registrar fecha, issue, estado fÃ­sico, archivos relevantes, tests, bloqueos y siguiente acciÃ³n.
- Los cambios no integrados se describen como **locales**, no como completados/remotos.
- Antes de editar un archivo, verificar su contenido actual y cambios dirty.
- Un comentario de Linear no reemplaza evidencia en repo.
- Un documento de diseÃ±o no prueba implementaciÃ³n.
- Un test unitario no prueba integraciÃ³n E2E.
- Una implementaciÃ³n local no implica issue Done.
- No crear versiones paralelas de contratos; reconciliar el documento canÃ³nico.
- No cambiar decisiones de seguridad para â€œhacer que funcioneâ€.
- Si Claude encuentra contradicciÃ³n entre Linear, docs y cÃ³digo, detener el cambio afectado, presentar la contradicciÃ³n con evidencia y resolverla antes de continuar.
- No usar Bot/BotInstance legacy como shortcut.
- No asumir permisos, grants, tenants, bindings o credentials desde datos no autoritativos.
- Mantener los repositorios separados y respetar su ownership.

---

## 10. Checklist de inicio de sesiÃ³n

- [ ] Workspace/path correcto.
- [ ] Plan leÃ­do.
- [ ] Checkpoint mÃ¡s reciente leÃ­do.
- [ ] Linear issues y comments consultados.
- [ ] Working trees inspeccionados.
- [ ] Scope del siguiente incremento confirmado.
- [ ] Dependencias/blockers identificados.
- [ ] Sin auditorÃ­a repetida innecesaria.
- [ ] Sin cambios de dependencias no autorizados.
- [ ] Sin commit/push/deploy.
- [ ] Criterios de aceptaciÃ³n definidos.

## 11. Checklist de cierre de incremento

- [ ] Cambios fÃ­sicos listados.
- [ ] Tests ejecutados y resultados exactos.
- [ ] Type-check/lint ejecutados segÃºn scope.
- [ ] Diff revisado.
- [ ] Limitaciones y blockers explÃ­citos.
- [ ] Docs canÃ³nicas/checkpoint reconciliados.
- [ ] Linear actualizado con evidencia.
- [ ] Estado local/remote explicado.
- [ ] Siguiente incremento y dependencia identificados.
- [ ] No se cerraron issues prematuramente.

---

**Ãšltima actualizaciÃ³n:** 20/11/2026.  
**Este documento debe actualizarse al final de cada incremento significativo y utilizarse como punto de entrada para nuevas sesiones.**


