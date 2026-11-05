# Administración y evolución del catálogo de agentes y Skills de Waby

**Estado:** dirección general de actualización acordada; quedan pendientes los contratos detallados de compatibilidad, notificación, rollout y excepciones de seguridad. No define todavía el catálogo inicial ni el modelo persistente.  
**Fecha:** 05/11/2026.  
**Fase:** WAB-78 — dejar preparado el enfoque para una etapa posterior de diseño e implementación.  
**Principio:** Waby administra y mejora el catálogo; los clientes utilizan agentes actualizados sin gestionar versiones técnicas.

## 1. Decisión de enfoque

No conviene cerrar ahora el listado inicial de categorías, plantillas y Skills. El diseño de experiencia y la arquitectura todavía se están cerrando, y el sistema no está implementado. Crear hoy un inventario detallado tendría alto riesgo de anticipar decisiones que luego cambien.

Sí conviene definir desde ahora **cómo Waby será dueño del catálogo, cómo lo crea, valida, publica, actualiza y retira**. El inventario inicial se elaborará más cerca de la implementación y con criterios de producto/UX y pruebas reales.

Waby mantendrá dos catálogos relacionados:

- Catálogo de configuraciones iniciales de agentes (AgentDefinitions).
- Catálogo reutilizable de Skills estándar.

Los clientes no administran el catálogo central ni necesitan comprender su versionado. Seleccionan una configuración de agente, personalizan su instancia y reciben las mejoras que Waby publique conforme a la política de actualización.

## 2. ¿Por qué conservar versiones?

**Sí: las versiones deben conservarse internamente**, aunque no sean una opción que el usuario final tenga que elegir.

Razones:

1. **Reproducibilidad y auditoría:** permite conocer exactamente qué instrucciones/Skill recibió una ejecución.
2. **P10:** cada ejecución conserva una configuración efectiva inmutable. Para que ese snapshot sea verificable debe identificar una versión exacta de cada Skill estándar utilizada.
3. **Calidad:** permite comparar comportamiento antes/después, reproducir regresiones y ejecutar pruebas sobre una release concreta.
4. **Seguridad:** permite identificar releases afectadas, bloquearlas y desplegar una corrección controlada.
5. **Rollback:** permite volver a una release anterior cuando existe una regresión, siempre que no sea vulnerable ni haya sido retirada por seguridad.

Las versiones publicadas deben ser inmutables. Una corrección no modifica una versión ya publicada: crea una nueva release.

No es necesario imponer Semantic Versioning ni mostrar números de versión al cliente desde el inicio. En la primera etapa puede bastar con un identificador interno único, fecha, estado, huella de contenido, compatibilidad y notas de cambio. La convención de numeración se definirá cuando se diseñe el catálogo real.

## 3. El cliente no administra versiones: política de actualización gestionada por Waby

La política general acordada es que **Waby administra las releases y distribuye automáticamente las mejoras compatibles de las Skills que cada AgentInstance ya tiene habilitadas**. El cliente no elige versiones ni realiza actualizaciones técnicas manuales.

Esta política no cambia la decisión empresarial sobre qué hace el agente: automatiza el mantenimiento de una Skill que el cliente ya eligió utilizar, no la incorporación de nuevas funciones al agente.

### Mejoras normales de una Skill habilitada

- Waby publica una nueva release después de validarla.
- La release pasa por despliegue progresivo y observación.
- Una vez promovida, las instancias que ya tienen habilitada esa misma Skill reciben la mejora automáticamente conforme al rollout.
- La actualización conserva la identidad, el propósito y el alcance funcional de la Skill que el cliente habilitó. No cambia su estado habilitado/deshabilitado, no amplía permisos, no habilita capabilities ni integraciones y no sustituye Rules o instrucciones empresariales.
- Waby puede mostrar un aviso informativo de actualización, con una explicación comprensible de la mejora y su fecha. El aviso informa; no requiere que el cliente elija una versión ni apruebe una actualización compatible.
- La personalización empresarial de la AgentInstance no se sobrescribe.

Si un cambio altera sustancialmente el propósito o alcance de una Skill, no debe presentarse como una simple mejora de su release. Debe tratarse como una Skill nueva o como un cambio de configuración que requiere una decisión explícita del cliente.

