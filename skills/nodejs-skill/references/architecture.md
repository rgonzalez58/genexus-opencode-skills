# Node.js Architecture

Separate HTTP/API concerns, business services, repositories, and external integrations.

Prefer:

```text
route/controller -> service -> repository
                         -> external client
```

Use background workers for long operations when business requirements allow it.

Reuse the existing project architecture unless there is a demonstrated problem.
