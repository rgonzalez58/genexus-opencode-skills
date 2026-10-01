# Async and Concurrency

Node.js handles I/O efficiently, but CPU-heavy JavaScript can block the event loop.

Avoid uncontrolled `Promise.all()` over very large collections.

Use bounded batches or a concurrency limiter. Select limits based on database pool size, CPU, memory, external service limits, and file I/O.

Use worker threads or separate processes when CPU-heavy JavaScript itself is the bottleneck.
