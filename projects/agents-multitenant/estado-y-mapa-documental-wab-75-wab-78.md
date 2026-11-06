# Mapa documental y estado de sincronización — WAB-75 / WAB-78

**Actualizado:** 06/11/2026  
**Propósito:** dejar trazabilidad del orden documental entre repositorios y de la reconciliación pendiente de los artefactos locales de WAB-75/WAB-78.  
**Estado:** revisión documental en curso; no constituye aprobación integral de WAB-78 ni autorización de implementación.

## 1. Responsabilidad de cada repositorio y herramienta

- **`sinies7ro/waby-docs`:** repositorio canónico de documentación transversal de WABY: arquitectura, decisiones entre servicios, contratos, auditorías multicomponente, análisis y planes.
- **`ElemeSoft/crm`:** backend CRM y documentación específica de su implementación, reglas y operación. No es el repositorio documental general de WABY.
- **`ElemeSoft/WABY_frontend`:** frontend y documentación propia de su implementación.
- **`ElemeSoft/infrastructure`:** infraestructura compartida y documentación DevOps propia.
- **Otros repositorios de ElemeSoft:** conservan el código y la documentación específica de cada componente.
- **Linear:** registro operativo de issues, decisiones aprobadas, responsables, estados, dependencias y seguimiento. No sustituye los contratos técnicos completos de los repositorios.

## 2. Clasificación de los documentos recibidos desde `crm/docs/plans/`

| Documento | Destino | Tratamiento |
|---|---|---|
| `wab-75-fase-1-auditoria.md` | `projects/agents-multitenant/audits/wab-77/` | Evidencia histórica de WAB-77. Preservar su contenido y fecha; no reescribir hallazgos para reflejar decisiones posteriores. |
| `wab-75-fase-2-diseno.md` | `projects/agents-multitenant/design/` | Debe convertirse en el documento canónico transversal de diseño WAB-78, después de reconciliar sus secciones obsoletas y referencias cruzadas. |
| `WAB-78-P4-diseno-tecnico-normalizado.md` | Integración en el documento canónico de Fase 2 | Adenda de normalización técnica. No debe coexistir como segundo contrato normativo. Conservar como antecedente solo si se necesita trazabilidad de revisión, señalando qué partes fueron absorbidas o superadas. |
| `wab-75-agentes-administrados-por-waby.md` | Archivo histórico, no fuente vigente | Propuesta inicial de Fase 0. Contiene supuestos que cambiaron (por ejemplo, cupos 1/2 y permisos aún pendientes). No publicar como contrato actual sin marcarla expresamente como superada. |

La auditoría WAB-77 es transversal porque reúne evidencia de CRM, ai-service y frontend. No corresponde exclusivamente a la documentación de implementación de CRM.

## 3. Contradicciones que deben corregirse en el documento de Fase 2

La copia recibida de `wab-75-fase-2-diseno.md` contiene decisiones y mecanismos anteriores que ya no representan el acuerdo vigente. No deben trasladarse sin reconciliación:

1. **P1 — cupo comercial:** el documento todavía especifica límites de una/dos instancias por Workspace, downgrade y gracia de cinco días. La dirección aprobada es que no existe límite comercial de cantidad de AgentInstance por Organización ni Workspace; el consumo de IA queda condicionado por la bolsa de créditos de la Organización. Los límites técnicos de protección son independientes.
2. **P2 / DEC-CM-04, 06 y 08 — resolución:** eliminar la asociación persistente canal → instancia (`channelAssignment`), la selección por `priority` y los fallbacks por prioridad. No existe AgentRouting persistente. CRM resuelve por ContactAssignment; ante un nuevo mensaje sin asignación activa intenta la única AgentInstance predeterminada si está activa. Si no hay predeterminada activa, el contacto queda sin asignación. Como máximo una instancia predeterminada por Workspace, con cambio atómico.
3. **ARQ-19 — autoridad híbrida:** el documento atribuye a Waby en exclusiva la creación y el ciclo de vida de AgentInstance. El acuerdo actual permite que usuarios autorizados del Workspace administren sus instancias; Waby conserva definiciones, catálogos, proveedores/modelos/credenciales, políticas, provisioning técnico y purga física bajo política.
4. **ARQ-20 — permisos:** `bot.configure`, `platform.*`, `wrk.agent.use` y `wrk.credits.read` no deben presentarse como contrato vigente. La administración y lectura de instancias usan `wrk.agents.*` y `wrk.agents.read`; asignar/reasignar/liberar contactos manualmente sigue usando `wrk.workspaces.update`. La asignación inicial automática es una operación interna autenticada y auditada de CRM.
5. **DEC-CM-02 — personalización:** sustituir el concepto de `systemPromptOverride` por `additionalPrompt` como instrucción empresarial adicional, limitada y validada, subordinada al prompt central y a las políticas obligatorias.
6. **ARQ-21 — autenticación entre servicios:** el documento canónico debe incorporar el contrato base aprobado: JWT de servicio de corta duración, credenciales diferenciadas por dirección, RS256, token por petición, scope mínimo y claims acordados. Inventario de operaciones, scopes por operación, rotación/revocación, replay, RabbitMQ y pruebas siguen como trabajo pendiente de WAB-78.
7. **P4 — autoridad y facturación:** CRM es autoridad transaccional del saldo; ai-service ejecuta y reporta consumo. La política híbrida G.4.6 y el período de gracia de 30 días para `reconciliation_required` ya fueron aprobados. La adenda P4 recibida todavía presenta la política de facturación como pendiente en §12 y contiene instrucciones de traspaso antiguas; esa parte está superada y no debe reabrirse.
8. **Estados de fase:** la adenda P4 recibida dice que WAB-76 permanece In Progress; Linear confirma WAB-76 Done. Los estados deben leerse desde Linear, no desde instrucciones antiguas incrustadas en documentos.

