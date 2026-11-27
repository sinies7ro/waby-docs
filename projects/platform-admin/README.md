# WABY Platform Admin — Diseño arquitectónico y plan de construcción

> **Estado:** diseño funcional y arquitectónico aprobado; proyecto y backlog formalizados en Linear.  
> **Tipo:** construcción nueva e independiente.  
> **Fecha de consolidación:** 2026-11-26.  
> **Fuente documental:** `sinies7ro/waby-docs`.

## 1. Propósito y alcance

WABY Platform Admin es una nueva plataforma administrativa interna para que WABY gestione la identidad de sus administradores, el catálogo de servicios de la plataforma y los accesos administrativos a esos servicios.

No es una funcionalidad del sistema de agentes multitenant, no es una extensión del CRM y no forma parte del frontend utilizado por los clientes de WABY. Tiene identidad, backend, frontend, persistencia, ciclo de despliegue y planificación propios.

El objetivo de este documento es consolidar el diseño aprobado antes de iniciar la implementación. No constituye autorización para crear repositorios, desplegar servicios o modificar los sistemas existentes.

### 1.1. Componentes previstos

| Componente | Responsabilidad | Tecnología acordada |
|---|---|---|
| `fr-platform-waby` | Interfaz administrativa de la plataforma | React |
| `bk-platform-waby` | Identidad, autenticación, autorización global, catálogo, coordinación y auditoría | Node.js, TypeScript, Express, Docker |
| Persistencia central | Administradores, servicios, grants, sesiones y auditoría | MongoDB |
| ODM | Modelado y acceso a MongoDB | Mongoose + Typegoose |
| Mensajería | Eventos, sincronización y reintentos | RabbitMQ |

La selección de GraphQL es opcional por operación: REST/Express será el mecanismo habitual para operaciones administrativas síncronas. GraphQL podrá utilizarse para consultas de frontend cuando aporte valor. RabbitMQ se reserva para procesos asíncronos y propagación de eventos; no es obligatorio en todos los flujos.

### 1.2. Independencia respecto del ecosistema existente

- `waby-docs` conserva el diseño transversal y los contratos de esta construcción.
- Linear tendrá un proyecto independiente para su planificación y seguimiento.
- `fr-platform-waby` y `bk-platform-waby` serán repositorios independientes previstos.
- `ElemeSoft/crm`, `ElemeSoft/WABY_frontend` e `infrastructure` no se convierten en el lugar de implementación de Platform Admin.
- CRM y los demás servicios solo incorporarán la integración local necesaria para aceptar y autorizar administradores de plataforma, mediante contratos explícitos.
- No se altera el dominio de autenticación/autorización de clientes, organizaciones y workspaces del CRM.
- Esta construcción no bloquea ni modifica el trabajo de AgentInstance Runtime (WAB-81) ni los trabajos de agentes multitenant.

## 2. Principios arquitectónicos aprobados

1. **Identidad central:** la identidad de los administradores de plataforma vive exclusivamente en Platform Backend.
2. **Autorización distribuida por propiedad:** Platform Backend administra identidad y grants centrales; cada servicio es dueño de sus roles, permisos y validación local.
3. **Separación de dominios:** los administradores de plataforma no son usuarios CRM ni heredan roles de organizaciones/workspaces.
4. **Frontend desacoplado:** Platform Frontend se comunica únicamente con Platform Backend. El backend coordina las operaciones hacia los servicios.
5. **JWT sin permisos de servicio:** el token humano identifica al administrador y la sesión; no transporta roles ni permisos internos de los servicios.
6. **Comunicación híbrida:** REST para operaciones síncronas y RabbitMQ para eventos, reintentos y reconciliación.
7. **Auditoría central:** Platform Backend mantiene el registro oficial de actividad administrativa.
8. **Sin registro público:** las cuentas se crean mediante operaciones autorizadas o el bootstrap inicial.
9. **Independencia de despliegue:** frontend y backend administrativos tienen ciclos de despliegue propios.
10. **Fail-closed:** un grant pendiente o fallido no habilita acceso; los controles de identidad, sesión y autorización no deben degradarse a acceso permisivo.
11. **Backend contenerizado:** `bk-platform-waby` se construye y ejecuta como contenedor Docker; la imagen es un artefacto de despliegue independiente.

