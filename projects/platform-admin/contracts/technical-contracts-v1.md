# WABY Platform Admin — Contratos técnicos v1

**Fecha:** 2026-11-26  
**Estado:** aprobado para la primera implementación  
**Alcance:** contratos transversales de autorización, integración, sesiones, seguridad, auditoría y operación.  
**Fuente arquitectónica:** [README de Platform Admin](../README.md)

Este documento concreta los pendientes de especificación de la sección 13 del documento arquitectónico. No reabre las cinco prioridades aprobadas. Los valores dependientes del entorno (dominios concretos, URLs, secretos y topología física) se configuran por entorno y no cambian los contratos aquí definidos.

## 1. Autorización global

### 1.1 Principios

- `Platform SuperAdmin` es la autoridad global de la plataforma.
- `Platform Admin` no obtiene autoridad global ni acceso a servicios por pertenecer a ese nivel.
- Todo permiso de Platform Admin es explícito, asignado por un SuperAdmin y revocable.
- No existe autoelevación, autodesactivación ni delegación de permisos globales.
- La autorización se evalúa en backend; ocultar controles en frontend no constituye autorización.
- El último SuperAdmin activo no puede desactivarse ni perder su condición mediante una operación ordinaria.

### 1.2 Permisos globales

Catálogo inicial de permisos asignables a Platform Admin:

| Permiso | Capacidad |
|---|---|
| `platform.admins.read` | Consultar administradores |
| `platform.admins.create` | Crear administradores de nivel Platform Admin |
| `platform.admins.update` | Actualizar datos permitidos de administradores |
| `platform.admins.deactivate` | Desactivar administradores, sujeto a restricciones |
| `platform.admins.reactivate` | Reactivar administradores sin restaurar grants |
| `platform.services.read` | Consultar catálogo y estado de servicios |
| `platform.grants.read` | Consultar grants |
| `platform.grants.create` | Conceder grants de servicio dentro del alcance delegado |
| `platform.grants.revoke` | Revocar grants |
| `platform.audit.read` | Consultar auditoría |
| `platform.sessions.read` | Consultar sesiones |
| `platform.sessions.revoke` | Revocar sesiones |

Reglas:
- Un Platform Admin nuevo comienza sin permisos globales asignados, salvo el acceso mínimo de autenticación y lectura de su propio perfil/sesión.
- Un SuperAdmin asigna permisos globales individualmente.
- `platform.admins.create` no permite crear SuperAdmins ni asignar permisos globales al nuevo administrador.
- `platform.grants.create` no permite asignar roles superiores al alcance delegado ni alterar el catálogo local de roles/permisos.
- Las acciones reservadas a SuperAdmin incluyen crear/promover/degradar SuperAdmins, delegar permisos globales, administrar configuración crítica y ejecutar recuperación de emergencia.
- Un administrador no puede modificar sus propios permisos, elevarse, desactivarse ni revocar su propia sesión mediante rutas administrativas generales. Logout de su sesión actual sí está permitido.
- No se puede desactivar al último SuperAdmin activo.

## 2. Estado de servicios y grants

Estados centrales del servicio: `active`, `disabled`.

Al pasar un servicio a `disabled`:
1. Se impiden nuevas concesiones y emisión de credenciales de acceso dirigidas al servicio.
2. Todos los grants centrales activos pasan a estado de revocación deseada.
3. El Platform Backend intenta aplicar la revocación mediante REST y publica el evento correspondiente.
4. Si el servicio confirma, el grant queda revocado. Si está inaccesible, queda `pending_revoke` o `failed`; nunca se informa como revocación efectiva confirmada.
5. El servicio debe reconciliar el estado deseado antes de aceptar de nuevo solicitudes administrativas cuando se recupere.
6. Volver a `active` no restaura grants revocados. Se requieren nuevas concesiones auditadas.

El estado `disabled` bloquea nuevas operaciones desde Platform Admin aunque el servicio siga técnicamente disponible.

## 3. Contrato REST interno v1

### 3.1 Convenciones

- Base path: `/internal/platform-admin/v1`.
- Transporte HTTPS, JSON UTF-8.
- Identidad técnica autenticada en cada operación; nunca aceptar un JWT humano como credencial de servicio.
- `X-Request-Id` identifica la petición; `X-Correlation-Id` correlaciona el flujo distribuido. Si no se reciben, el receptor genera IDs y los devuelve.
- Operaciones mutables requieren `Idempotency-Key` estable para reintentos.
- Fechas ISO-8601 UTC.
- Los identificadores son strings opacos; no asumir formato ObjectId en contratos entre servicios.
- Respuestas de catálogo incluyen `catalogVersion` para detectar cambios.
- Compatibilidad: cambios aditivos compatibles dentro de v1; cambios incompatibles requieren nueva versión mayor en el path. Los campos desconocidos deben ignorarse cuando no afecten a la seguridad.

