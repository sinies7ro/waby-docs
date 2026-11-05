# AGENTS.md — waby-docs

Instrucciones para agentes que trabajen directamente dentro de este repositorio documental.

## Propósito y alcance

- Este es el repositorio documental personal y privado de Waby: `sinies7ro/waby-docs`.
- Es independiente de la organización ElemeSoft y no contiene el código fuente de producción.
- Consultar `README.md` y `DOCUMENTATION-GUIDELINES.md` antes de modificar la organización o las convenciones documentales.
- La copia de respaldo del AGENTS raíz del workspace ElemeSoft se conserva en `backups/elemesoft-root/AGENTS.md`. Ese archivo no reemplaza estas instrucciones específicas del repositorio.

## Fuentes de verdad

- Este repositorio: documentación transversal, arquitectura, contratos, auditorías, análisis y planes.
- Linear — workspace Waby: decisiones aprobadas, tareas, responsables, estados, dependencias y bloqueos.
- Repositorios de código: implementación real, pruebas y documentación ligada estrechamente al código.

## Continuidad de trabajo

Antes de una tarea sustantiva:

1. Consultar en Linear la issue correspondiente, sus comentarios, estado, dependencias y decisiones relevantes.
2. Leer los documentos relacionados en este repositorio.
3. Consultar el código actual cuando la tarea dependa de la implementación.
4. Continuar desde el estado verificado; no repetir auditorías aceptadas sin evidencia nueva ni pedir archivos que ya estén disponibles.

Si una fuente no está disponible, informar el bloqueo concreto. No inventar estados, decisiones, acceso ni resultados.

## Decisiones y registro

- Diferenciar propuestas, decisiones pendientes y decisiones aprobadas.
- Registrar las decisiones aprobadas en Linear y desarrollar su detalle técnico aquí.
- Mantener enlaces entre documentos, issues y commits relevantes.
- No cambiar estados, cerrar issues, modificar dependencias ni iniciar fases bloqueadas sin autorización explícita.

## Política documental

- Los documentos vivos se mantienen en rutas estables; Git conserva su historial.
- Los informes de auditoría aceptados son evidencia histórica. No reescribirlos para incluir decisiones o hallazgos posteriores.
- Utilizar adendas o documentos de consolidación con referencias explícitas cuando sea necesario corregir o ampliar evidencia.
- Leer el contenido actual antes de editar; después de escribir, recuperar el archivo desde GitHub y verificar el resultado.
- No duplicar en Markdown el seguimiento operativo de Linear.
- No almacenar secretos, tokens, credenciales ni datos personales de clientes.

## Herramientas

- Verificar que GitHub y Linear estén disponibles y autenticados en el entorno actual.
- El acceso de OpenCode es independiente del acceso de ChatGPT.
- No afirmar que una operación se completó hasta verificarla en la fuente correspondiente.

## Secuencia de trabajo

Análisis suficiente → decisiones puntuales → diseño/documentación → implementación acotada → pruebas/verificación → registro de resultados y pendientes.

Evitar auditorías repetitivas y ciclos de reanálisis. Priorizar soluciones concretas sin sacrificar aislamiento multitenant, seguridad, consistencia ni escalabilidad.

## Seguridad

- No incluir secretos ni datos sensibles en el repositorio.
- No inferir que la visibilidad privada elimina la necesidad de proteger la información.
