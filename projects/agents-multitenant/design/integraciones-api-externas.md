# Integraciones API externas por AgentInstance — diseño de WAB-78

**Estado:** diseño conceptual de conexiones, credenciales y operaciones por AgentInstance administradas por usuarios autorizados del Workspace; plan comercial de agentes IA administrado a nivel Organización. El ownership se reconcilia con D-W79-08; los contratos técnicos y de runtime siguen pendientes de definición en el alcance posterior correspondiente.
**Fecha:** 05/11/2026.  
**Alcance:** capacidad para que una AgentInstance consulte o ejecute operaciones explícitamente configuradas contra APIs de sistemas de la Organización cliente. No define todavía el modelo persistente final ni autoriza implementación.

**Reconciliación puntual con WAB-79 D-W79-08 (09/11/2026):** se confirma que las credenciales LLM y las credenciales empresariales pertenecen a dominios distintos. Waby administra las cuentas/API keys LLM y las utiliza mediante el routing asociado a `AgentDefinition`; el cliente no aporta API keys LLM. Los usuarios autorizados del Workspace administran las credenciales de sus integraciones empresariales, mientras que la plataforma Waby opera su custodia segura y resolución. `AgentInstanceIntegration` es la ubicación conceptual de la referencia protegida a la credencial; `SkillConfiguration` no almacena credenciales ni referencias directas a ellas. Esta reconciliación no selecciona almacenamiento físico, cifrado, KMS/Vault ni esquema persistente.

**Verificación de alcance con D-W79-12, D-W79-13 y D-W79-14 (10/11/2026):** las decisiones sobre ciclo de vida de `AgentRules`, observabilidad de ejecuciones y contrato de `EvAssigned` no modifican el ownership ni el modelo conceptual de conexiones, credenciales y operaciones de integración definido en este documento.

## 1. Decisión de ownership y autoridad

- La **configuración de conexión externa se asocia a una AgentInstance concreta** y queda dentro del Workspace al que pertenece esa instancia.
- Un **usuario autorizado del Workspace** configura la conexión, endpoint/base URL permitida, método de autenticación, valores de las credenciales empresariales y operaciones que el agente podrá utilizar. También puede probar, modificar, deshabilitar o revocar esa configuración conforme a sus permisos.
- La credencial empresarial es administrada por el Workspace; la custodia técnica y resolución del secreto corresponden a la plataforma Waby. El secreto en claro no se expone en la configuración ordinaria: Waby mantiene una referencia protegida vinculada a Organization + Workspace + AgentInstance. El mecanismo físico de custodia, almacenamiento y cifrado queda pendiente de diseño técnico; no se presupone un producto concreto, KMS o Vault.
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
| AgentInstanceIntegration | Workspace + AgentInstance | Configuración de conexión privada por defecto para esa instancia: tipo de conector, destino permitido, parámetros no secretos y única referencia protegida a la credencial empresarial asociada. No contiene el secreto ni presupone una colección/clase física aprobada. |
| AgentInstanceOperation | AgentInstance | Operación concreta disponible para el agente: identificador, intención, método/ruta relativa, parámetros, schemas de entrada/salida, efecto y controles. |
| IntegrationInvocation | Organización + Workspace + AgentInstance + Execution | Registro auditable de cada invocación y su resultado operativo, con la cadena de ownership validada. |

Los nombres son conceptuales; no representan todavía nombres de colecciones, clases o esquemas aprobados.

### Ownership y configuración

