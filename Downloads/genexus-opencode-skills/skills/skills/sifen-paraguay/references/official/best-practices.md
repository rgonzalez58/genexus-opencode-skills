# Mejores prácticas oficiales para envío de DE

Fuente: Recomendaciones y mejores prácticas para SIFEN — Guía para el desarrollador — octubre 2024.

## Generación XML

La guía advierte evitar espacios al inicio/final de campos, comentarios/documentación XML innecesarios, caracteres de formato entre etiquetas, prefijos de namespace, etiquetas sin valor cuando no correspondan y valores no numéricos/negativos en campos numéricos. Los nombres de campos son sensibles a mayúsculas/minúsculas.

## Lotes

- Hasta 50 DE por lote.
- Un único RUC emisor por lote.
- Un único tipo de documento por lote.
- El mensaje de datos de entrada del WS no debe superar 1000 KB.

## Respuestas

- `0300`: lote recibido con éxito; consultar por número de lote.
- `0301`: lote no encolado; no será procesado.

## Pérdida de respuesta

Si el envío se realizó pero no se recibió respuesta/número de lote, la guía indica consultar utilizando un CDC incluido en el lote. Esta opción debe utilizarse solo cuando no se obtuvo el número de lote.

## Consulta

Se recomienda comenzar la consulta del lote pasados 10 minutos y utilizar intervalos no menores a 10 minutos.

## Duplicados y bloqueos

No reenviar un mismo CDC sin resultado definitivo (Aprobado, Aprobado con Observación o Rechazado). La guía identifica bloqueos temporales por RUC asociados a lotes vacíos/no válidos, CDC repetidos, CDC repetidos mientras siguen en procesamiento y reenvío de lotes.

## TLS y firma

La guía describe TLS 1.2 con autenticación mutua, XML Digital Signature Enveloped, certificado de PSC habilitada, RSA 2048, SHA-256, Base64 y transformaciones Enveloped/C14N para la firma.
