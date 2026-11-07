# Mapa documental y estado de sincronización — WAB-75 / WAB-78

**Actualizado:** 06/11/2026  
**Estado:** reconciliación parcial incorporada al PR; migración de los documentos fuente completos pendiente. No constituye aprobación integral de WAB-78 ni autorización de implementación.

## 1. Responsabilidad de repositorios y herramientas

- **`sinies7ro/waby-docs`:** repositorio documental privado transversal de WABY: arquitectura, contratos entre servicios, auditorías multicomponente, análisis y planes.
- **`ElemeSoft/crm`:** backend CRM y documentación específica de implementación, reglas y operación de CRM.
- **`ElemeSoft/WABY_frontend`:** frontend y documentación propia de implementación.
- **`ElemeSoft/infrastructure`:** infraestructura/DevOps compartida y documentación propia.
- **Otros repositorios de ElemeSoft:** código y documentación específica de cada componente.
- **Linear:** autoridad operativa para issues, decisiones aprobadas, estados, responsables, dependencias y seguimiento. No sustituye los contratos técnicos versionados.

ElemeSoft es la organización/workspace que agrupa proyectos independientes; no es un repositorio de código único.

## 2. Clasificación de los cuatro documentos recibidos

| Documento | Destino y tratamiento |
|---|---|
| `wab-75-fase-1-auditoria.md` | `audits/wab-77/informe-consolidado-fase-1.md`. Evidencia histórica transversal de CRM, ai-service y frontend. Migrar íntegra, sin reescribir hallazgos. |
| `wab-75-fase-2-diseno.md` | `design/fase-2-diseno-objetivo-y-contratos.md`. Debe ser el documento canónico integral de WAB-78, una vez reconciliado. |
| `WAB-78-P4-diseno-tecnico-normalizado.md` | Integrar el contenido técnico compatible en el documento canónico y ADR-002/ADR-006. No mantenerlo como segundo contrato normativo. Marcar sus apartados superados como antecedente si se conserva. |
| `wab-75-agentes-administrados-por-waby.md` | Propuesta inicial de Fase 0. No es contrato vigente. Archivar solo si se identifica explícitamente como histórico/superado. |

Los archivos fuente fueron adjuntados a la conversación, pero no estaban versionados en ninguno de los repositorios al iniciar esta reconciliación.

## 3. Reconciliación vigente incorporada

Se creó [Registro de reconciliación de vigencia de Fase 2](./design/reconciliacion-vigencia-fase-2-wab-78.md). Consolida:

- ARQ-19: autoridad híbrida por operación.
- ARQ-20: permisos `wrk.agents.*` y `wrk.agents.read`; asignaciones manuales mediante `wrk.workspaces.update`.
- ARQ-21: JWT de servicio RS256 por dirección, scopes mínimos y claims acordados; detalles operativos pendientes.
- DEC-CM-01..08: versión de AgentDefinition, `additionalPrompt`, parámetros técnicos, no routing canal→instancia, cambios explícitos de definición, autoridad CRM de ContactAssignment, suspensión sin fallback y default único.
- P4: bolsa única por Organización, ledger append-only, presupuesto operativo comprometido, asignación por invocación, fencing independiente, CRM como autoridad del saldo, política híbrida y gracia de 30 días.
- Correcciones requeridas en secciones A/C/D/E/F/G, ADR-004, ADR-002/006 y roadmap/checklist.

El registro es una guía de precedencia y correcciones. **No sustituye el documento integral de Fase 2 ni contiene el informe completo WAB-77.**

## 4. Contradicciones principales que debe resolver el documento integral

1. Retirar cupos comerciales de 1/2 instancias, downgrade y gracia de cinco días. No existe límite comercial de cantidad de AgentInstance por Organización o Workspace.
2. Eliminar `channelAssignment`, `priority` y routing persistente canal→instancia. CRM resuelve por ContactAssignment; si no hay asignación activa, asigna la única instancia predeterminada solo si está activa. Si no, queda sin asignación; no hay fallback a otra instancia.
3. Separar provisioning técnico Waby de administración empresarial/ciclo de vida lógico por usuarios autorizados del Workspace.
4. Reemplazar permisos obsoletos `platform.*`, `wrk.agent.use`, `wrk.credits.read` y `bot.configure` por el contrato ARQ-20.
5. Reemplazar `systemPromptOverride` por `additionalPrompt` limitado y subordinado al prompt central.
6. Corregir la responsabilidad de créditos: CRM mantiene saldo y ledger; ai-service ejecuta y reporta consumo.
7. Integrar P4 y la política G.4.6 aprobada; no importar como vigente la antigua sección de la adenda que marcaba facturación pendiente.
8. Reconciliar estados y dependencias con Linear; no inferir cierre de WAB-78.

## 5. Estado operativo contrastado con Linear

| Issue | Estado | Observación |
|---|---|---|
| WAB-76 — Fase 0 | Done | Cerrada |
| WAB-77 — Fase 1 | Done | Auditoría completada |
| WAB-78 — Fase 2 | In Progress | Sigue abierta; requiere revisión integral y aprobación expresa |
| WAB-79 — Fase 3 | Backlog | Bloqueada por WAB-78; no iniciar |
| WAB-80/81/82 | Backlog | No iniciar |

No se modificaron estados ni dependencias.

## 6. Estado GitHub

- [ElemeSoft/crm#10](https://github.com/ElemeSoft/crm/pull/10): cerrado sin fusionar; era el destino incorrecto para documentación transversal. La rama remota no se eliminó.
- [sinies7ro/waby-docs#1](https://github.com/sinies7ro/waby-docs/pull/1): PR correcto, abierto y sin fusionar. Se añadió el registro de reconciliación y se actualizó el README y el cuerpo del PR.
- La comparación actual indica 15 commits adelante y 0 detrás de `master`; GitHub sigue reportando `mergeable=false`. No fusionar; revisar el estado de mergeabilidad antes de solicitar revisión final.

## 7. Trabajo pendiente para completar la sincronización

1. Migrar íntegramente el informe WAB-77 a `audits/wab-77/informe-consolidado-fase-1.md`, preservando su contenido y carácter histórico.
2. Migrar y reconciliar íntegramente las 2027 líneas de Fase 2 a `design/fase-2-diseno-objetivo-y-contratos.md`. Preservar el contenido válido y corregir todas las secciones indicadas en el registro.
3. Integrar la adenda P4 en el documento integral, especialmente contratos, fórmulas, estados, fencing, leases, idempotencia, recuperación, reconciliación y ADR-002/ADR-006.
4. Verificar enlaces, diagramas, matrices, roadmap, checklist y referencias a issues.
5. Actualizar las referencias definitivas de WAB-78 en Linear cuando las rutas canónicas existan.
6. Revisar el PR #1 y fusionarlo únicamente con autorización expresa.

**Límite:** no modificar código, tests, schemas, datos, configuración ni infraestructura. No cerrar WAB-78 ni iniciar WAB-79.