- La configuración de conexión, la referencia protegida a credenciales y las operaciones pertenecen al ámbito de la AgentInstance/Workspace. Un usuario autorizado del Workspace administra los valores y la conexión para esa instancia; por defecto no se comparte entre AgentInstances.
- `AgentInstanceIntegration` es la única ubicación conceptual de la referencia protegida a la credencial de esa conexión. `SkillConfiguration` no almacena secretos ni referencias directas a credenciales; la asociación Skill–operación/capacidad es un contrato operativo separado y no duplica la referencia de conexión.
- Waby administra los ConnectorDefinitions, la custodia segura de secretos y los ejecutores. Las operaciones configuradas por el cliente deben ajustarse a esos mecanismos y validaciones; no se admite código arbitrario ni solicitudes HTTP construidas libremente por el LLM.
- Waby custodia los secretos mediante un mecanismo seguro cuya tecnología y ubicación física quedan pendientes de diseño. La AgentInstance conserva solo referencias protegidas y metadata no secreta; los secretos nunca se incorporan al prompt, al contexto del LLM ni a logs en claro.
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
5. Antes de cada invocación, un ejecutor controlado revalida ownership, autorización, estado de conexión y vigencia de la credencial; resuelve el secreto desde el mecanismo seguro de la plataforma y lo utiliza únicamente en el límite de ejecución de la integración.
6. El secreto, los tokens y los headers sensibles nunca se entregan al LLM ni se incorporan al prompt, contexto o logs. Waby valida, limita y normaliza la respuesta antes de devolver el resultado al pipeline/LLM.
7. La ejecución continúa o responde al usuario según el resultado y las reglas de la AgentInstance.

Las credenciales, tokens, headers sensibles y secretos nunca se incorporan al prompt, al contexto del LLM ni a los logs en claro. El agente no recibe acceso genérico a un cliente HTTP.

## 5. Seguridad y aislamiento obligatorios

- **Sin HTTP arbitrario:** el modelo no puede elegir host, esquema, puerto ni URL completa. Solo invoca operaciones registradas.
- **Control de salida:** el usuario autorizado del Workspace configura el destino de la conexión; Waby valida destinos HTTPS, DNS/resolución, redirecciones y direcciones privadas para proteger contra SSRF. Las excepciones de red deben ser explícitas y controladas por Waby. El LLM nunca puede cambiar el host/destino.
- **Secretos:** custodia protegida, referencias seguras y rotación/revocación sin exponer credenciales al agente. El almacenamiento físico y el mecanismo criptográfico quedan pendientes de diseño técnico.
- **Validación:** validar entradas antes de enviar y respuestas externas antes de incorporarlas al contexto.
- **Respuestas no confiables:** tratar los datos devueltos por la API como datos, nunca como instrucciones de sistema o cambios de política.
- **Límites:** timeout, tamaño máximo de request/response, rate limit, concurrencia, cuotas y límites de reintentos.
- **Aislamiento:** toda invocación queda vinculada a organizationId, workspaceId, agentInstanceId y executionId; no debe ser posible resolver grants o secretos de otro tenant.
- **Revocación:** deshabilitar una integración, revocar credenciales o retirar una autorización debe bloquear nuevas invocaciones. Esta revocación de seguridad prevalece sobre la continuidad de una ejecución en curso.

## 6. Versionado y consistencia con P10

La configuración efectiva inmutable por ejecución debe identificar, sin incluir secretos:

- Identidad y versión de la configuración de conexión de la AgentInstance.
- Identificador del conector y de las operaciones habilitadas, sus versiones y hashes de schemas/configuración efectiva.
- Identificador de la referencia segura de credencial mantenida por `AgentInstanceIntegration` (nunca el secreto) y versión/rotación aplicable.
- Autorización vigente de la AgentInstance que permitió la operación, vinculada a Organization, Workspace, AgentInstance y Execution.

Los cambios normales de catálogo, parámetros o grants se aplican a nuevas ejecuciones, en línea con P10. La revocación urgente de seguridad es una excepción deliberada: debe impedir nuevas llamadas aunque una ejecución conserve un snapshot previo. La semántica exacta de revocación y de credenciales rotadas debe cerrarse en el contrato de runtime.

## 7. Documentos y archivos devueltos por APIs

Una API puede devolver metadatos de un documento o una referencia a un archivo. No se debe enviar automáticamente cualquier URL devuelta por el proveedor externo.

La entrega de documentos requiere un flujo controlado por Waby que valide origen permitido, tipo MIME, tamaño, acceso, expiración y asociación con la ejecución/contacto. El mecanismo exacto de descarga, almacenamiento temporal y envío como medio de Waby queda pendiente de diseño técnico.

## 8. Auditoría

