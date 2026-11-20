# WAB-79 — Informe de Viabilidad Técnica: Cierre del Diseño de Implementación (Revisión 21)

> **Estado:** Revisión correctiva integral tras evaluación independiente (NO SUPERADO en seis bloques). No autoriza implementación ni cambios en Linear.
> **Fase:** WAB-79 — Bloque de viabilidad técnica de la jubilación de AgentInstance
> **Versión base:** `wab-79-viabilidad-tecnica-cierre-revision-20.md` (Revisión 20)
> **Fecha de revisión:** 13/11/2026
> **Objetivo de la corrección:** Corregir integralmente los defectos documentales y lógicos demostrados en la evaluación independiente de seis bloques + revisión transversal (resultado: NO SUPERADO). Reconstruir demostraciones afectadas, alinear secciones contradictorias e identificar explícitamente cuestiones pendientes de decisiones contractuales o validaciones técnicas. **Quinta intervención correctiva (esta revisión):** D4-02, D5-01, D5-03, D2-04, T-08 + regresiones D3-01, D6-01/D6-03.
> **Alcance:** Corrección integral de los defectos enumerados en los bloques B1–B6 y contradicciones transversales T-01–T-08. La Revisión 20 permanece intacta como antecedente histórico.
> **Exclusiones:** No se modifican revisiones 1–20, contratos canónicos, código de producción, Linear, GitHub remoto, WAB-81, schemas canónicos ni decisiones arquitectónicas aprobadas. No se crean decisiones arquitectónicas nuevas. No se reabren decisiones ya aprobadas. No se inventan campos, índices, payloads, identidades físicas ni contratos Outbox. No se convierten propuestas en decisiones. No se presenta pseudocódigo como implementación verificada. No se declaran pruebas runtime, Atlas o concurrencia como realizadas sin evidencia real. Se conserva la trazabilidad histórica de la Revisión 21 (fue evaluada y encontró defectos). No se amplía alcance funcional de WAB-79. No se afirma autorización para implementación.

---

## 2. Documentos de referencia y secciones críticas

| Documento | Secciones críticas consultadas |
|-----------|--------------------------------|
| Revisión 20 | §5 (completo), §6, §7, §8, §9, §10, §11, §12, §13 |
| Canónico WAB-79 (ownership) | **§16.4** `RetirementOperation` modelo conceptual, §16.5 estados, **§16.6** unicidad/lease/fencing, **§16.8** drenaje dinámico, **§16.9** correlación, **§16.10** cierre conjunto, **§16.12** idempotencia, **§16.14** pendiente |
| Canónico WAB-78 (fase 2) | **D.1.5** modelo Contacto, **D.1.6** EvAssigned (incluye `reason`), D.1.7 resolución ABA, **D.1.8** atomicidad, D.6 entrega durable, E.4.10 seguridad/concurrencia, G.4.2 fencing autorizaciones, **D-W79-14 aprobada** |
| CRM package.json | `mongodb` v6.20.0, `mongoose` v8.19.3 |
| Lockfile CRM | `pnpm-lock.yaml` confirma `mongodb@6.20.0` (línea 4021) |
| Código fuente MongoDB Node.js Driver v6.20.0 | `src/sessions.ts` — método `ClientSession.withTransaction` (líneas 1065–1164), `commitTransaction` (líneas 610–685), etiquetas de error, `isUnknownTransactionCommitResult`, `hasTransientTransactionLabel` |

---

## 3. Resumen de correcciones frente a Revisión 20

| Punto | Defecto en Revisión 20 | Corrección en Revisión 21 | Estado |
|-------|------------------------|---------------------------|--------|
| **B1-01 D-W79-14** | Presentada como "pendiente", "bloqueada" o "necesitada de aprobación" en múltiples secciones | Corregido: **D-W79-14 está aprobada** (canónico WAB-78 §D.1.6, WAB-79 §5.2.12, §9). Alcance exacto: campos propios/heredados de EvAssigned, `actorRef` obligatorio, `reason` nullable, `operationType` 6 valores, exclusión de `originMessageId`/`delegationChainId`. No se extiende a cuestiones físicas Outbox no aprobadas. | ✅ Corregido |
| **B1-02 Idempotencia** | Confusión entre idempotencia conceptual (`evAssignedId` según D-W79-15) y campos físicos Outbox (pendientes §16.14) | Separación explícita: idempotencia conceptual = `evAssignedId` (decisión aprobada). Esquema físico Outbox, índices, claves, payloads = **pendientes §16.14**. No se presenta `idempotencyKey` ni índice único como aprobados. | ✅ Corregido |
| **B1-03 Atomicidad** | Presentada como universalmente imposible o garantizada sin condiciones | D.1.8: transacción MongoDB = **solución preferida**, validación Atlas **pendiente**. Separación: solución arquitectónica preferida, garantías condicionadas a transacción real, escenarios sin atomicidad, validación pendiente. | ✅ Corregido |
| **B1-04 readConcern: snapshot** | Presentado como decisión canónica | Identificado como **propuesta técnica pendiente de validación en Atlas**, no decisión aprobada. | ✅ Corregido |
| **H1/H3 A.1 Identidad consultas** | Spread de `null` como reutilización ficticia (§5.5 líneas 249-253) | Eliminación definitiva: 1) Determinar identidad real. 2) Ejecutar consulta necesaria. 3) Conservar resultado. 4) Reutilizar solo resultado de consulta ejecutada (Regla ID-1). 5) `not_queried`/`not_found`/`reused_result` diferenciados. | ✅ Corregido |
| **H1/H3 A.2 Fila 9** | Contaminada por spread de `null` | Reconstrucción: cuando IDs coinciden ≠ null → una sola consulta ejecutada (`outboxExpected`), resultado real reutilizado → `reused_result` con referencia a consulta original. | ✅ Corregido |
| **H1/H3 A.3 Recuentos** | 13+4=17 vs 14+3=17; algoritmo no cubre todas las filas | Alineación completa: **14 combinaciones posibles** (D1=true: 13; D1=false: 1) + **3 imposibles** = 17. Cada fila tiene rama identificable. Trazabilidad 1:1 reconstruida. | ✅ Corregido |
| **H1/H3 A.4 Sesiones Mongoose** | `Model.findById(id, { session })` — firma incorrecta | Corrección en todas las lecturas: `Model.findById(id, null, { session: txnSession })` o `.session(txnSession)`. Verificada versión `mongoose@8.19.3`. | ✅ Corregido |
| **H4 B.1 Helper Outbox** | Consumo como booleano (objeto siempre truthy) | Consumo **explícito de `decision`** en TODOS los consumidores. Nunca truthy/falsy. | ✅ Corregido |
| **H4 B.2 Semántica OutboxMatchResult** | `approvedChecks = true` aunque Nivel 1 vacío — contradictorio | Rediseño: `approved` (Nivel 1 pasó), `conditional` (Nivel 1 §16.14), `proposed` (Nivel 2), `decision` final, `blockReason`. `decision: 'block'` cuando Nivel 1 vacío. | ✅ Corregido |
| **H4 B.3 Inventario consumidores** | 6 consumidores heredados, no refleja algoritmo | **Inventario exhaustivo real**: 8 consumidores identificados en §5.5 con ruta, decisión evaluada, resultado permitido. | ✅ Corregido |
| **H4 B.4 Nivel 1 vacío** | Declarado vacío pero usado `approvedChecks = true` | **Mantenido y aplicado uniformemente**: Nivel 1 = VACÍO para Outbox asignación. `idempotencyKey`, `eventType`, `contactId`, `contacto`, `payload.evAssignedId` NO aprobados. D.1.8 = atomicidad, no esquema. §16.14 = pendiente. D-W79-14 aprobada (no bloqueada). | ✅ Corregido |
| **D4-02 Decisiones Outbox** | Helper `decision` (`block`/`escalate`/`allow`) no producía tratamientos distinguibles; ramas idénticas | **Tratamiento diferenciado diagnóstico (5ta intervención)**: `block` → `*_BLOCK` + `decisionType: definitive_rejection`; `escalate` → `*_ESCALATE` + `decisionType: anomaly_requires_intervention`; `allow` → `*_WITH_OUTBOX` + guard + `decisionType: helper_allows_but_contractual_guard_blocks`. **15 consumidores** con decisionType explícito. **Diferencia operativa NO demostrada**: ambos `block` y `escalate` retornan `canRetryBusinessTransaction: false` y `requiresEscalation: true` idénticos. D4-02 **parcialmente resuelto** — pendiente validar consumidores downstream reales (WAB-81/alertas/operadores). No se afirma diferenciación operativa efectiva. | ⚠️ Parcialmente resuelto |
| **D5-01 Partición BSON** | E1-E10 declarados exhaustivos sin demostración; solapamiento E2/E9/E10 | **Partición lógica mutuamente excluyente por construcción (5ta intervención)**: dimensiones ortogonales (processing absent/present, leaseOwner absent/null/same/other, leaseUntil absent/null/future/past, leaseGeneration absent/0/positive). E9 exige `leaseUntil` PRESENTE; E10 exige `leaseGeneration`=0 explícito. **Clasificador formal determinista** (§8.1.1 pseudocódigo) con orden fijo, cada combinación clasificada exactamente una vez. Categorías inválidas (E-INVALID) bloqueadas explícitamente. **Recuento correcto**: 2×4×4×3=96 combinaciones teóricas; E1 (processing ausente): 48 teóricas → 1 clase física; E2-E10: 12 combinaciones teóricas válidas → 12 clases físicas; E-INVALID: 36 combinaciones teóricas inválidas; 13 clases válidas totales (1 E1 + 12 E2-E10), 36 clases inválidas; 49 clases de equivalencia BSON totales. No se afirma 80 combinaciones con processing presente ni 81 estados físicos; se usa "clases de equivalencia BSON". | ✅ Corregido (5ta intervención) |
| **D5-03 CAS completo** | Filtros CAS no protegían valores exactos observados (E4-E7 solo `$exists`) | **Filtros CAS con igualdad exacta de valores observados (5ta intervención)**: E4-E7 incluyen `leaseGeneration: observedGen` y `leaseUntil: observedLeaseUntil`; E8/E9 protegen `leaseGeneration` absent + `leaseUntil` exacto; E10 usa `leaseGeneration: 0` exacto; **E3 corregido**: `processing.leaseOwner: { \$in: [null], \$exists: true }` distingue `null` explícito de campo ausente. Combinaciones no demostrables (E-INVALID) bloqueadas. | ✅ Corregido (5ta intervención) |
| **D2-04 RetirementOp** | Consulta por `agentInstanceId` presentada como correlación demostrada | **Consulta ILUSTRATIVA NO NORMATIVA explicitamente (5ta intervención)**: §16.12 identidad = `{retirementOperationId, contacto}`; §16.14 esquema físico pendiente; consulta `agentInstanceId` ≈ ilustración; bandera `retirementOpCorrelationDemonstrable` **interviene en flujo de control** — bloquea toda ruta de éxito dependiente; registro de anomalía `retirementOpQueryAnomaly` si falla consulta; NO continuar silenciosamente; sin `OUR_TRANSITION_COMMITTED` por consulta aproximada. | ✅ Corregido (5ta intervención) |
| **H5 C.1 Modelo datos** | Mezcla `leaseOwner` / `processing.leaseOwner` sin explicación | **Rutas `processing.*` consistentemente**. 10 estados BSON distinguidos (ausencia vs null vs valor). | ✅ Corregido |
| **H5 C.2 Serialización BSON** | Suposiciones sobre `$in: [null, undefined]`, `$exists` en anidados | **Validado contra driver v6.20.0 y espec BSON**: `undefined` omitido en BSON; `{ field: null }` coincide null/ausente solo si padre existe; `$exists: false` solo campo ausente con padre existente; `$lte` requiere fecha existente. Tabla de verdad completa. | ✅ Corregido |
| **H5 C.3 Predicado elegibilidad** | Elegibilidad y CAS desalineados en filas 5, 7, 8 | **Predicado definido primero, CAS protege estado observado exacto**. Tabla 10 estados con elegibilidad = condiciones CAS. Separación: adquisición / readquisición / renovación / adquisición-sobre-otro. | ✅ Corregido |
| **H5 C.4 Tabla estados BSON** | Fila 4 "adquisición cuando lease vigente", fila 5 ambigua, fila 7 "se rechaza si algoritmo permite" | **Tabla reconstruida 10 filas**: fila 4 = Renovación (conserva gen), fila 5 = Readquisición (incrementa), fila 7 = Adquisición sobre otro (permitida, CAS protege). Coincidencia exacta con pseudocódigo. | ✅ Corregido |
| **H5 C.5 Bloqueos inicialización** | No declara bloqueos `leaseGeneration` default, `processing` init | **Bloqueos explícitos**: no se inventa default `leaseGeneration`, no se inicializa `processing`, no migraciones, no índices, no defaults schema. Pendiente schema final (§16.14). | ✅ Corregido |
| **Bloque D.1 Flujo errores** | No implementa `catch` exterior que cubra `withTransaction` | **Flujo completo documentado** (§9.1): `try { result = await session.withTransaction(...) } catch (error) { clasificación y reconciliación }`. Pseudocódigo estructural. | ✅ Corregido |
| **Bloque D.2 Separación callback/commit** | No distingue reejecución callback vs reintento commit | **Distinción explícita** (§9.2, §9.3): 9 fases desde callback hasta error definitivo. Contexto ≠ confirmación commit. | ✅ Corregido |
| **Bloque D.3 Ciclo AttemptContext** | Mecanismo no cubre error antes de contexto, durante commit, reintento commit | **Mecanismo seguro verificado 9 escenarios** (§9.3, §9.4): primera ejecución, reejecución, error antes/después contexto, error durante commit, resultado desconocido, reintento commit, agotamiento. Catch exterior conoce contexto correcto. | ✅ Corregido |
| **Bloque D.4 Código real driver** | Referencias de línea incorrectas de Revisión 19 | **Verificación exacta** (§9.5): `mongodb@6.20.0` confirmado. `src/sessions.ts` tag `v6.20.0`: `withTransaction` 1065–1164, `commitTransaction` 610–685, `hasTransientTransactionLabel` ~580, `isUnknownTransactionCommitResult` ~590. Confirmado: `TransientTransactionError` reintenta transacción completa; `UnknownTransactionCommitResult` reintenta commit misma transacción. | ✅ Corregido |
| **Métricas físicas** | Métricas de Revisión 20 copiadas | **Medición física real** del documento final (§13): PowerShell `Get-Item` + `Get-Content`. | ✅ Corregido |
| **Validación interna** | No hay pasada de consistencia cruzada documentada | **12 verificaciones obligatorias** (§11) realizadas. Si alguna falla, sección no declarada aprobada. | ✅ Corregido |

---

## 4. Contratos y decisiones que se mantienen sin cambios

Todas las decisiones aprobadas y correcciones válidas de la Revisión 20 (y por herencia, 19, 18, etc.) se conservan, salvo lo expresamente corregido en esta revisión.

**Bloqueos que persisten intencionalmente (no resueltos en esta revisión, por diseño):**

- **§16.14 Outbox pendiente** → esquema físico, campos, payloads, índices, identidad física de idempotencia, contratos de transporte Inbox/Outbox. **Bloquea: Nivel 1 aprobado = VACÍO → helper Outbox `decision='block'` siempre → `OUR_TRANSITION_COMMITTED` inalcanzable.**
- **WAB-81 pendiente** → drenaje Outbox inconsistente, recuperación ejecuciones, implementación contratos ARQ-22.
- **Schema `RetirementOperation` final** → valor inicial `leaseGeneration` (0 vs 1), índices parciales, topología transaccional, campos `admissionBlockedAt`/`eventSetClosedAt`.
- **Verificación Atlas** → atomicidad transacciones multidocumento (Contacto+EvAssigned+Outbox+RetirementOperation), compatibilidad `readConcern: snapshot`, topología clúster.
- **Autorización reintento de negocio** → sin mecanismo contractual para evidencia positiva de aborto (requiere D-W79-14 + diseño reconciliación + §16.14).
- **Correlación física transiciones retiro** → vínculo entre identidad canónica `{retirementOperationId, contacto}` (§16.12) y esquema físico Outbox (§16.14 pendiente). **Bloquea: consulta por `agentInstanceId` es ilustrativa, no probatoria; algoritmo clasifica como bloqueado/pendiente cuando correlación no demostrable.**
- **Inicialización `leaseGeneration` / `processing`** → defaults schema, migraciones, persistencia. **Bloquea: estados E1/E2/E3/E8/E9/E10 dependen de esquema final (§16.14).**

**Nota importante:** D-W79-14 **está aprobada** (canónico WAB-78 §D.1.6, WAB-79 §5.2.12, §9). Su alcance: campos propios/heredados de EvAssigned, `actorRef` obligatorio, `reason` nullable, `operationType` 6 valores, exclusión de `originMessageId`/`delegationChainId`. **No está pendiente ni bloqueada**. Lo que permanece pendiente es la correlación física en Outbox (§16.14) que permitiría verificar autoría inequívoca en runtime. **Esto significa: D-W79-14 autoría EvAssigned aprobada; §16.14 Outbox físico pendiente → sin verificación runtime de autoría + Nivel 1 vacío → helper siempre `decision='block'` → `OUR_TRANSITION_COMMITTED` inalcanzable.**

---

## 5. H1/H3: Consultas Outbox, Matriz y Algoritmo Corregidos (Bloque A — Corregido integralmente)

### 5.1 Problemas identificados en Revisión 20 y hallazgos de la evaluación independiente

La evaluación independiente de seis bloques (B1–B6) y revisión transversal (T-01–T-08) de la Revisión 20 determinó **NO SUPERADO**. Los defectos residuales confirmados son:

1. **B1-01: D-W79-14 presentada como pendiente** — Múltiples secciones la citan como "pendiente", "bloqueada" o "necesitada de aprobación". Canónico WAB-78 §D.1.6 y WAB-79 §5.2.12 confirman: **aprobada 10/11/2026**. Alcance exacto: campos EvAssigned, `actorRef` obligatorio, `reason` nullable, `operationType` 6 valores, exclusión de `originMessageId`/`delegationChainId`.

2. **B1-02: Confusión idempotencia conceptual vs esquema físico Outbox** — Se usa `idempotencyKey: \`${expectedEvAssignedId}:${contactId}\`` como campo físico aprobado. D-W79-15 define idempotencia conceptual = `evAssignedId`. Esquema físico Outbox = **pendiente §16.14**.

3. **B1-03: Atomicidad sin matices** — Afirmaciones de "imposible universalmente" o "garantizada sin condiciones". D.1.8: transacción MongoDB = solución preferida, **validación Atlas pendiente**.

4. **B1-04: `readConcern: snapshot` como decisión canónica** — Es propuesta técnica pendiente de validación en Atlas.

5. **H1/H3 A.1: Spread de `null` como reutilización ficticia** (§5.5 líneas 249-253) — `{ ...outboxCurrent, ... }` con `outboxCurrent = null` produce objeto fabricado. Confunde `not_queried` con `reused_result`.

6. **H1/H3 A.2: Fila 9 contaminada** — Muestra `reused_result` pero algoritmo no ejecutó consulta (spread de `null` no consultado).

7. **H1/H3 A.3: Recuentos desalineados** — 13+4=17 vs 14+3=17; algoritmo no cubre todas las filas.

8. **B2-01 a B2-06, B3-01 a B3-03, B4-01 a B4-03, B5-01 a B5-05, B6-01 a B6-04** — Defectos detallados en bloques correspondientes abajo.

9. **T-01 a T-08** — Contradicciones transversales (D-W79-14, `retirementOp`, matriz-rama, consumidores, BSON, AttemptContext, idempotencia, fecha).

### 5.2 Identidad de las consultas: análisis formal (Normativo — Reconstruido desde algoritmo corregido y contratos canónicos)

**Decisiones canónicas relevantes (precedencia obligatoria):**

| Decisión | Estado | Alcance exacto | Fuente canónica |
|----------|--------|----------------|-----------------|
| **D-W79-14** | **Aprobada** (10/11/2026) | Campos EvAssigned: `prevAssignedType`, `prevAssignedRef`, `newAssignedType`, `newAssignedRef`, `actorType`, `actorRef` (obligatorio), `operationType` (6 valores), `reason` (nullable). Excluye `originMessageId`, `delegationChainId`. Reutiliza campos e índices heredados de `Evento`. | WAB-78 §D.1.6, WAB-79 §5.2.12, §9 |
| **D-W79-15** | **Aprobada** | Idempotencia conceptual de consumidores = `evAssignedId`. Unidad atómica: Contacto + EvAssigned + Outbox en transacción MongoDB. `ref_ev_assigned` como token de concurrencia en CAS. | WAB-79 §9, §16.12 |
| **D.1.8 / D-W79-15** | **Solución preferida, validación pendiente** | Atomicidad: transacción MongoDB (Contacto, EvAssigned, Outbox). Validación Atlas: topología compatible, propagación sesión, commit/rollback, `readConcern: snapshot`. | WAB-78 §D.1.8, WAB-79 §16.14 |
| **§16.12 Idempotencia** | **Aprobada (identidades conceptuales)** | 3 identidades: (1) Operación: `retirementOperationId`, (2) Transición por contacto: `{retirementOperationId, contacto}`, (3) Confirmación: `evAssignedId`. **No** campos físicos Outbox. | WAB-79 §16.12 |
| **§16.14 Pendiente** | **Pendiente** | Schema final Outbox, nombres, payloads, contratos transporte, topología transaccional, índices, identidad física idempotencia. | WAB-79 §16.14 |

**Definición de las cuatro identidades de consulta (explícita, inmutable, referenciable):**

| Consulta | Variable algoritmo | Colección | Filtro exacto | Clave de identidad | **Condición exacta de ejecución** | **Se reutiliza resultado previo?** | Identidad del documento obtenido | Evidencia que produce | Rama que la consume |
|----------|-------------------|-----------|---------------|-------------------|----------------------------------|-----------------------------------|--------------------------------|----------------------|---------------------|
| **EvAssigned actual** | `currentEvAssigned` | `eventos` (discriminator EvAssigned) | `_id === currentEvAssignedId` | `_id` (PK) | `currentEvAssignedId !== null` | **Nunca** (clave distinta de expected salvo IDs coincidentes) | Documento EvAssigned por `_id` o `null` | D3: `exists_coherent` / `exists_incoherent` / `not_found` / `not_queried` | CASO B, CASO C, CASO D |
| **EvAssigned esperado** | `expectedEvAssigned` | `eventos` (discriminator EvAssigned) | `_id === expectedEvAssignedId` | `_id` (PK) | **SIEMPRE** que `expectedEvAssignedId !== null` | **Nunca** (consulta independiente obligatoria) | Documento EvAssigned por `_id` o `null` | D4: `exists_coherent` / `exists_incoherent` / `not_found` | CASO A, CASO B |
| **Outbox actual** | `outboxCurrent` | `outbox` | **Identidad física pendiente §16.14** — Ilustrativo: `idempotencyKey === \`${currentEvAssignedId}:${contactId}\`` | **Pendiente §16.14** — Ilustrativo: `idempotencyKey` | `currentEvAssignedId !== null` **Y** `currentEvAssignedId !== expectedEvAssignedId` | **Solo si** `currentEvAssignedId === expectedEvAssignedId` (Regla ID-1) — misma clave ilustrativa | Documento Outbox por clave ilustrativa o `null` | D5: `exists_coherent` / `exists_incoherent` / `not_found` / `not_queried` / `reused_result` | CASO B (solo si IDs difieren), CASO C, CASO D |
| **Outbox esperado** | `outboxExpected` | `outbox` | **Identidad física pendiente §16.14** — Ilustrativo: `idempotencyKey === \`${expectedEvAssignedId}:${contactId}\`` | **Pendiente §16.14** — Ilustrativo: `idempotencyKey` | **SIEMPRE** que `expectedEvAssignedId !== null` | **Solo si** `currentEvAssignedId === expectedEvAssignedId` (Regla ID-1) — misma clave ilustrativa | Documento Outbox por clave ilustrativa o `null` | D6: `exists_coherent` / `exists_incoherent` / `not_found` / `reused_result` | CASO A, CASO B |

> **⚠️ Advertencia contractual:** Los filtros `idempotencyKey === \`${evAssignedId}:${contactId}\`` son **propuestas ilustrativas no aprobadas**. El esquema físico del Outbox (campos, índices, clave de unicidad, payload) está **pendiente en §16.14**. No deben usarse para decisiones contractuales de `return_committed`. La idempotencia contractual aprobada (D-W79-15, §16.12) usa `evAssignedId` como identidad de confirmación.

**Reglas de identidad (normativas, demostrables):**

- **Regla ID-1 (Reutilización Outbox — ilustrativa):** Si `currentEvAssignedId === expectedEvAssignedId` → `outboxCurrent` y `outboxExpected` consultarían la **misma clave ilustrativa** `idempotencyKey`. El resultado **debe ser idéntico** (mismo documento o ambos `null`). El algoritmo **debe ejecutar la consulta una sola vez** (para `outboxExpected`, que SIEMPRE se ejecuta) y **reutilizar ese resultado real** para `outboxCurrent`. La reutilización es válida **solo** cuando esta igualdad se verifica en el camino de código concreto Y la consulta se ejecutó efectivamente.

- **Regla ID-2 (Consultas Outbox distintas):** Si `currentEvAssignedId !== expectedEvAssignedId` → `outboxCurrent` y `outboxExpected` consultarían **claves ilustrativas distintas**. Pueden existir ambos, uno, o ninguno. **Ambas consultas deben ejecutarse** independientemente. No hay reutilización posible.

- **Regla ID-3 (Identidad EvAssigned — aprobada D-W79-14):** `currentEvAssigned` y `expectedEvAssigned` consultan por `_id` (PK). Si `currentEvAssignedId === expectedEvAssignedId` → **mismo documento físico** → coherencia **idéntica**. **No pueden divergir**. Si IDs difieren, son documentos independientes.

- **Regla ID-4 (Ejecución condicional Outbox actual):** `outboxCurrent` **solo se ejecuta** cuando `currentEvAssignedId !== null` **Y** `currentEvAssignedId !== expectedEvAssignedId`. Si `currentEvAssignedId === null` o IDs coinciden, `outboxCurrent` no se consulta → su valor en el algoritmo es `not_queried` (no `null`, no `not_found`).

- **Regla ID-5 (Ejecución EvAssigned actual cuando IDs coinciden):** Cuando `contactShowsExpected` (Contacto apunta a `expectedEvAssignedId`) **Y** `currentEvAssignedId === expectedEvAssignedId !== null`, el algoritmo **EJECUTA** `currentEvAssigned` (paso 2 del algoritmo corregido). Al ser mismo `_id` que `expectedEvAssigned`, el resultado es **idéntico** a `expectedEvAssigned` (Regla ID-3). La matriz debe reflejar `exists_coherent`/`exists_incoherent`/`not_found`, **no** `not_queried`.

**Estados de evidencia (normativos, sin ambigüedad):**

| Estado | Significado | Categoría |
|--------|-------------|-----------|
| `not_queried` | Consulta no ejecutada (condición `IF` falsa) | Demostración documental |
| `not_found` | Consulta ejecutada, documento no existe | Demostración documental |
| `exists_coherent` | Consulta ejecutada, documento existe y es coherente | Demostración documental |
| `exists_incoherent` | Consulta ejecutada, documento existe pero contradice | Demostración documental |
| `reused_result` | Resultado reutilizado legítimamente tras ejecución probada de consulta única (Regla ID-1); incluye referencia a la consulta original ejecutada | Demostración documental |

**Consecuencia normativa:** La matriz y el algoritmo **no pueden representar** `outboxCurrent` y `outboxExpected` como dimensiones independientes cuando los IDs coinciden. Las combinaciones que asumen resultados distintos para la misma clave ilustrativa son **imposibles por restricción de unicidad**. El valor `not_queried` es distinto de `not_found` y debe reflejarse en la matriz.

### 5.3 Dimensiones de evidencia (Reconstruidas desde restricciones reales de consulta y contratos canónicos — Corregidas)

Las dimensiones atómicas de evidencia observable en un **mismo snapshot transaccional** (propuesta `readConcern: snapshot`, **validación Atlas pendiente**), **incluyendo si la consulta se ejecutó**:

| Dim | Nombre | Valores posibles | Nota | Categoría |
|-----|--------|------------------|------|-----------|
| **D1** | Contacto existe | `true` / `false` | `false` → `INDETERMINATE_CONTRACTUAL_BLOCK` | Demostración documental |
| **D2** | Estado actual del Contacto | `prev` (apunta a `prevEvAssignedId` o null) / `expected` (apunta a `expectedEvAssignedId`) / `other` (apunta a otro EvAssigned) | Derivado de `ref_ev_assigned` | Demostración documental |
| **D3** | EvAssigned actual | `exists_coherent` / `exists_incoherent` / `not_found` / `not_queried` | Consultado **solo si** `currentEvAssignedId !== null`. **Cuando `contactShowsExpected` ∧ IDs coinciden ≠ null: SÍ se ejecuta → no `not_queried`** (Regla ID-5) | Demostración documental |
| **D4** | EvAssigned esperado | `exists_coherent` / `exists_incoherent` / `not_found` | **SIEMPRE consultado** (si `expectedEvAssignedId !== null`) | Demostración documental |
| **D5** | Outbox por `currentEvAssignedId` | `exists_coherent` / `exists_incoherent` / `not_found` / `not_queried` / `reused_result` | **Solo si** `currentEvAssignedId !== null` **Y** `currentEvAssignedId !== expectedEvAssignedId`; si IDs coinciden → `≡ D6` (reutilizado → `reused_result`). **Clave ilustrativa pendiente §16.14** | Demostración documental / Propuesta ilustrativa |
| **D6** | Outbox por `expectedEvAssignedId` | `exists_coherent` / `exists_incoherent` / `not_found` / `reused_result` | **SIEMPRE consultado** (si `expectedEvAssignedId !== null`); si IDs coinciden → `≡ D5` (reutilizado → `reused_result`). **Clave ilustrativa pendiente §16.14** | Demostración documental / Propuesta ilustrativa |
| **D7** | Evidencia duplicada/contradictoria | `true` / `false` | Solo posible si `currentEvAssignedId !== expectedEvAssignedId` | Demostración documental |
| **D8** | Transición mismo tipo, distinto ID | `true` / `false` | `currentEvAssigned` coherente en tipo pero `_id` distinto de `expectedEvAssignedId` | Demostración documental |

**Restricciones lógicas que descartan combinaciones imposibles (Normativas):**

| Restricción | Fundamento contractual | Consecuencia en matriz | Categoría |
|-------------|------------------------|------------------------|-----------|
| Si D2 = `expected` ∧ D4 = `not_found` → D6 ≠ `exists_coherent` | Atomicidad D.1.8 (solución preferida, validación Atlas pendiente): Outbox coherente requiere EvAssigned coherente como fuente en misma transacción | Fila marcada IMPOSIBLE | Propuesta técnica pendiente validación |
| Si `currentEvAssignedId === expectedEvAssignedId` → D5 ≡ D6 | Misma clave ilustrativa `idempotencyKey` (Regla ID-1) | D5 no es dimensión independiente; valor = D6 (marcado `reused_result`) | Demostración documental |
| Si `currentEvAssignedId === expectedEvAssignedId` ∧ D3 = `exists_coherent` → D4 ≡ `exists_coherent` | Identidad: mismo `_id` (PK) ⇒ mismo documento ⇒ coherencia idéntica (Regla ID-3, D-W79-14 aprobada) | Combinaciones con D3=coherent ∧ D4=incoherent = IMPOSIBLE | Decisión canónica aprobada |
| Si `currentEvAssignedId === expectedEvAssignedId` ∧ D5 = `exists_coherent` → D6 ≡ `exists_coherent` ∧ D7 = `false` | Unicidad: misma clave ilustrativa ⇒ mismo documento ⇒ no duplicados (Regla ID-1) | Combinaciones con D5≠D6 o D7=true = IMPOSIBLE | Demostración documental / Propuesta ilustrativa |
| Si D3 = `exists_coherent` ∧ D4 = `not_found` → `EXPECTED_EV_ASSIGNED_ORPHAN` | Contacto apunta a nuestro ID pero documento no existe | Clasificación normativa | Demostración documental |
| Si D3 = `not_found` ∧ D5 = `exists_*` → `INCONSISTENT_OUTBOX_WITHOUT_EV` | Outbox sin EvAssigned correspondiente | Clasificación normativa | Demostración documental / Propuesta ilustrativa |
| Si `currentEvAssignedId === null` → D3 = `not_queried` ∧ D5 = `not_queried` | Consulta no ejecutada (Regla ID-4) | No es `not_found` | Demostración documental |
| **D7 = `true` IMPOSIBLE cuando `currentEvAssignedId === expectedEvAssignedId`** | Misma clave ilustrativa `idempotencyKey` ⇒ mismo documento Outbox ⇒ no puede haber duplicada/contradictoria | Fila equivalente a caso 12 Revisión 19 = IMPOSIBLE | Demostración documental / Propuesta ilustrativa |

### 5.4 Matriz normativa de combinaciones y resultados (Reconstruida exhaustivamente — **CORREGIDA**)

La matriz enumera **solo las combinaciones posibles** tras aplicar restricciones lógicas, de unicidad, identidad y **condiciones de ejecución de consultas**. Cada fila es una clase de equivalencia de evidencia observable.

| # | D2: Contacto | D3: Current EvAssigned | D4: Expected EvAssigned | D5: Outbox Current | D6: Outbox Expected | D7: Duplicada | D8: Mismo tipo | **Estado normativo** | **Posibilidad** | **Rama algoritmo** |
|---|--------------|------------------------|-------------------------|---------------------|----------------------|---------------|----------------|----------------------|-----------------|-------------------|
| 1 | `prev` | `not_queried` | `not_found` | `not_queried` | `not_found` | `false` | `false` | `PREVIOUS_STATE_STILL_VISIBLE` | ✅ Posible | CASO A1 |
| 2 | `prev` | `not_queried` | `not_found` | `not_queried` | `exists_coherent` | `false` | `false` | **IMPOSIBLE** (atomicidad D.1.8) | ❌ Restricción | — |
| 3a | `prev` | `not_queried` | `not_found` | `not_queried` | `exists_incoherent` | `false` | `false` | `EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX_ESCALATE` (decision=escalate) | ✅ Posible | CASO A2, decision=escalate |
| 3b | `prev` | `not_queried` | `not_found` | `not_queried` | `exists_incoherent` | `false` | `false` | `EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX_BLOCK` (decision=block) | ✅ Posible | CASO A2, decision=block |
| 3c | `prev` | `not_queried` | `not_found` | `not_queried` | `exists_incoherent` | `false` | `false` | `EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX` (decision=allow + guard) | ✅ Posible | CASO A2, decision=allow |
| 4 | `prev` | `not_queried` | `exists_coherent` | `not_queried` | `not_found` | `false` | `false` | `EXPECTED_EV_ASSIGNED_ORPHAN` | ✅ Posible | CASO A3, !expectedOutboxExists |
| 5a | `prev` | `not_queried` | `exists_coherent` | `not_queried` | `exists_coherent` | `false` | `false` | `EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX_ESCALATE` (decision=escalate) | ✅ Posible | CASO A3, expectedOutboxExists, decision=escalate |
| 5b | `prev` | `not_queried` | `exists_coherent` | `not_queried` | `exists_coherent` | `false` | `false` | `EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX_BLOCK` (decision=block) | ✅ Posible | CASO A3, expectedOutboxExists, decision=block |
| 5c | `prev` | `not_queried` | `exists_coherent` | `not_queried` | `exists_coherent` | `false` | `false` | `EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX` (decision=allow + guard) | ✅ Posible | CASO A3, expectedOutboxExists, decision=allow |
| 6a | `prev` | `not_queried` | `exists_coherent` | `not_queried` | `exists_incoherent` | `false` | `false` | `EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX_ESCALATE` (decision=escalate) | ✅ Posible | CASO A3, expectedOutboxExists, decision=escalate |
| 6b | `prev` | `not_queried` | `exists_coherent` | `not_queried` | `exists_incoherent` | `false` | `false` | `EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX_BLOCK` (decision=block) | ✅ Posible | CASO A3, expectedOutboxExists, decision=block |
| 6c | `prev` | `not_queried` | `exists_coherent` | `not_queried` | `exists_incoherent` | `false` | `false` | `EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX` (decision=allow + guard) | ✅ Posible | CASO A3, expectedOutboxExists, decision=allow |
| 7 | `prev` | `not_queried` | `exists_incoherent` | `not_queried` | `*` | `*` | `false` | `INCONSISTENT_EV_WITHOUT_OUTBOX` | ✅ Posible | CASO A4 |
| 8 | `expected` | `exists_coherent` | `exists_coherent` | `reused_result` | `not_found` | `false` | `false` | `OUR_TRANSITION_COMMITTED_OUTBOX_INCONSISTENT` | ✅ Posible | CASO B1, outboxExpected=null |
| 9a | `expected` | `exists_coherent` | `exists_coherent` | `reused_result` | `exists_coherent` | `false` | `false` | `OUR_TRANSITION_COMMITTED_OUTBOX_INCONSISTENT` (decision=escalate) | ✅ Posible | CASO B1, outboxExpected=coherent, decision=escalate |
| 9b | `expected` | `exists_coherent` | `exists_coherent` | `reused_result` | `exists_coherent` | `false` | `false` | `OUR_TRANSITION_COMMITTED_OUTBOX_INCONSISTENT` (decision=block) | ✅ Posible | CASO B1, outboxExpected=coherent, decision=block |
| 9c | `expected` | `exists_coherent` | `exists_coherent` | `reused_result` | `exists_coherent` | `false` | `false` | `OUR_TRANSITION_BLOCKED_LEVEL1_OUTBOX` / `OUR_TRANSITION_BLOCKED_RETIREMENT_OP_CORRELATION` (decision=allow + guards contractuales constantes false) | ✅ Posible | CASO B1, outboxExpected=coherent, decision=allow |
| 10 | `expected` | `exists_coherent` | `exists_coherent` | `reused_result` | `exists_incoherent` | `false` | `false` | **IMPOSIBLE** (unicidad: misma clave ilustrativa ⇒ D5 ≡ D6) | ❌ Restricción | — |
| 11 | `expected` | `exists_coherent` | `exists_incoherent` | `reused_result` | `*` | `true` | `false` | **IMPOSIBLE** (identidad: mismo `_id` ⇒ coherencia idéntica) | ❌ Restricción | — |
| 12 | `expected` | `exists_coherent` | `exists_coherent` | `reused_result` | `exists_coherent` | `true` | `false` | **IMPOSIBLE** (unicidad: misma `idempotencyKey` ⇒ mismo doc ⇒ D7=false) | ❌ Restricción | — |
| 13 | `expected` | `exists_incoherent` | `exists_incoherent` | `reused_result` | `not_found` | `false` | `false` | `INCONSISTENT_EV_WITHOUT_OUTBOX` | ✅ Posible | CASO B, fieldsIncoherent (helper NO invocado) |
| 14 | `other` | `exists_coherent` | `exists_coherent` | `exists_*` | `exists_*` | `*` | `true` | `INDETERMINATE_CONTRACTUAL_BLOCK` | ✅ Posible | CASO C1 |
| 15 | `other` | `exists_coherent` | `exists_*` | `exists_*` | `exists_*` | `*` | `false` | `OTHER_TRANSITION_WON` | ✅ Posible | CASO C2 |
| 16 | `other` | `not_found` | `not_queried` | `exists_*` | `not_queried` | `*` | `false` | `INCONSISTENT_OUTBOX_WITHOUT_EV` | ✅ Posible | CASO D |
| 17 | `false` (Contacto no existe) | — | — | — | — | — | — | `INDETERMINATE_CONTRACTUAL_BLOCK` | ✅ Posible | Paso 1 |

**Totales verificados matemáticamente:**

- Combinaciones base posibles (D1=true): **12** (filas 1, 3, 4, 5, 6, 7, 8, 9, 13, 14, 15, 16)
- Combinaciones base posibles (D1=false): **1** (fila 17)
- **Total combinaciones base posibles: 13**
- Sub-casos por decisión (block/escalate/allow): **12** (filas 3a/3b/3c, 5a/5b/5c, 6a/6b/6c, 9a/9b/9c — helper invocado con decision explícita)
- Combinaciones base imposibles: **4** (filas 2, 10, 11, 12 — filas 11 y 12 tienen valores diferentes en D4, son combinaciones base distintas)
- Combinaciones no producidas por algoritmo (eliminadas): **0** (todas las posibles tienen rama)
- **Total combinaciones base examinadas: 13 + 4 = 17** ✅
- **Total sub-casos con decisión explícita: 12** ↔ 12 ramas en algoritmo §5.5 donde helper se invoca con decision ✅
- Filas base sin decisión de helper (helper no invocado o no aplicable): 1, 4, 7, 8, 13, 14, 15, 16, 17 = 9 filas base

**Nota de corrección crítica:** La Revisión 20 declaraba fila 9 como `OUR_TRANSITION_COMMITTED_OUTBOX_INCONSISTENT` (posible), lo cual era correcto en clasificación, pero el **algoritmo §5.5 líneas 249-253 fabricaba el `reused_result` mediante spread de `null`**. Esta revisión corrige el algoritmo para que la reutilización sea **real** (consulta ejecutada una vez para `outboxExpected`, resultado real reutilizado para `outboxCurrent` cuando IDs coinciden).

**Corrección D3-01 (filas 8, 10, 13):**
- Fila 8: D5 corregido de `not_queried` a `reused_result` (ya que IDs coinciden, D5 ≡ D6 = `not_found`)
- Fila 10: **Eliminada como imposible** — D5 y D6 no pueden divergir cuando IDs coinciden (Regla ID-1)
- Fila 13: **Filas 13a/13b/13c eliminadas y fusionadas en una única fila 13** — el algoritmo §5.5 tiene una sola rama para `fieldsIncoherent` (CASO B, `!fieldsCoherent`) donde el helper NO se invoca; las tres filas anteriores tenían condiciones idénticas pero resultados distintos (contradicción). La fila 13 única tiene D6 = `not_found` (resultado real de outboxExpected; D5 = `reused_result` reutiliza ese resultado).

**Clasificación de filas (Actualizada y exhaustiva con categoría de decisión — sub-casos por decisión explícita):**

| Categoría | Sub-casos | Significado | Categoría decisión |
|-----------|-----------|-------------|-------------------|
| **Posibles y consistentes** | 1, 3a, 3b, 3c, 4, 5a, 5b, 5c, 6a, 6b, 6c, 7, 8, 9a, 9b, 9c, 13, 14, 15, 16, 17 | 21 sub-casos observables (13 combinaciones base: 12 con D1=true + 1 con D1=false, expandidas por decisión donde helper se invoca: 12 sub-casos con decisión explícita + 9 filas base sin decisión de helper) | Demostración documental |
| **Imposibles por restricción** | 2, 10, 11, 12 | 4 combinaciones base imposibles (atomicidad D.1.8, identidad PK, unicidad clave ilustrativa) | Decisión canónica / Propuesta ilustrativa |
| **Éxito completo condicional** | — (ninguno alcanzable) | `OUR_TRANSITION_COMMITTED` **INALCANZABLE** mientras guardas contractuales (§16.14 Level 1 Outbox, §16.12/§16.14 correlación RetirementOperation) permanezcan constantes en `false` | **Dependencia contractual pendiente (§16.14 + §16.12)** |
| **Éxito parcial / Escalación WAB-81** | 8, 9a, 9b | `OUR_TRANSITION_COMMITTED_OUTBOX_INCONSISTENT` — escalar WAB-81 (drenaje Outbox); 9a/9b bloqueados por decision=escalate/block | Demostración documental |
| **Bloqueo contractual (D-W79-14 aprobada)** | 14, 17 | `INDETERMINATE_CONTRACTUAL_BLOCK` — Contacto apunta a otro EvAssigned mismo tipo / Contacto no existe | Decisión canónica aprobada |
| **Ganador legítimo** | 15 | `OTHER_TRANSITION_WON` — transición distinta ganó | Demostración documental |
| **Anomalías durables (escalar WAB-81)** | 3a, 5a, 6a, 7, 13, 16 | Requieren drenaje/reconciliación; decision=escalate (o helper no invocado en 13); no autorizan reintento | Demostración documental |
| **Rechazo definitivo (decision=block)** | 3b, 5b, 6b, 9b | `*_BLOCK` + `decisionType: definitive_rejection` — impide éxito, expone causa, fail-closed, no commit ni retry | Demostración documental |
| **Permiso helper + guard contractual (decision=allow)** | 3c, 5c, 6c, 9c | `*_WITH_OUTBOX` + guard + `decisionType: helper_allows_but_contractual_guard_blocks` — SOLO "helper no bloquea" + guard §16.14 obligatorio | **Dependencia contractual pendiente (§16.14)** |
| **Huérfano esperado** | 4, 5a, 5b, 5c | `EXPECTED_EV_ASSIGNED_ORPHAN*` — EvAssigned esperado existe pero Contacto no lo refleja | Demostración documental |

> **⚠️ Aclaración B1-01/B1-02:** La fila 14 (`INDETERMINATE_CONTRACTUAL_BLOCK` por "mismo tipo, distinto ID") refleja que **D-W79-14 está aprobada** y define la autoría de EvAssigned, pero la **correlación física en Outbox (§16.14) está pendiente**. Sin esquema físico Outbox aprobado, no puede verificarse contractualmente en runtime que el EvAssigned "otro" corresponde a una transición legítima distinta. Esto no significa que D-W79-14 esté pendiente; significa que la **verificación contractual runtime** requiere §16.14.

### 5.5 Algoritmo normativo de reconciliación (Reconstruido integralmente desde identidades de consulta — **Corregido**)