### Skills nuevas y cambios de composición

- Publicar una Skill nueva en el catálogo **no la habilita automáticamente en ningún agente existente**.
- Añadir una Skill a las recomendaciones de una AgentDefinition tampoco modifica la selección de Skills de las AgentInstances ya creadas.
- Waby puede informar que existe una Skill nueva y explicar para qué sirve, pero debe quedar como opción disponible/recomendada hasta que un usuario autorizado del Workspace decida habilitarla.
- La configuración del agente debe conservar las elecciones de la empresa. Una Skill que el cliente no desea —por ejemplo, detectar señales de malestar del contacto— no debe ejecutarse ni producir efectos por el hecho de que Waby la haya añadido al catálogo.
- Los cambios de composición de una AgentDefinition se presentan como recomendaciones o propuestas de actualización; no activan Skills ni cambian silenciosamente la configuración de instancias existentes.

### Actualizaciones de seguridad

- La seguridad prevalece sobre la continuidad de una release anterior.
- Waby puede retirar/bloquear una versión afectada y forzar una corrección o deshabilitar temporalmente la Skill/capacidad comprometida.
- Una empresa no puede mantener habilitada una versión declarada vulnerable por preferencia o bloqueo de actualización.
- Las revocaciones urgentes de capabilities/integraciones deben ser efectivas en el punto de autorización, incluso si una ejecución tiene un snapshot previo.
- Una ejecución en curso conserva P10 para su configuración ordinaria, pero no puede iniciar una operación que haya sido revocada por seguridad. Una llamada externa ya completada no puede deshacerse retroactivamente; el diseño posterior debe especificar cómo cancelar o invalidar los pasos restantes.

### Regresiones y rollback

- Waby puede detener el rollout y volver a una release anterior compatible y segura.
- No se puede hacer rollback hacia una versión vulnerable, revocada o incompatible con políticas actuales.
- Cada promoción, pausa, sustitución, bloqueo, aviso y rollback debe quedar auditado.
## 4. Relación entre catálogo, AgentDefinition, AgentInstance y P10

Separar tres conceptos:

1. **Identidad de Skill:** ID estable de la Skill estándar, que representa el procedimiento (por ejemplo, clasificar consultas).
2. **Release de Skill:** versión inmutable del contenido/instrucciones, esquemas y metadatos de esa Skill.
3. **Configuración de Skill en AgentInstance:** indica si la instancia la tiene habilitada, qué opciones empresariales permitidas utiliza y bajo qué política de actualización gestionada la recibe.

La propuesta de resolución es:

- AgentDefinition referencia Skills por ID estable y declara si se incluyen por defecto o se recomiendan.
- AgentInstance conserva sus elecciones empresariales: Skill habilitada/deshabilitada y configuración propia permitida; no necesita administrar números de versión.
- El catálogo resuelve la release aprobada por Waby para esa instancia/canal.
- Al comenzar una ejecución, P10 captura el identificador/hash exacto de la release utilizada.
- Las ejecuciones siguientes resuelven la release estable vigente según el rollout y la política de seguridad.
- Los cambios de catálogo no reescriben snapshots de ejecuciones anteriores.

La transición automática de release no debe confundirse con cambiar la selección empresarial: actualizar el procedimiento de una Skill habilitada no debe habilitar Skills nuevas, ampliar permisos ni conectar integraciones por sí mismo.

Los cambios de una AgentDefinition (composición, selección de Skills, capacidades compatibles, configuración base) se gestionan como actualización del catálogo de plantillas y requieren una política de rollout propia. No deben sobrescribir personalizaciones empresariales. Las políticas obligatorias de Waby y los bloqueos de seguridad siempre prevalecen.

## 5. Proceso futuro para que Waby administre el catálogo

Se propone un flujo interno, operado por personal autorizado de Waby, no por los clientes.

### A. Autoría

- Crear una AgentDefinition o Skill en estado de borrador.
- Registrar objetivo, alcance, casos de uso y exclusiones.
- Definir versión/release, entradas, resultados esperados, instrucciones, compatibilidades y relaciones con otras Skills/capabilities.
- Identificar riesgos, datos utilizados y si la Skill puede influir en decisiones de derivación o acciones externas.