Cada invocación debe permitir reconstruir quién pidió qué operación, bajo qué autorización y con qué resultado operativo. Como mínimo: IDs de tenant/workspace/instancia/ejecución, integración y versión de operación, fecha, duración, estado HTTP/clasificación de error, intento y correlación. Requests y responses solo se conservan conforme a una política explícita de minimización y redacción; nunca se registran secretos en claro.

## 9. Puntos pendientes de diseño técnico posterior

1. **Autorización administrativa de la configuración:** el permiso de administración de AgentInstance quedó definido en ARQ-20: `wrk.agents.*` (Owner por herencia de `wrk.*.*`; Manager explícito), con `wrk.agents.read` para consulta. Incluye administrar conexiones, credenciales y operaciones asociadas a la instancia. La autorización backend de cada invocación/Capability y de cada efecto externo sigue siendo contextual e independiente; `wrk.agents.*` no la sustituye. La incorporación al enum, guards, ownership y pruebas corresponden a WAB-80.
2. Cómo se definen las operaciones de cada AgentInstance: selección de operaciones/plantillas soportadas por Waby, configuración manual, importación OpenAPI o combinación; qué validaciones realiza Waby antes de habilitarlas.
3. Métodos de autenticación y alcance de la primera versión (API key, Bearer estático, OAuth u otros).
4. Tratamiento de operaciones de escritura/sensibles: cuándo se exige confirmación humana o del usuario final y cómo evitar duplicaciones ante reintentos.
5. Versionado/publicación de operaciones y comportamiento de ejecuciones en curso.
6. Si se permite compartir una conexión entre varias AgentInstances del mismo Workspace en una etapa posterior; no forma parte del supuesto inicial, que es conexión privada por instancia.
7. Política detallada de retries, idempotencia, timeout, límites y circuit breaker.
8. Contrato de salida estructurada, redacción de PII y entrega de archivos.
9. Si se admiten destinos privados o redes corporativas y bajo qué mecanismo seguro de conectividad.
10. Diseño técnico del mecanismo de custodia: almacenamiento físico, cifrado, gestión/rotación de claves, resolución y límites entre servicios. D-W79-08 fija ownership y límites de exposición, no selecciona CRM DB, AI Service DB, Vault/KMS ni una implementación criptográfica.

Estos puntos no invalidan la decisión aprobada de ownership de D-W79-08. Son contratos técnicos y operativos pendientes para las fases posteriores; no reabren WAB-78 ni modifican la separación de responsabilidades ya aprobada.

## 10. Límites de alcance

Esta propuesta no introduce navegación libre, búsqueda web abierta ni acceso general del agente a Internet. Tampoco autoriza que clientes modifiquen el proveedor LLM contratado/administrado por Waby, ni que las integraciones accedan al código o configuración interna de Waby. Las integraciones son una capacidad explícita y gobernada para conectar los sistemas de negocio de la Organización cliente.


## 11. Implementación física WAB-100 — primera versión acotada (20/11/2026)

La inspección e implementación local confirma el boundary dentro de `crm/` (backend principal) y `crm/microservices/ai-service` (AgentInstance Runtime). No se agrega repo ni microservicio. `infrastructure/` sigue siendo proveedor de dependencias operativas, no owner del executor.

### Connector Catalog allowlisted

CRM implementa `AgentIntegrationConnectorCatalogService` como catálogo Waby allowlisted versionado. La primera definición es `http-v1` / `1.0.0`, HTTPS público, auth `none|bearer|api-key`, GET-only, redirects deshabilitados y custodia de credenciales CRM AES-256-GCM. Connector/version desconocido falla cerrado. No existe HTTP arbitrario, OpenAPI import ni código/headers definidos por el LLM.

### Configuración y operaciones

REST CRM (autorización Workspace `wrk.agents.read` / `wrk.agents.*`) permite administrar integración y operaciones:

- `POST/GET /api/agent-instances/:agentInstanceId/integrations`
- `PATCH /api/agent-instances/:agentInstanceId/integrations/:integrationId`
- `POST/GET /api/agent-instances/:agentInstanceId/integrations/:integrationId/operations`
- `PATCH /api/agent-instances/:agentInstanceId/integrations/:integrationId/operations/:operationRecordId`

