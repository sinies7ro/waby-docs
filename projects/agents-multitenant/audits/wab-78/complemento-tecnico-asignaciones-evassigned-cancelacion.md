# WAB-78 — Complemento Técnico: Asignaciones Unificadas, EvAssigned y Cancelación de Ejecuciones

**Estado:** Material técnico de apoyo de WAB-78; no es una segunda fuente normativa. Sin implementación.

> **Nota de vigencia documental (06/11/2026):** este complemento se conserva como material técnico de apoyo y evidencia de análisis; el contrato normativo vigente está integrado en [el diseño canónico de Fase 2](../../design/fase-2-diseno-objetivo-y-contratos.md). La afirmación de que ai-service no estaba disponible localmente quedó obsoleta: el repositorio crm contiene microservices/ai-service. Esta reconciliación no repite su auditoría ni valida la implementación actual de cancelación o AbortController. Los detalles pendientes deben leerse según el diseño canónico y Linear.


**Proyectos afectados:** crm/ (backend principal y microservicio crm/microservices/ai-service/).

---

## 1. Resumen ejecutivo

| Cuestión | Conclusión (decisión aprobada / pendiente) |
|---|---|
| CRM como autoridad única de la asignación | **Aprobado.** CRM es la única autoridad de la asignación efectiva de Contacto. ai-service mantiene `contact_assignments` como proyección operativa, no como autoridad. |
| Cancelación de ejecuciones IA | **Dirección aprobada.** Cancelar ejecuciones activas e invalidar resultados tardíos en agent → human, agent A → agent B y agent → unassigned. El mecanismo de cancelación en ai-service requiere diseño/validación externa (no verificado localmente). |
| EvAssigned exclusivamente en CRM | **Aprobado.** EvAssigned es el historial funcional de asignaciones, persiste exclusivamente en CRM, no se sincroniza con ai-service. |
| Modelo unificado de Contacto | **Aprobado.** `assignedType` enum (`human`/`agent`/`unassigned`) con `ref_user_assigned` y `ref_agent_assigned` según invariantes. Sin `assignmentVersion`. |
| `ref_agent_assigned → AgentInstance` en CRM | **Aprobado.** Referencia tipada a `AgentInstance._id`, ownership `AgentInstance → Workspace → Organización`. Modelo detallado diferido a WAB-79. |
| Orden desarrollo → migración | **Aprobado.** Campos opcionales primero, compatibilidad, despliegue de código compatible, backfill idempotente después, activar routing, retirar compatibilidad antigua al final. |
| Transacción Contacto + EvAssigned + Outbox | **Solución preferida.** Pendiente de verificar compatibilidad real de infraestructura (Atlas, sesiones, colecciones). |
| `contactAssignmentHistory` | **Conservar en esta fase.** EvAssigned cubre auditoría funcional futura; `contact_assignments` sigue necesario para runtime G1-G4. No eliminar durante esta fase. |

**Restricciones respetadas:** No se modifica código, schemas, modelos, RabbitMQ, tests, ni se ejecuta migración. No se introduce `assignmentVersion`. No se sincroniza historial de EvAssigned con ai-service. `operationType` es una propuesta, no una decisión aprobada.

---

## 2. Parte A — Cancelación de ejecuciones por contacto

### 2.1 Estado actual comprobado en código (CRM)

| Archivo | Línea | Elemento | Observación |
|---|---|---|---|
| `src/Rabbit/consumers/executionCompleted.consumer.ts` | 59, 161 | `executionId` | Dedup via `SET bot:exec:{executionId} NX EX 86400` (D5) |
| `src/Rabbit/consumers/executionCompleted.consumer.ts` | 221-267 | G3 | `evaluarVigenciaBot` consulta `contact_assignments` de ai-service; descarta si `human_takeover` / `assignment_changed` / `bot_deactivated` |
| `src/Rabbit/consumers/executionCompleted.consumer.ts` | 345-352 | dispatchOutgoing | Recibe `vigencia?: { botId, contactId }`; solo botId se valida en G4 |
| `src/infrastructure/ai-service/aiVigencia.repo.ts` | 65-123 | `evaluarVigenciaBot` | Lee `bots` + `contact_assignments` en DB `ai_service`; motivos decisivos: `bot_deactivated`, `human_takeover`, `assignment_changed`, `workspace_mismatch` |
| `src/infrastructure/ai-service/ai-service.client.ts` | 55-82 | `AiServiceClient.request` | HTTP con retry (máx 3), timeout 30s, códigos reintentables 502/503/504, ECONNRESET, ETIMEDOUT, ENOTFOUND |
| `src/Rabbit/queues.enum.ts` | 5, 9-11 | Colas | `AI_EXECUTION_COMPLETED` = `crm.execution.completed`; `AI_EXECUTION_COMPLETED_VIGENCIA_RETRY` para G3 delay |
| `src/services/BotRouting/BotRouting.policy.ts` | 66-75 | Routing | Lee `contactAssignmentClient.getActive(contactId)` (ai-service), NO lee CRM directamente |

