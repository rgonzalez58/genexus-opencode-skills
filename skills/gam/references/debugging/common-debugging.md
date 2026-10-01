---
name: common-debugging
description: Trace activation, log.config configuration, and diagnostic workflow
---

# GAM Debugging Core
Scope: activating GAM traces, reading log.config, and running a diagnostic workflow. Extracted topics live in sibling files — see pointers below

Related files:
- [GAM Trace Analyzer](common-trace-analyzer.md) — 5-pass trace analysis methodology
- [GAM Trace Signatures Catalog](trace-signatures-catalog.md) — trace markers organized by domain
- [GAM SQL Diagnostic Queries](sql-diagnostic-queries.md) — DB-level diagnostic queries
- [Common GAM Login Issues](common-login-issues.md) — common login failure patterns
- [GAM Cache Debugging](cache-debugging.md) — stale cache symptoms and invalidation
- [Playwright Testing for GAM](testing-playwright.md) — browser automation for GAM flows
- [Debugging SLO with Non-GAM Clients](../logout/slo-non-gam-debugging.md) — SLO chains with non-GAM clients
- [GeneXus Environment: .NET Core / .NET 5+](../environments/environment-netcore.md) — .NET Core specifics
- [GeneXus Environment: .NET Framework](../environments/environment-netframework.md) — .NET Framework specifics
- [GeneXus Environment: Java](../environments/environment-java.md) — Java specifics

---

## Activating GAM Traces (3-Step Process)
GAM tracing requires THREE independent switches. Missing ANY ONE results in zero trace output with no warning

Version note: starting from v18 Upgrade 15 (Beta), GAM uses a structured JSON trace format instead of the legacy `GAMTrace-` format. The activation switches are the same, but there is an additional runtime condition: `Log.IsDebugEnabled()` must be `true`. This means `log.config` root level must be `DEBUG` or `ALL` — not just for file output, but also as a condition for GAM to emit the trace. See [GAM Trace Analyzer](common-trace-analyzer.md) for format detection

### Master Switch (SysPar Table)
```sql
UPDATE gam.SysPar SET SysParVal = '1' WHERE SysParId = 'EnableTracing';

SELECT SysParId, SysParVal FROM gam.SysPar WHERE SysParId = 'EnableTracing';
```

Constraints:
- Value is boolean: `'0'` (off) or `'1'` (on). Do NOT use `'2'` or `'3'` — those are not valid trace levels
- Global switch. If this is `'0'`, no tracing occurs regardless of other settings

### Per-Repository Switch (RepositoryProp Table)
```sql
-- All repositories
UPDATE gam.RepositoryProp SET RepPropVal = '1' WHERE RepPropId = 'EnableTracing';

-- Specific repository
UPDATE gam.RepositoryProp SET RepPropVal = '1'
WHERE RepPropId = 'EnableTracing' AND RepId = <target_RepId>;

SELECT RepId, RepPropId, RepPropVal FROM gam.RepositoryProp WHERE RepPropId = 'EnableTracing';
```

Constraints:
- Must be enabled for EVERY repository you want to trace
- A repository with `RepPropVal = '0'` produces no traces even if the master switch is on

### log4net Configuration (log.config)
The `log.config` file controls whether GAM trace output actually reaches a file. Without this step, all GAM traces are silently discarded

File locations (BOTH must be updated if both exist):
- `<KB>/NetModel/web/bin/log.config`
- `<KB>/NetModel/web/log.config`

Required change — root logger level:

```xml
<!-- BEFORE — default blocks ALL traces -->
<root>
	<level value="OFF" />
	<appender-ref ref="RollingFile" />
</root>

<!-- AFTER — enables complete GAM tracing -->
<root>
	<level value="ALL" />
	<appender-ref ref="RollingFile" />
</root>
```

Critical: the default GeneXus `log.config` ships with `level value="OFF"` on the root logger. This is the single most common reason traces appear to be "not working" even after enabling both database switches

Critical (v18u15+): in v18 Upgrade 15 and later, the log root level serves a dual purpose: it controls whether log4net writes to the file AND GAM checks `Log.IsDebugEnabled()` before emitting ANY trace. With `level="OFF"`, GAM short-circuits and never generates any trace at all — the traces are not just suppressed from the file, they are never generated. Use `DEBUG` or `ALL`