### 3.2 Endpoints

| Método y ruta | Propósito |
|---|---|
| `GET /catalog` | Identidad del servicio, estado, versión del contrato y versión del catálogo |
| `GET /roles` | Roles administrativos disponibles |
| `GET /permissions` | Permisos locales disponibles |
| `POST /access/grant` | Aplicar/actualizar grant local |
| `POST /access/revoke` | Revocar grant local |
| `GET /access/status?platformAdminId=... ` | Consultar acceso efectivo local |

### 3.3 Catálogo y roles

Respuesta de `GET /catalog`:

```json
{
  "serviceId": "crm",
  "serviceName": "WABY CRM",
  "serviceStatus": "active",
  "contractVersion": "1.0",
  "catalogVersion": "2026-11-26.1"
}
```

Respuesta de `GET /roles`:

```json
{
  "serviceId": "crm",
  "catalogVersion": "2026-11-26.1",
  "roles": [
    {
      "roleKey": "platform_superadmin",
      "label": "Platform SuperAdmin",
      "isPlatformAdminRole": true,
      "assignable": true
    }
  ]
}
```

Cada servicio debe declarar exactamente un rol con `isPlatformAdminRole: true`. El servicio es dueño de las definiciones y permisos del rol. Platform Backend solo consume y asigna `roleKey`.

Respuesta de `GET /permissions`:

```json
{
  "serviceId": "crm",
  "catalogVersion": "2026-11-26.1",
  "permissions": [
    { "permissionKey": "crm.users.read", "label": "Read CRM users" }
  ]
}
```

### 3.4 Grant

Request de `POST /access/grant`:

```json
{
  "operationId": "opaque-operation-id",
  "platformAdminId": "opaque-admin-id",
  "grantId": "opaque-grant-id",
  "roleKey": "platform_superadmin",
  "catalogVersion": "2026-11-26.1",
  "reason": "Approved platform access change"
}
```

Respuesta confirmada:

```json
{
  "operationId": "opaque-operation-id",
  "serviceId": "crm",
  "platformAdminId": "opaque-admin-id",
  "grantId": "opaque-grant-id",
  "effective": true,
  "roleKey": "platform_superadmin",
  "appliedAt": "2026-11-26T12:00:00.000Z"
}
```

Un HTTP 2xx solo significa grant aplicado cuando `effective: true`. Si queda pendiente, responder `202 Accepted` con `effective: false` y `status: "pending"`; no presentarlo como concedido.

### 3.5 Revoke

Request de `POST /access/revoke`:

```json
{
  "operationId": "opaque-operation-id",
  "platformAdminId": "opaque-admin-id",
  "grantId": "opaque-grant-id",
  "reason": "Administrator access revoked"
}
```

Respuesta:

```json
{
  "operationId": "opaque-operation-id",
  "serviceId": "crm",
  "platformAdminId": "opaque-admin-id",
  "grantId": "opaque-grant-id",
  "effective": false,
  "status": "revoked",
  "appliedAt": "2026-11-26T12:00:00.000Z"
}
```

Repetir la misma operación con la misma clave de idempotencia devuelve el resultado original sin duplicar efectos. Una revocación prevalece sobre una concesión anterior si los eventos llegan fuera de orden; una concesión posterior solo se aplica si su generación/versión de estado es más reciente.

### 3.6 Access status

```json
{
  "serviceId": "crm",
  "platformAdminId": "opaque-admin-id",
  "status": "active",
  "effective": true,
  "roleKey": "platform_superadmin",
  "catalogVersion": "2026-11-26.1",
  "checkedAt": "2026-11-26T12:00:00.000Z"
}
```

Estados posibles: `active`, `revoked`, `pending`, `not_found`, `service_disabled`.

### 3.7 Errores

Formato común:

```json
{
  "error": {
    "code": "CATALOG_VERSION_CONFLICT",
    "message": "The submitted catalog version is no longer current.",
    "retryable": false,
    "requestId": "opaque-request-id",
    "details": {}
  }
}
```

Códigos HTTP:
- `400`: request inválido.
- `401`: identidad técnica ausente o inválida.
- `403`: emisor autenticado sin scope.
- `404`: servicio/recurso inexistente.
- `409`: conflicto de versión, idempotencia o estado.
- `422`: operación semánticamente no válida.
- `429`: rate limit.
- `500`: error interno no clasificado.
- `502/503/504`: dependencia no disponible, servicio temporalmente no disponible o timeout.

Nunca incluir stack traces, credenciales, tokens ni secretos en respuestas externas.

