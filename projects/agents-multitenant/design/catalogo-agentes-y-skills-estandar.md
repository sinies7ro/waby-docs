# Catálogo de agentes y Skills estándar de Waby

**Estado:** dirección de producto aprobada para mantener catálogos centrales; política de versionado, actualización y rollout propuesta, pendiente de confirmación; contratos de dominio y UX detallados pendientes dentro de WAB-78.  
**Fecha:** 05/11/2026.  
**Alcance:** catálogo central de configuraciones iniciales de agentes y catálogo reutilizable de Skills estándar. No define todavía pantallas finales ni esquemas persistentes.

## 1. Decisión de producto

Waby debe ofrecer dos catálogos relacionados:

1. **Catálogo de agentes:** configuraciones iniciales preparadas por Waby para distintos objetivos empresariales, por ejemplo Atención, Ventas, Soporte y Legal.
2. **Catálogo de Skills estándar:** procedimientos reutilizables mantenidos y versionados por Waby, que pueden formar parte de una o varias configuraciones de agente.

El cliente no comienza desde una pantalla técnica vacía. Elige una configuración cercana a su necesidad, aporta el conocimiento de su empresa y personaliza el comportamiento mediante una experiencia guiada.

El catálogo es un punto de partida, no una restricción: el usuario puede personalizar su agente dentro de las políticas y capacidades permitidas por Waby.

## 2. Catálogo de agentes

Las categorías iniciales son orientativas y pueden crecer:

| Configuración | Objetivo principal | Skills estándar que podría incluir |
|---|---|---|
| Atención | Resolver consultas habituales, identificar necesidades y gestionar derivaciones | Clasificar consulta, detectar señales de malestar, resumir conversación, preparar transferencia |
| Ventas | Identificar oportunidades y acompañar consultas comerciales | Detectar intención de compra, calificar oportunidad, recopilar necesidades y presupuesto |
| Soporte | Comprender problemas y orientar su resolución | Clasificar incidente, recopilar datos, guiar diagnóstico, resumir caso y escalar |
| Legal | Orientar sobre información documental autorizada y recopilar datos para revisión profesional | Clasificar consulta documental, extraer información relevante, resumir antecedentes y derivar a profesional |

La categoría Legal requiere límites y mensajes de alcance especialmente claros: el agente no debe presentarse como profesional ni sustituir revisión legal humana. La definición precisa de restricciones y flujos será parte del diseño específico de esa configuración.

### Contenido conceptual de una configuración

Una entrada del catálogo de agentes (basada en una AgentDefinition versionada y administrada por Waby) puede incluir:

- Nombre, categoría, propósito y descripción comprensible.
- Instrucciones base y comportamiento esperado.
- Rules centrales, distinguiendo las obligatorias de las personalizables.
- Referencias a versiones concretas de Skills estándar, clasificadas como incluidas por defecto o recomendadas.
- Capacidades compatibles o requeridas, siempre sujetas a autorización independiente.
- Campos de personalización que el usuario puede completar (por ejemplo, tono, idioma, políticas comerciales o información de atención).
- Sugerencias sobre el tipo de Knowledge que conviene cargar.
- Ejemplos de preguntas para probar el agente y criterios de preparación antes de activarlo.

El catálogo no debe exigir integraciones externas. Una configuración de Atención, Ventas o Soporte debe poder funcionar con instrucciones y Knowledge aportado por la empresa. Una integración (ERP, CRM externo, sistema de tickets, etc.) amplía el agente solo cuando el cliente decide incorporarla.

## 3. Catálogo de Skills estándar

Waby mantiene un catálogo de procedimientos reutilizables, publicados y versionados. Una Skill estándar contiene instrucciones estructuradas para realizar una tarea, criterios de entrada/salida y comportamiento ante incertidumbre.

Ejemplos iniciales:

| Skill | Resultado esperado |
|---|---|
| Calificar oportunidad de venta | Identificar necesidad, producto/servicio de interés, presupuesto aproximado, plazo e intención; señalar información faltante |
| Detectar señales de malestar | Identificar señales conversacionales de frustración, enojo o insatisfacción, con nivel de confianza y señales que sustentan la estimación |
| Detectar intención de compra | Clasificar señales de interés o intención de adquirir un producto/servicio |
| Clasificar consulta | Identificar categoría y motivo de la consulta |
| Resumir conversación | Preparar un resumen factual y útil para continuar la atención |
| Preparar transferencia | Resumir el motivo, contexto, datos recopilados y expectativa del contacto para el siguiente responsable |
| Recopilar datos de soporte | Obtener de forma ordenada los datos necesarios para entender un incidente |

Una misma Skill puede ser referenciada por varias configuraciones de agente. No se duplica su definición por cada categoría.

### Versionado y personalización

- Las Skills estándar son administradas por Waby. Cada release publicada es inmutable y se conserva para trazabilidad, pruebas, auditoría y rollback seguro.
- AgentDefinition referencia Skills por identidad estable y declara si se incluyen por defecto o se recomiendan; no obliga al cliente a administrar números de versión.
- AgentInstance conserva qué Skills tiene habilitadas y su configuración empresarial permitida. Waby resuelve la release vigente según el canal de actualización gestionado.
- Las actualizaciones compatibles se distribuyen automáticamente mediante rollout controlado, sin exigir intervención técnica del cliente y sin sobrescribir sus personalizaciones.
- P10 registra en cada ejecución la release exacta de Skill utilizada. Las ejecuciones anteriores conservan su snapshot; las siguientes pueden utilizar la release estable promovida por Waby.
- Las actualizaciones de seguridad pueden bloquear/revocar releases vulnerables y prevalecen sobre la continuidad de versiones anteriores.
- La empresa puede agregar procedimientos o instrucciones propios. Se distinguen de las Skills estándar, se revisan/versionan internamente como parte de la configuración de instancia y no se convierten en código ejecutable ni conceden permisos por sí mismos.

La política operativa completa (autoría, pruebas, aprobación, rollout, rollback, retirada y seguridad) está en [Plan de administración y evolución del catálogo](./plan-administracion-catalogo-waby.md).

## 4. Ejemplo: agente de Atención que detecta malestar y deriva a una persona

### Configuración inicial

El cliente selecciona **Atención**. Waby recomienda incluir estas Skills:

1. Clasificar consulta.
2. Detectar señales de malestar.
3. Resumir conversación.
4. Preparar transferencia.

El cliente aporta sus horarios, políticas de atención, preguntas frecuentes y documentación. No necesita conectar un ERP ni ninguna API.

### Comportamiento de la Skill de detección

La Skill analiza únicamente señales de la conversación actual y puede producir un resultado estructurado como:

- Clasificación: neutral / frustración / enojo / insatisfacción.
- Nivel de confianza: bajo / medio / alto.
- Señales observadas: expresiones o hechos conversacionales concretos.
- Recomendación: continuar con atención normal, responder con empatía, pedir aclaración o recomendar transferencia.

La etiqueta describe señales de interacción observadas, no un diagnóstico psicológico ni una característica permanente del contacto. Si la evidencia es ambigua, el agente no debe afirmar que conoce el estado emocional de la persona.

### Decisión y derivación

Ejemplo de regla de producto:

- Si el contacto pide hablar con una persona, se respeta la solicitud conforme a las reglas de atención.
- Si hay señales claras y reiteradas de enojo o una situación que el agente no puede resolver, se recomienda o inicia una transferencia según la política configurada.
- Si la confianza es baja, el agente responde con empatía o pregunta cómo puede ayudar; no etiqueta categóricamente ni transfiere automáticamente por una inferencia débil.

La Skill no realiza la transferencia. El flujo es:

1. La Skill genera la clasificación y la recomendación.
2. Las reglas de la AgentInstance y las políticas Waby determinan si corresponde derivar.
3. El runtime solicita una Capability autorizada de transferencia.
4. Waby valida el responsable/equipo de destino y ejecuta la transferencia conforme al contrato de asignaciones del Workspace.
5. Se prepara un resumen de la conversación para que la persona que recibe el caso no obligue al cliente a repetir todo.
6. La AgentInstance deja de responder cuando la transferencia/takeover se confirma, de acuerdo con el contrato de routing y asignación vigente.

La transferencia debe respetar las reglas aprobadas de P2: destinos elegibles del mismo Workspace, validación por Waby, manejo de falta de destinatarios y prevención de respuestas tardías del agente anterior.

### Auditoría y datos

Se debe conservar solo la información operativa necesaria para explicar la decisión y la derivación, según la política de retención que se defina. No se debe convertir una estimación momentánea de enojo en un perfil emocional permanente del contacto por defecto.

## 5. Relación entre los dos catálogos

- El catálogo de agentes organiza la experiencia por objetivos de negocio.
- El catálogo de Skills ofrece procedimientos reutilizables.
- Una configuración de agente puede recomendar, incluir o permitir habilitar Skills estándar.
- Una Skill puede utilizarse en varias configuraciones.
- La AgentInstance conserva las versiones concretas que utiliza.
- Las capacidades ejecutables se autorizan de manera independiente; una Skill no concede acceso a herramientas, datos ni integraciones.

## 6. Experiencia de usuario prevista

El recorrido deseado:

1. El usuario elige «Quiero un agente para Atención / Ventas / Soporte / Legal».
2. Waby explica en pocas líneas para qué sirve esa configuración.
3. Muestra qué sabe hacer inicialmente y qué Skills incluye, en términos cotidianos.
4. El usuario agrega información de su empresa y personaliza las preguntas/comportamientos que correspondan.
5. Waby recomienda Skills opcionales según el objetivo, explicando su utilidad sin jerga técnica.
6. El usuario prueba conversaciones de ejemplo y ve cómo responde o deriva el agente.
7. Activa el agente.

Las opciones avanzadas y las integraciones se ofrecen cuando tienen sentido, nunca como requisito para completar el flujo básico.

## 7. Límites y pendientes

- Este documento registra la dirección de producto aprobada, no un catálogo inicial cerrado ni el número definitivo de categorías/Skills.
- Falta definir el ciclo editorial y de publicación del catálogo Waby, responsables, pruebas, compatibilidad y retiro de versiones.
- Falta definir cuáles Skills son incluidas por defecto, recomendadas u opcionales por plantilla.
- Falta precisar el modelo de procedimientos empresariales propios y su versionado.
- Falta diseñar los umbrales, señales, explicaciones y controles configurables para la detección de malestar.
- Falta diseñar la política exacta para transferencia automática versus recomendación/confirmación, coordinada con P2 y permisos.
- Falta establecer retención y visibilidad de las señales de malestar, minimizando datos persistentes.
- No se aprueban todavía pantallas finales, schemas, permisos, migraciones ni implementación.

## 8. Relación con decisiones anteriores

- La propiedad de AgentDefinition y las políticas centrales sigue siendo de Waby.
- AgentInstance sigue perteneciendo al Workspace.
- Skills estándar son administradas/versionadas por Waby; procedimientos empresariales pertenecen al alcance de la instancia y no otorgan capacidades.
- Las integraciones siguen perteneciendo a la Organización y cada Workspace habilita operaciones por AgentInstance.
- Las integraciones son opcionales: el agente funciona con instrucciones y Knowledge.
- P10 sigue vigente: cada ejecución conserva un snapshot efectivo inmutable.
- Las reglas aprobadas de routing/delegación de P2 gobiernan la transferencia a humanos/equipos.
- Esta definición no cierra CTX-P3/CTX-P4 ni autoriza iniciar WAB-79.
