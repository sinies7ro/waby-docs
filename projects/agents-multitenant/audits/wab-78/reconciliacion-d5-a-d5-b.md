# WAB-78 — Reconciliación documental D5-A + D5-B

**Estado:** documento de trabajo de reconciliación (no canónico).  
**Ruta:** `projects/agents-multitenant/audits/wab-78/reconciliacion-d5-a-d5-b.md`  
**Fecha:** 05/11/2026  
**Modalidad:** Solo lectura sobre `crm` y sobre los informes originales. No se modifica código, matriz, documentos canónicos ni Linear. No se crea nuevo ID ARQ.

---

## 1. Objetivo y alcance

### 1.1 Objetivo

Realizar una reconciliación técnica y documental de los diagnósticos D5-A y D5-B para determinar:

- Qué hallazgos están confirmados con evidencia en código.
- Cuáles hallazgos se solapan entre ambos informes.
- Qué contratos de seguridad faltan definir en WAB-78.
- Qué asuntos deben diferirse a D5-C o a fases posteriores de implementación.

El resultado permite actualizar posteriormente la matriz acumulativa de WAB-78 sin duplicar hallazgos ni confundir decisiones arquitectónicas con tareas de implementación.

### 1.2 Alcance

- Se inspeccionan los informes de la carpeta `projects/agents-multitenant/audits/wab-78/` de `waby-docs`.
- Se usa el código de `crm/` (rama `master`, commit `b9e0aee`) únicamente para comprobar referencias concretas citadas en los informes.
- No se repiten auditorías D1-D4, D5-A ni D5-B. No se inicia D5-C.
- Se respetan las decisiones de línea base (P1, P2, P4, P9, P10, ownership de AgentInstance, P3 sin `bot.configure`) registradas en `00-linea-base.md`.

### 1.3 Fuera de alcance

- No se modifica `informe-bloque-5a.md` ni `informe-bloque-5b.md` originales.
- No se modifica `matriz-acumulativa.md`.
- No se crea nuevo identificador ARQ.
- No se declara aprobado ningún contrato que requiera decisión del usuario.
- No se afirma que una ausencia de programación implique necesariamente un fallo en producción.

---

## 2. Fuentes consultadas

| Fuente | Ruta | Uso en esta reconciliación |
|--------|------|---------------------------|
| Informe D5-B | `informe-bloque-5b.md` | Hallazgos 4.1-4.20, brechas B1-B13, escenarios R1-R13, contratos C1-C11, fases de implementación, sección 9 y 10. |
| Informe D4-A (runtime) | `informe-d4-a-runtime.md` | Menciones a D5-A como bloque futuro (Policy Engine, output filter, prompt injection, ARQ-10). Contratos C1-C10 pendientes. Exclusiones de D4-A. |
| Informe D4-B (contexto) | `informe-d4-b-contexto.md` | Menciones a D5-A como bloque futuro. Exclusiones de D4-B. |
| Línea base WAB-78 | `00-linea-base.md` | Decisiones aprobadas (P1, P2, P4, P9, P10, P3), convención de IDs, estados de control. |
| Matriz acumulativa | `matriz-acumulativa.md` | Fila ARQ-10 (Policy Engine / output filter), estados, reglas de no-duplicación. |
| Código `crm/` | `crm/microservices/ai-service/src/` | Verificación puntual de referencias citadas en D5-B (IPolicyEngine comentario, findAll, role system, catch return [], enabledCapabilities, etc.). |

### 2.1 Nota sobre D5-A

En el directorio `projects/agents-multitenant/audits/wab-78/` de `waby-docs` **no existe un archivo `informe-bloque-5a.md`**. Solo se recuperó `informe-bloque-5b.md` (D5-B). Los informes D4-A y D4-B mencionan a D5-A como un bloque futuro fuera de su scope (Policy Engine, output filter, prompt injection, ARQ-10), y el encargo de este trabajo da por sentado que D5-A identificó la ausencia de Policy Engine y Output Filter en el runtime y propuso tres puntos de control independientes.

Por tanto, en esta reconciliación:

- Las referencias a **D5-A** se toman del contexto de decisión del encargo y de las menciones en D4-A/D4-B, no de un archivo local denominado `informe-bloque-5a.md`.
- Las referencias a **D5-B** se toman directamente de `informe-bloque-5b.md`, con evidencia verificada en código cuando procede.

---

## 3. Resumen ejecutivo

D5-A y D5-B no son contradictorios: D5-B **confirma y amplía** las ausencias que D5-A había identificado (ausencia de Policy Engine y Output Filter, propuesta de puntos de control pre-LLM, post-LLM y pre-capability). D5-B añade evidencia concreta de código para cada ausencia, clasifica brechas por severidad, define escenarios de ataque y propone cinco fases de implementación.

La reconciliación arroja los siguientes resultados principales:

