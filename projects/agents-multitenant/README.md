# Agentes multitenant

Documentación del proyecto de arquitectura y evolución del sistema de agentes multitenant de Waby.

## Alcance

- Auditorías y diagnósticos del sistema actual.
- Arquitectura objetivo y decisiones de diseño.
- Contratos de dominio, runtime, contexto, aislamiento y persistencia.
- Planes de implementación y criterios de aceptación.

## Diseño objetivo

- [Configuración efectiva y capacidades de AgentInstance — propuesta CTX-P3 / CTX-P4](./design/configuracion-efectiva-y-capacidades.md) (pendiente de aprobación).
- [Experiencia de configuración de agentes — principios de producto](./design/experiencia-configuracion-agentes.md) (principios aprobados; diseño visual pendiente).
- [Catálogo de agentes y Skills estándar](./design/catalogo-agentes-y-skills-estandar.md) (dirección de producto aprobada; contenido inicial y contratos pendientes).
- [Plan de administración y evolución del catálogo](./design/plan-administracion-catalogo-waby.md) (propuesta; define cómo Waby autoriza, prueba, publica y actualiza catálogos sin fijar aún el inventario inicial).
- [Integraciones API externas](./design/integraciones-api-externas.md) (ownership aprobado; contratos técnicos pendientes).

## Documentación WAB-78

El conjunto de informes y material de revisión de la Fase 2 está centralizado en [Auditorías WAB-78](./audits/wab-78/README.md). El índice registra los ocho documentos migrados y sus fuentes originales de Dropbox.

## Principios de experiencia del agente

La configuración de agentes debe ser sencilla, guiada y accesible para usuarios sin conocimientos técnicos. Un agente debe poder crearse y utilizarse con instrucciones y Knowledge propios, sin ERP ni integraciones externas. Las capacidades avanzadas, incluidas las integraciones, son opcionales y progresivas. Ver [Experiencia de configuración de agentes](./design/experiencia-configuracion-agentes.md) y [Integraciones API externas](./design/integraciones-api-externas.md).

## Seguimiento

Linear es la fuente de verdad para el estado de las fases, decisiones aprobadas, dependencias y tareas. La documentación enlaza las issues relevantes, sin duplicar su seguimiento.

Referencia principal: [WAB-78 — Fase 2: Diseño objetivo y contratos](https://linear.app/waby/issue/WAB-78/fase-2-diseno-objetivo-y-contratos).

## Criterios de trabajo

- No reabrir auditorías aceptadas sin evidencia nueva que lo justifique.
- Separar hallazgos de auditoría, decisiones de diseño y trabajo pendiente de implementación.
- Mantener trazabilidad desde cada contrato relevante hacia sus decisiones y criterios de aceptación.
- No iniciar fases bloqueadas ni cambiar estados o dependencias de Linear sin autorización.
