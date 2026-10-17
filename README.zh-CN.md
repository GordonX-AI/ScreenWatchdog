# 🛡️ ScreenWatchdog

> An open-source PowerShell toolkit for detecting remote screen recording & surveillance risks on Windows — one-time **deep audit** plus **persistent monitoring**, with zero dependencies and fully read-only operations.

[English](README.md) | [中文](README.zh-CN.md)

[![Platform](https://img.shields.io/badge/Platform-Windows%2010%2F11-blue)](https://www.microsoft.com/windows)
[![PowerShell](https://img.shields.io/badge/PowerShell-5.1%2B-5391FE)](https://github.com/PowerShell/PowerShell)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)
[![CI](https://github.com/GordonX-AI/ScreenWatchdog/actions/workflows/ci.yml/badge.svg)](https://github.com/GordonX-AI/ScreenWatchdog/actions/workflows/ci.yml)

---

## ✨ Key Features

| Feature | Description |
|---|---|
| 🔍 **Deep Audit Mode** | 60+ checks across 11 categories: remote access channels, RAT/screen-recorder processes, persistence mechanisms, network connections, logon traces, accounts, camera/mic privacy |
| 🔄 **Persistence Baseline Diff** | `-ExportBaseline` on a clean machine, then `-CompareBaseline` to report only **new** persistence entries (inspired by PersistenceSniper) |
| 📈 **Standalone Beacon Detection** | Statistical analysis of connection-interval regularity per (process → destination IP), flagging C2/RAT heartbeat patterns (RITA-style, implemented locally) |
| 🎥 **Camera & Mic Trace Monitoring** | Registry-timestamp based detection of camera/microphone usage within the last 24h |
| 📦 **External Pattern Library** | Hot-swappable `config/patterns.json` — extend detection coverage by adding keywords, no code changes |
| 📤 **Structured Output** | `-OutputJson` emits machine-readable results (SIEM-ready); every run writes a `manifest.json` for reproducibility |
| 🚫 **Zero Dependencies** | Pure PowerShell 5.1+ against native event logs — no Sysmon, no agent, nothing installed |

## 🚀 Quick Start

```powershell
# After cloning/downloading, run from an ELEVATED PowerShell in the project root
cd ScreenWatchdog

# 1. Full deep audit (outputs to ~\Desktop\RemoteScreen_Audit_<timestamp>\)
powershell -ExecutionPolicy Bypass -File .\src\Invoke-ScreenAudit.ps1

# 2. Create a persistence baseline (once, on a clean system)
powershell -ExecutionPolicy Bypass -File .\src\Invoke-ScreenAudit.ps1 -ExportBaseline

# 3. Day-to-day: diff against baseline (weekly, or after installing new software)
powershell -ExecutionPolicy Bypass -File .\src\Invoke-ScreenAudit.ps1 -CompareBaseline

# 4. Persistent monitoring (press Q to quit; event-driven + snapshot diff + beacon analysis)
powershell -ExecutionPolicy Bypass -File .\src\Start-ScreenMonitor.ps1

# 5. Optional: register autostart at user logon
powershell -ExecutionPolicy Bypass -File .\src\Start-ScreenMonitor.ps1 -RegisterAutostart
```

> 📖 Full parameter reference & false-positive playbook: [docs/USAGE.md](docs/USAGE.md)

## 📋 Audit Coverage

| # | Module | Highlights |
|---|---|---|
| 1 | Remote Access | RDP service/port/NLA, Remote Assistance (Quick Assist), WinRM/SSH/SMB sessions, high-risk listener ports |
| 2 | Processes | 70+ RAT/screen-recorder/webcam process signatures, unsigned binaries in Temp/AppData |
| 3 | Services & Drivers | Authenticode verification of running services, non-Microsoft signed drivers |
| 4 | Persistence | Run keys, Startup folders, scheduled tasks, WMI event subscriptions |
| 4b | Advanced Persistence | IFEO debugger hijack, SilentProcessExit, AppInit_DLLs, AppCertDlls, Winlogon, LSA, BootExecute, Print Monitors, TimeProviders, Office add-ins, COM hijacking |
| 5 | Network | Per-connection process attribution, script-interpreter outbound (downloader pattern), proxy/PAC, hosts poisoning, DNS-cache DGA check |
| 6 | Artifacts | RAT install records, connection logs (AnyDesk/TeamViewer IDs & connection_trace) |
| 7 | Logon Traces | MSTSC history, RDP events (4624/4625/4778/4779), service installs (7045), PowerShell 4104 obfuscation, 4625 password-spray statistics |
| 8 | Accounts | Hidden accounts (`$` suffix), Remote Desktop Users group |
| 9 | Privacy | Camera/mic permission grants, last-used timestamps, GameDVR background recording |
| 11 | Baseline Diff | Persistence baseline comparison |

## 🧠 How It Works

**Beacon detection.** IOC matching fails against encrypted C2. ScreenWatchdog instead measures the
**temporal regularity** of `(process, destination)` connection intervals — computing the coefficient
of variation (CV): period ≥ 30s, ≥ 7 samples, CV < 0.25 ⇒ likely machine-generated heartbeat.

**False-positive suppression.** Multi-layer degradation: name-only matches on shared svchost services
are auto-suppressed, system-default WMI consumers (SCM) are whitelisted, and known benign heartbeat
processes (browsers, IM, AV) are excluded.

## 🔧 Customizing the Pattern Library

Edit `config/patterns.json` — just append lowercase keywords:

```json
{
  "remoteProcPatterns": ["teamviewer", "anydesk", "your-new-signature"]
}
```

## 🧪 Testing & CI

```powershell
# Local tests (Pester 5.x)
Invoke-Pester .\tests
```

CI (`.github/workflows/ci.yml`) runs on every push/PR:
PSScriptAnalyzer lint → Pester unit tests → `patterns.json` schema & regex validation (Windows runner).

## 🗺️ Roadmap

- [ ] Output integration with Sysmon / Windows Event Forwarding
- [ ] Optional pe-sieve memory scanning hook
- [ ] HTML summary report dashboard
- [ ] Multi-language report generation

## ⚠️ Disclaimer

This project is intended solely for **security self-audit of devices you own or are explicitly
authorized to assess**. All operations are strictly read-only and modify nothing. Users must comply
with applicable laws; the authors accept no liability for misuse.

## 📄 License

[MIT License](LICENSE)