### B. Revisión y validación

Antes de publicar, validar:

- Calidad, claridad y consistencia de instrucciones.
- Que la Skill no pretenda conceder permisos ni ejecutar acciones directamente.
- Que referencias a capabilities existan y estén autorizadas de forma independiente.
- Compatibilidad con AgentDefinitions y configuraciones empresariales admitidas.
- Seguridad, aislamiento, tratamiento de prompt injection y respuestas no confiables.
- Pruebas funcionales con ejemplos esperados, casos límite y entradas adversariales.
- Evaluación de regresión frente a la release estable anterior.
- Impacto de tokens, latencia y costo cuando sea pertinente.

### C. Publicación y rollout

1. Aprobación editorial/técnica por Waby.
2. Publicación inmutable como release candidata.
3. Pruebas internas automatizadas y manuales.
4. Despliegue controlado a un grupo reducido de instancias/entornos.
5. Observación de métricas, errores y resultados de calidad.
6. Promoción al canal estable o pausa/rollback.
7. Registro de release, responsables, cambios y población afectada.

Los porcentajes, ventanas y señales exactas del rollout se definirán cuando existan telemetría y volumen reales; no conviene fijarlos ahora.

### D. Mantenimiento

- Consultar métricas y feedback agregado de uso.
- Corregir defectos creando nuevas releases.
- Marcar releases obsoletas o incompatibles.
- Retirar versiones cuando ya no deban ejecutarse.
- Mantener trazabilidad de las instancias que utilizan cada release.
- Revisar Skills incluidas en cada AgentDefinition para evitar duplicaciones y combinaciones contradictorias.

## 6. Información mínima que Waby debería mantener por release

El futuro modelo del catálogo debe poder registrar, como mínimo:

- ID estable de Skill/AgentDefinition y release ID.
- Estado: draft, candidate, stable, deprecated o revoked (los estados definitivos se diseñarán después).
- Contenido/configuración publicada y hash de integridad.
- Fecha, autor y responsable de aprobación.
- Descripción del cambio y motivo.
- Compatibilidad con runtime, categorías de agente y otras Skills.
- Referencias a capabilities requeridas, sin conceder permisos.
- Resultados de pruebas y evaluación de calidad/seguridad.
- Canal y estado del rollout.
- Release reemplazada, release sucesora y posibilidad de rollback.
- Registro de instancias afectadas cuando sea necesario para seguridad o soporte.

No se almacena información sensible de clientes dentro del catálogo central. Los datos de empresa y las configuraciones de sus AgentInstances permanecen en sus ámbitos de ownership.

## 7. Qué se pospone deliberadamente

No definir todavía:

- Listado inicial definitivo de agentes, categorías ni Skills.
- Cantidad de Skills por plantilla o cuáles van incluidas/recomendadas.
- Pantallas de administración del catálogo.
- Esquema persistente, colecciones, índices y APIs.
- Convención de versionado (SemVer u otra).
- Porcentajes/ventanas del rollout y métricas numéricas de promoción.
- Política detallada para cambios funcionales incompatibles.
- Modelo final de actualización de AgentDefinitions existentes.
- Contrato detallado de Skills propias de cada empresa.

Estos puntos se definirán cuando estén cerrados el diseño de experiencia, el contrato de configuración efectiva y la estrategia de implementación. El objetivo ahora es preservar la decisión arquitectónica de versionado interno y actualización gestionada.

## 8. Relación con decisiones previas

- P10: cada ejecución conserva un snapshot efectivo inmutable e identificable.
- Skills son procedimientos/instrucciones; no son capabilities ejecutables ni permisos.
- AgentDefinition y catálogo estándar son administrados por Waby.
- AgentInstance pertenece al Workspace y conserva la personalización empresarial.
- Las integraciones son opcionales y pertenecen a la Organización; cada Workspace habilita operaciones por AgentInstance.
- Las políticas de seguridad y revocaciones urgentes prevalecen sobre releases antiguas.
- No se cierra WAB-78 ni se inicia WAB-79 con este plan.
