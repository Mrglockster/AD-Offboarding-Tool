# AD Offboarding Tool

PowerShell tooling to offboard a user in one pass: mailbox auto-reply, password reset, account disable, OU move, and Entra ID session revocation. Ships as a WinForms GUI for helpdesk use and a headless script for automation.

![PowerShell](https://img.shields.io/badge/PowerShell-5.1%2B-blue)
![Platform](https://img.shields.io/badge/platform-Windows-lightgrey)
![Version](https://img.shields.io/badge/version-1.0.0-green)

---

## Contents

| File | Purpose |
|---|---|
| `Offboarding-GUI.ps1` | WinForms front end. Search → confirm → offboard, with live activity log. |
| `Invoke-UserOffboarding.ps1` | Headless equivalent. Supports `-WhatIf`, returns a result object. Use for automation or bulk runs. |

Both are self-contained. Neither depends on the other.

---

## What it does

Five actions, each independently toggleable:

1. **Mailbox auto-reply** — enables an internal and external out-of-office via Exchange Online.
2. **Password reset** — generates a cryptographically random password (12–64 chars, default 16).
3. **Account disable** — disables the AD account and stamps the description with the date and operator.
4. **OU move** — relocates the object to your offboarding OU.
5. **Session revocation** — invalidates all Entra ID refresh tokens.

### Order of operations

The sequence is deliberate:

- **Auto-reply runs first**, while the account and mailbox are still fully live. Setting it after a disable works, but leaves a window where mailbox state and directory state disagree.
- **Session revocation runs last**, and goes through Graph rather than AD. An on-premises password reset does not reach Entra until the next Connect sync cycle (30 minutes by default), so the AD reset alone does not terminate cloud sessions. `Revoke-MgUserSignInSession` invalidates refresh tokens immediately.
- **A single DC is pinned** for the whole run via `Get-ADDomainController -Discover -Service PrimaryDC`. Without this, the reset, disable, and move can land on different replicas and produce confusing intermediate states.

Every step is independently error-handled. A failure in one does not abort the rest; the run completes and reports a per-step status.

---

## Requirements

### Modules

| Module | Required | Source | Used for |
|---|---|---|---|
| `ActiveDirectory` | Yes | RSAT | All directory operations |
| `ExchangeOnlineManagement` | Optional | PSGallery | Mailbox auto-reply |
| `Microsoft.Graph.Authentication` | Optional | PSGallery | Graph auth |
| `Microsoft.Graph.Users.Actions` | Optional | PSGallery | Session revocation |

The GUI detects missing PSGallery modules at startup and offers to install them. It handles TLS 1.2, the NuGet provider bootstrap, and PSGallery registration/trust as part of that. Actions whose modules are absent are greyed out rather than failing mid-run.

`ActiveDirectory` is **not** auto-installed — it is an RSAT capability, not a Gallery module:

```powershell
Add-WindowsCapability -Online -Name Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0
```

### Permissions

| Action | Required rights |
|---|---|
| Reset password, disable, move | Delegated control on the source **and** target OUs |
| Mailbox auto-reply | Exchange **Recipient Management** (or a role holding `Mail Recipients`) |
| Revoke sessions | Graph scopes `User.ReadWrite.All` and `User.RevokeSessions.All` |

If you run under a standard daily-driver account rather than an admin account, the Graph consent prompt will fail on first connect.

---

## Installation

```powershell
git clone https://github.com/<org>/ad-offboarding-tool.git
cd ad-offboarding-tool
```

Run the GUI elevated the first time so PSGallery modules install to `AllUsers` — otherwise every other technician on that workstation hits the install prompt again.

---

## Usage

### GUI

```powershell
powershell.exe -sta -ExecutionPolicy Bypass -File .\Offboarding-GUI.ps1 `
    -DefaultOffboardOU "OU=Offboarding,OU=Users,DC=example,DC=com"
```

> `-sta` is not optional. The **Copy** button uses `Clipboard.SetText`, which requires a single-threaded apartment. PowerShell 7 defaults to MTA.

Workflow: type a partial name, sAMAccountName, UPN, or email → **Enter** → pick a row → review the confirmation pane (UPN, title, manager, group count, last logon, current DN) → select actions → **Begin Offboarding** → confirm the summary dialog.

The target OU dropdown auto-populates from OUs matching `*offboard*`, `*disabled*`, `*terminated*`, or `*departed*`. It is editable, so a DN can be pasted directly.

**Parameters**

| Parameter | Default | Notes |
|---|---|---|
| `-DefaultOffboardOU` | — | Pre-selects the target OU |
| `-HRContact` | `hr@example.com` | Interpolated into the default auto-reply text |
| `-LogPath` | `%ProgramData%\Offboarding` | Log directory |
| `-AutoInstallModules` | off | Installs missing modules without prompting |
| `-SkipModuleCheck` | off | Skips the startup check entirely |

### Headless

```powershell
# Dry run first
.\Invoke-UserOffboarding.ps1 -Identity jdoe `
    -OffboardOU "OU=Offboarding,OU=Users,DC=example,DC=com" -WhatIf

# Live
.\Invoke-UserOffboarding.ps1 -Identity jdoe@example.com `
    -OffboardOU "OU=Offboarding,OU=Users,DC=example,DC=com" `
    -OOOMessage "John Doe is no longer with the organization. Contact hr@example.com."
```

**Parameters**

| Parameter | Default | Notes |
|---|---|---|
| `-Identity` | *required* | sAMAccountName, UPN, or DN |
| `-OffboardOU` | *required* | Target OU distinguished name |
| `-OOOMessage` | auto-generated | Auto-reply body |
| `-ForwardContact` | `hr@example.com` | Used only in the generated message |
| `-PasswordLength` | `16` | 12–64 |
| `-LeaveEnabled` | off | Skip the disable step |
| `-SkipMailbox` | off | Skip auto-reply |
| `-SkipRevokeSessions` | off | Skip Graph revocation |
| `-LogPath` | `%ProgramData%\Offboarding` | Log directory |

Returns a `PSCustomObject` with per-step status, the resolved UPN, the generated password, and an error collection — suitable for piping into a ticketing update.

---

## Password handling

Generated with `RNGCryptoServiceProvider` using rejection sampling to avoid modulo bias, then Fisher-Yates shuffled. One character is guaranteed from each of four sets (upper, lower, digit, symbol); visually ambiguous characters (`0`, `O`, `1`, `l`, `I`) are excluded.

**The password is never written to the log file.** The log records only that a reset occurred. In the GUI it appears in a read-only field with a Copy button; headless, it is returned on the result object. Treat the console/clipboard as the only transport.

---

## Logging

One timestamped log per run at `%ProgramData%\Offboarding\Offboard_<sam>_<yyyyMMdd-HHmmss>.log`, containing the operator identity, the DC used, and the outcome of each step. The GUI mirrors this to a colour-coded activity pane in real time.

---

## Troubleshooting

| Symptom | Cause |
|---|---|
| "Module not installed" despite having installed it | PS 7 installs to `Documents\PowerShell\Modules`; Windows PowerShell 5.1 reads `Documents\WindowsPowerShell\Modules`. The paths do not cross over. |
| Copy button throws | Launched without `-sta`. |
| Session revoke checkbox greyed out after install | Only `Microsoft.Graph.Beta.Users.Actions` is present. The v1.0 module is required — the beta cmdlet name differs. |
| `Set-MailboxAutoReplyConfiguration` not found | It is session-imported. It does not resolve until `Connect-ExchangeOnline` has actually run. |
| Module install fails silently / app appears frozen | Usually a proxy blocking PSGallery. Check the activity log. |
| User still has cloud access after revocation | Access tokens live out their remaining lifetime (up to ~60 minutes) unless Continuous Access Evaluation is enforced. |
| Move fails with access denied | Delegation is missing on the **target** OU, not the source. |

---

## Not included

Deliberately out of scope for 1.0 — these vary too much between organisations to ship a default:

- Group membership removal (capture memberships to the log first; removal is not reversible without it)
- Hiding from the GAL
- Mailbox conversion to shared
- License removal
- Manager delegation of mailbox access
- Litigation hold / retention application

---

## Disclaimer

These scripts make destructive, largely irreversible directory changes. Test against a lab OU and validate delegation before production use. Use `-WhatIf` on the headless script.
