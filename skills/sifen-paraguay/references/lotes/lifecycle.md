# Ciclo de vida de un lote

```text
GENERADO
  -> FIRMADO
  -> EMPAQUETADO
  -> ENVIADO
  -> 0300 / 0301 / error de transporte
       0300 -> ESPERA -> CONSULTA LOTE -> RESULTADO DE DE
       pérdida de respuesta -> CONSULTA POR CDC -> recuperar estado/número de lote
       0301 -> diagnosticar y corregir antes de reenviar
```

## Estado mínimo recomendado en la aplicación

- identificador interno del lote;
- número de lote SIFEN;
- CDCs incluidos;
- RUC emisor;
- tipo de documento;
- fecha/hora de generación;
- fecha/hora de envío;
- respuesta de recepción;
- estado de consulta;
- resultado individual por CDC;
- cantidad de intentos;
- último error técnico;
- último código SIFEN;
- timestamps de cada transición.

La persistencia exacta es una decisión del proyecto y no una exigencia normativa.