## 3. Identidad y ciclo de vida de administradores

### 3.1. Colección central `platform_admins`

Modelo conceptual:

| Campo | Descripción |
|---|---|
| `_id` | Identificador del administrador |
| `email` | Email normalizado y único |
| `passwordHash` | Hash de contraseña |
| `status` | `active` o `deactivated` |
| `mustChangePassword` | Obliga al cambio en el primer acceso cuando corresponda |
| `passwordChangedAt` | Fecha del último cambio de contraseña |
| `createdAt`, `updatedAt` | Trazabilidad temporal |

Los únicos estados de identidad definidos son `active` y `deactivated`. No se agregan estados `suspended` o `locked` al administrador. La protección contra fuerza bruta se implementa mediante controles de autenticación temporales, no mediante un estado permanente adicional.

La desactivación es lógica; no se elimina físicamente la identidad. Desactivar una cuenta revoca sus sesiones y sus accesos. Reactivarla no restaura grants revocados.

### 3.2. Niveles centrales de administración

Se aprueban dos niveles globales, independientes de los roles que cada servicio define:

- **Platform SuperAdmin:** autoridad global para administrar la plataforma, crear y administrar administradores y gestionar grants de servicios.
- **Platform Admin:** administrador sin autoridad global; sus capacidades se limitan a las atribuciones centrales que se le concedan.

El nivel global no debe confundirse con los roles locales de CRM, ai-service u otros servicios. Ser Platform Admin no concede automáticamente acceso a todos los servicios.

La política concreta de permisos globales de Platform Admin debe especificarse en el contrato de autorización central antes de implementar las operaciones correspondientes. Platform SuperAdmin conserva la autoridad global prevista.

### 3.3. Operaciones de administración

- Crear y consultar administradores.
- Actualizar datos permitidos, como email.
- Desactivar y reactivar administradores.
- Conceder, revocar y cambiar roles de acceso a servicios.
- Consultar los servicios y grants asociados a cada administrador.

No existe registro público. Las operaciones requieren autorización central y deben generar auditoría.

### 3.4. Protecciones de cuentas críticas

- Un administrador no puede desactivar su propia cuenta.
- No se puede desactivar al último Platform SuperAdmin activo.
- No se permite eliminación física ordinaria.
- La desactivación revoca todas las sesiones y todos los accesos del administrador.
- La reactivación no restaura automáticamente accesos anteriores.
- La creación o elevación a Platform SuperAdmin es explícita y auditada.

## 4. Autenticación humana, JWT y claves

### 4.1. Emisión y validación

Platform Backend es el único emisor de JWT para administradores de plataforma.

Se aprueba:

- Firma asimétrica **RS256**.
- Publicación de claves públicas mediante **JWKS** en `/.well-known/jwks.json`.
- Clave privada exclusivamente en Platform Backend.
- Los servicios consumen y cachean claves públicas.
- Rotación de claves mediante `kid`.
- Validación estricta de algoritmos permitidos.

Claims del Access Token:

| Claim | Contenido |
|---|---|
| `iss` | Issuer de Platform Backend |
| `sub` | `platformAdminId` |
| `aud` | Servicio destinatario |
| `iat` | Emisión |
| `exp` | Expiración |
| `jti` | Identificador único del JWT |
| `sid` | Identificador de sesión |
| `principal_type` | `platform_admin` |

El header incluye `alg: RS256` y `kid`.

Los servicios validan firma, issuer, audience, expiración y `principal_type`. No se incluyen `roleKey`, permisos ni catálogos de roles en el JWT.

### 4.2. Separación respecto de JWT de clientes