Las decisiones DEC-CM-01 a DEC-CM-08 están desarrolladas en [Decisiones operativas de AgentInstance](../design/decisiones-operativas-agent-instance.md). ARQ-19/20/21 están desarrolladas en [Configuración efectiva y capacidades](../design/configuracion-efectiva-y-capacidades.md). Esos documentos son parte del PR de sincronización documental en curso; no implican que las decisiones estén implementadas.

## 4. Estado operativo verificado en Linear

| Issue | Estado | Lectura |
|---|---|---|
| WAB-76 — Fase 0 | Done | Cerrada |
| WAB-77 — Fase 1 | Done | Auditoría completada |
| WAB-78 — Fase 2 | In Progress | Sigue abierta; requiere reconciliación y revisión integral |
| WAB-79 — Fase 3 | Backlog | Bloqueada por WAB-78; no iniciar |
| WAB-80 — Fase 4 | Backlog | No iniciar |
| WAB-81 — Fase 5 | Backlog | No iniciar |
| WAB-82 — Fase 6 | Backlog | No iniciar |

Linear mantiene WAB-77 como bloqueante ya completado de WAB-78 y WAB-78 como bloqueante de WAB-79. No se modificaron estados ni dependencias durante esta revisión.

## 5. Estado GitHub y acciones correctivas

- [ElemeSoft/crm#10](https://github.com/ElemeSoft/crm/pull/10), creado para incorporar documentación transversal en el backend CRM, fue **cerrado sin fusionar** el 06/11/2026. Su vínculo también fue retirado de los recursos de WAB-78 en Linear. La rama remota del PR sigue existiendo; no se pudo eliminar con las operaciones disponibles.
- [sinies7ro/waby-docs#1](https://github.com/sinies7ro/waby-docs/pull/1) permanece abierto y sin fusionar. Es el destino correcto para la sincronización documental transversal ya preparada. Debe ampliarse con la auditoría WAB-77 y el documento canónico WAB-78 normalizado antes de solicitar su revisión final.
- No se deben copiar los documentos de Fase 2 a `ElemeSoft/crm` solo porque sus borradores se hayan creado inicialmente bajo `crm/docs/plans/`.

## 6. Secuencia de trabajo para completar la sincronización

1. Incorporar la auditoría WAB-77 en `audits/wab-77/`, preservándola como evidencia histórica.
2. Preparar la versión canónica de Fase 2 en `design/`, conservando el contenido útil y reconciliando las contradicciones enumeradas en §3.
3. Integrar el diseño P4 normalizado en las secciones de presupuesto, invocaciones, fencing, liquidación, estados y ADR-002/ADR-006. No importar como vigentes la §12 pendiente de la adenda ni sus instrucciones de traspaso superadas.
4. Decidir si la propuesta inicial de Fase 0 se archiva como antecedente explícitamente superado o se conserva únicamente en el historial local; no usarla como contrato vigente.
5. Actualizar el índice del proyecto y la descripción/referencias de WAB-78 en Linear para que apunten a las rutas canónicas reales en `waby-docs`.
6. Agregar a Linear un comentario de sincronización con los enlaces definitivos y los pendientes restantes, sin cerrar WAB-78 ni iniciar WAB-79.
7. Revisar el PR de `waby-docs`; fusionarlo solo con autorización expresa.

**Límite:** esta reconciliación es documental. No autoriza cambios de código, schemas, migraciones, datos, infraestructura ni inicio de fases posteriores.
