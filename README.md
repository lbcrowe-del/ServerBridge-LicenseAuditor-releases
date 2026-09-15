# ServerBridge License Auditor: downloads

Official downloads of the **ServerBridge License Auditor**, the paid command-line tool that finds
Microsoft 365 licenses nobody uses and writes a PDF report of what you can recover.
Binaries only; the source code is private.

- **Buy a license:** https://server-bridge.com/license-auditor.html
- **Just want the free scan?** `Install-Module ServerBridge.LicenseScan -Scope CurrentUser`
  ([ServerBridge-LicenseScan](https://github.com/lbcrowe-del/ServerBridge-LicenseScan))

## Download

Get the newest version from **[Releases](https://github.com/lbcrowe-del/ServerBridge-LicenseAuditor-releases/releases/latest)**:

| Your computer | File | Signed |
|---|---|---|
| Windows (64-bit) | `licenseauditor-win-x64.zip` | Yes: Authenticode, Lee Crowe Software Solutions LLC |
| Mac with Apple silicon (M1 or newer) | `licenseauditor-osx-arm64.tar.gz` | Yes: Apple Developer ID, notarized by Apple |
| Linux (64-bit) | `licenseauditor-linux-x64.tar.gz` | No |

Intel Macs aren't supported yet. Each download is a single self-contained program; nothing else
needs installing.

## Run it

**Windows** (PowerShell):

```powershell
Expand-Archive licenseauditor-win-x64.zip -DestinationPath C:\Tools\LicenseAuditor
C:\Tools\LicenseAuditor\licenseauditor.exe --report audit.pdf --license-key YOUR-LICENSE-KEY
```

**macOS / Linux:**

```bash
mkdir -p ~/licenseauditor && tar -xzf licenseauditor-*.tar.gz -C ~/licenseauditor
~/licenseauditor/licenseauditor --report audit.pdf --license-key YOUR-LICENSE-KEY
```

A sign-in code appears. Open https://login.microsoft.com/device, enter it, and sign in with a
Microsoft 365 admin account (Global Reader is enough). Then open `audit.pdf`.

You only need `--license-key` the first time; the license is remembered on that computer. Without a
license it runs the free scan (on-screen summary and CSV, no PDF). Run `licenseauditor --help` for
every option.

## What it can access

The Auditor is **read-only**. It never changes anything in your tenant. It asks Microsoft for four
read-only permissions: `User.Read.All`, `Organization.Read.All`, `AuditLog.Read.All` and
`Reports.Read.All`. The first time anyone in your tenant runs it, a Global Administrator approves
them. Your tenant data stays on your computer; only your license key is checked with our licensing
server.

## Guides

- [Scheduled re-audits and drift tracking](docs/SCHEDULED-REAUDITS.md) (Team and Enterprise)
- [Multi-tenant roll-up for MSPs](docs/MULTI-TENANT.md) (Enterprise)

## Check the signature

**Windows:**

```powershell
Get-AuthenticodeSignature C:\Tools\LicenseAuditor\licenseauditor.exe | Format-List Status, SignerCertificate
# Status should be 'Valid'
```

**macOS:**

```bash
codesign -dv --verbose=2 ~/licenseauditor/licenseauditor 2>&1 | grep Authority
# Authority=Developer ID Application: Lee Crowe Software Solutions LLC (JH259S64PW)
```

## Help and legal

Questions: reply to the email your license key came in.

[EULA](https://server-bridge.com/license-auditor-eula.html) ·
[Terms](https://server-bridge.com/license-auditor-terms.html) ·
[Privacy](https://server-bridge.com/license-auditor-privacy.html) ·
[Refunds](https://server-bridge.com/license-auditor-refund.html)

© Lee Crowe Software Solutions LLC