Los servicios pueden reutilizar infraestructura criptográfica genérica de validación JWT, pero deben mantener separados:

- Identidad y claims aceptados.
- Dominio de autorización.
- Roles y permisos.
- Middlewares y reglas de acceso.
- Recursos y ciclo de vida.

En CRM, el dominio administrativo de plataforma será independiente del dominio existente de usuarios, organizaciones y workspaces. Los permisos existentes, por ejemplo `wrk.*`, no se reutilizan ni se modifican como consecuencia de este diseño.

## 5. Sesiones, refresh y revocación

### 5.1. Colección central `platform_admin_sessions`

Modelo conceptual:

| Campo | Descripción |
|---|---|
| `_id` | Identificador de sesión; se corresponde con `sid` |
| `platformAdminId` | Administrador propietario |
| `refreshTokenHash` | Hash del refresh token vigente |
| `tokenFamilyId` | Identificador de la familia de rotación |
| `status` | `active` o `revoked` |
| `createdAt` | Creación |
| `lastUsedAt` | Último uso |
| `expiresAt` | Expiración |
| `revokedAt` | Fecha de revocación |
| `revocationReason` | Motivo |

### 5.2. Duraciones y transporte

- Access Token: **10 minutos**.
- Refresh Token: **7 días**.
- Refresh Token opaco, aleatorio y almacenado únicamente como hash.
- Transporte del Refresh Token mediante cookie `HttpOnly` y `Secure`, con política `SameSite` adecuada.
- El endpoint de refresh debe incorporar protección CSRF.
- Los detalles finales de dominio, SameSite y origen se concretan al definir los dominios de despliegue.

### 5.3. Rotación y detección de reutilización

Cada renovación rota el Refresh Token e invalida el anterior. Si se detecta reutilización de un token anterior, se revoca únicamente la sesión o familia afectada; no se revocan automáticamente todas las sesiones del administrador.

Cada dispositivo/navegador genera una sesión independiente. Deben contemplarse listado de sesiones activas, cierre de sesión actual y cierre de todas las sesiones.

- Logout: revoca la sesión actual.
- Logout global: revoca todas las sesiones del administrador.
- Desactivación: revoca todas las sesiones y grants del administrador.

### 5.4. Validación local y disponibilidad

Cada servicio valida localmente el estado administrativo y el permiso requerido en cada solicitud. También debe aplicar la revocación de sesión cuando corresponda.

Las revocaciones se propagan por REST y RabbitMQ. Los servicios mantienen únicamente el estado operativo mínimo necesario para aplicar revocaciones; no replican la colección central de sesiones ni almacenan refresh tokens.

Durante una caída de Platform Backend:

- Los servicios pueden seguir validando JWT no expirados y la autorización local vigente.
- No se pueden renovar sesiones.
- Un acceso ya revocado localmente continúa bloqueado.
- La reconciliación recupera eventos de revocación que no se hayan recibido.

La expiración del JWT es defensa en profundidad, no el mecanismo principal de revocación. El mecanismo local exacto para mantener revocaciones por `sid` y el objetivo temporal máximo de propagación deben quedar especificados en el contrato operativo antes de implementar.

## 6. Catálogo de servicios y grants

### 6.1. Catálogo central `platform_services`

Platform Backend mantiene el catálogo central de servicios registrados. Ejemplos iniciales: `crm`, `ai-service` e `infrastructure`. La lista definitiva se determinará al registrar los servicios que realmente participen.

Un servicio deshabilitado centralmente no admite nuevas concesiones. El comportamiento exacto sobre grants existentes al deshabilitar un servicio queda pendiente de definición explícita.

### 6.2. Grants centrales `platform_admin_service_grants`

Modelo conceptual:

