# PDF and Email

PDF generation can be CPU and memory intensive. Puppeteer/Chromium should be treated as an external resource with explicit timeouts and cleanup.

For large documents:
- avoid unnecessary copies of huge HTML strings
- clean temporary files
- monitor CPU and memory
- separate database persistence from physical file generation when business rules allow it

Email workers should process bounded batches and record success/failure. Avoid duplicate sends with durable state and an idempotency strategy where possible.