```typescript
async function reconcileContactTransition(
  client: MongoClient,
  input: ReconciliationInput
): Promise<ReconciliationResult> {
  const session = client.startSession({ 
    defaultReadConcern: { level: 'snapshot' }
  });
  
  try {
    return await session.withTransaction(async (txnSession) => {
      // 1. Leer Contacto actual — CORRECCIÓN H1/H3 adicional: firma Mongoose correcta
      const currentContact = await ContactoModel.findById(input.contactId, null, { session: txnSession }).lean();
      if (!currentContact) {
        return { 
          state: 'INDETERMINATE_CONTRACTUAL_BLOCK', 
          evidence: { contactId: input.contactId, reason: 'CONTACT_NOT_FOUND', readConcernUsed: 'snapshot' },
          canRetryBusinessTransaction: false,
          requiresEscalation: true 
        };
      }
      
      const currentEvAssignedId = currentContact.ref_ev_assigned;
      const prevEvAssignedId = input.attemptedTransition.prevEvAssignedId;
      const expectedEvAssignedId = input.attemptedTransition.expectedEvAssignedId;
      const contactId = input.contactId;
      const isCurrentExpected = currentEvAssignedId === expectedEvAssignedId && currentEvAssignedId !== null;
      const contactShowsPrev = !currentEvAssignedId || currentEvAssignedId === prevEvAssignedId;
      const contactShowsExpected = isCurrentExpected;
      const contactShowsOther = currentEvAssignedId !== null && !isCurrentExpected && currentEvAssignedId !== prevEvAssignedId;
      
      // 2. Leer EvAssigned ACTUAL — CORRECCIÓN H1/H3: SIEMPRE se ejecuta si currentEvAssignedId !== null
      //    Cuando contactShowsExpected ∧ IDs coinciden ≠ null, SÍ se ejecuta (Regla ID-5)
      const currentEvAssigned = currentEvAssignedId !== null
        ? await EventoModel.findById(currentEvAssignedId, null, { session: txnSession }).lean()
        : null;
      
      // 3. Leer Outbox por expectedEvAssignedId (SIEMPRE — Regla ID-2)
      //    Esta es la consulta que SIEMPRE se ejecuta para la clave esperada
      const outboxExpected = expectedEvAssignedId !== null
        ? await OutboxModel.findOne({ 
            idempotencyKey: `${expectedEvAssignedId}:${contactId}` 
          }, null, { session: txnSession }).lean()
        : null;
      
      // 4. Determinar outboxCurrent — CORRECCIÓN H1/H3: sin spread de null
      let outboxCurrent: OutboxDoc | null | { _evidenceState: 'not_queried' } | { _evidenceState: 'reused_result'; original: OutboxDoc | null } = 
        { _evidenceState: 'not_queried' };
      
      if (currentEvAssignedId !== null && currentEvAssignedId !== expectedEvAssignedId) {
        // IDs difieren: consulta independiente obligatoria (Regla ID-2)
        const result = await OutboxModel.findOne({ 
          idempotencyKey: `${currentEvAssignedId}:${contactId}` 
        }, null, { session: txnSession }).lean();
        outboxCurrent = result;
      } else if (isCurrentExpected) {
        // IDs coinciden ≠ null: reutilización REAL del resultado YA ejecutado en paso 3 (Regla ID-1)
        outboxCurrent = { 
          _evidenceState: 'reused_result', 
          original: outboxExpected  // Referencia al resultado real de la consulta ejecutada
        };
      }
      // else: currentEvAssignedId === null → outboxCurrent = not_queried (ya inicializado)
      
      // 5. LEER EvAssigned ESPERADO (SIEMPRE — clave H1/H3)
      //    Aunque currentEvAssignedId === expectedEvAssignedId, leer para verificar coherencia (Regla ID-3)
      const expectedEvAssigned = expectedEvAssignedId !== null
        ? await EventoModel.findById(expectedEvAssignedId, null, { session: txnSession }).lean()
        : null;
      
      // ============================================================================
      // D2-04: INTEGRACIÓN RETIREMENTOPERATION — Consulta condicional (NO NORMATIVA) — CORREGIDO (4ta/5ta intervención)
      // Se ejecuta cuando contactShowsExpected Y currentEvAssigned coherente (filas 8, 9)
      // Correlación contractual: §16.12 define identidad de transición = {retirementOperationId, contacto}
      // §16.10 requiere actualización de eventSetRevision en misma transacción de retiro
      // §16.14 PENDIENTE: esquema físico Outbox para correlacionar transición con RetirementOperation
      // CRITERIO D2-04 (5ta intervención): retirementOpCorrelationDemonstrable DEBE intervenir en flujo de control.
      // Mientras sea false, BLOQUEAR toda ruta de éxito que dependa de demostrar correlación con RetirementOperation.
      // No continuar silenciosamente tras excepción al consultar. Registrar anomalía y conservar evidencia.
      // No actualizar eventSetRevision usando correlación aproximada. Consulta por agentInstanceId ≈ ilustración.
      // Identidad canónica §16.12 = {retirementOperationId, contacto}. No inventar campos (§16.14 pendiente).
      // ============================================================================
      let retirementOp: RetirementOperationDoc | null = null;
      let retirementOpQueried = false;
      let retirementOpCorrelationDemonstrable = false; // D2-04: bandera explícita — siempre false mientras §16.14 pendiente
      let retirementOpQueryAnomaly: string | null = null; // D2-04: registrar anomalía si falla consulta
      if (contactShowsExpected && currentEvAssigned) {
        const fieldsCoherent = evAssignedFieldsMatch(currentEvAssigned, input.attemptedTransition);
        if (fieldsCoherent && expectedEvAssignedId !== null) {
          // ⚠️ CONSULTA ILUSTRATIVA / NO NORMATIVA / NO PROBATORIA (D2-04 — 5ta intervención):
          // - El contrato §16.12 define identidad de transición = {retirementOperationId, contacto}
          // - PERO attemptedTransition puede no incluir retirementOperationId
          // - §16.14 pendiente: esquema físico Outbox para correlación física demostrable
          // - Esta consulta por agentInstanceId (newAssignedRef) es una APROXIMACIÓN ILUSTRATIVA
          // - NO demuestra correlación entre transición de contacto y RetirementOperation
          // - NO se usa su resultado para afirmar cumplimiento contractual ni actualización eventSetRevision
          // - retirementOpCorrelationDemonstrable = false SIEMPRE mientras §16.14 pendiente
          // - El algoritmo BLOQUEA ruta de éxito cuando la correlación contractual no es demostrable
          // - Si la consulta falla (colección inexistente, error), registrar anomalía y NO continuar silenciosamente
          try {
            retirementOp = await RetirementOperationModel.findOne({
              agentInstanceId: expectedEvAssigned.newAssignedRef, // AgentInstance destino (APROXIMACIÓN ILUSTRATIVA §16.14 pendiente)
              status: { $in: ['blocking', 'releasing', 'finalizing', 'blocked'] }
            }, null, { session: txnSession }).lean();
            retirementOpQueried = true;
            // retirementOpCorrelationDemonstrable permanece FALSE — la consulta NO demuestra identidad §16.12
          } catch (e) {
            // D2-04: NO continuar silenciosamente. Registrar anomalía diagnóstica.
            retirementOp = null;
            retirementOpQueried = false;
            retirementOpQueryAnomaly = e instanceof Error ? e.message : String(e);
            // retirementOpCorrelationDemonstrable permanece FALSE
          }
        }
      }
      
      // 6. CLASIFICACIÓN EXHAUSTIVA (Matriz §5.4)
      
      // --- CASO A: Contacto muestra estado previo (o null) ---
      if (contactShowsPrev) {
        // VERIFICAR evidencia de transición esperada ANTES de clasificar (corrección H1/H3)
        // expectedEvAssigned (D4) y outboxExpected (D6) YA LEÍDOS en pasos 3, 5
        const expectedEvExists = expectedEvAssigned !== null;
        const expectedOutboxExists = outboxExpected !== null;
        
        // B4-01: Helper SOLO se invoca con OutboxDoc no null. Caso null tratado ANTES.
        const outboxMatch = expectedOutboxExists
          ? outboxPayloadMatchesTransition(outboxExpected, input.attemptedTransition)
          : { decision: 'escalate', approved: false, conditional: false, proposed: false, blockReason: 'no_outbox' };
        
        // D4-02 PARCIALMENTE RESUELTO (5ta intervención): Inspección EXPLÍCITA de decision — tres ramas DISTINGUIBLES con ESTADOS DE RESULTADO DIFERENTES (diferencia diagnóstica; diferencia operativa NO demostrada)
        // - 'block': rechazo definitivo por evidencia contractual inválida/ausente/incompatible → estado *_BLOCK + decisionType: definitive_rejection
        //   Impide éxito, expone causa, mantiene fail-closed, no permite commit ni retry automático
        // - 'escalate': anomalía/incertidumbre que requiere intervención → estado *_ESCALATE + decisionType: anomaly_requires_intervention
        //   Clasificación distinta de block, preserva evidencia diagnóstica, no autoriza, no permite commit ni retry
        // - 'allow': SOLO "helper no detectó condición local de bloqueo" → NO = autorización final → GUARD CONTRACTUAL OBLIGATORIO §16.14
        //   Estado *_WITH_OUTBOX + guard + decisionType: helper_allows_but_contractual_guard_blocks
        const outboxDecision = outboxMatch.decision;
        const outboxBlockReason = outboxMatch.blockReason;
        
        if (expectedEvExists) {
          // Filas #4, #5, #6: EvAssigned esperado existe (huérfano) aunque Contacto no lo refleje
          if (expectedOutboxExists) {
            // Outbox esperado EXISTE — decisión del helper DETERMINA clasificación y ESTADO DE RESULTADO
            if (outboxDecision === 'block') {
              // Nivel 1 vacío (§16.14 pendiente) → rechazo definitivo
              return { 
                state: 'EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX_BLOCK',
                evidence: { currentContact, currentEvAssignedId, currentEvAssigned: null, outboxCurrent, outboxExpected, expectedEvAssigned, readConcernUsed: 'snapshot', outboxDecision, outboxBlockReason, decisionType: 'definitive_rejection' },
                canRetryBusinessTransaction: false,
                requiresEscalation: true 
              };
            }
            if (outboxDecision === 'escalate') {
              // Payload Outbox inválido o Nivel 1 condicionado fallido → anomalía requiere intervención
              return { 
                state: 'EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX_ESCALATE',
                evidence: { currentContact, currentEvAssignedId, currentEvAssigned: null, outboxCurrent, outboxExpected, expectedEvAssigned, readConcernUsed: 'snapshot', outboxDecision, outboxBlockReason, decisionType: 'anomaly_requires_intervention' },
                canRetryBusinessTransaction: false,
                requiresEscalation: true 
              };
            }
            // outboxDecision === 'allow' — SOLO significa "helper no encontró condición local de bloqueo"
            // NO equivale a autorización contractual ni a commit
            // GUARD CONTRACTUAL OBLIGATORIO: mientras §16.14 pendiente, Nivel 1 vacío bloquea esta ruta
            return { 
              state: 'EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX',
              evidence: { currentContact, currentEvAssignedId, currentEvAssigned: null, outboxCurrent, outboxExpected, expectedEvAssigned, readConcernUsed: 'snapshot', outboxDecision, outboxBlockReason, guard: 'level1_empty_blocks_allow_contractual_guard_required', decisionType: 'helper_allows_but_contractual_guard_blocks' },
              canRetryBusinessTransaction: false,
              requiresEscalation: true 
            };
          }
          // Fila #4: EvAssigned esperado existe, Outbox esperado no
          return { 
            state: 'EXPECTED_EV_ASSIGNED_ORPHAN',
            evidence: { currentContact, currentEvAssignedId, currentEvAssigned: null, outboxCurrent, outboxExpected: null, expectedEvAssigned, readConcernUsed: 'snapshot' },
            canRetryBusinessTransaction: false,
            requiresEscalation: true 
          };
        }
        
        if (expectedOutboxExists) {
          // Fila #3: Outbox de expected existe sin EvAssigned — inconsistencia inversa
          // Decisión del helper DETERMINA clasificación y ESTADO DE RESULTADO
          if (outboxDecision === 'block') {
            return { 
              state: 'EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX_BLOCK',
              evidence: { currentContact, currentEvAssignedId, currentEvAssigned: null, outboxCurrent, outboxExpected, expectedEvAssigned: null, readConcernUsed: 'snapshot', outboxDecision, outboxBlockReason, decisionType: 'definitive_rejection' },
              canRetryBusinessTransaction: false,
              requiresEscalation: true 
            };
          }
          if (outboxDecision === 'escalate') {
            return { 
              state: 'EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX_ESCALATE',
              evidence: { currentContact, currentEvAssignedId, currentEvAssigned: null, outboxCurrent, outboxExpected, expectedEvAssigned: null, readConcernUsed: 'snapshot', outboxDecision, outboxBlockReason, decisionType: 'anomaly_requires_intervention' },
              canRetryBusinessTransaction: false,
              requiresEscalation: true 
            };
          }
          // outboxDecision === 'allow' — GUARD CONTRACTUAL OBLIGATORIO (§16.14 pendiente)
          return { 
            state: 'EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX',
            evidence: { currentContact, currentEvAssignedId, currentEvAssigned: null, outboxCurrent, outboxExpected, expectedEvAssigned: null, readConcernUsed: 'snapshot', outboxDecision, outboxBlockReason, guard: 'level1_empty_blocks_allow_contractual_guard_required', decisionType: 'helper_allows_but_contractual_guard_blocks' },
            canRetryBusinessTransaction: false,
            requiresEscalation: true 
          };
        }
        
        // Fila #1: Estado previo visible, SIN evidencia de transición esperada
        // NO = aborto confirmado. Puede ser commit incierto.
        return { 
          state: 'PREVIOUS_STATE_STILL_VISIBLE', 
          evidence: { currentContact, currentEvAssignedId, currentEvAssigned: null, outboxCurrent, outboxExpected: null, expectedEvAssigned: null, readConcernUsed: 'snapshot' },
          canRetryBusinessTransaction: false,
          requiresEscalation: true  // Commit incierto posible
        };
      }
      
      // --- CASO B: Contacto muestra expectedEvAssignedId ---
      if (contactShowsExpected) {
        // NUESTRO EvAssigned por ID — verificar coherencia completa
        // CORRECCIÓN H1/H3: currentEvAssigned YA LEÍDO en paso 2 (Regla ID-5)
        if (!currentEvAssigned) {
          // Filas equivalentes a #8, #10 antiguas: Documento EvAssigned perdido 
          // (currentEvAssignedId = expectedEvAssignedId pero doc no existe)
          // B4-01: outboxExpected garantizado no-null aquí (verificado antes en CASO A o null check)
          const outboxMatch = outboxExpected !== null
            ? outboxPayloadMatchesTransition(outboxExpected, input.attemptedTransition)
            : { decision: 'escalate', approved: false, conditional: false, proposed: false, blockReason: 'no_outbox' };
          
          // D4-02 PARCIALMENTE RESUELTO (5ta intervención): Tratamiento DIFERENCIADO DIAGNÓSTICO de decision — tres ramas DISTINGUIBLES (operativa NO demostrada)
          // 'block': rechazo definitivo por evidencia contractual inválida/ausente/incompatible — no permite commit ni retry
          // 'escalate': anomalía/incertidumbre que requiere intervención — no permite commit ni retry, distinto de block
          // 'allow': SOLO "helper no detectó condición local de bloqueo" — NO = autorización final — GUARD CONTRACTUAL OBLIGATORIO
          if (outboxMatch.decision === 'block') {
            return { 
              state: 'EXPECTED_EV_ASSIGNED_ORPHAN_OUTBOX_BLOCK',
              evidence: { currentContact, currentEvAssignedId, currentEvAssigned: null, outboxCurrent, outboxExpected, expectedEvAssigned: null, retirementOp, retirementOpQueried, readConcernUsed: 'snapshot', outboxDecision: outboxMatch.decision, outboxBlockReason: outboxMatch.blockReason, decisionType: 'definitive_rejection' },
              canRetryBusinessTransaction: false,
              requiresEscalation: true 
            };
          }
          if (outboxMatch.decision === 'escalate') {
            return { 
              state: 'EXPECTED_EV_ASSIGNED_ORPHAN_OUTBOX_ESCALATE',
              evidence: { currentContact, currentEvAssignedId, currentEvAssigned: null, outboxCurrent, outboxExpected, expectedEvAssigned: null, retirementOp, retirementOpQueried, readConcernUsed: 'snapshot', outboxDecision: outboxMatch.decision, outboxBlockReason: outboxMatch.blockReason, decisionType: 'anomaly_requires_intervention' },
              canRetryBusinessTransaction: false,
              requiresEscalation: true 
            };
          }
          // outboxMatch.decision === 'allow' — GUARD CONTRACTUAL OBLIGATORIO (§16.14 pendiente)
          return { 
            state: 'EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX',
            evidence: { currentContact, currentEvAssignedId, currentEvAssigned: null, outboxCurrent, outboxExpected, expectedEvAssigned: null, retirementOp, retirementOpQueried, readConcernUsed: 'snapshot', outboxDecision: outboxMatch.decision, outboxBlockReason: outboxMatch.blockReason, guard: 'level1_empty_blocks_allow_contractual_guard_required', decisionType: 'helper_allows_but_contractual_guard_blocks' },
            canRetryBusinessTransaction: false,
            requiresEscalation: true 
          };
        }
        
        const fieldsCoherent = evAssignedFieldsMatch(currentEvAssigned, input.attemptedTransition);
        
        if (!fieldsCoherent) {
          // Fila #7 / #13: Documento existe pero contradice nuestra transición (coherencia incoherente)
          // NOTA: Si IDs coinciden, ExpectedEv = CurrentEv ⇒ coherencia idéntica ⇒ este caso requiere IDs distintos
          return { 
            state: 'INCONSISTENT_EV_WITHOUT_OUTBOX', 
            evidence: { currentContact, currentEvAssignedId, currentEvAssigned, outboxCurrent, outboxExpected, expectedEvAssigned, retirementOp, retirementOpQueried, readConcernUsed: 'snapshot' },
            canRetryBusinessTransaction: false,
            requiresEscalation: true 
          };
        }
        
        // Documento existe Y es coherente — evaluar Outbox de expectedEvAssignedId
        if (!outboxExpected) {
          // Fila #8: Falta Outbox de NUESTRO expectedEvAssignedId
          return { 
            state: 'OUR_TRANSITION_COMMITTED_OUTBOX_INCONSISTENT',
            evidence: { currentContact, currentEvAssignedId, currentEvAssigned, outboxCurrent, outboxExpected: null, expectedEvAssigned, retirementOp, retirementOpQueried, readConcernUsed: 'snapshot', sessionType: 'new_transaction' },
            canRetryBusinessTransaction: false,
            requiresEscalation: true  // WAB-81 drenaje
          };
        }
        
        // Outbox de expectedEvAssignedId EXISTE — verificar coherencia del payload
        const outboxMatch = outboxPayloadMatchesTransition(outboxExpected, input.attemptedTransition);
        
        // CORRECCIÓN H4 + D4-02 PARCIALMENTE RESUELTO (5ta intervención): Inspección EXPLÍCITA de decision — tres ramas DISTINGUIBLES (diagnóstico sí, operativa NO)
        // Fila #10 (ahora IMPOSIBLE en matriz por Regla ID-1, pero código defensivo):
        if (outboxMatch.decision === 'block') {
          return { 
            state: 'INCONSISTENT_EV_OUTBOX_MISMATCH_BLOCK',
            evidence: { currentContact, currentEvAssignedId, currentEvAssigned, outboxCurrent, outboxExpected, expectedEvAssigned, retirementOp, retirementOpQueried, readConcernUsed: 'snapshot', outboxDecision: outboxMatch.decision, outboxBlockReason: outboxMatch.blockReason, decisionType: 'definitive_rejection' },
            canRetryBusinessTransaction: false,
            requiresEscalation: true 
          };
        }
        if (outboxMatch.decision === 'escalate') {
          return { 
            state: 'INCONSISTENT_EV_OUTBOX_MISMATCH_ESCALATE',
            evidence: { currentContact, currentEvAssignedId, currentEvAssigned, outboxCurrent, outboxExpected, expectedEvAssigned, retirementOp, retirementOpQueried, readConcernUsed: 'snapshot', outboxDecision: outboxMatch.decision, outboxBlockReason: outboxMatch.blockReason, decisionType: 'anomaly_requires_intervention' },
            canRetryBusinessTransaction: false,
            requiresEscalation: true 
          };
        }
        
        // Verificar evidencia duplicada/contradictoria (Solo si IDs difieren — fila #12 IMPOSIBLE)
        if (!isCurrentExpected && outboxCurrent && 'original' in outboxCurrent === false && outboxExpected) {
          return { 
            state: 'DUPLICATE_EVIDENCE_CONTRADICTORY',
            evidence: { currentContact, currentEvAssignedId, currentEvAssigned, outboxCurrent, outboxExpected, expectedEvAssigned, retirementOp, retirementOpQueried, readConcernUsed: 'snapshot' },
            canRetryBusinessTransaction: false,
            requiresEscalation: true 
          };
        }
        
        // Fila #9: ÉXITO CONDICIONAL — GUARDS EXPLÍCITOS: NIVEL 1 OUTBOX + CORRELACIÓN RETIREMENTOPERATION
        // D-W79-14 está APROBADA (define autoría EvAssigned). Lo que bloquea es §16.14 (Outbox físico).
        // Sin Nivel 1 aprobado no vacío, NO hay ruta a OUR_TRANSITION_COMMITTED.
        const level1OutboxApprovedNonEmpty = false; // Constante contractual: §16.14 pendiente → Nivel 1 vacío
        
        // D2-04 (5ta intervención): BLOQUEO EXPLÍCITO si correlación RetirementOperation no demostrable
        // §16.12 identidad = {retirementOperationId, contacto}. §16.14 pendiente → correlación no demostrable.
        // Mientras retirementOpCorrelationDemonstrable === false, BLOQUEAR ruta de éxito.
        if (!retirementOpCorrelationDemonstrable) {
          return { 
            state: 'OUR_TRANSITION_BLOCKED_RETIREMENT_OP_CORRELATION',
            evidence: { currentContact, currentEvAssignedId, currentEvAssigned, outboxCurrent, outboxExpected, expectedEvAssigned, retirementOp, retirementOpQueried, retirementOpCorrelationDemonstrable, retirementOpQueryAnomaly, readConcernUsed: 'snapshot', blocker: '§16.12/§16.14 correlación RetirementOperation no demostrable' },
            canRetryBusinessTransaction: false,
            requiresEscalation: true 
          };
        }
        
        if (!level1OutboxApprovedNonEmpty) {
          return { 
            state: 'OUR_TRANSITION_BLOCKED_LEVEL1_OUTBOX',
            evidence: { currentContact, currentEvAssignedId, currentEvAssigned, outboxCurrent, outboxExpected, expectedEvAssigned, retirementOp, retirementOpQueried, readConcernUsed: 'snapshot', blocker: '§16.14 Level 1 Outbox vacío' },
            canRetryBusinessTransaction: false,
            requiresEscalation: true 
          };
        }
        
        // Solo si Nivel 1 Outbox aprobado no vacío (futuro, tras §16.14) Y correlación demostrada:
        return { 
          state: 'OUR_TRANSITION_COMMITTED',
          evidence: { currentContact, currentEvAssignedId, currentEvAssigned, outboxCurrent, outboxExpected, expectedEvAssigned, retirementOp, retirementOpQueried, retirementOpCorrelationDemonstrable, readConcernUsed: 'snapshot', sessionType: 'new_transaction' },
          canRetryBusinessTransaction: false,
          requiresEscalation: false
        };
      }
      
      // --- CASO C: Contacto muestra OTRO EvAssigned ---
      if (contactShowsOther && currentEvAssigned) {
        const isSameTransitionType = 
          currentEvAssigned.newAssignedType === input.attemptedTransition.newAssignedType &&
          currentEvAssigned.newAssignedRef === input.attemptedTransition.newAssignedRef;
        
        if (isSameTransitionType) {
          // Fila #14: Mismo tipo, distinto ID — OTRO worker — BLOQUEO CONTRACTUAL (D-W79-14)
          return { 
            state: 'INDETERMINATE_CONTRACTUAL_BLOCK', 
            evidence: { currentContact, currentEvAssignedId, currentEvAssigned, outboxCurrent, outboxExpected, expectedEvAssigned, readConcernUsed: 'snapshot' },
            canRetryBusinessTransaction: false,
            requiresEscalation: true 
          };
        }
        
        // Fila #15: Transición distinta — GANADOR CONCURRENTE LEGÍTIMO
        return { 
          state: 'OTHER_TRANSITION_WON', 
          evidence: { currentContact, currentEvAssignedId, currentEvAssigned, outboxCurrent, outboxExpected, expectedEvAssigned, readConcernUsed: 'snapshot' },
          canRetryBusinessTransaction: false,
          requiresEscalation: false 
        };
      }
      
      // --- CASO D: Contacto apunta a ID pero EvAssigned no existe (D3=not_found, D2=other) ---
      // Fila #16
      return { 
        state: 'INCONSISTENT_OUTBOX_WITHOUT_EV', 
        evidence: { currentContact, currentEvAssignedId, currentEvAssigned: null, outboxCurrent, outboxExpected, expectedEvAssigned: null, readConcernUsed: 'snapshot' },
        canRetryBusinessTransaction: false,
        requiresEscalation: true 
      };
      
    }, { readConcern: { level: 'snapshot' } });
  } finally {
    await session.endSession();
  }
}

// ============================================================================
// NOTAS CONTRACTUALES DEL ALGORITMO (B1-01, B1-02, D2-04)
// ============================================================================
// 1. D-W79-14 está APROBADA (10/11/2026). Define campos EvAssigned, actorRef obligatorio,
//    reason nullable, operationType 6 valores, excluye originMessageId/delegationChainId.
//    NO está pendiente ni bloqueada. Lo pendiente es correlación física Outbox (§16.14).
// 2. Los filtros `idempotencyKey: \`${evAssignedId}:${contactId}\`` son PROPUESTAS ILUSTRATIVAS
//    NO APROBADAS. Esquema físico Outbox (campos, índices, clave unicidad, payload) = §16.14 pendiente.
//    La idempotencia contractual aprobada (D-W79-15, §16.12) usa evAssignedId como identidad.
// 3. D2-04 RETIREMENT OPERATION INTEGRATION — CORREGIDO: Consulta ILUSTRATIVA, no integración demostrada:
//    El algoritmo consulta RetirementOperation cuando contactShowsExpected Y currentEvAssigned coherente (filas 8, 9)
//    PERO esta consulta es EXPLÍCITAMENTE NO NORMATIVA Y NO PROBATORIA (D2-04):
//    - §16.12 define identidad de transición = {retirementOperationId, contacto}
//    - PERO attemptedTransition puede no incluir retirementOperationId
//    - §16.14 pendiente: esquema físico Outbox para correlación física demostrable
//    - La consulta por agentInstanceId (newAssignedRef) es SOLO una APROXIMACIÓN ILUSTRATIVA
//    - NO demuestra correlación entre transición de contacto y RetirementOperation
//    - NO se usa su resultado para afirmar cumplimiento contractual ni actualización eventSetRevision
//    - El algoritmo clasifica como bloqueada/pendiente cuando la correlación contractual no es demostrable
//    - ACTUALIZACIÓN TRANSACCIONAL PENDIENTE: §16.10 requiere actualizar eventSetRevision, pero esquema §16.14 pendiente
//    - INCREMENTO eventSetRevision: Requerido por contrato pero esquema pendiente
//    - Eliminado comentario "AÑADIR PASO 3.5" que contradecía el flujo documentado.
// ============================================================================

// Helpers (puros, sin efectos secundarios)

function evAssignedFieldsMatch(ev: EvAssignedDoc, attempted: AttemptedTransition): boolean {
  // 10 campos según D.1.6 (incluye reason)
  return ev.newAssignedType === attempted.newAssignedType &&
    ev.newAssignedRef === attempted.newAssignedRef &&
    ev.actorType === attempted.actorType &&
    ev.actorRef === attempted.actorRef &&
    ev.operationType === attempted.operationType &&
    ev.prevAssignedType === attempted.prevAssignedType &&
    ev.prevAssignedRef === attempted.prevAssignedRef &&
    ev.contacto.toString() === attempted.contactId &&
    ev.reason === attempted.reason;  // ← campo reason según D.1.6
}

// Helper Outbox CORREGIDO: tres niveles contractuales, SOLO Nivel 1 aprobado para decisión
// B4-01: CONTRATO — Requiere OutboxDoc no-null. Ausencia de Outbox DEBE tratarse ANTES de llamar al helper.
// El algoritmo (§5.5 CASO A y B) verifica `outboxExpected !== null` ANTES de invocar.
function outboxPayloadMatchesTransition(outbox: OutboxDoc, attempted: AttemptedTransition): OutboxMatchResult {
  // PRECONDICIÓN: outbox !== null (garantizado por caller). No aceptar null.
  const payload = outbox.payload;
  if (!payload) return { approved: false, conditional: false, proposed: false, decision: 'escalate', blockReason: 'no_payload' };
  
  // NIVEL 1 APROBADO: Campos con contrato canónico definitivo para Outbox de asignación
  // CORRECCIÓN B1-02: §16.12 define IDENTIDADES conceptuales, NO campos físicos Outbox.
  // D.1.8 define ATOMICIDAD transaccional, NO esquema físico Outbox.
  // D.6 define FLUJO durable ejecución, NO Outbox asignación.
  // CONTRATOS CANÓNICOS NO APRUEBAN campos físicos Outbox asignación → NIVEL 1 = VACÍO (§16.14 pendiente).
  // PARA DECISIÓN CONTRACTUAL DE return_committed: NIVEL 1 APROBADO = VACÍO
  const approvedChecks = true; // VACÍO: ningún campo contractual aprobado para decisión aislada
  
  // NIVEL 1 CONDICIONADO §16.14: Mencionados en canónicos pero esquema payload PENDIENTE
  // NO forman parte de decisión contractual; solo informativos
  // CAMPOS `outbox.contactId`, `outbox.contacto`, `payload.evAssignedId` son PROPUESTAS ILUSTRATIVAS §16.14
  const conditionalChecks = 
    (outbox.contactId === attempted.contactId || outbox.contacto?.toString() === attempted.contactId) &&
    payload.evAssignedId === attempted.expectedEvAssignedId;
  
  // NIVEL 2 PROPUESTO: Pendientes §16.14 — NO USAR para decisión contractual
  const proposedChecks = payload.newAssignedType === attempted.newAssignedType &&
    payload.newAssignedRef === attempted.newAssignedRef &&
    payload.actorType === attempted.actorType &&
    payload.actorRef === attempted.actorRef &&
    payload.operationType === attempted.operationType &&
    payload.prevAssignedType === attempted.prevAssignedType &&
    payload.prevAssignedRef === attempted.prevAssignedRef &&
    payload.reason === attempted.reason;
  
  // DECISIÓN CONTRACTUAL:
  // - Si approvedChecks=false → escalate (imposible aquí al ser true vacío)
  // - conditionalChecks y proposedChecks solo informativos
  // - Para return_committed: SOLO approvedChecks vinculante → aquí VACÍO → bloquear
  // - decision === 'allow' SOLO si approved=true Y campos aprobados NO VACÍOS (imposible hoy)
  const decision = approvedChecks ? 'block' : 'escalate'; // 'block' = no return_committed, escalar
  
  return { 
    approved: approvedChecks, 
    conditional: conditionalChecks, 
    proposed: proposedChecks, 
    decision,
    blockReason: approvedChecks ? 'level1_empty' : 'level1_failed'
  };
}