**Hecho comprobado:** El CRM no tiene registro de ejecuciones activas indexado por contacto. El `executionId` solo se usa para dedup en el consumer de `execution.completed` (D5), no para cancelación.

**Hecho comprobado:** G3 (`executionCompleted.consumer.ts`) ya descarta resultados si `evaluarVigenciaBot` retorna inválido (human_takeover, assignment_changed, etc.). Esto es una forma de **invalidación del resultado**, pero ocurre después de que ai-service ya procesó la ejecución.

**Inferencia:** No existe en el CRM visible un mecanismo para:
- Listar ejecuciones activas por contacto
- Enviar orden de cancelación a un worker de ejecución
- Abortar una llamada LLM en curso
- Invalidar un resultado antes de que llegue a G3

### 2.2 ¿Qué puede cancelarse realmente? (inferencia basada en arquitectura)

El microservicio ai-service, ubicado en crm/microservices/ai-service/, se identifica en este análisis mediante la documentación de docs/business/bot-routing.md línea 198. La afirmación original de indisponibilidad local se considera obsoleta y no se usa como conclusión vigente.

- **G1:** `MessageReceivedHandler` (ai-service, entrada)
- **G2:** `ExecutionEngine.process` (ai-service, pre-publicación)
- **G3:** `handleExecutionCompleted` (CRM, `executionCompleted.consumer.ts`)
- **G4:** `vigenciaPermitida` (microservices/Difusion, outbound-interactive worker)

**Inferencia:** El `ExecutionEngine` vive en ai-service. La cancelación operativa debe coordinarse entre:
1. CRM (autoridad de asignación, detecta el cambio)
2. ai-service (propietario de la ejecución activa, puede cancelarla)
3. Difusion (G4, puede bloquear el envío final)

### 2.3 Cuándo cancelar (matriz de transiciones)

| Transición | Cancelar ejecuciones activas | Invalidar resultados tardíos |
|---|---|---|
| agent → human | Sí, del agente anterior | Sí |
| agent A → agent B | Sí, de A | Sí, de A |
| agent → unassigned | Sí, del agente | Sí |
| human → unassigned | N/A (no hay IA) | N/A |
| unassigned → agent | Habilitar nuevas | N/A |

**Política:** La cancelación se aplica por contacto, no suspende globalmente una AgentInstance que tenga otros contactos.

### 2.4 Cancelación operativa vs invalidación del resultado

Son garantías distintas y ambas requieren análisis:

| Garantía | Qué hace | Cuándo falla | Qué queda |
|---|---|---|---|
| **Cancelación operativa** | Interrumpe la ejecución activa: aborta LLM, cancela pasos pendientes, libera recursos | Proveedor no permite abortar HTTP, worker no recibe la orden, conexión caída | El resultado puede seguir generándose |
| **Invalidación del resultado** | Rechaza el resultado aunque el proveedor lo haya completado | No existe si no se valida vigencia antes de despachar | Resultado inválido podría enviarse al contacto |

**Conclusión:** No basta con cancelar operativamente. El sistema necesita invalidación obligatoria en G3/G4 incluso si la cancelación operativa falló.

### 2.5 Escenario ABA sin `assignmentVersion`

```
1. AgentInstance A asignado, comienza ejecución E1 por mensaje M1 (EvAssigned que autorizó: E1)
2. Humano toma contacto → assignedType = human (EvAssigned: E2)
3. Se solicita cancelar E1 de A
4. Humano se desasigna → assignedType = unassigned (EvAssigned: E3)
5. Llega mensaje M2
6. AgentInstance A asignado otra vez → assignedType = agent, ref_agent_assigned = A (EvAssigned: E4)
7. E1 termina después de la nueva asignación
```

**Mecanismos existentes para diferenciar E1 de E2:**

