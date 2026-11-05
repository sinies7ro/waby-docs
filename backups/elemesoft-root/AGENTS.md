# AGENTS.md - ElemenSoft

Instrucciones para agentes de codificación en este repositorio multi-proyecto.

---

## Repository Overview

ElemenSoft es un workspace que contiene proyectos independientes:

| Proyecto | Descripción |
|----------|-------------|
| `crm/` | Backend CRM principal |
| `WABY_frontend/` | Frontend plataforma WhatsApp Business |
| `wabyland/` | Landing page |
| `infrastructure/` | Infraestructura y DevOps |

---

## Project Boundaries

### Reglas de aislamiento

1. **Identificar proyecto afectado** antes de cualquier modificación.
2. **No aplicar convenciones** de un proyecto a otro.
3. **No modificar archivos** fuera del proyecto afectado salvo que la tarea requiera cambios cross-project.
4. **No realizar refactorizaciones** no relacionadas con el scope solicitado.

### Git worktrees

Algunos directorios pueden ser Git worktrees del mismo repositorio, no proyectos independientes.

**Regla:** Si un directorio es worktree de otro proyecto, documentarlo como parte del proyecto padre.

```
CRM
└── crm/       ← worktree principal
```

No modificar la estructura de worktrees sin confirmación explícita.

### Dependencias entre proyectos

- `WABY_frontend/` se conecta a `crm/` vía GraphQL
- `wabyland/` es independiente
- `infrastructure/` es compartido por servicios backend

---

## Instruction Hierarchy

Los agentes DEBEN seguir este orden de precedencia:

```
1. Este AGENTS.md raíz          → Reglas globales del repositorio
2. AGENTS.md del proyecto        → Reglas específicas del proyecto
3. Skills del proyecto           → Conocimiento técnico especializado
4. Documentación existente       → Arquitectura, ADRs, especificaciones
5. Código y tests existentes     → Implementación actual y evidencia comportamental
```

**Regla clave:** Instrucciones más específicas prevalecen sobre las generales, salvo conflicto con reglas de mayor nivel.

---

## Agent Workflow

### Antes de modificar código

1. Entender el cambio solicitado y su scope.
2. Identificar proyecto(s) afectado(s).
3. Leer el `AGENTS.md` del proyecto afectado.
4. Identificar y leer skills relevantes.
5. Inspeccionar implementación existente y tests relacionados.
6. Identificar dependencias, consumidores y efectos secundarios.

### Durante la implementación

7. Preferir el cambio más pequeño que resuelva correctamente el problema.
8. Implementar solo lo pedido (no rediseñar, no agregar features).
9. Seguir patrones existentes en el proyecto.

### Después de implementar

10. Ejecutar quality checks del proyecto afectado.
11. Revisar el diff final.
12. Reportar: qué cambió, qué se testearon, riesgos pendientes.

### Prohibición absoluta

**No modificar código basándose únicamente en suposiciones** cuando la implementación existente puede ser inspeccionada.

---

## Skill Usage

### Reglas globales de skills

1. **Cada proyecto puede tener su propio sistema de skills.**
   - No asumir que todos los proyectos tienen skills.
   - No asumir que las skills de un proyecto aplican a otro.

2. **Las skills son específicas de proyecto.**
   - Las skills de CRM no aplican a Frontend ni a otros proyectos.
   - Las skills de un proyecto no deben cargarse para otro proyecto.

3. **Consultar el AGENTS.md del proyecto** para saber qué skills están disponibles y cómo cargarlas.

### Fuente canónica vs tooling

Algunos proyectos utilizan un sistema de sincronización de skills:

```
.engineering/     → Fuente canónica del conocimiento
.opencode/skills/ → Representación generada por tooling (no editar directamente)
```

**Regla:** Si existe un directorio `.engineering/` como fuente canónica, editar ahí y ejecutar el comando de sincronización del proyecto. No editar directamente `.opencode/skills/`.

---

## Scope Control

### Reglas

- Implementar SOLO lo pedido.
- NO rediseñar arquitectura no solicitada.
- NO agregar features adicionales.
- NO optimizar sin pedido explícito.
- Si falta contexto → DETENERSE y preguntar.

### Cambios cross-project

Solo permitidos cuando:
- La tarea lo requiere explícitamente.
- Se identifican TODOS los proyectos afectados antes de comenzar.
- Se respetan las convenciones de cada proyecto.

---

## Evidence-Based Development

### Antes de asumir

1. **Verificar** que la tecnología/patrón mencionado realmente existe en el proyecto.
2. **Inspeccionar** el código existente antes de implementar.
3. **Consultar** documentación y ADRs del proyecto.
4. **No inventar** arquitectura para resolver contradicciones.

### Verificaciones obligatorias