1. **Hallazgos confirmados:** ausencia de Policy Engine, ausencia de Output Filter, ausencia de Domain Classifier, RAG fail-open sin obligatoriedad de grounding, knowledge fragments como system con misma prioridad, capabilities sin filtrar por `enabledCapabilities`, falta de sanitización de documentos, falta de distinción error-servicio vs. sin resultados en RAG, falta de validación determinista de respuestas (grounding), y varios controles parciales existentes.
2. **Solapamientos:** D5-A (puntos de control pre/post LLM y pre-capability) se solapa parcialmente con D5-B (contratos C1 Policy Engine, C3 Output Filter, C4 Grounding Verifier, C7 filtrado de capabilities, C8 separación de prioridad). Ningún hallazgo de D5-B contradice a D5-A; D5-B amplía con evidencia y escenarios.
3. **ARQ-10:** sigue como **Preliminar — sin validar** en la matriz acumulativa. WAB-78 debe definir los contratos de seguridad (Policy Engine, Output Filter, restricción de dominio, grounding/RAG, jerarquía de contexto, memoria y confidencialidad) sin que su cierre dependa de la implementación completa.
4. **Reservado para D5-C:** autorización del CapabilityExecutor y carga/filtrado de capabilities, aislamiento multitenant de datos sensibles, logs y telemetría. No anticipar este diagnóstico.
5. **Decisiones que requieren aprobación del usuario:** definición de cada contrato de seguridad (C1-C11 de D5-B adaptados), clasificación de qué bloquea vs. qué no bloquea el cierre de WAB-78, y destino de cada contrato (WAB-78 vs. fase posterior vs. D5-C).

---

## 4. Tabla de correspondencia D5-A ↔ D5-B

Como no existe el archivo `informe-bloque-5a.md`, las referencias a D5-A se anotan como **D5-A (contexto del encargo / menciones en D4-A y D4-B)**. Cada fila indica si el hallazgo de D5-B confirma, amplía, solapa o no contradice a D5-A.

| # | Referencia D5-A | Referencia D5-B | Evidencia y ruta de código | Qué confirma | Relación (duplica / amplía / solapa / contradice) | Relación ARQ existentes (especialmente ARQ-10) | Clasificación propuesta |
|---|-----------------|-----------------|---------------------------|--------------|--------------------------------------------------|------------------------------------------------|--------------------------|
| 1 | Ausencia de Policy Engine (contexto del encargo) | Hallazgo 4.1 «Ausencia total de Policy Engine» (CRÍTICO); Brecha B1 | `src/application/services/process-message.service.ts:24` (comentario "Policy evaluation (that will live in IPolicyEngine, Sprint 2.5+)"); búsqueda en todo `ai-service` sin clase/interfaz `IPolicyEngine` | Confirma la ausencia de Policy Engine que D5-A ya había identificado | Confirma (D5-B aporta evidencia concreta) | ARQ-10 (Policy Engine y output filter no existen); confirma que ARQ-10 sigue vigente | Confirmado |
| 2 | Ausencia de Output Filter (contexto del encargo) | Hallazgo 4.2 «No hay validación ni filtro de la respuesta del LLM antes de enviarla al usuario» (CRÍTICO); Brecha B2 | `src/engine/execution-engine.ts:93-100` (`markCompleted(response.content)` directo); `src/application/services/process-message.service.ts:42-50` (usa `response.content` sin inspección) | Confirma la ausencia de Output Filter que D5-A había identificado | Confirma (D5-B aporta evidencia concreta) | ARQ-10; confirma que ARQ-10 sigue vigente | Confirmado |
| 3 | Punto de control antes de cada invocación al LLM (D5-A) | Hallazgo 4.3 «No hay mecanismo de clasificación de consultas» (CRÍTICO); Brecha B3; Contrato C2 Domain Classifier | No existe `DomainClassifier` ni equivalente en el código | Confirma que no hay gate de clasificación de dominio antes del LLM | Amplía D5-A (especifica el mecanismo de clasificación como punto de control) | Relacionado con ARQ-10 (Policy Engine evalúa políticas antes del LLM) | Confirmado + ampliación |
| 4 | Punto de control después de la respuesta del LLM y antes de devolverla (D5-A) | Hallazgo 4.2 (Output Filter); Brecha B2; Contrato C3 Output Filter | `src/engine/execution-engine.ts:93-100`; `src/application/services/process-message.service.ts:42-50` | Confirma la ausencia de filtro post-LLM | Confirma (D5-B mapea el punto de control de D5-A a Output Filter) | ARQ-10 | Confirmado |
| 5 | Punto de control antes de cada Capability o efecto externo (D5-A) | Hallazgo 4.6 «Todas las capabilities se cargan sin filtrar por enabledCapabilities» (ALTA); Brecha B6; Contrato C7 Filtrado de capabilities | `src/engine/execution-engine.ts:65-67` (`capabilityRegistry.findAll()` sin filtro); `src/domain/value-objects/configuration/effective-config.ts:38` (`enabledCapabilities` existe pero no se usa para filtrar) | Confirma que no hay control previo a capabilities | Amplía D5-A (especifica el filtrado de capabilities como punto de control) | Relacionado con D5-C (autorización de CapabilityExecutor reservado a D5-C) | Confirmado + ampliación |
| 6 | Políticas obligatorias propiedad de Waby (contexto del encargo) | Hallazgo 4.10 «Intento de ignorar políticas globales» (CRÍTICO); Escenario 5.10 | Ausencia de `IPolicyEngine` en el código | Confirma que no hay autoridad de políticas globales prevalecientes | Amplía D5-A (define propiedad y prevalencia de políticas) | ARQ-10 | Confirmado + ampliación |
| 7 | Datos no confiables: Knowledge, memoria, historial, entradas de usuario, resultados de herramientas (contexto del encargo) | Hallazgo 4.5 «Knowledge fragments inyectados como mensajes de sistema» (ALTA); Brecha B5; Hallazgo 4.7 «Extracción de memoria vía LLM sin control de contenido» (MEDIA); Brecha B10; Hallazgo 4.13/4.14 (workspaceId/historial no validados) | `src/infrastructure/providers/openai/openai.mapper.ts:26-32` (knowledge como `role:'system'`); `src/engine/memory-persister.ts:104-117` y línea 149 (`visibility: SHARED`); `src/infrastructure/messaging/consumers/message-received-handler.ts:231-243` y `:200-209` | Confirma tratamiento inseguro de datos no confiables | Amplía D5-A (concretiza vectores de inyección vía conocimiento, memoria e historial) | ARQ-10 (jerarquía de contexto) | Confirmado + ampliación |
| 8 | Fail-closed ante errores, timeouts, versiones inválidas o imposibilidad de evaluar política obligatoria (contexto del encargo) | Hallazgo 4.4 «RAG falla en modo fail-open» (CRÍTICO); Brecha B4; Hallazgo 4.11 «No hay distinción entre ausencia de resultados, resultados irrelevantes y errores de RAG» (ALTA); Brecha B7 | `src/infrastructure/services/rag-service.ts:77-79` (`catch { return [] }`); `src/engine/steps/execute-rag.step.ts:16` (RAG failure is non-fatal) | Confirma fail-open silencioso en RAG (violación del principio fail-closed) | Amplía D5-A (concreta el comportamiento ante fallos de recuperación) | ARQ-10 (comportamiento ante fallos) | Confirmado + ampliación |
| 9 | (no hay hallazgo directo en D5-A) | Hallazgo 4.9 «No hay sanitización de contenido de documentos» (ALTA); Brecha B9 | `src/engine/steps/execute-rag.step.ts:57-64` (fragmentos puros); `src/infrastructure/services/rag-service.ts:71-74` (solo `content`, sin metadatos ni sanitización) | Nuevo hallazgo de D5-B sin contraparte explícita en D5-A | Amplía (defensa frente a prompt injection vía documentos) | ARQ-10 (defensa frente a prompt injection) | Confirmado (D5-B) |
| 10 | (no hay hallazgo directo en D5-A) | Hallazgo 4.12 «No hay mecanismo determinista de validación de respuestas» (CRÍTICO); Brecha B8; Contrato C4 Grounding Verifier | No existe `GroundingVerifier` ni equivalente; búsqueda en todo el código sin resultados | Nuevo hallazgo de D5-B | Amplía (grounding obligatorio y verificable) | ARQ-10 | Confirmado (D5-B) |
| 11 | (no hay hallazgo directo en D5-A) | Hallazgo 4.8 «LoadRulesSkillsStep carga Rules/Skills desde MinIO/cross-DB sin validación de pertenencia al workspace» (MEDIA, inferido) | `src/engine/steps/load-rules-skills.step.ts:125-155` y `fetchContent` :274-302 (solo verifica hash SHA-256) | Nuevo hallazgo de D5-B | Amplía (autorización de contenido recuperado) | Relacionado con D5-C (autorización de capabilities y datos) | Pendiente de D5-C / no verificable en runtime |
| 12 | (no hay hallazgo directo en D5-A) | Hallazgo 4.10 Middleware auth HTTP sin token en no-producción (MEDIA); Brecha B11 | `src/presentation/http/middleware/auth.middleware.ts:10-20` | Nuevo hallazgo de D5-B (afecta solo entornos no-productivos) | Independiente | No pertenece a ARQ-10 | Confirmado (No bloquea WAB-78) |
| 13 | (no hay hallazgo directo en D5-A) | Hallazgos 4.15-4.20 controles existentes (credenciales excluidas de toJSON, RedactSecrets parcial, error middleware genérico, BR-KB-001, D*-cred, vigencia G1/G2) | Rutas citadas en D5-B §4.15-4.20 | Confirma controles existentes positivos y parciales | Independiente (no es ausencia) | No pertenece a ARQ-10 | Confirmado (controles existentes) |

