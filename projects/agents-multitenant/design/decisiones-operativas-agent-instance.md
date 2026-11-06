# Decisiones operativas de AgentInstance — DEC-CM-01 a DEC-CM-08

**Estado:** decisiones conceptuales aprobadas por el usuario el 06/11/2026.  
**Fase:** WAB-78 — diseño objetivo y contratos.  
**Registro operativo:** [WAB-78 — Fase 2: Diseño objetivo y contratos](https://linear.app/waby/issue/WAB-78/fase-2-diseno-objetivo-y-contratos).  
**Alcance:** decisiones de configuración efectiva, selección, asignación y ciclo de vida de AgentInstance. Este documento desarrolla el contrato transversal; no sustituye los detalles de implementación que pertenecen a los repositorios de código.

## 1. Autoridad y límites

- **Waby** administra AgentDefinitions, versiones publicadas, catálogos centrales, proveedores/modelos/credenciales LLM, parámetros técnicos base, políticas obligatorias, provisioning técnico y purga física bajo política.
- **Workspace** es propietario de sus AgentInstances. Los usuarios humanos autorizados del Workspace administran la configuración empresarial y el ciclo de vida lógico de sus instancias dentro de los límites publicados por Waby.
- **Organización** conserva el ámbito superior de tenancy y administra el plan/entitlement comercial. El rol de administrador de Organización no concede por sí mismo administración de AgentInstances ni de sus integraciones.
- **CRM** es la autoridad de dominio para ContactAssignment y ownership. ai-service puede mantener proyecciones operativas derivadas, pero no una segunda autoridad editable.
- **La AgentInstance y el LLM no son principales administrativos.** Una Capability requiere autorización contextual independiente antes de cada operación o efecto externo.

## 2. Decisiones aprobadas

### DEC-CM-01 — Adopción automática de versiones publicadas

AgentInstance conserva una referencia estable a AgentDefinition, no una versión publicada fijada permanentemente como configuración activa. Cada ejecución nueva resuelve la última versión publicada de esa definición. Una ejecución ya iniciada conserva la versión y el snapshot efectivos con los que comenzó.

### DEC-CM-02 — Instrucciones adicionales por instancia

`additionalPrompt` es configuración propia de AgentInstance, sujeta a límites de tamaño y validación. Se incorpora al snapshot de cada ejecución como personalización empresarial adicional.

No reemplaza el prompt central ni puede neutralizar políticas obligatorias, límites de seguridad o instrucciones de plataforma. El contenido recuperado mediante RAG continúa siendo dato no confiable y no adquiere autoridad por combinarse con estas instrucciones.

### DEC-CM-03 — Parámetros técnicos administrados por Waby

Proveedor, modelo, credenciales LLM y parámetros técnicos base son administrados exclusivamente por Waby. La allowlist de parámetros configurables por instancia queda vacía por defecto.

No se expone al Workspace un objeto libre de parámetros del proveedor ni se permite que la instancia elija o reemplace credenciales/proveedor/modelo de plataforma.

### DEC-CM-04 — Resolución sin routing persistente canal → instancia

No se crea una entidad persistente AgentRouting ni una asociación permanente entre un canal y una AgentInstance.

El canal se conserva como contexto del mensaje entrante y de Execution. La instancia se resuelve mediante el ContactAssignment vigente y, cuando no existe asignación activa, mediante la AgentInstance predeterminada del Workspace conforme a DEC-CM-06.

### DEC-CM-05 — Cambio explícito de AgentDefinition

Cambiar la AgentDefinition asociada a una instancia es una operación explícita, autorizada y auditada. Si la nueva definición resulta incompatible con la configuración actual, la instancia queda suspendida hasta que un usuario autorizado revise y corrija la configuración.

Los cambios relevantes de AgentDefinition deben notificarse a los Workspaces afectados. No se cambia silenciosamente la definición de una instancia ni se omiten las validaciones de compatibilidad.

### DEC-CM-06 — Fuente de verdad y precedencia de asignación

CRM es la única fuente de verdad de ContactAssignment. La resolución sigue esta precedencia:

1. **Asignación humana vigente:** el agente no interviene.
2. **Asignación vigente a una AgentInstance activa:** esa instancia conserva la atención.
3. **Sin asignación activa:** ante un nuevo mensaje entrante, CRM intenta asignar la AgentInstance predeterminada del Workspace si está activa.
4. **Sin predeterminada activa o sin predeterminada:** el contacto permanece sin asignación para atención humana.

No existe selección alternativa por prioridad ni fallback automático a otra instancia. La evaluación inicial se dispara por un nuevo mensaje entrante, no por la mera desasignación.

### DEC-CM-07 — Suspensión de una instancia con contactos asignados

La suspensión conserva los ContactAssignment activos. Mientras la instancia está suspendida no recibe nuevas asignaciones ni ejecuta atención automática; los contactos asignados quedan temporalmente sin atención automática.

Al reactivarse esa misma instancia puede reanudar la atención de sus contactos asignados. Un usuario autorizado también puede reasignar explícitamente cada contacto.

No se liberan asignaciones ni se aplica fallback o reasignación automática a otra instancia por la suspensión. La conversación y su historial se conservan.

### DEC-CM-08 — Unicidad y cambio de instancia predeterminada

Como máximo una AgentInstance puede tener `isDefault=true` por Workspace. La garantía debe mantenerse también ante concurrencia y el cambio de predeterminada debe ser atómico.

Cambiar la predeterminada no altera las asignaciones existentes. La nueva predeterminada solo se utiliza para contactos que no tengan una asignación activa cuando se evalúe un nuevo mensaje entrante.

## 3. Contrato de autorización de asignaciones

La administración de una AgentInstance y la asignación de contactos son ámbitos distintos:

- `wrk.agents.*` administra la AgentInstance: creación desde definiciones publicadas, consulta, configuración y ciclo de vida lógico.
- `wrk.agents.read` permite consultar/ver instancias, sin facultades administrativas.
- Las operaciones manuales de asignar, reasignar o liberar contactos utilizan el permiso CRM existente `wrk.workspaces.update`.
- La asignación inicial automática es una operación interna de CRM, autenticada, validada y auditada.
- La AgentInstance no actúa como usuario humano ni recibe permisos humanos de Workspace.
- Toda asignación a una instancia valida que Contacto y AgentInstance pertenezcan al mismo Workspace.

La matriz de permisos y sus asignaciones por rol están desarrolladas en la matriz acumulativa de WAB-78 y en el contrato ARQ-20. La implementación y las pruebas corresponden a WAB-80.

## 4. CRM y proyección operativa de ai-service

CRM conserva la autoridad única de asignación efectiva. ai-service puede mantener una proyección operativa derivada, sincronizada desde CRM mediante Outbox, para que el runtime consulte el estado necesario; esa proyección no es una segunda fuente editable de verdad.

Las transiciones y su propagación deben ser idempotentes y seguras ante concurrencia. La validación de ownership debe preservar la cadena AgentInstance → Workspace → Organización. Los detalles de persistencia, eventos, vigencia y cancelación se desarrollan en los contratos técnicos de WAB-78 y se implementan en las fases correspondientes.

## 5. Estado y trabajo pendiente

Las decisiones DEC-CM-01 a DEC-CM-08 están aprobadas. Su aprobación no implica que estén implementadas ni que el sistema actual ya cumpla estos contratos.

Permanecen dentro del diseño de WAB-78 los contratos técnicos de ejecución CRM → RabbitMQ → ai-service, identidad y autenticación de servicios, vigencia/cancelación, reconciliación de la matriz de operaciones y revisión integral de consistencia.

El modelo persistente completo de AgentInstance y sus índices corresponden a WAB-79. Provisioning, autorización y pruebas de permisos corresponden a WAB-80. Runtime y composición de contexto corresponden a WAB-81. Knowledge compartido y Skills/Rules por agente corresponden a WAB-82.

**Control de fase:** WAB-78 permanece abierta. Este documento no autoriza iniciar WAB-79, WAB-80, WAB-81 o WAB-82 ni realizar cambios de código, schemas, datos o configuración.
