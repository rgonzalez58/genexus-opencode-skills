# Endpoints documentados en la guía de mejores prácticas

La guía usa `{ambiente}` para distinguir producción y test.

Producción: `sifen.set.gov.py`

Test: `sifen-test.set.gov.py`

Servicios citados:

- Recepción lote: `/de/ws/async/recibe-lote.wsdl`
- Consulta lote: `/de/ws/consultas/consulta-lote.wsdl`
- Consulta CDC: `/de/ws/consultas/consulta.wsdl`

Para obtener WSDL se agrega `?wsdl`.

No hardcodear endpoints en código de negocio; centralizarlos por ambiente y verificarlos contra documentación vigente.
