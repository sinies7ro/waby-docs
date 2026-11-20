# WAB-79 — Informe de reconciliación contractual Outbox e Inbox (§16.14)

> **Estado:** borrador de reconciliación contractual; no aprueba el contrato físico ni autoriza implementación.  
> **Fecha:** 16/11/2026.  
> **Ámbito:** asignaciones, Outbox, Inbox y retiro de AgentInstance relacionados con §16.14.  
> **Naturaleza:** documento independiente; no sustituye ni modifica la Revisión 21.

## 1. Propósito y límites

Este documento consolida la reconciliación contractual focalizada de Outbox pendiente de §16.14, incorporando las correcciones sobre garantías de entrega y aplicación, confirmación de publicación RabbitMQ, ciclo de vida del Outbox, alcance de las confirmaciones de retiro, cursor causal, independencia de decisiones pendientes y estado de las validaciones.

Distingue:
1. decisiones aprobadas en fuentes canónicas;
2. consecuencias contractuales derivadas;
3. propuestas de diseño físico no aprobadas;
4. decisiones pendientes de aprobación;
5. validaciones técnicas que requieren evidencia.

Quedan fuera del alcance: aprobar el schema físico definitivo de §16.14, modificar fuentes canónicas, implementar código, ejecutar pruebas, validar Atlas, actualizar Linear/GitHub o iniciar WAB-81.

La Revisión 21 permanece cerrada y sin modificaciones. Se utiliza solo como antecedente técnico secundario. Este informe no la reabre ni debe incorporarse automáticamente en ella.

## 2. Fuentes y jerarquía

### 2.1 Fuentes contractuales principales

- design/fase-2-diseno-objetivo-y-contratos.md, especialmente D.1.6 (EvAssigned), D.1.7 (resolución ABA) y D.1.8 (atomicidad y persistencia).
- Decisiones WAB-79 D-W79-14 y D-W79-15.
- Contratos de retiro y concurrencia de §16.8–§16.14.
- §8.1, §8.1.1, §8.3, §8.5 y §8.6 para estados lease/fencing.
- §16.9, §16.10 y §16.12 para causalidad, cierre del conjunto de eventos e identidades de retiro.

### 2.2 Fuentes técnicas secundarias

- design/wab-79-viabilidad-tecnica-cierre-revision-21.md, exclusivamente como evidencia de análisis, inventarios y bloqueos.
- Inspección técnica focalizada del Outbox CRM compartida durante el trabajo WAB-79. Sus observaciones describen el código inspeccionado, no el contrato objetivo.

### 2.3 Regla de interpretación

Prevalece el contrato canónico aprobado. Una recomendación, ejemplo de payload, clave ilustrativa o detalle observado en código no se eleva a decisión por aparecer en un informe. Cuando la fuente canónica no fija la estructura física, el punto permanece pendiente.

## 3. Resumen ejecutivo

1. D-W79-14 fija el contrato funcional de EvAssigned; no define el schema físico del Outbox. **D-W79-14 está aprobada** (canónico WAB-78 §D.1.6, WAB-79 §5.2.12, §9; Revisión 21 B1-01).
2. D-W79-15 establece la unidad lógica atómica de una asignación ordinaria: Contacto + EvAssigned + Outbox. **En flujos de retiro, la unidad se extiende a Contacto + EvAssigned + Outbox + actualización de RetirementOperation.eventSetRevision** (§16.8–§16.10). La solución preferida es una transacción MongoDB; la viabilidad en Atlas y la propagación efectiva de sesión siguen pendientes de validación.
3. Los flujos de retiro extienden esa unidad según §16.8–§16.10: incorporan RetirementOperation y actualizan eventSetRevision dentro de la misma transacción de la transición correspondiente.
4. Outbox es un mecanismo de publicación recuperable con posibilidad de duplicados. Publicación al menos una vez no equivale a aplicación exactamente una vez. **Nivel 1 Outbox permanece vacío en cuanto a schema físico aprobado** (Revisión 21).
5. Una aplicación de efecto único solo puede afirmarse condicionalmente, si existe deduplicación durable, aplicación atómica de Inbox/cursor/proyección y tratamiento idempotente de efectos externos. Este informe no declara esa garantía demostrada.
6. retirementOperationId es identidad de correlación contractual en flujos de retiro; su ubicación física en envelope o payload sigue pendiente. **EvAssigned de retiro necesita correlación con retirementOperationId** (§16.12).
7. La identidad física de idempotencia y su índice único siguen pendientes. No se aprueba una clave concreta ni una política de purga que pueda borrar evidencia necesaria para evitar duplicados. **D-W79-15 establece la idempotencia conceptual por evAssignedId**; las dimensiones físicas Outbox e Inbox son independientes y pendientes.
8. E9 (leaseGeneration ausente) permanece bloqueado; E10 requiere leaseGeneration: 0 explícito. No debe introducirse un default Mongoose que colapse ausente y cero. **La semántica de lease/fencing está aprobada en §16.6**; solo la inicialización física permanece pendiente.
9. previousEvAssignedId y el alcance exacto de su transmisión deben separarse de la decisión sobre cursor causal. No se presume valor inicial null ni obligatoriedad para toda transición sin respaldo contractual. **La semántica causal está aprobada en §16.9**; P-04 resuelve representación y alcance.
10. Identidad física de idempotencia e inicialización de leaseGeneration son decisiones independientes. El control de admisión también conserva independencia funcional.
11. No se aprueba el contrato físico definitivo de §16.14 ni se autoriza WAB-81.

## 4. EvAssigned y atomicidad de asignación

### 4.1 D-W79-14 — contrato funcional aprobado

**Aprobada** (canónico WAB-78 §D.1.6; WAB-79 §5.2.12, §9; Revisión 21 B1-01 corregido).

EvAssigned es un discriminator de Evento en la colección eventos de CRM. Reutiliza campos heredados y no duplica contacto ni assignedAt.

