# Integraciones API externas por AgentInstance — diseño de WAB-78

**Estado:** configuración de conexiones, credenciales y operaciones por AgentInstance administrada por usuarios autorizados del Workspace; plan comercial de agentes IA administrado a nivel Organización. Contratos técnicos detallados pendientes dentro de WAB-78.  
**Fecha:** 05/11/2026.  
**Alcance:** capacidad para que una AgentInstance consulte o ejecute operaciones explícitamente configuradas contra APIs de sistemas de la Organización cliente. No define todavía el modelo persistente final ni autoriza implementación.

## 1. Decisión de ownership y autoridad

- La **configuración de conexión externa se asocia a una AgentInstance concreta** y queda dentro del Workspace al que pertenece esa instancia.
- Un **usuario autorizado del Workspace** configura la conexión, endpoint/base URL permitida, método de autenticación, credenciales y operaciones que el agente podrá utilizar. También puede probar, modificar, deshabilitar o revocar esa configuración conforme a sus permisos.
- La credencial es parte de la configuración de la instancia, pero el secreto en claro no se almacena ni se expone en la configuración ordinaria: Waby lo guarda en un almacén seguro/cifrado y mantiene una referencia protegida vinculada a Organization + Workspace + AgentInstance.
- Waby mantiene los tipos de conectores, ejecutores y controles de seguridad soportados. El usuario configura cómo se conecta su agente al sistema externo y qué operaciones concretas necesita, dentro de los mecanismos validados por Waby.
- El plan de la **Organización** determina si la funcionalidad de creación/uso de agentes IA está comercialmente habilitada. Su administrador gestiona el plan; no administra las conexiones ni la configuración particular de las AgentInstances por el mero hecho de ser administrador de Organización.
- La Organización sigue siendo el ámbito superior de tenancy. Toda configuración e invocación de integración de una instancia debe pertenecer a la misma cadena Organización → Workspace → AgentInstance.
- Por defecto, una conexión es privada de la AgentInstance para la que se configuró. Compartir una conexión entre varias instancias del mismo Workspace podría diseñarse más adelante, pero requeriría una acción explícita y grants independientes; no se presume compartición.

La capacidad de invocar APIs configuradas por el cliente no equivale a navegación libre por Internet ni habilita conocimiento externo general. El agente solo puede operar sobre integraciones y operaciones registradas y autorizadas expresamente.

**Carácter opcional:** una AgentInstance puede funcionar sin ninguna integración externa. Una empresa que no tenga ERP ni otros sistemas conectables puede crear y utilizar un agente únicamente con sus instrucciones, Knowledge y demás recursos que aporte a Waby. Las integraciones amplían capacidades cuando existe una necesidad concreta; no son un requisito de configuración, activación ni funcionamiento. La experiencia de conexión debe estar guiada y no exigir que el usuario conozca APIs o conceptos técnicos. Ver [Experiencia de configuración de agentes](./experiencia-configuracion-agentes.md).

## 2. Modelo conceptual propuesto

| Concepto | Ownership / alcance | Responsabilidad |
|---|---|---|
| ConnectorDefinition | Plataforma Waby | Tipo de conector, protocolo/ejecutor y controles soportados por Waby. No contiene credenciales de clientes. |
| AgentInstanceIntegration | Workspace + AgentInstance | Configuración de conexión a un sistema externo para esa instancia: tipo de conector, destino permitido, parámetros no secretos y referencia segura a credenciales. |
| AgentInstanceOperation | AgentInstance | Operación concreta disponible para el agente: identificador, intención, método/ruta relativa, parámetros, schemas de entrada/salida, efecto y controles. |
| IntegrationInvocation | Organización + Workspace + AgentInstance + Execution | Registro auditable de cada invocación y su resultado operativo, con la cadena de ownership validada. |

Los nombres son conceptuales; no representan todavía nombres de colecciones, clases o esquemas aprobados.

### Ownership y configuración