| Campo | Descripción |
|---|---|
| `platformAdminId` | Administrador destinatario |
| `serviceId` | Servicio |
| `roleKey` | Clave estable del rol propiedad del servicio |
| `status` | `active` o `revoked` |
| `provisioningStatus` | `pending`, `synced` o `failed` |
| `grantedAt` | Fecha de concesión |
| `grantedBy` | Administrador que concede |
| `lastSyncedAt` | Última sincronización confirmada |

Índice único compuesto: `platformAdminId + serviceId`.

El grant central expresa el acceso deseado; el registro local del servicio expresa el acceso efectivo. Un grant en estado `pending` o `failed` no habilita acceso.

### 6.3. Propiedad de roles y permisos

Cada servicio es dueño de sus roles y permisos. Inicialmente son estáticos en código y versionados junto con el servicio; no se introduce un catálogo dinámico de roles en MongoDB.

Platform Backend puede consultar el catálogo y asignar un `roleKey`, pero no puede crear, modificar ni eliminar definiciones de roles o permisos. No interpreta el conjunto de permisos internos del servicio.

Cada servicio define exactamente un rol administrativo máximo, identificado por `isPlatformAdminRole: true`. Platform SuperAdmin recibe ese rol máximo en cada servicio activo registrado, mediante operaciones de asignación explícitas y auditadas.

## 7. Contrato de integración con servicios

### 7.1. Endpoints internos

Cada servicio que participe implementa el contrato común:

| Método y ruta | Responsabilidad |
|---|---|
| `GET /internal/platform-admin/catalog` | Catálogo y versión del servicio |
| `GET /internal/platform-admin/roles` | Roles disponibles |
| `GET /internal/platform-admin/permissions` | Permisos disponibles |
| `POST /internal/platform-admin/access/grant` | Conceder o actualizar acceso local |
| `POST /internal/platform-admin/access/revoke` | Revocar acceso local |
| `GET /internal/platform-admin/access/status` | Consultar estado efectivo local |

Los endpoints de catálogo forman parte del diseño aprobado en la prioridad de autenticación y catálogo. Los endpoints de grants completan el contrato de sincronización. Los esquemas de request/response, errores y versionado se especificarán en el contrato de integración antes de implementar.

### 7.2. Comunicación REST + RabbitMQ

- REST para operaciones síncronas que requieren confirmación inmediata: concesión, revocación y consulta.
- RabbitMQ para notificaciones, propagación de eventos, reintentos, recuperación y sincronización asíncrona.
- No se fuerza el uso de RabbitMQ en operaciones que no lo necesitan.
- Platform Frontend no llama directamente a los servicios; Platform Backend coordina todas las operaciones.

### 7.3. Identidad técnica servicio a servicio

Las llamadas internas usan una identidad técnica separada de la identidad humana.

Se aprueba JWT técnico firmado con RS256, con identidad del emisor, audiencia específica del servicio destinatario, expiración corta y permisos técnicos mínimos. No se reutiliza el JWT humano del administrador para llamadas internas.

El contrato de credenciales técnicas, rotación y distribución de claves se detalla durante el diseño de integración y despliegue.

### 7.4. Consistencia y resiliencia

Se aprueban los siguientes mecanismos:

- Idempotencia en concesiones, revocaciones y consumidores de eventos.
- Identificador de correlación por operación.
- Identificador único por evento para evitar procesamiento duplicado.
- Reintentos con backoff.
- Dead Letter Queue (DLQ) para eventos que agoten reintentos.
- Reconciliación periódica entre estado central deseado y estado efectivo local.
- Revocaciones con prioridad sobre concesiones cuando las operaciones llegan fuera de orden.

La reconciliación compara el estado deseado central con el estado local; no se limita a comprobar existencia de registros.

### 7.5. Servicio no disponible

Si un servicio está desconectado, el backend central conserva el estado de aprovisionamiento correspondiente y no informa que el acceso está concedido sin confirmación local. Se reintenta la operación y la reconciliación recupera diferencias cuando el servicio vuelve a estar disponible.

## 8. Bootstrap inicial y recuperación de SuperAdmin

### 8.1. Bootstrap idempotente