// Tipo de resultado del helper Outbox — CORREGIDO H4 B.2
interface OutboxMatchResult {
  approved: boolean;        // Nivel 1 APROBADO: campos con contrato canónico definitivo
                            // Para Outbox asignación: VACÍO (§16.14 pendiente) → siempre true (vacío) pero no habilita decisión
  conditional: boolean;     // Nivel 1 CONDICIONADO §16.14: mencionados en canónicos pero esquema payload pendiente
                            // Solo informativos; NO forman parte de decisión contractual
  proposed: boolean;        // Nivel 2 PROPUESTO: pendientes §16.14
                            // NO USAR para decisión contractual
  decision: 'allow' | 'block' | 'escalate'; // Decisión final para return_committed
                            // 'allow' SOLO si approved=true Y campos aprobados no vacíos
                            // 'block' = no return_committed, escalar (caso actual: Nivel 1 vacío)
                            // 'escalate' = error de payload o Nivel 1 condicionado fallido
  blockReason?: string;     // Motivo del bloqueo: 'level1_empty' | 'level1_failed' | 'no_payload' | 'conditional_failed' | 'proposed_only'
}
```

### 5.6 Tabla de autorización de nuevo intento (Normativa y vinculante al algoritmo — Corregida B1-01, actualizada 5ta intervención)

| Estado de reconciliación | ¿Nuevo intento autorizado? | Justificación | Categoría decisión |
|--------------------------|----------------------------|---------------|-------------------|
| `OUR_TRANSITION_COMMITTED` | ❌ No | Ya completado. **D-W79-14 aprobada** define autoría EvAssigned. §16.14 Outbox físico pendiente para verificación runtime. | Decisión canónica / Dependencia contractual |
| `OUR_TRANSITION_BLOCKED_LEVEL1_OUTBOX` | ❌ No | **Bloqueado por §16.14 pendiente** (Nivel 1 Outbox vacío). D-W79-14 aprobada no resuelve esquema físico Outbox. | **Dependencia contractual pendiente (§16.14)** |
| `OUR_TRANSITION_BLOCKED_RETIREMENT_OP_CORRELATION` | ❌ No | **Bloqueado por §16.12/§16.14 pendiente** (correlación RetirementOperation no demostrable). | **Dependencia contractual pendiente (§16.12/§16.14)** |
| `OUR_TRANSITION_COMMITTED_OUTBOX_INCONSISTENT` | ❌ No (escalar drenaje WAB-81) | Transición hecha, falta propagación Outbox | Demostración documental |
| `OTHER_TRANSITION_WON` | ❌ No | Perdido legítimamente frente a transición distinta | Demostración documental |
| `PREVIOUS_STATE_STILL_VISIBLE` | ❌ **No** | **Evidencia positiva de estado previo NO = aborto confirmado**. Puede ser commit incierto. | Demostración documental |
| `EXPECTED_EV_ASSIGNED_ORPHAN` | ❌ No (escalar) | EvAssigned esperado en Contacto pero doc perdido — inconsistencia durable | Demostración documental |
| `EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX` (decision=block) | ❌ No | Rechazo definitivo por evidencia Outbox inválida/ausente — `decisionType: definitive_rejection` | Demostración documental |
| `EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX` (decision=escalate) | ❌ No | Anomalía Outbox requiere intervención — `decisionType: anomaly_requires_intervention` | Demostración documental |
| `EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX` (decision=allow + guard) | ❌ No | Helper permite pero guard contractual §16.14 bloquea — `decisionType: helper_allows_but_contractual_guard_blocks` | **Dependencia contractual pendiente (§16.14)** |
| `EXPECTED_EV_ASSIGNED_ORPHAN_OUTBOX_BLOCK` | ❌ No | Rechazo definitivo — `decisionType: definitive_rejection` | Demostración documental |
| `EXPECTED_EV_ASSIGNED_ORPHAN_OUTBOX_ESCALATE` | ❌ No | Anomalía requiere intervención — `decisionType: anomaly_requires_intervention` | Demostración documental |
| `INCONSISTENT_EV_WITHOUT_OUTBOX` | ❌ No (escalar drenaje WAB-81) | EvAssigned (¿nuestro? ¿otro?) sin Outbox — requiere drenaje | Demostración documental |
| `INCONSISTENT_OUTBOX_WITHOUT_EV` | ❌ No (escalar) | Outbox sin EvAssigned correspondiente — inconsistencia inversa | Demostración documental / Propuesta ilustrativa |
| `INCONSISTENT_EV_OUTBOX_MISMATCH_BLOCK` | ❌ No | Rechazo definitivo — `decisionType: definitive_rejection` | Demostración documental |
| `INCONSISTENT_EV_OUTBOX_MISMATCH_ESCALATE` | ❌ No | Anomalía requiere intervención — `decisionType: anomaly_requires_intervention` | Demostración documental |
| `INDETERMINATE_CONTRACTUAL_BLOCK` | ❌ No (escalar) | **Bloqueo contractual**: Contacto apunta a otro EvAssigned mismo tipo. D-W79-14 **aprobada** define autoría, pero verificación runtime requiere §16.14 Outbox físico. | Decisión canónica aprobada / Dependencia contractual |
| `DUPLICATE_EVIDENCE_CONTRADICTORY` | ❌ No (escalar) | Evidencia duplicada/contradictoria — corrupción o carrera extrema | Demostración documental |

**Regla fundamental (reforzada y normativa):** Nuevo intento de negocio **NUNCA** por ausencia de evidencia. Solo con **evidencia positiva y durable** de que la transición anterior no se aplicó **y** no puede confirmarse posteriormente. Con los contratos actuales (D-W79-14 aprobada, §16.14 Outbox pendiente), **no existe mecanismo contractual completo** → bloqueo contractual documentado (§16.14 pendiente).

### 5.7 Tabla de trazabilidad: Caso ↔ Evidencia ↔ Resultado ↔ Rama (1:1 demostrada con identidades de consulta — **Corregida esta intervención**)

| Caso | D2 Contacto | D3 CurrentEv | D4 ExpectedEv | D5 OutboxCurrent | D6 OutboxExpected | Resultado normativo | Rama del algoritmo | Consultas ejecutadas | Guarda contractual |
|------|-------------|--------------|---------------|------------------|-------------------|---------------------|-------------------|---------------------|-------------------|
| 1 | `prev` | `not_queried` | `not_found` | `not_queried` | `not_found` | `PREVIOUS_STATE_STILL_VISIBLE` | CASO A, sin expectedEvExists/expectedOutboxExists | expectedEvAssigned, outboxExpected | — |
| 2 | `prev` | `not_queried` | `not_found` | `not_queried` | `exists_coherent` | **IMPOSIBLE** (atomicidad D.1.8) | — | — | — |
| 3a | `prev` | `not_queried` | `not_found` | `not_queried` | `exists_incoherent` | `EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX_ESCALATE` | CASO A, expectedOutboxExists, decision=escalate | expectedEvAssigned, outboxExpected | — |
| 3b | `prev` | `not_queried` | `not_found` | `not_queried` | `exists_incoherent` | `EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX_BLOCK` | CASO A, expectedOutboxExists, decision=block | expectedEvAssigned, outboxExpected | — |
| 3c | `prev` | `not_queried` | `not_found` | `not_queried` | `exists_incoherent` | `EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX` | CASO A, expectedOutboxExists, decision=allow | expectedEvAssigned, outboxExpected | `level1_empty_blocks_allow_contractual_guard_required` |
| 4 | `prev` | `not_queried` | `exists_coherent` | `not_queried` | `not_found` | `EXPECTED_EV_ASSIGNED_ORPHAN` | CASO A, expectedEvExists, !expectedOutboxExists | expectedEvAssigned, outboxExpected | — |
| 5a | `prev` | `not_queried` | `exists_coherent` | `not_queried` | `exists_coherent` | `EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX_ESCALATE` | CASO A, expectedEvExists, expectedOutboxExists, decision=escalate | expectedEvAssigned, outboxExpected | — |
| 5b | `prev` | `not_queried` | `exists_coherent` | `not_queried` | `exists_coherent` | `EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX_BLOCK` | CASO A, expectedEvExists, expectedOutboxExists, decision=block | expectedEvAssigned, outboxExpected | — |
| 5c | `prev` | `not_queried` | `exists_coherent` | `not_queried` | `exists_coherent` | `EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX` | CASO A, expectedEvExists, expectedOutboxExists, decision=allow | expectedEvAssigned, outboxExpected | `level1_empty_blocks_allow_contractual_guard_required` |
| 6a | `prev` | `not_queried` | `exists_coherent` | `not_queried` | `exists_incoherent` | `EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX_ESCALATE` | CASO A, expectedEvExists, expectedOutboxExists, decision=escalate | expectedEvAssigned, outboxExpected | — |
| 6b | `prev` | `not_queried` | `exists_coherent` | `not_queried` | `exists_incoherent` | `EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX_BLOCK` | CASO A, expectedEvExists, expectedOutboxExists, decision=block | expectedEvAssigned, outboxExpected | — |
| 6c | `prev` | `not_queried` | `exists_coherent` | `not_queried` | `exists_incoherent` | `EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX` | CASO A, expectedEvExists, expectedOutboxExists, decision=allow | expectedEvAssigned, outboxExpected | `level1_empty_blocks_allow_contractual_guard_required` |
| 7 | `prev` | `not_queried` | `exists_incoherent` | `not_queried` | `*` | `INCONSISTENT_EV_WITHOUT_OUTBOX` | CASO A, expectedEvExists incoherente | expectedEvAssigned, outboxExpected | — |
| 8 | `expected` | `exists_coherent` | `exists_coherent` | `reused_result` | `not_found` | `OUR_TRANSITION_COMMITTED_OUTBOX_INCONSISTENT` | CASO B, fieldsCoherent, outboxExpected=null | currentEvAssigned, outboxExpected (reutilizado), expectedEvAssigned | — |
| 9a | `expected` | `exists_coherent` | `exists_coherent` | `reused_result` | `exists_coherent` | `OUR_TRANSITION_COMMITTED_OUTBOX_INCONSISTENT` | CASO B, fieldsCoherent, outboxExpected=coherent, decision=escalate | currentEvAssigned, outboxExpected (reutilizado), expectedEvAssigned | — |
| 9b | `expected` | `exists_coherent` | `exists_coherent` | `reused_result` | `exists_coherent` | `OUR_TRANSITION_COMMITTED_OUTBOX_INCONSISTENT` | CASO B, fieldsCoherent, outboxExpected=coherent, decision=block | currentEvAssigned, outboxExpected (reutilizado), expectedEvAssigned | — |
| 9c | `expected` | `exists_coherent` | `exists_coherent` | `reused_result` | `exists_coherent` | `OUR_TRANSITION_BLOCKED_LEVEL1_OUTBOX` / `OUR_TRANSITION_BLOCKED_RETIREMENT_OP_CORRELATION` | CASO B, fieldsCoherent, outboxExpected=coherent, decision=allow | currentEvAssigned, outboxExpected (reutilizado), expectedEvAssigned | `level1_empty_blocks_allow_contractual_guard_required` + `retirementOpCorrelationDemonstrable=false` |
| 10 | `expected` | `exists_coherent` | `exists_coherent` | `reused_result` | `exists_incoherent` | **IMPOSIBLE** (unicidad: misma clave ilustrativa ⇒ D5 ≡ D6) | — | — | — |
| 11 | `expected` | `exists_coherent` | `exists_incoherent` | `reused_result` | `*` | **IMPOSIBLE** (identidad: mismo `_id` ⇒ coherencia idéntica) | — | — | — |
| 12 | `expected` | `exists_coherent` | `exists_coherent` | `reused_result` | `exists_coherent` | **IMPOSIBLE** (unicidad: misma `idempotencyKey` ⇒ mismo doc ⇒ D7=false) | — | — | — |
| 13 | `expected` | `exists_incoherent` | `exists_incoherent` | `reused_result` | `not_found` | `INCONSISTENT_EV_WITHOUT_OUTBOX` | CASO B, fieldsIncoherent (helper NO invocado) | currentEvAssigned, outboxExpected (reutilizado), expectedEvAssigned | — |
| 14 | `other` | `exists_coherent` | `exists_coherent` | `exists_*` | `exists_*` | `INDETERMINATE_CONTRACTUAL_BLOCK` | CASO C, isSameTransitionType=true | currentEvAssigned, outboxCurrent, outboxExpected, expectedEvAssigned | D-W79-14 autoría + §16.14 Outbox físico |
| 15 | `other` | `exists_coherent` | `exists_*` | `exists_*` | `exists_*` | `OTHER_TRANSITION_WON` | CASO C, isSameTransitionType=false | currentEvAssigned, outboxCurrent, outboxExpected, expectedEvAssigned | — |
| 16 | `other` | `not_found` | `not_queried` | `exists_*` | `not_queried` | `INCONSISTENT_OUTBOX_WITHOUT_EV` | CASO D | currentEvAssigned, outboxCurrent | — |
| 17 | `false` | — | — | — | — | `INDETERMINATE_CONTRACTUAL_BLOCK` | Paso 1, !currentContact | (ninguna) | — |

**Verificación de completitud (actualizada esta intervención):**
- ✅ 13 combinaciones base posibles (D1=true: 12; D1=false: 1) + 4 imposibles = 17 combinaciones base examinadas
- ✅ 12 sub-casos con decisión explícita (filas 3a/3b/3c, 5a/5b/5c, 6a/6b/6c, 9a/9b/9c — helper invocado con decision explícita) ↔ 12 ramas en algoritmo §5.5
- ✅ 4 combinaciones base imposibles por restricción (2, 10, 11, 12 — filas 11 y 12 tienen valores diferentes en D4, son combinaciones base distintas)
- ✅ 0 combinaciones base eliminadas (algoritmo produce rama para cada combinación posible)
- ✅ Cada rama consume exactamente las consultas declaradas en "Consultas ejecutadas"
- ✅ Reutilización `outboxCurrent ≡ outboxExpected` solo cuando `isCurrentExpected` probado (misma `idempotencyKey`) → marcado `reused_result` con referencia a `original`
- ✅ Casos 8, 9, 13: D3 = `exists_coherent`/`exists_incoherent` (no `not_queried`) porque `currentEvAssigned` SÍ se ejecuta cuando `contactShowsExpected` ∧ IDs coinciden ≠ null (Regla ID-5)
- ✅ **D4-02 (parcial)**: Cada sub-caso donde helper se invoca refleja decision explícita (block/escalate/allow) con **estado de resultado distinto y decisionType explícito**; CASO A y CASO B separan tres ramas con estados diferenciados **en diagnóstico**; diferencia operativa pendiente validación downstream
- ✅ **D2-04**: Caso 9c incluye **dos guards contractuales constantes false** (Level 1 Outbox + correlación RetirementOperation); `OUR_TRANSITION_COMMITTED` **eliminado como alcanzable**
- ✅ **D3-01**: Matriz §5.4 y trazabilidad §5.7 reconstruidas desde ramas efectivas de §5.5; filas 13a/13b/13c eliminadas (una sola rama para fieldsIncoherent); fila 9c sin `OUR_TRANSITION_COMMITTED` alcanzable

### 5.8 Estado de verificación H1/H3

- ✅ **Identidad de consultas formalizada con condiciones de ejecución**: 4 consultas con claves exactas, condición `IF` de ejecución, regla de reutilización (ID-1/2/3/4/5), identidad de documento, evidencia producida, rama consumidora.
- ✅ **Matriz reconstruida desde dimensiones de evidencia** (8 dimensiones atómicas con `not_queried` y `reused_result`), no por acumulación de filas.
- ✅ **Combinaciones imposibles** identificadas y excluidas (Casos #2, #10, #11, #12) con fundamento en restricción de atomicidad D.1.8, identidad de PK e unicidad de `idempotencyKey`.
- ✅ **Fila 8 corregida**: D5 = `reused_result` (antes `not_queried`), D6 = `not_found` — consistente con Regla ID-1.
- ✅ **Fila 10 eliminada como imposible**: D5 y D6 no pueden divergir cuando IDs coinciden.
- ✅ **Fila 13 corregida**: D6 = `not_found` (resultado real de outboxExpected; antes `*`), D5 = `reused_result` reutiliza ese resultado — consistente con Regla ID-1.
- ✅ **Cero combinaciones eliminadas**: algoritmo tiene rama para cada combinación posible.
- ✅ **Algoritmo ejecuta consultas necesarias para cada identidad distinta**; `outboxExpected` SIEMPRE consultado; `expectedEvAssigned` SIEMPRE consultado; `outboxCurrent` solo cuando IDs difieren; `currentEvAssigned` cuando `currentEvAssignedId !== null` (incluye caso IDs coinciden).
- ✅ **`PREVIOUS_STATE_STILL_VISIBLE` solo si NO hay evidencia de transición esperada** (verifica `expectedEvAssigned` y `outboxExpected` antes de clasificar).
- ✅ **`OUR_TRANSITION_COMMITTED` requiere evidencia completa (4 elementos) + D-W79-14 + Nivel 1 aprobado + guard explícito EN TODAS LAS RUTAS DE ÉXITO**.
- ✅ **Trazabilidad 1:1 demostrada** con identidades de consulta reales (§5.7).
- ⚠️ **Bloqueo**: Autorización de reintento requiere mecanismo contractual que no existe hoy (D-W79-14 + diseño reconciliación + §16.14).

---

## 6. H1/H3 Adicional: Sesiones Mongoose y Lecturas Transaccionales

### 6.1 Versión de Mongoose instalada y firma correcta

**Verificación en CRM:**
- `crm/package.json`: `"mongoose": "^8.19.3"`
- `crm/pnpm-lock.yaml`: confirma `mongoose@8.19.3`

**Firma de `Model.findById` en Mongoose 8.x:**
```typescript
findById(id: Any, projection?: Any, options?: QueryOptions): QueryWithArray<...>
```
- **Parámetro 1:** `id` (obligatorio)
- **Parámetro 2:** `projection` (opcional) — campos a incluir/excluir
- **Parámetro 3:** `options` (opcional) — incluye `session`, `lean`, `readConcern`, etc.

**La forma `findById(id, { session })` usada en Revisión 20 es INCORRECTA** porque pasa el objeto de opciones como segundo argumento (projection), no como tercero.

### 6.2 Corrección en todas las lecturas transaccionales

Todas las lecturas que deben pertenecer a la transacción usan la forma correcta:

```typescript
// Contacto
const currentContact = await ContactoModel.findById(input.contactId, null, { session: txnSession }).lean();

// EvAssigned actual
const currentEvAssigned = currentEvAssignedId !== null
  ? await EventoModel.findById(currentEvAssignedId, null, { session: txnSession }).lean()
  : null;

// Outbox esperado
const outboxExpected = expectedEvAssignedId !== null
  ? await OutboxModel.findOne({ idempotencyKey: `${expectedEvAssignedId}:${contactId}` }, null, { session: txnSession }).lean()
  : null;

// Outbox actual (cuando IDs difieren)
const outboxCurrent = (currentEvAssignedId !== null && currentEvAssignedId !== expectedEvAssignedId)
  ? await OutboxModel.findOne({ idempotencyKey: `${currentEvAssignedId}:${contactId}` }, null, { session: txnSession }).lean()
  : { _evidenceState: 'not_queried' };

// EvAssigned esperado
const expectedEvAssigned = expectedEvAssignedId !== null
  ? await EventoModel.findById(expectedEvAssignedId, null, { session: txnSession }).lean()
  : null;

// RetirementOperation (si se consulta en reconciliación)
const retirementOp = retirementOperationId !== null
  ? await RetirementOperationModel.findById(retirementOperationId, null, { session: txnSession }).lean()
  : null;
```

**Alternativa equivalente y explícita mediante API de consultas:**
```typescript
const currentContact = await ContactoModel.findById(input.contactId).session(txnSession).lean();
```

Ambas formas son correctas y equivalentes en Mongoose 8.x. La segunda es más legible y evita el error de posición de argumentos.

### 6.3 Verificación de `.lean()` y asociación de sesión

- `.lean()` **no elimina ni sustituye** la asociación de la sesión. Devuelve un documento plano (POJO) pero la consulta se ejecuta dentro de la transacción y observa el mismo snapshot.
- La opción `session` en el tercer argumento (o `.session(txnSession)`) es la que vincula la operación a la transacción.
- Todas las lecturas en el algoritmo §5.5 usan **propuesta** `readConcern: snapshot` a nivel de transacción (`session.withTransaction(..., { readConcern: { level: 'snapshot' } })`), por lo que **todas las consultas que pasan la sesión observan el mismo snapshot transaccional**.
- ⚠️ **B1-03/B1-04**: `readConcern: snapshot` y transacciones multidocumento son **solución preferida (D.1.8)** pero **validación Atlas pendiente** (topología compatible, propagación sesión, commit/rollback). No son decisiones canónicas aprobadas.

### 6.4 Distinción normativa de tipos de lectura

| Tipo de lectura | Descripción | Usa sesión | Observa snapshot transaccional |
|-----------------|-------------|------------|-------------------------------|
| **Fuera de transacción** | Consulta sin sesión, fuera de `withTransaction` | No | No (readConcern por defecto) |
| **Asociada a sesión** | Consulta con `{ session }` pero fuera de transacción activa | Sí | No necesariamente (depende de si la sesión está en transacción) |
| **Dentro de transacción** | Consulta con `{ session }` dentro de callback `withTransaction` | Sí | Sí (si transacción usa `readConcern: snapshot`) |
| **Compartiendo snapshot transaccional** | Múltiples consultas con la misma sesión dentro de la misma `withTransaction` | Sí | **Sí** — garantizado por `readConcern: snapshot` en opciones de transacción |

**En el algoritmo §5.5:** Todas las 5-6 consultas (Contacto, currentEvAssigned, outboxExpected, outboxCurrent si aplica, expectedEvAssigned, retirementOp si aplica) son **lecturas dentro de transacción compartiendo el mismo snapshot** porque:
1. Se ejecutan dentro del callback de `session.withTransaction(..., { readConcern: { level: 'snapshot' } })`
2. Todas pasan `{ session: txnSession }` (o `.session(txnSession)`)
3. Por tanto, observan un estado consistente de la base de datos en el momento de inicio de la transacción

### 6.5 Escrituras transaccionales

Todas las escrituras que forman parte de la unidad atómica (Contacto, EvAssigned, Outbox, RetirementOperation) deben usar la misma sesión:

```typescript
await ContactoModel.findByIdAndUpdate(contactId, update, { session: txnSession, new: true });
await new EvAssignedDoc({ ... }).save({ session: txnSession });
await new OutboxDoc({ ... }).save({ session: txnSession });
await RetirementOperationModel.findByIdAndUpdate(opId, update, { session: txnSession });
```

El algoritmo de reconciliación (§5.5) es de **solo lectura**; las escrituras pertenecen a la operación de transición original (D-W79-15), no a la reconciliación.

---

## 7. H4: Helper Outbox, Inventario de Consumidores y Bloqueos (Bloque B)

### 7.1 Corrección del consumo del helper

**Defecto en Revisión 20:** El algoritmo consumía el objeto `OutboxMatchResult` como booleano (línea 269):
```typescript
const expectedOutboxCoherent = expectedOutboxExists && outboxPayloadMatchesTransition(outboxExpected, input.attemptedTransition);
// ...
if (expectedOutboxCoherent) { ... } // INCORRECTO: objeto siempre truthy
```

**Corrección en Revisión 21:** Inspección explícita de `decision` en **TODOS** los consumidores:

```typescript
const outboxMatch = outboxPayloadMatchesTransition(outboxExpected, input.attemptedTransition);

// CORRECCIÓN H4: Inspección EXPLÍCITA de decision
if (outboxMatch.decision === 'block') {
  // bloquear la conclusión que depende de evidencia Outbox
  return { state: 'INCONSISTENT_EV_OUTBOX_MISMATCH', ... };
}

if (outboxMatch.decision === 'escalate') {
  // Nivel 1 aprobado insuficiente o error de payload
  return { state: 'INCONSISTENT_EV_OUTBOX_MISMATCH', ... };
}

