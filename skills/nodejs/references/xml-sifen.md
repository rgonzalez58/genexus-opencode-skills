# XML, Signing, QR and SIFEN

For electronic invoicing:

1. Generate XML.
2. Validate required data.
3. Sign XML.
4. Generate QR.
5. Submit to the external service.
6. Store response/state.
7. Query asynchronous results when required.
8. Generate PDF/KuDE.
9. Queue or send email.

Distinguish transport failures from business rejection/acceptance.

Do not mark a document successfully submitted before the external response/state justifies it.

Keep certificates and private keys outside source control.
