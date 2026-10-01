# Node.js para SIFEN

## Capas sugeridas

```text
controllers/routes
      |
application services
      |
SifenClient / SOAP-HTTP adapter
      |
XML / XSD / signature / QR
      |
repository / DB
```

## Workers

Para lotes masivos, usar workers con concurrencia acotada y estado persistente. Un worker de consulta no debe convertir una respuesta `0300` en aprobación; debe consultar y procesar el resultado individual.

## Observabilidad

Registrar correlation id, número de lote, CDC, endpoint lógico, duración, HTTP/SOAP result, código SIFEN y estado interno. No registrar secretos.