- La configuración de conexión, la referencia a credenciales y las operaciones pertenecen al ámbito de la AgentInstance/Workspace. Un usuario autorizado del Workspace es responsable de configurarlas para esa instancia.
- Waby administra los ConnectorDefinitions y los ejecutores seguros. Las operaciones configuradas por el cliente deben ajustarse a esos mecanismos y validaciones; no se admite código arbitrario ni solicitudes HTTP construidas libremente por el LLM.
- Waby almacena los secretos cifrados en un almacén seguro. La AgentInstance conserva solo referencias protegidas y metadata no secreta; los secretos nunca se incorporan al prompt, al contexto del LLM ni a logs en claro.
- La autorización debe comprobar que la AgentInstance pertenece al Workspace y que ambos pertenecen a la Organización indicada. El organizationId se utiliza para validar tenancy y no convierte la configuración en un recurso administrado a nivel Organización.
- Cada AgentInstance configura su propia conexión por defecto. Si en el futuro se permite compartir una conexión dentro del Workspace, se modelará explícitamente y exigirá autorización independiente por instancia.

## 3. Contrato de una operación

Cada operación registrada debería declarar, como mínimo:

- Identidad estable y versión.
- Nombre y descripción orientados a la intención de negocio; instrucciones claras sobre cuándo usarla y cuándo no.
- Método HTTP y ruta relativa respecto de la conexión configurada para esa AgentInstance.
- Parámetros de path/query y esquema estructurado de entrada, con tipos, obligatoriedad y validaciones.
- Esquema de salida esperado y reglas de normalización/redacción.
- Clasificación de efecto: lectura, escritura o sensible.
- Política de confirmación, límites operativos y comportamiento ante errores.

Ejemplo ilustrativo de una integración ERP:

| Operación | Efecto | Entradas |
|---|---|---|
| consultar_cliente_por_telefono | Lectura | phone |
| consultar_saldo_cliente | Lectura | customerId |
| generar_documento | Escritura | customerId, templateId, datos documentales |

El LLM no construye URLs ni solicitudes HTTP arbitrarias. Selecciona una operación autorizada y proporciona argumentos que se validan contra su esquema. El ejecutor de Waby resuelve la conexión, construye la solicitud y aplica las políticas de seguridad.

## 4. Exposición al agente y ejecución

Flujo objetivo:

1. Al preparar una ejecución, Waby resuelve la configuración efectiva de la AgentInstance y sus grants vigentes.
2. El contexto de herramientas del LLM contiene únicamente las operaciones habilitadas para esa instancia, con descripciones y esquemas de argumentos necesarios.
3. El LLM solicita una operación mediante su identificador y argumentos estructurados.
4. Waby vuelve a validar identidad, ownership, grant, estado de integración, versión de operación y argumentos antes de ejecutar.
5. Un ejecutor controlado realiza la llamada externa usando la credencial protegida asociada a esa AgentInstance.
6. Waby valida, limita y normaliza la respuesta antes de devolver el resultado al pipeline/LLM.
7. La ejecución continúa o responde al usuario según el resultado y las reglas de la AgentInstance.

Las credenciales, tokens, headers sensibles y secretos nunca se incorporan al prompt, al contexto del LLM ni a los logs en claro. El agente no recibe acceso genérico a un cliente HTTP.

## 5. Seguridad y aislamiento obligatorios

- **Sin HTTP arbitrario:** el modelo no puede elegir host, esquema, puerto ni URL completa. Solo invoca operaciones registradas.
- **Control de salida:** el usuario autorizado del Workspace configura el destino de la conexión; Waby valida destinos HTTPS, DNS/resolución, redirecciones y direcciones privadas para proteger contra SSRF. Las excepciones de red deben ser explícitas y controladas por Waby. El LLM nunca puede cambiar el host/destino.
- **Secretos:** almacenamiento cifrado, referencias seguras y rotación/revocación sin exponer credenciales al agente.
- **Validación:** validar entradas antes de enviar y respuestas externas antes de incorporarlas al contexto.
- **Respuestas no confiables:** tratar los datos devueltos por la API como datos, nunca como instrucciones de sistema o cambios de política.
- **Límites:** timeout, tamaño máximo de request/response, rate limit, concurrencia, cuotas y límites de reintentos.
- **Aislamiento:** toda invocación queda vinculada a organizationId, workspaceId, agentInstanceId y executionId; no debe ser posible resolver grants o secretos de otro tenant.
- **Revocación:** deshabilitar una integración, revocar credenciales o retirar una autorización debe bloquear nuevas invocaciones. Esta revocación de seguridad prevalece sobre la continuidad de una ejecución en curso.