**Duplicados detectados:** Ningún hallazgo de D5-B duplica exactamente otro de D5-B; todos son distintos. D5-A (como contexto) y D5-B se solapan en los cinco puntos de control vs. contratos C1, C2, C3, C4, C7, pero no hay duplicación de IDs ni de hallazgos: D5-B aporta evidencia y escenarios que D5-A (contexto) solo enuncia.

**Contradicciones detectadas:** Ninguna. D5-B no contradice a D5-A; lo confirma y amplía con evidencia de código.

---

## 5. Reconciliación de ARQ-10

La matriz acumulativa (`matriz-acumulativa.md`, fila ARQ-10) registra:

- **Origen:** Análisis previo D2–D4 (usuario).
- **Dominio:** D4.
- **Estado:** Preliminar — sin validar (original: sin implementar, diferible).
- **Evidencia:** Canónico F/G; informe Fase 1 R8/R11; H.1 L1689-1690. Policy Engine y output filter no existen en la implementación inspeccionada.
- **Impacto:** Medio (original, sin validar).
- **Bloquea WAB-78:** No (original, sin validar).
- **Justificación:** Diferible a WAB-81 si contratos, puntos de aplicación y comportamiento ante fallos están definidos (plan §6).
- **Destino:** WAB-81 (implementación) / WAB-78 (contratos).
- **Recomendación:** Validar en sesión D4 que los contratos de seguridad estén definidos en WAB-78.

### 5.1 Evaluación tras D5-A y D5-B

D5-A (contexto del encargo) y D5-B confirman la evidencia base de ARQ-10: **no existe Policy Engine ni Output Filter** en el runtime. D5-B aporta evidencia concreta de código para cada ausencia (sección 4 de esta reconciliación). Por tanto:

- **ARQ-10 sigue vigente como hallazgo de seguridad.** Su clasificación de impacto Medio y su indicador "Bloquea: No" de la matriz original se mantienen como **referencia histórica**, pero la evidencia de D5-B justifica revisar si el cierre de WAB-78 requiere que los contratos de seguridad estén definidos y aprobados (sin requerir su implementación).
- **No crear nuevo ID ARQ.** ARQ-10 cubre Policy Engine y output filter. Los demás contratos de esta reconciliación (restricción de dominio, grounding/RAG, jerarquía de contexto, memoria y confidencialidad) se documentan como contratos asociados a ARQ-10 en WAB-78, sin nuevos identificadores ARQ.
- **Estado recomendado de ARQ-10:** pasar de "Preliminar — sin validar" a **Confirmado** (con la evidencia de D5-B), manteniendo "Bloquea: No" para el cierre de diseño (los contratos sí deben definirse; la implementación no bloquea el cierre de WAB-78). Este cambio de estado requiere aprobación del usuario y se registra aquí como decisión pendiente, no se aplica aún a la matriz.

---

## 6. Contratos de seguridad requeridos en WAB-78

Esta sección consolida los contratos que WAB-78 debe definir y aprobar para cerrar el diseño. Se derivan de D5-B §9 (contratos C1-C11) y del contexto de decisión de D5-A. Ningún contrato se declara aprobado aquí: todos quedan como **decisión pendiente del usuario** hasta que la dirección arquitectónica los apruebe.

### 6.1 A. Policy Engine

| Subcontrato | Qué debe definir WAB-78 | Origen en D5-B | Relación D5-A | Categoría |
|-------------|------------------------|----------------|---------------|-----------|
| A1 | Autoridad y propiedad de las políticas: quién define, aprueba y revoca políticas obligatorias; políticas de Waby vs. configuración empresarial | C1 (Policy Engine) | D5-A: políticas obligatorias son propiedad de Waby | WAB-78 — Contrato obligatorio |
| A2 | Evaluación determinista de políticas obligatorias: puntos de inserción (antes del LLM, después del LLM, antes de capabilities), resultado admitir/rechazar/abstener | C1 | D5-A: tres puntos de control independientes | WAB-78 — Contrato obligatorio |
| A3 | Versionado y trazabilidad de políticas: identificación de versión aplicada por ejecución | C1 | D5-A (implícito en punto de control) | WAB-78 — Contrato obligatorio |
| A4 | Fallos, timeouts y versiones inválidas: conducta fail-closed; no se procede si la política no puede evaluarse | C1 | Contexto del encargo (regla 5) | WAB-78 — Contrato obligatorio |
| A5 | Revocación y precedencia de políticas: jerarquía (políticas globales Waby > system prompt > usuario > documentos) | C1 | D5-A | WAB-78 — Contrato obligatorio |
| A6 | Separación entre políticas de Waby y configuración empresarial (tenant) | C1 | Contexto del encargo (regla 3 y 5) | WAB-78 — Contrato obligatorio |

### 6.2 B. Output Filter

| Subcontrato | Qué debe definir WAB-78 | Origen en D5-B | Relación D5-A | Categoría |
|-------------|------------------------|----------------|---------------|-----------|
| B1 | Punto de aplicación: después de la respuesta del LLM y antes de devolverla o publicarla | C3 (Output Filter) | D5-A: punto de control post-LLM | WAB-78 — Contrato obligatorio |
| B2 | Datos y respuestas que inspecciona: código, prompts internos, secretos, información confidencial de Waby y de cada tenant, grounding | C3 | D5-A | WAB-78 — Contrato obligatorio |
| B3 | Protección de información confidencial de Waby y de cada tenant: no divulgación de arquitectura interna, código fuente, credenciales | C3 | D5-A | WAB-78 — Contrato obligatorio |
| B4 | Tratamiento de respuestas bloqueadas o modificadas: mensaje de bloqueo estándar, reescritura, rechazo | C3 | D5-A | WAB-78 — Contrato obligatorio |
| B5 | Registro de decisiones y motivos: trazabilidad de qué se filtró y por qué, sin datos sensibles en logs | C3 | D5-A | WAB-78 — Contrato obligatorio |
| B6 | Comportamiento ante errores del filtro: fail-closed (no enviar respuesta si el filtro no puede evaluarse) | C3 | Contexto del encargo (regla 5) | WAB-78 — Contrato obligatorio |

### 6.3 C. Restricción de dominio

| Subcontrato | Qué debe definir WAB-78 | Origen en D5-B | Relación D5-A | Categoría |
|-------------|------------------------|----------------|---------------|-----------|
| C1 | Cómo determinar si una consulta pertenece al contexto empresarial autorizado | C2 (Domain Classifier) | D5-A: punto de control pre-LLM | WAB-78 — Contrato obligatorio |
| C2 | Qué papel puede desempeñar un Domain Classifier: clasificación probabilística de consulta (autorizada / sin documentación / fuera de ámbito / confidencial / indeterminada) | C2 | D5-A | WAB-78 — Contrato obligatorio |
| C3 | Por qué un clasificador probabilístico no puede ser autoridad de autorización: el LLM/clasificador no decide qué información puede divulgar; solo el Policy Engine es autoridad determinista | C2 | D5-A | WAB-78 — Contrato obligatorio |
| C4 | Condiciones de abstención: cuándo no responder y mensaje estándar | C5 (Mecanismo de abstención) | D5-A | WAB-78 — Contrato obligatorio |
| C5 | Cómo evitar respuestas basadas exclusivamente en conocimiento preentrenado: grounding obligatorio cuando no hay fragmentos empresariales relevantes | C2, C4 | D5-A | WAB-78 — Contrato obligatorio |

