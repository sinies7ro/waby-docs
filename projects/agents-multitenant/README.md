# Agentes multitenant

Documentación del proyecto de arquitectura y evolución del sistema de agentes multitenant de Waby.

## Alcance

- Auditorías y diagnósticos del sistema actual.
- Arquitectura objetivo y decisiones de diseño.
- Contratos de dominio, runtime, contexto, aislamiento y persistencia.
- Planes de implementación y criterios de aceptación.

## Organización

- `audits/`: informes de diagnóstico; evidencia histórica.
- `architecture/`: arquitectura objetivo y vistas del sistema.
- `contracts/`: contratos funcionales y técnicos.
- `plans/`: fases, secuencias y planes de implementación.

## Seguimiento

Linear es la fuente de verdad para el estado de las fases, decisiones aprobadas, dependencias y tareas. La documentación debe enlazar las issues relevantes, no duplicar su seguimiento.

Referencia principal de diseño: [WAB-78 — Fase 2: Diseño objetivo y contratos](https://linear.app/waby/issue/WAB-78/fase-2-diseno-objetivo-y-contratos).

## Criterios de trabajo

- No reabrir auditorías aceptadas sin evidencia nueva que lo justifique.
- Separar hallazgos de auditoría, decisiones de diseño y trabajo pendiente de implementación.
- Mantener trazabilidad desde cada contrato relevante hacia sus decisiones y criterios de aceptación.
- No iniciar fases bloqueadas ni cambiar estados o dependencias de Linear sin autorización.
