# SIFEN service diagnostics

## Capture

For a failed service call record:
- environment;
- service category;
- endpoint;
- timestamp;
- duration;
- HTTP status if present;
- SOAP fault/body when safe;
- SIFEN code/message when present;
- internal correlation ID;
- lot/CDC when available.

Never log credentials or private-key material.

## ECONNRESET / timeout

Treat as transport uncertainty:
1. request may have failed before reaching SIFEN;
2. request may have reached SIFEN but response may have been lost;
3. SIFEN may have accepted the lot and be processing it.

Therefore:
- correlate;
- consult if supported;
- only resend after determining duplicate risk.

## HTTP/SOAP vs SIFEN business status

Keep transport result separate from SIFEN business result. A 200/HTTP success is not equivalent to DTE approval, and a timeout is not equivalent to rejection.
