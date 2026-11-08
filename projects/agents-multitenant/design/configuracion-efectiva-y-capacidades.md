> **Estado reconciliado — 07/11/2026:** ARQ-14 está resuelto en D.1.9/B del diseño canónico. ARQ-19/20/21 mantienen sus decisiones aprobadas; el endurecimiento operativo de ARQ-21 se difiere a WAB-80/WAB-81. Este documento conserva su alcance de configuración efectiva/capabilities y no sustituye la matriz de persistencia ni el contrato de mensajería D.6.

---

# Configuración efectiva y capacidades de AgentInstance — propuesta CTX-P3 / CTX-P4

**Estado:** CTX-P3/CTX-P4 aprobados como dirección y contrato conceptual. ARQ-19 (modelo híbrido por operación) y ARQ-20 (contrato `wrk.agents.*` / `wrk.agents.read`) están resueltos documentalmente. ARQ-21 tiene aprobada la Opción C y el contrato base de JWT de servicio RS256; el inventario exhaustivo de operaciones, scopes, rotación, verificación integral de flujos y evidencia de despliegue siguen pendientes.  
**Fecha:** 06/11/2026.  
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
| Plan y entitlement de agentes IA | Organización | Plan contratado y habilitación comercial de funcionalidades de agentes IA | Se modifica mediante la administración del plan de la Organización; no configura las AgentInstances |
| AgentDefinition versionada | Waby | Propósito/base del agente, instrucciones centrales, defaults y conjunto máximo de funcionalidades disponibles | No modifica la definición; solo configura su instancia dentro de los límites permitidos |
| AgentInstance | Workspace (una configuración por agente) | Personalización empresarial, instrucciones propias, reglas de negocio, Skills seleccionadas/configuradas y preferencias permitidas | Sí, mediante usuarios autorizados del Workspace |
| Knowledge | Workspace | Documentos y bases de conocimiento de la empresa; acceso efectivo seleccionado para la instancia | La instancia no cambia la propiedad; un usuario autorizado administra Knowledge del Workspace y lo asigna/autoriza a sus agentes |
| Integraciones de AgentInstance | Workspace / AgentInstance | Conexión externa, referencia segura a credenciales y operaciones configuradas/habilitadas para esa instancia | Sí, por usuarios autorizados del Workspace, dentro de los conectores y controles de Waby |
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

**Catálogo y configuraciones iniciales:** Waby mantiene un catálogo de configuraciones de agente por objetivo empresarial (por ejemplo, Atención, Ventas, Soporte y Legal). Cada AgentDefinition referencia Skills estándar por identidad estable y puede recomendarlas como opciones. El usuario autorizado del Workspace decide cuáles habilita durante la configuración; una recomendación o una nueva Skill publicada no la activa automáticamente. El cliente no crea una AgentDefinition ni un catálogo técnico; selecciona una configuración inicial y personaliza su AgentInstance. El contenido funcional de estos catálogos se desarrolla en [Catálogo de agentes y Skills estándar](./catalogo-agentes-y-skills-estandar.md).

**Punto por confirmar:** la dirección de producto previa indica que cada empresa debe tener procedimientos propios. Se propone admitir procedimientos empresariales asociados a la AgentInstance, separados de las Skills estándar publicadas por Waby. No deben convertirse en código ejecutable ni ampliar capabilities automáticamente. La representación exacta (Skill empresarial versionada o instrucciones/procedimientos de instancia) sigue pendiente de aprobación.

### 4.4 Defaults y cambios de AgentDefinition

Propuesta:
- `defaultSkills` y `defaultRules` definen la propuesta de configuración inicial de la plantilla, no una herencia dinámica de toda la AgentDefinition.
- Durante la creación de una AgentInstance, la experiencia presenta las Skills recomendadas y permite al usuario autorizado decidir cuáles habilitar. La selección resultante pertenece a la instancia; no se altera por cambios posteriores del catálogo.
- Para una Skill habilitada, Waby conserva una identidad estable y administra sus releases. Las mejoras compatibles se distribuyen automáticamente mediante rollout controlado y pueden comunicarse con un aviso informativo; el cliente no mantiene versiones.
- Publicar una Skill nueva, agregarla a las recomendaciones de una AgentDefinition o ampliar el catálogo no la habilita en instancias existentes. El usuario debe decidir expresamente incorporarla.
- Los cambios de una AgentDefinition que alteren composición, recomendaciones, Skills o capacidades no modifican silenciosamente la configuración de instancias existentes; se presentan como propuestas controladas. Las nuevas Skills requieren habilitación expresa.
- Las políticas obligatorias de plataforma y las revocaciones de seguridad prevalecen sobre cualquier configuración previa. Waby puede bloquear una release vulnerable aunque una ejecución o instancia la tenga referenciada.
- Las acciones de publicación, rollout, promoción, rollback y revocación son administradas y auditadas por Waby.

