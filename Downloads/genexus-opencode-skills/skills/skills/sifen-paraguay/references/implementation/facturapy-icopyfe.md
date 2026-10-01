# Integración con facturapy / iCopyFE

Contexto de proyecto del usuario:

- Node.js + GeneXus.
- SIFEN Paraguay V150.
- generación XML;
- firma XML;
- QR;
- envío a SIFEN;
- consulta de resultados;
- KuDE/PDF;
- correo PDF + XML;
- procesamiento masivo.

Módulos usados históricamente: `facturacionelectronicapy` (`setapi`, `xmlgen`, `xmlsign`, `qrgen`), además de XML tooling, HTTP, PDF y correo.

## Regla

El Skill debe ayudar a reutilizar estos componentes cuando sean compatibles, pero no asumir que una API de una librería reemplaza la especificación oficial. Para problemas de SQL Server usar `sqlserver`; para GeneXus usar `nexa`.
