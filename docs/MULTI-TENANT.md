# Multi-tenant roll-up (Enterprise)

The Enterprise tier can audit **several Microsoft 365 tenants in one run** and produce a single
combined report — built for MSPs and CSPs managing multiple customer tenants. This is the
lightweight, CLI-driven version: each tenant is audited independently and the results are rolled
up locally. (A hosted multi-tenant portal with a consent UI and a drift dashboard is a larger,
separate product — see "What this isn't," below.)

## 1. Write a tenant list

Create a JSON file listing the tenants to audit. Minimum shape — one `tenantId` per tenant:

```json
[
  { "tenantId": "contoso.onmicrosoft.com", "displayName": "Contoso" },
  { "tenantId": "fabrikam.onmicrosoft.com", "displayName": "Fabrikam" }
]
```

- `tenantId` — required. A tenant GUID or verified domain (e.g. `contoso.onmicrosoft.com`).
- `displayName` — optional. Used in the report and console output instead of the raw tenant id.
- `clientId` / `clientSecret` — optional, **per-tenant override**. Most MSPs use one multi-tenant
  app registration consented into every customer tenant, so you won't need these — just pass the
  shared `--client-id`/`--client-secret` (or `LICENSEAUDITOR_*` env vars) once for the whole run.
  Set them per-entry only if a specific customer requires a separate app registration.

Tenant ids must be unique in the file — duplicates are rejected before any scanning starts.

## 2. Set up app-only auth once

Multi-tenant roll-ups authenticate the same way as [scheduled re-audits](SCHEDULED-REAUDITS.md):
app-only (unattended), via an app registration with admin-consented Application permissions
(`User.Read.All`, `Organization.Read.All`, `AuditLog.Read.All`, `Reports.Read.All`). Follow
Section 1 of that guide, then have your customer's admin (or your own if it's a CSP-delegated
tenant) grant consent to the same app in **each** tenant listed in your JSON file.

## 3. Run it

```bash
licenseauditor --tenants-file msp-tenants.json \
  --client-id <your-app-client-id> \
  --client-secret <your-app-client-secret> \
  --license-key <your-enterprise-key> \
  --report rollup.pdf --output-csv rollup.csv
```

(Credentials can also come from `LICENSEAUDITOR_CLIENT_ID`/`LICENSEAUDITOR_CLIENT_SECRET` env
vars, same as `--scheduled`.) The auth mode is app-only automatically for a multi-tenant run —
`--scheduled` is optional and only changes the sign-in message, not the behavior.

Console output shows each tenant as it's scanned, then a roll-up summary:

```
--- Contoso ---
  3 reclaimable seat(s) found.
--- Fabrikam ---
  Failed: AADSTS700016: Application not found in tenant.

Roll-up summary
---------------
  Tenants scanned  : 1 of 2
  Reclaimable seats: 3
  Wasted spend     : $1,296 / year

  Tenants that failed to scan:
    Fabrikam: AADSTS700016: Application not found in tenant.

Roll-up PDF written to:
  rollup.pdf
```

**One tenant's failure never aborts the run** — a missing consent or expired credential for one
customer just gets logged and skipped, and the rest of the roll-up completes. The PDF calls out
any failed tenants by name and error so you know exactly which ones need attention.

## What you get

- **One PDF** (`--report`) — grand-total recoverable spend across all tenants, a per-tenant table
  (licensed users, reclaimable seats, recoverable $/yr), and a callout for any tenants that
  couldn't be scanned.
- **One CSV** (`--output-csv`) — every dormant seat across every tenant, with a `Tenant` column so
  you can filter/pivot per customer, plus `Flag` and `FlagDetail` columns for accounts to review.
- **A review of licenses that may not be needed**, in the PDF and CSV — licensed guest accounts and
  licensed shared, room and equipment mailboxes in each tenant, with a possible extra saving that is
  *not* added to the total. Mailbox types come from each tenant's mailbox usage report
  (`Reports.Read.All`, already required), which Microsoft refreshes every day or two.

## What this isn't

This is the lightweight, CLI-driven roll-up — you run it yourself, on your own schedule, against a
JSON file you maintain. It doesn't include a hosted consent UI, a drift dashboard across tenants,
or server-side scheduling per customer — that's a larger, separate **hosted multi-tenant portal**
tracked as future work (Phase 3 in the roadmap), only worth building once there's revenue demand
for it. If you want scheduled *drift* per tenant today, run this alongside each tenant's own
`--scheduled` / drift setup (Team+) — see [SCHEDULED-REAUDITS.md](SCHEDULED-REAUDITS.md).

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `Multi-tenant roll-ups requires the Enterprise plan…` | The key isn't Enterprise, or `--license-key` was omitted. |
| `Could not read tenant list: …` | The JSON file is missing, malformed, empty, or has a duplicate/missing `tenantId`. The error names which. |
| One tenant shows `Failed: AADSTS…` | That tenant's admin hasn't consented to your app, or the tenant id is wrong. Other tenants still complete. |
| PDF total doesn't match a tenant's own single-tenant report | The roll-up counts *reclaimable seats + unassigned licenses* the same way a single audit does — check the per-tenant table for the exact split. |
