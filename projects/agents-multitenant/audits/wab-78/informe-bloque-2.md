# WAB-78 — Informe del Bloque 2: Provisioning y permisos (D2)

**Fecha:** 04/11/2026 · **Alcance:** diagnóstico exclusivo del dominio D2 (plan de revisión §4, Bloque 2).
**Modo:** solo lectura (plan §10). No se modificó código, permisos, roles, `ResourceOwnership`, schemas, modelos, seeds, tests, documentos canónicos ni registro de decisiones; no se crearon/editaron issues, estados o relaciones en Linear; no se emitió veredicto.

---

## 1. Fuentes utilizadas (nivel primario salvo indicación)

1. **Plan de revisión** (Linear `fca2514ac4ee`): §1 (estados de control), §2 (línea base de decisiones), §3 (clasificación de hallazgos), §4 Bloque 2 (alcance y preguntas), §6 (reutilización del análisis previo), §7 (matriz acumulativa), §8 (formato obligatorio), §10 (restricciones).
2. **Registro de decisiones 03/11** (Linear `345541575496`): §1 (P1), §2 (P9), §3 (suspensión operativa), §4.1–§4.9 (P2, incl. checkbox `isDefault` y prohibiciones a agentes IA), §6 (P3: sin decisión de permisos; no introducir `bot.configure`), §8.1 (AgentInstance persiste en CRM; ownership `AgentInstance → Workspace → Organización`), §9.
3. **Descripciones de issues:** WAB-78 (estado de decisiones, P3, gate), WAB-79 (dominio/ownership; "no se crean definiciones de plataforma desde el ámbito del cliente"), WAB-80 (provisioning y autorización: ciclo de vida a Waby, "la matriz determina qué operaciones son exclusivas de Waby y cuáles se delegan"), WAB-75 (decisiones arquitectónicas 1, 2, 5, 6, 18: administración centralizada, unidad de propiedad = Workspace, proveedores/credenciales de Waby).
4. **`analisis-previo-d2-d4.md` §D.2** — material preliminar con ARQ-03 y ARQ-04 (IDs y procedencia conservados).
5. **`informe-bloque-1.md`** §5 (contradicción 6) y §6.1–§6.3 (cuestiones y restricciones heredadas).
6. **`matriz-acumulativa.md`** y **`00-linea-base.md`** (estado de ARQ-03/04, convenciones de ID).
7. **Canónico D.1/D.2/D.3/D.4/D.5** (`crm/docs/plans/wab-75-fase-2-diseno.md` L690-868).
8. **Auditoría de permisos** (`crm/docs/reports/AUDITORIA_PERMISOS_WABY.md`): §2.4 (`bot.configure`), §3 (catálogo), §6 (H1–H9), §8 (matriz de agentes IA), §11 (validación en `b9e0aee`). **AUD-02/03/05** (comentarios en WAB-88/WAB-89/WAB-91) se **reutilizan como evidencia, no se repiten** (plan §6).
9. **Documentación de negocio:** `crm/docs/business/permisos-roles.md` (BR-PERM-001 L15-26, BR-PERM-002 L32-43, BR-PERM-006 L86, H2–H5 L162-199), `org-user-hierarchy.md`, `wrk-user-hierarchy.md`.
10. **Código CRM verificado en esta sesión** (commit `b9e0aee` = `origin/master`; `crm/src` sin `platform`, sin `bot.configure`, sin `AgentInstance`):
    - `src/auth/permisions/permission.enum.ts` L19-26, L125-130.
    - `src/auth/hasPermission.ts` L34-79.
    - `src/graphql/Auth/AuthChecker.ts` L7-69; `src/graphql/Auth/WrkGuard.ts` L6-32.
    - `src/auth/resolveAuthContext.ts` L87-117; `src/server.ts` L236-249.
    - `src/auth/ResourceOwnership.ts` L33-45, L92, L101, L160-190, L204-373.
    - `src/graphql/resolvers/AiBot.resolver.ts` (L18-135); `AiContactAssignment.resolver.ts` (L10-70); `TipAccessWorks.resolver.ts` (L10-25); `Media.resolver.ts` (L14, L26, L37); `Contacto.resolver.ts` (L81, L238, L855-857).
    - Seeds: `src/utils/autoloader/tipAccessOrg.roles.ts`, `tipAccessWorks.roles.ts`.
    - ai-service: `microservices/ai-service/src/presentation/http/middleware/auth.middleware.ts`, `routes/bot.routes.ts`, `src/infrastructure/crm/client/crm-graphql-client.ts` L112-137, `src/infrastructure/config/config-manager.ts` L195-197, `src/infrastructure/composition-root.ts` L358.

---

## 2. Conclusión ejecutiva

1. **El sistema de permisos actual sí existe y es reutilizable:** modelo `org.*`/`wrk.*` con formato `ámbito.módulo.acción` (BR-PERM-001, CANONICAL), roles predefinidos (BR-PERM-002), wildcards, flujo T0→T5, guard de workspace (`assertWrkAccess`), ownership T3 (`ResourceOwnership`, 12 tipos), restricciones de membresía (BR-PERM-006), seeds y tests. Cumple el objetivo de P3/§6 de "reutilizar el sistema actual y evitar un segundo sistema" **si** los nuevos permisos se emiten dentro de su formato y con su mecanismo de alcance.
2. **El contrato de D2 para AgentInstance no existe en ninguna capa:** no hay permiso de ciclo de vida, no hay tipo `AGENT_INSTANCE`, no hay actor declarado para crear/activar/suspender/retirar/designar predeterminada, y no hay identidad de servicio representable para las operaciones internas de D.2.
3. **Los permisos que propone el canónico no son representables de forma segura en el modelo real:** `platform.agent.*` y `bot.configure` no pertenecen al formato BR-PERM-001; si se añadieran tal cual, el atajo `org.*.*` (`hasPermission.ts:45`) los aprobaría para **cualquier Admin/User Admin de cualquier organización**, sin scoping de workspace (AuthChecker solo aplica `assertWrkAccess` a permisos `wrk.*`), y **no existe actor "Waby"** porque `AuthChecker` exige rol de organización (L27-29). `bot.configure` además está **prohibido por P3** (registro §6) pese a estar asumido en D.1/D.5.
4. **La separación Waby/cliente de D.4 hoy no tiene mecanismo:** la superficie real de administración de bots está abierta a roles de organización (`org.workspaces.*` sin T3, AUD-02) y el cliente aporta `apiKey`/modelo (AUD-03 + `AiBot.resolver.ts:28-37`); ai-service expone CRUD con token compartido (AUD-02). La "evidencia de separación" de D.4 ("`platform.*` no asignable a clientes") es circular: el permiso no existe y su ausencia no protege la superficie actual.
5. **Bloqueantes D2:** ARQ-03 (validado y ajustado), ARQ-19, ARQ-20, ARQ-21. **Diferibles:** ARQ-04 (H1/H4/H5/H7, sin cambios en clasificación) y el contrato de notificación a sesiones abiertas (H7). Siguen vigentes como bloqueantes de D1: ARQ-13 y ARQ-14 (referenciados, no duplicados).