| Mecanismo | Archivo | ¿Resuelve ABA? |
|---|---|---|
| `executionId` dedup Redis | `executionCompleted.consumer.ts` D5 | Parcial: evita duplicado del MISMO executionId, pero E1 y E2 tienen executionIds distintos |
| `botId` vigencia G3 | `aiVigencia.repo.ts` | Parcial: valida si el bot sigue activo y asignado, pero si A sigue siendo el asignado en el momento de G3, pasa |
| `originMessageId` | No encontrado en CRM | No implementado aún; **no es suficiente por sí solo** para resolver ABA |
| `EvAssigned` (historial) | Por diseñar en CRM | **Sí.** La identidad única de cada EvAssigned identifica qué transición autorizó cada ejecución. |

**Resolución de ABA con identidad de EvAssigned (decisión aprobada):**

- Cada transición confirmada de asignación genera un `EvAssigned` con identificador único (E1, E2, E3, E4 en el ejemplo).
- La ejecución originada por M1 conserva la referencia al EvAssigned que la autorizó (E1).
- Aunque Agent A vuelva a ser responsable mediante E4, la ejecución antigua de M1 **no recupera autorización**: su EvAssigned autorizante (E1) ya fue invalidada por las transiciones posteriores (E2, E3).
- `originMessageId` sirve para trazabilidad e idempotencia del mensaje de origen; **no sustituye** la identidad de la transición EvAssigned.

**Validación en G3/G4 contra autoridad de CRM:**

Antes de despachar el resultado de una ejecución, el consumidor debe verificar contra CRM (autoridad de vigencia):
1. Contacto con `assignedType = agent`.
2. `ref_agent_assigned` coincide con la AgentInstance ejecutora.
3. La identidad de EvAssigned vigente coincide con la que autorizó la ejecución.
4. Contacto e instancia pertenecen al mismo Workspace.
5. Instancia sigue en estado que permite ejecutar.

Si no se puede confirmar la vigencia, no se despacha el resultado.

**Impacto:** Requiere cambios en ai-service (grabar el EvAssigned autorizante como metadata de la ejecución) y en `executionCompleted.consumer.ts` (validar vigencia contra CRM antes de `dispatchOutgoing`). La identidad de EvAssigned puede propagarse como metadata operativa a ai-service sin sincronizar el historial funcional.

---

## 3. Parte B — Persistencia de EvAssigned en CRM

### 3.1 Sistema de eventos actual comprobado

| Archivo | Elemento | Observación |
|---|---|---|
| `src/models/events/Evento.model.ts` | Clase base `@InterfaceType` | Colección `eventos`, discriminators por `__t`, índices `{ contacto:1, __t:1, createdAt:-1 }` y `{ contacto:1, createdAt:-1 }` |
| `src/models/events/EvStartConvers.model.ts` | Discriminator | `openByUser` (UOW), `openByContact` (Contacto) |
| `src/models/events/EvEndConvers.model.ts` | Discriminator | `closedBy` (UOW), `category`, `note` |
| `src/models/events/EvMensaje.model.ts` | Discriminator | Mensaje de chat con `my`, `wamid`, `status`, `type`, media, etc. |
| `src/graphql/resolvers/EvStartConvers.resolver.ts` | Resolver | `openByUser` field resolver |
| `src/graphql/resolvers/EvMensaje.resolver.ts` | Resolver | `edited`, `contacto` field resolvers |
| `src/graphql/resolvers/EvEndConvers.resolver.ts` | Resolver | `closedBy` field resolver |
| `src/repositories/mongo/Evento.repository.ts` | Repository | `findInChatContact`, `findInAnyChat` |

### 3.2 Diseño propuesto de EvAssigned (solo diseño, sin crear archivos)

**Campos mínimos necesarios:**