### 6.4 D. Grounding y RAG

| Subcontrato | Qué debe definir WAB-78 | Origen en D5-B | Relación D5-A | Categoría |
|-------------|------------------------|----------------|---------------|-----------|
| D1 | Qué significa que una respuesta esté fundamentada: citación de fragmentos recuperados, evidencia trazable | C4 (Grounding Verifier) | D5-A | WAB-78 — Contrato obligatorio |
| D2 | Evidencia y trazabilidad de fragmentos recuperados: fuente, score, metadatos, qué documentos se usaron por ejecución | C6 (Contrato de RAG) | D5-A | WAB-78 — Contrato obligatorio |
| D3 | Diferencia entre búsqueda sin resultados y fallo de recuperación: distinción error de servicio vs. ausencia de resultados vs. resultados irrelevantes | C6 | D5-A | WAB-78 — Contrato obligatorio |
| D4 | Comportamiento ante Qdrant no disponible, errores o resultados insuficientes: fail-closed, abstención, mensaje de servicio no disponible | C6 | Contexto del encargo (regla 5) | WAB-78 — Contrato obligatorio |
| D5 | Condiciones de abstención: no responder sin evidencia de grounding | C4, C5 | D5-A | WAB-78 — Contrato obligatorio |
| D6 | Límites del GroundingVerifier: no es garantía absoluta de veracidad; verifica citación/fundamento, no verdad factual | C4 | D5-A | WAB-78 — Contrato obligatorio |

### 6.5 E. Jerarquía y tratamiento de contexto

| Subcontrato | Qué debe definir WAB-78 | Origen en D5-B | Relación D5-A | Categoría |
|-------------|------------------------|----------------|---------------|-----------|
| E1 | Separación entre instrucciones de sistema, políticas, contexto empresarial y datos no confiables | C8 (Separación de prioridad) | D5-A | WAB-78 — Contrato obligatorio |
| E2 | Tratamiento de Knowledge, memoria, historial, mensajes y resultados de herramientas como datos no confiables | C8, C9 | D5-A (regla 4) | WAB-78 — Contrato obligatorio |
| E3 | Riesgos de introducir fragmentos empresariales como instrucciones privilegiadas: knowledge no debe tener misma prioridad que system prompt | C8 | D5-A | WAB-78 — Contrato obligatorio |
| E4 | Defensa frente a prompt injection: sanitización de documentos, separación de roles, knowledge sin prioridad de sistema | C9 (Sanitización de documentos) | D5-A | WAB-78 — Contrato obligatorio |
| E5 | Límites de sanitización textual: no es barrera absoluta; complementa controles determinísticos | C9 | D5-A | WAB-78 — Contrato obligatorio |

### 6.6 F. Memoria y confidencialidad

| Subcontrato | Qué debe definir WAB-78 | Origen en D5-B | Relación D5-A | Categoría |
|-------------|------------------------|----------------|---------------|-----------|
| F1 | Aislamiento por tenant y workspace: lectura y escritura de memoria aisladas | C11 (Control de memoria) | D5-A (regla 4) | WAB-78 — Contrato obligatorio |
| F2 | Autorización de lectura y escritura de memoria: quién puede leer/escribir facts por contacto/workspace | C11 | D5-A | WAB-78 — Contrato obligatorio |
| F3 | Retención y eliminación de memoria: políticas de retención y derecho al olvido por tenant | C11 | D5-A | WAB-78 — Contrato obligatorio |
| F4 | Prevención de filtraciones entre empresas: ningún fact de un tenant visible en otro | C11 | D5-A | WAB-78 — Contrato obligatorio |
| F5 | Separación entre memoria empresarial y conocimiento/configuración interna de Waby: no mezclar facts de clientes con prompts internos de Waby | C11 | D5-A | WAB-78 — Contrato obligatorio |

**Total de subcontratos:** 6 bloques (A-F) × 6 subcontratos cada uno = 36 subcontratos, todos clasificados como `WAB-78 — Contrato obligatorio` para el diseño. Ninguno requiere implementación para aprobar el diseño de WAB-78.

---

## 7. Separación entre contratos y trabajo posterior

Para cada requisito de esta reconciliación, se clasifica en una de las siguientes categorías. Esta clasificación es **recomendación**, no decisión aprobada.

| Categoría | Definición | Aplica a |
|-----------|------------|----------|
| `WAB-78 — Contrato obligatorio` | Debe definirse y aprobarse para cerrar el diseño. No requiere implementación previa para aprobar el diseño. | Todos los subcontratos A1-A6, B1-B6, C1-C5, D1-D6, E1-E5, F1-F5 de la sección 6. |
| `WAB-79/WAB-80 — Dominio, persistencia o provisioning` | Pertenece al alcance ya establecido de WAB-79/WAB-80 (modelo AgentInstance, índices, provisión, créditos, permisos). | Modelo de AgentInstance, ownership de persistencia, índices, matriz de permisos. |
| `Fase posterior — Implementación runtime` | Requiere contrato previo (definido en WAB-78), pero la implementación no es necesaria para aprobar el diseño. | Creación de Policy Engine, OutputFilterStep, DomainClassifier, GroundingVerifier, filtrado de capabilities, sanitización, etc. |
| `D5-C — Pendiente de auditoría` | No anticipar su resolución; reservado para el diagnóstico D5-C. | Autorización del CapabilityExecutor, carga/filtrado de capabilities, aislamiento multitenant de datos sensibles, logs y telemetría. |

