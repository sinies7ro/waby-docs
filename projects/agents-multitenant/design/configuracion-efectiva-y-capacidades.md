# Configuración efectiva y capacidades de AgentInstance — propuesta CTX-P3 / CTX-P4

**Estado:** propuesta de diseño para revisión del usuario; CTX-P3 y CTX-P4 todavía no están aprobados como contratos cerrados.  
**Fecha:** 05/11/2026.  
**Fase:** WAB-78 — diseño objetivo y contratos.  
**No autoriza:** cambios de código, schemas, permisos, datos ni inicio de WAB-79/WAB-80.

## 1. Problema que debe resolver el contrato

D4-A/D4-B identificaron dos contratos bloqueantes:

- **CTX-P3:** composición determinista de la configuración efectiva entre AgentDefinition, AgentInstance y overrides.
- **CTX-P4:** precedencia entre los valores controlados por Waby (plataforma) y los valores configurables por la instancia.

El diseño debe evitar fallbacks implícitos tipo `instanceValue || definitionValue`, especialmente cuando ocultan diferencias de ownership o permiten que una configuración inferior sustituya una política superior.

Esta propuesta también incorpora dos principios de producto aprobados:
- Las integraciones externas son opcionales. Un agente puede funcionar con instrucciones y Knowledge sin ERP ni APIs.
- La creación y configuración deben ser sencillas para usuarios no técnicos. Waby absorbe la complejidad de permisos, composición, validación y runtime.

**Aclaración de nomenclatura:** CTX-P4 no es la decisión comercial P4 de créditos, ya aprobada. Son identificadores diferentes.

## 2. Capas y ownership propuestos

| Capa | Owner | Contenido | ¿La instancia puede modificarla? |
|---|---|---|---|
| Políticas obligatorias de plataforma | Waby | Seguridad, límites de producto, aislamiento, confidencialidad y restricciones globales | No |
| Configuración de runtime de plataforma | Waby | Proveedor LLM, modelo, credenciales, límites máximos y capacidades globales disponibles | No |
| AgentDefinition versionada | Waby | Propósito/base del agente, instrucciones centrales, defaults y conjunto máximo de funcionalidades disponibles | No modifica la definición; solo configura su instancia dentro de los límites permitidos |
| AgentInstance | Workspace (una configuración por agente) | Personalización empresarial, instrucciones propias, reglas de negocio, Skills seleccionadas/configuradas y preferencias permitidas | Sí, mediante usuarios autorizados del Workspace |
| Knowledge | Workspace | Documentos y bases de conocimiento de la empresa; acceso efectivo seleccionado para la instancia | La instancia no cambia la propiedad; un usuario autorizado administra Knowledge del Workspace y lo asigna/autoriza a sus agentes |
| Integraciones | Organización | Conexiones externas, credenciales y catálogo de operaciones | No desde la instancia; el Workspace solo habilita operaciones autorizadas |
| Contexto de ejecución | Runtime | Mensajes, historial, memoria y fragmentos RAG recuperados | No es configuración mutable de la instancia durante una ejecución |

## 3. Precedencia determinista propuesta (CTX-P4)

La precedencia no se trata como una única sustitución de objetos completos. Se define por dominio de configuración:

1. **Políticas obligatorias Waby:** prevalecen siempre. No pueden ser sustituidas por AgentDefinition, AgentInstance, Skills, Rules, Knowledge, contenido recuperado ni respuestas de APIs externas.
2. **Configuración de runtime Waby:** proveedor, modelo, credenciales y límites superiores son controlados exclusivamente por Waby. El cliente no selecciona proveedor/modelo ni aporta API keys del LLM.
3. **AgentDefinition:** establece la configuración base, instrucciones centrales y el máximo conjunto de Skills/capacidades que esa definición puede ofrecer.
4. **AgentInstance:** agrega personalización empresarial y selecciona/configura opciones permitidas por la definición. No puede elevar límites, habilitar capacidades fuera del conjunto autorizado ni reemplazar políticas o instrucciones obligatorias de plataforma.
5. **Contexto de ejecución:** Knowledge, memoria e historial se inyectan como contexto/datos separados; no adquieren prioridad de instrucciones por provenir de documentos o sistemas externos.

