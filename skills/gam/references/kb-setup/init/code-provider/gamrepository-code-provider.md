---
name: gamrepository-code-provider
description: Capabilities and quirks for idempotent initialization of GAMRepository. Idempotency strategy C — Singleton. EO is source of truth for properties and methods
---

# GAMRepository Code Provider
**EO**: `GAMRepository` in `ref/GeneXusSecurity/` — locate by content header, not by an assumed file-name suffix (see [global-constraints.md § Where GAM's EOs live](../../../global-constraints.md#where-gams-eos-live))
Derive the SDT using the [SDT Generation Workflow](../../../global-constraints.md#sdt-generation-workflow--derive-the-sdt-11-from-the-eo)

---

## What you can use
- **Idempotency strategy**: C, Singleton. `GetId()` (static, `BasedOn = 'GAMKeyNumShort'`) returns the unique repository ID; call `Load(id)` on the instance. `Get()` is the alternative that returns the loaded instance directly. Never use `new()`, since exactly one repository always exists
- **Capabilities**: resolve the singleton id, load/save, error/cache clearing, and event-subscription activation — read the EO for exact method names and parameters
- **Read/Write properties**: derived from EO — covers name, description, default authentication type, default security policy, OAuth access flag, email uniqueness, login attempt limit, and SMTP/email sub-object
- **Email sub-object**: `&GAMRepository.Email` — a nested sub-EO exposing SMTP server settings and notification template flags. Open the `GAMRepositoryEmail` EO (or the Email section of the `GAMRepository` EO) to get its property list
- **Read-only / computed**: `NameSpace` — do not set

## Known facts (not derivable from the EO)
- `&GAMRepository.Email` properties are stored in the GAM DB, NOT in `client.cfg`
- `ClearLastErrors()` MUST come before `ClearCache()`, order is mandatory; always call both at the end of any init orchestration. `ClearLastErrors` returns `GAMBoolean` and needs an assignment target; `ClearCache` is void
- `&GAMRepository.ID = 2` points to the Repository of the KnowledgeBase in any GAM installation 
- `&GAMRepository.ID = 1` points to the GAM Administration Repository of the Knowledgebase in ANY GAM installation
- **Master SDT placement**: `Repository` is the ONLY scalar member of the Master SDT (`<Name>InitializationDefinition`). **Why**: the platform guarantees exactly one repository; `new()` is forbidden. All other entities in the Master SDT are `Collection = 'True'`. The Procedure accesses it as `&Master.Repository` — no `For` loop

## API quirks and order-of-operations
- Call `ClearLastErrors()` then `ClearCache()` LAST in the overall init — after all entities are saved
- Email authentication fields (`ServerAuthenticationUserName`, `ServerAuthenticationUserPassword`) should only be set when `ServerUsesAuthentication = True`
- Subject/Body notification fields: leave empty (`!""`) to keep GAM built-in templates. `%1` = app name, `%2` = link URL, `%3` = new email (change-email only)

---

## Decision Points — Email configuration (Email configuration it's an example)
### DP-EmailSMTP: SMTP Server
- Trigger: User asks to configure email (SMTP, OTP delivery, account activation, password recovery, or any `&GAMRepository.Email.*` property)
- Action: Ask ALL of the following before generating any DataProvider or Procedure:
	* SMTP server host? — e.g. `smtp.gmail.com`, `smtp.office365.com`
	* SMTP port? — DEFAULT: `587` (STARTTLS); `465` = implicit SSL; `25` = unencrypted
	* Use secure connection (SSL/TLS)? — DEFAULT: `True`
	* Server timeout in seconds? — DEFAULT: `30`
	* Sender email address? — the "from" address recipients see
	* Sender display name?
	* Does the server require authentication? — DEFAULT: `True`
	* *(if auth = True)* SMTP authentication username?
	* *(if auth = True)* SMTP password or app-password? — Gmail requires an App Password, not the account password
- Phase: configure

### DP-EmailNotifications: Notification templates
- Trigger: User mentions account activation email, password recovery, or email templates
- Action: Ask whether to customize (defaults work if left empty):
	* Send email when user activates account? — DEFAULT: `True`
	* Send email when user changes password? — DEFAULT: `True`
	* Send email when user changes email/username? — DEFAULT: `True`
	* Send email for password recovery? — DEFAULT: `True`
	* Custom subject/body for each? — DEFAULT: leave empty (GAM uses built-in templates)
- Phase: configure

---

## Cross-references
- EO: `GAMRepository` in `ref/GeneXusSecurity/`
- [SDT Generation Workflow](../../../global-constraints.md#sdt-generation-workflow--derive-the-sdt-11-from-the-eo)
- [GAM Entity Initialization — Declarative Pattern](../entity-initialization.md)
- [GAM Entity Initialization — Consolidated Pattern](../entity-initialization-consolidated.md) (default pattern)
- [Backoffice: repository settings](../../../backoffice/repository.md)
