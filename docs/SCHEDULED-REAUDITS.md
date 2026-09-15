# Scheduled re-audits & drift tracking (Team tier)

The Team tier can run the auditor **unattended** on a schedule and report **drift** —
what changed since the last audit ("+3 newly dormant seats since last month"). This guide
sets that up.

Interactive scans use device-code sign-in (a human approves the sign-in). A scheduled task
has no human, so it authenticates **app-only** against an app registration you create once in
your own tenant.

---

## 1. Register an application (one-time, ~5 minutes)

In the [Entra admin center](https://entra.microsoft.com) → **Identity → Applications → App
registrations → New registration**:

1. Name it e.g. `ServerBridge License Auditor (scheduled)`. Single tenant is fine.
2. No redirect URI is needed. Register.
3. Copy the **Application (client) ID** and **Directory (tenant) ID** from the Overview page.

### Grant application permissions

**API permissions → Add a permission → Microsoft Graph → Application permissions**, add:

| Permission | Why |
|------------|-----|
| `User.Read.All` | Read licensed users and their assigned licenses |
| `Organization.Read.All` | Read subscribed SKUs (license inventory) and tenant name |
| `AuditLog.Read.All` | Sign-in activity (last sign-in), when Entra ID P1 is present |
| `Reports.Read.All` | Microsoft 365 usage reports: activity fallback without P1, and (Team) which mailboxes are shared, room or equipment |

Then click **Grant admin consent for &lt;tenant&gt;**. Each permission should show a green check.
These are **read-only** — the app cannot change anything.

> Tip: if your usage reports show concealed (de-identified) names, sign-in-activity via
> `AuditLog.Read.All` + Entra ID P1 gives the most accurate results. Without P1 the auditor
> falls back to usage reports automatically.

### Create a client secret

**Certificates & secrets → Client secrets → New client secret.** Pick an expiry (e.g. 12 or
24 months) and copy the **Value** immediately — you can't see it again. Put a reminder in your
calendar to rotate it before it expires, or the scheduled task will start failing.

---

## 2. Test it once, by hand

Supply the three values as environment variables (preferred — keeps the secret out of the
command line and process list) and run with `--scheduled`:

```bash
export LICENSEAUDITOR_TENANT_ID="<directory-tenant-id>"
export LICENSEAUDITOR_CLIENT_ID="<application-client-id>"
export LICENSEAUDITOR_CLIENT_SECRET="<client-secret-value>"

licenseauditor --scheduled --license-key <your-team-key> --report audit.pdf
```

The first run records a baseline and prints:

> Drift tracking is on. This is the first recorded audit for this tenant.

Run it again and it reports the change since the baseline, both on screen and in the PDF's
**"Change since last audit"** section.

You can also pass the values as flags (`--tenant`, `--client-id`, `--client-secret`) instead
of env vars; explicit flags win over env vars. Scheduled re-audits and drift require a **Team**
(or Enterprise) license key.

---

## 3. Schedule it

### Windows (Task Scheduler)

The most reliable pattern is a tiny wrapper script that sets the secret from a protected store
and invokes the auditor, scheduled to run as a service account.

`run-audit.cmd`:

```bat
@echo off
set LICENSEAUDITOR_TENANT_ID=<directory-tenant-id>
set LICENSEAUDITOR_CLIENT_ID=<application-client-id>
set LICENSEAUDITOR_CLIENT_SECRET=<client-secret-value>
"C:\Tools\LicenseAuditor\licenseauditor.exe" --scheduled ^
  --license-key <your-team-key> ^
  --report "C:\Reports\license-audit-%date:~-4%%date:~4,2%%date:~7,2%.pdf"
```

Register it (every 4 weeks, on a Monday at 07:00), running as the service account:

```powershell
$action  = New-ScheduledTaskAction -Execute "C:\Tools\LicenseAuditor\run-audit.cmd"
$trigger = New-ScheduledTaskTrigger -Weekly -WeeksInterval 4 -DaysOfWeek Monday -At 7am
Register-ScheduledTask -TaskName "License Auditor - monthly" -Action $action -Trigger $trigger `
  -User "DOMAIN\svc-licenseaudit" -RunLevel Limited
```

Lock down `run-audit.cmd` with NTFS permissions since it holds the secret, or set the three
`LICENSEAUDITOR_*` values as **machine** environment variables for the service account instead
of putting the secret in the script.

### Linux / macOS (cron)

Put the secret in a root-only env file (`chmod 600`), e.g. `/etc/license-auditor.env`:

```bash
LICENSEAUDITOR_TENANT_ID=<directory-tenant-id>
LICENSEAUDITOR_CLIENT_ID=<application-client-id>
LICENSEAUDITOR_CLIENT_SECRET=<client-secret-value>
```

Crontab entry (1st of the month, 07:00):

```cron
0 7 1 * * set -a; . /etc/license-auditor.env; set +a; \
  /opt/licenseauditor/licenseauditor --scheduled --license-key <team-key> \
  --report /var/reports/license-audit-$(date +\%Y\%m\%d).pdf
```

---

## Where drift state is stored

To compare month-over-month, the auditor keeps a **small local summary** of each run — tenant
name, dormant-seat identities, counts, and recoverable-spend totals — one JSON file per tenant.
Default locations:

- **Windows:** `%APPDATA%\ServerBridge\LicenseAuditor\drift`
- **macOS:** `~/Library/Application Support/ServerBridge/LicenseAuditor/drift`
- **Linux:** `~/.config/ServerBridge/LicenseAuditor/drift` (or `$XDG_CONFIG_HOME`)

The files are readable only by the account that runs the audit.

Override with `--state-dir <path>` — useful so a service account and your interactive runs
share one history, or to keep the history on a backed-up volume. **Nothing is transmitted**;
the file never leaves the machine. The free and Starter tiers do not write it.

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `Scheduled re-audits requires the Team plan…` | The key isn't Team/Enterprise, or `--license-key` was omitted. |
| `App-only … requires a client secret` | Set `LICENSEAUDITOR_CLIENT_SECRET` or pass `--client-secret`. |
| `App-only auth requires a specific tenant id…` | Use your tenant GUID/domain, not the default. |
| Auth fails with `insufficient privileges` | Admin consent wasn't granted, or a permission is missing. Re-check step 1. |
| Every run says "first recorded audit" | Each run uses a different `--state-dir` (or user profile). Pin one with `--state-dir`. |
