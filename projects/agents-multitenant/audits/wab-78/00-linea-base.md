# WAB-78 — Línea base de la revisión por bloques

**Fecha de verificación:** 04/11/2026
**Sesión:** individual 1 — Bloque 1 (D1), Dominio y ownership
**Propósito de este archivo:** fijar el estado de control, las decisiones que no se reabren, el inventario de fuentes y las convenciones de identificadores, para que cualquier sesión independiente continúe sin depender del historial de conversación.

---

## 1. Estado de control verificado (04/11/2026)

| Elemento | Estado verificado | Regla durante la revisión |
|---|---|---|
| WAB-75 — Plan general | In Progress | No cambiar |
| WAB-76 — Fase 0 | In Progress | Gate de aprobación propio; no cambiar estado |
| WAB-77 — Fase 1 | Done | Auditoría concluida; no repetir (reutilizar AUD-01 y entregables) |
| **WAB-78 — Fase 2** | **In Progress** | Permanece abierto hasta aprobación expresa; no cerrar |
| **WAB-79 — Fase 3** | **Backlog, bloqueada por WAB-78** | No iniciar, no cambiar estado |
| WAB-80 a WAB-86 | Backlog | No iniciar |
| WAB-87 a WAB-92 (AUD-01..06) | Done | No repetir |
| Proyecto Agentes de IA | Backlog | No cambiar |

**Verificación:** estados leídos en Linear el 04/11/2026 antes de iniciar el Bloque 1. Esta sesión **no modificó** ningún issue, descripción, relación, estado ni documento de Linear.

---

## 2. Decisiones aprobadas — no reabrir (línea base)

Procedencia: **WAB-78 — Registro de decisiones de producto y operación (03/11/2026)** (Linear, documento `345541575496` / `3599e1e4`) y **WAB-78 — Plan de revisión integral** (Linear, `fca2514ac4ee`) §2. Se verifica su correcta integración en el diseño; no se debaten de nuevo.

| Decisión | Contenido aprobado (resumen) | Relevancia para D1 |
|---|---|---|
| **P1** | Sin cupo comercial de AgentInstance por Organización ni Workspace; limitación económica por bolsa única `OrganizationAICredit`. **Consecuencia documental:** eliminar I3 y los apartados de cupos/gracia; no mantener gracia de reducción de cupo. | Invalida cupos/gracia/índices de cupo en el canónico (ARQ-11, ARQ-16) |
| **P2** | Recepción automática, asignación, transferencia, delegación, takeover, fallback y distribución aprobados, **con pendientes expresos** (§4.1: ¿más de una predeterminada?, selección, reemplazo atómico). | `isDefault` sin regla de unicidad (ARQ-13) |
| **P4** | Política híbrida G.4.6; bolsa única; `committedBudget`; ADR-002 y ADR-006 aprobados 03/11/2026. | Ownership de créditos: Organización |
| **P9** | Cinco estados: `draft`, `active`, `suspended`, `retirement_pending`, `retired` + reglas 1-6. Pendiente: reversibilidad de `retirement_pending`. | Enum de 4 estados en A.2 (ARQ-12) |
| **P10** | Snapshot inmutable de configuración efectiva por ejecución; cambios aplican a ejecuciones nuevas. Versionado/retención de Knowledge conserva pendientes. | ADR-005 sigue "Propuesto" (ARQ-17) |
| **Ownership AgentInstance** | `AgentInstance` se **persiste en CRM**; cadena `AgentInstance → Workspace → Organización`; `Contacto.ref_agent_assigned` referencia tipada a `AgentInstance._id`. | Base de la respuesta de ownership hasta Organización |
| **CRM autoridad** | CRM es única autoridad de la asignación efectiva; `EvAssigned` es historial funcional exclusivo de CRM (discriminator de `Evento`, colección `eventos`); `AssignedType` = `human`/`agent`/`unassigned`; sin `assignmentVersion`. | Relación Contacto/EvAssigned (ARQ-18) |
| **Transacción preferida** | Contacto + EvAssigned + Outbox en una transacción MongoDB; HTTP a ai-service tras el commit. **No verificada en Atlas.** | Dominio D3/D6, registrado aquí como contexto |
| **`contactAssignmentHistory`** | Se conserva en esta fase; no eliminar. | Omisión del canónico (ARQ-18) |
| **P3 — permisos** | **Sin decisión.** Debe auditarse el sistema de permisos actual. **No introducir `bot.configure`** hasta concluir esa auditoría. | Contradicción con D.1/D.5 del canónico (evidencia de ARQ-03, actualización en informe D1) |

