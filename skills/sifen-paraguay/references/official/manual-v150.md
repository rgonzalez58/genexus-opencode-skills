# Manual Técnico V150 — mapa operativo

Fuente: Manual Técnico Sistema Integrado de Facturación Electrónica Nacional (SIFEN), Versión 150, 10/09/2019.

## Secciones relevantes

- 6: Modelo Operativo.
- 7: Características tecnológicas del formato.
- 8: Aspectos tecnológicos de los Servicios Web.
- 9: Descripción de los Servicios Web.
- 10: Código de Control (CDC).
- 11: Gestión de eventos.
- 12: Validaciones.
- 13: KuDE y representación gráfica / QR.

## Schemas identificados en el índice del Manual

- `xmldsig-core-schema-v150.xsd`
- `siRecepDE_v150.xsd`
- `resRecepDE_v150.xsd`
- `ProtProcesDE_v150.xsd`
- `SiRecepLoteDE_v150.xsd`
- `ProtProcesLoteDE_v150.xsd`
- `resRecepLoteDE_v150.xsd`
- `SiResultLoteDE_v150.xsd`
- `resResultLoteDE_v150.xsd`
- `siConsDE_v150.xsd`
- `resConsDE_v150.xsd`
- `ContenedorDE_v150.xsd`
- `ContenedorEvento_v150.xsd`
- `siRecepEvento_v150.xsd`
- `resRecepEvento_v150.xsd`
- `siConsRUC_v150.xsd`
- `resConsRUC_v150.xsd`
- `ContenedorRUC_v150.xsd`
- `DE_v150.xsd`
- `Evento_v150.xsd`

## Modelo operativo

El DE es el archivo electrónico firmado digitalmente. La aprobación por SIFEN convierte el DE en DTE. El receptor debe poder verificar la existencia y correspondencia del DTE mediante los mecanismos de consulta establecidos.

## Servicios

El Manual distingue servicios síncronos y asíncronos. En el flujo asíncrono, la recepción del lote y la consulta posterior del resultado son operaciones separadas.

## Regla de diagnóstico

Cuando una respuesta de transporte no es concluyente, no inferir aprobación ni rechazo. Debe determinarse el estado mediante el mecanismo de consulta correspondiente.
