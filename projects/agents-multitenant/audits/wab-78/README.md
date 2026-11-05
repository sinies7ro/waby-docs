# WAB-78 — Índice documental y registro de migración

**Estado:** migración documental completada desde Dropbox a `sinies7ro/waby-docs`.
**Destino:** `projects/agents-multitenant/audits/wab-78/`.
**Origen:** `/WABY/docs/revision-wab-78/` en Dropbox.
**Fecha de migración:** 05/11/2026.

## Alcance y preservación

Se trasladaron los ocho archivos Markdown inventariados en la carpeta de origen. Los documentos se mantienen con sus nombres y contenidos recuperados, sin reescritura editorial. Este README es un índice nuevo, creado para registrar la procedencia y facilitar la navegación.

Los originales de Dropbox no se modificaron ni eliminaron. Los enlaces de origen se conservan como referencia de procedencia; GitHub pasa a ser la ubicación documental de trabajo para este conjunto.

## Inventario

| Documento | Fecha de modificación en Dropbox (UTC) | Tamaño de origen | Líneas recuperadas | Enlace de origen |
|---|---:|---:|---:|---|
| [00-linea-base.md](./00-linea-base.md) | 2026-11-05T14:28:04Z | 11,128 bytes | 119 | [Dropbox](https://www.dropbox.com/scl/fi/0agjdgbm07b9bq0diga0i) |
| [informe-bloque-1.md](./informe-bloque-1.md) | 2026-11-05T14:28:04Z | 31,647 bytes | 310 | [Dropbox](https://www.dropbox.com/scl/fi/iwrbk1lf81wfecr3gsscg) |
| [informe-bloque-2.md](./informe-bloque-2.md) | 2026-11-05T14:28:04Z | 46,562 bytes | 295 | [Dropbox](https://www.dropbox.com/scl/fi/ckxvi4jed453onjypfrkc) |
| [informe-bloque-3.md](./informe-bloque-3.md) | 2026-11-05T14:28:04Z | 110,461 bytes | 992 | [Dropbox](https://www.dropbox.com/scl/fi/7wlmkg8em31donbjjag7t) |
| [matriz-acumulativa.md](./matriz-acumulativa.md) | 2026-11-05T14:28:04Z | 39,019 bytes | 87 | [Dropbox](https://www.dropbox.com/scl/fi/b9tptq5pm8mct3tolxfxd) |
| [analisis-previo-d2-d4.md](./analisis-previo-d2-d4.md) | 2026-11-05T14:28:04Z | 6,690 bytes | 103 | [Dropbox](https://www.dropbox.com/scl/fi/qw4zzvcht8km6g54amsep) |
| [informe-d4-a-runtime.md](./informe-d4-a-runtime.md) | 2026-11-05T15:58:31Z | 61,689 bytes | 674 | [Dropbox](https://www.dropbox.com/scl/fi/rhkuzveq3sixkr6lyvbxe) |
| [informe-d4-b-contexto.md](./informe-d4-b-contexto.md) | 2026-11-05T16:18:11Z | 84,609 bytes | 1,028 | [Dropbox](https://www.dropbox.com/scl/fi/1olijogpapdt8k04tf7qj) |

## Clasificación

- `00-linea-base.md`: línea base, fuentes y convenciones de la revisión.
- `informe-bloque-1.md`, `informe-bloque-2.md`, `informe-bloque-3.md`: diagnósticos de D1, D2 y D3.
- `informe-d4-a-runtime.md`, `informe-d4-b-contexto.md`: diagnósticos de D4-A y D4-B.
- `matriz-acumulativa.md`: matriz acumulativa de hallazgos ARQ.
- `analisis-previo-d2-d4.md`: material previo preservado para D2–D4.

## Integridad y método

Cada archivo se recuperó desde Dropbox mediante su identificador de archivo y se creó en GitHub con el texto Markdown completo devuelto por Dropbox. La verificación posterior compara el contenido leído desde GitHub con el contenido recuperado desde Dropbox, archivo por archivo.

Los tamaños indicados corresponden al tamaño del archivo fuente en Dropbox; las líneas corresponden al texto recuperado para la migración. No son métricas de tamaño del blob de GitHub.

## Seguimiento

- Issue principal: [WAB-78 — Fase 2: Diseño objetivo y contratos](https://linear.app/waby/issue/WAB-78/fase-2-diseno-objetivo-y-contratos).
- WAB-79 permanece fuera del alcance de esta migración y no se modificó su estado ni sus dependencias.
- Esta migración no implica aceptación global de WAB-78 ni cierre de sus bloques.