---

## 3. Respuestas a las cuestiones del Bloque 2 (§6.2)

### 3.1 Cuestión 1 — Validación/ajuste de ARQ-03

**Resultado: validado y ajustado** (conserva ID y procedencia; se actualiza con evidencia nueva de D2).

**¿Qué contrato de permisos/ownership necesita cada fase?**

| Fase | Qué necesita | Estado real hoy |
|---|---|---|
| **WAB-79** (dominio, ownership, persistencia) | (a) ownership de AgentInstance hasta Organización (ya aprobado: `AgentInstance → Workspace → Organización`, registro §8.1); (b) cadena `workspaceId → ref_organizacion` verificable (existe: `Works.model.ts:63`); (c) unicidad/`isDefault` (ARQ-13, bloqueante de D1); (d) frontera de persistencia (ARQ-14, bloqueante de D1). **No requiere permisos nuevos para modelar.** | Dependencias de D1 ya registradas; el modelo de permisos no lo frena. |
| **WAB-80** (provisioning y autorización) | (1) matriz de permisos P3 aprobada (registro §6, todavía pendiente); (2) permisos con formato y alcance resueltos (ARQ-20); (3) actor del ciclo de vida decidido (ARQ-19); (4) identidad de servicio para operaciones internas (ARQ-21); (5) mecanismo T3 sobre instancias (propuesta `AGENT_INSTANCE`, §3.4). | **Ninguno de los cinco existe.** El mecanismo base (roles/scopes/T3/wildcards) sí existe y es reutilizable. |

**Cuál es el estado real (no es "ausencia de decisión de permisos en general"):**

- El **diseño de permisos del producto actual** existe y está testeado: `org.*`/`wrk.*`, wildcards, seeds, guards, `AuthChecker.test.ts`.
- Lo que **no existe** es el contrato para las operaciones de D2: en `permission.enum.ts` no hay `platform.*`, ni `bot.configure`, ni `wrk.agent.*`, ni `wrk.credits.*`; en `crm/src` no hay `AgentInstance` ni `AGENT_INSTANCE` (grep 0, coincide con H9/AUD §11.1); `ResourceOwnership.ts:33-45` tiene 12 tipos y no incluye `AGENT_INSTANCE`.
- El canónico **sí propone** permisos (D.2 L803-834, D.4 L850-860, D.5 L862-868), pero esa propuesta es **inintegra con el modelo real** → nuevo hallazgo **ARQ-20** (§4).
- `bot.configure` está **prohibido hasta que exista la matriz** (registro §6) mientras D.1 L702 y D.5 lo dan por existente → contradicción 6 ya registrada en D1 §5 (se amplía en §6 de este informe).
- Conclusión: **ARQ-03 se mantiene como bloqueante**, pero su formulación original ("sin permisos definidos… no pueden implementar") se ajusta: el bloqueo no es la ausencia de código (esperada antes de WAB-79/80, plan §6), sino **la ausencia del contrato de permisos/ownership/actor para las operaciones de D2 y una propuesta canónica no implementable con seguridad**.

### 3.2 Cuestión 2 — Validación de ARQ-04 (H1, H4, H5, H7 con aclaraciones)

**Resultado: ARQ-04 validado parcialmente; se mantiene Medio / No bloquea.** Ninguno de los cuatro impide definir quién/qué recurso/ownership/permiso/ámbito para provisioning (criterio del plan §8), pero todos quedan con trabajo concreto y destino.

| Hallazgo | Estado tras D2 | Evidencia (esta sesión) | Aclaración aplicada | ¿Bloquea D2? | Destino |
|---|---|---|---|---|---|
| **H1** — roles sin `ref_organizacion` | **Vigente** (confirmado en `b9e0aee`, AUD §11.1) | `TipAccessOrg.model`/`TipAccessWorks.model` sin `ref_organizacion`, `key` único global; mutaciones de roles globales con permisos de organización: `TipAccessWorks.resolver.ts:16-25` (`createTipAccessWorks` = `org.workspaces.update`, `del…` = `org.workspaces.delete`); `permisos-roles.md:152-160` (OPEN-09) | **No se introduce `ref_organizacion` automáticamente.** Efecto para D2: la matriz P3 no debe depender de roles custom por-organización para conceder permisos de agentes (el rol sería global); dejar la dependencia explícita | No | WAB-80 / decisión (OPEN-09 ya registrado) |
| **H4** — permisos muertos | **Parcialmente corregido con evidencia nueva** | `wrk.media.read` **sí se exige** hoy: `Media.resolver.ts:14` (`@Authorized(permissions.mediaRead)`) — corrige AUD §11.2 y `permisos-roles.md:162-167` ("ningún resolver lo pide"), que quedaron desactualizados. Siguen sin exigir: `wrk.workspaces.read` (grep 0 en `src/`; comentario `tipAccessWorks.roles.ts:54-57`), `wrk.media.create/delete` (ningún resolver; `deleteMedia` usa `@Authorized()` vacío, `Media.resolver.ts:37`) y `wrk.users.*` (H5) | Investigar `wrk.workspaces.read` y `wrk.*` familia media sigue pendiente. Relevancia D2: si la matriz agrega permisos nuevos, **verificar exigencia backend** antes de aprobar (lección BR-PERM-003/H10: un permiso sin exigir lo define el front) | No | WAB-80 (verificación) + actualización documental de AUD/`permisos-roles.md` en su momento |
| **H5** — Manager no invita/quita miembros | **Vigente** (confirmado AUD §11.1) | `UsOrgWork.resolver.ts` L138, 167, 188, 200, 221 protegidos con `org.workspaces.invite/delete/update` (permisos de **organización**), no `wrk.users.*`; con T3 nivel TENANT aplicado. Un Manager con rol Member de org solo tiene `org.workspaces.read` (`tipAccessOrg.roles.ts:46`) | **H5 está separado de las transferencias de conversaciones:** las transferencias/delegaciones de contactos (P2 §4.3) las valida Waby y no son operación de membresías; no confundir ambos permisos | No | WAB-80 (independiente del provisioning) |
| **H7** — alcance WebSocket no se actualiza | **Vigente** (confirmado AUD §11.1) | `alcanceContactos.ts:41-46` (contexto congelado al abrir el socket); mecanismo de cierre por usuario existe: `socketsDeSesion.ts` (`cerrarSocketsDeUsuario` 4404 / `cerrarSocketsAnteriores` 4403) — **por usuario, no por bot/instancia** | **Evaluar el riesgo de sockets abiertos tras revocación:** aplicado a D2, ni la suspensión/retiro de una instancia (P9 regla 3: notificar a usuarios autorizados) ni la revocación de membresía invalidan el alcance ya congelado; hoy solo hay cierre por usuario con token viejo | No (no impide definir actor/permiso/ownership) | Contrato a definir en WAB-80/D4 (notificación/invalidación a sesiones abiertas ante cambios de estado de instancia) |