- ¿Este archivo/función/clase realmente existe?
- ¿Esta dependencia está en package.json?
- ¿Este patrón se usa actualmente en el proyecto?
- ¿Esta configuración es la correcta?

---

## Architectural Safety

### Reglas globales

- No introducir dependencias sin verificar que el proyecto las use.
- No mezclar frameworks/patrones diferentes sin justificación documentada.
- No asumir tecnologías que no estén verificadas en configuración o código.
- Respetar la arquitectura existente del proyecto.

---

## Security

### Reglas absolutas

1. **Nunca commitear:**
   - Archivos `.env`
   - Secretos, API keys, tokens
   - Credenciales de cualquier tipo

2. **Headers de autenticación:**
   - `x-org-id`
   - `x-wrk-id`
   - `Authorization: Bearer <token>`

3. **Variables de entorno:**
   - Nunca hardcodear valores sensibles.
   - Usar `.env.example` como plantilla.

---

## Testing

### Política global

Cada proyecto define sus propios comandos de testing. Consultar el AGENTS.md del proyecto afectado.

### Verificación obligatoria

Antes de confirmar un cambio:
1. Ejecutar tests unitarios del proyecto.
2. Verificar que no hay errores de TypeScript.
3. Verificar que lint pasa.

---

## Quality Gates

### Antes de commit

1. TypeScript sin errores.
2. Linting sin errores.
3. Tests pasando.
4. Build exitoso (si aplica).

---

## Git Workflow

### Convenciones

- **Branches:** `feature/`, `bugfix/`, `hotfix/`
- **Commits:** Conventional Commits (`feat:`, `fix:`, `docs:`, etc.)
- **Reviews:** Requeridos antes de merge
- **PRs:** Mantener enfocados y de tamaño razonable

### Reglas

- No commitear secrets.
- No hacer force-push sin autorización.
- No saltar hooks.
- Describir cambios claramente.

---

## Completion Report

Al finalizar una tarea, el agente debe reportar:

### Información mínima

1. **Qué cambió:** Archivos modificados y naturaleza del cambio.
2. **Qué se verificó:** Tests ejecutados, checks pasados.
3. **Scope:** Si se mantuvo dentro del scope solicitado.
4. **Riesgos:** Dependencias pendientes, breaking changes potenciales.
5. **Scope no cubierto:** Elementos identificados que requieren atención futura.

### Formato sugerido

```
## Cambios realizados
- [archivos modificados]

## Verificación
- [tests ejecutados]
- [checks pasados]

## Scope
- [dentro/fuera de scope]

## Riesgos
- [identificados]

## Pendiente
- [requiere revisión humana]
```
## GitHub y GitHub Actions

Cuando una tarea requiera consultar el estado de GitHub Actions, commits, workflows, PRs o checks remotos:

1. **Preferir GitHub CLI (`gh`)** si está instalada y disponible en el entorno de ejecución.
2. Antes de diagnosticar problemas de acceso, comprobar la autenticación con `gh auth status`.
3. Verificar el repositorio remoto correcto antes de consultar sus ejecuciones. No asumir que el nombre del directorio local coincide con `OWNER/REPO`.
4. Para consultar ejecuciones de CI, utilizar:
   - `gh run list --repo OWNER/REPO --branch master --limit 10`
   - `gh run list --repo OWNER/REPO --commit FULL_SHA --limit 10`
   - `gh run view RUN_ID --repo OWNER/REPO`
   - `gh run view RUN_ID --repo OWNER/REPO --log-failed`
5. Cuando se necesite validar un commit concreto, comprobar el SHA completo de la ejecución. No asumir que un CI verde de un commit posterior valida todos los cambios anteriores, especialmente si solo modificó documentación.
6. Si una consulta falla, investigar primero la autenticación, los permisos, el repositorio y el contexto de ejecución. Un HTTP 404 no demuestra por sí solo que el workflow o recurso no exista.
7. Si `gh` no está disponible o no puede autenticarse, informar el error concreto y solicitar intervención únicamente cuando sea necesaria. No afirmar que el CI remoto está inaccesible sin verificar las alternativas disponibles.
8. No imprimir, compartir ni registrar tokens o credenciales. No utilizar `gh auth status --show-token` ni otros comandos que expongan secretos.
9. Las consultas deben ser de solo lectura, salvo autorización explícita para ejecutar, cancelar o volver a ejecutar workflows, modificar GitHub o realizar otras acciones remotas.

En el informe final, distinguir claramente entre CI verificado, CI fallido y CI pendiente de verificación.

---

## Waby — Continuidad documental y coordinación del trabajo

Estas reglas se aplican cuando la tarea esté relacionada con Waby, independientemente de cuál de sus repositorios de código se esté utilizando. No modifican los límites ni las convenciones técnicas particulares de cada proyecto.

