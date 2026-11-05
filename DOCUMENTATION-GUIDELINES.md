# Convenciones documentales

Estas convenciones se aplican al repositorio privado `waby-docs`.

## Tipos de documentos

### Evidencia y auditorías

- Los informes aceptados se conservan como registro histórico.
- No se reescriben retrospectivamente para incorporar decisiones posteriores.
- Los errores materiales se aclaran mediante una adenda que identifica el documento y las secciones afectadas.
- Las consolidaciones pueden sintetizar varios informes, pero deben mantener referencias a sus fuentes.

### Documentos vivos

- Arquitectura objetivo, contratos, especificaciones, planes y guías se actualizan en su ruta estable.
- Git conserva el historial; no crear archivos `v2`, `v3` por cada modificación.
- Los cambios relevantes explican qué cambió y por qué.
- Cuando un cambio depende de una decisión, enlazar la issue o el comentario de Linear que la registra.

## Estado de las decisiones

- **Propuesta:** alternativa planteada, todavía no aprobada.
- **Pendiente:** requiere definición o validación.
- **Aprobada:** decisión confirmada y registrada en Linear.
- **Obsoleta:** reemplazada por otra decisión o contrato; conservar referencia a su reemplazo.

La documentación no debe presentar una propuesta como decisión tomada.

## Referencias y trazabilidad

Cuando aplique, incluir el identificador y enlace de Linear, repositorio/rama/commit analizados, fecha de revisión, archivos de código que sustentan las afirmaciones y estado de aceptación.

Evitar copiar el tablero completo de tareas o reproducir conversaciones extensas de Linear.

## Cambios

- Cambios pequeños y autocontenidos: commit directo en la rama principal.
- Cambios estructurales, consolidaciones importantes o revisiones que requieran discusión: rama y Pull Request.
- Antes de reemplazar un archivo, recuperar su versión actual y verificar que no haya cambios concurrentes.
- Después de una actualización importante, volver a leer el archivo desde GitHub para comprobar el resultado.

## Seguridad y alcance

- No guardar credenciales, tokens, claves privadas, secretos de infraestructura ni datos personales de clientes.
- No copiar documentación específica de implementación que deba mantenerse en el repositorio de código.
- Este repositorio es privado y personal; no pertenece a ElemeSoft ni representa un repositorio de código de producción.

## Idioma

La documentación se redacta en español, salvo nombres técnicos, identificadores, rutas o términos cuyo uso en inglés sea más preciso.
