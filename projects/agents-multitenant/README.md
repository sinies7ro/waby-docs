# Agentes multitenant

Documentación del proyecto de arquitectura y evolución del sistema de agentes multitenant de Waby.

## Alcance

- Auditorías y diagnósticos del sistema actual.
- Arquitectura objetivo y decisiones de diseño.
- Contratos de dominio, runtime, contexto, aislamiento y persistencia.
- Planes técnicos y criterios de aceptación.

## Diseño objetivo

- [Diseño integral y contratos de Fase 2 — WAB-78](./design/fase-2-diseno-objetivo-y-contratos.md) (**documento canónico de diseño**; integra ARQ-19/20/21, DEC-CM-01..08 y P4).
- [Configuración efectiva y capacidades de AgentInstance — CTX-P3 / CTX-P4](./design/configuracion-efectiva-y-capacidades.md) (contrato conceptual de precedencia, configuración y capabilities).
- [Decisiones operativas de AgentInstance — DEC-CM-01 a DEC-CM-08](./design/decisiones-operativas-agent-instance.md) (registro de decisiones conceptuales aprobadas el 06/11/2026).
- [Reconciliación de vigencia de Fase 2 — WAB-78](./design/reconciliacion-vigencia-fase-2-wab-78.md) (precedencia documental y registro de contradicciones corregidas).
- [Experiencia de configuración de agentes — principios de producto](./design/experiencia-configuracion-agentes.md).
- [Catálogo de agentes y Skills estándar](./design/catalogo-agentes-y-skills-estandar.md).
- [Plan de administración y evolución del catálogo](./design/plan-administracion-catalogo-waby.md).
- [Integraciones API externas por AgentInstance](./design/integraciones-api-externas.md).

## Auditorías y evidencia

- [WAB-77 — Auditoría consolidada de Fase 1](./audits/wab-77/README.md) (evidencia histórica, WAB-77 Done).
- [WAB-78 — Índice documental y material de revisión](./audits/wab-78/README.md) (D1-D5, matriz acumulativa y material técnico de soporte).

## Seguimiento

Linear es la fuente de verdad para estados, decisiones aprobadas, dependencias, responsables y bloqueos. La documentación desarrolla el diseño transversal sin duplicar el seguimiento operativo.

- [WAB-77 — Fase 1](https://linear.app/waby/issue/WAB-77/fase-1-auditoria-del-sistema-actual-sin-modificar-codigo)
- [WAB-78 — Fase 2](https://linear.app/waby/issue/WAB-78/fase-2-diseno-objetivo-y-contratos)

WAB-78 permanece In Progress y bloquea WAB-79. Esta sincronización no cierra WAB-78 ni inicia WAB-79.

## Criterios de trabajo

- No reabrir auditorías aceptadas sin evidencia nueva.
- Separar hallazgos históricos, decisiones aprobadas y trabajo pendiente de implementación.
- Mantener trazabilidad desde cada contrato hacia las decisiones de Linear.
- No modificar repositorios de implementación durante tareas documentales.