No todo valor tiene fallback. Cada campo debe declarar su owner, si admite personalización y cuál es su regla de combinación. Un valor ausente debe producir el default explícito o un error de configuración, no una sustitución genérica silenciosa.

## 4. Composición propuesta (CTX-P3)

### 4.1 Instrucciones y prompt

Componer el prompt efectivo en bloques con responsabilidad separada:

- **Bloque de plataforma:** políticas y restricciones obligatorias, gestionadas por Waby.
- **Bloque base de AgentDefinition:** rol, propósito, comportamiento central y límites funcionales definidos por Waby.
- **Bloque empresarial de AgentInstance:** información e instrucciones propias de la empresa, siempre subordinadas a los dos bloques anteriores.
- **Bloques de contexto:** Rules de negocio, Knowledge, memoria, historial y resultados de operaciones, cada uno delimitado y con su procedencia.

La instancia no debe reemplazar el bloque de plataforma ni borrar el propósito/función base del agente. La personalización empresarial se agrega como instrucciones/contexto propio y puede ajustar tono, lenguaje y procedimientos dentro de las opciones permitidas.

**Cambio conceptual recomendado:** no exponer un campo de UX llamado `systemPromptOverride`. Representar la personalización como instrucciones empresariales adicionales (por ejemplo, `instanceInstructions`), evitando que “override” sugiera que el cliente puede sustituir el system prompt central. El nombre final del campo queda sujeto al modelo de WAB-79.

### 4.2 Rules

Distinguir por origen y autoridad:

- **Políticas de plataforma:** obligatorias, no editables ni desactivables por el Workspace. No deben confundirse con las Rules empresariales.
- **Rules base de AgentDefinition:** defaults centrales de comportamiento. Cada una debe declararse obligatoria o configurable; no puede quedar implícito.
- **Rules empresariales de AgentInstance:** reglas propias de esa empresa/agente. Se agregan al contexto de instrucciones y no pueden contradecir políticas superiores.

Cuando se detecte una contradicción, la regla de menor autoridad no se aplica. El resultado debe ser determinista y quedar observable para diagnóstico; no se deja al LLM resolver qué política prevalece.

### 4.3 Skills

Separar Skill de Capability:

- **Skill:** procedimiento/instrucción reutilizable que orienta cómo realizar una tarea; por sí solo no ejecuta código ni concede permisos.
- **Capability:** operación ejecutable disponible en el runtime (por ejemplo, asignar una conversación o invocar una operación de integración); siempre requiere autorización técnica.

La propuesta es un catálogo Waby de Skills estándar con releases publicadas e inmutables. AgentDefinition referencia una identidad estable de Skill; AgentInstance conserva su selección/habilitación y configuración empresarial permitida, sin exponer al cliente la elección de release. Waby distribuye automáticamente releases compatibles mediante un canal gestionado y rollout controlado. La ejecución resuelve la release vigente y P10 captura el identificador/hash exacto utilizado. Así se preservan trazabilidad y reproducibilidad sin trasladar mantenimiento de versiones a la empresa.

**Catálogo y configuraciones iniciales:** Waby mantiene un catálogo de configuraciones de agente por objetivo empresarial (por ejemplo, Atención, Ventas, Soporte y Legal). Cada AgentDefinition versionada referencia Skills estándar por identidad estable, marcadas como incluidas por defecto o recomendadas. El cliente no crea una AgentDefinition ni un catálogo técnico; selecciona una configuración inicial y personaliza su AgentInstance. El contenido funcional de estos catálogos se desarrolla en [Catálogo de agentes y Skills estándar](./catalogo-agentes-y-skills-estandar.md).

