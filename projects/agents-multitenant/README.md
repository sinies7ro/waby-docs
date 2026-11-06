# Agentes multitenant

Documentación del proyecto de arquitectura y evolución del sistema de agentes multitenant de Waby.

## Alcance

- Auditorías y diagnósticos del sistema actual.
- Arquitectura objetivo y decisiones de diseño.
- Contratos de dominio, runtime, contexto, aislamiento y persistencia.
- Planes de implementación y criterios de aceptación.

## Diseño objetivo

- [Configuración efectiva y capacidades de AgentInstance — CTX-P3 / CTX-P4](./design/configuracion-efectiva-y-capacidades.md) (CTX-P3/P4 aprobados; ARQ-19/20 resueltos documentalmente; ARQ-21 con Opción C/RS256 seleccionada y detalles de implementación pendientes).
- [Decisiones operativas de AgentInstance — DEC-CM-01 a DEC-CM-08](./design/decisiones-operativas-agent-instance.md) (decisiones conceptuales aprobadas el 06/11/2026; implementación diferida a sus fases).
- [Reconciliación de vigencia de Fase 2 — WAB-78](./design/reconciliacion-vigencia-fase-2-wab-78.md) (registro de precedencia, decisiones vigentes y correcciones obligatorias; no sustituye aún el documento integral).
- [Experiencia de configuración de agentes — principios de producto](./design/experiencia-configuracion-agentes.md) (principios aprobados; diseño visual pendiente).
- [Catálogo de agentes y Skills estándar](./design/catalogo-agentes-y-skills-estandar.md) (dirección de producto aprobada; contenido inicial y contratos pendientes).
- [Plan de administración y evolución del catálogo](./design/plan-administracion-catalogo-waby.md) (propuesta; define cómo Waby autoriza, prueba, publica y actualiza catálogos sin fijar aún el inventario inicial).
- [Integraciones API externas por AgentInstance](./design/integraciones-api-externas.md) (conexión, credenciales y operaciones configuradas por usuarios autorizados del Workspace; plan de agentes IA administrado por Organización; contratos técnicos pendientes).
- [Mapa documental y estado de sincronización WAB-75/WAB-78](./estado-y-mapa-documental-wab-75-wab-78.md) (responsabilidad de repositorios, clasificación de documentos recibidos, contradicciones y secuencia de reconciliación).

## Documentación WAB-78

El conjunto de informes y material de revisión de la Fase 2 está centralizado en [Auditorías WAB-78](./audits/wab-78/README.md). La auditoría consolidada de Fase 1 (WAB-77) y el documento integral normalizado de Fase 2 todavía deben migrarse íntegramente desde los adjuntos locales originales; el registro de reconciliación no sustituye esos artefactos.

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