| Campo | Tipo | Finalidad |
|---|---|---|
| `contacto` | `Ref<Contacto>` | Contacto afectado |
| `prevAssignedType` | `AssignedType \| null` | Tipo de responsable anterior (`human` / `agent` / `unassigned`). Siempre explícito cuando el estado previo es conocido (`unassigned` en una primera asignación). |
| `prevAssignedRef` | `string \| null` | UOW o AgentInstance anterior |
| `newAssignedType` | `AssignedType` | Tipo de nuevo responsable (`human` / `agent` / `unassigned`) |
| `newAssignedRef` | `string \| null` | UOW o AgentInstance nuevo |
| `actorRef` | `string \| null` | Quién produjo el cambio (UOW, AgentInstance o sistema) |
| `actorType` | `"user" \| "agent" \| "system"` | Tipo del actor |
| `operationType` | `string` (propuesta) | **Propuesta pendiente de aprobación.** Catálogo cerrado separado de `AssignedType`. Valores candidatos: `automatic_assignment`, `self_assignment`, `manual_assignment`, `takeover`, `delegation`, `unassignment`. No confundir con `AssignedType`. |
| `reason` | `string \| null` | Justificación de la transición |
| `originMessageId` | `string \| null` | Mensaje entrante que originó la evaluación. Solo trazabilidad/idempotencia; no sustituye la identidad de EvAssigned. |
| `delegationChainId` | `string \| null` | Identificador de cadena de delegación para detectar ciclos |
| `assignedAt` | `Date` | Momento de confirmación de la transición |

**Índices propuestos:**
- `{ contacto: 1, assignedAt: -1 }` — historial por contacto
- `{ contacto: 1, __t: 1, assignedAt: -1 }` — junto con otros eventos del contacto
- `{ delegationChainId: 1, assignedAt: 1 }` — reconstruir cadenas

### 3.3 Transición y persistencia

**Flujo recomendado por transición confirmada:**

1. Validar invariantes del nuevo estado (human/agent/unassigned)
2. Ejecutar `updateOne` en `Contacto` con filtro CAS (estado previo completo)
3. Si `modifiedCount > 0`, crear `EvAssigned` con prev/new/actor/operación/motivo
4. Encolar Outbox `contact.assignment.sync` para propagar estado operativo a ai-service
5. Publicar realtime (`CHANGE_CONTACTO_FROM_CHAT` con `change_origin`)

**Atomicidad y persistencia:**
- Requisito funcional: los cambios de Contacto, `EvAssigned` y Outbox deben preservar una única transición lógica consistente. No deben quedar estados parciales.
- Solución técnica preferida: actualizar Contacto, persistir `EvAssigned` y registrar Outbox dentro de una misma transacción MongoDB.
- Validación pendiente: todavía no se verificó la compatibilidad de esa solución con la infraestructura real de MongoDB Atlas, incluyendo soporte de transacciones, sesiones y compatibilidad de las colecciones involucradas. No presentar la transacción como capacidad ya verificada ni como implementación existente.
- Alternativa recuperable: si la transacción no resulta viable, deberá diseñarse y aprobarse una alternativa recuperable que garantice consistencia, idempotencia y reconciliación. Trabajo futuro sujeto a diseño y aprobación; no se diseña en esta fase.

**Regla:** Una transferencia fallida no genera `EvAssigned`. Solo transiciones confirmadas.

### 3.4 Detección de ciclos de delegación

`delegationChainId` permite agrupar eventos de una misma cadena. Para detectar A → B → C → A:
- Consultar `EvAssigned` del contacto ordenado por `assignedAt`
- Si el `newAssignedRef` de un evento coincide con un `prevAssignedRef` de un evento anterior en la misma cadena, hay ciclo
- Ante ciclo: contacto pasa a `unassigned`, termina la cadena, no se activa recepción automática por el mismo evento

**Distinciones necesarias:**
- Cadena de delegación: múltiples `EvAssigned` con mismo `delegationChainId`
- Nuevo mensaje entrante: genera nueva evaluación, puede o no generar `EvAssigned`
- Reintento de mismo mensaje: mismo `originMessageId`, no debe duplicar `EvAssigned`
- Evento duplicado: mismo `delegationChainId` + `assignedAt`, idempotente
- Transferencia fallida: no se crea `EvAssigned`

### 3.5 Catálogo propuesto de `operationType` (pendiente de aprobación)

**Enum cerrado propuesto**, separado y distinto de `AssignedType`:

| Valor | Significado |
|---|---|
| `automatic_assignment` | Asignación automática por recepción de mensaje entrante |
| `self_assignment` | La propia AgentInstance se asigna un contacto sin responsable |
| `manual_assignment` | Asignación realizada por un usuario humano |
| `takeover` | Un humano toma el control de una conversación atendida por IA |
| `delegation` | Transferencia de un responsable a otro (humano → humano, agente → agente, agente → humano) |
| `unassignment` | Desasignación del contacto (queda sin responsable) |

**Estado:** Propuesta técnica. No está aprobada por el usuario. Queda pendiente de revisión y aprobación antes de implementar.