**Punto por confirmar:** la dirección de producto previa indica que cada empresa debe tener procedimientos propios. Se propone admitir procedimientos empresariales asociados a la AgentInstance, separados de las Skills estándar publicadas por Waby. No deben convertirse en código ejecutable ni ampliar capabilities automáticamente. La representación exacta (Skill empresarial versionada o instrucciones/procedimientos de instancia) sigue pendiente de aprobación.

### 4.4 Defaults y cambios de AgentDefinition

Propuesta:
- `defaultSkills` y `defaultRules` definen la selección inicial de la plantilla, no una herencia dinámica de toda la AgentDefinition.
- Al crear una AgentInstance, Waby copia la selección inicial y los parámetros de los defaults permitidos. Para las Skills estándar conserva una identidad estable y una política de actualización gestionada por Waby, no una versión que el cliente deba mantener.
- Las nuevas releases compatibles de una Skill ya habilitada se distribuyen automáticamente mediante rollout controlado; no habilitan nuevas Skills ni amplían permisos.
- Los cambios de la AgentDefinition que alteren la composición, añadan/quiten Skills o cambien capacidades no modifican silenciosamente las instancias existentes; se gestionan mediante una actualización de plantilla explícita y controlada.
- Las políticas obligatorias de plataforma y las revocaciones de seguridad prevalecen sobre cualquier configuración previa. Waby puede bloquear una release vulnerable aunque una ejecución o instancia la tenga referenciada.
- Las acciones de publicación, rollout, promoción, rollback y revocación son administradas y auditadas por Waby.

Esta regla es consistente con P10: cada ejecución conserva un snapshot efectivo inmutable que identifica la release exacta de Skill utilizada. Las ejecuciones nuevas resuelven la release estable vigente; las ejecuciones anteriores no se reescriben. La política de rollout y la administración central se desarrollan en [Plan de administración y evolución del catálogo](./plan-administracion-catalogo-waby.md).

## 5. Capacidades ejecutables y autorización

La exposición de capacidades debe resultar de la intersección de todas las autorizaciones aplicables:

`CapabilitiesEffective = PlatformAllowed ∩ DefinitionAllowed ∩ InstanceEnabled ∩ CurrentAuthorization`

Para operaciones de integración se agrega la condición de ownership y grant:
- Integración y credenciales pertenecen a la Organización.
- La operación debe estar registrada y activa en esa Integración.
- El Workspace debe habilitar esa operación para la AgentInstance concreta.
- Waby vuelve a validar el grant y el estado antes de ejecutar; el LLM no puede llamar directamente a una URL ni construir HTTP arbitrario.

Una Capability no se habilita por el mero hecho de que una Skill la mencione. El prompt no es un mecanismo de autorización.

## 6. Matriz conceptual de autoridad (P3)

La siguiente matriz define responsabilidades de producto; **no aprueba nombres de permisos ni sustituye la matriz técnica de autorización**.

| Operación | Actor/owner propuesto | Alcance |
|---|---|---|
| Crear, versionar y publicar AgentDefinition | Waby | Plataforma |
| Elegir proveedor/modelo, gestionar credenciales LLM y límites globales | Waby | Plataforma |
| Mantener políticas obligatorias y catálogo de capabilities/Skills estándar | Waby | Plataforma |
| Crear/administrar conexiones y credenciales de una integración | Administrador autorizado de la Organización | Organización |
| Registrar/publicar operaciones de una integración | Administrador autorizado de la Organización | Organización |
| Crear una AgentInstance desde una definición disponible | Usuario autorizado del Workspace, con provisioning controlado por Waby | Workspace |
| Personalizar instrucciones y Rules empresariales de la instancia | Usuario autorizado del Workspace | AgentInstance |
| Seleccionar Knowledge disponible para la instancia | Usuario autorizado del Workspace, dentro del Knowledge de su Workspace | Workspace → AgentInstance |
| Habilitar Skills/opciones publicadas permitidas | Usuario autorizado del Workspace | AgentInstance |
| Habilitar operaciones de integración para una AgentInstance | Usuario autorizado del Workspace | Workspace + AgentInstance |
| Ejecutar una Capability | Runtime de Waby con identidad técnica acotada | Organización + Workspace + AgentInstance + Execution |
| Cambiar permisos, roles, miembros o estructura del Workspace | Usuario humano autorizado; nunca la AgentInstance | Workspace/Organización |

