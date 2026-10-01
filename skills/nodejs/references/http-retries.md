# HTTP Timeouts and Retries

Every external HTTP integration should define appropriate timeouts.

Separate connection failure, timeout, DNS/network error, HTTP 4xx, HTTP 5xx, and malformed response.

Retry only transient failures and only when the operation is safe to repeat.

Use bounded retries with backoff. Avoid retry storms.

For document submission systems, persist request/result state so a network failure does not automatically imply that the remote system did not receive the request.
