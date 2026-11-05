# Integraciones API externas — diseño de WAB-78

**Estado:** ownership de Integración a nivel Organización y habilitación por AgentInstance a nivel Workspace aprobados; actor de administración de conexiones/credenciales/catálogo de ámbito Organización y contratos técnicos siguen pendientes de definición en WAB-78.  
**Fecha:** 05/11/2026.  
**Alcance:** capacidad para que una AgentInstance consulte o ejecute operaciones explícitamente configuradas contra APIs de sistemas de la Organización cliente. No define todavía el modelo persistente final ni autoriza implementación.

## 1. Decisión aprobada

- Una **Integración pertenece a una Organización**.
- La Integración, la conexión con el sistema externo, las credenciales y el catálogo de operaciones pertenecen al ámbito de la Organización.
- El plan de la Organización determina si la funcionalidad de agentes IA está comercialmente habilitada; su administrador gestiona el plan, no la configuración particular de las AgentInstances.
- Cada **Workspace decide qué operaciones habilita para cada una de sus AgentInstances**, mediante un usuario autorizado del Workspace.
- El administrador de la Organización no habilita Skills ni operaciones de integración para una AgentInstance concreta.
- Una AgentInstance solo puede utilizar las operaciones que estén autorizadas para ella en su Workspace.
- Una autorización de un Workspace nunca amplía el catálogo definido por la Organización ni permite acceder a integraciones de otra Organización.

La capacidad de invocar APIs configuradas por el cliente no equivale a navegación libre por Internet ni habilita conocimiento externo general. El agente solo puede operar sobre integraciones y operaciones registradas y autorizadas expresamente.

**Carácter opcional:** una AgentInstance puede funcionar sin ninguna integración externa. Una empresa que no tenga ERP ni otros sistemas conectables puede crear y utilizar un agente únicamente con sus instrucciones, Knowledge y demás recursos que aporte a Waby. Las integraciones amplían capacidades cuando existe una necesidad concreta; no son un requisito de configuración, activación ni funcionamiento. La experiencia de conexión debe estar guiada y no exigir que el usuario conozca APIs o conceptos técnicos. Ver [Experiencia de configuración de agentes](./experiencia-configuracion-agentes.md).

## 2. Modelo conceptual propuesto

| Concepto | Ownership / alcance | Responsabilidad |
|---|---|---|
| Integration | Organización | Conexión lógica a un sistema externo: identificador, tipo, configuración de conexión y referencia segura a credenciales. |
| IntegrationOperation | Integración / Organización | Operación de negocio permitida: identificador estable, descripción, método y ruta internos, parámetros, esquema de entrada/salida y clasificación de riesgo. |
| AgentInstanceIntegrationGrant | Workspace + AgentInstance | Conjunto explícito de operaciones de una integración habilitadas para esa AgentInstance. |
| IntegrationInvocation | Organización + Workspace + AgentInstance + Execution | Registro auditable de cada invocación y su resultado operativo. |

Los nombres son conceptuales; no representan todavía nombres de colecciones, clases o esquemas aprobados.

### Ownership y configuración

- La conexión, las credenciales y el catálogo de operaciones son recursos de ámbito Organización. Su actor administrador concreto queda pendiente; no debe asumirse que el administrador de Organización, cuya responsabilidad de producto es gestionar el plan/entitlement, configura las AgentInstances.
- El Workspace administra las autorizaciones para sus AgentInstances mediante usuarios autorizados, sin poder editar credenciales ni ampliar el catálogo de operaciones.
- La relación de autorización debe comprobar que Integración, Workspace y AgentInstance pertenecen a la misma Organización y que la AgentInstance pertenece al Workspace indicado.
- Si una empresa necesita conectar distintos entornos o credenciales (por ejemplo, ERP de dos unidades), puede registrar integraciones separadas bajo la misma Organización. No se propone que un Workspace sobrescriba silenciosamente las credenciales de una integración compartida.

## 3. Contrato de una operación

Cada operación registrada debería declarar, como mínimo:

- Identidad estable y versión.
- Nombre y descripción orientados a la intención de negocio; instrucciones claras sobre cuándo usarla y cuándo no.
- Método HTTP y ruta relativa respecto de la conexión registrada.
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
5. Un ejecutor controlado realiza la llamada externa usando las credenciales de la Organización.
6. Waby valida, limita y normaliza la respuesta antes de devolver el resultado al pipeline/LLM.
7. La ejecución continúa o responde al usuario según el resultado y las reglas de la AgentInstance.

Las credenciales, tokens, headers sensibles y secretos nunca se incorporan al prompt, al contexto del LLM ni a los logs en claro. El agente no recibe acceso genérico a un cliente HTTP.