**No reabrir:** P1, P2, P4, P9, P10. **No solicitar** decisiones de producto en las sesiones de diagnóstico: solo registrarlas.

### Pendientes de producto ya registrados (contexto, no se reabren ni se resuelven aquí)

- P2 §4.1: múltiples instancias predeterminadas / selección / reemplazo.
- P2 §4.3 y §4.4: permisos `wrk.teams.*` para distribución (auditoría técnica).
- P9: reversibilidad de `retirement_pending`.
- P3: auditoría de permisos en GitHub antes de definir la matriz.
- P10: política de versionado/publicación/retención de Knowledge.
- §8.2: `operationType` de EvAssigned (propuesta), contrato G3/G4, ventana de backfill, compatibilidad Atlas.

---

## 3. Inventario de fuentes (con nivel y estado)

| Fuente | Ruta / referencia | Nivel | Estado |
|---|---|---|---|
| Plan de revisión (metodología, formato de hallazgos) | Linear: `wab-78-plan-de-revision-integral-por-bloques-y-matriz-de-preparacion-fca2514ac4ee` | Primario (método) | Vigente |
| Registro de decisiones 03/11/2026 | Linear: documento `345541575496` (`3599e1e4`) | Primario (decisiones) | Vigente |
| Documento canónico Fase 2 | `crm/docs/plans/wab-75-fase-2-diseno.md` | Primario (diseño) | **Vigente con secciones obsoletas** (ver §4) |
| Informe Fase 1 (as-is) | `crm/docs/plans/wab-75-fase-1-auditoria.md` | Primario (evidencia as-is) | Vigente; AUD-01..06 no se repiten |
| Plan general WAB-75 | `crm/docs/plans/wab-75-agentes-administrados-por-waby.md` | Primario (invariantes) | **I3 superseded por P1** (ver §4) |
| Complemento técnico WAB-78 | `WAB-78-complemento-tecnico.md` (raíz del workspace) | Primario (EvAssigned, Parte D/E) | Vigente; su premisa "ai-service no disponible" es incorrecta (ver §5) |
| Auditoría de permisos | `crm/docs/reports/AUDITORIA_PERMISOS_WABY.md` (§2.4, H9) | Primario (permisos) | Vigente; útil para D2 |
| Código CRM | `crm/src/**` (ver §6 del informe para evidencias puntuales) | Primario (estado actual) | Verificado en esta sesión |
| ai-service (local) | `crm/microservices/ai-service/` | Primario (estado actual) | **Disponible localmente** (ver §5) |
| Análisis previo D2–D4 del usuario | `analisis-previo-d2-d4.md` (esta carpeta) | **Preliminar** | Sin validar; sujeto a verificación por dominio |

---

## 4. Fuentes superseded / con secciones obsoletas (registrar, no resolver unilateralmente)

Estas contradicciones se documentan con evidencia e impacto en los hallazgos; **no se editan documentos canónicos en la revisión diagnóstica** (restricción del plan §7 y §10).