### Fuentes de verdad

| Fuente | Responsabilidad |
|---|---|
| `https://github.com/sinies7ro/waby-docs` | Documentación transversal: arquitectura, contratos, auditorías, análisis y planes. Es un repositorio privado, personal e independiente de ElemeSoft. |
| Linear — workspace Waby | Registro operativo: decisiones aprobadas, tareas, responsables, estados, dependencias y bloqueos. |
| Repositorios de código correspondientes | Implementación real, pruebas y documentación específica que debe evolucionar junto con el código. |

Dropbox ya no es el repositorio documental de trabajo para WAB-78. Los originales que permanezcan allí son referencias históricas de la migración, no una segunda fuente editable.

### Inicio y continuidad de una tarea

Antes de comenzar trabajo sustantivo relacionado con Waby:

1. Identificar la issue o fase correspondiente en Linear.
2. Consultar su descripción, comentarios, estado, dependencias y decisiones registradas que afecten a la tarea.
3. Consultar los documentos relevantes en `waby-docs`.
4. Inspeccionar la rama, commit y código actual cuando la tarea dependa de la implementación.
5. Continuar desde el estado verificado. No solicitar archivos que ya estén disponibles en las fuentes compartidas ni repetir auditorías aceptadas sin evidencia nueva que lo justifique.

Si todavía no existe una issue, no inventar que existe ni crearla salvo que la tarea lo solicite o el usuario lo autorice. Si una fuente no está accesible, informar exactamente qué no se pudo consultar y no asumir su contenido.

### Decisiones y trazabilidad

- Diferenciar siempre propuestas, decisiones pendientes y decisiones aprobadas.
- Registrar las decisiones aprobadas en Linear, en la issue correspondiente, con su contexto y consecuencias relevantes.
- Desarrollar el detalle técnico en `waby-docs` y enlazarlo desde Linear. No duplicar el tablero de tareas en Markdown.
- Mantener referencias entre issues, documentos, repositorios, ramas y commits cuando corresponda.
- No cambiar estados, cerrar issues, alterar dependencias ni iniciar fases bloqueadas sin autorización explícita.
- No presentar una recomendación o propuesta del agente como decisión aprobada por el usuario.

### Mantenimiento documental

- Los documentos vivos (arquitectura, contratos, especificaciones y planes vigentes) se actualizan en su ruta estable. El historial Git conserva las versiones anteriores; no crear copias `v2`, `v3` por cada edición.
- Los informes de auditoría aceptados son evidencia histórica. No reescribirlos para incorporar conclusiones posteriores; utilizar adendas o documentos de consolidación con referencias explícitas al informe original.
- Antes de actualizar un documento, leer su versión actual y comprobar que no haya cambios concurrentes.
- Después de escribir, volver a leer el archivo desde GitHub y verificar el contenido o el diff.
- Conservar en `waby-docs` la documentación transversal. Mantener en cada repositorio de código los documentos estrechamente ligados a su implementación.
- No guardar secretos, tokens, credenciales, datos personales de clientes ni información sensible innecesaria, aunque el repositorio sea privado.

### Secuencia de trabajo preferida

Para tareas de diseño e implementación, priorizar el siguiente ciclo sin convertirlo en una ceremonia rígida:

1. Análisis suficiente del problema y del estado actual.
2. Identificación puntual de dudas y definición de las decisiones necesarias.
3. Diseño/decisiones y actualización de Linear y documentación cuando corresponda.
4. Implementación acotada al alcance aprobado.
5. Pruebas y verificaciones pertinentes.
6. Registro de resultados, cambios, riesgos y siguiente paso.

Las auditorías sirven para ajustar el diseño al código existente; no deben convertirse en ciclos de reanálisis indefinidos. Separar los problemas pendientes, priorizarlos y resolverlos con propuestas concretas. No sacrificar aislamiento multitenant, seguridad, consistencia ni escalabilidad para acelerar el trabajo.

### Herramientas y permisos

- La autenticación y los permisos de OpenCode son independientes de los de ChatGPT. No asumir que una conexión disponible en ChatGPT también está disponible en OpenCode.
- Antes de operar con GitHub o Linear, comprobar que la herramienta correspondiente esté disponible y autenticada en el entorno actual.
- Preferir las integraciones o CLI disponibles para consultar y actualizar las fuentes; no inventar comandos, resultados ni accesos.
- No afirmar que una consulta, escritura, commit o actualización se completó hasta verificar el resultado en la fuente correspondiente.
- Si OpenCode no tiene acceso a una fuente necesaria, informar el bloqueo concreto y continuar solo con las partes que puedan realizarse con evidencia disponible.