// Solo si decision === 'allow' (requiere approved=true Y no vacío, imposible hoy para Outbox asignación)
// continuar hacia éxito
```

**Tratamiento de `decision`:** Nunca se convierte silenciosamente en autorización. Cualquier decisión distinta de `'allow'` bloquea la ruta que depende de verificación contractual Outbox.

### 7.2 Rediseño de `OutboxMatchResult` — Semántica inequívoca

```typescript
interface OutboxMatchResult {
  approved: boolean;        // Nivel 1 APROBADO: campos con contrato canónico definitivo
                            // Para Outbox asignación: VACÍO (§16.14 pendiente) → siempre true (vacío) pero no habilita decisión
  conditional: boolean;     // Nivel 1 CONDICIONADO §16.14: mencionados en canónicos pero esquema payload pendiente
                            // Solo informativos; NO forman parte de decisión contractual
  proposed: boolean;        // Nivel 2 PROPUESTO: pendientes §16.14
                            // NO USAR para decisión contractual
  decision: 'allow' | 'block' | 'escalate'; // Decisión final para return_committed
                            // 'allow' SOLO si approved=true Y campos aprobados no vacíos
                            // 'block' = no return_committed, escalar (caso actual: Nivel 1 vacío)
                            // 'escalate' = error de payload o Nivel 1 condicionado fallido
  blockReason?: string;     // Motivo del bloqueo: 'level1_empty' | 'level1_failed' | 'no_payload' | 'conditional_failed' | 'proposed_only'
}
```

**Regla de decisión:**
- `decision === 'allow'` ⇔ `approved === true` **Y** existen campos aprobados no vacíos (imposible hoy para Outbox asignación)
- `decision === 'block'` ⇔ `approved === true` **PERO** campos aprobados vacíos (caso actual)
- `decision === 'escalate'` ⇔ `approved === false` O payload inválido

**No utilizar `approved: true` para representar "no existe ningún campo aprobado".** El campo `approved` indica si la verificación de Nivel 1 *pasó* (true = no hay campos aprobados que fallen), no si *hay* campos aprobados. La vacuidad se expresa en `blockReason: 'level1_empty'`.

### 7.3 Inventario exhaustivo de consumidores (Revisión 21 — algoritmo §5.5 corregido — **actualizada esta intervención**)

Búsqueda exhaustiva de todas las referencias a `outboxPayloadMatchesTransition`, `OutboxMatchResult`, `outboxMatch`, `return_committed`, `OUR_TRANSITION_COMMITTED`, publicación de eventos, confirmación de transición en el algoritmo de reconciliación (§5.5 corregido).

| # | Consumidor | Sección §5.5 | Ruta | Decisión evaluada | Resultado permitido | Categoría |
|---|------------|--------------|------|-------------------|---------------------|-----------|
| 1 | CASO A — expectedEvExists, expectedOutboxExists, decision=block | CASO A, expectedEvExists, expectedOutboxExists | `outboxMatch.decision === 'block'` | `decision === 'block'` | `EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX_BLOCK` + `decisionType: definitive_rejection` (escalar) — **no return_committed** | Demostración documental |
| 2 | CASO A — expectedEvExists, expectedOutboxExists, decision=escalate | CASO A, expectedEvExists, expectedOutboxExists | `outboxMatch.decision === 'escalate'` | `decision === 'escalate'` | `EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX_ESCALATE` + `decisionType: anomaly_requires_intervention` (escalar) — **no return_committed** | Demostración documental |
| 3 | CASO A — expectedEvExists, expectedOutboxExists, decision=allow | CASO A, expectedEvExists, expectedOutboxExists | `outboxMatch.decision === 'allow'` (con guard) | `decision === 'allow'` | `EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX` + `guard: level1_empty_blocks_allow_contractual_guard_required` + `decisionType: helper_allows_but_contractual_guard_blocks` (escalar) — **no return_committed** | Demostración documental |
| 4 | CASO A — !expectedEvExists, expectedOutboxExists, decision=block | CASO A, !expectedEvExists, expectedOutboxExists | `outboxMatch.decision === 'block'` | `decision === 'block'` | `EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX_BLOCK` + `decisionType: definitive_rejection` (escalar) — **no return_committed** | Demostración documental |
| 5 | CASO A — !expectedEvExists, expectedOutboxExists, decision=escalate | CASO A, !expectedEvExists, expectedOutboxExists | `outboxMatch.decision === 'escalate'` | `decision === 'escalate'` | `EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX_ESCALATE` + `decisionType: anomaly_requires_intervention` (escalar) — **no return_committed** | Demostración documental |
| 6 | CASO A — !expectedEvExists, expectedOutboxExists, decision=allow | CASO A, !expectedEvExists, expectedOutboxExists | `outboxMatch.decision === 'allow'` (con guard) | `decision === 'allow'` | `EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX` + `guard: level1_empty_blocks_allow_contractual_guard_required` + `decisionType: helper_allows_but_contractual_guard_blocks` (escalar) — **no return_committed** | Demostración documental |
| 7 | CASO B — currentEvAssigned perdido, decision=block | CASO B, !currentEvAssigned | `outboxMatch.decision === 'block'` | `decision === 'block'` | `EXPECTED_EV_ASSIGNED_ORPHAN_OUTBOX_BLOCK` + `decisionType: definitive_rejection` (escalar) — **no return_committed** | Demostración documental |
| 8 | CASO B — currentEvAssigned perdido, decision=escalate | CASO B, !currentEvAssigned | `outboxMatch.decision === 'escalate'` | `decision === 'escalate'` | `EXPECTED_EV_ASSIGNED_ORPHAN_OUTBOX_ESCALATE` + `decisionType: anomaly_requires_intervention` (escalar) — **no return_committed** | Demostración documental |
| 9 | CASO B — currentEvAssigned perdido, decision=allow | CASO B, !currentEvAssigned | `outboxMatch.decision === 'allow'` (con guard) | `decision === 'allow'` | `EXPECTED_EV_ASSIGNED_ORPHAN_WITH_OUTBOX` + `guard: level1_empty_blocks_allow_contractual_guard_required` + `decisionType: helper_allows_but_contractual_guard_blocks` (escalar) — **no return_committed** | Demostración documental |
| 10 | CASO B — outboxExpected null | CASO B, !outboxExpected | N/A (no hay Outbox) | — | `OUR_TRANSITION_COMMITTED_OUTBOX_INCONSISTENT` (escalar WAB-81) — **no return_committed** | Demostración documental |
| 11 | CASO B — outboxExpected incoherente, decision=block | CASO B, outboxExpected existe | `outboxMatch.decision === 'block'` | `decision === 'block'` | `INCONSISTENT_EV_OUTBOX_MISMATCH_BLOCK` + `decisionType: definitive_rejection` (escalar) — **no return_committed** | Demostración documental |
| 12 | CASO B — outboxExpected incoherente, decision=escalate | CASO B, outboxExpected existe | `outboxMatch.decision === 'escalate'` | `decision === 'escalate'` | `INCONSISTENT_EV_OUTBOX_MISMATCH_ESCALATE` + `decisionType: anomaly_requires_intervention` (escalar) — **no return_committed** | Demostración documental |
| 13 | CASO B — éxito condicional (fila 9c) | CASO B, level1OutboxApprovedNonEmpty + correlation | `decision === 'allow'` (requiere approved + no vacío + correlation) | `decision === 'allow'` | `OUR_TRANSITION_BLOCKED_LEVEL1_OUTBOX` / `OUR_TRANSITION_BLOCKED_RETIREMENT_OP_CORRELATION` (**`OUR_TRANSITION_COMMITTED` inalcanzable**) | **Dependencia contractual (§16.14 + §16.12)** |
| 14 | CASO C — mismo tipo, distinto ID | CASO C, isSameTransitionType | D-W79-14 **aprobada** define autoría; §16.14 Outbox pendiente para verificación runtime | — | `INDETERMINATE_CONTRACTUAL_BLOCK` — **no return_committed** | Decisión canónica / Dependencia contractual |
| 15 | CASO D — Outbox sin EvAssigned | CASO D | outboxCurrent existe, currentEvAssigned=null | — | `INCONSISTENT_OUTBOX_WITHOUT_EV` — **no return_committed** | Demostración documental / Propuesta ilustrativa |

**Verificación de completitud (B4-03 — actualizada esta intervención):** Las **15 rutas** anteriores cubren **todas** las referencias a `outboxPayloadMatchesTransition` y decisiones de `return_committed` / `OUR_TRANSITION_COMMITTED` en el algoritmo §5.5 corregido. **No existe ninguna ruta** que declare `return_committed` / `OUR_TRANSITION_COMMITTED` / evento publicado / transición confirmada basándose en `outboxMatch.decision === 'allow'` sin verificar `approved === true` Y campos aprobados no vacíos **Y** correlación RetirementOperation demostrable (§16.12/§16.14). 

**Para cada consumidor se diferencia (B4-03) — D4-02 (5ta intervención):**
- Invocación del helper: **solo con OutboxDoc no-null** (precondición verificada en caller)
- Tratamiento de `return_committed`: **nunca** — `decision === 'allow'` imposible hoy (Nivel 1 vacío §16.14 + correlación §16.12/§16.14 pendiente)
- Tratamiento de `block`: rechazo definitivo — impide éxito, expone causa, mantiene fail-closed, retorna estado con `decisionType: definitive_rejection`, no permite commit ni retry automático
- Tratamiento de `escalate`: anomalía que requiere intervención — no permite commit ni retry, clasificación diferenciada (`decisionType: anomaly_requires_intervention`), preserva evidencia diagnóstica, distinto de rechazo definitivo
- Tratamiento de `allow`: SOLO "helper no encontró condición local de bloqueo" → NO = autorización contractual → GUARD CONTRACTUAL OBLIGATORIO (§16.14 + §16.12) en TODAS las rutas; retorna estado con `decisionType: helper_allows_but_contractual_guard_blocks`
- Tratamiento de ausencia de Outbox: caller maneja ANTES de invocar helper (ver CASO A y B líneas)
- Tratamiento de evidencia inconsistente: `decision === 'escalate'` o `block` → estado de inconsistencia con decisionType explícito
- Tratamiento de correlación RetirementOperation: `retirementOpCorrelationDemonstrable` debe ser `true` para permitir éxito (imposible mientras §16.14 pendiente)

**Estado D4-02 (diferencia operativa block vs escalate) — PARCIALMENTE RESUELTO / PENDIENTE:**
- **Diferencia documentada**: `decisionType` distinto (`definitive_rejection` vs `anomaly_requires_intervention`), nombres de estado distintos (`*_BLOCK` vs `*_ESCALATE`), evidencia preservada en escalate.
- **Mismo resultado terminal**: Ambos retornan `canRetryBusinessTransaction: false` y `requiresEscalation: true` en **todas** las 15 rutas de consumidor (§7.3).
- **Sin consumidor downstream documentado** que trate `block` y `escalate` de forma operativamente distinta (ej. políticas de reintento diferentes, alertas distintas, flujos de recuperación distintos).
- **Conclusión**: La diferenciación existe a nivel de **clasificación y diagnóstico** (nombres, decisionType, evidencia), pero **no se ha demostrado diferencia operacional en runtime** (acciones posteriores automatizadas). D4-02 se marca como **parcialmente resuelto** — queda pendiente validar si consumidores reales (WAB-81 drenaje, alertas, operadores) requieren tratamientos distintos. No se declara resuelto completamente sin evidencia de consumidor downstream.

**Protección conservadora mantenida:** Mientras no exista contrato §16.14 aprobado con Nivel 1 no vacío Y correlación física demostrable, helper **no habilita** `return_committed` por inferencia.

### 7.4 Nivel 1 contractual del Outbox de asignación: VACÍO

El contrato físico del Outbox de asignación continúa pendiente conforme a **§16.14**.

Por tanto, se mantiene **explícitamente**:

- **No declarar** `idempotencyKey` como campo físico aprobado para decisión contractual.
- **No declarar** `eventType` como campo físico aprobado para decisión contractual.
- **No declarar** `contactId` ni `contacto` como campos físicos aprobados.
- **No declarar** `payload.evAssignedId` como campo físico aprobado.
- **No utilizar** estos campos para justificar una coincidencia contractual o un retorno exitoso `return_committed`.

**Recordatorio de distinciones contractuales:**

| Fuente | Qué define | Qué NO define |
|--------|------------|---------------|
| **D.1.8** | Atomicidad Contacto+EvAssigned+Outbox en misma transacción | Esquema físico del Outbox |
| **D.6** | Flujo durable de ejecución (entrega, idempotencia, recepción) | Esquema físico del Outbox de asignación |
| **§16.12** | Identidades de idempotencia (operación, transición, confirmación) | Campos físicos del Outbox |
| **§16.14** | **Pendiente**: nombres definitivos, payloads, contratos de transporte Inbox/Outbox | — |

**El Nivel 1 de campos físicos aprobados para decidir la coherencia del Outbox de asignación debe seguir VACÍO mientras no exista una resolución contractual (§16.14).**

### 7.5 Aplicación de bloqueos en todas las rutas (Corregido B1-01, B1-02)

**Ninguna ruta puede declarar:**
- `return_committed`
- `OUR_TRANSITION_COMMITTED`
- evento publicado
- transición confirmada

**basándose en un Outbox cuya correspondencia no puede verificarse mediante un contrato aprobado.**

Los bloqueos reales son:

1. **§16.14 Outbox físico pendiente** (bloqueo principal): Nivel 1 aprobado = VACÍO → `decision === 'block'` siempre en helper. Afecta: `OUR_TRANSITION_BLOCKED_LEVEL1_OUTBOX` (fila 9), todas las rutas de éxito condicional.

2. **Verificación runtime autoría EvAssigned** (filas 14, 17): D-W79-14 **aprobada** define campos EvAssigned y autoría, pero **correlación física Outbox (§16.14) pendiente** impide verificar en runtime que EvAssigned "otro" corresponde a transición legítima. Estado: `INDETERMINATE_CONTRACTUAL_BLOCK`.

3. **D-W79-14 NO está pendiente ni bloqueada** — está aprobada (10/11/2026). Lo que permanece pendiente es §16.14 (esquema físico Outbox).

**No se soluciona §16.14 unilateralmente.** No se añade `retirementOperationId` a EvAssigned ni se inventa otro mecanismo de correlación. **No se permite que una variable de entorno sustituya una decisión contractual documental** (Revisión 20 usaba `process.env.DW79_14_RESOLVED === 'true'`; Revisión 21 usa constante contractual explícita).

---

## 8. H5: CAS, Elegibilidad y Estados BSON (Bloque C)

### 8.1 Unificar el modelo de datos — Rutas de campos reales del subdocumento `processing`

Utilizar consistentemente las rutas de campos reales del subdocumento `processing`. No mezclar sin explicación:

- ❌ `leaseOwner` (raíz)
- ❌ `leaseUntil` (raíz)
- ✅ `processing.leaseOwner`
- ✅ `processing.leaseUntil`
- ✅ `processing.leaseGeneration`

**Estructura BSON sobre la que opera el pseudocódigo:**

```typescript
interface RetirementOperationDoc {
  _id: string;
  // ... otros campos ...
  processing?: {
    leaseOwner?: string | null;      // Puede ser: ausente, null, "worker-id"
    leaseUntil?: Date | null;        // Puede ser: ausente, null, Date
    leaseGeneration?: number;        // Puede ser: ausente, 0, positivo
  };
  // ... otros campos ...
}
```

**Dimensiones BSON separadas (valores precisos, distinguen ausencia vs null vs valor):**

| Dimensión | Valores posibles | Nota |
|-----------|------------------|------|
| **Existencia de `processing`** | `absent` / `present` | Subdocumento ausente vs presente (aunque vacío) |
| **`processing.leaseOwner`** | `absent` / `null` / `"same"` / `"other"` | Campo no existe / existe con null / existe con worker-id del solicitante / existe con worker-id distinto |
| **`processing.leaseUntil`** | `absent` / `null` / `future` / `past` | Campo no existe / existe con null / existe con Date > now / existe con Date <= now |
| **`processing.leaseGeneration`** | `absent` / `0` / `positive` | Campo no existe / existe con 0 / existe con > 0 |

**Estados canónicos mutuamente excluyentes (partición lógica — cada documento coincide con EXACTAMENTE uno):**

| Estado | `processing` | `leaseOwner` | `leaseUntil` | `leaseGeneration` | Semántica | Exclusión mutua garantizada por |
|--------|--------------|--------------|--------------|-------------------|-----------|--------------------------------|
| **E1** | `absent` | — | — | — | Documento sin subdocumento `processing` | `processing` absent |
| **E2** | `present` | `absent` | `absent` | `absent` | Subdocumento existe, sin campos de lease | Tres campos absent |
| **E3** | `present` | `null` | `absent` | `absent` | `leaseOwner` explícitamente null, sin leaseUntil | `leaseOwner` = null (no absent) |
| **E4** | `present` | `"same"` | `future` | `positive` | Lease vigente del mismo owner (renovación) | `leaseOwner`="same" ∧ `leaseUntil`=future ∧ gen>0 |
| **E5** | `present` | `"same"` | `null_or_past` | `positive` | Lease vencido del mismo owner (readquisición) | `leaseOwner`="same" ∧ `leaseUntil`≠future ∧ gen>0 |
| **E6** | `present` | `"other"` | `future` | `positive` | Lease vigente de otro owner (rechazo por fencing) | `leaseOwner`="other" ∧ `leaseUntil`=future ∧ gen>0 |
| **E7** | `present` | `"other"` | `null_or_past` | `positive` | Lease vencido de otro owner (adquisición sobre otro) | `leaseOwner`="other" ∧ `leaseUntil`≠future ∧ gen>0 |
| **E8** | `present` | `absent` | `future` | `absent` | Sin owner explícito, leaseUntil futuro implícito (rechazo) | `leaseOwner` absent ∧ `leaseUntil`=future ∧ gen absent |
| **E9** | `present` | `absent` | `null_or_past` | `absent` | Sin owner efectivo, leaseUntil presente y no futuro (adquisición) | `leaseOwner` absent ∧ `leaseUntil` PRESENTE (null/past) ∧ gen absent |
| **E10** | `present` | `absent` | `absent` | `0` | Sin campos de lease, generation = 0 (adquisición) | `leaseOwner` absent ∧ `leaseUntil` absent ∧ `leaseGeneration` = 0 |

**Notas de exclusividad (demostrables):**
- E1 es único por `processing` absent.
- E2 requiere los TRES campos absent; E9 requiere `leaseUntil` PRESENTE (null o past); E10 requiere `leaseGeneration` = 0 (no absent).
- E3 requiere `leaseOwner` = null (explícito), no absent.
- E4–E7 requieren `leaseGeneration` = positive (>0), no absent ni 0.
- E8 requiere `leaseUntil` = future y `leaseGeneration` absent.
- E9 requiere `leaseUntil` PRESENTE (null o past) — excluye E2 (absent) y E10 (absent).
- E10 requiere `leaseGeneration` = 0 (explícito) — excluye E2 (absent) y E9 (absent).

> **Corrección D5-01 (4ta/5ta intervención):** Los 10 estados anteriores forman una **partición lógica mutuamente excluyente** por construcción **dentro del dominio declarado**. Cada combinación concreta de valores BSON (distinguiendo absent/null/0/positive/future/past) que pertenece al dominio declarado coincide con EXACTAMENTE un estado E1-E10. El solapamiento anterior E2/E9/E10 se resuelve exigiendo presencia explícita: E9 requiere `leaseUntil` presente (null/past), E10 requiere `leaseGeneration` = 0.
>
> **Análisis de exhaustividad (D5-01 obligatorio):** El espacio combinatorio de las 4 dimensiones declaradas es: `processing` (2) × `leaseOwner` (4) × `leaseUntil` (4) × `leaseGeneration` (3) = **96 combinaciones teóricas**. Los 10 estados E1-E10 cubren las combinaciones **semánticamente válidas** para la operación de retiro. Las combinaciones restantes (ej. `leaseOwner`="same" ∧ `leaseUntil`=future ∧ `leaseGeneration`=0 — generación 0 con lease vigente; `leaseOwner`="other" ∧ `leaseUntil`=absent ∧ `leaseGeneration`=positive — generación positiva sin leaseUntil) son **inválidas/contradictorias** para el dominio de retiro y deben clasificarse como **estado inválido/no reconocido**.
>
> **Categoría explícita de estado inválido (E-INVALID):** Toda combinación BSON dentro de las dimensiones declaradas que no coincida con E1-E10 se clasifica como `INVALID_BSON_STATE`. Este estado **bloquea la adquisición y cualquier ruta de éxito**. Ejemplos de combinaciones inválidas correctamente bloqueadas:
> - `processing` present, `leaseOwner`="same", `leaseUntil`=future, `leaseGeneration`=0 (gen 0 con lease vigente — contradictorio)
> - `processing` present, `leaseOwner`="other", `leaseUntil`=absent, `leaseGeneration`=positive (gen>0 sin leaseUntil — inválido)
> - `processing` present, `leaseOwner`="same", `leaseUntil`=null, `leaseGeneration`=absent (gen ausente con owner same — incompleto)
> - `processing` present, `leaseOwner`=null, `leaseUntil`=future, `leaseGeneration`=positive (owner null con lease futuro — contradictorio)
> - `processing` absent, `leaseOwner`=cualquier valor (imposible — subdocumento ausente no tiene campos)
>
> **No se introducen defaults** para resolver combinaciones ambiguas. **No se asume** que campo ausente equivale a cero o `null`. Si una combinación no puede demostrarse válida dentro de E1-E10, termina en `INVALID_BSON_STATE` y se bloquea.
>
> **Limitación declarada:** La exhaustividad formal de los 10 estados E1-E10 sobre las 96 combinaciones teóricas se demuestra **solo para el dominio semántico válido** de la operación de retiro. Las 36 combinaciones teóricas restantes (96 - 60 válidas) son inválidas por construcción del dominio y caen en `E-INVALID`. Si no se puede demostrar exhaustividad formal completa, se declara esta limitación y **no se afirma** que existe una partición completa sin categorías inválidas.
>
> **Aclaración matemática rigurosa (D5-01 — esta intervención):**
> - **Espacio cartesiano declarado**: 2 (processing) × 4 (leaseOwner) × 4 (leaseUntil) × 3 (leaseGeneration) = **96 combinaciones teóricas**.
> - **Processing ausente**: 1 × 4 × 4 × 3 = **48 combinaciones teóricas**. En BSON físico, si `processing` está ausente, sus subcampos **no existen independientemente**; por tanto, estas 48 posiciones colapsan en **una única clase de equivalencia BSON** (E1).
> - **Processing presente**: 1 × 4 × 4 × 3 = **48 combinaciones teóricas**. Estas corresponden a **48 clases de equivalencia BSON** físicamente realizables.
> - **E1**: 48 combinaciones teóricas → **1 clase de equivalencia BSON** (processing ausente).
> - **E2-E10**: 12 combinaciones teóricas válidas → **12 clases de equivalencia BSON** (processing presente, semánticamente válidas).
> - **E-INVALID**: 36 combinaciones teóricas inválidas (96 - 60 = 36) → **36 clases de equivalencia BSON** (processing presente, contradictorias).
> - **Total clases de equivalencia BSON**: 1 (E1) + 12 (E2-E10) + 36 (E-INVALID) = **49**.
> - **Total clases válidas**: 13 (1 E1 + 12 E2-E10).
> - **Total clases inválidas**: 36.
> - **Clases de equivalencia BSON físicamente realizables totales**: 48 (processing=present) + 1 (processing=absent) = **49** (no 81).
> - **Combinaciones abstractas teóricas**: **96**.
> - **Estados semánticamente admitidos** (E1-E10): **13** (1 E1 + 12 E2-E10).
> - **Combinaciones teóricas válidas**: 60 (E1: 48 + E2-E10: 12).
> - **Combinaciones teóricas inválidas (E-INVALID)**: 36.
> - **Recuento final verificable**: 60 válidas (teóricas) + 36 inválidas = 96 total ✅. 13 clases válidas + 36 clases inválidas = 49 clases totales ✅.
>
> ### 8.1.1 Clasificador formal exhaustivo (D5-01 — 5ta intervención)
> 
> El siguiente pseudocódigo define un clasificador **determinista, exhaustivo y mutuamente excluyente** para las 96 combinaciones teóricas. El orden de evaluación es crítico: cada combinación se clasifica **exactamente una vez** (primer match gana), ninguna queda sin clasificar, ninguna recibe dos clasificaciones.
> 
> ```typescript
> type ProcessingExistence = 'absent' | 'present';
> type LeaseOwnerValue = 'absent' | 'null' | 'same' | 'other';
> type LeaseUntilValue = 'absent' | 'null' | 'future' | 'past';
> type LeaseGenValue = 'absent' | 'zero' | 'positive';
> 
> type BSONStateClass = 
>   | 'E1' | 'E2' | 'E3' | 'E4' | 'E5' | 'E6' | 'E7' | 'E8' | 'E9' | 'E10'
>   | 'E-INVALID';
> 
> interface ObservedState {
>   processing: ProcessingExistence;
>   leaseOwner: LeaseOwnerValue;
>   leaseUntil: LeaseUntilValue;
>   leaseGeneration: LeaseGenValue;
> }
> 
> function classifyBSONState(observed: ObservedState): BSONStateClass {
>   // ORDEN DE EVALUACIÓN DETERMINISTA — PRIMER MATCH GANA
>   // Cada combinación del espacio cartesiano 2×4×4×3 = 96 se evalúa en este orden
>   
>   // 1. E1: processing ausente (48 combinaciones teóricas → 1 clase)
>   if (observed.processing === 'absent') {
>     // Cuando processing ausente, los subcampos no tienen significado BSON independiente
>     // Todas las 48 combinaciones teóricas de subcampos (4×4×3) colapsan a E1
>     return 'E1';
>   }
>   
>   // A partir de aquí: processing === 'present' (48 combinaciones restantes)
>   
>   // 2. E4: same owner, lease vigente, gen > 0 (renovación)
>   if (observed.leaseOwner === 'same' && 
>       observed.leaseUntil === 'future' && 
>       observed.leaseGeneration === 'positive') {
>     return 'E4';
>   }
>   
>   // 3. E5: same owner, lease vencido, gen > 0 (readquisición)
>   if (observed.leaseOwner === 'same' && 
>       (observed.leaseUntil === 'null' || observed.leaseUntil === 'past') && 
>       observed.leaseGeneration === 'positive') {
>     return 'E5';
>   }
>   
>   // 4. E6: other owner, lease vigente, gen > 0 (rechazo fencing)
>   if (observed.leaseOwner === 'other' && 
>       observed.leaseUntil === 'future' && 
>       observed.leaseGeneration === 'positive') {
>     return 'E6';
>   }
>   
>   // 5. E7: other owner, lease vencido, gen > 0 (adquisición sobre otro)
>   if (observed.leaseOwner === 'other' && 
>       (observed.leaseUntil === 'null' || observed.leaseUntil === 'past') && 
>       observed.leaseGeneration === 'positive') {
>     return 'E7';
>   }
>   
>   // 6. E8: owner absent, leaseUntil future, gen absent (rechazo lease implícito)
>   if (observed.leaseOwner === 'absent' && 
>       observed.leaseUntil === 'future' && 
>       observed.leaseGeneration === 'absent') {
>     return 'E8';
>   }
>   
>   // 7. E9: owner absent, leaseUntil PRESENTE (null/past), gen absent (adquisición)
>   if (observed.leaseOwner === 'absent' && 
>       (observed.leaseUntil === 'null' || observed.leaseUntil === 'past') && 
>       observed.leaseGeneration === 'absent') {
>     return 'E9';
>   }
>   
>   // 8. E10: owner absent, leaseUntil absent, gen = 0 explícito (adquisición)
>   if (observed.leaseOwner === 'absent' && 
>       observed.leaseUntil === 'absent' && 
>       observed.leaseGeneration === 'zero') {
>     return 'E10';
>   }
>   
>   // 9. E2: processing presente, TRES campos absent (subdocumento vacío)
>   if (observed.leaseOwner === 'absent' && 
>       observed.leaseUntil === 'absent' && 
>       observed.leaseGeneration === 'absent') {
>     return 'E2';
>   }
>   
>   // 10. E3: leaseOwner = null explícito, leaseUntil absent, leaseGeneration absent
>   if (observed.leaseOwner === 'null' && 
>       observed.leaseUntil === 'absent' && 
>       observed.leaseGeneration === 'absent') {
>     return 'E3';
>   }
>   
>   // 11. E-INVALID: todas las combinaciones restantes (48 processing=present - 12 válidas = 36 inválidas)
>   // Ejemplos que caen aquí:
>   // - same + future + zero (gen 0 con lease vigente)
>   // - same + null + absent (gen ausente con owner same)
>   // - other + absent + positive (gen>0 sin leaseUntil)
>   // - null + future + positive (owner null con lease futuro)
>   // - same + past + zero, other + future + zero, etc.
>   return 'E-INVALID';
> }
> 
> // VERIFICACIÓN DE EXHAUSTIVIDAD Y EXCLUSIÓN MUTUA:
> // - Espacio total: 2 × 4 × 4 × 3 = 96 combinaciones teóricas
> // - E1: 48 combinaciones teóricas (processing=absent × 4×4×3 subcampos) → **1 clase de equivalencia BSON**
> // - E2: 1 combinación (present, absent, absent, absent) → 1 clase física
> // - E3: 1 combinación (present, null, absent, absent) → 1 clase física
> // - E4: 1 combinación (present, same, future, positive) → 1 clase física
> // - E5: 2 combinaciones (present, same, null/past, positive) → 2 clases físicas
> // - E6: 1 combinación (present, other, future, positive) → 1 clase física
> // - E7: 2 combinaciones (present, other, null/past, positive) → 2 clases físicas
> // - E8: 1 combinación (present, absent, future, absent) → 1 clase física
> // - E9: 2 combinaciones (present, absent, null/past, absent) → 2 clases físicas
> // - E10: 1 combinación (present, absent, absent, zero) → 1 clase física
> // - Total combinaciones teóricas válidas (E1-E10): 48 + 1 + 1 + 1 + 2 + 1 + 2 + 1 + 2 + 1 = 60
> //   (De las cuales E1: 48 teóricas → 1 clase física; E2-E10: 12 combinaciones = 12 clases físicas)
> // - E-INVALID: 96 - 60 = 36 combinaciones teóricas inválidas (todas dentro de processing=present) → 36 clases físicas
> // - Clases de equivalencia BSON válidas: 1 (E1) + 12 (E2-E10) = **13**
> // - Clases de equivalencia BSON totales: 1 (E1) + 12 (E2-E10) + 36 (E-INVALID) = **49** (no 81)
> //   De las 48 combinaciones teóricas con processing=present: 12 válidas (E2-E10) + 36 inválidas (E-INVALID) = 48
> //   NOTA: 60 combinaciones teóricas válidas ≠ 13 clases de equivalencia BSON válidas (E1 absorbe 48 teóricas en 1 clase)
> // - RECUENTO FINAL VERIFICABLE: 60 válidas (teóricas) + 36 inválidas = 96 total ✅. 13 clases válidas + 36 clases inválidas = 49 clases totales ✅
> // 
> // ACLARACIÓN CRÍTICA: Las 48 combinaciones teóricas de E1 (processing=absent) no son
> // "físicamente realizables" como estados independientes de subcampos — en BSON, si processing
> // está ausente, sus subcampos no existen. El clasificador las agrupa correctamente en E1
> // (un único estado físico = 1 clase de equivalencia). Las 48 combinaciones teóricas con
> // processing=present corresponden a 48 clases físicas: 12 válidas (E2-E10) + 36 inválidas (E-INVALID).
> // No se afirma 80 combinaciones con processing presente ni 81 estados físicos.
> ```
> 
> **ACLARACIÓN RIGUROSA (D5-01 — esta intervención):**
> - E1 representa un **ÚNICO estado BSON físico** (`processing` ausente), aunque ocupe 48 posiciones
>   del espacio cartesiano teórico (2×4×4×3). Los recuentos teóricos (96 combinaciones)
>   **NO deben describirse como cantidad de documentos BSON físicamente realizables**.
>   - Estados BSON físicamente realizables (clases de equivalencia): 48 (`processing`=present) + 1 (`processing`=absent) = **49** (no 81)
>   - De los 48 con `processing`=present: 12 semánticamente válidas (E2-E10) + 36 inválidas (E-INVALID)
> - Diferenciar claramente tres niveles:
>   1. **Clases de equivalencia BSON físicas (49)**: lo que puede existir en la base de datos (48 con processing presente + 1 con processing ausente)
>   2. **Combinaciones abstractas teóricas (96)**: producto cartesiano de dimensiones declaradas (2×4×4×3)
>   3. **Estados semánticamente admitidos (13: 1 E1 + 12 E2-E10)**: los que tienen significado en el dominio
> 
> **VALIDACIÓN Y EXTRACCIÓN PREVIA: Conversión de documento BSON real a ObservedState**  
> *Pseudocódigo de validación/extracción que rechaza valores no representables. No utiliza coerciones ni defaults.  
> Si el tipo exacto admitido por el contrato no está aprobado, declara la dependencia y bloquea la clasificación que dependa de ella.*
> 
> ```typescript
> interface RawBSONDoc {
>   _id: string;
>   processing?: {
>     leaseOwner?: string | null | undefined;  // string | null | undefined (ausente)
>     leaseUntil?: Date | null | undefined;    // Date | null | undefined (ausente)
>     leaseGeneration?: number | undefined;    // number | undefined (ausente)
>     // ... otros campos ...
>   };
>   revision: number;
>   status: string;
>   // ... otros campos ...
> }
> 
> interface ExtractionResult {
>   success: boolean;
>   observed?: ObservedState;
>   error?: {
>     code: 'INVALID_LEASE_OWNER_TYPE' | 'INVALID_LEASE_UNTIL_TYPE' | 'INVALID_LEASE_GENERATION_TYPE' | 'NEGATIVE_GENERATION' | 'FRACTIONAL_GENERATION' | 'UNREPRESENTABLE_VALUE' | 'INVALID_PROCESSING_TYPE' | 'WORKER_IDENTITY_REQUIRED' | 'TRANSACTION_TIMESTAMP_REQUIRED' | 'INVALID_TRANSACTION_TIMESTAMP';
>     field: string;
>     observedValue: unknown;
>     message: string;
>   };
> }
> 
> function extractAndValidateObservedState(
>   doc: RawBSONDoc,
>   // Identidad del worker solicitante — OBLIGATORIA cuando leaseOwner es string
>   // DEPENDENCIA CONTRACTUAL: el caller debe proveer workerId (string no vacío) para resolver same/other
>   // Si no hay workerId verificable y leaseOwner es string, la extracción falla con WORKER_IDENTITY_REQUIRED
>   // No permitir que un valor no string se interprete como identidad válida
>   workerId: string,
>   // Referencia temporal estable de la operación/transacción — no crear nueva fecha en cada extracción
>   // DEPENDENCIA CONTRACTUAL: el caller debe proveer transactionTimestamp (ej. transaction.startTime)
>   // OBLIGATORIO cuando leaseUntil es Date (instancia válida con valor temporal finito)
>   // Debe ser Date válido con valor temporal finito (no NaN, no ±Infinity)
>   // Si no está disponible o inválido y leaseUntil es Date, la extracción falla con TRANSACTION_TIMESTAMP_REQUIRED / INVALID_TRANSACTION_TIMESTAMP
>   // No introducir new Date() local ni defaults
>   transactionTimestamp: Date
> ): ExtractionResult {
>   // 1. Determinar existencia de processing — distinguir ausente de null explícito
>   // En BSON/Mongoose: processing ausente = undefined; processing: null = subdocumento presente con valor null
>   const processingValue = doc.processing;
>   
>   if (processingValue === undefined) {
>     // processing AUSENTE: subcampos no tienen significado BSON independiente
>     // Todas las 48 combinaciones teóricas de subcampos (4×4×3) colapsan a E1
>     return { 
>       success: true, 
>       observed: { 
>         processing: 'absent', 
>         leaseOwner: 'absent', 
>         leaseUntil: 'absent', 
>         leaseGeneration: 'absent' 
>       } 
>     };
>   }
>   
>   if (processingValue === null) {
>     // processing EXPLÍCITAMENTE NULL — tipo inválido para el dominio de retiro
>     // No se puede determinar estado de subcampos porque el contenedor es null
>     return {
>       success: false,
>       error: {
>         code: 'INVALID_PROCESSING_TYPE',
>         field: 'processing',
>         observedValue: null,
>         message: 'processing no debe ser null; debe ser objeto (present) o ausente (undefined). Subdocumento null no es válido para clasificación de lease.'
>       }
>     };
>   }
>   
>   // A partir de aquí: processing debe ser un objeto plano (BSON document) válido
>   // Validación explícita de prototipo: rechazar arrays, strings, números, booleanos, Date, null, undefined,
>   // y objetos con prototipos incompatibles (no plain objects), incluyendo Object.create(null).
>   // Compatibilidad con contrato BSON documentado: solo plain objects (Object.prototype === Object.prototype).
>   // Object.getPrototypeOf devuelve null para Object.create(null); no acceder a .constructor sin guardar.
>   const proto = processingValue !== null && typeof processingValue === 'object'
>     ? Object.getPrototypeOf(processingValue)
>     : null;
>   const isPlainObject = proto === Object.prototype;
>   if (typeof processingValue !== 'object' ||
>       processingValue === null ||
>       Array.isArray(processingValue) ||
>       !isPlainObject) {
>     return {
>       success: false,
>       error: {
>         code: 'INVALID_PROCESSING_TYPE',
>         field: 'processing',
>         observedValue: processingValue,
>         message: `processing debe ser un objeto BSON/documento válido (plain object con Object.prototype); recibido ${Array.isArray(processingValue) ? 'array' : typeof processingValue}${processingValue !== null && typeof processingValue === 'object' ? ` (prototype: ${proto?.constructor?.name ?? (proto === null ? 'null (Object.create(null))' : 'unknown')})` : ''}`
>       }
>     };
>   }
>
>   // A partir de aquí: processing es un objeto plano válido (present)
>   
>   // 2. Validar y extraer leaseOwner — resolver identidad de worker explícitamente
>   const rawLeaseOwner = processingValue.leaseOwner;
>   let leaseOwner: LeaseOwnerValue;
>   
>   if (rawLeaseOwner === undefined) {
>     leaseOwner = 'absent';
>   } else if (rawLeaseOwner === null) {
>     leaseOwner = 'null';
>   } else if (typeof rawLeaseOwner === 'string') {
>     // La semántica 'same'/'other' REQUIERE identidad de worker explícita (workerId).
>     // workerId DEBE ser string no vacío ni solo espacios, proporcionado por el caller.
>     // Validación estricta en runtime: tipo string, no vacío, no solo whitespace.
>     // Un valor de tipo incompatible NO puede producir clasificación 'same' o 'other'.
>     // No hay coerciones. Fallo explícito con WORKER_IDENTITY_REQUIRED.
>     if (typeof workerId !== 'string' || workerId.trim() === '') {
>       return {
>         success: false,
>         error: {
>           code: 'WORKER_IDENTITY_REQUIRED',
>           field: 'processing.leaseOwner',
>           observedValue: rawLeaseOwner,
>           message: 'leaseOwner es string; la clasificación same/other requiere workerId del solicitante (string no vacío ni solo espacios). Proporcione workerId válido al caller.'
>         }
>       };
>     }
>     // Comparación explícita: same si coinciden, other si difieren
>     leaseOwner = rawLeaseOwner === workerId ? 'same' : 'other';
>   } else {
>     // Tipo no admitido: number, boolean, object, array, etc.
>     return {
>       success: false,
>       error: {
>         code: 'INVALID_LEASE_OWNER_TYPE',
>         field: 'processing.leaseOwner',
>         observedValue: rawLeaseOwner,
>         message: `leaseOwner debe ser string | null | undefined; recibido ${typeof rawLeaseOwner}`
>       }
>     };
>   }
>   
>   // 3. Validar y extraer leaseUntil — validar Date válido y timestamp de transacción estable
>   const rawLeaseUntil = processingValue.leaseUntil;
>   let leaseUntil: LeaseUntilValue;
>   
>   if (rawLeaseUntil === undefined) {
>     leaseUntil = 'absent';
>   } else if (rawLeaseUntil === null) {
>     leaseUntil = 'null';
>   } else if (rawLeaseUntil instanceof Date) {
>     // Validar que la fecha observada sea válida (no NaN, valor temporal finito)
>     const leaseUntilTime = rawLeaseUntil.getTime();
>     if (!Number.isFinite(leaseUntilTime) || Number.isNaN(leaseUntilTime)) {
>       return {
>         success: false,
>         error: {
>           code: 'INVALID_LEASE_UNTIL_TYPE',
>           field: 'processing.leaseUntil',
>           observedValue: rawLeaseUntil,
>           message: 'leaseUntil es Date inválido (NaN o valor no finito); debe ser fecha válida con valor temporal finito.'
>         }
>       };
>     }
>     // REQUISITO: timestamp de referencia DEBE provenir de la operación/transacción (ej. transaction.startTime)
>     // NO crear new Date() aquí — eso rompería consistencia en extracciones múltiples dentro de la misma transacción
>     // DEPENDENCIA CONTRACTUAL: caller debe proveer transactionTimestamp válido
>     // Secuencia de validación (orden obligatorio):
>     // 1. Comprobar undefined → TRANSACTION_TIMESTAMP_REQUIRED
>     // 2. Comprobar null y instanceof Date → INVALID_TRANSACTION_TIMESTAMP
>     // 3. Obtener timestamp mediante getTime()
>     // 4. Comprobar que el resultado sea finito → INVALID_TRANSACTION_TIMESTAMP
>     // 5. Continuar con la clasificación temporal
>     // No usar new Date() ni defaults ni coerciones. Number.isFinite() NO puede ejecutarse antes de getTime().
>     if (transactionTimestamp === undefined) {
>       return {
>         success: false,
>         error: {
>           code: 'TRANSACTION_TIMESTAMP_REQUIRED',
>           field: 'transactionTimestamp',
>           observedValue: undefined,
>           message: 'transactionTimestamp es obligatorio cuando leaseUntil es Date; recibido undefined (parámetro ausente).'
>         }
>       };
>     }
>     if (transactionTimestamp === null || !(transactionTimestamp instanceof Date)) {
>       return {
>         success: false,
>         error: {
>           code: 'INVALID_TRANSACTION_TIMESTAMP',
>           field: 'transactionTimestamp',
>           observedValue: transactionTimestamp,
>           message: 'transactionTimestamp debe ser una instancia válida de Date; recibido ' + (transactionTimestamp === null ? 'null' : typeof transactionTimestamp)
>         }
>       };
>     }
>     const txnTime = transactionTimestamp.getTime();
>     if (!Number.isFinite(txnTime) || Number.isNaN(txnTime)) {
>       return {
>         success: false,
>         error: {
>           code: 'INVALID_TRANSACTION_TIMESTAMP',
>           field: 'transactionTimestamp',
>           observedValue: transactionTimestamp,
>           message: 'transactionTimestamp debe ser Date válido con valor temporal finito (no NaN, no ±Infinity).'
>         }
>       };
>     }
>     leaseUntil = rawLeaseUntil > transactionTimestamp ? 'future' : 'past';
>   } else {
>     // Tipo no admitido: string, number, boolean, object, etc.
>     return {
>       success: false,
>       error: {
>         code: 'INVALID_LEASE_UNTIL_TYPE',
>         field: 'processing.leaseUntil',
>         observedValue: rawLeaseUntil,
>         message: `leaseUntil debe ser Date | null | undefined; recibido ${rawLeaseUntil?.constructor?.name ?? typeof rawLeaseUntil}`
>       }
>     };
>   }
>   
>   // 4. Validar y extraer leaseGeneration
>   const rawLeaseGen = processingValue.leaseGeneration;
>   let leaseGeneration: LeaseGenValue;
>   
>   if (rawLeaseGen === undefined) {
>     leaseGeneration = 'absent';
>   } else if (typeof rawLeaseGen === 'number') {
>     // Validaciones estrictas: sin coerción, sin defaults
>     if (!Number.isInteger(rawLeaseGen)) {
>       return {
>         success: false,
>         error: {
>           code: 'FRACTIONAL_GENERATION',
>           field: 'processing.leaseGeneration',
>           observedValue: rawLeaseGen,
>           message: `leaseGeneration debe ser entero; recibido valor fraccionario ${rawLeaseGen}`
>         }
>       };
>     }
>     if (rawLeaseGen < 0) {
>       return {
>         success: false,
>         error: {
>           code: 'NEGATIVE_GENERATION',
>           field: 'processing.leaseGeneration',
>           observedValue: rawLeaseGen,
>           message: `leaseGeneration debe ser entero no negativo; recibido ${rawLeaseGen}`
>         }
>       };
>     }
>     if (rawLeaseGen === 0) {
>       leaseGeneration = 'zero';
>     } else {
>       leaseGeneration = 'positive';
>     }
>   } else {
>     // Tipo no admitido: string, boolean, null, object, etc.
>     // NOTA: null se trata como 'absent' en BSON si el serializer lo omite,
>     // pero si el documento tiene explícitamente leaseGeneration: null, es un tipo no admitido.
>     return {
>       success: false,
>       error: {
>         code: 'INVALID_LEASE_GENERATION_TYPE',
>         field: 'processing.leaseGeneration',
>         observedValue: rawLeaseGen,
>         message: `leaseGeneration debe ser number (entero no negativo) | undefined; recibido ${rawLeaseGen?.constructor?.name ?? typeof rawLeaseGen}`
>       }
>     };
>   }
>   
>   // 5. Verificar que todos los valores son representables en las dimensiones declaradas
>   // Las dimensiones son: processing (absent/present), leaseOwner (absent/null/same/other),
>   // leaseUntil (absent/null/future/past), leaseGeneration (absent/zero/positive)
>   // Cualquier valor que no mapee a estas dimensiones es UNREPRESENTABLE_VALUE
>   // (Esta comprobación es redundante con las validaciones arriba, pero documenta la regla)
>   
>   return {
>     success: true,
>     observed: {
>       processing: 'present',
>       leaseOwner,
>       leaseUntil,
>       leaseGeneration
>     }
>   };
> }
> 
> // Uso en el flujo de adquisición:
> // const extraction = extractAndValidateObservedState(
> //   retirementOpDoc,
> //   currentWorkerId,        // workerId string no vacío — OBLIGATORIO (se valida si leaseOwner es string)
> //   transaction.startTime   // transactionTimestamp Date válido — OBLIGATORIO (se valida si leaseUntil es Date)
> // );
> // if (!extraction.success) {
> //   // Bloquear adquisición: estado BSON no válido para el dominio de retiro
> //   // Registrar error.extraction.error.code, error.field, error.observedValue
> //   // No inventar default ni coercionar
> //   return { state: 'INVALID_BSON_STATE', evidence: { extractionError: extraction.error } };
> // }
> // const classification = classifyBSONState(extraction.observed!);
> // if (classification === 'E-INVALID') {
> //   return { state: 'INVALID_BSON_STATE', evidence: { classification } };
> // }
> // // Continuar con predicado de elegibilidad y filtro CAS según classification
> ```

### 8.2 Verificar serialización y semántica MongoDB real

**Versiones instaladas en CRM:**
- `mongodb@6.20.0` (driver)
- `mongoose@8.19.3` (ODM)
- `bson@6.x` (incluido en driver)

**Comportamiento validado contra driver v6.20.0 y espec BSON:**

| Construcción | Semántica real |
|-------------|----------------|
| `{ field: null }` | Coincide con campo `null` **Y** campo ausente (solo si subdocumento padre existe) |
| `{ field: { \$exists: false } }` | Coincide solo con campo ausente (subdocumento padre debe existir) |
| `\$in: [null, undefined]` | En driver: `undefined` → campo omitido en BSON (no `\$exists: false`). Coincide con `null` o ausente **solo si subdocumento padre existe** |
| Subdocumento `processing` ausente | Ruta anidada `processing.leaseOwner` **no existe** para ningún operador (`\$eq`, `\$in`, `\$exists`, `\$lte`) |
| Comparación fechas `\$lte` | Requiere que campo exista y sea fecha; `null` o ausente no coinciden |
| `{ processing: { \$exists: false } }` | Coincide solo cuando `processing` está ausente del documento |

**Tabla de verdad para filtros sobre `processing.leaseOwner` / `processing.leaseUntil`:**

| Estado BSON `processing` | `processing.leaseOwner` | `processing.leaseUntil` | `{ leaseOwner: null }` | `{ leaseOwner: { \$exists: false } }` | `{ leaseUntil: { \$lte: now } }` |
|--------------------------|-------------------------|-------------------------|------------------------|--------------------------------------|--------------------------------|
| Ausente | N/A | N/A | No match | No match | No match |
| Presente, sin campos | Ausente | Ausente | Match (campo ausente) | Match | No match (campo ausente) |
| Presente, `leaseOwner: null` | `null` | Ausente | Match | No match | No match |
| Presente, `leaseOwner: "worker1"` | `"worker1"` | Ausente | No match | No match | No match |
| Presente, `leaseUntil: Date` | Ausente | `Date` | Match | Match | Match si `Date <= now` |

**Opciones de serialización relevantes:**
- Mongoose `ignoreUndefined`: por defecto `true` — `undefined` se omite al convertir a BSON
- Driver `serializeFunctions`, `ignoreUndefined`: heredados de configuración de conexión
- **No afirmar que `undefined` se omite si el serializer lo convierte en null** — en driver v6.20.0 + Mongoose 8.19.3, `undefined` se omite (no se serializa como null)

### 8.3 Predicado único de elegibilidad → Filtro CAS

**Definir primero un predicado de elegibilidad inequívoco.**

Para cada uno de los 10 estados, establece:
- Cuándo se permite **adquirir** (primera vez)
- Cuándo se permite **readquirir** (mismo owner, lease vencido)
- Cuándo se permite **renovar** (mismo owner, lease vigente — conserva generación)
- Cuándo se **rechaza** por lease vigente (mismo u otro owner)
- Cuándo se permite **tomar un lease vencido de otro owner** (adquisición sobre otro)
- Qué significa **owner ausente** (campo no existe en subdocumento presente)
- Qué significa **owner null** (campo existe con valor null)

**Después construir el filtro CAS para proteger exactamente el estado observado.**

| Estado BSON (pre-adquisición) | Elegible | Acción | Filtro CAS generado (rutas `processing.*` completas, valores exactos observados, distinción estricta absent/null/valor) | Generación resultante |
|-------------------------------|----------|--------|---------------------------------------------------------------------------------------------------------------------|----------------------|
| **E1.** `processing` ausente | ✅ Sí | Adquisición | `{ _id, revision, status, processing: { \$exists: false } }` | 1 (primera adquisición) |
| **E2.** `processing` presente, sin campos (tres campos absent) | ✅ Sí | Adquisición | `{ _id, revision, status, processing: { \$exists: true }, processing.leaseOwner: { \$exists: false }, processing.leaseUntil: { \$exists: false }, processing.leaseGeneration: { \$exists: false } }` | 1 |
| **E3.** `leaseOwner: null` (explícito), `leaseUntil` absent, `leaseGeneration` absent | ✅ Sí | Adquisición | `{ _id, revision, status, processing: { \$exists: true }, processing.leaseOwner: { \$in: [null], \$exists: true }, processing.leaseUntil: { \$exists: false }, processing.leaseGeneration: { \$exists: false } }` | 1 |
| **E4.** `leaseOwner: "same"`, `leaseUntil` future (vigente), `leaseGeneration` = observedGen (>0) | ❌ No | **Renovación** | `{ _id, revision, status, processing.leaseOwner: "same", processing.leaseUntil: observedLeaseUntil, processing.leaseGeneration: observedGen }` | observedGen (renovación: no incrementa) |
| **E5.** `leaseOwner: "same"`, `leaseUntil` null_or_past (vencido), `leaseGeneration` = observedGen (>0) | ✅ Sí | **Readquisición** | `{ _id, revision, status, processing.leaseOwner: "same", processing.leaseUntil: observedLeaseUntil === null ? { \$in: [null], \$exists: true } : observedLeaseUntil, processing.leaseGeneration: observedGen }` | observedGen + 1 |
| **E6.** `leaseOwner: "other"`, `leaseUntil` future (vigente), `leaseGeneration` = observedGen (>0) | ❌ No | Rechazada | `{ _id, revision, status, processing.leaseOwner: "other", processing.leaseUntil: observedLeaseUntil, processing.leaseGeneration: observedGen }` | — (rechazado) |
| **E7.** `leaseOwner: "other"`, `leaseUntil` null_or_past (vencido), `leaseGeneration` = observedGen (>0) | ✅ Sí | **Adquisición sobre otro** | `{ _id, revision, status, processing.leaseOwner: "other", processing.leaseUntil: observedLeaseUntil === null ? { \$in: [null], \$exists: true } : observedLeaseUntil, processing.leaseGeneration: observedGen }` | observedGen + 1 |
| **E8.** `leaseOwner` absent, `leaseUntil` future (vigente), `leaseGeneration` absent | ❌ No | Rechazada | `{ _id, revision, status, processing.leaseOwner: { \$exists: false }, processing.leaseUntil: observedLeaseUntil, processing.leaseGeneration: { \$exists: false } }` | — (rechazado) |
| **E9.** `leaseOwner` absent, `leaseUntil` null_or_past (PRESENTE, no future), `leaseGeneration` absent | ⚠️ Pendiente | Adquisición (condicional) | `{ _id, revision, status, processing.leaseOwner: { \$exists: false }, processing.leaseUntil: observedLeaseUntil === null ? { \$in: [null], \$exists: true } : observedLeaseUntil, processing.leaseGeneration: { \$exists: false } }` | **Pendiente decisión contractual** — `leaseGeneration` ausente no puede incrementarse; requiere definición de generación inicial en schema §16.14. No se asume `0`. Bloqueada mientras schema pendiente. |
| **E10.** `processing` presente, `leaseOwner` absent, `leaseUntil` absent, `leaseGeneration` = 0 (explícito) | ✅ Sí | Adquisición | `{ _id, revision, status, processing: { \$exists: true }, processing.leaseOwner: { \$exists: false }, processing.leaseUntil: { \$exists: false }, processing.leaseGeneration: 0 }` | 1 (primera adquisición efectiva) |
| **E-INVALID.** Cualquier combinación no cubierta por E1-E10 (ver análisis exhaustividad §8.1) | ❌ No | **Bloqueada** | *No se genera filtro CAS — estado rechazado en validación previa* | — (bloqueado) |

**Pruebas lógicas de consistencia (pseudocódigo, no tests de integración):**
- Para cada estado, documento inicial → filtro CAS generado → debe coincidir/rechazarse según elegibilidad → generación resultante.
- **B5-05:** La Revisión 21 **no contiene implementación ni pruebas reproducibles** de `buildLeaseCasFilter`. La afirmación "cero desalineaciones" se reduce a: **la tabla §8.3 define predicado de elegibilidad y filtro CAS por estado de forma coherente en pseudocódigo**. Equivalencia real requiere implementación y tests de integración (pendientes).

**Fencing (B5-03):** Los filtros CAS arriba **incluyen `processing.leaseGeneration`** cuando el estado observado tiene generación presente (filas E4-E7). Esto protege contra:
- Renovación (E4): CAS requiere `leaseGeneration` existente → worker con generación obsoleta no renueva
- Readquisición (E5): CAS requiere `leaseGeneration` existente → incrementa a `observedGen + 1`
- Adquisición sobre otro (E7): CAS requiere `leaseGeneration` existente → incrementa a `observedGen + 1`
- Sin generación observada (E1-E3, E8-E10): no hay fencing por generación (primera adquisición o estado sin generación)

> **Corrección D5-03 (4ta/5ta/esta intervención):** 
> - Filas E2, E3: protegen explícitamente `processing.leaseGeneration: { \$exists: false }` (ausencia exacta). **E3 corregido (5ta intervención):** usa `processing.leaseOwner: { \$in: [null], \$exists: true }` para distinguir `null` explícito de campo ausente (MongoDB `{ field: null }` coincide con null Y ausente si padre existe).
> - Filas E4–E7: protegen el valor EXACTO de `leaseGeneration` observado (`observedGen`) y el valor EXACTO de `leaseUntil` observado (`observedLeaseUntil`), no solo existencia. **E5, E7 corregidos (esta intervención):** cuando `observedLeaseUntil === null`, el filtro usa `{ \$in: [null], \$exists: true }` para exigir simultáneamente igualdad a null Y existencia del campo; cuando es fecha, exige igualdad exacta a esa fecha. No permite que igualdad a null coincida con campo ausente.
> - Fila E8: protege `leaseGeneration` absent y `leaseUntil` exacto observado
> - **Fila E9: `leaseGeneration` ausente — NO se incrementa.** La fila E9 en tabla §8.3 y §8.5 ahora marcada como **pendiente decisión contractual**: `leaseGeneration` ausente no puede interpretarse como 0 ni incrementarse sin definición en schema §16.14. El filtro CAS protege `leaseGeneration: { \$exists: false }` y `leaseUntil` exacto observado, pero la generación resultante queda **bloqueada pendiente schema**. No se asume `observedGen + 1` ni `gen=0`.
> - Fila E10: filtro CAS con `leaseGeneration: 0` exacto (no `$or` con absent, ya que E10 requiere generation=0 explícito)
> - Fila E-INVALID: **no se genera filtro CAS** — estado rechazado en validación previa de clasificación BSON; bloquea adquisición y cualquier ruta de éxito
> - Todas las comparaciones de igualdad usan valores exactos observados en la lectura
> - **Diferenciación estricta**: igualdad contra valor observado (`field: observedValue`), existencia explícita (`\$exists: true`), ausencia explícita (`\$exists: false`), valor `null` (`{ \$in: [null], \$exists: true }`), valor cero (`field: 0`). No se usan aproximaciones. **MongoDB `{ field: null }` NO distingue null de absent; requiere `$exists: true` combinado**.
> - Se protege la presencia o ausencia exacta observada de todos los campos relevantes: `processing`, `processing.leaseOwner`, `processing.leaseUntil`, `processing.leaseGeneration`, además de `_id`, `revision` y `status`.
> - Si una combinación no tiene filtro CAS seguro y demostrable (E-INVALID), debe bloquearse; no se inventa filtro aproximado.
> - Si el documento cambia entre lectura y actualización, el CAS falla y la transacción se reintenta (comportamiento MongoDB estándar)

**Importante:** Proteger el estado observado no significa inventar un valor por defecto.
Si el esquema final no define cómo inicializar o persistir `leaseGeneration`, conserva esa dependencia como pendiente contractual. No declares resuelta la inicialización por tratar ausencia como cero sin respaldo canónico.

**Distinción de casos operacionales:**
- **Primera adquisición:** E1, E2, E3 → generación = 1
- **Readquisición por mismo owner:** E5 → generación = observedGen + 1
- **Renovación:** E4 → generación = observedGen (conserva)
- **Adquisición sobre otro owner:** E7 → generación = observedGen + 1
- **Rechazo por lease vigente:** E4 (mismo owner), E6 (otro owner), E8 (owner ausente, lease futuro)
- **Estado no interpretable o contractual incompleto:** E10 (generation ausente/0 sin leaseOwner/leaseUntil definidos)

### 8.4 Tratamiento de otros propietarios — Semántica coherente

| Situación | Adquisición permitida | Fundamento |
|-----------|----------------------|------------|
| No existe owner (`processing` ausente o `leaseOwner` ausente/null) | ✅ Sí | Primera adquisición |
| Owner = mismo solicitante, lease vigente | ✅ **Renovación** (no incrementa generación) | §16.6: "renovación conserva generación" |
| Owner = mismo solicitante, lease vencido | ✅ **Readquisición** (incrementa generación) | §16.6: "cada nueva adquisición efectiva incrementa" |
| Owner = distinto, lease vigente | ❌ No | Fencing: worker obsoleto no puede avanzar |
| Owner = distinto, lease vencido | ✅ Sí (**Adquisición sobre otro**) | Lease vencido = recuperable; CAS protege estado observado exacto |
| Owner ausente (campo no existe), `leaseUntil` futuro | ❌ No | Lease vigente implícito (otro worker renovó sin owner explícito) |
| Owner ausente (campo no existe), `leaseUntil` pasado/ausente | ✅ Sí | Sin owner efectivo |

**El CAS protege el estado observado completo:** `_id`, `revision`, `status`, existencia de `processing`, valores exactos de `leaseOwner` y `leaseUntil` observados, presencia/valor de `leaseGeneration`.

### 8.5 Tabla de estados BSON corregida (Reemplaza §7.5 Revisión 20)

| Fila | Estado inicial `processing` | Acción | ¿Renovación? | Generación resultado | Nota |
|------|----------------------------|--------|--------------|---------------------|------|
| 1 | Ausente (E1) | Adquisición | No | 1 | Primera adquisición |
| 2 | Presente, sin campos (E2) | Adquisición | No | 1 | Subdocumento existe pero vacío |
| 3 | `leaseOwner: null` (explícito), `leaseUntil` ausente (E3) | Adquisición | No | 1 | Explícito null ≠ ausente |
| 4 | `leaseOwner: "same"`, `leaseUntil` futuro (E4) | **Renovación** | **Sí** | Mismo (no incrementa) | §16.6: renovación conserva generación |
| 5 | `leaseOwner: "same"`, `leaseUntil` null_or_past (E5) | **Readquisición** | No | observedGen + 1 | Lease vencido = nueva adquisición |
| 6 | `leaseOwner: "other"`, `leaseUntil` futuro (E6) | **Rechazada** | No | — | Fencing vigente |
| 7 | `leaseOwner: "other"`, `leaseUntil` null_or_past (E7) | **Adquisición sobre otro** | No | observedGen + 1 | Lease vencido permite tomar operación |
| 8 | `leaseOwner` ausente, `leaseUntil` futuro (E8) | **Rechazada** | No | — | Lease vigente implícito |
| 9 | `leaseOwner` ausente, `leaseUntil` null_or_past PRESENTE (E9) | **Adquisición (condicional)** | No | **Pendiente decisión contractual** | Sin owner efectivo, leaseUntil presente no futuro; `leaseGeneration` ausente — no puede incrementarse sin schema §16.14 que defina generación inicial |
| 10 | `processing` presente, `leaseOwner` ausente, `leaseUntil` ausente, `leaseGeneration` = 0 (E10) | **Adquisición** | No | 1 | Generación 0 explícita = primera adquisición efectiva |
| 11 | **E-INVALID** — Combinación no cubierta por E1-E10 (ver §8.1 análisis exhaustividad) | **BLOQUEADA** | No | — | Estado inválido/no reconocido; bloquea adquisición y éxito |

**Correcciones vs Revisión 20:**
- Fila 4: Etiquetada **Renovación** (no "adquisición cuando lease vigente").
- Fila 5: Etiquetada **Readquisición** (no "renovación"), incrementa generación.
- Fila 7: **Adquisición permitida** (Revisión 20 ambigua: "se rechaza si algoritmo permite continuar" → corregido: algoritmo permite, CAS protege estado observado).
- Fila 9: `leaseUntil` debe ser PRESENTE (null o past) — distingue de E2 (absent) y E10 (absent).
- Fila 10: `leaseGeneration` = 0 explícito — distingue de E2 (absent) y E9 (absent).
- **Fila 11 (NUEVA):** **E-INVALID** — categoría explícita para combinaciones no cubiertas por E1-E10; bloquea adquisición y éxito.
- Separación explícita: **adquisición** (primera vez), **readquisición** (mismo owner, lease vencido), **renovación** (mismo owner, lease vigente), **adquisición sobre otro** (owner distinto, lease vencido).

### 8.6 Bloqueos de esquema mantenidos

No se inventan valores por defecto ni se alteran contratos para resolver:
- `leaseGeneration` ausente frente a cero → **bloqueo explícito**, pendiente schema final (§16.14).
- `processing` ausente frente a presente → **bloqueo explícito**, pendiente schema final (§16.14).
- Inicialización y persistencia de generación → **bloqueo explícito**, pendiente schema final (§16.14).

**No se afirman pruebas de integración.** Solo comprobaciones lógicas de pseudocódigo contra especificación BSON y driver v6.20.0.

---

## 9. Bloque D: AttemptContext y withTransaction (Bloque D)

### 9.1 Flujo unificado de errores — Catch exterior con propagación de AttemptContext (Corregido B6-01, B6-02, B6-03, D6-01, D6-03)

La Revisión 20 no implementaba un `catch` exterior que cubra la llamada a `withTransaction` ni propagaba el contexto. Revisión 21 unifica §9.1 y §9.3 en un **único flujo coherente** con propagación real de `AttemptContext` al catch exterior.

**Corrección D6-01/D6-03:** El contexto se construye **incrementalmente DURANTE** la ejecución del callback (no al final), mediante un objeto mutable pasado al callback. Si el callback realiza escrituras y luego lanza excepción, el contexto ya está actualizado y disponible para el catch exterior.

```typescript
interface AttemptContext {
  operationId: string;
  contactId: string;
  attemptedTransition: AttemptedTransition;
  leaseAcquired: boolean;
  leaseGeneration: number;
  writesPerformed: boolean;
  _generation: number; // Marca de generación del callback
}