**Justificación de variantes:** Se evalúa si se necesita alguna variante adicional (por ejemplo, `reactivation` para volver a asignar tras desasignación, o `system` para asignaciones automáticas del sistema). Actualmente el catálogo mínimo cubre los seis escenarios principales. Cualquier variante adicional debe justificarse y aprobarse por separado.

`operationType` describe **cómo** se produjo la transición; `AssignedType` (`prevAssignedType` / `newAssignedType`) describe **qué tipo** de responsable intervino. Son enums independientes y no deben confundirse.

---

## 4. Parte C — Orden de desarrollo y migración

### 4.1 Orden aprobado

| Paso | Acción | Dependencias |
|---|---|---|
| 1 | Incorporar campos opcionales `assignedType` y `ref_agent_assigned` en `Contacto` | Ninguna |
| 2 | Adaptar lecturas y escrituras manteniendo compatibilidad con el modelo anterior | Paso 1 |
| 3 | Desplegar código compatible en CRM, ai-service y workers | Paso 2 |
| 4 | Ejecutar backfill idempotente: `ref_user_assigned` UOW → `human`, null → `unassigned`, `ref_agent_assigned = null` para todos los contactos existentes | Paso 3 |
| 5 | Verificar invariantes post-backfill | Paso 4 |
| 6 | Activar routing de AgentInstance basado en el nuevo modelo | Paso 5 |
| 7 | Retirar compatibilidad antigua solo cuando ningún consumidor dependa del modelo anterior | Paso 6 |

**Mapeo aprobado:**
- UOW presente → `human`
- UOW nulo → `unassigned`
- `ref_agent_assigned` nulo para todos los contactos existentes

**Nota:** El usuario confirmó que no hubo asignaciones productivas anteriores de bots (solo una prueba posteriormente eliminada). No ejecutar la migración en esta fase.

### 4.2 Escrituras concurrentes durante backfill

**Riesgo:** Un contacto asignado a humano durante el backfill podría quedar con `ref_user_assigned` seteado pero `assignedType` en valor por defecto (null o no seteado).

**Mitigación:**
- El backfill debe ser idempotente: leer `ref_user_assigned`, si no es null y `assignedType` no es `human`, setear `assignedType = human`
- Usar `updateOne` con filtro CAS: `{ _id, ref_user_assigned: <valor>, assignedType: { $ne: "human" } }` → `{ $set: { assignedType: "human" } }`
- Si el contacto cambió entre la lectura y el write, el CAS falla y no se pisa

### 4.3 Comandos de verificación, métricas y rollback

**Verificación de integridad:**
```javascript
// Contactos con assignedType inconsistente
db.contacto.countDocuments({
  $or: [
    { assignedType: "human", ref_user_assigned: null },
    { assignedType: "human", ref_agent_assigned: { $ne: null } },
    { assignedType: "agent", ref_user_assigned: { $ne: null } },
    { assignedType: "agent", ref_agent_assigned: null },
    { assignedType: "unassigned", ref_user_assigned: { $ne: null } },
    { assignedType: "unassigned", ref_agent_assigned: { $ne: null } },
    { assignedType: null, ref_user_assigned: { $ne: null } },
    { assignedType: null, ref_agent_assigned: { $ne: null } }
  ]
})
```

**Métricas de progreso:** Contactos migrados, contactos con invariantes violadas, eventos EvAssigned creados, operaciones Outbox encoladas.

**Rollback:**
- Antes de activar asignaciones de AgentInstance: revertir el despliegue del código nuevo (los campos `assignedType`/`ref_agent_assigned` son nuevos; su eliminación no afecta `ref_user_assigned` existente).
- Después de habilitar asignaciones de AgentInstance: un rollback no puede volver simplemente a código que solo entiende `ref_user_assigned`. Se requiere un plan de reversión que mantenga consistencia de los contactos ya asignados a agente (documentar ventana y criterios de rollback, pendiente de detalle operativo).

---

## 5. Parte D — Ownership de AgentInstance en CRM

### 5.1 Decisión aprobada (referencia tipada + propiedad en CRM)

- `AgentInstance` se persistirá en CRM. **Decisión aprobada.**
- `Contacto.ref_agent_assigned` será una referencia tipada a `AgentInstance._id`. **Decisión aprobada.**
- Ownership: `AgentInstance → Workspace → Organización`. **Decisión aprobada.**
- La asignación solamente se permite cuando contacto e instancia pertenecen al mismo Workspace. **Decisión aprobada.**