Campos heredados relevantes: _id, contacto, workspace, createdAt, updatedAt y searchText cuando corresponda.

Campos propios:
- prevAssignedType y prevAssignedRef;
- newAssignedType y newAssignedRef;
- actorType y actorRef obligatorio;
- operationType;
- reason nullable.

operationType admite los seis valores aprobados: automatic_assignment, self_assignment, manual_assignment, takeover, delegation y unassignment.

originMessageId y delegationChainId no son campos de EvAssigned. originMessageId mantiene su función independiente en el contrato Inbox/mensajería.

Contacto.ref_ev_assigned referencia el último EvAssigned confirmado, también después de una liberación. La identidad del evento resuelve ABA y no se sustituye por originMessageId.

**Alcance exacto:** campos propios/heredados de EvAssigned, `actorRef` obligatorio, `reason` nullable, `operationType` 6 valores, exclusión de `originMessageId`/`delegationChainId`. Reutiliza campos e índices heredados de `Evento`. **No se extiende a cuestiones físicas Outbox no aprobadas (§16.14 pendiente).**

### 4.2 D-W79-15 — atomicidad contractual aprobada

**Aprobada** (WAB-79 §9, §16.12, §16.14; Ownership §16.8–§16.10; Revisión 21).

La transición ordinaria debe preservar una única unidad lógica consistente:
- actualización de Contacto, incluido el estado de asignación y ref_ev_assigned;
- persistencia de EvAssigned;
- registro de Outbox.

**En flujos de retiro (§16.8–§16.10), la unidad se extiende a:**
- actualización de Contacto, incluido el estado de asignación y ref_ev_assigned;
- persistencia de EvAssigned;
- registro de Outbox;
- **actualización de RetirementOperation.eventSetRevision**.

El identificador de EvAssigned se genera antes de la operación condicional y se reutiliza en los elementos correlacionados con esa transición. El CAS valida el estado previo completo de Contacto, incluyendo assignedType, ref_user_assigned, ref_agent_assigned y ref_ev_assigned, según el contrato aplicable.

Un fallo de la unidad atómica debe abortar la transición completa. Las comunicaciones HTTP y proyecciones externas se realizan después del commit mediante el mecanismo recuperable; no se mantiene una transacción abierta durante operaciones de red.

**Requisito contractual D-W79-15:** los consumidores deben ser idempotentes por `evAssignedId`.

### 4.3 Contrato frente a validación técnica

La atomicidad es exigencia contractual. La transacción MongoDB es la solución técnica preferida en D.1.8/D-W79-15, pero su viabilidad concreta requiere comprobar:
- topología y compatibilidad de MongoDB Atlas;
- soporte de transacciones multidocumento;
- propagación de la misma sesión a todas las lecturas y escrituras;
- comportamiento de commit, abort y reintentos;
- compatibilidad de índices, discriminators y colecciones;
- nivel de consistencia de lectura requerido.

readConcern: snapshot permanece como propuesta técnica sujeta a validación, no como decisión canónica confirmada. Si la transacción no resulta viable, una alternativa recuperable requiere diseño y aprobación propios.

**Separación contractual obligatoria:**
- **Garantía contractual:** unidad atómica Contacto + EvAssigned + Outbox (+ RetirementOperation.eventSetRevision en retiro) — aprobada en D-W79-15 y §16.8–§16.10.
- **Decisión de representación física:** schema Outbox, identidad física, envelope, estados — pendiente en §16.14 (Nivel 1 = VACÍO).
- **Verificación técnica:** V-01 a V-03 validan que la implementación satisface el contrato contractual; no sustituyen la decisión contractual.

## 5. Flujos de retiro y RetirementOperation

### 5.1 Extensión de la unidad atómica

La transición de retiro por contacto descrita en §16.8 amplía la unidad ordinaria a Contacto + EvAssigned + Outbox + actualización de RetirementOperation.eventSetRevision.

El CAS verifica que el contacto siga asignado a la instancia que se retira y que ref_ev_assigned coincida con la identidad esperada. La operación preserva correlación con retirementOperationId.

El cierre del conjunto y la fase finalizing siguen §16.10: estado releasing, lease/fencing vigente y ausencia de contactos asignados a la instancia. eventSetClosedAt pertenece al cierre del conjunto; no debe adelantarse ni tratarse como atributo genérico de Outbox.

### 5.2 Identidades contractuales

Las tres identidades lógicas definidas en §16.12 son:

1. **retirementOperationId** — identidad de la operación de retiro completa.
2. **{retirementOperationId, contacto}** — identidad de la transición de retiro por contacto.
3. **evAssignedId** — identidad del evento EvAssigned; sirve como clave de idempotencia de consumidores (D-W79-15) y referencia para la confirmación de aplicación en retirement_confirmations (§16.9).

Son identidades lógicas aprobadas; no determinan nombres de campos, ubicación en Outbox ni clave física de índice. **No se presenta la identidad lógica de retiro como una decisión abierta.**

### 5.3 Alcance de retirement_confirmations

retirement_confirmations se limita a confirmaciones asociadas al retiro de una AgentInstance. No es un registro obligatorio para todas las asignaciones ordinarias.

**Elementos conceptuales ya establecidos en §16.9:**

- Colección: `retirement_confirmations`
- Campos: `evAssignedId`, `retirementOperationId`, `confirmedAt`
- Índice único: `{evAssignedId: 1}`
- Índice de consulta: `{retirementOperationId: 1, evAssignedId: 1}`

Estos elementos conceptuales forman parte del contrato aprobado. **P-07 debe concentrarse exclusivamente en la definición física de RetirementOperation y en las cuestiones de implementación que todavía no están decididas**; no se reabren los campos e índices conceptuales de retirement_confirmations.