1. **Cupos por plan (P1 vs canónico):** el canónico mantiene "cupo por plan" (§1 L13), lista P1 como pendiente (L17), tabla de cupos Básico/Profesional (§C.2 L451-460), gracia de 5 días (§C.2.1 L464-471), restricción de integridad #2 (L432), cardinalidad "1 o 2 instancias" (L97), flujo de creación con verificación de cupo (§C.3 L558), concurrencia/índice de cupo (§C.5 L570-571), transiciones de ciclo de vida condicionadas a cupo/gracia (§C.6 L574-616) y Fase 3 con "índices de cupo" (L1792). → **ARQ-11**.
2. **Invariante I3 de WAB-75 (L24, L70-76) y BR-AI-009 (`docs/business/bot-routing.md:130`, citada por Fase 1 §8):** siguen sin marcarse como superseded pese a la consecuencia documental de P1. → **ARQ-16**.
3. **Enum de estados de AgentInstance:** A.2 declara 4 estados etiquetados "(P9)" (L82); C.6 usa 5 estados incluyendo `retirement_pending` (L581-611); P9 aprobado define 5 estados. → **ARQ-12**.
4. **`isDefault`:** campo opcional sin regla de unicidad (A.2 L84); única regla existente (C.6 L614) depende de la gracia de cupo obsoleta; P2 §4.1 exige "al menos una predeterminada" con integridad atómica y tiene 3 pendientes. → **ARQ-13**.
5. **Responsabilidades CRM/ai-service:** D.1 (L694-708) mezcla superficie de API y persistencia; solo la persistencia de AgentInstance está aprobada explícitamente (registro §8.1). → **ARQ-14**.
6. **`bot.configure`:** D.1 (L702), D.4 (L859) y D.5 (L862-868) asumen el permiso, mientras P3/§6 prohíbe introducirlo hasta auditar. → evidencia adicional a **ARQ-03** (sin nuevo ID).
7. **Estados de ADRs:** ADR-001, ADR-003, ADR-004 y ADR-005 siguen "Propuesto" (L1811, L1884, L1900, L1917) pese a decisiones aprobadas de línea base (P10 → ADR-005; separación AgentDefinition/AgentInstance → ADR-001). → **ARQ-17**.
8. **Inventario de entidades:** A.2 y el diagrama B no incluyen `EvAssigned` ni `contactAssignmentHistory`, definidos/aprobados en D.1.6 y en el complemento §8.1. → **ARQ-18**.

---

## 5. Observación de fuente: ai-service disponible localmente

El complemento técnico (cabecera) y el registro §8.2 afirman que "ai-service no disponible localmente". **Verificado el 04/11/2026:** el repositorio existe en `crm/microservices/ai-service/` con `src/` (domain, engine, infrastructure, application, presentation, observability), `dist/`, `tests/`, `docs/`, `package.json`, `jest.config.js`. El informe Fase 1 (§2.1) leyó sus entidades exactamente desde esa ruta.

**Consecuencia:** la clasificación **ARQ-07** ("No verificado — repositorio no disponible") conserva su clasificación original hasta que la sesión D3 decida revisitarla con evidencia primaria del repo local. No se modifica aquí. Ver `analisis-previo-d2-d4.md` (observación de fuente).

---

## 6. Convenciones de identificadores

- **ARQ-01 y ARQ-02:** no localizados en las fuentes disponibles. **IDs reservados**: no se renumeran ni se reasignan.
- **ARQ-03 a ARQ-10:** análisis preliminar D2–D4 del usuario (procedencia en `analisis-previo-d2-d4.md`). Se conservan IDs, clasificaciones y conclusiones originales, marcados como **preliminares sin validar**.
- **ARQ-11 a ARQ-18:** emitidos por esta sesión (Bloque 1 / D1). Cualquier hallazgo nuevo posterior continúa desde ARQ-19.
- **H1–H7** (permisos), **R1–R15** (riesgos Fase 1), **I1–I21** (invariantes WAB-75), **AUD-01..06**, **P1–P10**: son referencias de otras auditorías/decisiones, **no identificadores ARQ**. No renumerar ni convertirlos en ARQ.
- Formato de hallazgo: plan de revisión §8 (10 campos: ID, Dominio, Estado, Evidencia, Hallazgo, Impacto, Bloquea, Justificación, Destino, Recomendación).

---

## 7. Restricciones operativas de la revisión (recordatorio)

- No modificar código, schemas, modelos, datos, tests, infraestructura.
- No editar el documento canónico, el complemento ni el registro de decisiones durante el diagnóstico.
- No cambiar estados ni descripciones en Linear; no crear issues; no iniciar WAB-79.
- No repetir AUD-01..06; no reabrir P1/P2/P4/P9/P10; no introducir propuestas como decisiones aprobadas.
- Un pendiente técnico solo bloquea WAB-78 si impide definir frontera, ownership, garantía o contrato necesario para la fase dependiente (plan §8). **No bloquear por mera ausencia de código.**
- Artefactos de trabajo (esta carpeta) ≠ documentos canónicos.

---

**Continuidad:** la siguiente sesión debe leer este archivo, `matriz-acumulativa.md`, el informe del bloque anterior y las fuentes citadas; no depende del historial de esta conversación.

