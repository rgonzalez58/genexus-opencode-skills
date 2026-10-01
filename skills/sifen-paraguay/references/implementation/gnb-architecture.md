# Patrón de arquitectura GNB

El árbol entregado como ejemplo muestra una aplicación Node.js organizada por responsabilidades.

## Estructura observada

- `basedatos/`: conexión e iSeries.
- `controllers/`: descarga KuDE/XML, envío masivo, generación, consulta y SET.
- `services/`: actualización, consulta de lote, worker de consulta, documentos unidos, correo, estado de envío y procesamiento de lote.
- `routes/`: rutas HTTP.
- `utils/`: funciones, logger, acceso iSeries y worker pool.
- `workaround/`: generación/regeneración XML, envío de lote y consultas masivas.
- `sifenClient.js`: cliente SIFEN.

## Aplicación al Skill

Esta arquitectura es una referencia práctica para separar transporte SIFEN, procesamiento de lotes, consultas y persistencia. No asumir que cada nombre de archivo o patrón de GNB debe copiarse a otro proyecto.