Su retención física y materialización deben seguir el contrato específico de retiro. No se extrapolan a transiciones ordinarias ni justifican una garantía exactly-once general.

## 6. Entrega Outbox, Inbox y semántica de aplicación

### 6.1 Publicación: entrega al menos una vez

Outbox permite recuperar publicaciones pendientes y tolerar fallos entre la publicación al broker y la persistencia del estado local. Por ello, un mensaje puede publicarse más de una vez.

Escenario:
1. el worker publica;
2. RabbitMQ acepta;
3. el proceso falla antes de persistir el estado terminal del Outbox;
4. el worker recupera el registro pendiente y vuelve a publicar.

La publicación y el cambio de estado local no son una operación atómica única. La semántica debe describirse como publicación recuperable con posibilidad de duplicados, habitualmente entrega al menos una vez.

**Nivel 1 Outbox = VACÍO** (Revisión 21): no existe un schema físico de Nivel 1 aprobado para el Outbox de asignaciones. Las garantías contractuales de idempotencia, correlación, causalidad y entrega durable existen independientemente del schema físico pendiente.

### 6.2 channel.publish() no equivale a confirmación del broker

channel.publish() por sí solo no demuestra que RabbitMQ confirmó la aceptación durable del mensaje.

En amqplib, el booleano retornado por channel.publish() se relaciona con backpressure del canal; no es por sí mismo un publisher ACK. Cuando el contrato requiera confirmación del broker, debe evaluarse ConfirmChannel y publisher confirms, junto con las garantías de durabilidad/configuración del broker.

Este informe no afirma que el CRM inspeccionado utilice actualmente ConfirmChannel ni que exista confirmación durable demostrada. Requiere inspección y validación específicas.

### 6.3 Ciclo de vida del estado Outbox

SENT solo debe representar que se obtuvo la confirmación definida por el contrato de publicación. No debe asignarse únicamente porque channel.publish() retornó o porque la llamada local no lanzó excepción.

Si el broker acepta el mensaje pero el worker falla antes de persistir SENT, el registro local permanece en el estado pendiente que corresponda y podrá volver a publicarse. El consumidor debe tolerar la repetición.

Los estados observados en la inspección CRM (PENDING, SENT, FAILED) describen el código inspeccionado, no un contrato objetivo aprobado. RETRY o DEAD_LETTER y sus transiciones son propuestas pendientes, no decisiones incorporadas aquí.

### 6.4 Inbox: deduplicación y aplicación idempotente

La entrega duplicada exige que el consumidor reconozca una identidad ya procesada y evite repetir la aplicación de la transición.

**Atomicidad local contractual (§16.9):**
En AI Service, el procesamiento de cada evento del conjunto de retiro ocurre en una transacción local que agrupa atómicamente:

1. Actualización de `contact_assignments`.
2. Avance de `lastProcessedEvAssignedId`.
3. Marcado del registro Inbox como aplicado.
4. Creación de la confirmación durable en el Outbox de AI Service.

Esta atomicidad es **requisito contractual aprobado**, no una propuesta opcional. V-06 valida su cumplimiento técnico (concurrencia, unicidad, orden, recuperación).

No se declara garantía absoluta de exactly-once en la aplicación. La formulación correcta es:

> El consumidor debe aplicar transiciones de forma idempotente y evitar repetir efectos de dominio mediante deduplicación durable y control causal. Una garantía de efecto único solo puede afirmarse bajo condiciones verificadas de persistencia, atomicidad y tratamiento de efectos secundarios.

Para sostener una afirmación más fuerte deben verificarse:
- identidad estable del mensaje y clave durable de deduplicación;
- persistencia de Inbox y actualización del cursor causal dentro de una unidad atómica;
- aplicación atómica de la proyección junto con el registro de procesamiento, cuando pertenezcan a la misma base transaccional;
- comportamiento ante reintentos, concurrencia, mensajes fuera de orden y reinicio;
- idempotencia o reconciliación para efectos externos que no puedan incluirse en la misma transacción;
- retención que no elimine evidencia de deduplicación mientras puedan reaparecer mensajes.

Hasta documentar y validar esos elementos, la garantía es idempotencia requerida por contrato, no exactly-once demostrada.

### 6.5 Confirmación del consumidor y ACK

El ACK de RabbitMQ debe producirse después de persistir durablemente el resultado de procesamiento requerido por el contrato. No debe preceder a la persistencia de Inbox/cursor/proyección que permite recuperar el procesamiento tras una caída.

La forma exacta de confirmar, reencolar o enviar a dead-letter depende del contrato físico del consumer y permanece pendiente. ACK exitoso no demuestra que efectos externos hayan finalizado.

## 7. Correlación, idempotencia y cursor causal

### 7.1 retirementOperationId

En transiciones de retiro, retirementOperationId es identidad contractual necesaria para correlacionar el evento con su operación. **EvAssigned de retiro necesita correlación con retirementOperationId** (§16.12). Su ubicación física —campo de primer nivel, metadata del envelope o payload— no está fijada y permanece pendiente.

**P-02 debe resolver exclusivamente su representación física y propagación en Outbox, envelope y transporte.**

### 7.2 Identidad física de mensajes Outbox

La idempotencia es un invariante: repetir la misma transición no debe crear una segunda aplicación lógica ni efectos de dominio duplicados.

**Requisito contractual D-W79-15:** los consumidores deben ser idempotentes por `evAssignedId`.

**Tres dimensiones de identidad a distinguir:**

1. **Idempotencia lógica de asignaciones ordinarias** — clave `evAssignedId` (aprobada D-W79-15).
2. **Identidad de transiciones de retiro** — `{retirementOperationId, contacto}` (aprobada §16.12).
3. **Identidad física de mensajes Outbox** — pendiente (§16.14).

La **identidad física y deduplicación de mensajes Inbox** es una dimensión distinta que corresponde a **P-03**, no a P-01.