Base URL debe ser origin HTTPS port 443, sin userinfo/path/query/hash ni IP literal/local hostname. Secrets se aceptan por body de configuración/rotación pero no se retornan, no se almacenan en `connectionConfig` ni operationConfig; se cifran con AES-256-GCM en `integration_credential_secrets`, con ref opaca asociada a Organization+Workspace+AgentInstance+Integration. Requiere `CRM_AGENT_INTEGRATION_CREDENTIAL_ENCRYPTION_KEY` (base64 de 32 bytes); el key no debe ser versionado. Credential rotation cambia ref/revision y deshabilita integración hasta reactivación; revocation es terminal.

Operation IDs/versiones son inmutables como binding/config; schema object requiere `additionalProperties=false`; path relativo y placeholders declarados; método GET-only; efecto READ-only en runtime. Operaciones WRITE/SENSITIVE no se publican ni ejecutan hasta existir un contrato de confirmación humana/idempotencia.

### Snapshot → tool → CurrentAuthorization → effect

CRM `AgentRuntimeSnapshotService` incorpora operaciones read habilitadas de integraciones ACTIVE `http-v1` al snapshot (metadata/schema sin baseUrl/credential). El snapshot/hash cubre integración revision, operation version/schema/name/effect. AI Service `AgentInstanceExecutionRuntime` genera capabilities dinámicas por operationId, solo cuando `policy.allowedCapabilities` autoriza exactamente esa capability. El policy resolver sigue fail-closed y no se auto-grantean capabilities.

Binding resolver direccional AI Service→CRM (`agent.integration.binding.resolve`) obtiene integration/connector/version/revision/operation binding desde persistencia CRM; el LLM no suministra IDs de integración, connector ni revision como evidencia de autoridad. `RegistryAgentRuntimeCapabilityExecutor` revalida CurrentAuthorization live antes del efecto. CRM internal execution endpoint (`agent.integration.execute`) vuelve a validar tenancy/binding y CurrentAuthorization justo antes de resolver credential y realizar GET. Credential plaintext nunca sale CRM hacia AI Service/LLM.

### Seguridad de salida y audit

- HTTPS origin only, port 443, public DNS/IP only; se rechazan local/private/link-local/reserved, redirects y hosts IP literales.
- DNS se valida y la IP pública resuelta se pinnea en HTTPS request para evitar DNS rebinding entre resolución y conexión.
- Param schemas, path placeholders y query scalar values se validan; host/path destination no provienen de tool arguments.
- Timeout 10 s; response max 64 KiB; no custom headers salvo auth header catalogado; redacción de credential echoes.
- `IntegrationInvocation` registra execution/integration/operation/version/status/duration/httpStatus/errorCode/correlation; no request/response ni secretos. Audit write failure queda en log de error y se debe tratar como alerta operativa.

### Limitaciones/bloqueos explícitos de esta primera implementación

- Runtime sólo READ/GET. WRITE/SENSITIVE se deniega `INTEGRATION_WRITE_CONFIRMATION_REQUIRED`; falta contrato de human confirmation, idempotency y retry.
- `FailClosedAgentRuntimePolicyResolver`/authority source aún debe ser configurado para permitir capabilities; no se auto-grant por tener operación registrada. Sin policy allow no se expone tool.
- El key de custodia está generado para el `.env` local ignorado; producción debe provisionar key de 32 bytes, backup/rotation plan y env. No ejecutar deployment.
- No se implementa OpenAPI import, connector catalog UI, tests E2E contra un proveedor real, OAuth, custom/private networking ni entrega de documentos.
- No se implementó frontend de administración en `WABY_frontend`; los REST endpoints quedan como boundary backend.

Esta implementación concreta es un primer executor versionado, allowlisted y read-only; no reabre ownership/contratos WAB-78/79 ni autoriza ampliar capabilities.


### Audit append-only antes/después del efecto

CRM registra un evento `REQUESTED` antes de la evaluación live/credential resolution/effect. Si no se puede persistir el preflight audit, el executor no realiza el efecto. Después registra `SUCCEEDED`, `FAILED` o `DENIED` con el mismo correlationId; ambos eventos son append-only y no contienen payload ni secret. Si falla el post-audit se emite log operacional (el efecto remoto puede ya haber ocurrido, por lo que requiere alerting).
