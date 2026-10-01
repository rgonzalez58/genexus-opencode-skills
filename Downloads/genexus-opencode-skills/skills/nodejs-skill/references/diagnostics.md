# Node.js Production Diagnostics

Collect evidence before changing architecture.

Check:
- Node.js version
- process uptime/restarts
- RSS/heap usage
- CPU
- event-loop delay when relevant
- PM2 status
- database pool state
- slow SQL
- external HTTP latency
- file/PDF generation time
- retry counts
- batch size and concurrency

Useful commands:

```powershell
node --version
npm --version
pm2 list
pm2 logs
```

Do not infer a memory leak from a single high RSS observation. Compare memory over time and workload completion.
