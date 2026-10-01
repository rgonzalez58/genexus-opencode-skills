# Suggested SIFEN acceptance matrix

Use the supplied DNIT test guide as the baseline.

| Area | Minimum verification |
|---|---|
| Connectivity | mutual TLS/authentication |
| XML | schema-valid DE |
| Signature | valid signed DE |
| Reception | individual/lote service as applicable |
| Async | lot received and later consulted |
| Validation | approved and rejected cases |
| Errors | intentionally invalid DE cases |
| Events | registration and association |
| Consultation | DTE and events |
| QR | validation |
| KuDE | generation/transmission where applicable |
| Recovery | timeout/ECONNRESET without duplicate blind resend |
| Audit | correlation between internal document, lot, CDC and SIFEN result |

A single successful invoice is not sufficient acceptance evidence.
