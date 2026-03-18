<div align="center">

```
     ____                  _    ____            
    |_  /__ ___ _____ ___ | |_ / __/__ ___     
     / // _` \ V / -_)  _||  _\__ \/ -_) _|    
    /___\__,_|\_/\___\__| |_| |___/\___\__|    
```

**Windows Security Hardening Baseline**

[![PowerShell](https://img.shields.io/badge/PowerShell-5.1%2B-0078d4?style=flat-square&logo=powershell)](https://docs.microsoft.com/powershell)
[![Windows](https://img.shields.io/badge/Windows-10%2F11%20%7C%20Server%202016--2022-0078d4?style=flat-square&logo=windows)](https://microsoft.com/windows)
[![CIS](https://img.shields.io/badge/Standard-CIS%20%7C%20DISA%20STIG%20%7C%20MS%20Baseline-00b4d8?style=flat-square)](https://cisecurity.org)
[![License](https://img.shields.io/badge/License-MIT-30d158?style=flat-square)](LICENSE)
[![Version](https://img.shields.io/badge/Version-1.0-ff6b00?style=flat-square)](#)

*One script. 60+ checks. Three modes. Zero bloat.*

</div>

---

## `>_ the problem`

Most Windows environments ship with settings that are actively dangerous:
LLMNR broadcasting credentials to anyone who asks, WDigest storing plaintext
passwords in memory, SMBv1 waiting for EternalBlue, audit logs sized at 20 MB
that fill in hours. These are not edge cases — they are defaults.

`ZavetSecHardeningBaseline` fixes this. It audits your current state, applies
a hardened baseline aligned to **CIS Benchmark**, **DISA STIG**, and
**Microsoft Security Baseline**, and generates a signed HTML report you can
hand to a customer or attach to a ticket. If something breaks, rollback from
the JSON backup created before every change.

---

## `>_ files`

```
ZavetSecHardeningBaseline.ps1   — main script (PowerShell 5.1+)
Run-Hardening.bat               — interactive launcher with menu
```

---

## `>_ design philosophy`

**Idempotent.** Run Apply twice — result is identical. Safe for scheduled tasks
and re-deployment after GPO conflicts.

**Non-destructive by default.** Every Apply creates a timestamped JSON backup
of prior state. Rollback is a single command.

**Audit-first.** Audit mode makes zero changes. Run it first, read the report,
then decide what to apply.

**PsExec-compatible.** `-NonInteractive` flag suppresses all prompts. Deploy
across a subnet without touching keyboards.

**Locale-independent.** Audit policy is configured via auditpol GUIDs, not
subcategory names. Works identically on Russian, English, and any other
Windows localization.

---

## `>_ what attacks does this stop`

| Threat | Technique | Controls applied |
|---|---|---|
| Responder / MITM | T1557.001 | LLMNR off, NBT-NS off, mDNS off, WPAD off, SMB signing required |
| Mimikatz / credential dump | T1003.001 | WDigest off, LSA PPL on, Credential Guard enabled |
| Pass-the-Hash | T1550.002 | NTLMv2 only, LM hash storage disabled, 128-bit session security |
| EternalBlue / WannaCry | T1210 | SMBv1 disabled on server and client driver |
| Lateral movement | T1021 | Remote Registry disabled, anonymous enumeration restricted |
| Pre-auth RDP exploits | T1021.001 | NLA enforced, encryption level high |
| USB payload delivery | T1091 | AutoRun / AutoPlay fully disabled |
| PowerShell living-off-land | T1059.001 | Script Block + Module logging, PSv2 disabled, Exec Policy set |
| Log gap / blind spot | — | Security log 1 GB, 27 audit subcategories configured |

---

## `>_ coverage`

**Network** `NET-001 — NET-010`
Disables LLMNR, mDNS, WPAD, NBT-NS, LMHOSTS. Disables SMBv1 on both server
and client driver. Enforces SMB signing. Restricts anonymous SAM/share
enumeration. Disables Remote Registry.

**Credential Protection** `CRED-001 — CRED-006`
Disables WDigest plaintext caching. Enables LSA Protected Process Light.
Enables Credential Guard (VBS). Forces NTLMv2. Disables LM hash storage.
Requires 128-bit NTLM session security.

**PowerShell Hardening** `PS-001 — PS-005`
Enables Script Block Logging (4104) and Module Logging (4103). Enables
transcription to `C:\ProgramData\PSTranscripts`. Disables the PSv2 engine
(AMSI bypass vector). Sets Execution Policy at machine scope.

**Audit Policy** `AUD-001 — AUD-027`
27 subcategories via `auditpol` with GUID references — locale-independent.
Covers Logon/Logoff, Kerberos, Process Creation, Account Management, Object
Access, Privilege Use, Policy Change, DPAPI, Scheduled Tasks, Removable
Storage, and Firewall events.

**System Hardening** `SYS-001 — SYS-010`
Enables UAC with full secure desktop enforcement. Disables AutoRun/AutoPlay.
Enables Windows Firewall on all profiles. Requires RDP NLA. Enables DEP
(AlwaysOn). Sets Security log to 1 GB with overwrite retention (no archive
files). Enables DoH policy. Sets RDP encryption to High. Optionally disables
Print Spooler (PrintNightmare mitigation).

---

## `>_ safe to run — read this first`

> ⚠️ **Test in a non-production environment before deploying at scale.**

**What may break:**

- **SMBv1 disable** — legacy devices (old printers, NAS, XP/2003) that only
  speak SMBv1 will lose network access. Audit first with `Get-SmbConnection`
  to identify dependent devices.
- **SMB signing required** — clients that do not support signing will be
  rejected. Negligible in modern environments, significant in mixed/legacy ones.
- **Credential Guard** — requires UEFI + Secure Boot + VBS-capable hardware.
  Silently skipped on incompatible hardware, but verify before deploying on
  older servers.
- **NTLMv2 only** — legacy systems that only support LM/NTLMv1 will fail
  authentication. Rare, but exists in some OT/industrial environments.
- **Print Spooler disable** (`-EnablePrintSpoolerDisable`) — printing stops.
  Apply only to servers and workstations that do not print.
- **PSv2 disable** — requires a reboot. Some older automation scripts
  that explicitly call `powershell -version 2` will break.

**What never breaks:**
Audit mode is fully read-only. Rollback restores prior registry state
from the JSON backup. Services are restored to their original startup type.

**Reboot required for:** Credential Guard, DEP (AlwaysOn), PSv2 disable,
SMBv1 client driver.

---

## `>_ quickstart`

### Option A — BAT launcher (recommended for manual use)

Right-click `Run-Hardening.bat` → **Run as administrator.**

```
  ============================================================
   ZavetSec - Windows Security Hardening Baseline
  ============================================================

   [1]  AUDIT    - Check current state (no changes)
   [2]  APPLY    - Apply all hardening settings
   [3]  ROLLBACK - Revert changes (requires backup file)
   [4]  EXIT