Esta regla es consistente con P10: cada ejecución conserva un snapshot efectivo inmutable que identifica la release exacta de Skill utilizada. Las ejecuciones nuevas resuelven la release estable vigente; las ejecuciones anteriores no se reescriben. La política de rollout y la administración central se desarrollan en [Plan de administración y evolución del catálogo](./plan-administracion-catalogo-waby.md).

## 5. Capacidades ejecutables y autorización

La exposición de capacidades debe resultar de la intersección de todas las autorizaciones aplicables:

`CapabilitiesEffective = PlatformAllowed ∩ DefinitionAllowed ∩ InstanceEnabled ∩ CurrentAuthorization`

Para operaciones de integración se agrega la condición de ownership y autorización por instancia:
- La configuración de conexión, la referencia segura a credenciales y las operaciones disponibles se asocian a la AgentInstance y quedan dentro del Workspace propietario.
- Un usuario autorizado del Workspace configura, prueba, habilita, modifica o revoca esa conexión y sus operaciones para la instancia concreta.
- Waby mantiene los tipos de conectores y ejecutores soportados, protege los secretos y valida endpoint, esquema, argumentos, destino y límites. La configuración del cliente no equivale a código ejecutable ni habilita HTTP arbitrario.
- La Organización aporta el ámbito superior de tenancy y su plan determina la disponibilidad comercial de agentes IA; el administrador de Organización no administra la conexión de una AgentInstance por el mero hecho de ser administrador.
- Waby valida la cadena Organización → Workspace → AgentInstance y la autorización vigente antes de ejecutar; el LLM no puede seleccionar otro tenant, conexión ni URL arbitraria.

Una Capability no se habilita por el mero hecho de que una Skill la mencione. El prompt no es un mecanismo de autorización.

## 6. Matriz conceptual de autoridad (P3)

La siguiente matriz define responsabilidades de producto; **no aprueba nombres de permisos ni sustituye la matriz técnica de autorización**.

| Operación | Actor/owner propuesto | Alcance |
|---|---|---|
| Crear, versionar y publicar AgentDefinition | Waby | Plataforma |
| Elegir proveedor/modelo, gestionar credenciales LLM y límites globales | Waby | Plataforma |
| Mantener políticas obligatorias y catálogo de capabilities/Skills estándar | Waby | Plataforma |
| Administrar el plan que habilita la creación/uso de agentes IA | Administrador de la Organización | Organización |
| Configurar conexión externa y credenciales de una AgentInstance | Usuario autorizado del Workspace | Workspace + AgentInstance |
| Definir/seleccionar, habilitar, modificar o revocar operaciones para una AgentInstance | Usuario autorizado del Workspace | Workspace + AgentInstance |
| Crear una AgentInstance desde una definición disponible | Usuario autorizado del Workspace, con provisioning controlado por Waby | Workspace |
| Personalizar instrucciones y Rules empresariales de la instancia | Usuario autorizado del Workspace | AgentInstance |
| Seleccionar Knowledge disponible para la instancia | Usuario autorizado del Workspace, dentro del Knowledge de su Workspace | Workspace → AgentInstance |
| Habilitar Skills/opciones publicadas permitidas | Usuario autorizado del Workspace | AgentInstance |
| Habilitar operaciones de integración para una AgentInstance | Usuario autorizado del Workspace | Workspace + AgentInstance |
| Ejecutar una Capability | Runtime de Waby con identidad técnica acotada y contexto validado de tenant | Organización + Workspace + AgentInstance + Execution |
| Cambiar permisos, roles, miembros o estructura del Workspace | Usuario humano autorizado; nunca la AgentInstance | Workspace/Organización |
| Administrar miembros, invitar usuarios o modificar roles/permisos | Usuario humano autorizado conforme al modelo actual; nunca una AgentInstance | Workspace/Organización |

### Restricciones derivadas de D2

