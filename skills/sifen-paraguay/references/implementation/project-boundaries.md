# Boundaries for the current project

This skill may be used with projects such as `facturapy` and `iCopyFE`.

Known project responsibilities include:
- XML generation;
- XML signing;
- QR generation;
- SIFEN submission;
- lot consultation;
- PDF/KuDE generation;
- email delivery;
- database updates;
- PM2/background jobs.

Do not assume that the GNB architecture must be copied exactly.

Project-specific constraints should be placed in the project's `AGENTS.md` or project-specific documentation, not hard-coded as universal SIFEN rules.

Examples of project-specific decisions that must remain explicit:
- SQL Server vs DB2;
- PM2 on Windows;
- filesystem storage;
- PDF generation technology;
- email worker schedule;
- batch size if different from the SIFEN maximum;
- retry counts/timeouts;
- database transaction boundaries.