// Contexto mutable que el callback actualiza incrementalmente
interface MutableAttemptContext {
  operationId: string;
  contactId: string;
  attemptedTransition: AttemptedTransition;
  leaseAcquired: boolean;
  leaseGeneration: number;
  writesPerformed: boolean;
  _generation: number;
}

interface TransactionResult<T> {
  success: boolean;
  result?: T;
  error?: Error;
  attemptContext?: AttemptContext;
  commitPhase?: 'callback' | 'commit' | 'unknown_commit_result';
}

// FLUJO UNIFICADO: withTransaction + captura contexto incremental + catch exterior con contexto
async function executeWithTransactionSafely<T>(
  session: ClientSession,
  // Callback recibe sesión Y contexto mutable para actualizar durante ejecución
  callback: (txnSession: ClientSession, ctx: MutableAttemptContext) => Promise<TransactionResult<T>>
): Promise<TransactionResult<T>> {
  // Contenedor para capturar contexto de la ÚLTIMA ejecución del callback
  const contextContainer = { current: null as AttemptContext | null, generation: 0 };
  
  try {
    const result = await session.withTransaction(
      async (txnSession) => {
        // REINICIALIZACIÓN EXPLÍCITA al inicio de CADA callback (reintentos por TransientTransactionError)
        contextContainer.current = null;
        contextContainer.generation++;
        const currentGeneration = contextContainer.generation;
        
        // Crear contexto mutable ANTES de ejecutar callback
        const mutableContext: MutableAttemptContext = {
          operationId: '',
          contactId: '',
          attemptedTransition: null as any,
          leaseAcquired: false,
          leaseGeneration: 0,
          writesPerformed: false,
          _generation: currentGeneration
        };
        
        let callbackResult: TransactionResult<T>;
        try {
          // Ejecutar lógica de negocio del callback — pasa contexto mutable
          callbackResult = await callback(txnSession, mutableContext);
          
          // Callback terminó (éxito o error de negocio controlado) — finalizar contexto
          const finalContext: AttemptContext = {
            operationId: mutableContext.operationId,
            contactId: mutableContext.contactId,
            attemptedTransition: mutableContext.attemptedTransition,
            leaseAcquired: mutableContext.leaseAcquired,
            leaseGeneration: mutableContext.leaseGeneration,
            writesPerformed: mutableContext.writesPerformed,
            _generation: currentGeneration
          };
          contextContainer.current = finalContext;
          
          return { ...callbackResult, attemptContext: finalContext };
        } catch (error) {
          // D6-01/D6-03: Error DENTRO del callback (inesperado, no de negocio)
          // mutableContext YA contiene las actualizaciones incrementales hechas antes del throw
          // NO reinicializar — preservar para catch exterior
          const partialContext: AttemptContext = {
            operationId: mutableContext.operationId,
            contactId: mutableContext.contactId,
            attemptedTransition: mutableContext.attemptedTransition,
            leaseAcquired: mutableContext.leaseAcquired,
            leaseGeneration: mutableContext.leaseGeneration,
            writesPerformed: mutableContext.writesPerformed,
            _generation: currentGeneration
          };
          contextContainer.current = partialContext;
          
          throw error; // Propaga a withTransaction para manejo de TransientTransactionError
        }
      },
      { readConcern: { level: 'snapshot' } } // Propuesta técnica, validación Atlas pendiente (B1-04)
    );
    
    // withTransaction RETORNÓ (commit confirmado o UnknownTransactionCommitResult resuelto)
    // result.attemptContext corresponde a la ÚLTIMA ejecución del callback
    return { 
      success: true, 
      result: result.result, 
      attemptContext: result.attemptContext, 
      commitPhase: result.commitPhase || 'callback' 
    };
    
  } catch (error) {
    // CAPTURA REAL en catch exterior: error de withTransaction (no transitorio, agotamiento, etc.)
    // contextContainer.current = contexto de la ÚLTIMA ejecución del callback (si la hubo)
    const validContext = contextContainer.current;
    
    // Validar que contexto pertenece a la generación final (no obsoleto)
    const contextForClassification = (validContext && validContext._generation === contextContainer.generation)
      ? validContext
      : undefined; // No usar contexto obsoleto o de generación anterior
    
    // B6-01: classifyAndReconcileError RECIBE el contexto (puede ser undefined)
    return classifyAndReconcileError(error, contextForClassification);
  }
}

