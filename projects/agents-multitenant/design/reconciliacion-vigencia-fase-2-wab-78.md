# Reconciliación de vigencia del diseño — WAB-75 / WAB-78

**Fecha de corte:** 06/11/2026  
**Estado:** registro de reconciliación documental; no sustituye todavía al documento integral de diseño de Fase 2.  
**Autoridad operativa:** Linear WAB-78 y decisiones aprobadas enlazadas abajo.  
**Destino documental:** `sinies7ro/waby-docs`.

> Este documento fija qué contenido de los borradores locales sigue vigente y qué contenido debe reemplazarse. No constituye aprobación integral de WAB-78, no autoriza implementación y no habilita el inicio de WAB-79.

## 1. Fuentes y precedencia

En caso de contradicción, aplicar este orden:

1. Decisiones expresamente aprobadas por el usuario y registradas en Linear.
2. Contratos y registros de decisión vigentes en `projects/agents-multitenant/design/`.
3. Documento integral de Fase 2, una vez reconciliado con las fuentes anteriores.
4. Adendas técnicas, solo en los puntos compatibles con las decisiones aprobadas.
5. Borradores iniciales y auditorías históricas, que no prevalecen sobre decisiones posteriores.

Linear es la fuente operativa para estados, dependencias y seguimiento. GitHub es la fuente de los documentos versionados. Los repositorios de implementación conservan únicamente documentación específica de cada componente.

## 2. Decisiones vigentes que el documento integral debe reflejar

### 2.1 Administración y autoridad — ARQ-19

Modelo híbrido por operación:

- **Waby:** administra AgentDefinitions y catálogos centrales; proveedores, modelos y credenciales LLM; políticas obligatorias; provisioning técnico; operaciones de plataforma y purga física conforme a política.
- **Workspace:** usuarios humanos autorizados administran sus AgentInstances y su ciclo de vida lógico, configuración empresarial, Skills permitidas, integraciones de instancia y designación predeterminada.
- **CRM:** autoridad de dominio para ContactAssignment y ownership. ai-service puede mantener proyecciones operativas derivadas, nunca una autoridad editable paralela.
- Los servicios ejecutan operaciones técnicas; no reemplazan al actor de negocio. El LLM nunca es principal administrativo.

No describir la creación, activación, suspensión o retiro lógico de AgentInstance como operaciones exclusivamente administradas por Waby. Distinguir siempre el provisioning técnico bajo control de plataforma de la administración empresarial de la instancia por el Workspace.

### 2.2 Permisos — ARQ-20

- Administración de AgentInstance: `wrk.agents.*`.
- Lectura de AgentInstance: `wrk.agents.read`.
- Los permisos se declaran explícitamente en `permission.enum.ts`; el wildcard no sustituye su declaración.
- Workspace Owner hereda `wrk.agents.*` mediante `wrk.*.*`; Workspace Manager recibe `wrk.agents.*` explícito; Workspace Agent recibe `wrk.agents.read`.
- La administración de la instancia comprende configuración, Skills, Rules e integraciones permitidas; no se inventan permisos separados por subrecurso.
- Estos permisos no autorizan administrar AgentDefinitions, políticas o recursos centrales de Waby y no sustituyen ownership, membership ni autorización contextual de Capabilities.
- Asignación, reasignación y liberación manual de contactos: permiso CRM existente `wrk.workspaces.update`.
- Asignación inicial automática: operación interna de CRM, autenticada, validada y auditada.

No presentar `platform.*`, `wrk.agent.use`, `wrk.credits.read` ni `bot.configure` como permisos vigentes del contrato de administración de instancias.

### 2.3 Autenticación entre servicios — ARQ-21

Contrato acordado: JWT de servicio firmados RS256, de corta duración, con credenciales diferenciadas por dirección (CRM → ai-service y ai-service → CRM), token por petición y scope mínimo.

Claims acordados: `iss`, `sub`, `aud`, `iat`, `nbf`, `exp`, `jti`, `scope`, `orgId`, `workspaceId`, `agentInstanceId`, `initiatedBy`.

El receptor valida firma/algoritmo, issuer, subject esperado, audience, vigencia, scope y contexto tenant/recurso; además verifica ownership y estado actuales en CRM. `initiatedBy` es dato de auditoría, no principal ni autorización.

