# Experiencia de configuración de agentes — principios de producto

**Estado:** principios de producto aprobados para orientar el diseño de WAB-78.  
**Fecha:** 05/11/2026.  
**Alcance:** experiencia de creación, configuración, prueba y puesta en marcha de agentes de IA para usuarios de Waby. No define todavía pantallas finales ni un prototipo visual.

## 1. Principios aprobados

### 1.1 Un agente debe funcionar sin integraciones externas

- Conectar un ERP, CRM externo u otra API es **opcional**.
- Una empresa puede crear y utilizar un agente sin tener ERP ni conocimientos técnicos de integración.
- El agente debe poder trabajar con las instrucciones, el conocimiento y los recursos que la empresa le proporcione dentro de Waby.
- La ausencia de integraciones no debe bloquear la creación, activación ni el funcionamiento normal del agente.
- Las integraciones se incorporan después, cuando exista una necesidad concreta, como una ampliación de capacidad.

### 1.2 La configuración debe ser sencilla para personas no técnicas

La interfaz está dirigida a usuarios que pueden no conocer conceptos de IA, programación, APIs o infraestructura. Waby debe resolver la complejidad técnica internamente y presentar decisiones comprensibles para el usuario.

La experiencia debe priorizar:

- Lenguaje cotidiano y orientado a tareas de negocio.
- Pasos guiados, cortos y con una secuencia clara.
- Valores predeterminados seguros y razonables.
- Ayuda contextual, ejemplos y explicaciones breves sobre el efecto de cada opción.
- Validaciones comprensibles que indiquen qué falta y cómo resolverlo.
- Una forma simple de probar el agente antes de activarlo.
- Posibilidad de completar la configuración básica sin atravesar opciones avanzadas.

No se deben exigir al usuario conceptos como system prompt, embeddings, JSON Schema, tool calling, endpoints, headers HTTP o configuración de modelos para completar la creación básica de un agente.

### 1.3 Complejidad progresiva, no impuesta

La experiencia inicial debe cubrir el caso común y mantener las capacidades avanzadas disponibles solo cuando sean necesarias.

- La configuración básica debe permitir crear un agente, definir para qué ayuda, aportar conocimiento, ajustar instrucciones de comportamiento mediante controles comprensibles, probarlo y activarlo.
- Las opciones avanzadas se presentan en un segundo nivel, con explicación de su impacto y sin interferir en el flujo básico.
- La conexión de APIs e integraciones se ofrece como una capacidad opcional; no debe aparecer como requisito ni como paso obligatorio para terminar la configuración.
- Cuando el usuario habilite una capacidad avanzada, Waby debe guiarlo y explicar qué información necesita, por qué la necesita y qué podrá hacer el agente después.

## 2. Catálogo de agentes y Skills estándar

Waby debe ofrecer dos catálogos centrales para que el usuario no tenga que diseñar un agente desde cero:

- **Catálogo de agentes:** configuraciones iniciales por objetivo empresarial, por ejemplo Atención, Ventas, Soporte y Legal. Cada opción explica qué hace, qué puede resolver y qué información conviene aportar.
- **Catálogo de Skills estándar:** procedimientos reutilizables, como calificar una oportunidad de venta, clasificar consultas, detectar señales de malestar, resumir conversaciones o preparar una transferencia a una persona.

Una configuración de agente puede incluir algunas Skills por defecto y recomendar otras. El usuario puede comprender para qué sirve cada una y habilitar las que necesite sin conocer términos técnicos. Las Skills estándar son mantenidas y versionadas por Waby; los procedimientos propios de la empresa se mantienen diferenciados y no otorgan permisos ejecutables.

Las categorías y ejemplos iniciales se describen en [Catálogo de agentes y Skills estándar](./catalogo-agentes-y-skills-estandar.md). Son puntos de partida, no una lista cerrada.

## 3. Recorrido básico de referencia

Este recorrido expresa la intención de producto, no una secuencia de pantallas aprobada:

