# XML y XSD V150

## Principios

El XML debe respetar exactamente los nombres de campos y namespaces publicados. Los campos son sensibles a mayúsculas/minúsculas.

Antes de enviar:

1. Validar XML contra XSD de la versión.
2. Validar obligatoriedad y cardinalidad.
3. Validar tipos/longitudes.
4. Validar valores y reglas de negocio conocidas.
5. Firmar después de producir el XML definitivo.
6. No modificar el XML después de la firma salvo que el flujo oficial lo contemple.

## Archivos entregados

El paquete del usuario contiene `Estructura xml_DE.rar` y `Estructura_DE xsd.rar`. En el entorno de construcción de este Skill los RAR no pudieron ser descomprimidos por ausencia de un backend RAR; se mantienen como fuentes entregadas y deben incorporarse físicamente al repositorio de referencia si se dispone de sus XML/XSD extraídos.
