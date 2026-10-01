# Servicios de recepción

## Recepción síncrona

Servicio para transmisión y respuesta dentro del mismo flujo. Diagnosticar por separado conexión, autenticación, XML, firma y resultado de negocio.

## Recepción asíncrona

El envío del lote no equivale a la aprobación de los DE. El flujo es:

1. Generar y firmar DE.
2. Construir lote.
3. Enviar lote.
4. Registrar respuesta y número de lote si existe.
5. Esperar el periodo recomendado.
6. Consultar resultado de lote.
7. Procesar resultado individual de cada DE.

## Consulta por CDC

Se usa para consultar un DE y también como mecanismo de recuperación cuando un envío de lote no devuelve el número de lote, según la guía de mejores prácticas.