```

Creates `Reports\` subfolder automatically. Timestamped HTML reports and JSON
backups saved there. ROLLBACK lists available backups by number.

### Option B — PowerShell directly

```powershell
# Audit only — zero changes
.\ZavetSecHardeningBaseline.ps1 -Mode Audit

# Apply (interactive confirmation)
.\ZavetSecHardeningBaseline.ps1 -Mode Apply

# Apply — no prompts (PsExec / automation)
.\ZavetSecHardeningBaseline.ps1 -Mode Apply -NonInteractive

# Rollback
.\ZavetSecHardeningBaseline.ps1 -Mode Rollback `
    -BackupPath .\Reports\HardeningBackup_20260318_120000.json

# Skip sections
.\ZavetSecHardeningBaseline.ps1 -Mode Apply -SkipAuditPolicy
.\ZavetSecHardeningBaseline.ps1 -Mode Apply -SkipNetworkHardening
.\ZavetSecHardeningBaseline.ps1 -Mode Apply -SkipCredentialProtection
.\ZavetSecHardeningBaseline.ps1 -Mode Apply -SkipPowerShell

# PrintNightmare mitigation (opt-in)
.\ZavetSecHardeningBaseline.ps1 -Mode Apply -EnablePrintSpoolerDisable
```

### Option C — Mass deployment via PsExec