- No introducir `bot.configure` ni permisos `platform.*` como si ya fueran representables.
- Los permisos delegables al Workspace deben encajar en el modelo existente `org.*` / `wrk.*`, con validación de membresía y ownership; el alcance real debe probarse para evitar escalada mediante `org.*.*`.
- Las operaciones exclusivas de plataforma requieren un actor/identidad de Waby que el sistema actual todavía no representa. No se debe simular ese actor usando un usuario cliente ni un token compartido sin tenant.
- El runtime de una AgentInstance no hereda roles humanos. La identidad técnica debe recibir únicamente el contexto y las operaciones necesarias para esa ejecución.
- La matriz acumulativa conserva la procedencia de ARQ-03 y ARQ-13/ARQ-14; ARQ-14 queda resuelto en D.1.9/B. ARQ-19/20/21 mantienen sus decisiones aprobadas; el detalle operativo de ARQ-21 se difiere a WAB-80/WAB-81.

Los permisos de administración y lectura ya están definidos por ARQ-20; su incorporación al enum, asignación a roles y verificación de guards/ownership corresponden a WAB-80. La identidad de servicio base está seleccionada por ARQ-21 (JWT RS256); el inventario exhaustivo de operaciones/scopes, claims por flujo, rotación, replay y pruebas negativas es endurecimiento operativo diferido a WAB-80/WAB-81. Cada ejecución debe portar y validar el contexto aplicable (`orgId`, `workspaceId` y `agentInstanceId` cuando corresponda), comprobando la relación AgentInstance → Workspace → Organización antes de leer datos o ejecutar operaciones. No se crea un segundo sistema de permisos en WAB-78.

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

## 8. Acuerdos de producto y contratos aprobados

1. **Precedencia y composición (CTX-P3 / CTX-P4):** las políticas obligatorias y la configuración de plataforma de Waby prevalecen; AgentDefinition establece la base del agente; AgentInstance agrega personalización empresarial permitida; el contexto de ejecución aporta datos y no adquiere autoridad de instrucciones. Los conflictos se resuelven determinísticamente, no delegándolos al LLM.
2. **Instrucciones y Rules empresariales:** se agregan como bloques propios y no reemplazan el núcleo ni las políticas de Waby. No se ofrece un `systemPromptOverride` libre.
3. **Selección de Skills:** AgentDefinition puede recomendar Skills, pero el usuario autorizado del Workspace decide cuáles habilita. Publicar una Skill nueva o incorporarla a recomendaciones no la activa automáticamente. Las mejoras compatibles de una Skill ya habilitada se actualizan automáticamente bajo el rollout gestionado por Waby; el cliente puede recibir un aviso informativo y no administra versiones.
4. **Procedimientos empresariales:** cada AgentInstance puede disponer de procedimientos/instrucciones propios, separados del catálogo de Skills estándar de Waby. No son código ejecutable ni conceden permisos/capabilities. Su representación persistente exacta se definirá en WAB-79.
5. **Administración de AgentInstance:** un usuario autorizado del Workspace puede crear y configurar una instancia desde una AgentDefinition publicada. Waby conserva el provisioning y las operaciones técnicas de plataforma. La habilitación comercial de agentes IA depende del plan de la Organización, administrado por su administrador.
6. **Integraciones por instancia:** un usuario autorizado del Workspace configura la conexión externa, administra sus credenciales de forma segura y define/selecciona qué operaciones quedan disponibles para esa AgentInstance, dentro de los conectores y controles soportados por Waby. El plan de la Organización habilita comercialmente la funcionalidad de agentes IA, pero no convierte al administrador de Organización en responsable de configurar las instancias.
7. **Límites de la AgentInstance:** nunca administra usuarios, invita miembros, modifica membresías, roles, permisos ni estructura de la Organización/Workspace.
8. **Identidad y ámbito:** toda ejecución actúa dentro de una Organización, un Workspace y una AgentInstance concretos. Waby valida la cadena de ownership y las autorizaciones en backend/runtime; no se confía en IDs aportados por el LLM ni en un token interno sin contexto de tenant.

**Pendientes técnicos de integraciones:** flujo UX/técnico para configurar credenciales; cómo se definen/importan operaciones; validaciones de destino/esquema; y si en el futuro se permitirá compartir explícitamente una conexión entre varias AgentInstances del mismo Workspace. La configuración inicial se considera privada de cada AgentInstance. ARQ-19 y ARQ-20 ya no son pendientes documentales. ARQ-21 conserva los pendientes de especificación e implementación enumerados en §8.1.