Siguen pendientes de detalle en WAB-78: inventario exhaustivo de operaciones internas, scope por operación, obligatoriedad de claims por flujo, provisión/rotación/revocación de claves, semántica anti-replay de `jti`, autenticación RabbitMQ, pruebas negativas y despliegue. No presentar `jsonwebtoken` como dependencia aprobada.

### 2.4 Decisiones operativas AgentInstance — DEC-CM-01 a DEC-CM-08

1. AgentInstance mantiene referencia estable a AgentDefinition. Cada ejecución nueva resuelve la última versión publicada; una ejecución en curso conserva su versión y snapshot efectivos.
2. `additionalPrompt` es configuración propia de la instancia, validada y limitada. No reemplaza el prompt central ni políticas; RAG sigue siendo dato no confiable.
3. Proveedor, modelo, credenciales y parámetros técnicos base son controlados por Waby. Allowlist de parámetros por instancia vacía por defecto.
4. No existe routing persistente canal → instancia ni entidad AgentRouting. El canal es contexto del mensaje/Execution; la resolución usa ContactAssignment y, si falta asignación activa, la instancia predeterminada.
5. Cambiar AgentDefinition es explícito, autorizado y auditado. Una incompatibilidad suspende la instancia hasta corrección por usuario autorizado. Cambios relevantes se notifican a Workspaces afectados.
6. CRM es única fuente de verdad de ContactAssignment. Precedencia: asignación humana vigente (IA no interviene); asignación vigente a AgentInstance activa (se conserva); sin asignación activa, ante nuevo mensaje entrante, CRM asigna la instancia predeterminada si está activa; si no existe o no está activa, queda sin asignación. Sin fallback a otra instancia.
7. Suspender una instancia conserva asignaciones activas. La instancia suspendida no ejecuta; no se libera ni se reasigna automáticamente. La atención automática solo vuelve al reactivarse esa instancia o tras reasignación manual explícita.
8. Como máximo una AgentInstance predeterminada por Workspace. El cambio es atómico y no modifica asignaciones existentes; solo afecta contactos sin asignación activa al evaluar un nuevo mensaje.

CRM conserva ContactAssignment como autoridad. ai-service puede mantener una proyección derivada sincronizada mediante Outbox. Contacto y AgentInstance deben pertenecer al mismo Workspace.

### 2.5 Créditos — P4 aprobada

- Una única bolsa `OrganizationAICredit` por Organización, compartida entre Workspaces. Workspace es dimensión de atribución, no bolsa ni límite individual.
- `CreditLedger` append-only; no existen asientos `hold`/`release`.
- `committedBudget` es compromiso operativo temporal fuera del ledger.
- Presupuesto máximo por ejecución y asignación independiente por invocación LLM.
- Fencing individual por autorización y asignación. Renovar lease conserva generación; recuperar/reasignar a otro worker la incrementa. Escrituras de asignación validan también fencing/ownership de autorización padre.
- CRM es autoridad transaccional del saldo. ai-service ejecuta, reporta usage y solicita operaciones autorizadas; no valida ni autoriza el saldo como autoridad.
- Débito final único por `executionId` cuando todas las invocaciones están resueltas. Sin débito de importe cero.
- Política híbrida de G.4.6 aprobada. Período de gracia de 30 días naturales para `reconciliation_required`; no hay débito retroactivo automático.
- P4 cubre exclusivamente consumo LLM. Skills HTTP y operaciones no-LLM quedan fuera.

La adenda técnica P4 se utiliza como material para completar contratos, fórmulas, transiciones, fencing, leases, idempotencia, recuperación, reconciliación y criterios de aceptación en el cuerpo integral y ADR-002/ADR-006. No mantenerla como segundo contrato canónico. Sus afirmaciones antiguas de política pendiente, fases pendientes o estado de WAB-76 se consideran superadas.

## 3. Reconciliaciones obligatorias sobre el borrador integral