1. **Crear agente:** elegir un nombre y describir en términos sencillos qué función cumplirá.
2. **Definir comportamiento:** seleccionar o redactar instrucciones mediante preguntas y ejemplos, evitando obligar a escribir prompts técnicos.
3. **Agregar conocimiento:** cargar documentos o aportar contenido de la empresa; explicar qué tipo de información ayuda al agente y cómo se utiliza.
4. **Personalizar:** configurar tono, idioma y otras preferencias comunes con opciones comprensibles y valores predeterminados.
5. **Probar:** conversar con el agente en un entorno de prueba y mostrar cómo utiliza el conocimiento cargado.
6. **Activar:** revisar un resumen claro de lo configurado y activar el agente.
7. **Ampliar después:** permitir agregar Skills, integraciones, operaciones y ajustes avanzados cuando aparezca esa necesidad.

Un usuario debe poder completar este recorrido sin registrar una integración externa.

## 4. Cómo presentar las integraciones

Las integraciones son extensiones opcionales del agente, no una condición para que exista.

- Presentarlas a partir de necesidades concretas: por ejemplo, consultar datos de clientes en el sistema de la empresa o generar un documento.
- No exigir que el usuario conozca HTTP, métodos, headers, autenticación, rutas ni esquemas.
- Guiar la conexión mediante pasos y explicaciones; cuando se requiera información técnica que el usuario no tenga, ofrecer instrucciones claras para obtenerla o indicar qué responsable de su empresa puede ayudar.
- Mostrar qué puede hacer cada operación y qué datos utiliza, en lenguaje de negocio.
- Distinguir claramente operaciones de consulta de aquellas que modifican datos o producen efectos externos.
- Permitir probar la conexión y la operación antes de habilitarla para una AgentInstance.
- Mantener los controles técnicos, de seguridad y aislamiento en Waby, aunque la interfaz sea sencilla.

La Organización sigue siendo propietaria de las integraciones; cada Workspace decide qué operaciones habilita para sus AgentInstances, conforme al documento [Integraciones API externas](./integraciones-api-externas.md).

## 5. Separación entre simplicidad de UX y rigor interno

La simplicidad visible no implica reducir los controles internos.

Waby debe conservar, aunque no exponga complejidad innecesaria al usuario:

- Aislamiento por Organización y Workspace.
- Autorización explícita de capacidades y operaciones.
- Protección y gestión segura de credenciales.
- Validación de entradas, salidas y documentos.
- Auditoría, límites operativos, concurrencia e idempotencia cuando corresponda.
- Versionado y snapshot efectivo por ejecución según P10.
- Mensajes de error accionables, sin filtrar detalles internos o secretos.

La interfaz explica el resultado y las decisiones relevantes; Waby se responsabiliza de los mecanismos técnicos.

## 6. Criterios para evaluar futuras propuestas de UX

Una propuesta de pantalla, flujo o configuración debe poder responder afirmativamente:

1. ¿Un usuario no técnico entiende qué está configurando y para qué sirve?
2. ¿Puede completar el caso común sin conocer conceptos internos de IA o programación?
3. ¿Puede crear y activar un agente útil sin ERP ni integraciones?
4. ¿Las opciones avanzadas son opcionales y aparecen cuando aportan valor?
5. ¿La ayuda explica cómo avanzar ante dudas o errores?
6. ¿Los permisos, riesgos y efectos externos se comunican antes de habilitar una capacidad?
7. ¿La experiencia mantiene controles de seguridad y aislamiento sin trasladar su complejidad al usuario?

Si una propuesta obliga a configurar integraciones, escribir prompts técnicos o entender conceptos de infraestructura para crear un agente básico, no cumple estos principios.

## 7. Pendiente de diseño

Estos principios están aprobados, pero siguen pendientes de elaboración dentro del diseño de producto:

- Estructura y contenido exacto del asistente de creación.
- Plantillas iniciales por objetivo o rubro, y cómo evitar que limiten la personalización.
- Modelo de ayuda contextual, ejemplos y estados vacíos.
- Experiencia de carga, actualización y publicación del conocimiento.
- Interfaz de prueba y criterios de preparación antes de activar.
- Separación visual entre configuración básica y avanzada.
- Recorrido guiado para integraciones y operaciones sensibles.
- Accesibilidad, responsive y validación con usuarios no técnicos.

Este documento establece el criterio de producto; no aprueba todavía el diseño visual final ni autoriza implementación.