**Modelo completo de AgentInstance:** Sus campos, índices y ciclo de vida detallado corresponden a **WAB-79**. No diseñar ni implementar ahora toda esa entidad.

**Impacto en servicios y resolvers (solo dirección de arquitectura):**
- `Contacto.service.ts`: validar ownership workspace al asignar a agent.
- `Contacto.resolver.ts`: field resolver para `ref_agent_assigned`.
- `BotRouting.policy.ts`: posible lectura de `assignedType = agent` desde CRM para routing.
- Frontend: adaptar para mostrar/popular `ref_agent_assigned`.

**Inferencia:** ai-service mantiene sus propios registros operativos de AgentInstance para ejecución, pero no es propietario de la entidad. Recibe la información necesaria vía Outbox o consulta directa.

---

## 6. Parte E — Evaluar contactAssignmentHistory

### 6.1 Estado actual comprobado

| Archivo | Línea | Elemento | Observación |
|---|---|---|---|
| `src/graphql/resolvers/AiContactAssignment.resolver.ts` | 20 | `contactAssignmentHistory` query | Consulta ai-service vía `aiContactAssignmentService.getHistory(contactId)` |
| `src/services/AiContactAssignment.service.ts` | 23-29 | `getHistory` | Delegado a `contactAssignmentClient.getHistory(contactId)` |
| `src/infrastructure/ai-service/clients/contact-assignment.client.ts` | 41-46 | `getHistory` | `GET /internal/contacts/${contactId}/assignment/history` en ai-service |
| `src/graphql/types/Objects/AiContactAssignment.object.ts` | 1-17 | Tipo GraphQL | Incluye `id`, `contactId`, `workspaceId`, `assigneeType`, `assigneeId`, `assignedByType`, `assignedById`, `assignedAt`, `releasedAt`, `status`, `reason`, `createdAt` |

**Hecho comprobado:** `contactAssignmentHistory` es una consulta de solo lectura que obtiene el historial de asignaciones desde ai-service. No interviene en reglas de ejecución (los gates G1-G4 leen `contact_assignments` estado activo, no el historial).

### 6.2 ¿Puede EvAssigned reemplazar su finalidad de auditoría?

**Sí, para auditoría funcional.** `EvAssigned` en CRM capturaría todas las transiciones confirmadas con actor, prev/new, operación, motivo, fecha. Esto cubre quién tomó el control, quién delegó, responsable anterior y nuevo.

**Pero `contact_assignments` en ai-service sigue siendo necesario** para el runtime de ejecución (gates G1-G4 leen el estado activo, no el historial).

**No eliminar ni modificar `contactAssignmentHistory` durante esta fase.** Cualquier retirada futura debe demostrar que no rompe dependencias (resolver GraphQL, consumidores externos potenciales).

---

## 7. Parte F — Pruebas y calidad

### 7.1 Tests existentes relacionados

| Test | Archivo | Qué cubre |
|---|---|---|
| Autoasignación | `src/services/__tests__/sendMessage.assign.test.ts` | `wasUnassigned`, BR-AI-015 HTTP falla/Outbox, self-heal |
| Routing policy | `src/services/BotRouting/__tests__/BotRouting.policy.test.ts` | Humano bloquea bot, bot asignado/inactivo, workspace_default |
| Assignment sync | `src/infrastructure/ai-service/__tests__/assignmentSync.test.ts` | Pre/Post CRM sync, Outbox idempotente, CAS, supersede, identidad BR-AI-014 |
| Execution completed consumer | `src/Rabbit/consumers/__tests__/executionCompleted.consumer.test.ts` | G3 vigencia, dedup executionId, revalidación diferida, D5 |
| Ai-service message pipeline | `src/modules/message-pipeline/__tests__/ai-service.test.ts` | No publica a ai-service cuando `ref_user_assigned` en CRM |
| Outbox retry | `src/Workers/__tests__/outboxRetry.worker.test.ts` | Reintentos `contact.assignment.sync` |

### 7.2 Brechas de cobertura (sin generar tests)