### 7.1 Reglas de clasificación aplicadas

1. **Cierre de WAB-78 no depende de implementación.** La ausencia de código de Policy Engine, Output Filter u otro control no bloquea el cierre del diseño de WAB-78, siempre que los contratos estén definidos y aprobados.
2. **Cierre arquitectónico ≠ autorización para operar.** Que WAB-78 apruebe los contratos de seguridad no implica autorización para operar agentes sin controles de seguridad implementados. La implementación pertenece a fases posteriores.
3. **D5-C no se anticipa.** El filtrado de capabilities y la autorización del CapabilityExecutor quedan expresamente reservados a D5-C. Esta reconciliación no resuelve ni duplica ese diagnóstico.
4. **No nuevos IDs ARQ.** Todos los contratos de esta sección se asocian a ARQ-10 sin crear identificadores nuevos.

### 7.2 Tabla de clasificación consolidada

| Requisito | Contrato asociado | Categoría | Motivo |
|-----------|------------------|-----------|--------|
| Policy Engine (autoridad, evaluación, versionado, fallos, revocación, separación) | A1-A6 | `WAB-78 — Contrato obligatorio` | Define la autoridad de gobernanza transversal. Implementación en fase posterior. |
| Output Filter (punto, datos inspeccionados, confidencialidad, bloqueo, registro, errores) | B1-B6 | `WAB-78 — Contrato obligatorio` | Define el filtro de salida. Implementación en fase posterior. |
| Restricción de dominio (determinación, rol del clasificador, por qué no es autoridad, abstención, preentrenamiento) | C1-C5 | `WAB-78 — Contrato obligatorio` | Define la frontera del ámbito empresarial. Implementación en fase posterior. |
| Grounding y RAG (fundamentación, trazabilidad, sin resultados vs. error, Qdrant caído, abstención, límites del verifier) | D1-D6 | `WAB-78 — Contrato obligatorio` | Define el contrato de grounding obligatorio. Implementación en fase posterior. |
| Jerarquía y tratamiento de contexto (separación, datos no confiables, fragmentos privilegiados, prompt injection, límites sanitización) | E1-E5 | `WAB-78 — Contrato obligatorio` | Define la jerarquía de instrucciones y defensa contra inyección. Implementación en fase posterior. |
| Memoria y confidencialidad (aislamiento, autorización, retención, filtraciones entre empresas, separación Waby/tenant) | F1-F5 | `WAB-78 — Contrato obligatorio` | Define el aislamiento multitenant de memoria. Implementación en fase posterior. |
| Filtrado de capabilities por `enabledCapabilities` | C7 (D5-B) / D5-C | `D5-C — Pendiente de auditoría` | Reservado a D5-C (autorización del CapabilityExecutor). |
| Autorización de CapabilityExecutor | D5-C | `D5-C — Pendiente de auditoría` | Reservado a D5-C. |
| Aislamiento multitenant de datos sensibles, logs, telemetría | D5-C | `D5-C — Pendiente de auditoría` | Reservado a D5-C. |
| Modelo AgentInstance, índices, ownership de persistencia | WAB-79/WAB-80 | `WAB-79/WAB-80 — Dominio, persistencia o provisioning` | Alcance ya establecido de WAB-79/WAB-80.

---

## 8. Hallazgos reservados para D5-C

Los siguientes asuntos están expresamente reservados para el diagnóstico D5-C (aislamiento multitenant y autorización de capabilities, herramientas, datos sensibles, logs y telemetría). Esta reconciliación **no anticipa ni duplica** ese diagnóstico.

| # | Asunto reservado | Motivo de la reserva | Evidencia en D5-B |
|---|------------------|----------------------|-------------------|
| 1 | Autorización del CapabilityExecutor: quién puede ejecutar cada capability, bajo qué condiciones, y cómo se valida la autorización en tiempo de ejecución | D5-C tiene reservado el análisis de autorización del CapabilityExecutor | Contexto del encargo (regla 6); D5-B hallazgo 4.6 y escenario 5.7 mencionan la ausencia de filtrado pero no resuelven la autorización |
| 2 | Carga y filtrado de capabilities: cómo se expone `enabledCapabilities` al pipeline, qué pasa con capabilities no listadas, comportamiento de `assign_contact` con restricciones | D5-C tiene reservado la carga/filtrado de capabilities | D5-B contrato C7; D5-B fase 3 menciona filtrado de capabilities |
| 3 | Aislamiento multitenant de datos sensibles: prevención de filtraciones entre empresas en memoria, historial y conocimiento | Pertenece al alcance de D5-C (datos sensibles, aislamiento) | D5-B hallazgo 4.12, 4.13, 4.14; escenario 5.12 |
| 4 | Logs y telemetría de seguridad: registro estructurado de rechazos y abstenciones sin datos sensibles | Pertenece al alcance de D5-C (logs y telemetría) | D5-B fase 5, ítem de log estructurado |
| 5 | Validación de workspaceId del payload contra el bot, contacto y historial por workspace | Control de entrada multitenant; se menciona en D5-B pero pertenece a D5-C | D5-B hallazgos 4.13, 4.14; contratos C10 |

**Regla de no-anticipación:** Ningún asunto de esta tabla se resuelve en esta reconciliación. Se registran únicamente para trazabilidad y para evitar duplicación con D5-C.

---

## 9. Inconsistencias, dudas y decisiones que requieren aprobación del usuario

### 9.1 Inconsistencias detectadas

