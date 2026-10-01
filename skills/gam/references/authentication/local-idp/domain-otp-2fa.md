---
name: domain-otp-2fa
description: OTP and Two-Factor Authentication — enrollment, verification, recovery, TOTP
---

# GAM OTP & 2FA Encyclopedia
Read [Error Handling](../../debugging/common-genexus-patterns.md#error-handling) in `common-genexus-patterns.md`

---

## OTP and TOTP in GAM
In GAM, OTP and TOTP are **Authentication Types** (standalone AuthType objects), not Security Policy toggles. They can be used in two ways:

- **As first factor (standalone / passwordless)** — the user authenticates directly against the OTP or TOTP AuthType. No password is involved: the user receives a code (OTP via email/SMS) or generates one in an authenticator app (TOTP), and that is the only credential. Useful for passwordless flows. Controlled by the **"Use For First Factor Authentication?"** checkbox on the OTP/TOTP AuthType
- **As second factor (2FA)** — the OTP/TOTP AuthType is referenced from another primary AuthType via its `TwoFactorAuthentication` sub-SDT. The user first authenticates with the primary AuthType, and then must complete the OTP/TOTP step. The "Use For First Factor Authentication?" checkbox stays unchecked in this mode

TOTP inherits this dual-role behaviour from OTP: if OTP can be used as either factor, TOTP can too

- OTP (One Time Password)
	* GAM AuthType: OTP (also `OTP Custom` variant available)
	* Mechanism: Numeric code delivered via email (SMS not listed in the GAM 2FA wiki as a supported medium — verify with the repository's email/SMS gateway configuration)
	* Duration: Configurable (code expiration in seconds)
	* Enrollment: Automatic (user must have Email loaded)
	* Available since: GeneXus 17 Upgrade 5
- TOTP (Time-Based OTP)
	* GAM AuthType: TOTP
	* Mechanism: Rotating code from an authenticator app (Google Authenticator, Microsoft Authenticator, Authy, etc.)
	* Duration: 30 seconds (standard TOTP window)
	* Enrollment: Manual — user clicks **"Enable authenticator"** in their profile, scans the QR with the app, enters the generated code, confirms
	* Secret key length default: 16 characters (iPhone/Google Authenticator limit)
	* Available since: GeneXus 17 Upgrade 8

---

## Configuration (Backoffice)
### 2.A — OTP/TOTP as FIRST factor (passwordless)
Use this pattern when the OTP or TOTP AuthType is the only credential the user provides

- **Create the OTP (or TOTP) Authentication Type** (see Step 1 below)
- **Set the user's `AuthenticationTypeName`** to the OTP (or TOTP) AuthType in Backoffice → GAM_Users → user → Identity. No password is stored or required
- For OTP: ensure the user has Email (for `OTPDelivery = Email`) or Phone (for SMS), and SMTP/SMS is configured at repository level
	For TOTP: the user enrolls (QR scan) on first login

No `TwoFactorAuthentication.Enable` is needed in this mode — the OTP/TOTP AuthType IS the primary credential

### 2.B — OTP/TOTP as SECOND factor (2FA)
2FA wiring uses a 3-object flow, in order:

- **Create an OTP (or TOTP) Authentication Type** — the second-factor AuthType (checkbox "Use For First Factor Authentication?" stays UNCHECKED)
- **Edit the primary Authentication Type** and enable its **Two Factor Authentication** section pointing to the OTP/TOTP AuthType from step 1
- **Enable 2FA on the user** (or force it globally from the primary AuthType's 2FA section)

Primary AuthType eligibility for 2FA wiring (per GAM 2FA wiki): **OTP, Local, Custom, WebService, GAMRemoteRest**. Other external-IDP AuthTypes (OAuth 2.0, SAML 2.0, Google, Facebook, Apple, Twitter, LinkedIn, Instagram) are not listed as supported — for those, enforce MFA at the external IDP side

Do NOT look for 2FA/OTP fields under Security Policies — they do not live there

### Step 1 — Create the OTP or TOTP Authentication Type
Backoffice → **Settings → Authentication Types → Add**. Select the type (`OTP` or `TOTP`) and set its properties:

#### OTP AuthType properties
- `Name`
	* Purpose: Unique AuthType name (e.g. `otp`)
- `IsEnable`
	* Purpose: Enables this AuthType
	* Default: True
- **"Use For First Factor Authentication?"** (Backoffice checkbox)
	* Purpose: If checked, this OTP AuthType acts as a primary (passwordless) AuthType. If unchecked, it is used exclusively as a second factor
	* Default: Unchecked
- `OTPLength`
	* Purpose: Length of the numeric code
	* Default: 6
- Code expiration timeout (seconds)
	* Purpose: Seconds the code is valid
- Maximum daily number of codes
	* Purpose: Daily cap on OTP codes generated per user
- Number of unsuccessful retries to lock OTP
	* Purpose: Attempts before the OTP codeset is locked
- Automatic OTP unlock time
	* Purpose: Time after which the lock is released automatically
- Number of unsuccessful retries to block user based on OTP locks
	* Purpose: Repeated OTP locks that lead to user block
- SMTP (for Email delivery) must be configured at repository level — Backoffice: Repositories → select repo → **Emails** tab, or via code `&GAMRepository.Email.*` — see [GAM Backoffice — Repository (GAM_Repository)](../../backoffice/repository.md#tab-emails-gam_emails)

#### TOTP AuthType properties
- `Name`
	* Purpose: Unique AuthType name (e.g. `totp`)
- `IsEnable`
	* Purpose: Enables this AuthType
	* Default: True
- **"Use For First Factor Authentication?"** (Backoffice checkbox)
	* Purpose: Same semantics as OTP — primary vs. second-factor mode
	* Default: Unchecked
- TOTP secret key length
	* Purpose: Length of the shared secret
	* Default: 16 (iPhone/Google Authenticator upper limit)
- `TOTPIssuer`
	* Purpose: Name displayed in the authenticator app (appears next to the account)
	* Default: Application name
- Code expiration timeout / retry / lock / daily-cap properties apply as in OTP

### Step 2 — Enable 2FA on the primary Authentication Type
Backoffice → **Settings → Authentication Types** → open the primary AuthType (OTP, Local, Custom, WebService, or GAMRemoteRest) → section **Two Factor Authentication**

Backoffice checkboxes / properties:

- **"Enable Two Factor Authentication?"** → `TwoFactorAuthentication.Enable`
	* Purpose: Master switch for 2FA on this primary AuthType
	* Default: False
- **"Authentication Type Name"** (combo) → `TwoFactorAuthentication.AuthenticationTypeName`
	* Purpose: Reference to the OTP or TOTP AuthType created in Step 1
	* Required when "Enable Two Factor Authentication?" is checked
- `TwoFactorAuthentication.FirstAuthenticationFactorExpiration`
	* Purpose: **Seconds** the first factor remains valid while awaiting the second factor
	* Numeric 9
	* Default: 900 (15 minutes)
- **"Force 2FA for all users?"** → `TwoFactorAuthentication.ForceForAllUsers`
	* Purpose: All users authenticating with this primary AuthType MUST complete 2FA, regardless of their individual flag
	* Default: False
- `TwoFactorAuthentication.IsOTP` / `IsTOTP`
	* Purpose: Read-only flags reflecting whether the linked AuthType is OTP or TOTP

### Step 3 — Enable 2FA on the user (unless forced globally)
Backoffice → **GAM_Users** → user → **"Enable two factor authentication?"** checkbox (maps to `GAMUser.EnableTwoFactorAuthentication`, exposed at IDE level as `&UserEnableTwoFactorAuth` — Boolean, allows null)

Per-user requirements:
- OTP delivery via Email → user must have **Email** loaded
- OTP delivery via SMS (if gateway configured) → user must have **Phone** loaded
- TOTP → user enrolls (clicks "Enable authenticator" in their profile, scans QR, enters code, confirms)

If `TwoFactorAuthentication.ForceForAllUsers = True` on the primary AuthType, the user-level flag is overridden and every user must complete 2FA

### Where properties do NOT live
- **Security Policy** does NOT contain 2FA / OTP / TOTP fields. The Security Policy governs passwords, sessions, OAuth tokens — not the second factor
- **Repository Properties** do NOT contain 2FA switches either. The wiring is entirely at AuthType level

---

## SDT Structure
The `TwoFactorAuthentication` sub-SDT exists on the AuthTypes eligible as primary for 2FA (OTP, Local, Custom, WebService, GAMRemoteRest). It is how the primary AuthType links to the second-factor OTP/TOTP AuthType:

```json
{
  "TwoFactorAuthentication": {
	"Enable": false,
	"AuthenticationType": "",
	"AuthenticationTypeName": "",
	"FirstAuthenticationFactorExpiration": 0,
	"ForceForAllUsers": false,
	"IsOTP": false,
	"IsTOTP": false
  },
  "UseTwoFactorAuthentication": false
}
```

- `Enable` — master switch for 2FA on the primary AuthType
- `AuthenticationType` / `AuthenticationTypeName` — reference to the OTP or TOTP AuthType
- `FirstAuthenticationFactorExpiration` — **seconds** (not minutes) the first factor is valid awaiting 2FA. Default 900 (15 min)
- `ForceForAllUsers` — override per-user opt-in
- `IsOTP` / `IsTOTP` — read-only, reflects the linked AuthType kind
- `UseTwoFactorAuthentication` — top-level flag at user/session level

---

## Flows
### Login with 2FA -- Complete Flow
- User submits username and password
- GAM validates the password. If correct, GAM checks the primary AuthType's `TwoFactorAuthentication.Enable` and whether 2FA is required for this user
- If 2FA is required, GAM saves the first-factor state (user ID, validation timestamp, expiration) as a temporary record and returns a "2FA Required" response without creating a session
- The UI presents the second-factor screen
- OTP path: GAM generates a random numeric code (length defined by `OTPLength` on the OTP AuthType), stores a hashed copy with an expiration timestamp, and sends the plaintext code to the user via email or SMS
	TOTP path: The user opens their authenticator app, which generates a time-based code from the shared secret
- The user enters the code
- GAM validates the code:
	* OTP: GAM compares the hash of the submitted code against the stored hash and checks that the code has not expired
	* TOTP: GAM generates the expected code for the current 30-second window using the shared secret. It also checks the immediately preceding and following windows to allow for minor clock drift
- If valid, GAM creates a full session and returns the access token
- If invalid, GAM increments the failed attempt counter and returns an error

### TOTP Enrollment (First Time)
- GAM generates a cryptographic secret key (Base32-encoded)
- The secret is associated with the user's account
- GAM constructs an `otpauth://totp/` URI containing the `TOTPIssuer`, user identifier, and secret
- This URI is rendered as a QR code for the user to scan with their authenticator app
- After scanning, the user enters a verification code to confirm enrollment is working

### OTP Generation and Delivery
- GAM generates a random numeric code of the configured length (`OTPLength`)
- The code is stored in hashed form with an expiration timestamp (based on the code expiration timeout, in seconds)
- GAM sends the plaintext code to the user's registered email or phone number, depending on the OTP delivery setting
- The stored record is single-use: once validated (or expired), it is deleted

### REST 2FA Flow (`/oauth/gam/v2.0/access_token`)
REST clients complete 2FA in two POST calls against the GAM REST IDP endpoint:

**First POST** — first factor:

```
POST https://<domain>/<virtual_directory>/oauth/gam/v2.0/access_token
Body (form):
  client_id=<…>
  client_secret=<…>
  grant_type=password
  username=<user>
  password=<password>
  authentication_type_name=<primary AuthType name>   // e.g. local
  scope=<…>                                          // conditional
```

Result: GAM generates and sends the OTP code via email (or SMS). The response indicates that the second factor is required — no access_token is returned yet

**Second POST** — second factor:

```
POST https://<domain>/<virtual_directory>/oauth/gam/v2.0/access_token
Body (form):
  client_id=<…>
  client_secret=<…>
  grant_type=password
  username=<user>
  password=<OTP code>                                 // the code just received
  authentication_type_name=<primary AuthType name>
  use_2fa=true
  otp_step=2
```

Result: if the code is valid and the first-factor state has not expired, GAM returns the full `access_token`

**UserInfo (after 2FA)**:

```
GET https://<domain>/<virtual_directory>/oauth/gam/v2.0/userinfo
Header: Authorization: <access_token>
```

Returns the user profile JSON (guid, username, email, name, phone, etc.)

Key points:
- `use_2fa=true` + `otp_step=2` are what signal the second-factor submission
- The `password` field in the second POST carries the OTP code, not the original password
- If `FirstAuthenticationFactorExpiration` elapses between the two POSTs, the second POST fails with "First factor expired"

---

## Trace Signatures -- OTP/2FA
### Login with 2FA
```
OK  GAMTrace-GAMAuthenticationLogin 2FA - Valid &AuthenticationType2FA:<value>
OK  GAMTrace-2FA: First factor OK, awaiting second factor
OK  GAMTrace-OTP: Code sent to <email>
OK  GAMTrace-TOTP: Enrollment QR generated for user <userId>
OK  GAMTrace-2FA: Second factor validated, session created
ERR GAMTrace-2FA: First factor expired
ERR GAMTrace-2FA: Second factor FAILED
```

### Detection by Absence
- `2FA - Valid &AuthenticationType2FA:0`
	* Means: the primary AuthType does NOT have `TwoFactorAuthentication.Enable = True`, or the linked OTP/TOTP AuthType is missing
- `First factor OK` but NO `Second factor validated`
	* Means: User did not complete the 2FA step
- `OTP: Code sent` but NO `Second factor validated`
	* Means: Code expired or was entered incorrectly

---

## Error Codes -- OTP/2FA
- "OTP code not received"
	* Cause: Email/SMTP config incorrect, or user has no Email/Phone loaded
	* Error Code: --
	* Fix: Verify SMTP in Backoffice → Repositories → select repo → **Emails** tab (or via `&GAMRepository.Email.*` code), and that the user's Email (or Phone for SMS) is populated
- "Code expired"
	* Cause: User took longer than the OTP AuthType's code expiration timeout (seconds)
	* Error Code: 17
	* Fix: Generate a new code, or raise the code expiration timeout on the OTP AuthType
- "Invalid code"
	* Cause: Wrong code entered
	* Error Code: --
	* Fix: Retry. If persistent, check TOTP clock sync
- "TOTP validation fails always"
	* Cause: Clock drift > 30s between server and phone
	* Error Code: --
	* Fix: Synchronize the server's NTP
- "2FA forced but can't enroll"
	* Cause: `TwoFactorAuthentication.ForceForAllUsers = True` on the primary AuthType, but the user never completed TOTP enrollment
	* Error Code: --
	* Fix: Add an enrollment screen before the login flow, or provision users with pre-enrolled TOTP secrets
- "First factor expired"
	* Cause: `TwoFactorAuthentication.FirstAuthenticationFactorExpiration` too low on the primary AuthType
	* Error Code: 17
	* Fix: Increase `FirstAuthenticationFactorExpiration` in the primary AuthType's 2FA section (NOT in the Security Policy)

---

## Recovery (Device Loss)
### TOTP Recovery Options
- Admin reset
	* How it works: An administrator deactivates TOTP for the user. The user re-enrolls on their next login. This clears the stored TOTP secret and enrollment flag
- Backup codes
	* How it works: Single-use recovery codes generated during enrollment (if implemented). The user stores these offline as a fallback
- Fallback to OTP
	* How it works: If both an OTP AuthType and a TOTP AuthType exist, an alternate primary AuthType can reference the OTP one so the user receives a code via email as an alternative second factor

---

## Debugging 2FA -- Checklist
### Pass 1 -- Configuration
First determine the mode: OTP/TOTP as first factor, or as second factor (2FA)

If OTP/TOTP is the FIRST factor (passwordless):
- [ ] An **OTP** or **TOTP** Authentication Type exists and `IsEnable = True`
- [ ] The user's `AuthenticationTypeName` is the OTP/TOTP AuthType
- [ ] Email/SMTP configured (if OTP via Email); user has Email loaded
- [ ] Server NTP is in sync (if TOTP)

If OTP/TOTP is the SECOND factor:
- [ ] An **OTP** or **TOTP** Authentication Type exists and `IsEnable = True`
- [ ] The **primary AuthType** (Local or Custom) has `TwoFactorAuthentication.Enable = True` and `AuthenticationTypeName` points to the OTP/TOTP AuthType
- [ ] `FirstAuthenticationFactorExpiration` > 0 on the primary AuthType's 2FA section
- [ ] Either `ForceForAllUsers = True` on the primary AuthType, OR the specific user has `EnableTwoFactorAuthentication = True`
- [ ] Email/SMTP configured (if OTP via Email); user has Email loaded
- [ ] Correct type selected: OTP vs TOTP

### Pass 2 -- Traces
- [ ] Is `AuthenticationType2FA` present with a value > 0? If 0, the primary AuthType does NOT have 2FA enabled, or the linked OTP/TOTP AuthType is missing or disabled
- [ ] Is `First factor OK` present? If not, the password failed (see domain-auth.md)
- [ ] Is `OTP: Code sent` present (if OTP)? If not, there is a delivery problem (SMTP, missing Email on user)
- [ ] Is `Second factor validated` present? If not, the code was invalid or expired

### Pass 3 -- Deep Investigation
- [ ] If traces are insufficient, enable full GAM tracing (see common-debugging.md) and reproduce the issue
- [ ] Check the decision points in the validation flow: is the code comparison failing, or is the first-factor state expiring before the user submits?
- [ ] For TOTP failures, verify server clock accuracy (NTP sync) -- a drift of more than 30 seconds will cause persistent validation failures

---

## Decision Points
Before configuring 2FA/OTP, Claude MUST check which Decision Points apply

### DP-0: Factor Role
- Trigger: User asks to configure OTP / TOTP / 2FA / MFA / passwordless
- Question: "Do you want OTP/TOTP as the **first factor** (passwordless — the user authenticates directly with the code) or as the **second factor** (after password / custom login)?"
- Options:
	* `first_factor` — OTP or TOTP replaces the password. User's `AuthenticationTypeName` is set to the OTP/TOTP AuthType. No 2FA wiring needed
	* `second_factor` — (DEFAULT) OTP/TOTP runs after a primary AuthType. Requires wiring via `TwoFactorAuthentication` on the primary AuthType (Local or Custom)
- Impact:
	* `first_factor`: Stop after creating the OTP/TOTP AuthType and assigning it to the user. Skip DP-2 and the 2FA wiring steps
	* `second_factor`: Proceed with the full 3-object flow (OTP/TOTP AuthType → primary AuthType 2FA section → user)
- Phase: configure

### DP-1: OTP Method
- Trigger: User asks to configure second factor / 2FA / MFA
- Question: "What second factor method? `OTP` (code via email/SMS, single use) or `TOTP` (authenticator app like Google Authenticator, time-based)?"
- Options:
	* `otp` -- Code sent via email or SMS. Simpler for users. Requires SMTP/SMS configured
	* `totp` -- (DEFAULT) Code generated by authenticator app (QR enrollment). Does not require email/SMS. More secure
- Impact:
	* `otp`: Create an **OTP** AuthType. Requires SMTP (email) or SMS service configuration and user with Email/Phone. Code has a fixed expiration
	* `totp`: Create a **TOTP** AuthType. Requires enrollment via QR code. User needs an authenticator app. Based on 30-second time windows
- Phase: configure

### DP-2: Enforcement Level (second-factor mode only)
- Trigger: When configuring 2FA with `second_factor` selected in DP-0
- Question: "Second factor mandatory for all users, or only for certain users?"
- Options:
	* `all` -- (DEFAULT) 2FA mandatory for everyone authenticating with the primary AuthType. Set `ForceForAllUsers = True` on the primary AuthType's 2FA section
	* `opt_in` -- User-by-user. Keep `ForceForAllUsers = False`; set `EnableTwoFactorAuthentication = True` on each user that must use it
	* `by_auth_type` -- Only certain primary AuthTypes have 2FA wired (e.g. `Local` has 2FA, `Custom` does not, or vice versa). Enable `TwoFactorAuthentication.Enable` only on the AuthTypes (Local/Custom) where you want 2FA
- Impact: 2FA activation lives entirely on the primary AuthType (Local or Custom) + per-user flag. It does NOT live in the Security Policy, does NOT live in the Repository, and is not available on external IDP-backed AuthTypes (OAuth/SAML/Google/…)
- Phase: configure

### DP-3: Recovery
- Trigger: When configuring TOTP
- Question: "Configure recovery codes (backup codes) for when the user loses access to the authenticator app?"
- Options:
	* `yes` -- (DEFAULT) Generate recovery codes during enrollment. User stores them as backup
	* `no` -- No recovery. If the app is lost, an admin must manually reset the user's 2FA
- Impact: Recovery codes add operational resilience but require the user to save them securely. Without recovery, the burden falls on support/admin. The GAM 2FA wiki does NOT explicitly document a built-in backup-codes feature — confirm availability against the target GAM version before promising it to the user
- Phase: configure

---

## Availability (version matrix)
- OTP as AuthType + 2FA wiring: since GeneXus 17 Upgrade 5
- TOTP as AuthType (incl. 2FA wiring): since GeneXus 17 Upgrade 8
- Mobile flows: separate wiki pages exist for OTP mobile, TOTP mobile, and 2FA for mobile. Consult those when the client is a Smart Device app, as behaviour (e.g. code-delivery UI, enrollment) may differ from Web

## Primary-text wiki references
- GAM — Two Factor Authentication (2FA): docs.genexus.com wiki 48254
- GAM — One Time Password (OTP): docs.genexus.com wiki 48197
- GAM — Time Based One Time Password (TOTP): docs.genexus.com wiki 49974
- OTP for mobile: 50664 / TOTP for mobile: 50708 / 2FA for mobile: 50726