Se permite un único bootstrap inicial, únicamente cuando no existe ningún SuperAdmin.

Variables de entorno previstas:

- `PLATFORM_BOOTSTRAP_ADMIN_EMAIL`
- `PLATFORM_BOOTSTRAP_ADMIN_PASSWORD`

Los valores se suministran mediante secretos de despliegue; nunca se versionan ni se incluyen credenciales por defecto. La contraseña inicial es aleatoria y debe cambiarse en el primer acceso (`mustChangePassword: true`).

El bootstrap debe ser atómico/idempotente para evitar administradores duplicados ante ejecuciones concurrentes. En despliegues posteriores no crea ni modifica cuentas. Si el SuperAdmin existente está desactivado, no se crea un reemplazo automáticamente.

La inicialización debe conceder al primer SuperAdmin el rol administrativo máximo de cada servicio activo registrado y rastrear la sincronización de esos grants. La expiración o eliminación segura del secreto de bootstrap después de inicializar queda definida como requisito operativo de implementación.

### 8.2. Recuperación de contraseña

Flujo aprobado:

1. El administrador solicita recuperar su contraseña.
2. Platform Backend genera un token aleatorio de un solo uso.
3. Se almacena únicamente el hash del token.
4. Se envía un enlace al email registrado.
5. El administrador establece una contraseña nueva.
6. El token se invalida.
7. Se revocan las sesiones existentes del administrador.

Expiración del token: **15 minutos**. No se envían contraseñas temporales por correo.

### 8.3. Recuperación de emergencia del único SuperAdmin

No existe recreación automática ni registro público alternativo. La recuperación del único SuperAdmin se realiza mediante un procedimiento explícito, autorizado y controlado desde la infraestructura. Debe documentarse, protegerse y dejar constancia en la auditoría central cuando sea posible.

## 9. Política de contraseñas y protección de autenticación

Se aprueba:

- Hash de contraseñas con **Argon2id**.
- Longitud mínima de 12 caracteres.
- Aceptar contraseñas largas.
- No imponer reglas artificiales de composición de caracteres.
- Comprobar contraseñas contra listas de contraseñas comprometidas cuando sea viable.
- Limitar intentos de autenticación y aplicar protección contra fuerza bruta.
- No bloquear permanentemente cuentas ni introducir un estado permanente `locked`.

Los parámetros concretos de Argon2id y los umbrales de rate limiting se definirán con la configuración de seguridad de producción.

## 10. Auditoría administrativa

### 10.1. Fuente oficial

La única fuente oficial de auditoría administrativa es la colección central `platform_admin_audit_logs`, mantenida exclusivamente por Platform Backend.

Campos conceptuales:

| Campo | Descripción |
|---|---|
| `actorAdminId` | Administrador que ejecutó la acción |
| `action` | Acción realizada |
| `targetType` | Tipo de recurso afectado |
| `targetId` | Identificador del recurso |
| `serviceId` | Servicio involucrado, si corresponde |
| `result` | Resultado de la operación |
| `metadata` | Detalles no sensibles |
| `ip` | Dirección IP de origen |
| `userAgent` | Información del cliente |
| `createdAt` | Fecha del registro |

### 10.2. Reglas

- Comportamiento append-only para operaciones ordinarias: no se modifican ni eliminan registros.
- Registrar operaciones administrativas relevantes exitosas y fallidas.
- Incluir cambios de identidad, roles, grants, sesiones, contraseñas, bootstrap y recuperación de emergencia.
- Nunca registrar contraseñas, tokens, claves privadas ni secretos.
- La retención inicial aprobada es de **12 meses**, configurable y sujeta a revisión antes de producción.

Los servicios no mantienen colecciones duplicadas de auditoría administrativa.

### 10.3. Separación de registros

| Tipo | Ubicación | Propósito |
|---|---|---|
| Auditoría administrativa | Platform Backend | Historial oficial de acciones |
| Logs operativos | Cada servicio | Diagnóstico y funcionamiento |
| Estado de autorización | Cada servicio | Accesos efectivos y revocaciones |