## 6. Versionado y consistencia con P10

La configuración efectiva inmutable por ejecución debe identificar, sin incluir secretos:

- Identidad y versión de la configuración de conexión de la AgentInstance.
- Identificador del conector y de las operaciones habilitadas, sus versiones y hashes de schemas/configuración efectiva.
- Identificador de la referencia segura de credencial (nunca el secreto) y versión/rotación aplicable.
- Autorización vigente de la AgentInstance que permitió la operación, vinculada a Organization, Workspace, AgentInstance y Execution.

Los cambios normales de catálogo, parámetros o grants se aplican a nuevas ejecuciones, en línea con P10. La revocación urgente de seguridad es una excepción deliberada: debe impedir nuevas llamadas aunque una ejecución conserve un snapshot previo. La semántica exacta de revocación y de credenciales rotadas debe cerrarse en el contrato de runtime.

## 7. Documentos y archivos devueltos por APIs

Una API puede devolver metadatos de un documento o una referencia a un archivo. No se debe enviar automáticamente cualquier URL devuelta por el proveedor externo.

La entrega de documentos requiere un flujo controlado por Waby que valide origen permitido, tipo MIME, tamaño, acceso, expiración y asociación con la ejecución/contacto. El mecanismo exacto de descarga, almacenamiento temporal y envío como medio de Waby queda pendiente de diseño técnico.

## 8. Auditoría

Cada invocación debe permitir reconstruir quién pidió qué operación, bajo qué autorización y con qué resultado operativo. Como mínimo: IDs de tenant/workspace/instancia/ejecución, integración y versión de operación, fecha, duración, estado HTTP/clasificación de error, intento y correlación. Requests y responses solo se conservan conforme a una política explícita de minimización y redacción; nunca se registran secretos en claro.

## 9. Puntos aún pendientes de contrato en WAB-78

1. Permisos exactos para que usuarios autorizados del Workspace creen, configuren, prueben, actualicen y revoquen conexiones, credenciales y operaciones por AgentInstance. Deben derivarse de la matriz real de permisos; no inventar permisos nuevos en esta propuesta.
2. Cómo se definen las operaciones de cada AgentInstance: selección de operaciones/plantillas soportadas por Waby, configuración manual, importación OpenAPI o combinación; qué validaciones realiza Waby antes de habilitarlas.
3. Métodos de autenticación y alcance de la primera versión (API key, Bearer estático, OAuth u otros).
4. Tratamiento de operaciones de escritura/sensibles: cuándo se exige confirmación humana o del usuario final y cómo evitar duplicaciones ante reintentos.
5. Versionado/publicación de operaciones y comportamiento de ejecuciones en curso.
6. Si se permite compartir una conexión entre varias AgentInstances del mismo Workspace en una etapa posterior; no forma parte del supuesto inicial, que es conexión privada por instancia.
7. Política detallada de retries, idempotencia, timeout, límites y circuit breaker.
8. Contrato de salida estructurada, redacción de PII y entrega de archivos.
9. Si se admiten destinos privados o redes corporativas y bajo qué mecanismo seguro de conectividad.

Estos puntos no invalidan la decisión aprobada de ownership, pero sí impiden considerar cerrado el contrato completo de integraciones.

## 10. Límites de alcance

Esta propuesta no introduce navegación libre, búsqueda web abierta ni acceso general del agente a Internet. Tampoco autoriza que clientes modifiquen el proveedor LLM contratado/administrado por Waby, ni que las integraciones accedan al código o configuración interna de Waby. Las integraciones son una capacidad explícita y gobernada para conectar los sistemas de negocio de la Organización cliente.