### Cache Expiry Requirement
After updating the database switches, GAM does NOT read them immediately. The repository configuration is cached in memory

- Default cache timeout: 60 seconds (controlled by `CacheTimeout` in RepositoryProp)
- Wait for expiry: the new `EnableTracing` value takes effect only after the cache expires
- Force immediate reload: recycle the IIS AppPool (or restart Kestrel / Tomcat)

```powershell
& "$env:windir\system32\inetsrv\appcmd.exe" recycle apppool /apppool.name:"DefaultAppPool"
```

### Trace Output Location
GAM traces write to the file configured in the `RollingFile` appender of `log.config`:

```xml
<appender name="RollingFile" type="log4net.Appender.RollingFileAppender">
	<file value="client.log"/>
	<!-- … -->
</appender>
```

- The `<file value="client.log"/>` path is relative to the bin directory
- Default output: `<KB>/NetModel/web/bin/client.log`
- If the file does not appear after enabling all three switches, check file system permissions on the `bin/` directory

### Verification Checklist
- `SysPar.EnableTracing = '1'`
- `RepositoryProp.EnableTracing = '1'` for the target repo
- `log.config` root level = `ALL` or `DEBUG` (both file locations)
- AppPool recycled or 60s cache waited
- `client.log` file appears in `bin/` directory
- `client.log` contains GAM trace lines — v18u14 and earlier: lines with `GAMTrace-` prefix; v18u15+ (Beta): lines with `DEBUG genexus.security.api.` logger

---

## log.config Deep Dive
### Default Configuration
The default GeneXus `log.config` has these relevant sections:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<log4net>
	<appender name="RollingFile" type="log4net.Appender.RollingFileAppender">
		<file value="client.log"/>
		<appendToFile value="true"/>
		<maximumFileSize value="9000KB"/>
		<maxSizeRollBackups value="4"/>
		<layout type="log4net.Layout.PatternLayout">
			<conversionPattern value="%date [%thread] %-5level %logger - %message%newline"/>
		</layout>
	</appender>

	<logger name="GeneXusUserLog">
		<level value="ERROR"/>
	</logger>

	<root>
		<level value="OFF"/>
		<appender-ref ref="RollingFile"/>
	</root>
</log4net>
```

### Key Points
- Root level: default `OFF`, required `ALL`
- GeneXusUserLog level: default `ERROR`, leave as-is (root level override is sufficient)
- File path: default `client.log` (relative to `bin/`), no change needed
- Max file size: default 9000KB (~9MB), increase if tracing high-traffic scenarios
- Max backups: default 4, up to 4 rolled files: `client.log.1`, `.2`, `.3`, `.4`

### File Path Resolution
- `<file value="client.log"/>` resolves to `<working_directory>/client.log`
- For IIS deployments, the working directory is the `bin/` folder
- For Kestrel (.NET Core), the working directory is typically the `web/` folder
- To use an absolute path: `<file value="C:\Logs\gam_trace.log"/>`

### Log Rotation
When `client.log` reaches 9000KB:
- `client.log` is renamed to `client.log.1`
- Previous `.1` becomes `.2`, etc
- `.4` is deleted (max 4 backups)
- A new empty `client.log` is created

Warning: in high-traffic scenarios with tracing enabled, log rotation can happen rapidly. Increase `maximumFileSize` and `maxSizeRollBackups` if needed

---

## Browser Automation Testing
Moved to [Playwright Testing for GAM](testing-playwright.md) — Playwright script pattern for GAM login verification, network capture, and AJAX response interpretation

---

## Common GAM Login Issues
Moved to [Common GAM Login Issues](common-login-issues.md) — page-refresh-on-login, no-redirect, locked admin, silent skip, login loop, shared-DB collision

---

## Key Database Queries for Diagnosis
Moved to [GAM SQL Diagnostic Queries](sql-diagnostic-queries.md) — queries for system configuration, users, repositories, applications, authentication types, security policies, and sessions

---

## Environment-Specific Trace Details
- [.NET Core](../environments/environment-netcore.md) — stdout logging, Kestrel OutOfProcess, session cookies
- [.NET Framework](../environments/environment-netframework.md) — IIS InProcess, `ASP.NET_SessionId`
- [Java](../environments/environment-java.md) — Tomcat, `client.cfg`, log4j/logback

---

## Diagnostic Workflow (Step by Step)
### Phase 1 — Verify Infrastructure
```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost/MyApp/login.aspx
# Expected: 200