| Sección del borrador | Tratamiento requerido |
|---|---|
| A.2 AgentInstance | Mantener referencia estable a AgentDefinition e incluir `additionalPrompt`; retirar `channelAssignment` y `priority` como mecanismos de routing; revisar cualquier `systemPromptOverride` y sustituirlo por `additionalPrompt`. Mantener `isDefault` con unicidad por Workspace. |
| C.1 / C.3 Provisioning | Separar provisioning técnico Waby de administración empresarial de AgentInstance por usuarios autorizados del Workspace. |
| C.2 y C.2.1–C.2.4 | Retirar cupos comerciales 1/2, downgrade, gracia de 5 días, `retirement_pending` derivado de exceso y selección automática por prioridad. No existe límite comercial de cantidad de instancias por Workspace/Organización. |
| C.4–C.6 | Quitar supuestos de routing por canal, cupo y ciclo de retiro por downgrade. Reconciliar estados de ciclo de vida con ARQ-19 y DEC-CM-07; no inventar transiciones no aprobadas. |
| D.1 | CRM es autoridad de saldo y asignación; ai-service reporta consumo y mantiene solo proyecciones operativas derivadas. |
| D.2–D.5 | Sustituir permisos y operaciones obsoletos por ARQ-20. No asignar a Waby en exclusiva operaciones de instancia que corresponden al Workspace. |
| E.2–E.3 | Sustituir canal → instancia, `channelAssignment` y `priority` por resolución vía ContactAssignment y default activa conforme DEC-CM-06/08. |
| E.4 | Preservar los acuerdos funcionales de delegación, takeover, transferencia, equipos y concurrencia que siguen vigentes; reconciliar su punto de entrada con ContactAssignment y DEC-CM-06/07. No borrar toda la sección por el cambio de routing inicial. |
| F | `additionalPrompt` no puede reemplazar prompt central/políticas. Knowledge, memoria, historial, inputs y resultados de tools son datos no confiables. |
| G.4 y ADR-002/ADR-006 | Mantener la política aprobada y absorber el diseño técnico P4. Corregir la responsabilidad de CRM/ai-service y retirar cualquier estado de decisión pendiente ya resuelto. |
| ADR-004 | Reescribir: no routing persistente por Workspace + canal. La selección se basa en ContactAssignment y default activa. |
| ADR-001/ADR-003/ADR-005 | Revalidar su estado y contenido contra las decisiones vigentes. DEC-CM-01 define resolución de última versión publicada para nuevas ejecuciones y snapshot estable para ejecuciones en curso; no inferir aprobación de detalles de persistencia no acordados. |
| K / L | Eliminar dependencias de P1/P2/P3 como decisiones pendientes cuando ya fueron resueltas. Separar diseño documentado de implementación futura. No marcar la Fase 2 como aprobada/cerrada: WAB-78 permanece In Progress. |

## 4. Estado y límites de aprobación

**Aprobado:** ARQ-19, ARQ-20, ARQ-21 como dirección/contrato base; DEC-CM-01 a DEC-CM-08; P4 y política híbrida de facturación descrita en G.4.6.

**Pendiente dentro de WAB-78:** inventario y scopes completos de operaciones internas; detalle de seguridad/rotación/replay y RabbitMQ; contrato técnico completo de ejecución, vigencia y cancelación; consistencia integral de matrices, diagramas, entidades y ADRs; resolución explícita de aspectos de P9/P10 que no estén cubiertos por decisiones aprobadas; revisión final del diseño completo.

No convertir propuestas D5 ni detalles técnicos pendientes en decisiones aprobadas por inferencia. Mantener separados los acuerdos funcionales y los mecanismos de implementación.

## 5. Estado operativo y siguiente paso

- WAB-76: Done.
- WAB-77: Done.
- WAB-78: In Progress; no cerrar hasta revisión y autorización expresa del diseño completo.
- WAB-79: Backlog, bloqueada por WAB-78; no iniciar.
- WAB-80, WAB-81 y WAB-82: Backlog.

El siguiente trabajo es incorporar este registro al documento integral de Fase 2, sección por sección, integrando la adenda P4 y preservando el contenido válido. La auditoría WAB-77 debe migrarse íntegra, sin reescritura, como evidencia histórica independiente.

## 6. Referencias canónicas

- [ARQ-19/20/21 — Configuración efectiva y capacidades](./configuracion-efectiva-y-capacidades.md)
- [DEC-CM-01 a DEC-CM-08 — Decisiones operativas de AgentInstance](./decisiones-operativas-agent-instance.md)
- [WAB-78 — Fase 2 en Linear](https://linear.app/waby/issue/WAB-78/fase-2-diseno-objetivo-y-contratos)
- [WAB-77 — Auditoría en Linear](https://linear.app/waby/issue/WAB-77/fase-1-auditoria-del-sistema-actual-sin-modificar-codigo)

**Nota de integridad documental:** el documento integral original de Fase 2 y el informe completo WAB-77 fueron recibidos como adjuntos de una sesión anterior, no están todavía versionados en este repositorio. Este registro no suplanta esos archivos ni afirma que su migración íntegra ya se haya realizado.