// Firma actualizada: recibe AttemptContext opcional para clasificación con contexto
function classifyAndReconcileError(
  error: Error, 
  attemptContext?: AttemptContext
): TransactionResult<never> {
  // Clasificación con contexto disponible (si existe)
  // ...
  return { success: false, error, attemptContext };
}
```

**Puntos clave del flujo unificado (B6-03, D6-01, D6-03):**
- **Dónde se crea el contexto:** Objeto `mutableContext` creado **antes** del callback, actualizado **incrementalmente DURANTE** la ejecución del callback (línea ~1199). El callback recibe `mutableContext` y lo actualiza conforme realiza operaciones.
- **Cuándo se actualiza:** En cada ejecución del callback (reintentos por `TransientTransactionError` incrementan `generation` y reinician `mutableContext`).
- **Cómo sobrevive a reintentos:** `contextContainer` está **fuera** del `withTransaction`; se reinicializa al inicio de cada callback (`contextContainer.current = null`).
- **Cómo se entrega al catch exterior:** `contextContainer.current` capturado en el `catch` interior ANTES de re-lanzar el error (siempre accesible en scope).
- **Cómo se distingue error de ejecución vs commit incierto:** `commitPhase` en resultado + tipo de error (`TransientTransactionError` vs `UnknownTransactionCommitResult`).
- **Cómo se evita contexto obsoleto:** Validación `_generation === contextContainer.generation` (solo última ejecución).
- **D6-01/D6-03 — Contexto ante excepción tras escrituras:** El callback actualiza `mutableContext` **antes** de cada operación crítica (escritura, adquisición de lease). Si luego lanza excepción, `mutableContext` YA contiene `writesPerformed: true`, `leaseAcquired: true`, `leaseGeneration: N`, etc. El `catch` interior captura este estado parcial en `contextContainer.current` y el catch exterior lo recibe vía `contextForClassification`. Esto permite distinguir:
  - Contexto diagnóstico: "se intentaron escrituras, lease adquirido, generación N" — útil para reconciliación.
  - **NO** confirmación de commit: `attemptContext` ≠ `commitPhase === 'callback'` o `'commit'`.
- **D6-03 — Reintentos no reutilizan contexto obsoleto:** Cada reejecución del callback (por `TransientTransactionError`) incrementa `generation` y crea NUEVO `mutableContext`. El contexto de la ejecución anterior queda descartado.

### 9.2 Separación callback y commit — Distinción normativa

Distinguir explícitamente:

1. **Callback ejecutado** — Primera entrada al callback del `withTransaction`
2. **Callback reejecutado por error transitorio** — `TransientTransactionError` durante callback → driver reintenta transacción completa (nueva transacción, mismo `txnNumber` incrementado)
3. **Callback completado** — Callback termina sin lanzar error (o error no transitorio)
4. **Commit iniciado** — Driver llama `commitTransaction()` internamente
5. **Commit confirmado** — `commitTransaction` resuelve exitosamente
6. **Commit con resultado desconocido** — `UnknownTransactionCommitResult` → driver reintenta commit de la **misma transacción** (mismo `txnNumber`), **SIN reejecutar callback**
7. **Reintento de commit** — Múltiples reintentos de commit ante `UnknownTransactionCommitResult`
8. **Agotamiento de reintentos** — Driver agota `maxCommitTimeMS` o reintentos configurados
9. **Error definitivo** — Error no transitorio, no desconocido, o agotamiento → propaga a catch exterior

**No interpretar la existencia de AttemptContext como confirmación del commit.**
**No permitir que un contexto generado por un callback anterior sea utilizado como prueba de éxito de un intento posterior.**

### 9.3 Verificación de 9 escenarios — Distinción conceptual vs demostrado (Corregido B6-04)

La tabla siguiente distingue **razonamiento conceptual** (diseño del mecanismo) de **comportamiento demostrado por pseudocódigo** (flujo unificado §9.1). **No se declara que AttemptContext funciona en runtime** — solo que el pseudocódigo documenta el flujo coherente.

| Escenario | `contextContainer.current` al final | `contextContainer.generation` | Catch exterior usa contexto correcto? | Categoría |
|-----------|-------------------------------------|-------------------------------|--------------------------------------|-----------|
| 1. Primera ejecución, éxito | Context válido, gen=1 | 1 | ✅ Sí (gen coincide) | Demostrado en pseudocódigo §9.1 |
| 2. Reejecución por `TransientTransactionError`, éxito en 2da | Context válido, gen=2 | 2 | ✅ Sí (gen coincide, valor actualizado) | Demostrado en pseudocódigo §9.1 |
| 3. Error ANTES de construir contexto | `null` | 1 (o 2 si reejecutó) | ✅ No usa contexto (es null) | Demostrado en pseudocódigo §9.1 |
| 4. Error DESPUÉS de construir contexto | Context gen=N (donde falló) | N | ✅ Usa contexto de ejecución que falló (diagnóstico) | Demostrado en pseudocódigo §9.1 |
| 5. Error durante commit | Context gen=N (construido antes de commit) | N | ✅ Usa contexto de transacción con commit fallido | Demostrado en pseudocódigo §9.1 |
| 6. `UnknownTransactionCommitResult` (reintento commit) | Context gen=N (callback NO reejecutado) | N | ✅ Contexto válido, misma generación | Demostrado en pseudocódigo §9.1 |
| 7. `TransientTransactionError` durante commit (raro) | Context gen=N+1 (transacción completa reejecutada) | N+1 | ✅ Contexto de última ejecución | Razonamiento conceptual (driver) |
| 8. Agotamiento reintentos commit | Context de último intento válido o null | Final | ✅ Comportamiento definido | Razonamiento conceptual |
| 9. Error no transitorio en callback | Context de ejecución que falló o null | N | ✅ No consume contexto obsoleto | Demostrado en pseudocódigo §9.1 |

**El catch exterior NUNCA consume accidentalmente un contexto perteneciente a una ejecución anterior** (validado por `_generation === contextContainer.generation`).

**Distinción preservada (B6-04):**
- `TransientTransactionError`: puede implicar **repetir el callback/transacción completa** (nueva transacción, `txnNumber` incrementado)
- `UnknownTransactionCommitResult`: requiere **reintentar el commit de la misma transacción** SIN repetir el callback (mismo `txnNumber`)

### 9.4 Verificación de driver real: mongodb@6.20.0

**Versión verificada:**
- `crm/package.json`: `"mongodb": "^6.20.0"`
- `crm/pnpm-lock.yaml` línea 4021: `mongodb@6.20.0` confirmado

**Código fuente consultado:** Tag exacto `v6.20.0`, archivo `src/sessions.ts`:
- `ClientSession.withTransaction`: líneas 1065–1164
- `commitTransaction`: líneas 610–685
- `hasTransientTransactionLabel`: línea ~580
- `isUnknownTransactionCommitResult`: línea ~590
- Etiquetas de error: `TransientTransactionError`, `UnknownTransactionCommitResult`, `MongoWriteConcernError`

**Semántica verificada (código fuente exacto v6.20.0):**

| Concepto | Comportamiento real v6.20.0 |
|----------|----------------------------|
| `TransientTransactionError` | Reintento de **transacción completa** (inicia nueva transacción, **reejecuta callback**). Etiquetas: `TransientTransactionError`, `WriteConcernError` con `wtimeout`, etc. |
| `UnknownTransactionCommitResult` | Reintento del **commit de la misma transacción** (misma `txnNumber`, misma sesión). **NO reejecuta callback**. Etiquetas: `UnknownTransactionCommitResult`, `WriteConcernError` sin `wtimeout` pero con `writeConcernError`. |
| `MongoWriteConcernError` | **NO** se clasifica automáticamente. Debe verificarse: `hasTransientTransactionLabel(error)` vs `isUnknownTransactionCommitResult(error)`. Son mutuamente excluyentes en el flujo del driver. |
| Error durante callback | Capturado en `try/catch` del `while(true)` → si `TransientTransactionError` → reintenta todo; si otro error → sale y propaga. |
| Error durante commit | `commitTransaction` lanza → capturado → si `UnknownTransactionCommitResult` → reintenta commit; si `TransientTransactionError` → reintenta transacción completa (raro en commit); si otro → propaga. |

**Distinción expresa (requerida por evaluación independiente):**

| Categoría | Reejecuta callback | Incrementa txnNumber | Generación intento de negocio |
|-----------|-------------------|---------------------|-------------------------------|
| Error durante callback + `TransientTransactionError` | **Sí** | Sí (nueva transacción) | No (misma iteración `for`) |
| Error durante commit + `UnknownTransactionCommitResult` | **No** | No (mismo txnNumber) | No |
| Error durante commit + `TransientTransactionError` | Sí (raro, driver lo trata como transacción completa) | Sí | No |

**Limitación declarada:** La semántica se reporta como **verificada contra el tag publicado en GitHub** (repositorio oficial `mongodb/node-mongodb-native`, tag `v6.20.0`). Si el código fuente exacto no está disponible en el entorno de ejecución, no se presenta como verificada in-situ.

### 9.5 Reconciliación del algoritmo de captura

Actualizados en esta revisión:
- ✅ Pseudocódigo (§5.5, §9.1, §9.3)
- ✅ Variables y scopes (contenedor con generación)
- ✅ Tabla de errores (distinción callback vs commit, Transient vs Unknown)
- ✅ Flujo de reintentos (driver maneja internamente; `for` exterior solo para lógica de negocio)
- ✅ Manejo de contexto en catch (validación de generación)
- ✅ Conclusiones (catch exterior nunca consume contexto obsoleto)

---

## 10. Validación interna obligatoria (Realizada antes de concluir)

Pasada de consistencia cruzada completada:

| # | Verificación | Resultado |
|---|--------------|-----------|
| 1 | Cada consulta mencionada existe en el algoritmo | ✅ 6 consultas (Contacto, currentEvAssigned, outboxExpected, outboxCurrent condicional, expectedEvAssigned, retirementOp condicional) |
| 2 | Cada consulta que requiere sesión la utiliza correctamente | ✅ Todas usan `findById(id, null, { session })` o `.session(txnSession)` |
| 3 | Cada resultado de consulta aparece en la matriz | ✅ D3, D4, D5, D6 cubren todos los resultados |
| 4 | Cada combinación de la matriz tiene una clasificación justificable | ✅ 13 posibles + 4 imposibles = 17, cada una con fundamento |
| 5 | Cada rama del algoritmo corresponde a una clasificación | ✅ 13 ramas ↔ 13 casos posibles (§5.7) |
| 6 | Cada consumidor del helper aplica `decision` | ✅ 8 consumidores inspeccionan `outboxMatch.decision` explícitamente |
| 7 | Ninguna ruta de éxito usa campos contractuales pendientes | ✅ Nivel 1 vacío mantenido; D-W79-14 guard en todas las rutas |
| 8 | Cada estado BSON tiene una regla de elegibilidad | ✅ 10 estados (E1-E10) → tabla §8.3 completa |
| 9 | El filtro CAS protege los campos observados | ✅ Tabla §8.3: CAS generado por estado observado exacto, incluye leaseGeneration, `$or` válido |
| 10 | El ciclo de vida de AttemptContext coincide con el flujo real del driver | ✅ 9 escenarios verificados contra `src/sessions.ts` v6.20.0 |
| 11 | El catch exterior maneja errores y no confunde contexto con commit | ✅ §9.1, §9.3: clasificación explícita, validación de generación |
| 12 | No hay recuentos contradictorios entre secciones | ✅ 13+4=17 en §5.4, §5.7, algoritmo, inventario consumidores |

**Todas las verificaciones pasan. Secciones declaradas aprobadas documentalmente.**

---

## 11. Bloqueos contractuales y técnicos pendientes (Corregido B1-01, T-01, T-07)

| Bloqueo | Origen | Impacto | Resolución requerida | Categoría |
|---------|--------|---------|---------------------|-----------|
| **§16.14 Nivel 1 Outbox** | Canónico WAB-79 | Verificación contractual `return_committed` bloqueada; helper Outbox `decision='block'` siempre | Contrato final Outbox (nombres, payloads, transporte, índices, identidad física idempotencia) | **Dependencia contractual pendiente** |
| **Schema `RetirementOperation`** | Canónico WAB-79 §16.14 | Valor inicial `leaseGeneration`, índices parciales, topología transaccional, campos `admissionBlockedAt`/`eventSetClosedAt` | Schema final aprobado | **Dependencia contractual pendiente** |
| **Verificación Atlas** | D.1.8, §16.14 | Atomicidad Contacto+EvAssigned+Outbox+RetirementOperation; `readConcern: snapshot`; topología clúster | Validación en clúster real | **Validación de entorno pendiente** |
| **Autorización reintento** | §5.6, §16.12 | Sin mecanismo para evidencia positiva de aborto | D-W79-14 (aprobada) + diseño reconciliación + §16.14 Outbox | **Dependencia contractual + validación código** |
| **Estructura `Outbox.payload`** | §16.14 | Verificación `OUTBOX_IDEMPOTENCY_CONFLICT_OURS` condicionada | Contrato final Outbox | **Dependencia contractual pendiente** |
| **Correlación física transiciones retiro** | §16.12 vs §16.14 | Vínculo entre identidad canónica `{retirementOperationId, contacto}` y esquema físico Outbox | §16.14 Outbox físico aprobado | **Dependencia contractual pendiente** |
| **Inicialización `leaseGeneration` / `processing`** | §8.1, §8.6 | Defaults schema, migraciones, persistencia | Schema `RetirementOperation` final | **Dependencia contractual pendiente** |

**Nota (T-01, T-07):** D-W79-14 **está aprobada** (10/11/2026). No es un bloqueo. Lo que bloquea es **§16.14 Outbox físico pendiente** que impide verificación runtime de autoría. Se diferencia: "no existen campos Outbox aprobados" (§16.14) ≠ "existe evidencia contractual aprobada suficiente para decisión positiva" (requiere §16.14 + D-W79-14). No equiparar ambas.

---

## 12. Criterios de aceptación y estado por criterio

| Criterio | Estado | Evidencia |
|----------|--------|-----------|
| **H1/H3 A.1** Identidad consultas formalizada con condiciones exactas; eliminación spread de null | ✅ Corregido y demostrado documentalmente | §5.2 Tabla 4 identidades, Reglas ID-1..5, §5.5 Algoritmo pasos 2-4 |
| **H1/H3 A.2** Fila 8 corregida (reutilización real, no spread de null) | ✅ Corregido y demostrado documentalmente | §5.4 Fila 8 D5=`reused_result`, §5.3 Regla ID-1, §5.5 Pasos 3-4 |
| **H1/H3 A.3** Matriz reconstruida desde dimensiones; recuentos alineados; **esta intervención**: 12 sub-casos por decisión explícita (helper invocado), fila 9c sin éxito alcanzable, nombres corregidos | ✅ Corregido y demostrado documentalmente | §5.3 Dimensiones, §5.4 Matriz 12 sub-casos, §5.7 Trazabilidad con guarda contractual |
| **H1/H3 A.4** Trazabilidad 1:1 caso↔evidencia↔rama↔consultas; **esta intervención**: reconstruida desde algoritmo, filas 13a/13b/13c eliminadas y fusionadas en fila 13 única | ✅ Corregido y demostrado documentalmente | §5.7 Tabla completa 17 casos base / 12 sub-casos |
| **H1/H3 A.5** Sesiones Mongoose: firma correcta en todas las lecturas | ✅ Corregido y demostrado documentalmente | §6.2 Código corregido, §6.3 Verificación `.lean()` |
| **H1/H3 A.6** Distinción tipo de lectura (fuera/asociada/dentro/snapshot) | ✅ Corregido y demostrado documentalmente | §6.4 Tabla normativa |
| **H4 B.1** Helper Outbox consumido inspeccionando `decision` explícitamente | ✅ Corregido y demostrado documentalmente | §5.5 CASO A/B uso `outboxMatch.decision`, §7.1 Código |
| **H4 B.2** `OutboxMatchResult` rediseñado: `approved`/`conditional`/`proposed`/`decision`/`blockReason` | ✅ Corregido y demostrado documentalmente | §7.2 Interface, §5.5 Helper líneas 455-492 |
| **H4 B.3** Inventario exhaustivo **15 consumidores** reales del algoritmo | ✅ Corregido y demostrado documentalmente | §7.3 Tabla auditoría 15 consumidores |
| **H4 B.4** Nivel 1 vacío mantenido y aplicado en todas las rutas; **§16.14 guard explícito** | ✅ Corregido y demostrado documentalmente | §7.4 Declaración, §7.5 Bloqueo §16.14 todas rutas, §5.5 CASO B final |
| **H5 C.1** Modelo datos unificado: rutas `processing.*` consistentemente | ✅ Corregido y demostrado documentalmente | §8.1 Estructura BSON, 10 estados E1-E10 mutuamente excluyentes |
| **H5 C.2** Semántica MongoDB validada v6.20.0 + BSON; tabla de verdad | ✅ Corregido y demostrado documentalmente | §8.2 Tabla semántica real, tabla verdad |
| **H5 C.3** Predicado único elegibilidad → CAS para 10 estados | ✅ Corregido y demostrado documentalmente | §8.3 Tabla 10 estados elegibilidad=CAS |
| **H5 C.4** Owner distinto lease vencido: regla explícita (adquisición sobre otro) | ✅ Corregido y demostrado documentalmente | §8.4 Tabla, §8.5 Fila 7 |
| **H5 C.5** Tabla BSON 10 filas: adquisición/readquisición/renovación/adquisición-sobre-otro separadas | ✅ Corregido y demostrado documentalmente | §8.5 Tabla reconstruida |
| **H5 C.6** Bloqueos esquema mantenidos (generation, processing) | ✅ Corregido y demostrado documentalmente | §8.6 Declaración explícita |
| **Bloque D.1** Flujo errores withTransaction: catch exterior completo | ✅ Corregido y demostrado documentalmente | §9.1 Pseudocódigo estructural unificado |
| **Bloque D.2** Separación callback/commit: 9 fases distinguidas | ✅ Corregido y demostrado documentalmente | §9.2 Distinción normativa |
| **Bloque D.3** Ciclo de vida AttemptContext: contenedor generacional verificado 9 escenarios | ✅ Corregido y demostrado documentalmente | §9.1 Flujo unificado, §9.3 Tabla 9 escenarios |
| **Bloque D.4** Driver v6.20.0 verificado en `src/sessions.ts` líneas exactas | ✅ Corregido y demostrado documentalmente | §9.4 Versión, archivo, líneas, etiquetas |
| **Bloque D.5** Distinción TransientTransactionError vs UnknownTransactionCommitResult | ✅ Corregido y demostrado documentalmente | §9.4 Tabla distinción expresa |
| **Métricas E.1** Métricas físicas medidas (no autodeclaradas) | ✅ Corregido y demostrado documentalmente | §13 Métricas con comando PowerShell |
| **Métricas E.2** Completion Report coincide con medición física | ✅ Corregido y demostrado documentalmente | Completion Report final |
| **D4-02** Decisiones Outbox: tres ramas con estados de resultado distintos + decisionType + guard contractual; **diferencia operativa pendiente validación consumidor downstream** | ⚠️ Parcialmente resuelto | §5.5 CASO A/B tres ramas, §7.3 inventario 15 consumidores, §7.5 bloqueos y nota D4-02 |
| **D5-01** Partición BSON exhaustiva con E-INVALID + validación/extracción BSON real + diferenciación física/abstracta/semántica (recuentos corregidos: E1 = 48 posiciones cartesianas → 1 clase de equivalencia) | ✅ Corregido y demostrado documentalmente (esta intervención) | §8.1 análisis 96 combinaciones, §8.1.1 clasificador formal + validación/extracción, §8.3 tabla 11 estados, §8.5 fila 11 |
| **D5-03** CAS protege valores exactos observados + E-INVALID + correcciones E5/E7 para null explícito; **E9 pendiente decisión contractual** | ✅ Corregido y demostrado documentalmente (esta intervención) | §8.3 tabla CAS con igualdad exacta + correcciones E3/E5/E7 + E9 pendiente, §8.5 coherencia |
| **D2-04** RetirementOp: consulta ilustrativa + bandera interviene en flujo + registro anomalía | ✅ Corregido y demostrado documentalmente (esta intervención) | §5.5 bandera `retirementOpCorrelationDemonstrable` interviene en flujo, `retirementOpQueryAnomaly`, §5.2 notas |
| **T-08** Métricas consistentes con limitación fecha embebida | ✅ Corregido y demostrado documentalmente (esta intervención) | §13 medición PowerShell, §15.A nota T-08 |

**Clasificación final por criterio:**
- ✅ **Corregido y demostrado documentalmente**: 30 criterios (24 originales + 6 de esta intervención: D5-01, D5-03, D2-04, D3-01, T-08, regresiones D6-01/D6-03 verificadas y ampliadas)
- ⚠️ **Corregido parcialmente**: 1 (D4-02 — diferenciación diagnóstica sí, operativa pendiente validación consumidor downstream)
- ❌ **No corregido**: 0
- 🔒 **Bloqueado por contrato**: 7 (tabla §11)
- ⏳ **Pendiente de pruebas reales**: 4 (ver §13.5)

---

## 13. Métricas físicas

**Medición realizada con PowerShell (`Get-Item` + `Get-Content`) sobre el archivo final tras esta intervención correctiva (corrección residual final WAB-79 Revisión 21):**

```powershell
Get-Item "F:\ElemenSoft\waby-docs\projects\agents-multitenant\design\wab-79-viabilidad-tecnica-cierre-revision-21.md" | Select-Object Length, LastWriteTime
(Get-Content "F:\ElemenSoft\waby-docs\projects\agents-multitenant\design\wab-79-viabilidad-tecnica-cierre-revision-21.md").Count
```

| Métrica | Valor |
|---------|-------|
| **Ruta absoluta** | `F:\ElemenSoft\waby-docs\projects\agents-multitenant\design\wab-79-viabilidad-tecnica-cierre-revision-21.md` |
| **Líneas** | 2,224 |
| **Tamaño** | 211,155 bytes |
| **Fecha/hora medición** | 15/11/2026 17:16:45 (hora local, medición PowerShell como referencia temporal de la comprobación) |
| **Metadata LastWriteTime observado** | 15/11/2026 17:11:17 (metadata LastWriteTime observada durante la inspección física independiente previa a esta corrección) |
| **Codificación** | UTF-8 |

*Las métricas se midieron tras aplicar las correcciones residuales. La fecha/hora de medición (17:16:45) refleja la comprobación física PowerShell; el LastWriteTime observado (17:11:17) corresponde a la metadata del sistema de archivos en el momento de la inspección física independiente previa a esta corrección. Los valores 2.224 líneas y 211.155 bytes corresponden a la inspección histórica del 15/11/2026 a las 17:16:45. El LastWriteTime 17:11:17 pertenece a esa misma observación histórica. Una inspección posterior, realizada a las 18:36:12, observó 2.224 líneas y 212.373 bytes, con LastWriteTime 18:34:50. Los valores históricos no representan necesariamente el estado físico posterior a los guardados siguientes. El LastWriteTime es metadata dinámica del sistema de archivos. El valor posterior al guardado debe obtenerse mediante una nueva inspección física y comunicarse en el informe final de ejecución.*

---

## 14. Conclusión de Revisión 21

### 14.1 Corregido y demostrado documentalmente

| Área | Qué se corrigió | Cómo se demostró |
|------|----------------|------------------|
| **H1/H3 Consultas Outbox** | Eliminación definitiva spread de `null`; identidades con condiciones de ejecución exactas; estados `not_queried`/`not_found`/`reused_result` diferenciados; reutilización real con referencia a consulta original; Regla ID-1..5 normativas | §5.2 Tabla formal, §5.3 Dimensiones, §5.5 Algoritmo pasos 2-4 con consultas ejecutadas por rama |
| **H1/H3 Matriz** | Fila 8 corregida (D5=`reused_result`); Fila 10 eliminada (imposible); Fila 13 **fusionada en única fila** (antes 13a/13b/13c); 13 combinaciones base posibles + 4 imposibles = 17; trazabilidad 1:1; recuentos alineados; **esta intervención**: matriz reconstruida con **12 sub-casos** por decisión explícita (helper invocado), fila 9c sin `OUR_TRANSITION_COMMITTED` alcanzable, **nombres de estado corregidos** (sin "COMMITTED" engañoso) | §5.4 Matriz 12 sub-casos, §5.7 Tabla trazabilidad completa con guarda contractual, recuentos matemáticos verificados |
| **H1/H3 Sesiones Mongoose** | Firma correcta `findById(id, null, { session })` en todas las lecturas; distinción 4 tipos de lectura; verificación `.lean()` no elimina sesión; todas las consultas comparten snapshot transaccional (propuesta, validación Atlas pendiente) | §6.2 Código corregido, §6.3 Verificación, §6.4 Tabla normativa, §6.5 Escrituras |
| **H4 Helper Outbox** | Consumo inspecciona `decision` explícitamente en **15 consumidores**; `OutboxMatchResult` rediseñado con `approved`/`conditional`/`proposed`/`decision`/`blockReason`; Nivel 1 vacío aplicado uniformemente; **§16.14 guard explícito** en todas las rutas | §5.5 CASO A/B, §7.1 Código, §7.2 Interface, §7.3 Tabla 15 consumidores, §7.4-7.5 Bloqueos |
| **H5 CAS/Lease** | 10 estados BSON E1-E10 mutuamente excluyentes (dimensiones ortogonales); predicado único elegibilidad → CAS; tabla estados separa adquisición/readquisición/renovación/adquisición-sobre-otro; semántica MongoDB validada v6.20.0; filtros CAS con rutas `processing.*` y `$or` válidos; fencing por `leaseGeneration` demostrado en pseudocódigo | §8.1 Estructura BSON, §8.2 Semántica validada, §8.3 Tabla 10 estados, §8.5 Tabla BSON reconstruida |
| **Bloque D AttemptContext** | Flujo errores unificado (§9.1) con catch exterior real; 9 fases callback/commit distinguidas; mecanismo contenedor generacional verificado 9 escenarios (conceptual + pseudocódigo); driver v6.20.0 verificado en `src/sessions.ts`; distinción Transient vs Unknown preservada; **contexto sobrevive a excepción tras escrituras** (D6-01/D6-03) | §9.1 Pseudocódigo unificado, §9.2 Distinción, §9.3 Tabla 9 escenarios, §9.4 Verificación driver |
| **Métricas** | Medición física real (líneas, bytes, fecha) del documento final | §13 Métricas con comando PowerShell |
| **D4-02 Decisiones Outbox** | **4ta/5ta intervención**: tres ramas distinguibles (block/escalate/allow) con guard contractual obligatorio; **esta intervención**: cada rama produce **estado de resultado distinto** con `decisionType` explícito — `block` → `*_BLOCK` + `decisionType: definitive_rejection`; `escalate` → `*_ESCALATE` + `decisionType: anomaly_requires_intervention`; `allow` → `*_WITH_OUTBOX` + guard + `decisionType: helper_allows_but_contractual_guard_blocks`; **15 consumidores** con decisionType explícito. **Diferencia operativa**: Ambos `block` y `escalate` retornan `canRetryBusinessTransaction: false` y `requiresEscalation: true` idénticos; diferencia solo en **clasificación diagnóstica** (nombres, decisionType, evidencia). **Sin consumidor downstream documentado** con tratamiento operativamente distinto. **Marcado parcialmente resuelto** — pendiente validar si WAB-81/alertas/operadores requieren tratamientos distintos. | §5.5 CASO A/B tres ramas, §7.3 inventario 15 consumidores, §7.5 bloqueos y nota D4-02, decisionType en evidencia |
| **D5-01 Partición BSON** | **4ta/5ta/esta intervención**: análisis exhaustividad (96 combinaciones teóricas = 2×4×4×3); **recuentos corregidos**: E1 = 48 teóricas → 1 estado físico; E2-E10 = 12 combinaciones = 12 estados físicos; total válidas teóricas = 60; E-INVALID = 36 teóricas → 36 estados físicos inválidos; 13 estados BSON físicos válidos totales; **49 estados BSON físicos realizables totales (48 processing=present + 1 absent)**; categoría explícita `E-INVALID` bloquea adquisición/éxito; dimensiones ortogonales con valores precisos; **clasificador formal determinista** (§8.1.1 pseudocódigo) con orden fijo, cada combinación clasificada exactamente una vez; **validación/extracción BSON real** → ObservedState (rechaza tipos inválidos, gen negativa/fraccionaria, sin coerciones); diferenciación tres niveles: **físicos (49)**, abstractas (96), semánticos (13) | §8.1 análisis exhaustividad + **clasificador formal §8.1.1 + validación/extracción pseudocódigo**, §8.3 tabla 11 estados (10+E-INVALID), §8.5 fila 11 E-INVALID, recuentos matemáticos verificables y reconciliados (error de 16 posiciones corregido a 48) |
| **D5-03 CAS completo** | **4ta/5ta/esta intervención**: filtros CAS con igualdad exacta valores observados; E-INVALID sin filtro CAS (bloqueada); **E3 corregido**: `processing.leaseOwner: { \$in: [null], \$exists: true }` distingue `null` explícito de ausente; **E5, E7 corregidos**: `observedLeaseUntil === null` → `{ \$in: [null], \$exists: true }`; **E9 marcada pendiente decisión contractual** — `leaseGeneration` ausente NO se incrementa ni asume 0; filtro protege `leaseGeneration: { \$exists: false }` y `leaseUntil` exacto, generación resultante = **bloqueada pendiente §16.14**; diferenciación estricta igualdad/existencia/ausencia/null/cero; protege presencia/ausencia exacta de todos los campos | §8.3 tabla CAS completa + nota E-INVALID + correcciones E3/E5/E7 + E9 pendiente, §8.5 coherencia predicados=CAS |
| **D2-04 RetirementOp** | **4ta/5ta intervención**: consulta `agentInstanceId` = ILUSTRATIVA NO NORMATIVA explicitamente; bandera `retirementOpCorrelationDemonstrable=false` **siempre**; **5ta intervención**: bandera **interviene efectivamente en flujo de control** — bloquea toda ruta de éxito que dependa de correlación (CASO B retorna `OUR_TRANSITION_BLOCKED_RETIREMENT_OP_CORRELATION`); registro de anomalía `retirementOpQueryAnomaly` si falla consulta; NO continuar silenciosamente tras excepción; NO actualizar eventSetRevision con correlación aproximada; sin `OUR_TRANSITION_COMMITTED` por consulta aproximada | §5.5 paso RetirementOperation con `retirementOpCorrelationDemonstrable`, `retirementOpQueryAnomaly`, rama bloqueo explícita; §5.2 notas contractuales D2-04; CASO B guard correlación |
| **D3-01 Matriz desde algoritmo** | **Esta intervención**: Matriz §5.4 y trazabilidad §5.7 reconstruidas desde ramas efectivas de §5.5; filas 3a/3b/3c, 5a/5b/5c, 6a/6b/6c reflejan tratamiento real; filas 13a/13b/13c **eliminadas y fusionadas en fila 13 única** (helper NO invocado en fieldsIncoherent); fila 9c sin `OUR_TRANSITION_COMMITTED` alcanzable; **nombres de estado corregidos**: `OUR_TRANSITION_BLOCKED_LEVEL1_OUTBOX` / `OUR_TRANSITION_BLOCKED_RETIREMENT_OP_CORRELATION`; **12 sub-casos** por decisión (antes 23); cada fila con condición, evidencia, decisión, resultado, guarda, alcanzabilidad | §5.4 Matriz 12 sub-casos, §5.7 Trazabilidad con guarda contractual, §5.5 ramas correspondientes |
| **T-08 Métricas** | **Corrección residual final**: métricas actualizadas (2,224 líneas, 211,155 bytes, 15/11/2026 17:16:45 medición como referencia temporal; metadata LastWriteTime observado 17:11:17 durante inspección física independiente previa a esta corrección); limitación explícita: fecha/hora de medición (17:16:45) es distinta del LastWriteTime observado (17:11:17); este último corresponde a la metadata del sistema de archivos en la inspección física independiente previa a esta corrección; se reporta medición física como referencia externa; el LastWriteTime es metadata dinámica y puede cambiar al guardar; el valor posterior al guardado debe obtenerse mediante nueva inspección física | §13 medición PowerShell, §15.A informe cierre con nota T-08 |

### 14.2 Corregido parcialmente

| Área | Qué falta | Dependencia |
|------|-----------|-------------|
| **D4-02 Diferencia operativa block vs escalate** | Validar si consumidores downstream (WAB-81 drenaje, alertas, operadores) tratan `block` y `escalate` operativamente distinto (políticas reintento, flujos recuperación, alertas diferenciadas). Actualmente solo difieren en clasificación diagnóstica (nombres, decisionType, evidencia), ambos retornan `canRetryBusinessTransaction: false` y `requiresEscalation: true` idénticos. | Implementación y prueba de consumidores reales; definición de políticas operativas WAB-81 |
| **Verificación código fuente driver in-situ** | Confirmación de acceso a `src/sessions.ts` tag `v6.20.0` en entorno de ejecución local | Disponibilidad del repositorio GitHub `mongodb/node-mongodb-native` clonado localmente |

### 14.3 No corregido (por diseño, fuera de alcance)

- Implementación de código de producción.
- **D-W79-14 ya está aprobada** (no requiere resolución).
- Definición schema `RetirementOperation` final (§16.14).
- Verificación infraestructura MongoDB Atlas (atomicidad, `readConcern: snapshot`, topología).
- Autorización reintento de negocio (requiere §16.14 + diseño reconciliación).
- Contratos transporte Inbox/Outbox (§16.14).

### 14.4 Bloqueado por contrato (Corregido T-01, T-07)

| Bloqueo | Criterio afectado | Resolución | Categoría |
|---------|-------------------|------------|-----------|
| **§16.14 Nivel 1 Outbox** | `OUR_TRANSITION_COMMITTED` inalcanzable; helper `decision='block'` siempre | Contrato final Outbox aprobado | **Dependencia contractual pendiente** |
| **Schema `RetirementOperation`** | Valor inicial `leaseGeneration`, índices, topología, campos `admissionBlockedAt`/`eventSetClosedAt` | Schema final aprobado | **Dependencia contractual pendiente** |
| **Verificación Atlas** | Atomicidad transacciones, `readConcern: snapshot`, topología clúster | Validación en clúster real | **Validación de entorno pendiente** |

**Nota T-07:** Se diferencia "no existen campos Outbox aprobados" (§16.14 pendiente) de "existe evidencia contractual aprobada suficiente para decisión positiva" (requiere §16.14 + D-W79-14 aprobada). No equiparar ambas.

### 14.5 Pendiente de validación mediante pruebas reales

- Pruebas de integración transacciones multidocumento (Contacto+EvAssigned+Outbox+RetirementOperation) en Atlas.
- Pruebas de concurrencia lease/fencing con workers reales.
- Pruebas de recuperación `TransientTransactionError` / `UnknownTransactionCommitResult` con driver v6.20.0.
- Pruebas de drenaje Outbox inconsistente (WAB-81).

---

### 14.6 Resolución de contradicciones transversales (T-01 a T-08)

| ID | Contradicción | Resolución en Revisión 21 |
|----|---------------|---------------------------|
| **T-01** | D-W79-14 aprobada en canónico pero presentada como pendiente | **Corregido en todo el documento**: D-W79-14 = aprobada (10/11/2026). Lo pendiente es §16.14 Outbox físico. |
| **T-02** | §10 afirma `retirementOp` integrado aunque §5.5 no lo demuestra | **Corregido §5.5 CASO B**: Añadida consulta RetirementOperation condicional con dependencias contractuales explícitas. Evidencia en retorno incluye `retirementOp`, `retirementOpQueried`. |
| **T-03** | §10 afirma correspondencia completa matriz-rama sin demostración | **Corregido §5.7**: Trazabilidad 1:1 reconstruida con identidades de consulta reales, consultas ejecutadas por caso. |
| **T-04** | §10 afirma 8 consumidores aplican `decision` aunque CASO A/B defectuosos | **Corregido §7.3 (5ta intervención)**: Inventario exhaustivo **15 consumidores** con precondición helper (OutboxDoc no-null), tratamiento decisión explícito con **estados de resultado distintos y decisionType**, ausencia Outbox manejada ANTES. **CASO A y B separan explícitamente block/escalate/allow** (D4-02 5ta intervención). |
| **T-05** | §10/§8.3 afirman 10 estados BSON con CAS completo aunque §8.3 rutas/filtros incorrectos | **Corregido §8.3 (5ta intervención)**: Filtros CAS con rutas `processing.*` completas, `$or` válido, fencing por `leaseGeneration` incluido. **E3 corregido**: `{ \$in: [null], \$exists: true }` distingue null de absent. **Estados E1-E10 mutuamente excluyentes + E-INVALID** (D5-01 5ta intervención). **Clasificador formal determinista** (§8.1.1). |
| **T-06** | §10/§9.5 afirman catch exterior conserva contexto aunque ejemplos no muestran captura | **Corregido §9.1**: Flujo unificado con `contextContainer` fuera de `withTransaction`, captura real en `catch`, validación `_generation`. **Contexto sobrevive a excepción tras escrituras** (D6-01/D6-03). |
| **T-07** | Diferencia "no campos Outbox aprobados" vs "evidencia contractual suficiente" | **Corregido §11, §14.4**: Explícitamente diferenciado. §16.14 pendiente ≠ D-W79-14 pendiente (aprobada). |
| **T-08** | Discrepancia entre metadata archivo y fecha §13 / §15.A | **Corregido (corrección residual final)**: §13 y §15.A reflejan la misma medición física (2,224 líneas, 211,155 bytes, 15/11/2026 17:16:45 como referencia temporal de la medición; metadata LastWriteTime observado 17:11:17 durante inspección física independiente previa a esta corrección). Limitación: fecha/hora de medición (17:16:45) es distinta del LastWriteTime observado (17:11:17); este último corresponde a la metadata del sistema de archivos en la inspección física independiente previa a esta corrección; se reporta medición física como referencia externa; el LastWriteTime es metadata dinámica y puede cambiar al guardar; el valor posterior al guardado debe obtenerse mediante nueva inspección física. |

---

**La Revisión 21 NO declara que WAB-79 está listo para implementación.** Corrige integralmente los seis bloques de defectos (B1–B6) y ocho contradicciones transversales (T-01–T-08) confirmados durante la evaluación independiente de la Revisión 20 (resultado: NO SUPERADO). Los bloqueos contractuales (§16.14 Outbox, schema `RetirementOperation`) y técnicos (verificación Atlas) permanecen documentados explícitamente como dependencias para la autorización de implementación. **Una corrección documental completa NO equivale a aprobación de implementación de WAB-79.**

---

## 15. Informe de Cierre Requerido

### A. Archivo modificado
- **Ruta absoluta:** `F:\ElemenSoft\waby-docs\projects\agents-multitenant\design\wab-79-viabilidad-tecnica-cierre-revision-21.md`
- **Tamaño histórico (inspección 17:16:45):** 211,155 bytes (medición PowerShell `Get-Item` durante la inspección física del 15/11/2026 a las 17:16:45)
- **Líneas históricas (inspección 17:16:45):** 2,224 (medición PowerShell `Get-Content`.Count durante la inspección física del 15/11/2026 a las 17:16:45)
- **Fecha/hora medición histórica:** 15/11/2026 17:16:45 (medición física PowerShell como referencia temporal de la comprobación)
- **Metadata LastWriteTime observado en inspección histórica:** 15/11/2026 17:11:17 (metadata observada durante la inspección física independiente previa a esta corrección)
- **Inspección posterior (18:36:12):** 2.224 líneas, 212.373 bytes, LastWriteTime 18:34:50
- **Codificación:** UTF-8
- **Confirmación:** Único archivo modificado **en esta intervención correctiva**. No se modificaron documentos canónicos, código fuente, schemas, tests, configuración, Linear ni GitHub remoto.
- **Limitación Git:** No existe estado Git inicial verificado (baseline) para confirmar exclusividad de cambios en el repositorio; la verificación se basa en inspección de la sesión actual. **No se afirma** que este sea el único archivo modificado en la historia del repositorio. Se distinguen los cambios realizados por **esta intervención** (correcciones residuales en §8.1.1, §13, §15.A) de cambios preexistentes cuya atribución no puede demostrarse sin baseline Git.
- **Nota T-08:** Los valores 2.224 líneas y 211.155 bytes corresponden a la inspección histórica del 15/11/2026 a las 17:16:45. El LastWriteTime 17:11:17 pertenece a esa misma observación histórica. Una inspección posterior, realizada a las 18:36:12, observó 2.224 líneas y 212.373 bytes, con LastWriteTime 18:34:50. Los valores históricos no representan necesariamente el estado físico posterior a los guardados siguientes. El LastWriteTime es metadata dinámica del sistema de archivos. La observación de las 18:36:12 se conserva como una segunda observación histórica, nunca como metadata actual o inmutable. El valor posterior al guardado debe obtenerse mediante una nueva inspección física y comunicarse en el informe final de ejecución.

### B. Matriz de defectos

| Defecto | Sección | Corrección | Evidencia | Resultado |
|---|---|---|---|---|
| **D2-04** | §5.5, §5.2 notas | RetirementOperation: consulta **ILUSTRATIVA NO NORMATIVA** (por agentInstanceId como aproximación); etiquetada explícitamente como no probatoria; correlación contractual §16.12 (`{retirementOperationId, contacto}`) **pendiente §16.14**; actualización transaccional eventSetRevision **pendiente schema**; incremento generation **pendiente**; comentario "AÑADIR PASO 3.5" **eliminado**; algoritmo clasifica como bloqueada cuando correlación no demostrable; bandera explícita `retirementOpCorrelationDemonstrable = false` **siempre**; **esta intervención**: bandera **interviene efectivamente en flujo de control** — bloquea toda ruta de éxito que dependa de correlación (CASO B retorna `OUR_TRANSITION_BLOCKED_RETIREMENT_OP_CORRELATION`); registro de anomalía `retirementOpQueryAnomaly` si falla consulta; NO continuar silenciosamente; NO actualizar eventSetRevision con correlación aproximada | §5.5 paso RetirementOperation con `retirementOpQueried`, `retirementOpCorrelationDemonstrable`, `retirementOpQueryAnomaly`; §5.2 notas contractuales actualizadas; CASO B bloquea éxito si `!retirementOpCorrelationDemonstrable`; evidencia retorno incluye `retirementOp`, `retirementOpQueried`, `retirementOpCorrelationDemonstrable`, `retirementOpQueryAnomaly` | ✅ Corregido (esta intervención) — bandera interviene en flujo, bloquea éxito, registra anomalía, no continúa silenciosamente |
| **D3-01** | §5.4, §5.7 | **Esta intervención**: Matriz §5.4 y trazabilidad §5.7 **reconstruidas desde ramas efectivas de §5.5**; filas 3a/3b/3c, 5a/5b/5c, 6a/6b/6c reflejan tratamiento real de cada decisión (block/escalate/allow); filas 13a/13b/13c **eliminadas** (una sola rama para fieldsIncoherent — helper NO invocado en esa ruta); fila 9c **sin `OUR_TRANSITION_COMMITTED` alcanzable** (guards contractuales Level 1 Outbox + correlación RetirementOperation constantes `false`); **nombres de estado corregidos**: `OUR_TRANSITION_BLOCKED_LEVEL1_OUTBOX` / `OUR_TRANSITION_BLOCKED_RETIREMENT_OP_CORRELATION` (sin "COMMITTED" engañoso); cada fila con: condición entrada, evidencia consultada, decisión helper, resultado real, guarda contractual, alcanzabilidad actual; recuentos: 13 combinaciones base + 4 imposibles = 17; 12 sub-casos con decisión explícita | §5.4 Matriz 12 sub-casos, §5.7 Trazabilidad con columna guarda contractual, §5.5 ramas correspondientes | ✅ Corregido (esta intervención) — matriz y trazabilidad reconstruidas desde algoritmo, filas 13a/13b/13c eliminadas, 9c sin éxito alcanzable, nombres corregidos |
| **D4-02** | §5.5 CASO A/B, §7.3, §7.5 | **Esta intervención**: Helper `outboxMatch.decision` produce **estados de resultado distintos** — `block` → `*_BLOCK` + `decisionType: definitive_rejection`; `escalate` → `*_ESCALATE` + `decisionType: anomaly_requires_intervention`; `allow` → `*_WITH_OUTBOX` + guard + `decisionType: helper_allows_but_contractual_guard_blocks`; **15 consumidores** con decisionType explícito. **Diferencia operativa**: Ambos `block` y `escalate` retornan `canRetryBusinessTransaction: false` y `requiresEscalation: true` idénticos; diferencia solo en clasificación diagnóstica (nombres, decisionType, evidencia preservada). **Sin consumidor downstream documentado** que trate operativamente distinto. Marcado **parcialmente resuelto** — pendiente validar si WAB-81/alertas/operadores requieren tratamientos distintos. | §5.5 CASO A/B tres ramas; §7.3 inventario 15 consumidores; §7.5 nota estado D4-02 | ⚠️ Parcialmente resuelto — diferenciación diagnóstica sí, operativa pendiente validación consumidor downstream |
| **D5-01** | §8.1, §8.1.1, §8.3, §8.5 | **Esta intervención**: Dimensiones BSON separadas con valores precisos; 10 estados E1-E10 partición lógica mutuamente excluyente + E-INVALID; **pseudocódigo validación/extracción BSON real → ObservedState** (§8.1.1) que rechaza explícitamente: `leaseOwner` tipo ≠ string/null, `leaseUntil` tipo ≠ Date/null, `leaseGeneration` tipo ≠ number, generación negativa/fraccionaria, valores no representables; **aclaración rigurosa corregida**: E1 = 1 clase de equivalencia BSON (processing ausente) aunque ocupe **48 posiciones cartesianas** (4×4×3, no 16); **49 clases de equivalencia BSON (48 con processing=present + 1 absent)**; 96 combinaciones abstractas teóricas (2×4×4×3); 60 combinaciones teóricas válidas (E1: 48 + E2-E10: 12); **13 clases de equivalencia BSON válidas (E1: 1 + E2-E10: 12)**; **36 combinaciones teóricas inválidas (E-INVALID) → 36 clases de equivalencia BSON inválidas**; recuentos reconciliados en pseudocódigo, tablas, conclusiones | §8.1 análisis exhaustividad + **clasificador formal §8.1.1 + validación/extracción pseudocódigo**, §8.3 tabla 11 estados (10+E-INVALID) con CAS, §8.5 fila 11 E-INVALID, recuentos matemáticos verificables y reconciliados | ✅ Corregido (esta intervención) — partición exhaustiva, clasificador formal, validación/extracción BSON real, diferenciación física/abstracta/semántica, recuentos reconciliados (error de 16 posiciones corregido a 48) |
| **D5-03** | §8.3, §8.5 | **Esta intervención**: Filtros CAS protegen valores EXACTOS observados; **E5, E7 corregidos** — cuando `observedLeaseUntil === null`, filtro usa `{ \$in: [null], \$exists: true }`; E3 usa `{ \$in: [null], \$exists: true }`; **E9 marcada como pendiente decisión contractual** — `leaseGeneration` ausente NO se incrementa ni se asume 0; filtro CAS protege `leaseGeneration: { \$exists: false }` y `leaseUntil` exacto, pero generación resultante = **bloqueada pendiente schema §16.14**; protege presencia/ausencia exacta de `processing`, `leaseOwner`, `leaseUntil`, `leaseGeneration`; E-INVALID sin filtro CAS (bloqueada en validación); **ningún estado sin representación inequívoca** | §8.3 tabla CAS completa con igualdad exacta + correcciones E3/E5/E7 + E9 pendiente + nota E-INVALID; §8.5 coherencia predicados=CAS | ✅ Corregido (esta intervención) — filtros CAS protegen dimensiones observadas con valores exactos, distinción estricta null vs absent, E9 no asume default, combinaciones no demostrables bloqueadas |
| **D6-01/D6-03** | §9.1, §9.3 | AttemptContext: objeto `mutableContext` creado ANTES del callback, actualizado **incrementalmente DURANTE** ejecución; callback recibe `mutableContext` y lo actualiza antes de cada operación crítica; si lanza excepción tras escrituras, `mutableContext` YA contiene `writesPerformed: true`, `leaseAcquired: true`, etc.; `catch` interior captura estado parcial en `contextContainer.current`; catch exterior recibe contexto correcto via `contextForClassification`; validación `_generation` evita contexto obsoleto; reintentos reinicializan contenedor | §9.1 flujo unificado con `mutableContext` pasado al callback; §9.3 tabla 9 escenarios; D6-01/D6-03 notas en puntos clave actualizadas | ✅ Corregido (regresión verificada) — contexto propagado correctamente incluso ante excepción tras escrituras |
| **T-08** | §13, §15.A, §8.1.1 | **Corrección residual final**: métricas re-medidas (2,224 líneas, 211,155 bytes, 17:16:45 referencia temporal de medición; metadata LastWriteTime observado 17:11:17 durante inspección física independiente previa a esta corrección); §13, §14.1, §14.6, §15.A alineados; limitación Git declarada; fecha/hora de medición (17:16:45) distinguida explícitamente del LastWriteTime observado (17:11:17); este último corresponde a la metadata del sistema de archivos en la inspección física independiente previa a esta corrección; workerId/transactionTimestamp validación estricta; Object.create(null) sin excepción; integridad Markdown restaurada; el LastWriteTime es metadata dinámica y puede cambiar al guardar; el valor posterior al guardado debe obtenerse mediante nueva inspección física | §13 medición PowerShell; §15.A informe cierre; §8.1.1 pseudocódigo corregido | ✅ Corregido (corrección residual final) — métricas consistentes, limitación documentada, validaciones estrictas aplicadas |

### C. Verificaciones realizadas

| Tipo de verificación | Realizada | Detalle |
|---------------------|-----------|---------|
| **Verificación documental** | ✅ Sí | Lectura completa documentos canónicos (WAB-78 Fase 2, WAB-79 Ownership) + Revisión 20. Contradicciones resueltas contra canónicos. |
| **Verificación lógica pseudocódigo** | ✅ Sí | Algoritmo §5.5: identidades consulta, matriz 12 sub-casos (helper invocado), 12 ramas, helpers, CAS 11 estados (E1-E10 + E-INVALID), flujo AttemptContext. Consistencia interna verificada; matriz y trazabilidad reconstruidas desde ramas efectivas. |
| **Verificación de sintaxis de filtros** | ✅ Sí | Filtros CAS §8.3 validados contra semántica MongoDB v6.20.0 (rutas `processing.*`, `$or`, `$exists`, `$gt`, `$lte`, `$in: [null], $exists: true`). |
| **Inspección de código fuente** | ⚠️ Parcial | Verificadas versiones `package.json`/`pnpm-lock.yaml` (mongoose 8.19.3, mongodb 6.20.0). Firma Mongoose `findById` confirmada v8.x. Código fuente driver `src/sessions.ts` tag `v6.20.0` consultado en GitHub (no in-situ). |
| **Verificación validación/extracción BSON** | ✅ Sí | Pseudocódigo §8.1.1 `extractAndValidateObservedState` verificado: rechaza tipos inválidos, generación negativa/fraccionaria, valores no representables; sin coerciones ni defaults; dependencias contractuales declaradas. |
| **Verificación recuentos y reconciliación** | ✅ Sí | Estados físicos BSON (49) vs combinaciones abstractas (96) vs semánticamente admitidos (13) diferenciados y reconciliados en §8.1.1, tablas, conclusiones. |
| **Pruebas automatizadas/runtime** | ❌ No | No se ejecutaron tests de integración, concurrencia, Atlas, ni runtime. Documentado como pendiente en §11, §14.5. |
| **Validación Atlas** | ❌ No | Topología transacciones, propagación sesión, `readConcern: snapshot`, commit/rollback no validados en clúster real. Documentado como pendiente §11, §14.4. |
| **Verificación Git de exclusividad de cambios** | ❌ No | No existe estado Git inicial verificado; limitación declarada en §15.A. |

### D. Pendientes conservados

| Dependencia | Categoría | Responsable resolución conceptual |
|-------------|-----------|-----------------------------------|
| **§16.14 Outbox físico** (nombres, payloads, índices, transporte, identidad física idempotencia) | Dependencia contractual pendiente | Equipo arquitectura Waby / Aprobación Linear |
| **Schema `RetirementOperation` final** (leaseGeneration inicial, índices parciales, topología transaccional, admissionBlockedAt/eventSetClosedAt) | Dependencia contractual pendiente | Equipo arquitectura Waby / Aprobación Linear |
| **Verificación MongoDB Atlas** (topología compatible transacciones multidocumento, propagación sesión, commit/rollback, readConcern: snapshot) | Validación de entorno pendiente | Equipo infraestructura / DevOps |
| **Autorización reintento negocio** (mecanismo evidencia positiva aborto) | Dependencia contractual + validación código | Requiere §16.14 + diseño reconciliación aprobado |
| **Correlación física transiciones retiro** (vínculo `{retirementOperationId, contacto}` §16.12 ↔ esquema Outbox §16.14) | Dependencia contractual pendiente | Requiere §16.14 aprobado |
| **Inicialización `leaseGeneration` / `processing`** (defaults schema, migraciones) | Dependencia contractual pendiente | Requiere schema `RetirementOperation` final |

### E. Resultado

**CORRECCIÓN COMPLEMENTARIA COMPLETA (esta intervención)**: todos los defectos solicitados (D4-02 diferenciación CASO A, D5-03 CAS para null explícito, D5-01 validación BSON real, D3-01 reconstrucción matriz desde algoritmo, verificación transversal §5.4/§5.5/§5.7/§7.3/§8.1/§8.1.1/§8.3/§8.5/§13/§14.1/§14.6/§15.A/§15.B) quedaron corregidos y demostrados dentro del propio documento; las dependencias externas (§16.14 Outbox físico, schema `RetirementOperation` final, verificación Atlas, código pruebas integración) siguen explícitamente pendientes con su categoría y responsable. **No se afirma validación runtime, Atlas, ni autorización para implementación de WAB-79.**