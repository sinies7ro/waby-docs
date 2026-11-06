# WAB-78 — Índice documental y registro de migración

**Destino:** `projects/agents-multitenant/audits/wab-78/`.
**Origen histórico:** `/WABY/docs/revision-wab-78/` en Dropbox.
**Issue:** [WAB-78 — Fase 2: Diseño objetivo y contratos](https://linear.app/waby/issue/WAB-78/fase-2-diseno-objetivo-y-contratos).

## Alcance y preservación

Se conservan los ocho informes Markdown recuperados originalmente desde Dropbox, sin reescritura editorial. Los originales de Dropbox no se modificaron ni eliminaron. GitHub es la ubicación documental versionada de trabajo.

Los documentos incorporados posteriormente se identifican por separado. Las auditorías aceptadas se preservan como evidencia histórica; las decisiones posteriores se documentan en el diseño vivo de WAB-78 y en Linear.

## Inventario de la migración original

| Documento | Clasificación |
|---|---|
| [00-linea-base.md](./00-linea-base.md) | Línea base, fuentes y convenciones de revisión |
| [informe-bloque-1.md](./informe-bloque-1.md) | Diagnóstico D1 — dominio y ownership |
| [informe-bloque-2.md](./informe-bloque-2.md) | Diagnóstico D2 — provisioning y permisos |
| [informe-bloque-3.md](./informe-bloque-3.md) | Diagnóstico D3 — routing y asignaciones |
| [informe-d4-a-runtime.md](./informe-d4-a-runtime.md) | Diagnóstico D4-A — runtime |
| [informe-d4-b-contexto.md](./informe-d4-b-contexto.md) | Diagnóstico D4-B — contexto |
| [matriz-acumulativa.md](./matriz-acumulativa.md) | Matriz única de hallazgos ARQ |
| [analisis-previo-d2-d4.md](./analisis-previo-d2-d4.md) | Material preliminar de análisis D2-D4 |

## Material incorporado posteriormente

| Documento | Función y vigencia |
|---|---|
| [informe-bloque-5b.md](./informe-bloque-5b.md) | Evidencia histórica D5-B. No es contrato normativo; sus propuestas deben leerse junto a decisiones posteriores. |
| [reconciliacion-d5-a-d5-b.md](./reconciliacion-d5-a-d5-b.md) | Reconciliación D5-A/D5-B y referencias a D5-C. D5-A y D5-C están registrados como comentarios de Linear; no existe un informe 5a separado. |
| [complemento-tecnico-asignaciones-evassigned-cancelacion.md](./complemento-tecnico-asignaciones-evassigned-cancelacion.md) | Material técnico de apoyo sobre ContactAssignment, EvAssigned, Outbox y cancelación. El diseño vigente está integrado en el documento canónico de Fase 2. |

## Documento canónico de diseño

El contrato vigente y la integración del diseño P4 se encuentran en [Diseño integral y contratos de Fase 2](../../design/fase-2-diseno-objetivo-y-contratos.md). La adenda P4 ya no se mantiene como fuente normativa separada.

La [reconciliación de vigencia](../../design/reconciliacion-vigencia-fase-2-wab-78.md) registra precedencia y decisiones aprobadas. Los comentarios D5-A, D5-B y D5-C se conservan en la issue WAB-78 de Linear.

## Estado y límites

- WAB-77: Done.
- WAB-78: In Progress; no se cierra hasta revisión y autorización expresa del diseño completo.
- WAB-79: Backlog, bloqueada por WAB-78; no iniciar.
- Esta sincronización no implica aceptación global de WAB-78 ni autorización de implementación.