1. **Cancelación de ejecuciones:** No existe test de cancelación agent → human, agent A → agent B, agent → unassigned.
2. **Cancelación de LLM en curso:** No existe test de aborto de llamada a proveedor.
3. **Resultado tardío después de takeover:** No existe test de invalidación de resultado en G3/G4.
4. **Escenario ABA:** No existe test de ejecución antigua descartada tras reasignación.
5. **EvAssigned:** No existe (no se ha creado aún).
6. **Asignación a agent:** `assignmentSync.test.ts` solo cubre `assigneeType: "user"`, no `"agent"`.
7. **Transiciones atómicas:** No existe test de CAS sobre `assignedType` + `ref_agent_assigned` combinados.
8. **Backfill idempotente:** No existe test de migración con escrituras concurrentes.
9. **Eventos duplicados de asignación:** No existe test de idempotencia para `EvAssigned`.
10. **Frontend:** No hay tests automatizados de UI para asignación (según AGENTS.md del front, pruebas funcionales manuales).

### 7.3 Pruebas de regresión necesarias (propuesta, sin generar)

- Cancelación agent → human: ejecuciones activas del agente se cancelan, resultados tardíos rechazados
- Transferencia agent A → agent B: ejecuciones de A se cancelan, B puede ejecutar
- Desasignación agent → unassigned: ejecuciones del agente se cancelan
- Cancelación de LLM en curso: aborto de llamada HTTP al proveedor
- Provider que no puede abortarse: resultado invalidado en G3/G4 aunque proveedor completó
- Resultado tardío después de takeover: respuesta no se despacha
- Escenario ABA: ejecución de M1 no envía respuesta aunque A vuelva a ser asignado para M2
- Fallo entre actualización de Contacto y persistencia de EvAssigned: reconciliación o compensación
- Fallo de sincronización CRM → ai-service: Outbox PENDING, recuperación posterior
- Backfill idempotente: escrituras concurrentes no producen estados inconsistentes
- Eventos y mensajes duplicados: `EvAssigned` no se duplica ante reintentos

---

## 8. Conclusiones y recomendación de arquitectura

### 8.1 Viabilidad

| Elemento | Viable | Condición |
|---|---|---|
| CRM como autoridad única | Sí | `Contacto` con `assignedType` + `ref_agent_assigned` |
| Cancelación de ejecuciones | Sí, con cambios en ai-service + G3 | Requiere mecanismo de cancelación en ai-service y validación de vigencia en G3 contra CRM |
| EvAssigned en CRM | Sí | Reutilizar discriminators `Evento`, colección `eventos` |
| `contactAssignmentHistory` conservado | Sí | No eliminar en esta fase |
| Orden desarrollo → migración | Sí | Campos opcionales primero, backfill después |
| AgentInstance en CRM | Sí | Nueva entidad con ownership Workspace → Organización |

### 8.2 Recomendación final (solo elementos todavía pendientes de implementar)

Los siguientes elementos son direcciones de diseño aprobadas que requieren implementación futura:

1. **`assignedType` y `ref_agent_assigned`** en `Contacto` con validación de invariantes (modelo unificado aprobado).
2. **`EvAssigned`** como discriminator de `Evento` en colección `eventos`, exclusivamente en CRM (historial funcional aprobado).
3. **Mantener `contact_assignments` en ai-service** para runtime G1-G4; `EvAssigned` es para auditoría funcional en CRM (aprobado).
4. **No introducir `assignmentVersion`** (decisión ya acordada).
5. **Resolver ABA** con la identidad del `EvAssigned` que autorizó la ejecución + validación de vigencia en G3/G4 contra CRM (fuente de verdad). `originMessageId` no es suficiente por sí solo.
6. **Transacción Contacto + EvAssigned + Outbox** como solución preferida, pendiente de verificar compatibilidad de infraestructura.
7. **Orden de desarrollo:** campos opcionales → compatibilidad → despliegue → backfill idempotente → activar routing agent → retirar compatibilidad antigua (orden aprobado).

### 8.3 Clasificación de decisiones (estado actual)

No conservar la antigua sección 8.3 como si todas las decisiones siguieran abiertas. Clasificación correcta:

| Tema | Estado correcto |
|---|---|
| Nomenclatura y valores de `AssignedType` | **Aprobados** |
| Referencia `ref_agent_assigned → AgentInstance` en CRM | **Aprobada** |
| Transacción Contacto + EvAssigned + Outbox | **Solución preferida; verificar infraestructura** |
| G3/G4 contra CRM | **Estrategia aprobada; cerrar contrato técnico** |
| `assigneeType: "agent"` vía Outbox | **Dirección de diseño aprobada; detallar payload y procesamiento** |
| Modelo completo `AgentInstance` | **Diferido a WAB-79** |
| Orden código → despliegue → backfill | **Aprobado** |
| Ventana y rollback del backfill | **Pendiente de detalle operativo** |
| `operationType` de `EvAssigned` | **Pendiente de propuesta y aprobación** (catálogo en §3.5) |
| Cancelación efectiva en ai-service | **Requiere diseño/validación en el servicio externo** |

