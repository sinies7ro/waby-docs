# Waby — Repositorio documental

Documentación de trabajo de Waby, mantenida por el titular del repositorio y ChatGPT.

Este repositorio es **personal, privado e independiente de la organización ElemeSoft**. Su finalidad es conservar y evolucionar el contexto documental de nuestros proyectos sin mezclarlo con los repositorios de implementación.

## Qué contiene

- Arquitectura y diseños objetivo.
- Contratos, decisiones técnicas desarrolladas y criterios de aceptación.
- Informes de auditoría, diagnósticos y evidencia de análisis.
- Planes técnicos y documentación transversal de proyectos.

## Cómo se relaciona con las demás herramientas

| Herramienta | Fuente de verdad para |
|---|---|
| Este repositorio (`waby-docs`) | Documentación técnica y conocimiento transversal. |
| Linear | Decisiones aprobadas como registro operativo, tareas, responsables, dependencias y estado. |
| Repositorios de código | Código fuente, pruebas y documentación específica que debe evolucionar junto con la implementación. |

No duplicamos el seguimiento de tareas en Markdown. Los documentos pueden enlazar las issues de Linear relacionadas.

## Organización

```text
projects/
  agents-multitenant/
    README.md
    audits/
    architecture/
    contracts/
    plans/
templates/
```

Cada proyecto puede ampliar su estructura cuando exista contenido real que lo justifique. Evitamos crear carpetas y documentos vacíos innecesarios.

## Reglas de mantenimiento

1. Consultar el contenido actual antes de editarlo.
2. Mantener los informes de auditoría aceptados como evidencia histórica. Las correcciones o conclusiones posteriores se documentan mediante adendas o documentos de consolidación, sin reescribir silenciosamente el diagnóstico original.
3. Mantener los documentos vivos en su ruta estable y actualizarlos mediante commits; Git conserva el historial de versiones.
4. Diferenciar claramente propuestas, decisiones pendientes y decisiones aprobadas. Una propuesta no se considera aprobada por aparecer en un documento.
5. Registrar en Linear las decisiones aprobadas y el seguimiento operativo; enlazar desde aquí al registro correspondiente.
6. No duplicar código ni documentación específica del código que deba mantenerse junto a su implementación.
7. No almacenar secretos, credenciales, tokens, datos personales de clientes ni información sensible innecesaria. Que el repositorio sea privado no convierte esos datos en apropiados para Git.

## Idioma y estilo

La documentación se redacta en español, salvo nombres técnicos, identificadores, rutas o términos cuyo uso en inglés sea más preciso. Se priorizan documentos claros, verificables y con referencias a evidencia.

Consulta [DOCUMENTATION-GUIDELINES.md](DOCUMENTATION-GUIDELINES.md) para las convenciones de trabajo.