La clave física de Outbox no está aprobada. Alternativas a evaluar:
- clave derivada de evAssignedId y contacto;
- clave derivada de retirementOperationId y contacto para retiro;
- clave sintética estable que represente inequívocamente la operación;
- claves separadas por tipo de transición si se demuestra necesario.

No se elige una alternativa. La discrepancia entre usar evAssignedId + contactId para ambos flujos y retirementOperationId + contactId para retiro debe resolverse antes de definir el índice.

**P-01 resuelve exclusivamente la identidad física de mensajes Outbox (dimensión 3).**

### 7.3 Índices únicos y ciclo de vida

Un índice único parcial que excluya estados terminales puede permitir que una identidad vuelva a insertarse tras SENT o FAILED. No garantiza idempotencia durable sin analizar el ciclo completo.

Un índice único permanente sobre identidad estable es propuesta candidata, no decisión aprobada. Debe considerar unicidad por transición/tipo, asignaciones ordinarias y retiros, terminalidad/reintentos, retención/purga, reconciliación y consultas reales.

No se aprueba TTL general sobre createdAt (por ejemplo, 30 días). Una política que pueda borrar Outbox pendiente, fallido o evidencia de deduplicación sería incompatible con recuperación segura. Retención y purga requieren decisión independiente.

### 7.4 previousEvAssignedId y causalidad

**Semántica causal aprobada en §16.9:**
- Aplicación cuando el cursor coincide con `previousEvAssignedId`.
- Espera de eventos futuros cuando falta su predecesor.
- Tratamiento de eventos obsoletos sin regresión de la proyección.

El cursor causal debe distinguirse de la decisión sobre un campo físico `previousEvAssignedId` (nombre canónico conservado).

No se eleva a contrato general su transmisión en todas las transiciones. Antes deben verificarse:
- tipos de transición que requieren encadenamiento;
- si representa evento previo del contacto, cursor de proyección o relación causal del mensaje;
- semántica de la primera transición;
- si ausencia se representa con campo ausente o null;
- validación de orden ante concurrencia y duplicados.

No se presume que la primera transición envíe null ni que el campo sea obligatorio en todos los payloads sin respaldo explícito en §16.9 y contratos relacionados.

**P-04 debe concentrarse en:**
- Alcance de transmisión.
- Primera transición.
- Representación física de ausencia o null.
- Validación de concurrencia y orden.

## 8. Lease, fencing y distinción absent/zero

### 8.1 Estados relevantes

Se distinguen leaseGeneration ausente, leaseGeneration: 0 explícito y leaseGeneration positivo. Ausencia no equivale a cero y no debe colapsarse mediante defaults automáticos.

**Semántica de lease/fencing aprobada en §16.6:**
- Adquisición atómica.
- Incremento de generación en cada nueva adquisición efectiva.
- Renovación sin incrementar generación.
- Validación de propietario, generación y vigencia en toda escritura.
- Rechazo de workers obsoletos (generación vencida no puede avanzar estado ni escribir progreso).
- Recuperación conserva estrategia, destino, bloqueos y transiciones confirmadas; consulta autoridades con el mismo ID, reanuda desde `resumeFrom` y no repite efectos confirmados.

### 8.2 E9 y E10

- **E9 — leaseGeneration ausente:** permanece bloqueado/no autorizado como estado de transición hasta decisión contractual explícita.
- **E10 — leaseGeneration: 0 explícito:** exige que el cero se escriba explícitamente en la inicialización definida por el contrato.

La recomendación de inicializar con cero no autoriza default: 0 en Mongoose. Un default podría hacer indistinguible una propiedad omitida de una inicializada explícitamente, contradiciendo §8.1/§8.6 y la semántica aprobada en §16.6.

El punto exacto del ciclo de vida en que se escribe el cero y las condiciones de primera adquisición deben documentarse sin reinterpretar E9 como transición permitida hacia E10.

**P-08 mantiene pendiente la inicialización física de `leaseGeneration` (ausente vs. cero), sin reabrir la semántica aprobada en §16.6.**

## 9. Propuestas técnicas no aprobadas y matriz de contextos Outbox

### 9.1 Matriz de contextos Outbox

Los siguientes **tres contextos Outbox** son distintos con propósitos, productores, consumidores, garantías y definiciones físicas independientes. No se traslada automáticamente un contrato de un flujo a otro.

| Contexto | Propósito | Productor | Consumidor | Garantías contractuales aplicables | Identidad lógica disponible | Definiciones físicas pendientes | Referencias canónicas |
|---|---|---|---|---|---|---|---|
| **1. Outbox asignaciones CRM** | Publicar transiciones de asignación ordinaria | CRM (transacción D-W79-15) | AI Service (Inbox) | D-W79-15 (atomicidad), §16.9 (causalidad Inbox), contratos específicos de asignación | `evAssignedId` | Schema, envelope, estados, identidad física, índices | D.1.8, D-W79-15, §16.9, §16.14 |
| **2. Outbox CRM → AI Service** | Mensaje de origen para iniciar ejecución; órdenes de bloqueo, drenaje y finalización de retiro | CRM | AI Service | **D.6/ARQ-22: solo al mensaje de origen que inicia una ejecución** (Fase 2 §D.6.1). Órdenes de bloqueo, drenaje y finalización de retiro: contratos específicos §16.11–§16.13 | `executionId`, `retirementOperationId` | Schema, envelope, confirmación broker, estados | D.6, ARQ-22, §16.11, §16.13 |
| **3. Outbox AI Service → CRM** | Resultado terminal de ejecución; confirmaciones de aplicación, drenaje y finalización de retiro | AI Service | CRM | **D.6/ARQ-22: solo al resultado terminal de una ejecución** (Fase 2 §D.6.1). Confirmaciones de aplicación: §16.9. Confirmaciones de drenaje y finalización de retiro: §16.11–§16.13 | `evAssignedId`, `retirementOperationId`, `executionId`, `eventId` | Schema, envelope, confirmación broker, estados | D.6, ARQ-22, §16.9, §16.11, §16.13 |