## 4. Identidad técnica servicio-a-servicio

- JWT técnico RS256, separado del JWT humano.
- Platform Backend firma los tokens salientes hacia servicios; cada servicio valida con las claves públicas de Platform Backend/JWKS.
- Para callbacks/eventos autenticados desde un servicio hacia Platform Backend, cada servicio tiene su propio par de claves y emisor registrado; Platform Backend confía en la clave pública registrada para ese servicio.
- Claims obligatorios: `iss`, `sub`, `aud` específico, `iat`, `exp`, `jti`, `scope`; `kid` en header.
- Expiración máxima de 2 minutos para tokens técnicos.
- Audience exacta por servicio/receptor; scopes mínimos por endpoint.
- Claves privadas solo en secret manager/entorno del emisor. Nunca se comparten claves privadas entre servicios.
- Rotación con solapamiento controlado de claves públicas y `kid`; revocación de emisor/claves comprometidas.
- No usar API keys compartidas de larga duración ni el JWT humano para llamadas internas.

## 5. Revocación de sesiones humanas

- Cada servicio que acepte tokens humanos mantiene un registro local mínimo de `sid` revocados; no replica la colección central de sesiones ni almacena refresh tokens.
- El registro local incluye `sid`, `revokedAt`, `expiresAt`, `sourceEventId` y metadatos mínimos de trazabilidad.
- TTL: hasta `exp` del Access Token más 60 segundos de margen de reloj.
- Platform Backend envía revocaciones síncronas por REST y publica evento RabbitMQ como mecanismo complementario.
- Objetivo de propagación: menos de 30 segundos a servicios online.
- Reconciliación incremental cada 60 segundos y reconciliación completa al recuperar conectividad.
- El servicio no debe reanudar aceptación de identidades administrativas después de una caída/reconexión hasta completar la reconciliación de revocaciones pendientes.
- Si Platform Backend cae, los servicios siguen validando firma, issuer, audience, expiración y su estado local conocido. No se renuevan sesiones. Un token expirado se rechaza.
- La expiración del JWT es defensa en profundidad, no reemplaza la revocación.
- Las operaciones de revoke son idempotentes y prevalecen ante mensajes fuera de orden.

## 6. Cookies, CSRF y orígenes

Topología lógica requerida: frontend y API bajo el mismo dominio registrable (same-site), en hosts separados. Los nombres concretos son configuración de cada entorno.

Refresh cookie:
- `HttpOnly`, `Secure`, `SameSite=Lax`.
- Host-only: no definir atributo `Domain`.
- Path limitado al endpoint de refresh cuando sea compatible con el flujo.
- No exponer Refresh Token a JavaScript ni persistirlo en localStorage/sessionStorage.
- Requests del frontend incluyen credenciales.

CSRF:
- Token sincronizador ligado a la sesión, entregado por API y mantenido en memoria por el frontend.
- Enviar en header `X-CSRF-Token` en refresh/logout y operaciones autenticadas con cookie.
- Validar token, método/origen y `Origin` contra allowlist exacta.
- CORS con credenciales solo para el origen administrativo autorizado; nunca wildcard.
- Si una futura topología requiere cross-site real, debe revisarse explícitamente y utilizar `SameSite=None; Secure` junto con controles CSRF equivalentes. No se habilita por defecto.

## 7. Auditoría e integridad

Catálogo mínimo de acciones:
- `admin.created`, `admin.updated`, `admin.deactivated`, `admin.reactivated`, `admin.permission_granted`, `admin.permission_revoked`.
- `auth.login_succeeded`, `auth.login_failed`, `auth.logout`, `auth.logout_all`, `auth.password_changed`, `auth.password_reset_requested`, `auth.password_reset_completed`.
- `session.created`, `session.refreshed`, `session.revoked`, `session.reuse_detected`.
- `service.registered`, `service.enabled`, `service.disabled`, `service.catalog_sync_failed`.
- `grant.created`, `grant.applied`, `grant.revocation_requested`, `grant.revoked`, `grant.sync_failed`.
- `bootstrap.completed`, `recovery.break_glass_started`, `recovery.break_glass_completed`, `recovery.break_glass_failed`.

Metadata:
- Allowlist por acción; solo IDs opacos, estado anterior/nuevo, reason code, serviceId, operationId, requestId y campos de diagnóstico no sensibles.
- Nunca almacenar passwords, hashes de password, tokens, cookies, claves privadas, secretos, códigos de recuperación ni cuerpos completos de requests.
- Registrar IP y user-agent conforme a política de privacidad aplicable.
- Acceso de aplicación a la colección con inserción y lectura; sin update/delete ordinarios.
- Índices por fecha, actor, acción, recurso y servicio.
- Retención inicial 12 meses, configurable y revisable antes de producción.
- Exportación a almacenamiento inmutable/WORM cuando la infraestructura lo permita. La colección central sigue siendo la fuente oficial.
- Auditoría de acciones sensibles tanto exitosas como fallidas; si la escritura de auditoría falla, una mutación privilegiada no debe confirmarse como exitosa.