**Nota:** No presentar una recomendación técnica como decisión aprobada por el usuario. Los temas marcados como "pendiente" o "requiere diseño" siguen abiertos.

---

## 9. Hechos comprobados vs inferencias vs recomendaciones

### Hechos comprobados en código (evidencia directa)

1. `Contacto.model.ts` línea 156-157: `ref_user_assigned` es `Ref<UsOrgsWorks>` (UOW, no ID directo de usuario)
2. `assignmentSync.service.ts` líneas 43-45: `AssignmentTarget` solo soporta `assigneeType: "user"`
3. `executionCompleted.consumer.ts` líneas 279-303: dedup por `executionId` con `SET NX EX 86400`
4. `aiVigencia.repo.ts` líneas 94-115: G3 lee `contact_assignments` de ai-service para validar vigencia
5. `BotRouting.policy.ts` líneas 66-75: routing lee `contactAssignmentClient.getActive` (ai-service, no CRM)
6. `Evento.model.ts`: sistema de discriminators con `__t`, colección `eventos`
7. `AiContactAssignment.resolver.ts` línea 20: `contactAssignmentHistory` query existe y consulta ai-service
8. `queues.enum.ts`: `AI_EXECUTION_COMPLETED` y `AI_EXECUTION_COMPLETED_VIGENCIA_RETRY` existen

### Cobertura verificada por tests

1. `sendMessage.assign.test.ts`: autoasignación, BR-AI-015, Outbox self-heal
2. `BotRouting.policy.test.ts`: humano bloquea bot, bot asignado/inactivo, workspace_default
3. `assignmentSync.test.ts`: pre/post CRM sync, Outbox idempotente, CAS, supersede
4. `executionCompleted.consumer.test.ts`: G3 vigencia, dedup executionId, revalidación
5. `ai-service.test.ts`: no publica a ai-service cuando `ref_user_assigned` en CRM

### Inferencias del análisis original (no verificadas directamente en esa sesión)

1. El ExecutionEngine vive en ai-service (referencia del análisis original a documentación de routing; no se revalidó aquí).
2. No existe cancelación por `executionId` por contacto en el CRM visible
3. No existe registro de ejecuciones activas indexado por contacto
4. `AgentInstance` no existe como entidad en CRM (no encontrada en código)
5. Los adaptadores LLM de ai-service pueden o no soportar `AbortController`

### Recomendaciones

1. Implementar `assignedType` + `ref_agent_assigned` con invariantes
2. Crear `EvAssigned` como discriminator de `Evento` exclusivamente en CRM
3. Mantener `contact_assignments` en ai-service para runtime; no sincronizar `EvAssigned` a ai-service
4. Resolver ABA con la identidad del `EvAssigned` que autorizó la ejecución + validación G3/G4 contra CRM. `originMessageId` no es suficiente por sí solo.
5. No introducir `assignmentVersion`
6. Seguir el orden desarrollo → migración propuesto
7. **Limitación de evidencia:** el análisis original no verificó directamente el mecanismo de cancelación ni el soporte AbortController de los providers. No afirmar que estén implementados sin evidencia específica.
8. Verificar en una revisión técnica futura el mecanismo de cancelación de ejecución, soporte AbortController en adaptadores LLM y registro de ejecuciones activas.

---

## 10. Restricciones respetadas

- ✅ No se modificó código
- ✅ No se modificaron schemas ni modelos
- ✅ No se creó `EvAssigned`
- ✅ No se eliminó `contactAssignmentHistory`
- ✅ No se crearon colecciones ni eventos
- ✅ No se modificó RabbitMQ
- ✅ No se implementó cancelación
- ✅ No se ejecutaron migraciones
- ✅ No se generaron ni modificaron tests
- ✅ No se modificaron permisos ni roles
- ✅ Se actualiza documento canónico y Linear como parte de esta tarea de cierre documental
- ✅ No se cerró WAB-78
- ✅ No se inició WAB-79
- ✅ No se introdujo `assignmentVersion` en Contacto
- ✅ No se sincronizó historial de EvAssigned con ai-service
- ✅ No se reabrieron decisiones P1, P2 ni P4

---

**Siguiente paso:** Revisión del usuario del complemento técnico corregido, del documento canónico sincronizado y de los cambios en Linear.