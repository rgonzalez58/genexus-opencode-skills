# Digital signature, certificates and TLS

The supplied best-practices guide assumes:
- TLS 1.2;
- mutual authentication;
- XML Digital Signature in Enveloped format;
- X509Data certificate representation;
- RSA 2048;
- SHA-2/SHA-256 message digest;
- Base64 encoding;
- Enveloped and exclusive C14N transformations as described in the guide.

## Diagnostic separation

There are at least two different security layers:
1. transport authentication/TLS;
2. XML digital signature.

Do not diagnose a signature error as a TLS error or vice versa.

## Secrets

Never expose:
- P12/PFX contents;
- private key;
- certificate password;
- CSC;
- access credentials.
