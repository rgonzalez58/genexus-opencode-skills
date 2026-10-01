# XML generation

## Rules from supplied DNIT best-practices guide

For DE XML:
- use the exact element names/case;
- use the applicable namespace/schema;
- do not add comments/annotations/documentation;
- avoid formatting whitespace prohibited by the guide;
- avoid empty elements when not required;
- numeric fields must contain valid numeric values;
- do not add namespace prefixes when the guide prohibits them;
- validate against the applicable XSD.

## Example namespace/version

The supplied best-practices guide shows the V150 reception structure using the SIFEN namespace and `siRecepDE_v150.xsd` schema location.

Do not copy an example blindly: verify the service/document and applicable schema before generating XML.

## Pre-validation

Use the DNIT SIFEN pre-validator during development when available.