```powershell
psexec \\TARGET -s -c .\ZavetSecHardeningBaseline.ps1 -Mode Apply -NonInteractive
```

---

## `>_ recommended deployment timeline`

```
Day 0    Run Audit on a representative sample of machines.
         Review the HTML report. Identify legacy dependencies
         (SMBv1 users, NTLMv1 devices, old automation scripts).

Day 1-7  Fix identified dependencies. Test Apply in a lab VM.
         Verify rollback works from the generated backup.

Day 7    Apply to a pilot group (5-10 machines).
         Monitor for 48h. Check application and helpdesk tickets.

Day 14   Apply to the broader environment in batches.
         Reboot machines requiring it (Credential Guard, DEP, PSv2).

Day 30   Re-run Audit across all machines.
         Compare compliance % before and after.
         Attach HTML report to change management record.
```

---

## `>_ vs alternatives`

| Tool | Approach | Rollback | Report | Offline | PS 5.1 |
|---|---|---|---|---|---|
| **ZavetSecHardeningBaseline** | Script, 3 modes | ✅ JSON backup | ✅ HTML | ✅ | ✅ |
| CIS CAT Pro | GUI scanner | ❌ | ✅ | ❌ requires Java | ❌ |
| LGPO.exe | GPO import | manual | ❌ | ✅ | N/A |
| GPO baseline | Policy-based | via GPO restore | ❌ | ❌ needs DC | N/A |
| Microsoft SCT | GPO + scripts | partial | ❌ | ✅ | partial |

---

## `>_ tested environments`

| OS | Arch | Status |
|---|---|---|
| Windows 10 21H2+ | x64 | ✅ |
| Windows 11 22H2+ | x64 | ✅ |
| Windows Server 2016 | x64 | ✅ |
| Windows Server 2019 | x64 | ✅ |
| Windows Server 2022 | x64 | ✅ |
| Windows Server Core | x64 | ✅ (no browser for report) |
| Domain-joined | — | ✅ |
| Workgroup | — | ✅ |

---

## `>_ parameters`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `-Mode` | `Audit\|Apply\|Rollback` | `Audit` | Operation mode |
| `-BackupPath` | string | script dir | JSON backup path |
| `-OutputPath` | string | script dir | HTML report path |
| `-SkipAuditPolicy` | switch | — | Skip audit policy section |
| `-SkipNetworkHardening` | switch | — | Skip network section |
| `-SkipPowerShell` | switch | — | Skip PowerShell section |
| `-SkipCredentialProtection` | switch | — | Skip credentials section |
| `-EnablePrintSpoolerDisable` | switch | — | Disable Print Spooler (opt-in) |
| `-NonInteractive` | switch | — | Suppress all prompts |

---

## `>_ output`

Every run produces a dark-themed, filterable **HTML report** containing:
compliance score gauge, per-category breakdown, full check table with severity /
MITRE reference / apply status / remediation command, and the rollback command
pre-filled with the backup path.

Reports are saved to `.\Reports\` when using the BAT launcher, or to the script
directory when running PowerShell directly (override with `-OutputPath`).

---

## `>_ disclaimer`

> This script modifies security-relevant system settings. Always run Audit
> mode first. Always test Apply in a non-production environment before
> deploying at scale. The author assumes no responsibility for system
> instability, application breakage, or data loss resulting from use of
> this tool.

---

<div align="center">

**ZavetSec** — security tooling for those who read logs at 2am

[GitHub](https://github.com/zavetsec) · MIT License

</div>
