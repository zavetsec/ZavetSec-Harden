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

## `>_ what is this`

A single PowerShell script that **audits, applies, and rolls back** Windows security hardening settings across workstations and servers. Built for SOC/DFIR engineers who need consistent, repeatable baseline hardening at scale — without GPO overhead, without third-party agents, without noise.

Covers **CIS Benchmark L1/L2**, **DISA STIG**, and **Microsoft Security Baseline** recommendations, mapped to **MITRE ATT&CK** where applicable.

---

## `>_ files`

```
ZavetSecHardeningBaseline.ps1   — main script
Run-Hardening.bat               — interactive launcher with menu (recommended)
```

---

## `>_ quickstart`

### Option A — BAT launcher (recommended for manual use)

Right-click `Run-Hardening.bat` → **Run as administrator**.

```
  ============================================================
   ZavetSec - Windows Security Hardening Baseline
  ============================================================

   [1]  AUDIT    - Check current state (no changes)
   [2]  APPLY    - Apply all hardening settings
   [3]  ROLLBACK - Revert changes (requires backup file)
   [4]  EXIT
```

The launcher handles everything automatically:
- Checks for Administrator rights before proceeding
- Creates a `Reports\` subfolder next to the scripts
- Saves timestamped HTML reports and JSON backups to `Reports\`
- Offers to open the HTML report in browser after each run
- ROLLBACK lists available backup files by number — no manual path entry needed

### Option B — PowerShell directly

```powershell
# Audit only — no changes, generates HTML report
.\ZavetSecHardeningBaseline.ps1 -Mode Audit

# Apply all hardening (interactive confirmation prompt)
.\ZavetSecHardeningBaseline.ps1 -Mode Apply

# Apply without prompts — safe for PsExec / scheduled tasks / automation
.\ZavetSecHardeningBaseline.ps1 -Mode Apply -NonInteractive

# Apply with explicit output paths
.\ZavetSecHardeningBaseline.ps1 -Mode Apply `
    -OutputPath C:\Reports\hardening.html `
    -BackupPath C:\Reports\backup.json

# Rollback from a specific backup file
.\ZavetSecHardeningBaseline.ps1 -Mode Rollback `
    -BackupPath .\Reports\HardeningBackup_20260318_120000.json

# Partial apply — skip categories as needed
.\ZavetSecHardeningBaseline.ps1 -Mode Apply -SkipAuditPolicy
.\ZavetSecHardeningBaseline.ps1 -Mode Apply -SkipNetworkHardening
.\ZavetSecHardeningBaseline.ps1 -Mode Apply -SkipCredentialProtection
.\ZavetSecHardeningBaseline.ps1 -Mode Apply -SkipPowerShell

# Also disable Print Spooler — PrintNightmare mitigation (opt-in)
.\ZavetSecHardeningBaseline.ps1 -Mode Apply -EnablePrintSpoolerDisable
```

### Option C — Mass deployment via PsExec

```powershell
psexec \\TARGET -s -c .\ZavetSecHardeningBaseline.ps1 -Mode Apply -NonInteractive
```

---

## `>_ what it covers`

| Category | Checks | Focus |
|---|---|---|
| 🌐 **Network** | NET-001 — NET-010 | LLMNR, mDNS, WPAD, SMBv1, NBT-NS, SMB Signing, Anon Enum, Remote Registry |
| 🔑 **Credentials** | CRED-001 — CRED-006 | WDigest, LSA PPL, Credential Guard, NTLMv2, LM Hash, 128-bit Session |
| 🐚 **PowerShell** | PS-001 — PS-005 | Script Block Logging, Module Logging, Transcription, PSv2 Disable, Exec Policy |
| 📋 **Audit Policy** | AUD-001 — AUD-027 | 27 subcategories via `auditpol` — Logon, Kerberos, Process, DPAPI, and more |
| 🖥️ **System** | SYS-001 — SYS-010 | UAC, AutoRun, Firewall, RDP NLA, DEP, Event Log sizing, DoH, RDP Encryption |

**Total: 60+ checks across 5 categories.**

---

## `>_ three modes`

```
┌───────────────┬────────────────────────────────────────────────────────────┐
│  -Mode Audit   │  Read-only. Check current state. Generate HTML report.     │
│  -Mode Apply   │  Apply all hardening. Save JSON backup before any changes. │
│  -Mode Rollback│  Restore previous state from a JSON backup file.           │
└───────────────┴────────────────────────────────────────────────────────────┘
```

---

## `>_ output`

Every run generates a **dark-themed HTML report** with:

- ZavetSec branded header + compliance score gauge
- Per-category compliance breakdown table
- Full filterable check list — ID, severity, MITRE reference, result, apply status, remediation command
- Backup path + rollback command embedded directly in the report

**Where reports are saved:**

| Launch method | Reports location |
|---|---|
| `Run-Hardening.bat` | `.\Reports\` subfolder (created automatically) |
| PowerShell direct | Script directory (override with `-OutputPath`) |

---

## `>_ parameters`

| Parameter | Type | Default | Description |
|---|---|---|---|
| `-Mode` | `Audit\|Apply\|Rollback` | `Audit` | Operation mode |
| `-BackupPath` | string | script dir | JSON backup file path |
| `-OutputPath` | string | script dir | HTML report output path |
| `-SkipAuditPolicy` | switch | — | Skip audit policy section |
| `-SkipNetworkHardening` | switch | — | Skip network hardening section |
| `-SkipPowerShell` | switch | — | Skip PowerShell hardening section |
| `-SkipCredentialProtection` | switch | — | Skip credential protection section |
| `-EnablePrintSpoolerDisable` | switch | — | Also disable Print Spooler service |
| `-NonInteractive` | switch | — | Suppress all prompts (PsExec-safe) |

---

## `>_ attack surface reduced`

```
Responder / MITM     ──  LLMNR off, NBT-NS off, mDNS off, WPAD off, SMB signing enforced
Mimikatz / LSASS     ──  WDigest off, LSA PPL on, Credential Guard on
Pass-the-Hash        ──  NTLMv2 only, LM hash storage off, 128-bit session security
Lateral Movement     ──  Remote Registry off, anonymous enum restricted
Persistence via PS   ──  Script Block + Module logging on, PSv2 disabled, Exec Policy set
Pre-auth RDP         ──  NLA enforced, encryption level high
USB attacks          ──  AutoRun / AutoPlay fully disabled
EternalBlue/WannaCry ──  SMBv1 disabled (server + client driver)
Logging gaps         ──  Security log 1 GB, 27 audit subcategories configured
```

---

## `>_ requirements`

```
OS        :  Windows 10 / 11 / Server 2016 / 2019 / 2022
PowerShell:  5.1+
Rights    :  Local Administrator (Apply and Rollback modes)
Reboot    :  Required for: Credential Guard, DEP, PSv2 disable, SMBv1 client driver
```

---

## `>_ disclaimer`

> Test in a lab environment before deploying to production. Some settings require a reboot to take effect. Credential Guard requires UEFI + Secure Boot + VBS-capable hardware. The author is not responsible for misuse or system instability caused by applying this script without prior testing.

---

<div align="center">

**ZavetSec** — security tooling for those who read logs at 2am

[GitHub](https://github.com/zavetsec) · MIT License

</div>
