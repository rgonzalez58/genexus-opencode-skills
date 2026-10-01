# Matriz de pruebas recomendada

| Área | Caso positivo | Caso negativo | Recuperación |
|---|---|---|---|
| Certificado | válido | no válido/no autorizado | renovar/configurar |
| XML | XSD válido | campo/estructura inválida | corregir/generar |
| Firma | válida | firma/certificado inválido | firmar nuevamente |
| Síncrono | aprobado | rechazo | consultar según caso |
| Lote | 0300 | 0301 | diagnosticar/reintentar cuando corresponda |
| Consulta lote | definitivo | aún procesando/error | esperar intervalo |
| CDC | encontrado | inexistente | revisar CDC |
| QR | consulta válida | QR inválido | revisar generación |
| Evento | aceptado | rechazado | revisar estado/reglas |
| Transporte | respuesta | timeout/ECONNRESET | consulta antes de reenviar |
