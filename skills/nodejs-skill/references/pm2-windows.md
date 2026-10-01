# PM2 on Windows

Use PM2 for process lifecycle, restarts, logs, and separate workers.

Keep API and background workers separate when their failure/restart characteristics differ.

For an exclusive worker, ensure only one instance claims a given job.

Useful commands:

```powershell
pm2 list
pm2 status
pm2 logs
pm2 monit
```