### Restricciones derivadas de D2

- No introducir `bot.configure` ni permisos `platform.*` como si ya fueran representables.
- Los permisos delegables al Workspace deben encajar en el modelo existente `org.*` / `wrk.*`, con validación de membresía y ownership; el alcance real debe probarse para evitar escalada mediante `org.*.*`.
- Las operaciones exclusivas de plataforma requieren un actor/identidad de Waby que el sistema actual todavía no representa. No se debe simular ese actor usando un usuario cliente ni un token compartido sin tenant.
- El runtime de una AgentInstance no hereda roles humanos. La identidad técnica debe recibir únicamente el contexto y las operaciones necesarias para esa ejecución.
- La matriz final debe resolver ARQ-03, ARQ-19, ARQ-20 y ARQ-21; ARQ-13/ARQ-14 siguen como bloqueos preexistentes referenciados, no duplicados.

Los nombres y mecanismos concretos de permisos, ownership T3 y service identity deben validarse en WAB-80 sobre la base de este contrato. No se modifica el enum ni se crea un segundo sistema de permisos en WAB-78.

## 7. Experiencia simple de configuración

El usuario no necesita conocer las capas internas descritas en este documento.

Flujo básico propuesto:
1. Elegir una configuración del catálogo por objetivo (por ejemplo, Atención, Ventas, Soporte o Legal).
2. Revisar qué hace el agente y qué Skills estándar incluye o recomienda.
3. Definir nombre y datos básicos.
4. Agregar documentos/conocimiento de su empresa.
5. Personalizar el comportamiento mediante preguntas y opciones claras.
6. Probarlo con ejemplos de conversación.
7. Activarlo.

Las Skills avanzadas, integraciones, operaciones, límites especializados y configuraciones técnicas aparecen opcionalmente cuando el usuario decide ampliar el agente. Sin ERP ni integraciones, el agente debe funcionar normalmente con Knowledge e instrucciones propias.

La interfaz no debe exponer los nombres técnicos de capas, hashes, versiones, providers, esquemas ni permisos. Sí debe explicar de forma simple qué puede hacer el agente y qué datos podrá consultar o modificar.

## 8. Decisiones que esta propuesta necesita confirmar

1. **Precedencia del prompt:** aceptar que las instrucciones propias de la empresa se agregan al bloque base y no reemplazan el núcleo/políticas de Waby.
2. **Defaults de definición:** aceptar que Skills/Rules defaults se copian como referencias versionadas al crear la instancia y no se heredan dinámicamente.
3. **Skills empresariales:** confirmar que las empresas pueden definir procedimientos propios para cada instancia, separados de las Skills estándar que Waby mantiene; dichos procedimientos no conceden por sí solos permisos/capabilities.
4. **Autoridad humana:** confirmar que un usuario autorizado del Workspace puede crear y configurar su AgentInstance desde una definición publicada, mientras Waby conserva las operaciones exclusivas de plataforma y el provisioning técnico. El permiso exacto no se decide aquí.

## 9. Fuera de alcance / no reabrir

- P4 de créditos G.4.6 ya fue aprobada: no se modifica.
- P10 snapshot inmutable por ejecución ya fue aprobado: esta propuesta solo detalla cómo aplicarlo a la configuración.
- No se repiten D1-D4 ni se cambian hallazgos ARQ.
- No se define el esquema persistente final, índices, migraciones ni código: corresponden a las fases posteriores.
- WAB-78 no se cierra ni WAB-79 se inicia con esta propuesta.
