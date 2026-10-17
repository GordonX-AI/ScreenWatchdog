# 🛡️ ScreenWatchdog

&gt; Windows 11 远程录屏 / 远程监控风险检测工具包 —— 纯 PowerShell 5.1+，零依赖，单项目同时提供**深度审计**与**常驻监控**两种模式。

[English](README.md) | [中文](README.zh-CN.md)

[![Platform](https://img.shields.io/badge/Platform-Windows%2010%2F11-blue)](https://www.microsoft.com/windows)
[![PowerShell](https://img.shields.io/badge/PowerShell-5.1%2B-5391FE)](https://github.com/PowerShell/PowerShell)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)
[![CI](https://github.com/GordonX-AI/ScreenWatchdog/actions/workflows/ci.yml/badge.svg)](https://github.com/GordonX-AI/ScreenWatchdog/actions/workflows/ci.yml)

对"我的电脑是否被远程控制/偷录屏"这一高频焦虑场景，给出**工程化、可复现、可持续**的检测方案：
不依赖任何商业 EDR，不联网上传数据，全部检测逻辑开源可查。

---

## ✨ 核心特性

| 特性 | 说明 |
|---|---|
| 🔍 **深度审计模式** | 11 大类 60+ 检查项：远程访问通道、远控/录屏进程、持久化机制、网络外连、登录痕迹、账户、摄像头/麦克风隐私等 |
| 🔄 **基线 Diff** | 干净系统上 `-ExportBaseline` 建立持久化基线，之后 `-CompareBaseline` 只报告**新增**项（借鉴 PersistenceSniper） |
| 📈 **单机 Beacon 检测** | 对"进程→目标IP"连接间隔做变异系数统计，识别 C2/远控心跳的**时间规律性**（借鉴 RITA，单机实现） |
| 🎥 **摄像头/麦克风痕迹监控** | 基于 Windows 隐私注册表的时间戳变化检测，24 小时内有使用即告警 |
| 📦 **外置特征库** | `config/patterns.json` 热更新，加关键词即可扩充检测面，无需改代码 |
| 📤 **结构化输出** | 除文本报告外支持 `-OutputJson`，结果可对接 SIEM / 自建分析 |
| 🚫 **零依赖** | 原生 PowerShell + 系统事件日志，无需安装 Sysmon/Agent（保留 Sysmon 生态接口设计） |

## 🚀 快速开始

```powershell
# 克隆或下载后，【管理员】PowerShell 进入项目目录
cd ScreenWatchdog

# 1. 全量深度审计（输出到桌面 RemoteScreen_Audit_&lt;时间戳&gt;/）
powershell -ExecutionPolicy Bypass -File .\src\Invoke-ScreenAudit.ps1

# 2. 干净系统上建立持久化基线（一次即可）
powershell -ExecutionPolicy Bypass -File .\src\Invoke-ScreenAudit.ps1 -ExportBaseline

# 3. 日常对比基线（每周或安装新软件后）
powershell -ExecutionPolicy Bypass -File .\src\Invoke-ScreenAudit.ps1 -CompareBaseline

# 4. 常驻监控（按 Q 退出；事件增量 + 快照 Diff + Beacon 分析）
powershell -ExecutionPolicy Bypass -File .\src\Start-ScreenMonitor.ps1

# 5. 注册登录自启（可选）
powershell -ExecutionPolicy Bypass -File .\src\Start-ScreenMonitor.ps1 -RegisterAutostart
```

&gt; 📖 完整参数手册与常见误报处置：[docs/USAGE.md](docs/USAGE.md)

## 📋 审计模式检测覆盖

| # | 模块 | 内容 |
|---|---|---|
| 1 | 远程访问通道 | RDP 服务/端口/NLA、远程协助(Quick Assist)、WinRM/SSH/SMB 会话、高危端口监听 |
| 2 | 进程排查 | 70+ 远控/录屏/摄像头进程特征、Temp/AppData 无签名进程 |
| 3 | 服务与驱动 | 运行中服务签名校验、非微软签名驱动 |
| 4 | 持久化 | Run 键、启动文件夹、计划任务、WMI 事件订阅 |
| 4b | 扩展持久化 | IFEO 调试器劫持、SilentProcessExit、AppInit_DLLs、AppCertDlls、Winlogon、LSA、BootExecute、Print Monitors、TimeProviders、Office 加载项、COM 劫持 |
| 5 | 网络 | 外连归属进程、解释器外连（下载器特征）、系统代理、hosts 投毒、DNS 缓存 DGA |
| 6 | 软件痕迹 | 远控软件安装记录、连接日志（AnyDesk/TeamViewer ID 与 connection_trace） |
| 7 | 登录痕迹 | MSTSC 历史、RDP 事件(4624/4625/4778/4779)、服务安装 7045、PowerShell 4104 混淆、4625 密码喷洒统计 |
| 8 | 账户 | 隐藏账户（`$` 结尾）、Remote Desktop Users 组 |
| 9 | 摄像头/麦克风 | 权限授权清单、最近使用时间、GameDVR 后台录制 |
| 11 | 基线 Diff | 持久化条目基线对比 |

## 🧠 工作原理

**Beacon 检测（创新点）**：传统 IOC 匹配依赖已知域名/IP，对加密 C2 无效。ScreenWatchdog 借鉴
RITA 的思路——对每对 `(进程, 目标IP)` 的**连接出现间隔**计算变异系数（CV）：
周期 ≥30s、样本 ≥7、CV &lt; 0.25 即判定为规律性心跳，提示远控/C2 可能。

**误报抑制**：特征匹配采用多层降级（如 svchost 共享进程服务仅名称命中时自动抑制并提示复核）、
系统默认 WMI 消费者（SCM）白名单、良性心跳进程白名单（浏览器/IM/杀软）。

## 🔧 自定义特征库

编辑 `config/patterns.json`，向 `remoteProcPatterns` 等数组追加小写关键词即可，立即生效：

```json
{
  "remoteProcPatterns": ["teamviewer", "anydesk", "你的新特征"]
}
```

## 🧪 测试与 CI

```powershell
# 本地运行测试（需 Pester 5.x）
Invoke-Pester .\tests
```

CI 流水线（`.github/workflows/ci.yml`）在每次 push/PR 时执行：
PSScriptAnalyzer 静态检查 + Pester 单元测试 + patterns.json 合法性校验（Windows runner）。

## 🗺️ 路线图

- [ ] 输出对接 Sysmon / Windows Event Forwarding
- [ ] 内存扫描接口（pe-sieve 可选集成）
- [ ] 图形化摘要面板（WinForms/HTML 报告）
- [ ] 多语言报告

## ⚠️ 免责声明

本项目仅用于**合法授权的自有设备安全自查与防御**。所有检测均为只读操作，不修改系统。
使用者须遵守所在法域法律法规，禁止用于未经授权的设备。作者不对误用导致的任何后果负责。

## 📄 许可

[MIT License](LICENSE)