| # | Inconsistencia | Descripción | Origen | Severidad | Resolución propuesta |
|---|----------------|-------------|--------|-----------|---------------------|
| I1 | `informe-bloque-5a.md` no existe en el directorio | El encargo refiere a D5-A como informe separado, pero solo se recuperó `informe-bloque-5b.md` (D5-B). Las referencias a D5-A se toman del contexto de decisión y de menciones en D4-A/D4-B. | waby-docs / auditoría WAB-78 | Media | Documentar en esta reconciliación; solicitar al usuario confirmar si D5-A existe con otro nombre o si se da por sentado el contexto del encargo. |
| I2 | ARQ-10 estado "Preliminar — sin validar" vs. evidencia de D5-B | La matriz acumulativa mantiene ARQ-10 como preliminar, pero D5-A y D5-B confirman la ausencia de Policy Engine y Output Filter con evidencia de código. | matriz-acumulativa.md vs. D5-A/D5-B | Alta | Cambiar ARQ-10 a "Confirmado" (evidencia de D5-B), manteniendo "Bloquea: No" para cierre de diseño. Requiere aprobación del usuario. |
| I3 | D5-B clasifica 5 hallazgos CRÍTICO como "bloquean cierre WAB-78" | D5-B §4 marca B1, B2, B3, B4, B8 (y más) como "Bloquea WAB-78: Sí". El plan de línea base establece que una ausencia de implementación no bloquea por sí sola el cierre del diseño. | D5-B §4 vs. 00-linea-base.md §7 | Alta | Reinterpretar: los contratos deben definirse en WAB-78, pero la implementación no bloquea el cierre de diseño. Requiere aprobación del usuario. |
| I4 | D5-B propone 5 fases de implementación como requisito previo al cierre de WAB-78 | D5-B conclusión (§Conclusión de D5-B) afirma "Se requieren controles en las 5 fases propuestas antes de que WAB-78 pueda considerarse cerrado". Esto excede el alcance de WAB-78 (contratos, no implementación). | D5-B §Conclusión | Alta | Corregir conceptualmente: WAB-78 cierra con contratos definidos; la implementación es de fases posteriores. |

### 9.2 Dudas abiertas

| # | Duda | Contexto | Impacto si no se resuelve |
|---|------|----------|------------------------|
| D1 | ¿Existe realmente un informe D5-A separado con otro nombre, o el contexto del encargo es suficiente? | No se localizó `informe-bloque-5a.md` en `waby-docs`. | Las referencias a D5-A se basan en el contexto del encargo, no en un archivo verificable. |
| D2 | ¿El cambio de estado de ARQ-10 de "Preliminar — sin validar" a "Confirmado" requiere actualizar la matriz acumulativa? | Esta reconciliación no modifica `matriz-acumulativa.md` (restricción estricta). | El cambio de estado queda registrado aquí pero no aplicado a la matriz. |
| D3 | ¿Todos los 36 subcontratos de la sección 6 son obligatorios para cerrar WAB-78, o algunos pueden diferirse? | La clasificación propuesta los marca a todos como `WAB-78 — Contrato obligatorio`. | Si algún contrato no es obligatorio, el alcance de WAB-78 se reduce. |

### 9.3 Decisiones que requieren aprobación del usuario

| # | Decisión | Opciones | Recomendación del agente | Estado |
|---|----------|----------|-------------------------|--------|
| U1 | Estado de ARQ-10 en la matriz acumulativa | Mantener "Preliminar — sin validar" / Cambiar a "Confirmado" | Cambiar a "Confirmado" con evidencia de D5-B, manteniendo "Bloquea: No" | Pendiente |
| U2 | Cierre de WAB-78 con contratos definidos pero sin implementación | Aprobar cierre con contratos / Requerir implementación previa | Aprobar cierre con contratos (implementación es de fases posteriores) | Pendiente |
| U3 | Clasificación de los 36 subcontratos de la sección 6 | Todos obligatorios / Algunos diferibles | Todos obligatorios para el diseño; implementación posterior | Pendiente |
| U4 | Qué contratos van a D5-C vs. WAB-78 | Filtrado de capabilities a D5-C / Parte a WAB-78 | Filtrado de capabilities y autorización de CapabilityExecutor a D5-C (regla 6 del encargo) | Pendiente |
| U5 | Reinterpretación de D5-B "bloqueantes" | Mantener clasificación de D5-B / Reinterpretar como contratos pendientes | Reinterpretar: los contratos definen el diseño; la implementación no bloquea el cierre | Pendiente |

**Nota:** Ninguna decisión de esta sección se aplica automáticamente. Todas requieren aprobación explícita del usuario antes de actualizar la matriz acumulativa, Linear o cualquier documento canónico.

---

## 10. Criterios de aceptación para considerar D5-A y D5-B reconciliados

La reconciliación se considera completa cuando se cumplen todos los siguientes criterios:

| # | Criterio | Estado actual | Evidencia |
|---|----------|---------------|-----------|
| CA1 | Tabla de correspondencia D5-A ↔ D5-B construida con referencia exacta, evidencia, relación y clasificación para cada hallazgo | Completado | Sección 4 de este documento (13 filas) |
| CA2 | Hallazgos duplicados entre D5-A y D5-B identificados (o confirmada ausencia de duplicación) | Completado | Sección 4: ninguna duplicación de IDs ni de hallazgos; solapamiento sin duplicación |
| CA3 | Hallazgos contradictorios identificados (o confirmada ausencia de contradicción) | Completado | Sección 4: ninguna contradicción detectada |
| CA4 | ARQ-10 evaluado con la evidencia de D5-A y D5-B; estado recomendado documentado | Completado | Sección 5: estado recomendado "Confirmado", "Bloquea: No", cambio requiere aprobación del usuario |
| CA5 | Contratos de seguridad requeridos en WAB-78 definidos y clasificados (A-F, 36 subcontratos) | Completado | Sección 6 |
| CA6 | Separación entre contratos (WAB-78) y trabajo posterior (fases, WAB-79/WAB-80, D5-C) documentada | Completado | Sección 7 |
| CA7 | Hallazgos reservados para D5-C identificados sin anticipar su resolución | Completado | Sección 8 (5 asuntos reservados) |
| CA8 | Inconsistencias, dudas y decisiones pendientes de aprobación del usuario listadas | Completado | Sección 9 |
| CA9 | Criterios de aceptación definidos y revisables | Completado | Esta sección |
| CA10 | Distinción clara entre evidencia verificada, interpretación arquitectónica, recomendación y decisión pendiente del usuario | Completado | Aplicado en todo el documento mediante secciones diferenciadas |

**Condición de cierre:** D5-A y D5-B se consideran reconciliados cuando CA1-CA10 están cumplidos y el usuario ha aprobado las decisiones de la sección 9 (especialmente U1-U5). La actualización de la matriz acumulativa queda pendiente de esa aprobación y no se realiza en esta reconciliación.

---

## 11. Conclusión y recomendación de siguiente paso

### 11.1 Conclusión

D5-A y D5-B son compatibles y no se contradicen. D5-B confirma con evidencia de código las ausencias que D5-A había identificado (Policy Engine, Output Filter, tres puntos de control independientes) y las amplía con hallazgos concretos, brechas por severidad, escenarios de ataque y contratos propuestos. Ningún hallazgo de D5-B duplica o contradice a otro de D5-B ni a D5-A.

El estado recomendado es:

- **ARQ-10:** Confirmado (evidencia de D5-B), con "Bloquea: No" para el cierre de diseño. Cambio pendiente de aprobación del usuario.
- **Contratos de WAB-78:** 36 subcontratos en 6 bloques (A-F), todos como `WAB-78 — Contrato obligatorio` para el diseño. Ninguno requiere implementación para aprobar el diseño.
- **D5-C:** 5 asuntos expresamente reservados (autorización de CapabilityExecutor, filtrado de capabilities, aislamiento de datos sensibles, logs y telemetría, validación de entrada multitenant). No anticipar resolución.
- **WAB-79/WAB-80:** Modelo AgentInstance, índices, ownership de persistencia, matriz de permisos — fuera del alcance de esta reconciliación.

### 11.2 Correcciones conceptuales a D5-B (registradas, sin modificar el original)

1. **Las 5 fases de implementación no son requisito previo para cerrar WAB-78.** D5-B afirma que se requieren controles en las 5 fases antes de cerrar WAB-78. Esta reconciliación corrige: WAB-78 cierra con contratos definidos; la implementación es de fases posteriores.
2. **Los hallazgos de D5-B no son todos "bloqueantes" del cierre de diseño.** D5-B clasifica varios como "Bloquea WAB-78: Sí". Esta reconciliación los reinterpretan como contratos pendientes de definición en WAB-78, no como bloqueos del cierre arquitectónico.
3. **Domain Classifier no es autoridad de autorización.** Es un clasificador probabilístico; solo el Policy Engine es autoridad determinista. Esto se aclara en el contrato C3 de la sección 6.3.
4. **GroundingVerifier no es garantía absoluta de veracidad.** Verifica citación/fundamento, no verdad factual. Se aclara en el contrato D6 de la sección 6.4.
5. **Filtrado de capabilities pertenece a D5-C.** No se anticipa ni se resuelve en WAB-78.

### 11.3 Recomendación de siguiente paso

1. El usuario revisa y aprueba (o rechaza) las decisiones pendientes de la sección 9 (U1-U5).
2. Una vez aprobado, se actualiza la matriz acumulativa (`matriz-acumulativa.md`) con el estado de ARQ-10 y los contratos asociados, sin crear nuevos IDs ARQ.
3. Los contratos de la sección 6 se llevan a la dirección arquitectónica para su aprobación formal.
4. D5-C se inicia cuando esté autorizado, con los 5 asuntos de la sección 8 como alcance inicial.
5. La implementación de Policy Engine, Output Filter, Domain Classifier, GroundingVerifier y controles relacionados se programa en las fases de trabajo correspondientes (posteriores a WAB-78).

### 11.4 Confirmación de restricciones estrictas

Se confirma explícitamente que:

- No se modificó código, tests, schemas, configuración ni documentación canónica del repositorio `crm`.
- No se modificaron los informes D5-A ni D5-B originales (D5-A no existe como archivo separado; D5-B no se tocó).
- No se modificó `matriz-acumulativa.md`.
- No se modificó Linear ni sus issues/documentos.
- No se inició D5-C.
- No se crearon nuevos IDs ARQ.
- No se declaró aprobado ningún contrato que requiera decisión del usuario.
- No se repitieron auditorías D1-D4 ni D5-A/D5-B.
- No se afirmó que una ausencia de implementación implique necesariamente un fallo en producción.
- No se convirtieron recomendaciones de implementación en bloqueantes arquitectónicos sin justificar la dependencia concreta.

Este documento queda a la espera de revisión y aprobación del usuario. No se realizará ninguna actualización adicional hasta recibir dicha aprobación. |
<tool_call>
<function=edit>