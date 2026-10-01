# Firma digital, certificado y TLS

La documentación cargada describe:

- TLS 1.2 con autenticación mutua.
- Certificado digital cualificado emitido por una PSC habilitada.
- XML Digital Signature en formato Enveloped.
- RSA 2048 para cifrado por software.
- SHA-256 para message digest.
- Base64.
- Transformaciones Enveloped y C14N.

## Diagnóstico

Separar:

1. certificado ausente/incorrecto;
2. RUC del certificado;
3. cadena/PSC/validez;
4. autenticación mTLS;
5. canonicalización y transforms;
6. XML modificado después de firmar;
7. firma inválida;
8. validación de negocio posterior.

Nunca guardar certificados privados o contraseñas en el repositorio. En logs registrar solo identificadores no sensibles.