### 3.3 Cuestión 3 — Matriz de actores × operaciones de AgentInstance

**Tabla A — Detalle por operación (recurso, ownership, permiso, estado).**

| # | Operación (D.2/D.4) | Recurso | Ownership | Permiso canónico propuesto | Estado real en código (evidencia) | Actor definido hoy |
|---|---|---|---|---|---|---|
| 1 | Crear / provisionar instancia | AgentInstance (CRM) | `AgentInstance → Workspace → Organización` (registro §8.1, aprobado) | `platform.agent.instance.create` → Waby (D.2 L810) | No existe entidad ni resolver ni permiso (grep 0). Análogo: `createAiBot` (`AiBot.resolver.ts:18-20`) con `org.workspaces.create`, **sin T3** (AUD-02) y aceptando `apiKey`/`model` del cliente (L28-37) | **No** (ARQ-19/20) |
| 2 | Consultar / listar instancias | AgentInstance | Idem | `platform.agent.instance.read` → Waby, **incluida "listar instancias del workspace"** (D.2 L814) | Análogo: `getAiBot`/`listAiBots` = `org.workspaces.read` sin T3; `workspaceId` lo aporta el cliente (`AiBot.resolver.ts:57-58`) | **No** (ARQ-19: hasta la lectura está reservada a Waby en D.2) |
| 3 | Configurar Skills/Rules de la instancia | Skills/Rules (`bot_resources`; por instancia en D.2) | Decisión aprobada: Skills/Rules pertenecen a la AgentInstance (plan §2); hoy RULE/SKILL tienen T3 (`ResourceOwnership.ts:338-339`) y `assertOwnershipOrgAware` en `BotResource.resolver` | `bot.configure` (D.2 L823, D.5 L862-868) — **prohibido por P3 hasta matriz** (registro §6) | Hoy: `wrk.rules.*`/`wrk.skills.*` + ownership; Owner/Manager escriben, Agent solo lee (`tipAccessWorks.roles.ts:38-39, 74-75`) | Parcial: cliente "solo con permiso específico" (D.2), pero el permiso está prohibido → **P3** |
| 4 | Activar | AgentInstance | Idem | `platform.agent.instance.update` → Waby (D.2 L811) | No existe. Análogo: `activateAiBot` = `org.workspaces.update` sin T3 (`AiBot.resolver.ts:100-102`) | **No** (ARQ-19/20) |
| 5 | Suspender | AgentInstance | Idem (también por condición operativa: registro §3, a nivel Organización) | `platform.agent.instance.update` → Waby (D.2 L812) | No existe. Análogo: `deactivateAiBot` = `org.workspaces.update` (`AiBot.resolver.ts:108-110`) | **No** (ARQ-19/20) |
| 6 | Retirar (lógico, P9 `retirement_pending/retired`) | AgentInstance | Idem | `platform.agent.instance.delete` → Waby (D.2 L813) | No existe. Análogo: `deleteAiBot` = `org.workspaces.delete` (`AiBot.resolver.ts:91-93`) | **No** (ARQ-19/20) |
| 7 | Eliminar (purga física) | AgentInstance / recursos | D.1 L708: delete-wrk → CRM + ai-service | No distinguido en D.2 del retirar | Análogo: DELETE `/bots/:id` de ai-service con token compartido (AUD-02). Retiro lógico (P9) vs purga física sin separar | **No** |
| 8 | Asignar contacto a instancia | Contacto (+ `EvAssigned` futuro, CRM autoridad) | Contacto → Workspace (aprobado: CRM única autoridad) | D.2 no asigna permiso humano explícito para asignar a agente; delegación/transferencia: Waby valida destino (P2 §4.3, aprobado) | Hoy: asignar a **bot** = `org.workspaces.update` sin T3 y con `workspaceId` del cliente (`AiContactAssignment.resolver.ts:35-49`); asignar a **humano** = `wrk.contacts.update` (`Contacto.resolver.ts:855-857`); runtime vía capability (AUD-05) | Parcial (Waby valida, no ejecuta humano) → **P3** |
| 9 | Desasignar / transferir | Contacto | Idem | Waby valida elegibilidad/fallback (P2 §4.3/§4.5, aprobado) | `unassignContact` = `org.workspaces.update` (`AiContactAssignment.resolver.ts:66-70`); agentes IA **no** gestionan membresías/usuarios/equipos (P2 §4.3, aprobado) | Parcial → **P3** |
| 10 | Designar predeterminada (`isDefault`) | AgentInstance | Idem | Sin permiso declarado; checkbox "al crear o editar" (P2 §4.1) **sin declarar actor** | No existe el campo ni el mecanismo (ARQ-13, bloqueante D1) | **No** (ARQ-19 + ARQ-13) |
| 11 | Definiciones AgentDefinition (crear/editar/versionar/retirar/catálogo) | AgentDefinition | Waby (decisiones aprobadas: plan §2; WAB-79 CA "no se crean definiciones desde el ámbito del cliente") | `platform.agent.definition.*` → Waby, "no asignable a clientes" (D.2 L805-809, D.4 L854) | No existe `AgentDefinition`. Análogo actual: bots de ai-service **administrables por el cliente** hoy (AUD-03 mapa UI→API con `wrkCreate/Update/Delete` = `org.workspaces.*`) | Decisión aprobada (Waby) pero **sin mecanismo** (ARQ-20) |
| 12 | Elegir proveedor / modelo / credenciales | Credenciales de proveedor | Waby (WAB-75 #5/#6/#18; D.4 L855 "❌ prohibido") | Sin permiso propio en D.2 (va implícito en `platform.agent.definition.*`) | **Hoy al revés:** el cliente envía `apiKey` y `model` en `createAiBot`/`updateAiBot` (`AiBot.resolver.ts:28-37`) y la UI lo permite (AUD-03) | Decisión aprobada (Waby) pero **hoy incumplida** (AUD-03, referenciado) |
| 13 | Ejecutar / usar instancia (conversación) | Runtime (ai-service) | Capacidades del runtime, **sin heredar roles humanos** (plan §4) | `wrk.agent.use` (D.2 L820) — **no existe** | Hoy: gate real de mensajes = `wrk.messages.create` (seeds L66-67); resolución por `botId` con token compartido; `workspaceId` del payload no validado contra el bot (AUD-05 → dominio D3/D4) | Waby opera el runtime (infraestructura); permiso de uso → **P3** |

**Tabla B — Matriz actores × operaciones (permiso efectivo HOY).**

Leyenda: `org.R/C/U/D` = `org.workspaces.read/create/update/delete` (ámbito **organización**, sin T3 en AiBot/AiContactAssignment) · `wrk.x` = permiso `wrk.x` efectivo (exige membresía: `assertWrkAccess`) · `tok` = token interno compartido de ai-service, sin identidad de tenant (AUD-02) · `✗` = operación/entidad no existe hoy · `P3` = sin definir, pendiente de la matriz P3 (ARQ-19/20/21) · `—` = sin acceso efectivo para ese rol.
Filas de rol de workspace asumen rol **Member** en la organización (BR-PERM-002); los permisos de organización y de workspace se combinan (`AuthChecker.ts:31-33`).

| Actor | 1 Crear | 2 Consultar | 3 Skills/Rules | 4 Activar | 5 Suspender | 6 Retirar | 7 Eliminar | 8 Asignar | 9 Desasig./Transferir | 10 isDefault | 11 Definiciones | 12 Credenciales | 13 Ejecución |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **Waby (plataforma)** | P3 | P3 | P3 | P3 | P3 | P3 | P3 | P3 | P3 | P3 | P3 | P3 | — |
| **Org Admin / User Admin** (`org.*.*`) | org.C | org.R | wrk.rules/skills.* (solo si es miembro del ws) | org.U | org.U | org.D | org.D | org.U | org.U | ✗/P3 | org.C/U/D (sobre bots) | org.U | — |
| **Org Billing Admin** | — | org.R | — | — | — | — | — | — | — | ✗ | org.R | — | — |
| **Org Member** | — | org.R | — | — | — | — | — | — | — | ✗ | org.R | — | — |
| **wrk Owner** (`wrk.*.*`) | — | org.R | wrk.rules/skills.* | — | — | — | — | wrk.contacts.update¹ | wrk.contacts.update¹ | ✗/P3 | org.R | — | wrk.messages.create² |
| **wrk Manager** | — | org.R | wrk.rules/skills.* | — | — | — | — | wrk.contacts.update¹ | wrk.contacts.update¹ | ✗/P3 | org.R | — | wrk.messages.create² |
| **wrk Agent** | — | org.R | wrk.rules.read/skills.read | — | — | — | — | wrk.contacts.update¹ | wrk.contacts.update¹ | ✗ | org.R | — | wrk.messages.create² |
| **runtime / ai-service** | tok | tok | — | tok | tok | tok | tok | tok (capability) | tok | — | tok | tok | — (es el propio) |
| **proceso interno CRM** (Outbox, delete-wrk, workers) | — | — | — | — | — | — | tok (delete-wrk) | — | — | — | — | — | — (Rabbit, AUD-05) |
| **Sin membresía en el ws** (autenticado) | — | — | — | — | — | — | — | — | — | ✗ | — | — | — |

**Notas de la Tabla B:**

1. Asignar a **humano** usa `wrk.contacts.update` (Owner/Manager/Agent pueden); asignar a **bot** exige `org.workspaces.update` (`AiContactAssignment.resolver.ts:35`) → hoy un Owner/Manager/Agent **no puede** asignar contactos a un agente IA por su rol de workspace. El "trampa de nombres" documentado en `permisos-roles.md:196-198` (`wrk*` del enum = `org.workspaces.*`) es el mismo que AUD-02 describió para AiBot: **se corrige aquí la clasificación de AUD-02** ("ámbito workspace" → ámbito **organización**), con evidencia `permission.enum.ts:19-23`, sin reabrir la auditoría.
2. D.2 propone `wrk.agent.use` para "usar instancia" (L820); el gating real de conversación hoy es `wrk.messages.create`.
3. `org.R` **no aplica T3** en AiBot/AiContactAssignment: un usuario con `org.workspaces.read` puede consultar bots de workspaces a los que no pertenece (riesgo ya registrado en AUD-02 → referenciado, no repetido).
4. Celdas `P3` en la fila Waby reflejan que **no hay actor Waby representable**: `AuthChecker.ts:14-16` y `:27-29` exigen usuario autenticado con rol de organización para cualquier `@Authorized` con permisos (ver ARQ-21).
5. `@Authorized()` vacío (roles = 0) devuelve `true` solo con usuario presente (`AuthChecker.ts:19-21`) — no gatea estas 13 operaciones, pero sí otras superficies (p. ej. `deleteMedia`, `Media.resolver.ts:37`).

### 3.4 Cuestión 4 — ¿Hace falta un tipo `AGENT_INSTANCE` en `ResourceOwnership`? (propuesta pendiente, NO aprobada)

**Propuesta a considerar en P3 (no se introduce ahora):** sí conviene un tipo `AGENT_INSTANCE`, por las siguientes razones verificables:

1. **El recurso persiste en CRM** (aprobado, registro §8.1), o sea que T3 puede resolverlo en una sola base, igual que `WORKSPACE`/`CONTACT`/`TAG` (`ResourceOwnership.ts:204-296`).
2. **El mecanismo ya existe y no crea un segundo sistema:** `assertResourceOwnership` (T101) y `assertOwnershipOrgAware` (L92, con nivel TENANT para Org Admin/UA) resuelven la cadena recurso→workspace→organización; un caso nuevo `case "AGENT_INSTANCE"` seguiría el patrón de `case "BOT"` (L362-373, hoy cross-DB vía `findAiServiceBot`) pero **sin cross-DB** porque la entidad será de CRM.
3. **Para operaciones servicio-a-servicio ya existe `assertServiceResourceOwnership`** (L160-190): recibe `organizationId`/`workspaceId` explícitos y valida CROSS_TENANT/WORKSPACE — reutilizable para la fila "runtime" de la matriz una vez definida la identidad (ARQ-21).
4. **Alternativa descartada como recomendación:** validar `workspaceId == x-wrk-id` ad-hoc en cada resolver (lo que hoy no se hace en `AiBot.resolver`, AUD-02) duplica lógica y se desincroniza (mismo patrón que BR-PERM-007/BM-MT-007 marcan como riesgo).

**Condiciones (todas pendientes, ninguna aprobada):** decisión P3 sobre el tipo, aplicación de T3 en los resolvers futuros de instancias, y que las operaciones **no hereden roles humanos** de la AgentInstance (plan §4: separación permisos humanos / capacidades del runtime). **No se añade `AGENT_INSTANCE` en esta revisión.**

### 3.5 Cuestión 5 — Contradicción 6 (canónico asume `bot.configure` vs P3): opciones registradas, no decididas

| Opción | Descripción | Consecuencia | Estado |
|---|---|---|---|
| **A** | Mantener la prohibición P3: actualizar D.1 L702, D.2 L823 y D.5 L862-868 para no presentar `bot.configure` como existente hasta que la matriz P3 lo defina | Actualización documental en WAB-78 (ya listada en registro §9.5); no bloquea WAB-79 | **Recomendada por coherencia con registro §6, no decidida aquí** |
| **B** | Aprobar `bot.configure` tal cual (D.5) | Requiere decisión expresa del usuario + resolver ARQ-20 (formato BR-PERM-001 y atajo `org.*.*`: hoy solo `org.*.*` lo aprobaría, contradictiendo "Owner/Admin del workspace puede concederlo", D.5 L864) | Requiere decisión de producto |
| **C** | Definir el permiso de configuración dentro del formato vigente (p. ej. `wrk.<módulo>.<acción>` para lo delegable al Workspace) dentro de la matriz P3 | Hereda `assertWrkAccess` y `wrk.*.*` de Owner; evita tocar BR-PERM-001 | Requiere decisión de producto |

Ninguna se aplicó: registro §6 prohíbe introducir el permiso hasta concluir el diseño de la matriz, y esta sesión es diagnóstico (plan §10).

### 3.6 Cuestión 6 — Separación Waby/cliente (D.4): qué impide hoy al cliente administrar definiciones/instancias

**Respuesta corta: nada estructural.** Evidencia:

1. **No hay actor Waby en el modelo:** `AuthChecker.ts:27-29` exige rol de organización; no existe scope/identidad de plataforma (grep `platform` = 0 en `crm/src`). Un operador de Waby sin membresía no puede pasar ningún `@Authorized` con permisos.
2. **La superficie de administración está abierta al cliente con permisos de organización:** `AiBot.resolver` completo con `wrkCreate/Read/Update/Delete` (= `org.workspaces.*`, `permission.enum.ts:19-23`) y **sin T3** (AUD-02, referenciado); el `workspaceId` lo aporta el payload (`AiBot.resolver.ts:25, 58`).
3. **El cliente hoy elige proveedor/modelo/credenciales** (§3.3 fila 12; AUD-03) — exactamente lo que WAB-75 #5/#6/#18 y D.4 L855 reservan a Waby.
4. **ai-service expone CRUD de bots con un token compartido sin ownership** (AUD-02, referenciado; `auth.middleware.ts` + `bot.routes.ts`), y los controllers aceptan `workspaceId` sin validarlo (AUD-02).
5. **La "evidencia de separación" de D.4 es circular:** "permiso `platform.agent.definition.*` no asignable a clientes" (L854, L856) — el permiso no existe y, si existiera tal cual, sería aprobable por `org.*.*` de cualquier organización (ARQ-20). La ausencia de un permiso no protege una superficie que usa otros permisos.
6. **La mitad interna tampoco está separada:** el "token interno" de D.2 no tiene camino de autenticación definido en CRM (ARQ-21).

**Consecuencia para D.4:** la tabla D.4 es un objetivo sin mecanismo. Su implementación depende de ARQ-19 (actor), ARQ-20 (permisos con alcance) y ARQ-21 (identidad interna); WAB-80 es la fase que debe retirar/restringir la superficie actual.

### 3.7 Cuestión 7 — Estado de ARQ-13 y ARQ-14 desde permisos/ownership (sin duplicar)

- **ARQ-13 (`isDefault` sin unicidad, Decisión pendiente P2 §4.1) — sigue bloqueando, sin cambios.** Aportación de D2, sin nuevo ID: la operación "designar predeterminada" (fila 10 de la matriz) tiene **tres** dependencias acumuladas: (a) integridad/mecanismo (ARQ-13), (b) actor que ejecuta el checkbox (ARQ-19), (c) permiso (P3). Resolver solo ARQ-13 no habilita la fila.
- **ARQ-14 (ownership de persistencia sin declarar en D.1) — sigue bloqueando, sin cambios.** Aportación de D2: T3 puede validar instancias en CRM **solo si** la persistencia en CRM se confirma por entidad (justo lo que pide ARQ-14) y si existe el tipo `AGENT_INSTANCE` (§3.4). D.1 L700 ("Gestión de AgentInstance: ✅ CRM (Waby-only)") mezcla API y persistencia — referenciar ARQ-14, no duplicar.

---

## 4. Hallazgos nuevos del Bloque 2 (formato obligatorio, plan §8)

### ARQ-19 — Actor del ciclo de vida de AgentInstance sin definir (D.2/D.4 Waby-only vs administración delegada al Workspace)

| Campo | Contenido |
|---|---|
| **ID** | ARQ-19 |
| **Dominio** | D2 |
| **Estado** | **Inconsistencia** (fuentes en tensión) + **decisión de producto pendiente** (matriz P3, registro §6) |
| **Evidencia** | Canónico `wab-75-fase-2-diseno.md`: D.2 L810-814 (provisionar/activar/suspender/retirar/**listar instancias del workspace** con `platform.agent.instance.*`, actor Waby) y D.4 L856 ("Asignar/activar/suspender instancias: Waby exclusivo / Cliente ❌ prohibido") vs. Registro 03/11 §4.1 (checkbox `isDefault` "al crear o editar una AgentInstance" **sin declarar quién**) vs. Plan §2 línea base ("Waby administra las definiciones centrales; AgentInstance pertenece a un Workspace") y WAB-75 §1/decisión 2 ("unidad de propiedad es el Workspace"; los clientes "aportan/configuran el contexto empresarial que tengan permitido administrar") vs. WAB-80 descripción ("conservar para los clientes las operaciones de uso y **configuración empresarial autorizadas**"; "la matriz determina qué operaciones son exclusivas de Waby y cuáles se delegan") vs. aclaración previa aprobada de la revisión (los usuarios del Workspace pueden administrar instancias y configuraciones empresariales **que permita la matriz de permisos**; plan §4/§6 la referencia como "aclaraciones ya aprobadas") |
| **Hallazgo** | La autoridad sobre cada operación de ciclo de vida de AgentInstance no está definida individualmente en ninguna fuente: creación de AgentInstance, consulta/listado, activación, suspensión, retiro lógico, eliminación física, configuración empresarial y designación predeterminada (`isDefault`) siguen sin un actor responsable declarado. D.2/D.4 reservan estas operaciones a un actor Waby con permisos `platform.*`, mientras el registro §6, P2 §4.1 y WAB-80 dejan la delegación a la matriz P3 y la aclaración previa asume administración de instancias por usuarios del Workspace, sin resolver quién ejecuta cada operación. La decisión debe resolverse operación por operación, no como una alternativa global única (Waby-only / delegación total / híbrido). |
| **Impacto** | **Alto** |
| **Bloquea WAB-78** | **Sí** |
| **Justificación** | La matriz de actores × operaciones es el entregable de D2: sin el actor no se puede completar ninguna fila de ciclo de vida (1, 4, 5, 6, 10) ni definir permiso y ámbito para WAB-80; aprobar D.4 tal cual resolvería en contra la decisión P3 que está abierta (dependencia concreta: guards/permisos de las cinco operaciones de ciclo de vida en WAB-80). |
| **Destino** | **WAB-78** (decisión en P3/matriz, ya pendiente en registro §6 — no reabrir P2) → detalle de implementación en **WAB-80** |
| **Recomendación** | Registrar como decisión de producto pendiente para el usuario (no solicitarla en diagnóstico, plan §10). **Criterio verificable:** cada operación de ciclo de vida (creación de AgentInstance, consulta/listado, activación, suspensión, retiro lógico, eliminación física, configuración empresarial, designación predeterminada `isDefault`) tiene actor responsable definido de forma individual, coherente entre D.2, D.4, P2 §4.1 y WAB-80, sin asumir una alternativa global Waby-only, delegación total o híbrida. |

### ARQ-20 — Permisos propuestos por D.2/D.4/D.5 inintegrales con el modelo real de autorización

| Campo | Contenido |
|---|---|
| **ID** | ARQ-20 |
| **Dominio** | D2 |
| **Estado** | **Contrato de autorización pendiente** (decisiones de producto/arquitectura + verificación técnica; contrato de permisos por diseñar en P3) |
| **Evidencia** | `permission.enum.ts:19-26, 125-130` (formato real `org./wrk.<módulo>.<acción>`; sin `platform`, `bot.configure`, `wrk.agent.*`, `wrk.credits.*`) · `permisos-roles.md:15-26` BR-PERM-001 (CANONICAL, Invariant: un permiso es `org.<módulo>.<acción>` o `wrk.<módulo>.<acción>`) · `hasPermission.ts:45` (`*` u `org.*.*` aprueban **cualquier** string) y `:77-78` (namespace desconocido → `false`) · `AuthChecker.ts:27-29` (sin rol de organización → `false`), `:43-57` (solo permisos `wrk.*` ejecutan `assertWrkAccess`) · `WrkGuard.ts:13-30` (exige `wrkRole` y coincidencia de workspace/organización) · Canónico D.2 L805-814, L820, L824-825; D.4 L854-856; D.5 L862-868 · Registro §6 (prohibición de `bot.configure`) |
| **Hallazgo** | (a) `platform.agent.*`/`platform.agent.definition.*` no existen, violan BR-PERM-001 y, de añadirse tal cual, serían aprobados por **cualquier Admin/User Admin de cualquier organización** vía `org.*.*` (`hasPermission.ts:45`), sin scoping de workspace (AuthChecker solo aplica `assertWrkAccess` a `wrk.*`) y sin actor Waby representable (AuthChecker exige orgRole). (b) `bot.configure` (namespace `bot`) solo pasaría por el mismo atajo, lo que contradice D.5 ("Owner/Admin del workspace puede concederlo") y WAB-80 ("no por roles ordinarios de Workspace"); además su introducción está prohibida por P3. (c) `wrk.agent.use` y `wrk.credits.read` (D.2 L820, L824-825) no existen: sin exigencia backend definida gatearían solo el front (lección H4/BR-PERM-003). Adoptar `platform.*` exige además cambiar una regla canónica (BR-PERM-001) por el protocolo de `crm/AGENTS.md`, no solo diseñar. |
| **Impacto** | **Alto** |
| **Bloquea WAB-78** | **Sí** |
| **Justificación** | WAB-80 define su aceptación sobre estos permisos ("operaciones exclusivas de Waby no accesibles desde credenciales de cliente", "edición exige permisos específicos"): aprobar D.2/D.4/D.5 tal cual trasladaría a la fase dependiente un mecanismo de autorización no implementable con seguridad (escalada de alcance y sin actor declarado), que es exactamente una decisión arquitectónica crítica que el plan §1 prohíbe trasladar. |
| **Destino** | **WAB-78** (matriz P3 + decisión de formato/actor de permisos de plataforma) → implementación en **WAB-80** |
| **Recomendación** | Diseñar en P3 los permisos delegables al Workspace dentro de `wrk.<módulo>.<acción>` (heredan `assertWrkAccess`, `wrk.*.*` y las restricciones BR-PERM-006) y resolver explícitamente el camino de permisos de plataforma (actor + alcance) antes de implementar; no introducir `bot.configure` hasta entonces. **Criterio verificable:** para cada permiso de la Tabla A se prueba su aprobación con los `orgPerms`/`wrkPerms` de cada rol de la Tabla B y **ningún** actor distinto del declarado lo obtiene (incluido el atajo `org.*.*`). |

### ARQ-21 — Operaciones internas de D.2 sin identidad de servicio representable en CRM

| Campo | Contenido |
|---|---|
| **ID** | ARQ-21 |
| **Dominio** | D2 |
| **Estado** | **Pendiente de contrato de identidad de servicio** (despliegue: **No verificado** — sin acceso a `.env`/producción) |
| **Evidencia** | Canónico D.2 L827-834 (operaciones servicio-a-servicio con "Auth: Token interno": `resolveAgentInstance`, `executionCompleted`, `deductCredits`, `deleteWorkspaceResources`) · `crm/microservices/ai-service/src/infrastructure/crm/client/crm-graphql-client.ts` L118-124 (`Content-Type: application/json` **siempre**; `Authorization: Bearer <CRM_API_KEY>` solo `if (this.apiKey)` — `config.crm.apiKey = process.env.CRM_API_KEY` en `crm/microservices/ai-service/src/infrastructure/config/config-manager.ts` L197; **no envía `x-org-id` ni `x-wrk-id`**) · `crm/src/server.ts` L236-249 + `crm/src/auth/resolveAuthContext.ts` L96-117 (contexto `/graphql`: lee `Authorization` como Bearer; `jwt.verify(token, ACCESS_TOKEN_SECRET)` + `accesoVigente`; espera **JWT de usuario**, **sin rama de API key/servicio**; sin `x-org-id` no se resuelve orgRole) · `crm/src/graphql/Auth/AuthChecker.ts` L14-16, L27-29 (sin usuario u orgRole → `false`, incluso con `@Authorized()` vacío) · `crm/src/auth/resolveAuthContext.ts` L107-111 (además exige sesión vigente `accesoVigente`, incompatible con un token de servicio estático) · Dirección CRM → ai-service: `crm/microservices/ai-service/src/presentation/http/middleware/auth.middleware.ts` L7-41 valida `Authorization: Bearer <AI_SERVICE_INTERNAL_TOKEN>` (`config.http.internalToken = process.env.AI_SERVICE_INTERNAL_TOKEN` en `config-manager.ts` L203); en producción sin token devuelve 503. · ai-service consulta `GetContactosWorks`/`GetContactoByID` (`crm-contact-gateway.ts:22-103`), resolvers protegidos con `wrk.contactsdb.read`/`wrk.contacts.read` (`crm/src/graphql/resolvers/Contacto.resolver.ts:81, 238`) · AUD-02 (token compartido en la dirección ai-service→CRM entrante) referenciado, no repetido |
| **Hallazgo** | **Dirección ai-service → CRM:** el cliente saliente envía `Content-Type` siempre y `Authorization: Bearer <CRM_API_KEY>` solo si `this.apiKey` es truthy (valor de `process.env.CRM_API_KEY`); **no envía `x-org-id` ni `x-wrk-id`**. CRM no tiene camino de autenticación de servicio: toda operación GraphQL exige un JWT de usuario firmado con `ACCESS_TOKEN_SECRET` (`jwt.verify` en `resolveAuthContext.ts` L99) más sesión vigente (`accesoVigente`, L107-111) y, para permisos `wrk.*`, membresía de workspace. Si `process.env.CRM_API_KEY` no es un JWT válido firmado con `ACCESS_TOKEN_SECRET`, `jwt.verify` lo rechazará y AuthChecker devolvería `false` (sin orgRole); si sí lo fuera, sería tratado como credencial de usuario. No se conoce el valor real de `CRM_API_KEY` en producción. **Dirección CRM → ai-service:** `auth.middleware.ts` L7-41 valida `Authorization: Bearer <AI_SERVICE_INTERNAL_TOKEN>` (`process.env.AI_SERVICE_INTERNAL_TOKEN`); en producción sin este token devuelve 503. Los dos sentidos usan mecanismos de autenticación distintos. **Verificable localmente:** headers enviados por ai-service, mecanismo de JWT en CRM, middleware de token interno en ai-service, ausencia de rama de servicio en CRM. **Depende de despliegue (no verificable localmente):** si `CRM_API_KEY` tiene valor en producción, rotación del token interno, comportamiento real en entorno productivo. No se afirma que las llamadas necesariamente fallen ni que funcionen en producción sin evidencia de despliegue. |
| **Impacto** | **Alto** |
| **Bloquea WAB-78** | **Sí** |
| **Justificación** | La fila "runtime" de la matriz D2 y la tabla "Operaciones internas" de D.2 (L827-834) no pueden especificar actor, permiso ni ámbito sin una identidad de servicio definida; aprobar D.2 con "Token interno" tal cual trasladaría a WAB-80/WAB-81 una decisión arquitectónica crítica de autenticación entre servicios (plan §1). |
| **Destino** | **WAB-78** (definir el contrato de identidad de servicio) → **WAB-80**/**WAB-81** |
| **Recomendación** | Definir cómo se autentica cada operación interna de D.2 (identidad de servicio con claims/headers de tenant, u otro mecanismo existente verificado), quién la valida y cómo se impide el acceso a otro tenant sin depender solo de un secreto compartido. **Criterio verificable:** cada fila de D.2 L827-834 declara mecanismo de autenticación, claims/headers portados y prueba de que un llamador sin tenant no puede operar recursos ajenos; en despliegue queda como **No verificado** hasta evidencia de producción. |

---

## 5. Actualización de hallazgos existentes (sin nuevo ID, sin renumerar)

### ARQ-03 (D2) — validado y ajustado en esta sesión

- **Estado:** pasa de "Preliminar — sin validar" a **validado en D2** (conserva ID, origen y procedencia).
- **Impacto/Bloquea:** se confirman **Crítico** y **Sí**, ahora con justificación basada en contrato (no en ausencia de código): sin matriz P3, sin permisos representables (ARQ-20) y con `bot.configure` asumido pese a la prohibición del registro §6, WAB-80 no puede implementar provisioning/activación/suspensión/configuración con seguridad.
- **Evidencia nueva incorporada:** propuesta canónica D.2/D.4/D.5 (L799-868); atajo `org.*.*` (`hasPermission.ts:45`) y namespace desconocido (`:77-78`); ausencia de actor Waby (`AuthChecker.ts:27-29`); 12 tipos de `ResourceOwnership` sin `AGENT_INSTANCE` (L33-45) y mecanismo reutilizable para servicios (L160-190).
- **Se mantiene registrado en D1 §5 (contradicción 6)** sin reabrir nada.

### ARQ-04 (D2) — validado con aclaraciones en esta sesión

- **Estado:** pasa de "Preliminar — sin validar" a **validado en D2 (parcial)**; clasificación **Medio / No bloquea** sin cambios.
- **Detalle por hallazgo:** H1 vigente (sin `ref_organizacion`; no se introduce automáticamente); H4 parcialmente corregido con evidencia nueva (`wrk.media.read` **sí** se exige en `Media.resolver.ts:14`; siguen muertos `wrk.workspaces.read`, `wrk.media.create/delete`, `wrk.users.*`); H5 vigente y **separado de transferencias de conversaciones**; H7 vigente, con el riesgo de sockets abiertos tras revocación explicitado (§3.2).
- **Destino:** WAB-80 (verificación), más actualización documental de AUD/`permisos-roles.md` donde sus afirmaciones quedaron desactualizadas.

---

## 6. Contradicciones detectadas (registradas, no resueltas unilateralmente)

| # | Contradicción | Evidencia | Impacto | Recomendación (sin ejecutar) |
|---|---|---|---|---|
| 1 | D.1/D.4/D.5 asumen `bot.configure` vs P3/§6 que lo prohíbe (contradicción 6 de D1) | ARQ-03 (evidencia añadida), §3.5 | Crítico (preliminar en D1) | Opción A/B/C de §3.5 → decisión P3 |
| 2 | D.2/D.4 reservan ciclo de vida (y lectura) de instancias a Waby "❌ prohibido" vs administración delegada/checkbox P2/WAB-80/aclaración previa | **ARQ-19** | Alto (bloquea) | Decidir actor en la matriz P3 antes de aprobar D.4 |
| 3 | Permisos propuestos (`platform.*`, `bot.configure`, `wrk.agent.use`, `wrk.credits.read`) vs modelo real (BR-PERM-001, `hasPermission`, `AuthChecker`) | **ARQ-20** | Alto (bloquea) | Rediseñar en P3 con formato `wrk.*` delegable y camino de plataforma explícito |
| 4 | D.2 declara "Token interno" para operaciones servicio-a-servicio vs auth real de CRM (JWT de usuario + orgRole + headers de tenant) | **ARQ-21** | Alto (bloquea) | Definir identidad de servicio antes de aprobar la tabla interna |
| 5 | D.4 reserva proveedor/modelo/credenciales a Waby vs superficie actual que los entrega el cliente | D.4 L855 vs `AiBot.resolver.ts:28-37` y AUD-03 | Alto | WAB-80 debe retirar/restringir la superficie actual (AUD-03 ya la registró; aquí solo se constata para D2) |

---

## 7. Clasificación: bloqueantes y diferibles (D2)

**Bloqueantes vigentes para la aprobación de WAB-78 (con criterio del plan §8):**

| ID | Por qué bloquea (dependencia concreta) |
|---|---|
| **ARQ-03** (validado) | Sin contrato de permisos/ownership/actor para las operaciones de D2, WAB-80 no puede iniciar su matriz de autorización |
| **ARQ-19** | Sin actor decidido, las filas 1, 4, 5, 6 y 10 de la matriz no pueden completarse |
| **ARQ-20** | Los permisos propuestos no son representables con seguridad: WAB-80 los implementaría escalables o no implementables |
| **ARQ-21** | Sin identidad de servicio, la fila runtime y la tabla interna de D.2 no tienen actor/ámbito |
| **ARQ-13 / ARQ-14** (D1, referenciados, sin duplicar) | Siguen vigentes: ARQ-13 además es prerequisito de la fila 10 junto con ARQ-19; ARQ-14 condiciona que T3 pueda validar instancias (§3.4) |

**Diferibles (no bloquean; con destino y criterio):**

- **ARQ-04** completo (H1, H4, H5, H7): Medio, WAB-80/decisión; con las aclaraciones aplicadas en §3.2.
- Contrato de **notificación/invalidación a sesiones WebSocket abiertas** ante suspensión/retiro de instancia o revocación de membresía (H7): definir en WAB-80/D4, apoyado en el mecanismo existente `socketsDeSesion.ts`.
- Actualización documental de AUD §11.2 / `permisos-roles.md` H2 (`wrk.media.read` ya exigido) y de la clasificación de AUD-02 sobre el ámbito de `wrk*` (corrección con evidencia, sin reabrir la auditoría).

**No se bloquea por:** ausencia de `AgentInstance`/`AGENT_INSTANCE`/permisos nuevos en código (esperado antes de WAB-79/80, plan §6).

---

## 8. Decisiones que requieren al usuario (registrar; no solicitar en sesión)

1. **Actor del ciclo de vida de AgentInstance** (ARQ-19): alcance exacto de "Waby exclusivo" vs operaciones delegadas a usuarios del Workspace, incluida la lectura de instancias (D.2 L814) y el checkbox `isDefault` de P2 §4.1.
2. **Diseño de permisos de P3** (ARQ-20): formato y actor de los permisos de plataforma; destino de `bot.configure` (opciones A/B/C de §3.5).
3. **Identidad de servicio para operaciones internas** (ARQ-21).
4. Pendientes **ya registrados** que aquí solo se referencian: P2 §4.1 (`isDefault`, ARQ-13), OPEN-09 (roles globales, H1).

---

## 9. Riesgos para la aprobación de WAB-78

- Aprobar D.2/D.4/D.5 **tal cual** incorporaría al diseño: un actor de ciclo de vida sin definir (ARQ-19), permisos con ruta de escalada vía `org.*.*` y violación de BR-PERM-001 (ARQ-20), una autenticación interna inexistente (ARQ-21) y un permiso prohibido por P3 presentado como existente (contradicción 6).
- La separación Waby/cliente de D.4 quedaría como tabla declarativa sin mecanismo: la superficie actual (`AiBot.resolver` sin T3, `apiKey` del cliente, token compartido de ai-service) seguiría operativa si WAB-80 no la retira explícitamente.
- Riesgo de confusión de ámbitos al implementar: los nombres `wrkCreate/wrkRead/wrkUpdate/wrkDelete` del enum valen `org.workspaces.*` (corrección a AUD-02 con evidencia); replicar ese patrón en los permisos nuevos produciría gates de organización donde se esperaba workspace.

---

## 10. Qué debe leer la próxima sesión (D3 — Bloque 3)

1. `informe-bloque-2.md` (este) y `matriz-acumulativa.md` actualizada (ARQ-19…ARQ-21; ARQ-03/04 validados).
2. Plan §4 Bloque 3 y §6 (reutilizar ARQ-05…ARQ-09 sin redeclararlos todos como bloqueantes).
3. Superficie de asignación relevante para D3: `AiContactAssignment.resolver.ts` (`assignContact`/`reassignContact`/`unassignContact` con `org.workspaces.*`, `workspaceId` del cliente, sin T3) y `Contacto.resolver.ts:855` (asignación humana con `wrk.contacts.update`); AUD-05 (runtime, capabilities globales, `workspaceId` sin validar en entrada) y AUD-01 (persistencia) como evidencia ya publicada.
4. Evidencia del repo local de ai-service para revisitar ARQ-07 (observación D1 en `analisis-previo-d2-d4.md`).
5. Registro §8.1/§8.2 y canónico D.1.5-D.1.8 (EvAssigned, transacción, ABA).

---

## 11. Cierre de la sesión

- **Qué se cambió:** se creó `F:\ElemenSoft\docs\revision-wab-78\informe-bloque-2.md` (este informe) y se actualizó `F:\ElemenSoft\docs\revision-wab-78\matriz-acumulativa.md` (ARQ-03/ARQ-04 validados, ARQ-19…ARQ-21 añadidos, Bloque 2 → Revisado, siguiente ID ARQ-22). Ningún archivo fuera de esa carpeta fue modificado: sin cambios de código, permisos, roles, seeds, tests, canónicos ni registro de decisiones.
- **Qué se verificó:** cuestiones §6.2 respondidas con evidencia citada por archivo/línea; estados en Linear **intactos** (WAB-78 In Progress, WAB-79 Backlog/bloqueada — solo lectura); ARQ-03/ARQ-04 conservados sin renumerar; IDs nuevos desde ARQ-19; AUD-01..06 no repetidos (reutilizados como evidencia); P1/P2/P4/P9/P10 no reabiertos; `bot.configure`/`AGENT_INSTANCE`/`platform.*` no introducidos (solo propuestas marcadas como pendientes).
- **Scope:** D2 completo en diagnóstico; D3/D4 solo referenciados (ARQ-05…ARQ-10 sin validar aquí).
- **Bloqueantes D2 para la aprobación de WAB-78:** ARQ-03, ARQ-19, ARQ-20, ARQ-21 (más los de D1: ARQ-13, ARQ-14). Diferibles: ARQ-04 y contrato H7. **No se emite veredicto** (plan §9: solo al final de bloques + integración).
- **Pendiente humano (no solicitado en sesión):** decisiones de la §8, especialmente el actor del ciclo de vida y el diseño P3 de permisos.

