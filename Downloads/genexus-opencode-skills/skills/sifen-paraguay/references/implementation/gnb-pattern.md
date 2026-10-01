# GNB project pattern

Source: supplied GNB project tree.

The example contains:
- `controllers/`
  - `envioMasivo.controller.js`
  - `gencontroller.js`
  - `genxmllote.js`
  - `masivecontroller.js`
  - `setcontroller.js`
  - `wconsultalote.js`
- `services/`
  - `consultaLote.service.js`
  - `consultaLote.worker.js`
  - `estadoEnvio.service.js`
  - `procesarLote.js`
  - `actualizar.service.js`
  - `envioCorreo.service.js`
- `basedatos/`
  - `conexion.js`
  - `iseries.js`
- `utils/`
  - `workerpool.js`
  - `logger.js`
- `workaround/`
  - `generarxml.js`
  - `regenerarxml.js`
  - `masiveconsultacdc.js`
- `sifenClient.js`, `server.js`, `index.js`.

## Architectural lesson

The project separates:
- generation;
- sending;
- asynchronous consultation;
- worker processing;
- state update;
- database access;
- delivery/email;
- operational workarounds.

## Important boundary

This is a GNB implementation pattern, not an official DNIT architecture requirement.

When applying it to another project, retain the separation of concerns only where it fits the current project's constraints.