sqlcmd -S localhost -d MyAppDB -E -Q "SELECT TOP 1 SysParId FROM gam.SysPar"
# Expected: rows returned without error

sqlcmd -S localhost -d MyAppDB -E -Q "SELECT SCHEMA_NAME FROM INFORMATION_SCHEMA.SCHEMATA WHERE SCHEMA_NAME = 'gam'"
# Expected: 'gam' row returned
```

### Phase 2 — Enable Tracing
```sql
UPDATE gam.SysPar SET SysParVal = '1' WHERE SysParId = 'EnableTracing';
UPDATE gam.RepositoryProp SET RepPropVal = '1' WHERE RepPropId = 'EnableTracing';
```

```xml
<root>
	<level value="ALL" />
	<appender-ref ref="RollingFile" />
</root>
```

```powershell
& "$env:windir\system32\inetsrv\appcmd.exe" recycle apppool /apppool.name:"DefaultAppPool"
```

### Phase 3 — Reproduce the Issue
Use Playwright — see [Playwright Testing for GAM](testing-playwright.md)

```bash
node playwright_login_test.js 2>&1 | tee playwright_output.log
```

Capture:
- Network log (requests/responses)
- Console messages
- Screenshot
- AJAX response body

### Phase 4 — Read and Analyze Traces
- Locate the log file: `<KB>/NetModel/web/bin/client.log`
- Detect format and search for GAM trace markers (see [GAM Trace Analyzer](common-trace-analyzer.md)):
	* v18u14 and earlier: lines containing `GAMTrace-` prefix
	* v18u15+ (Beta): lines with logger `genexus.security.api.*` at `DEBUG` level, with optional `{"data":{…}}` JSON payload
- Key principle: every GAM trace is emitted inside a conditional block. If an expected trace is absent from the log, the condition was `false`. The absence of a trace is the strongest diagnostic evidence — stronger than any error message
- Apply 5-pass analysis from [GAM Trace Analyzer](common-trace-analyzer.md):
	* Pass 1 (Detect + Segment) — detect format (A or B), extract chronological sequence, classify domain
	* Pass 2 (Code-to-Log) — build present/absent checklist using format-specific patterns
	* Pass 3 (SDT/Data Correlation) — match JSON dumps to decision branches
	* Pass 4 (Cross-log) — correlate events across multiple logs using state parameters
	* Pass 5 (Deep dive) — insert debug statements in generated code if needed

### Phase 5 — Database Verification
Run the diagnostic queries from [GAM SQL Diagnostic Queries](sql-diagnostic-queries.md) to verify user state, repository configuration, application setup, authentication types, and security policies

### Phase 6 — Formulate Diagnosis
Extended diagnostic format — canonical in `SKILL.md` § OUTPUT (Root Cause → Evidence → Code Path → Recommendation → Confidence; add DB state to Evidence when SQL queries were run). Do not restate here

---

## Trace Activation Cheat Sheet
```
+----------------------------------------------------+
|  CAN I SEE GAM TRACES?                             |
|                                                    |
|  gam.SysPar.EnableTracing = '1'?                   |
|    NO  --> UPDATE it to '1'                        |
|    YES |                                           |
|        v                                           |
|  gam.RepositoryProp.EnableTracing = '1'?           |
|    NO  --> UPDATE it to '1'                        |
|    YES |                                           |
|        v                                           |
|  log.config root level = 'ALL' or 'DEBUG'?         |
|    NO  --> Change from 'OFF' to 'ALL'              |
|    YES |   (CRITICAL for v18u15+: also enables     |
|        |    Log.IsDebugEnabled() in GAM code)      |
|        v                                           |
|  AppPool recycled / 60s waited?                    |
|    NO  --> Recycle or wait                         |
|    YES |                                           |
|        v                                           |
|  Check: {KB}/NetModel/web/bin/client.log           |
|  v18u14-: Should contain "GAMTrace-" lines         |
|  v18u15+: Should contain "genexus.security.api."   |
+----------------------------------------------------+
```

---

## Troubleshooting Tracing Itself
### Traces Still Not Appearing
- `client.log` does not exist — file system permissions. Grant write permission to IIS AppPool identity on `bin/` directory
- `client.log` exists but empty — root level still `OFF`. Check BOTH `log.config` files (`bin/` and `web/`)
- `client.log` has entries but no `GAMTrace-` and no `genexus.security.api.` — DB switches not active. Verify both SysPar AND RepositoryProp, then recycle AppPool
- `client.log` has old traces but not new — cache not expired. Recycle AppPool or wait 60 seconds
- `client.log` has GAM traces but missing expected ones — condition was false in GAM code. This IS the diagnosis: the absent trace means the branch was not taken (see [GAM Trace Analyzer](common-trace-analyzer.md))
- `client.log` has framework DEBUG lines but no `genexus.security.api.` (v18u15+) — `Log.IsDebugEnabled()` returning false for GAM loggers. Check log.config: root level must be `DEBUG` or `ALL`, and no logger-specific filter is blocking `genexus.security.api.*`

### log.config Not Being Read
- Verify the file is well-formed XML (no encoding issues, no BOM problems)
- Verify the file name is exactly `log.config` (case-sensitive on Linux deployments)
- For .NET Core: ensure `log.config` is in the correct directory relative to the DLL

---

## SLO with Non-GAM Clients
Moved to [Debugging SLO with Non-GAM Clients](../logout/slo-non-gam-debugging.md) — diagnosing SLO chains where sub-clients do not have GAM applied

---

## Cache Debugging
Moved to [GAM Cache Debugging](cache-debugging.md) — stale cache symptoms, invalidation via `GAMRepository.ClearCache()`, cache timeout queries

---

## Decision Points
Before starting any diagnostic workflow, check which Decision Points apply and present them

### DP-1: Generator / Platform
- Trigger: user reports an issue and hasn't specified the platform
- Question: "What platform is the application deployed on — `.NET Core`, `.NET Framework`, or `Java`?"
- Options:
	* `.NET Core` — log paths: `web/logs/stdout_*.log`, `web/bin/client.log`. Config: `appsettings.json`, `log.config` in `web/bin/`
	* `.NET Framework` — log paths: Windows Event Log, `web/client.log`. Config: `web.config`, `log.config` in `web/`
	* `Java` — log paths: `catalina.out`, `web/client.log`. Config: `client.cfg`, `log4j.properties`
- Impact: changes log paths, trace config format, and diagnostic commands
- Phase: debug

### DP-2: Trace State
- Trigger: ALWAYS — before any trace analysis
- Question: "Are GAM traces already activated (all 3 switches: SysPar + RepositoryProp + log.config)? Or do you need guidance to activate them?"
- Options:
	* `activated` — go directly to trace analysis with 5-pass methodology
	* `unknown` — (DEFAULT) first verify with SQL queries whether all 3 switches are active. If not, guide activation
	* `need_to_activate` — guide step-by-step activation of all 3 switches (see "Activating GAM Traces (3-Step Process)" above)
- Impact: without active traces, any analysis is incomplete. This DP prevents hours of empty diagnostics
- Phase: debug

### DP-3: Access Level
- Trigger: when diagnostic requires SQL queries or server-side access
- Question: "What level of server access do you have — `SQL + server`, `server only` (logs), or `browser only`?"
- Options:
	* `sql_server` — (DEFAULT) access to SQL Server + server files. Full diagnostics: traces + SQL + logs
	* `server_only` — access to server files/logs but no direct SQL. Diagnostics via traces and stdout
	* `browser_only` — browser access only. Diagnostics limited to Playwright / network capture / screenshots
- Impact: determines which debugging techniques are available — SQL diagnosis ([GAM SQL Diagnostic Queries](sql-diagnostic-queries.md)), traces ("Activating GAM Traces (3-Step Process)" above), or Playwright ([Playwright Testing for GAM](testing-playwright.md))
- Phase: debug

### DP-4: Issue Domain
- Trigger: when the user reports a generic problem ("not working", "error 500", "broken")
- Question: "Can you provide more detail? Is it a problem with `login`, `logout/SLO`, `permissions`, `expired session`, `deploy/connection`, or `other`?"
- Options:
	* `login` — load `../authentication/external-providers/common/domain-auth.md`
	* `logout` — load `../logout/domain-slo.md`
	* `permissions` — load `../authorization/domain-authz.md`
	* `session` — load `../authentication/external-providers/common/domain-session-token.md`
	* `deploy` — load `../multi-tenant/domain-connection-config.md` + environment skill
	* `other` — request a more specific description
- Phase: debug