Un servicio puede registrar en sus logs operativos el rechazo de una solicitud por sesión revocada, pero eso no crea una segunda auditoría administrativa.

## 11. Despliegue e infraestructura

### 11.1. Separación de despliegues

Frontend y backend se despliegan de manera independiente, con ciclos de vida propios y sin dependencia operativa del despliegue de CRM o del frontend de clientes.

### 11.2. Requisitos de seguridad y operación

- HTTPS obligatorio.
- Claves privadas RS256 y demás secretos exclusivamente en mecanismos de secretos del entorno.
- Nunca incluir secretos en Git, archivos de ejemplo con valores reales ni imágenes Docker.
- CORS limitado a orígenes autorizados.
- Configuración separada para desarrollo, staging y producción.
- Health checks para backend y dependencias.
- MongoDB autenticado, restringido y respaldado.
- RabbitMQ con credenciales y permisos específicos.
- Logs operativos sin información sensible.
- Rotación documentada de claves y secretos.
- Persistencia y recuperación de datos verificables.

### 11.3. Topología lógica

- Frontend administrativo: dominio propio.
- Backend administrativo: dominio propio o subdominio de API.
- MongoDB y RabbitMQ: infraestructura privada, sin exposición pública innecesaria.
- Servicios: comunicación interna autenticada.
- Los dominios definitivos y la topología concreta quedan pendientes del diseño de despliegue.

### 11.4. Dockerización de Platform Backend

`bk-platform-waby` debe estar dockerizado desde su fundación y ejecutarse como contenedor Docker en los entornos previstos. Docker forma parte del modo de empaquetado y despliegue del backend, no es una mejora opcional posterior.

Requisitos mínimos:

- Mantener un `Dockerfile` versionado en el repositorio del backend.
- Construir una imagen reproducible, preferentemente mediante build multi-stage, separando dependencias de compilación y runtime.
- Ejecutar el proceso con un usuario no privilegiado.
- No copiar secretos, archivos `.env` reales, credenciales ni claves privadas dentro de la imagen.
- Inyectar configuración y secretos en tiempo de ejecución mediante el mecanismo de secretos/configuración del entorno.
- Definir correctamente el comando de arranque y el manejo de señales para cierre ordenado.
- Incorporar health check del servicio y documentar cómo se comprueba la disponibilidad.
- Mantener `.dockerignore` para excluir dependencias locales, artefactos, archivos sensibles y contenido innecesario.
- Documentar build, ejecución local y configuración de los contenedores por entorno.
- La imagen debe poder desplegarse independientemente de CRM y del frontend administrativo.

La estrategia concreta de orquestación y el uso de Docker Compose en desarrollo se definirán según la topología de infraestructura; este requisito no presupone que Compose sea el orquestador de producción.

## 12. Construcción y etapas de trabajo

El diseño se considera cerrado a nivel de prioridades. La implementación debe planificarse como una construcción nueva, con entregables verificables y dependencias explícitas.

Secuencia de trabajo recomendada para el proyecto independiente:

1. **Formalización del proyecto:** proyecto independiente creado en Linear; alcance, decisiones aprobadas, responsable, dependencias y roadmap registrados. No crear un issue hijo de WAB-78/WAB-81.
2. **Contratos detallados:** cerrar esquemas de API, errores, versionado de catálogos, autorización central, eventos, idempotencia y protocolo técnico entre servicios.
3. **Fundación de Platform Backend:** repositorio y aplicación, Dockerfile e imagen Docker, configuración, MongoDB, bootstrap, identidad, contraseñas, JWT/JWKS y sesiones.
4. **Autorización y administración:** roles centrales, administración de administradores, grants y auditoría.
5. **Integración de servicios:** contrato interno, validación local, aprovisionamiento, revocación, RabbitMQ, reintentos y reconciliación.
6. **Platform Frontend:** login, sesiones, administración de administradores, catálogo de servicios y gestión de accesos.
7. **Seguridad y operación:** recuperación de cuentas, hardening, despliegues, secretos, backups y pruebas de recuperación.
8. **Validación integral:** pruebas de seguridad, consistencia, caída de dependencias, eventos duplicados/fuera de orden y revocación.

