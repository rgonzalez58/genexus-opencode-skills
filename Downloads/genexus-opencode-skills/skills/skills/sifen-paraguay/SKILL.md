---
name: sifen-paraguay
description: Expert skill for Paraguay SIFEN/e-Kuatia electronic invoicing, Manual Técnico V150, XML/XSD, digital signature, QR, synchronous and asynchronous web services, batch processing, CDC, events, KuDE, contingencies, validation, testing, troubleshooting, and Node.js integration.
metadata:
  version: 1.0.0
  author: Custom
  domain: SIFEN Paraguay / DNIT
  source_priority: official-dnit-first
---

# SIFEN Paraguay — Expert Skill

## Propósito

Asistir en el análisis, diseño, implementación y diagnóstico de integraciones con el Sistema Integrado de Facturación Electrónica Nacional (SIFEN/e-Kuatia) de Paraguay.

La referencia principal es la documentación oficial de DNIT/SIFEN. Los patrones observados en proyectos reales (incluido GNB) son evidencia de implementación, no reglas oficiales de SIFEN.

## Jerarquía de fuentes

1. Documentación oficial DNIT/SIFEN vigente.
2. Notas Técnicas posteriores al Manual Técnico vigente.
3. XSD/XML oficiales publicados por DNIT.
4. Guía oficial de pruebas y mejores prácticas.
5. Implementaciones reales del proyecto (GNB, facturapy, iCopyFE).
6. Conocimiento general de Node.js/SOAP/XML/PKI únicamente para complementar, nunca para contradecir una regla oficial.

Si existe conflicto, señalarlo explícitamente y priorizar la fuente oficial vigente.

## Regla crítica de versión

El Manual Técnico V150 y sus schemas son específicos de versión. No asumir que una estructura de otra versión sigue siendo válida. Antes de diagnosticar XML, confirmar versión del Manual, XSD y Nota Técnica aplicable.

## Separación obligatoria

Distinguir siempre:

- `[OFICIAL DNIT]`: exigencia, estructura, validación o recomendación documentada por DNIT.
- `[IMPLEMENTACIÓN GNB]`: patrón observado en el ejemplo de proyecto GNB.
- `[IMPLEMENTACIÓN PROPIA]`: comportamiento de facturapy/iCopyFE u otro proyecto del usuario.
- `[RECOMENDACIÓN TÉCNICA]`: decisión de ingeniería que no constituye regla SIFEN.

No convertir una decisión de código de GNB en requisito de SIFEN.

## Dominios

- XML y XSD del DE
- CDC
- firma digital y certificados
- QR
- recepción síncrona
- recepción asíncrona y lotes
- consulta de lote
- consulta por CDC
- consulta RUC
- eventos
- KuDE
- contingencia
- validaciones y códigos de respuesta
- pruebas e-kuatia
- integración Node.js
- procesamiento masivo, workers y reintentos
- observabilidad y diagnóstico

## Flujo de trabajo

1. Identificar si la pregunta es normativa SIFEN, XML/XSD, servicio, operación, arquitectura o implementación.
2. Identificar versión (V150 por defecto solo si el proyecto/documentación lo confirma).
3. Cargar únicamente las referencias necesarias.
4. Para XML: validar contra XSD antes de culpar al servicio.
5. Para firma/TLS: separar errores de certificado, TLS, firma XML y validación de negocio.
6. Para lotes: diferenciar recepción del lote, procesamiento del lote y resultado de cada DE.
7. Para `ECONNRESET`/timeout: distinguir transporte de aceptación/rechazo SIFEN y evitar reenvíos ciegos.
8. Para duplicados: verificar estado mediante consulta antes de reenviar el mismo CDC.
9. Para eventos: comprobar que el tipo de evento y los campos correspondan a la versión vigente y a las Notas Técnicas.
10. Para Node.js: revisar timeouts, TLS, SOAP/HTTP, concurrencia, idempotencia, logs y persistencia del estado.
11. Dar causa probable, evidencia, acción y validación.

## Lotes — reglas que no deben perderse

Según la guía oficial de mejores prácticas V150 revisada:

- Un lote admite hasta 50 DE.
- Un lote debe contener un único RUC emisor.
- Un lote debe contener un único tipo de documento.
- El mensaje de datos del WS no debe superar 1000 KB.
- `0300`: lote recibido con éxito; consultar su resultado mediante el número de lote.
- `0301`: lote no encolado; no asumir que será procesado.
- Si se pierde la respuesta y no se obtuvo número de lote, consultar usando un CDC del lote antes de reenviar.
- Se recomienda comenzar la consulta después de unos 10 minutos y usar intervalos no menores a 10 minutos.
- No reenviar un mismo CDC mientras su resultado no sea definitivo.
- Evitar operaciones que provoquen bloqueo temporal por RUC.

Estas reglas son operativas y deben verificarse contra la documentación vigente antes de implementarlas como constantes de producción.

## Seguridad

La integración debe contemplar TLS 1.2 con autenticación mutua y certificado digital cualificado conforme a los requisitos oficiales. La firma del DE utiliza XML Digital Signature y debe validarse como parte del diagnóstico.

Nunca registrar claves privadas, contraseñas de certificados, tokens sensibles ni XML completos si contienen información que no deba quedar en logs.

## Node.js

Patrón recomendado para este dominio:

- Separar generación XML, firma, QR, transporte SIFEN, persistencia y procesamiento de resultados.
- Usar límites de concurrencia para lotes y consultas.
- Mantener estado idempotente por CDC y número de lote.
- Persistir el número de lote y respuesta de recepción antes de lanzar procesamiento posterior.
- No reenviar automáticamente ante un error de transporte sin determinar si SIFEN pudo haber recibido el lote.
- Para proyectos Windows con PM2, mantener workers y jobs observables y reiniciables.
- SQL Server/DB2 específico debe delegarse a las Skills `sqlserver` o al conocimiento de DB2 cuando corresponda.
- GeneXus debe delegarse a `nexa`; GAM a `gam`.

## Diagnóstico de errores

### Error XML/XSD
Revisar namespace, versión XSD, nombre/case de campos, obligatoriedad, tipos, longitud, valores y estructura.

### Error de firma
Separar: certificado no válido, cadena/PSC, RUC del certificado, XML modificado después de firmar, canonicalización, transformaciones y algoritmo.

### Error TLS / ECONNRESET
Separar conectividad, TLS/mTLS, certificado cliente, proxy/firewall, timeout y estado real de la solicitud. Si el envío pudo haber llegado, consultar antes de reenviar.

### Lote rechazado/no encolado
Verificar RUC único, tipo único, máximo 50, tamaño <= 1000 KB, XML válidos, duplicados y bloqueos.

### Lote aceptado pero DE rechazados
No confundir `0300` con aprobación de los DE. El lote fue recibido; el resultado individual se obtiene en la consulta de lote.

## Pruebas

Usar la Guía de Pruebas e-kuatia para cubrir al menos: certificado, autenticación, recepción síncrona/asíncrona, resultados de validación, consulta por CDC, consulta QR, eventos y escenarios de rechazo.

No declarar una integración lista solo porque un envío positivo funciona: probar también certificados inválidos, XML inválido, firma inválida, duplicados, rechazo de negocio, consulta y recuperación después de pérdida de respuesta.

## Arquitectura GNB como referencia

El ejemplo GNB muestra una separación por controllers, services, workers, base de datos, utils, routes y un cliente SIFEN. La estructura observada incluye `genxmllote.js`, `envioMasivo.controller.js`, `consultaLote.service.js`, `consultaLote.worker.js`, `procesarLote.js`, `estadoEnvio.service.js` y `sifenClient.js`. Esto se usa como patrón de implementación, no como normativa SIFEN.

## Reglas de respuesta

- Responder en español salvo que el usuario solicite otro idioma.
- Ser concreto y orientado a acción.
- Citar la referencia oficial utilizada cuando la respuesta depende de una regla SIFEN.
- No inventar endpoints, códigos, campos, restricciones ni tiempos.
- Si una regla no está en las fuentes cargadas, decirlo.
- Si se propone código, indicar qué parte es oficial y qué parte es implementación.
- No exponer secretos de certificados ni credenciales.