## 8. Password policy y rate limiting

- Argon2id, parámetros iniciales: memoria 19 MiB, 2 iteraciones, paralelismo 1. Medir en infraestructura objetivo antes de producción y aumentar si el presupuesto de latencia lo permite.
- Mínimo 12 caracteres; aceptar contraseñas largas; sin reglas artificiales de composición.
- No truncar silenciosamente; máximo de entrada 128 caracteres Unicode normalizados de forma consistente.
- Comprobar contraseñas comprometidas al crear/cambiar contraseña mediante mecanismo de consulta que no revele la contraseña completa (k-anonymity o dataset local).
- Rate limit inicial: 5 fallos por cuenta normalizada en 15 minutos y 20 intentos por IP en 15 minutos; respuesta genérica para evitar enumeración. Aplicar enfriamiento temporal progresivo con máximo de 15 minutos; nunca bloqueo permanente ni estado `locked`.
- Umbrales configurables por entorno y monitorizados; el almacenamiento del limitador debe funcionar entre réplicas, no ser solo memoria local.
- Recuperación de contraseña: token aleatorio de un uso, hash almacenado, expiración 15 minutos, sesiones revocadas al completar el cambio.

## 9. Recuperación de emergencia y bootstrap

### 9.1 Break-glass

- Procedimiento operativo fuera de rutas públicas de la aplicación.
- Requiere acceso autenticado a infraestructura, aprobación de dos operadores autorizados y registro del ticket/razón.
- No crea una identidad automáticamente ni habilita registro público.
- Solo permite recuperar la identidad SuperAdmin existente o promover una identidad preexistente mediante comando auditado.
- Fuerza cambio de contraseña al próximo login, revoca todas las sesiones y exige revisión posterior.
- Si Platform Backend no puede escribir auditoría central, la evidencia de emergencia se conserva en logs de infraestructura protegidos y se importa/reconcilia al recuperar el servicio.
- Ejecutar simulacro documentado antes de producción.

### 9.2 Bootstrap inicial

- Comando explícito de una sola ejecución, ejecutado como tarea operativa; nunca en cada arranque HTTP.
- Solo procede si no existe ningún SuperAdmin y la operación es atómica/idempotente.
- Secretos `PLATFORM_BOOTSTRAP_ADMIN_EMAIL` y `PLATFORM_BOOTSTRAP_ADMIN_PASSWORD` inyectados mediante secret manager para esa ejecución, con TTL operativo máximo de 1 hora.
- Password generado aleatoriamente, almacenado con Argon2id y `mustChangePassword=true`.
- El proceso termina tras éxito; retirar/inutilizar el secreto en el secret manager inmediatamente. No recrear automáticamente si la cuenta existe y fue desactivada.
- Primer SuperAdmin recibe el rol máximo de los servicios activos que ya estén registrados; servicios incorporados posteriormente requieren operación de asignación auditada.

## 10. Topología y servicios iniciales

- Desarrollo local: Docker Compose permitido para levantar componentes cuando se incorporen; no es requisito de orquestación productiva.
- Staging y producción aislados, HTTPS, secretos externos, MongoDB y RabbitMQ privados, backups y pruebas de restauración.
- Frontend y API usan hosts propios bajo el mismo dominio registrable; cookies host-only en API.
- Backend dockerizado desde el primer commit; imagen multi-stage, usuario no privilegiado, health check y configuración runtime.
- Catálogo inicial: vacío hasta que un servicio complete y valide el contrato interno v1. CRM será el primer candidato de integración; no se marcará activo ni se concederán grants hasta que implemente el adaptador, autenticación técnica y revocación local. `ai-service` se evaluará como servicio separado solo si se confirma que tiene una frontera propia de identidad/autorización y cumple el contrato.
- `infrastructure` es infraestructura compartida, no un servicio administrativo registrable por sí mismo.
- URLs, dominios, nombres de secretos del proveedor, réplicas y topología física son valores de despliegue por entorno; no quedan hardcodeados en el contrato.

## 11. Estado de cierre

Los doce pendientes de especificación quedan concretados para iniciar la fundación y desarrollar los contratos v1. Los valores de dominio/topología física y la incorporación efectiva de servicios se resuelven por configuración y por la fase de integración, sin bloquear WAB-113.

Cambios incompatibles con este documento requieren decisión explícita, actualización de este contrato y trazabilidad en Linear.