El orden es una propuesta de ejecución; el desglose definitivo y las dependencias se registran en Linear antes de comenzar a implementar.

## 13. Estado de especificación técnica

Los doce pendientes iniciales de especificación fueron concretados en el contrato técnico v1:

[`contracts/technical-contracts-v1.md`](contracts/technical-contracts-v1.md).

El contrato cierra autorización global, grants ante deshabilitación de servicios, REST interno, identidad técnica, revocación local de sesiones, cookies/CSRF, auditoría, parámetros de contraseñas, recuperación de emergencia, bootstrap y catálogo inicial. Los valores de dominios, hosts, secretos de proveedor y topología física permanecen parametrizados por entorno; la incorporación efectiva de cada servicio depende de que implemente y supere la validación del contrato v1.

La implementación debe seguir el contrato v1. Una modificación incompatible requiere decisión explícita, actualización documental y trazabilidad en Linear.


## 14. Estado y trazabilidad

- Prioridad 1 — Identidad, autenticación, JWT/JWKS y catálogo: **aprobada**.
- Prioridad 2 — Sesiones y revocación: **aprobada**.
- Prioridad 3 — Integración y sincronización con servicios: **aprobada**.
- Prioridad 4 — Administración de administradores y recuperación: **aprobada**.
- Prioridad 5 — Auditoría, despliegue e infraestructura: **aprobada**.
- Contratos técnicos v1: **cerrados** y publicados en [contracts/technical-contracts-v1.md](contracts/technical-contracts-v1.md).

**Estado de construcción:** PA-02 / WAB-103 continúa en curso. WAB-113 (fundación) y WAB-114 (persistencia de administradores y bootstrap seguro) están implementados localmente.

**Backend:** [ElemeSoft/bk-platform-waby](https://github.com/ElemeSoft/bk-platform-waby). La fundación publicada permanece en `f32c82c`. WAB-114 está en el commit local `ca57c3680e5929e3c11cf495e302638e51c90050` (`feat(platform-admin): implement admin persistence and secure bootstrap`), con 34 archivos. Ese commit no se ha publicado; `origin/master` continúa en `f32c82c` y la CI remota del commit queda pendiente.

**Verificaciones WAB-114:** typecheck, lint, build y `git diff --check` correctos; 28/28 pruebas en 8 archivos; integración/concurrencia 14/14; Docker build y smoke de readiness positiva, pérdida de MongoDB, readiness 503 y cierre SIGTERM correctos.

**Ejecución local:** se preparó un Compose independiente en `bk-platform-waby/compose.yaml`, conectado a la base lógica `platform_admin` de MongoDB Atlas. La validación de Compose/build se registra en el repositorio backend; la conexión real con Atlas requiere completar la URI local y no se ha verificado desde este documento. El bootstrap sigue siendo una tarea explícita y nunca se ejecuta al iniciar HTTP.

**Frontend:** [ElemeSoft/fr-platform-waby](https://github.com/ElemeSoft/fr-platform-waby), repositorio privado creado; alcance documentado en `eba3c59`; implementación aún no iniciada.

**Linear:** PA-01 completado; PA-02 en curso. WAB-114 se registra como implementación local completada, con publicación y CI remota pendientes. La siguiente entrega de backend será WAB-115 (autenticación humana y sesiones).

**Implementaciones existentes:** no se modifican como parte de esta iniciativa.

Este documento es la referencia arquitectónica inicial de WABY Platform Admin. Las decisiones futuras deben actualizarse aquí y registrarse en el proyecto independiente de Linear, preservando la separación con la iniciativa de agentes multitenant.