## 5. Seguridad y aislamiento obligatorios

- **Sin HTTP arbitrario:** el modelo no puede elegir host, esquema, puerto ni URL completa. Solo invoca operaciones registradas.
- **Control de salida:** destinos HTTPS permitidos y protección SSRF, incluyendo validación de DNS/resolución, redirecciones y direcciones privadas; las excepciones de red deben ser explícitas y controladas por Waby.
- **Secretos:** almacenamiento cifrado, referencias seguras y rotación/revocación sin exponer credenciales al agente.
- **Validación:** validar entradas antes de enviar y respuestas externas antes de incorporarlas al contexto.
- **Respuestas no confiables:** tratar los datos devueltos por la API como datos, nunca como instrucciones de sistema o cambios de política.
- **Límites:** timeout, tamaño máximo de request/response, rate limit, concurrencia, cuotas y límites de reintentos.
- **Aislamiento:** toda invocación queda vinculada a organizationId, workspaceId, agentInstanceId y executionId; no debe ser posible resolver grants o secretos de otro tenant.
- **Revocación:** deshabilitar una integración, revocar credenciales o retirar una autorización debe bloquear nuevas invocaciones. Esta revocación de seguridad prevalece sobre la continuidad de una ejecución en curso.

## 6. Versionado y consistencia con P10

La configuración efectiva inmutable por ejecución debe identificar, sin incluir secretos:

- Integraciones y operaciones habilitadas para la AgentInstance.
- Versión de la definición de cada operación y hashes de sus esquemas/configuración efectiva.
- Grants que autorizaron las operaciones en esa ejecución.
- Referencia/versionado no sensible de la configuración de conexión utilizada, según se defina el mecanismo de secretos.

Los cambios normales de catálogo, parámetros o grants se aplican a nuevas ejecuciones, en línea con P10. La revocación urgente de seguridad es una excepción deliberada: debe impedir nuevas llamadas aunque una ejecución conserve un snapshot previo. La semántica exacta de revocación y de credenciales rotadas debe cerrarse en el contrato de runtime.

## 7. Documentos y archivos devueltos por APIs

Una API puede devolver metadatos de un documento o una referencia a un archivo. No se debe enviar automáticamente cualquier URL devuelta por el proveedor externo.

La entrega de documentos requiere un flujo controlado por Waby que valide origen permitido, tipo MIME, tamaño, acceso, expiración y asociación con la ejecución/contacto. El mecanismo exacto de descarga, almacenamiento temporal y envío como medio de Waby queda pendiente de diseño técnico.

## 8. Auditoría

Cada invocación debe permitir reconstruir quién pidió qué operación, bajo qué autorización y con qué resultado operativo. Como mínimo: IDs de tenant/workspace/instancia/ejecución, integración y versión de operación, fecha, duración, estado HTTP/clasificación de error, intento y correlación. Requests y responses solo se conservan conforme a una política explícita de minimización y redacción; nunca se registran secretos en claro.

## 9. Puntos aún pendientes de contrato en WAB-78

1. Roles/permisos exactos para administrar conexiones, credenciales y el catálogo de operaciones de ámbito Organización (actor aún no definido), así como permisos de usuario autorizado del Workspace para habilitar/revocar operaciones por AgentInstance. Esta última responsabilidad sí está asignada al Workspace. Deben derivarse de la matriz real de permisos; no inventar permisos nuevos en esta propuesta.
2. Cómo se define el catálogo: formulario manual, importación OpenAPI o ambos, y qué validaciones realiza Waby antes de publicar una operación.
3. Métodos de autenticación y alcance de la primera versión (API key, Bearer estático, OAuth u otros).
4. Tratamiento de operaciones de escritura/sensibles: cuándo se exige confirmación humana o del usuario final y cómo evitar duplicaciones ante reintentos.
5. Versionado/publicación de operaciones y comportamiento de ejecuciones en curso.
6. Política detallada de retries, idempotencia, timeout, límites y circuit breaker.
7. Contrato de salida estructurada, redacción de PII y entrega de archivos.
8. Si se admiten destinos privados o redes corporativas y bajo qué mecanismo seguro de conectividad.

Estos puntos no invalidan la decisión aprobada de ownership, pero sí impiden considerar cerrado el contrato completo de integraciones.

## 10. Límites de alcance

Esta propuesta no introduce navegación libre, búsqueda web abierta ni acceso general del agente a Internet. Tampoco autoriza que clientes modifiquen el proveedor LLM contratado/administrado por Waby, ni que las integraciones accedan al código o configuración interna de Waby. Las integraciones son una capacidad explícita y gobernada para conectar los sistemas de negocio de la Organización cliente.