**Aclaración sobre `retirement_confirmations`:**
La colección `retirement_confirmations` **no es un Outbox**. Es un registro durable de confirmaciones de aplicación persistido por CRM (§16.9). Las órdenes y confirmaciones del protocolo de retiro utilizan los mecanismos Outbox de los servicios correspondientes según los contratos canónicos:
- CRM publica la orden de finalización en su **Outbox CRM → AI Service** (contexto 2), identificado por `retirementOperationId` (§16.13).
- AI Service procesa la orden vía Inbox y publica la confirmación terminal en su **Outbox AI Service → CRM** (contexto 3), dentro de la misma transacción local que marca la orden Inbox como procesada (§16.13).
- CRM recibe la confirmación terminal vía Inbox y la persiste en `RetirementOperation.finalizationConfirmation` (no en `retirement_confirmations`).

**Alcance de D.6 / ARQ-22:**
D.6 cubre **exclusivamente dos tramos** (Fase 2 §D.6.1):
1. **CRM → AI Service (contexto 2)**: publicación del **mensaje de origen** que inicia una ejecución en AI Service.
2. **AI Service → CRM (contexto 3)**: publicación del **resultado terminal** de una ejecución para que CRM lo procese.

**No se extiende D.6/ARQ-22 a:**
- Outbox de asignaciones de D.1.8 (contexto 1) — rige D-W79-15 y contratos específicos de asignación.
- Órdenes de bloqueo, drenaje y finalización de retiro (viajan por contexto 2) — rigen contratos específicos §16.11–§16.13.
- Confirmaciones de aplicación (viajan por contexto 3) — rige §16.9.
- Confirmaciones de drenaje y finalización de retiro (viajan por contexto 3) — rigen §16.11–§16.13.

Cuando una garantía sea compartida por otro contrato, citar ese contrato específico. Si no está definida, conservarla como pendiente de §16.14.

La tabla debe permitir identificar qué garantía corresponde a cada tipo de mensaje, sin inferirla únicamente por compartir productor, consumidor o colección Outbox.

**Notas:**
- Cada contexto Outbox puede tener schema, envelope, estados, ciclo de vida e identidad física propios.
- La identidad lógica disponible no determina la identidad física (§16.14 pendiente).

### 9.2 Propuestas técnicas (no aprobadas)

Las siguientes propuestas permanecen **exclusivamente como propuestas no aprobadas**. Cada una debe indicar fundamento, alcance, limitaciones contractuales y decisión pendiente.

| Propuesta | Estado | Fundamento / Condición | Limitaciones contractuales |
|---|---|---|---|
| idempotencyKey de primer nivel | Pendiente | Resolver identidad física Outbox por tipo de transición y correlación | No confundir con idempotencia lógica `evAssignedId` (D-W79-15) |
| Índice único permanente | Candidata, no aprobada | Definir clave, alcance, retención y ciclo de vida | Debe preservar deduplicación y recuperación; no borrar pendientes |
| retirementOperationId en primer nivel | Pendiente | Definir envelope/payload y consultas | Solo para contextos que lo requieran (2, 3) |
| previousEvAssignedId en payload | Pendiente | Verificar alcance causal y primera transición | Semántica causal ya aprobada §16.9; resolver representación física |
| ConfirmChannel / publisher confirms | Diseño/validación pendiente | Confirmación broker, durabilidad, NACK/timeout | No aprobado; evaluar por contexto Outbox |
| Estados RETRY / DEAD_LETTER | Pendiente | Máquina de estados y política de reintentos | Por contexto Outbox; no generalizable |
| attempts con inicialización explícita | Pendiente | Contrato y defaults | Evitar defaults que colapsen absent/zero |
| readConcern: snapshot | Validación técnica pendiente | Topología Atlas y comportamiento real | Propuesta técnica, no decisión canónica |
| TTL o retención temporal | No aprobado | Preservar pendientes y deduplicación | Retención por tipo/estado, no TTL general sobre createdAt |
| Retención de RetirementOperation por un año | No aprobada | Justificar lifecycle, auditoría y recuperación | Política independiente por contexto |
| Estrategias de recuperación y reintentos | No aprobada | Definir por contexto Outbox y tipo de fallo | No generalizable |
| Configuración específica de RabbitMQ | No aprobada | Durabilidad, colas, TTL mensaje, DLQ | Por contexto Outbox y flujo |

## 10. Separación de decisiones y dependencias

Estas cuestiones no forman una única secuencia:
1. **Identidad física de idempotencia:** depende de semántica de transición, correlación y retención.
2. **Inicialización de leaseGeneration:** depende de E9/E10 y lifecycle de lease.
3. **Control de admisión:** conserva independencia funcional respecto del schema Outbox.
4. **Validación de transacciones Atlas:** valida la solución preferida; no reemplaza la decisión contractual de atomicidad.
5. **Contrato Inbox/cursor:** debe especificarse para consumidores y proyecciones; no se resuelve automáticamente al cerrar schema Outbox.

La resolución de identidad de idempotencia no es prerequisito secuencial para decidir leaseGeneration, ni viceversa.

## 11. Matriz de clasificación contractual

