# Idempotencia y reintentos

## Regla central

Un timeout, `ECONNRESET` o corte de conexión no demuestra que SIFEN no recibió el lote.

Antes de reenviar:

1. conservar el payload/identificador del intento;
2. si no hay número de lote, consultar un CDC incluido, según la guía;
3. determinar si el DE/lote está en procesamiento;
4. solo reenviar cuando el estado permita hacerlo.

## No hacer

- `retry` ciego de un lote completo;
- generar un CDC nuevo para ocultar un problema de transporte;
- enviar el mismo CDC repetidamente mientras sigue procesándose;
- paralelizar consultas y reenvíos sin límites.