## 8.1 Resolución y estado de ARQ-19, ARQ-20 y ARQ-21

### ARQ-19 — Autoridad híbrida por operación

La decisión está resuelta. Waby administra AgentDefinitions, catálogos centrales, proveedores/modelos/credenciales LLM, políticas obligatorias, provisioning técnico y purga física bajo política. Los usuarios humanos autorizados del Workspace administran integralmente las AgentInstances de su Workspace: creación desde definiciones publicadas, consulta, configuración empresarial, Knowledge/Skills/Rules permitidas, integraciones y credenciales asociadas a la instancia, activación, suspensión/reactivación, designación predeterminada y eliminación lógica.

El administrador de Organización conserva la administración del plan/entitlement comercial, pero ese rol no concede por sí solo control de AgentInstances. La administración de instancia no permite modificar AgentDefinitions, políticas ni recursos centrales de Waby. La ejecución de Capabilities mantiene autorización contextual independiente.

### ARQ-20 — Permisos de Workspace

El contrato documental está aprobado y cerrado:

- `wrk.agents.*`: concesión integral para administrar AgentInstances.
- `wrk.agents.read`: consulta/visualización sin facultades administrativas.
- Workspace Owner: `wrk.agents.*` por herencia de `wrk.*.*`.
- Workspace Manager: `wrk.agents.*` explícito.
- Workspace Agent: `wrk.agents.read`.

Ambos identificadores deben declararse explícitamente en `permission.enum.ts`; la interpretación genérica de comodines no sustituye su declaración formal. No se crea `wrk.agents.manage`, `wrk.agent.use` ni permisos administrativos separados por operación. Estos permisos no sustituyen membership/ownership ni la autorización contextual de cada Capability. La implementación y las pruebas corresponden a WAB-80.

### ARQ-21 — Identidad y autenticación entre servicios

La decisión aprobada es la **Opción C: JWT de servicio firmados, de corta duración y con credenciales diferenciadas por dirección**:

- CRM → ai-service: identidad de servicio CRM, token destinado a ai-service.
- ai-service → CRM: identidad de servicio ai-service, token destinado a CRM.
- Algoritmo de firma seleccionado: **RS256** (firma asimétrica). Cada dirección utiliza credenciales de firma diferenciadas; el receptor valida con la clave pública correspondiente.
- Se emite un JWT por petición y con el scope mínimo necesario para esa operación.
- Claims acordados: `iss`, `sub`, `aud`, `iat`, `nbf`, `exp`, `jti`, `scope`, `orgId`, `workspaceId`, `agentInstanceId` y `initiatedBy`.
- `initiatedBy` conserva la identidad del usuario iniciador para auditoría; **no** autentica al servicio ni concede autorización.
- El receptor valida algoritmo/firma, emisor, sujeto de servicio esperado, audiencia, vigencia, scope y contexto tenant/recurso. Después verifica el ownership y el estado vigentes en CRM. Los claims no sustituyen la autorización de dominio.
- Cuando una operación no corresponde a una AgentInstance, el contexto de instancia se aplica según el alcance real de esa operación; no se debe inventar un recurso.

**Pendiente para completar ARQ-21:** inventario exhaustivo de operaciones internas en ambas direcciones; scope mínimo por operación; claims obligatorios/opcionales por flujo; provisión, rotación y revocación de claves; semántica de unicidad/replay de `jti`; confianza/autenticación de mensajes RabbitMQ; pruebas negativas y verificación de despliegue. La decisión de arquitectura está seleccionada, pero la implementación y el comportamiento real en producción no se consideran verificados.

El detalle de DEC-CM-01 a DEC-CM-08 —versionado efectivo, `additionalPrompt`, parámetros técnicos, selección de instancia, cambio de definición, asignación, suspensión y predeterminada— está en [Decisiones operativas de AgentInstance](./decisiones-operativas-agent-instance.md) y en el registro operativo de WAB-78.

## 9. Fuera de alcance / no reabrir

- P4 de créditos G.4.6 ya fue aprobada: no se modifica.
- P10 snapshot inmutable por ejecución ya fue aprobado: esta propuesta solo detalla cómo aplicarlo a la configuración.
- No se repiten D1-D4 ni se cambian hallazgos ARQ.
- No se define el esquema persistente final, índices, migraciones ni código: corresponden a las fases posteriores.
- WAB-78 no se cierra ni WAB-79 se inicia con esta propuesta.