| Tema | Decisión aprobada | Consecuencia derivada | Propuesta técnica | Pendiente |
|---|---|---|---|---|
| EvAssigned | Campos y semántica D-W79-14 | Identidad única de transición | Schema/índice adicional | Ninguno sobre campos aprobados |
| Asignación atómica | D-W79-15 y D.1.8 como requisito | Contacto, EvAssigned y Outbox (+ RetirementOperation.eventSetRevision en retiro) forman unidad | Transacción MongoDB/snapshot | Atlas y sesiones |
| Retiro por contacto | §16.8–§16.10 | RetirementOperation participa en unidad de retiro | Forma física RetirementOperation | Validación y schema físico |
| Publicación Outbox | Publicación recuperable (al menos una vez) | Duplicados posibles | ConfirmChannel/publisher confirms | D.6/ARQ-22 aplica exclusivamente al mensaje de origen CRM → AI Service que inicia una ejecución (contexto 2) y al resultado terminal AI Service → CRM (contexto 3). Los restantes mensajes de esos contextos se rigen por sus contratos específicos (§16.9, §16.11–§16.13) cuando estén definidos. Garantías no definidas permanecen pendientes de §16.14. No se deben inferir garantías por compartir productor, consumidor o colección Outbox. |
| Estado Outbox | No se fija nuevo enum | SENT requiere confirmación definida por contrato | PENDING/SENT/FAILED, RETRY, DLQ | Máquina de estados por contexto (§9.1) |
| Inbox | **Atomicidad local §16.9 aprobada** (contact_assignments + lastProcessedEvAssignedId + Inbox aplicado + Outbox confirmación) | Reintentos no duplican aplicación lógica | Inbox durable y transacción local | Garantías verificables y efectos externos (V-06) |
| Correlación de retiro | retirementOperationId + {retirementOperationId, contacto} (§16.12) | Cada transición de retiro debe correlacionarse | Campo/envelope/payload | Ubicación física (P-02) |
| Idempotencia | **D-W79-15: consumidores idempotentes por evAssignedId**; §16.12: 3 identidades lógicas | Repetición no crea efecto lógico duplicado | Clave e índice físicos Outbox/Inbox | Clave e índice físicos aprobados (P-01) |
| retirement_confirmations | Limitada al retiro; **campos/índices conceptuales aprobados §16.9** | No aplica a toda asignación ordinaria | Colección/índice específico | Schema físico y retención (P-07 reducido) |
| Cursor causal | **Semántica §16.9 aprobada** (aplicar si cursor=prevEvAssignedId; esperar predecesor; ignorar obsoletos) | Orden y reconciliación verificables | previousEvAssignedId (nombre canónico) | Alcance transmisión, 1ra transición, absent/null (P-04) |
| Lease E9 | Ausencia no equivale a cero; bloqueada; **semántica §16.6 aprobada** | No aceptar estado ambiguo | Ninguna | Inicialización física absent/zero (P-08) |
| Lease E10 | Cero explícito; **semántica §16.6 aprobada** | Escritura explícita al inicializar | Código de inicialización | Momento y validación (P-08) |
| Retención | Sin TTL general aprobado | No borrar evidencia recuperable | TTL por estado/tipo | Política por contexto Outbox (P-06) |
| Control de admisión | Independiente | No mezclar con Outbox | Diseño propio | Contrato en su ámbito |

## 12. Registro de decisiones pendientes

### 12.1 Pendientes de aprobación contractual

- **P-01 — Identidad física Outbox:** **Identidad lógica por `evAssignedId` aprobada (D-W79-15)**; resolver identidad física y deduplicación de mensajes Outbox (clave, índice, alcance) para asignación ordinaria y retiro, separando de la identidad de transiciones de retiro `{retirementOperationId, contacto}`. **No incluye Inbox.**
- **P-02 — Ubicación de correlación:** definir representación física y propagación de `retirementOperationId` en Outbox, envelope y transporte (contextos 2 y 3 de §9.1). **Identidad contractual ya aprobada (§16.12)**.
- **P-03 — Contrato físico Inbox:** **Atomicidad local aprobada §16.9** (contact_assignments + lastProcessedEvAssignedId + Inbox aplicado + Outbox confirmación); resolver schema físico, identidad durable, unicidad y deduplicación física de Inbox, cursor causal, orden, ACK/retry y límites de aplicación. **Conserva como requisito aprobado la atomicidad local de §16.9.**
- **P-04 — Alcance de previousEvAssignedId:** **Semántica causal aprobada §16.9**; resolver representación física, alcance de transmisión, primera transición, ausencia/null y validación de concurrencia/orden. Conservar nombre canónico `previousEvAssignedId`.
- **P-05 — Máquina de estados Outbox:** estados y transiciones, reintentos y terminalidad **por contexto Outbox** (§9.1).
- **P-06 — Retención/purga:** retención por tipo/estado preservando recuperación y deduplicación; **política independiente por contexto Outbox/Inbox** (§9.1).
- **P-07 — Definición física RetirementOperation:** **Campos/índices conceptuales de retirement_confirmations ya aprobados §16.9**; resolver schema físico, índices y lifecycle de RetirementOperation sin reabrir confirmaciones.
- **P-08 — Inicialización leaseGeneration:** **Semántica §16.6 aprobada**; resolver punto de escritura explícita de cero, tratamiento absent vs. zero, y mantenimiento del bloqueo E9.
- **P-09 — Envelope/transporte RabbitMQ:** confirmación broker, NACK/timeout, persistencia y ACK del consumidor **por contexto Outbox** (§9.1).

Cada decisión incluye: qué está aprobado, qué sigue pendiente, alternativas relevantes (si están respaldadas), consecuencias, dependencias y referencias canónicas exactas. No se eliminan decisiones sin justificación documental de que están resueltas. **P-01 y P-03 tienen responsabilidades separadas: P-01 = Outbox físico; P-03 = Inbox físico.**

### 12.2 Validaciones técnicas pendientes

Las diez validaciones se clasifican por tipo de evidencia requerida:

| Validación | Tipo de evidencia | Contrato que verifica | Evidencia necesaria | Criterio verificable de aceptación |
|---|---|---|---|---|
| **V-01 — Atlas** | Inspección infraestructura | D.1.8 / D-W79-15: transacciones multidocumento | Configuración clúster Atlas, versión MongoDB, topología (replica set / sharded) | Clúster compatible con transacciones ACID multidocumento |
| **V-02 — Sesiones** | Inspección código | D.1.8 / D-W79-15: propagación misma sesión | Código de transacción Contacto+EvAssigned+Outbox+RetirementOperation | Una sola `ClientSession` propagada a todas las operaciones |
| **V-03 — Commit/rollback MongoDB** | Prueba técnica | D.1.8 / D-W79-15: atomicidad completa | Pruebas de fallo en cada punto (Contacto, EvAssigned, Outbox, RetirementOperation); manejo de `UnknownTransactionCommitResult`, `TransientTransactionError`, reintentos de commit/transacción según driver | Abort completo sin estados parciales; reintentos de transacción y commit según driver; resultado incierto de commit consultando estado persistido |
| **V-04 — Publisher confirms RabbitMQ** | Prueba técnica | **D.6/ARQ-22 (solo tramos contractuales: mensaje de origen CRM→AI Service y resultado terminal AI Service→CRM); §16.14 (contextos 2 y 3); contratos específicos §16.9, §16.11–§16.13 para el resto** | Pruebas con **mecanismo de confirmación de publicación candidato (ConfirmChannel u equivalente sujeto a aprobación)**: ACK/NACK del broker, timeout de confirmación, reconexión, backpressure (publish devuelve false), mensajes no enrutables, caída post-confirm. **Verificación de condiciones de persistencia en broker**: durabilidad de exchange/cola, modo persistente del mensaje, configuración/topología pertinente. | **Mensaje persistido en broker antes de SENT local** (para los tramos D.6 y contratos específicos que lo requieran); **manejo de confirmación incierta sin absorber como éxito**; **distinción ACK/NACK, timeout, confirmación incierta, backpressure, mensajes no enrutables**; deduplicación por consumidor obligatoria. **No se eleva ConfirmChannel a decisión aprobada**: figura como mecanismo candidato sujeto a aprobación. **No se atribuye D.6/ARQ-22 a todos los mensajes de los contextos 2 y 3**. **Distinción mantenida**: `UnknownTransactionCommitResult` pertenece a V-03. |
| **V-05 — Duplicados** | Prueba técnica | §16.9 / D-W79-15: idempotencia consumidor | Publicación duplicada intencional; procesamiento repetido | Consumidor no duplica aplicación lógica; deduplicación por `evAssignedId` |
| **V-06 — Inbox/cursor/proyección** | Inspección código + Prueba técnica | **§16.9 atomicidad local aprobada** | Código transacción Inbox; pruebas concurrencia, orden, recuperación | Atomicidad 4 operaciones; cursor causal respeta predecesor; sin regresión proyección |
| **V-07 — Efectos externos** | Inspección código | §16.9: idempotencia/reconciliación para confirmaciones de aplicación; D.6/ARQ-22: exclusivamente para mensaje de origen CRM→AI Service y resultado terminal AI Service→CRM dentro de sus dos tramos contractuales; contratos específicos §16.11–§16.13 para órdenes/confirmaciones de retiro. Efectos externos (notificaciones, webhooks, proyecciones externas) requieren idempotencia o reconciliación específica y validación por tipo de efecto. Garantías no definidas para efectos externos permanecen pendientes de §16.14. No se extiende D.6 como garantía general de idempotencia de efectos externos. | Código de efectos externos (notificaciones, webhooks, etc.) | Idempotencia o reconciliación documentada y probada por tipo de efecto externo; distinción contractual mantenida |
| **V-08 — Índices/discriminators** | Inspección código | §16.14 / D-W79-14: compatibilidad Mongoose | Definiciones schema, índices parciales/únicos, discriminators | Índices creados sin conflicto; discriminators funcionan en transacciones |
| **V-09 — Retención y purga** | Verificación infraestructura + Inspección código | §16.14 / P-06: preservar recuperación y deduplicación | Políticas de retención/purga por contexto Outbox/Inbox, estado contractual y política aprobada | No eliminar registros necesarios para recuperación o procesamiento pendiente; no eliminar evidencia de deduplicación mientras sea necesaria para idempotencia; no purgar eventos sin estado terminal verificable; purga definida según contexto, estado y política aprobada (P-06 pendiente) |
| **V-10 — Lease/fencing** | Inspección código | **§16.6 semántica aprobada** | Código leaseGeneration, adquisición, renovación, validación worker | Absent ≠ zero; generación incrementa en adquisición; workers obsoletos rechazados |

V-01…V-10 son etiquetas de seguimiento; **no significan que las pruebas estén iniciadas ni que exista evidencia de aprobación**. No declarar completada ninguna validación sin evidencia verificable. Distinguir requisito contractual de evidencia de cumplimiento.

## 13. Criterios de cierre de §16.14

La reconciliación contractual distingue **cuatro estados independientes** que no se conceden mutuamente:

### A. Reconciliación documental

El informe puede considerarse documentalmente reconciliado cuando:
1. Las fuentes canónicas están correctamente reflejadas.
2. Las decisiones aprobadas no se reabren.
3. Las decisiones pendientes están delimitadas con trazabilidad a §16.9, §16.12, §16.14.
4. Las propuestas técnicas se identifican como no aprobadas (Sección 9).
5. Existe trazabilidad suficiente para que un tercero distinga: aprobado, derivado, propuesta, pendiente, requiere evidencia.
6. Las contradicciones documentales identificadas fueron corregidas o registradas.

**Este estado no significa que las decisiones físicas de §16.14 estén resueltas.**

### B. Cierre contractual de §16.14

Requiere resolver y aprobar las decisiones contractuales pendientes que correspondan, con sus representaciones y garantías suficientemente definidas. No se declara este cierre por el solo hecho de que P-01 a P-09 estén enumeradas o delimitadas; requiere aprobación explícita de cada decisión física.

### C. Validación técnica V-01 a V-10

