# Diagnóstico práctico

## ECONNRESET / timeout

No concluir “SIFEN rechazó”. Puede existir una solicitud recibida sin respuesta observable por el cliente. Recuperar estado antes de reenviar.

## SOAP/HTTP 400

Separar envelope/namespace, certificado/TLS, XML interno, headers, endpoint y respuesta SIFEN. Conservar respuesta técnica para diagnóstico.

## Error de XSD

Comparar XML generado con XSD exacto de V150 y reglas modificadas por NT.

## Rechazo por duplicado

Buscar el CDC en estado local y consultar SIFEN antes de generar un nuevo envío.

## Problemas masivos

Medir: tamaño de lote, tiempo de generación, tiempo de firma, tiempo de envío, tiempo de consulta, cantidad de workers, errores por minuto y estados pendientes. Evitar `Promise.all` sin límite para decenas de miles de documentos.
