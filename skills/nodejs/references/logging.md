# Logging

Structured logs should make a job traceable across API, worker, database, and external service calls.

Useful fields include operation, job/batch ID, document ID when appropriate, duration, status, error category, and correlation ID.

Never log passwords, private keys, bearer tokens, or complete sensitive payloads.

Winston can be used when it is already established in the project.
