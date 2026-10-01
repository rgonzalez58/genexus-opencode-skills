# DNIT best practices — asynchronous DE sending

Source: supplied `Guía de Mejores Prácticas para la Gestión del Envío de DE`, October 2024.

## Document generation

The guide assumes knowledge of XML, SOAP 1.2, HTTP, TLS 1.2 mutual authentication and XML Digital Signature.

For DE XML generation it warns against:
- leading/trailing whitespace in numeric/alphanumeric fields;
- comments/annotations/documentation;
- formatting characters such as line-feed, carriage-return, tabs and spaces between tags;
- namespace prefixes;
- empty field tags where the field is not required;
- negative/non-numeric values in numeric fields;
- incorrect case in field names.

It also points to the DNIT SIFEN pre-validator for development-time validation.

## Lot generation

The guide states:
- lots are processed asynchronously;
- up to 50 DE can be included in a lot;
- production and test environments use different domains;
- the reception service and lot-consultation service are separate;
- the individual CDC consultation is separate.

## Reception responses

Documented examples include:
- `0300`: lot received successfully and will be processed; the returned lot number must be consulted.
- `0301`: lot was not queued for processing; investigate before retrying.

## Retry principle

If the application does not receive a response after sending a lot, do not automatically assume that SIFEN did not receive it. Determine whether the lot can be identified/consulted before resending.

## Consultation timing

The guide recommends not hammering the consultation service. It describes beginning consultation after an initial waiting period and using intervals no shorter than 10 minutes in the documented scenario.

Always verify whether later DNIT documentation changes these recommendations.