Requiere evidencia verificable de inspecciones y pruebas según la clasificación de §12.2. No se presume completada por el cierre documental (A) ni contractual (B).

### D. Autorización de implementación funcional

Es una autorización independiente y explícita. **Ninguno de los tres estados anteriores (A, B, C) la concede automáticamente.**
- Requiere: cierre contractual (B) + validaciones técnicas ejecutadas con evidencia (C) + decisión explícita del responsable.
- No declarar que WAB-79 está listo para implementar si las validaciones o autorizaciones correspondientes siguen pendientes.

---

**Criterios operativos de la reconciliación documental (A):**
1. Decisiones aprobadas correctamente identificadas y referenciadas a fuentes canónicas.
2. Decisiones pendientes (P-01 a P-09) delimitadas con trazabilidad a §16.9, §16.12, §16.14.
3. Contratos coherentes entre fuentes (Fase 2, Ownership, Revisión 21).
4. Propuestas técnicas identificadas como no aprobadas (Sección 9).
5. Trazabilidad documental suficiente para que un tercero distinga: aprobado, derivado, propuesta, pendiente, requiere evidencia.
6. Contradicciones documentales identificadas corregidas o registradas.

**El cierre contractual (B) requiere además:**
- Resolución y aprobación de P-01 a P-09 con representaciones y garantías definidas.
- Aprobación explícita de cada decisión física.

**La validación técnica (C) requiere:**
- Evidencia verificable según §12.2 para V-01 a V-10.

**La autorización de implementación (D) requiere:**
- B + C + decisión explícita del responsable.

No se condiciona la reconciliación documental (A) a la resolución de decisiones físicas ni a la ejecución de validaciones técnicas.

## 14. Orden de trabajo recomendado

1. **Resolver en paralelo P-01 (identidad física Outbox) y P-08 (inicialización leaseGeneration)**, porque son independientes (Sección 10). P-03 (Inbox físico) puede avanzar en paralelo según §10.
2. **Definir P-02 (correlación física retirementOperationId), P-03 (contrato físico Inbox), P-04 (previousEvAssignedId físico), P-05 (máquina estados por contexto) y P-09 (envelope/transporte por contexto)** con trazabilidad a §16.9, §16.12 y §16.14 y a la matriz de contextos §9.1 (tres contextos Outbox).
3. **Definir P-06 (retención/purga por contexto) y P-07 (schema físico RetirementOperation)** sin políticas de retención genéricas ni reabrir confirmaciones conceptuales.
4. Mantener el control de admisión en su ámbito funcional separado (Sección 10, fila 11 de Matriz §11).
5. **Tras aprobar contratos físicos (P-01 a P-09), diseñar schemas físicos por contexto y preparar validaciones V-01…V-10** según clasificación §12.2.
6. Someter cualquier incorporación a fuentes canónicas a revisión y autorización explícitas separadas.

Este orden no autoriza implementación ni anticipa WAB-81. No imponer dependencias artificiales entre decisiones independientes (P-01 y P-08 son paralelas; P-02/P-03/P-04/P-05/P-09 pueden avanzar en paralelo tras P-01).

## 15. Exclusiones y salvaguardas

- No modificar D-W79-14 ni D-W79-15.
- No modificar §16.14 ni otra sección canónica como parte de esta reconciliación.
- No modificar la Revisión 21; permanece cerrada.
- No modificar código en crm/, WABY_frontend/, infrastructure/ ni otros repositorios de ElemeSoft.
- No actualizar Linear ni GitHub.
- No ejecutar pruebas ni realizar cambios en Atlas.
- No iniciar WAB-81 ni interpretar este documento como autorización de implementación.
- No elevar recomendaciones técnicas a decisiones aprobadas.
- No afirmar exactly-once sin evidencia de deduplicación durable, atomicidad local y tratamiento de efectos externos.

## 16. Estado final de esta reconciliación

**Resultado:** reconciliación documental focalizada consolidada como documento independiente.

**Cerrado en este informe (nivel contractual/documental):**
- Distinción entre contrato y mecanismo.
- Semántica de publicación con duplicados posibles (entrega al menos una vez).
- Diferencia entre `channel.publish()` y publisher confirms.
- Ciclo de vida de SENT requiere confirmación contractual definida.
- **Alcance y elementos conceptuales de retirement_confirmations aprobados en §16.9** (campos, índices).
- **Semántica causal aprobada en §16.9** (aplicar si cursor=prevEvAssignedId; esperar predecesor; ignorar obsoletos sin regresión).
- **Semántica lease/fencing aprobada en §16.6** (adquisición atómica, incremento generación, validación owner/gen/vigencia, rechazo workers obsoletos).
- **Atomicidad Inbox aprobada en §16.9** (4 operaciones atómicas en AI Service).
- **Tres identidades lógicas aprobadas en §16.12** (retirementOperationId, {retirementOperationId, contacto}, evAssignedId).
- **Idempotencia por evAssignedId aprobada en D-W79-15**.
- **Nivel 1 Outbox = VACÍO** (Revisión 21) — no schema físico aprobado.
- Independencia de decisiones (P-01/P-08 paralelas; control admisión separado).
- Clasificación de validaciones por tipo de evidencia (V-01 a V-10).
- Cuatro estados de cierre diferenciados: reconciliación documental (A), cierre contractual (B), validación técnica (C), autorización implementación (D).

**Permanece abierto (requiere aprobación/validación):**
- Decisiones físicas P-01…P-09 (schema, identidad física, envelope, estados, retención, inicialización leaseGeneration).
- Validaciones técnicas V-01…V-10 (sin evidencia ejecutada).
- Este informe no las declara aprobadas ni ejecutadas.

**Siguiente hito:** revisión del contenido por el usuario. Cualquier modificación del diseño canónico se tratará como tarea posterior, con alcance y autorización propios.
