# WAB-78 — Análisis previo D2–D4 (material preservado)

## Procedencia y nivel del material

| Campo | Valor |
|---|---|
| **Procedencia** | Análisis preliminar compartido por el usuario en una sesión anterior (revisión previa de WAB-78). Transcrito textualmente en la sesión individual 1 (Bloque 1 / D1) porque esta sesión no dispone del historial anterior. |
| **Fecha / contexto** | Revisión previa de WAB-78 (anterior al plan de revisión por bloques del 04/11/2026). |
| **Nivel** | **Hallazgos preliminares, sujetos a verificación por dominio.** No constituyen evidencia primaria verificada. |
| **IDs originales** | ARQ-03 a ARQ-10 (conservados sin renumerar ni duplicar). |
| **Límite** | **No sustituye la evidencia primaria** (informe Fase 1, auditoría de permisos, canónico, registro de decisiones, código). Cuando se reutilice un hallazgo, conservar su fuente y nivel de verificación (plan de revisión §6). |

> **Reglas de conservación aplicadas:** las clasificaciones originales (crítico/alto/medio, bloquea/no bloquea) se mantienen tal cual, sin marcarlas como validadas. H1–H7, R1–R15 e I1–I21 son referencias de auditorías anteriores, **no identificadores ARQ**. Las decisiones P2 pendientes listadas más abajo siguen siendo pendientes de producto: se registran, no se solicitan ni se resuelven en esta revisión.

---

## D.2 — Provisioning y permisos

**Conclusión preliminar:** el modelo de permisos actual (`org.*` / `wrk.*` con wildcards, flujo T0→T5, T3 ResourceOwnership) está bien definido y testeado. Sin embargo, P3 no existe en implementación: no hay `bot.configure`, no hay `AgentInstance` ni tipo `AGENT_INSTANCE` en `ResourceOwnership`.

**ARQ-03 — Crítico, bloquea:**
Sin permisos definidos ni entidad AgentInstance, WAB-79/WAB-80 no pueden implementar provisioning, activación, suspensión ni configuración.

**ARQ-04 — Medio, no bloquea:**
Hallazgos H1, H4, H5 y H7:
- H1: roles sin `ref_organizacion`, contradicción aparente con BR-MT-001.
- H4: `wrk.workspaces.read` sin resolver.
- H5: Manager no puede invitar/quitar miembros.
- H7: WebSocket no actualiza el alcance de acceso.
Estos asuntos fueron considerados diferibles a WAB-80, sujetos a verificación y aclaraciones previas.

La auditoría de permisos también indica que no existen `OrganizationAICredit`, `AgentInstance` ni `bot.configure` en el código inspeccionado.

---

## D.3 — Routing y asignaciones

**Conclusión preliminar:** CRM es autoridad de asignación efectiva mediante Contacto + EvAssigned + Outbox. ai-service mantiene `contact_assignments` como proyección operativa. La resolución de ABA mediante identidad de EvAssigned está aprobada conceptualmente.

**ARQ-05 — Alto, bloquea:**
No está especificado cómo se obtiene la identidad de EvAssigned vigente: consulta del último evento frente a referencia directa en Contacto.

**ARQ-06 — Crítico, bloquea:**
La transacción Contacto + EvAssigned + Outbox no está verificada en MongoDB Atlas.

**ARQ-07 — Alto, bloquea:**
La cancelación en ai-service no está verificada. El repositorio no está disponible localmente.

**ARQ-08 — Medio, bloquea:**
Falta definir cómo se propaga `evAssignedId` como metadata a ai-service y cómo G3/G4 lo valida.

**ARQ-09 — Medio, no bloquea:**
`operationType` es una propuesta no aprobada. Se considera diferible a WAB-79.

**Decisiones P2 todavía pendientes:**
- ¿Puede existir más de una instancia predeterminada?
- ¿Cómo se selecciona una instancia cuando hay varias?
- ¿Cómo se reemplaza una designación sin dejar al Workspace sin receptor?

Preguntas técnicas identificadas:
1. Fuente de verdad: CRM/Contacto, definido.
2. Identidad EvAssigned vigente: pendiente.
3. Unicidad concurrente de transiciones: requiere definir un mecanismo seguro.
4. Dos workers intentando asignar el mismo contacto: requiere resolver concurrencia.
5. Respuesta antigua después de takeover: invalidación mediante G3/G4.
6. Proyección ai-service desactualizada: validar contra CRM.
7. Contrato G3/G4: pendiente.
8. CRM no disponible durante validación: comportamiento no definido.
9. Ciclos de delegación: se contempla `delegationChainId` y detección de ciclos.
10. Ejecuciones no abortables: invalidación de resultados tardíos es obligatoria.

Importante: la propuesta de CAS dentro de una transacción es una alternativa que debe investigarse, no una decisión aprobada.

---

## D.4 — Runtime y seguridad

**Conclusión preliminar:** la jerarquía de prioridad del contexto está definida en F.1 y las fronteras de confianza están documentadas. Knowledge se trata como contenido de usuario, no como instrucciones `system`. Policy Engine y output filter no existen en la implementación inspeccionada.

**ARQ-10 — Medio, no bloquea:**
Policy Engine y output filter sin implementar. Su implementación se considera diferible a WAB-81, siempre que sus contratos de seguridad estén definidos.

Otros puntos identificados:
- El cliente no puede anular políticas de plataforma mediante Knowledge, Skills o Rules (F.3).
- No hay acceso exterior implícito (I10).
- Las credenciales deben permanecer protegidas (I6, G.2).
- El snapshot de configuración efectiva por ejecución está aprobado (P10).
- El aislamiento por `workspaceId` está aprobado, pero debe verificarse durante la implementación de WAB-79.

Estos hallazgos son antecedentes de la revisión, no sustituyen las fuentes originales.

---

## Observación de fuente (Sesión D1, 04/11/2026) — para D3

**No altera ARQ-07 ni su clasificación original.** Se registra como evidencia para que la sesión D3 decida:

- **Premisa del material:** ARQ-07 afirma "El repositorio no está disponible localmente"; el complemento técnico (cabecera, línea 4) y el registro de decisiones §8.2 repiten esa premisa.
- **Verificación de esta sesión:** ai-service **sí está disponible localmente** en `F:\ElemenSoft\crm\microservices\ai-service\` (carpetas `src/domain`, `src/engine`, `src/infrastructure`, `src/application`, `src/presentation`, `src/observability`, además de `dist/`, `tests/`, `docs/`, `package.json`, `jest.config.js`).
- **Corroboración:** el informe Fase 1 (`crm/docs/plans/wab-75-fase-1-auditoria.md` §2.1) declara haber leído las entidades "directamente de `crm/microservices/ai-service/src/domain/entities/` y `src/infrastructure/database/models/`".
- **Efecto previsto:** en D3, ARQ-07 puede revisarse de "No verificado por indisponibilidad del repositorio" a verificación primaria sobre el código local (cancelación/abortación de ejecuciones). Hasta entonces se conserva la clasificación original **Alto, bloquea / No verificado**.

