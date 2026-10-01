# Notas de extracción de las fuentes cargadas

Este archivo resume únicamente puntos identificados durante la lectura de los PDFs entregados. Para decisiones normativas, consultar el documento original y la versión vigente publicada por DNIT.

## Manual V150

- Índice: secciones 6-13 cubren modelo operativo, tecnología, servicios, CDC, eventos, validaciones y KuDE/QR.
- Índice de schemas: incluye DE_v150, Evento_v150, recepción, consultas, eventos y firma.
- El Manual indica servicios síncronos y asíncronos y separa recepción de lote y consulta posterior.

## Mejores prácticas — octubre 2024

- XML: evitar espacios/formato innecesario y respetar nombres exactos.
- Lote: hasta 50 DE, mismo RUC, mismo tipo, <=1000 KB.
- 0300: recibido; 0301: no encolado.
- Pérdida de respuesta: consultar por CDC para recuperar estado/número de lote.
- Consulta inicial: después de 10 minutos; intervalos >=10 minutos.
- No duplicar CDC mientras está en procesamiento.
- Bloqueos temporales por RUC ante determinadas reincidencias.

## Guía de pruebas

Incluye pruebas de certificado, autenticación, recepción, validaciones/rechazos, consultas, eventos y QR.

## NT 026

Modifica campos/validaciones relacionados con Compras Públicas.

## NT 027

Fecha 09/03/2026. Modifica campos del evento de nominación de FE relacionados con identificación del receptor.
