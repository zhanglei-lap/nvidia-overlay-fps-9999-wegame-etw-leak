# NVIDIA App 统计浮窗 FPS 恒为 `9999` —— 根因分析、证据与解决方案

> **English abstract** — NVIDIA App's statistics overlay (FrameView/PresentMon based) reported
> `9999` for "FPS avg" and "1% low" in every game, on a machine where the same overlay used to
> work. After ruling out driver version, app version, DLSS/Frame-Generation, anti-cheat, remote
> desktop and overlay conflicts, the cause was found to be a **leaked real-time ETW trace session
> named `ETWSessionRecorder`**, created by **WeGame's FPS-monitor component
> (`wegame_environment\wegame_env.exe`, source file `fps_monitor.cpp`, class `CFPSMonitor`,
> accompanied by `PerfOverlay.exe`)**. That session subscribes `Microsoft-Windows-DxgKrnl` and
> `Microsoft-Windows-Dwm-Core` with *all* keywords and is never consumed (its own `[StopETW]`
> routine fails), so its buffers stay permanently full. Because NVIDIA's capture needs the same
> providers, it loses hundreds of thousands of events (`ETW events were lost` up to 1,830,859),
> the "display change" companion event disappears, every frame is reported as
> `msBetweenDisplayChange = 0 → N/A`, and the overlay shows `9999`.
> Fix: disable WeGame's FPS monitor / stop the leaked session with
> `logman stop "ETWSessionRecorder" -ets`.

---

## TL;DR

| 项目 | 内容 |
|---|---|
| **现象** | NVIDIA App 信息浮窗里 `FPS avg` 与 `1% Low` 恒显示 `9999`（有时全部指标 N/A），换游戏、换驱动、装 FrameView 2.0 均无效，但**手动停掉一个 ETW 会话后立即恢复** |
| **根因** | **WeGame 的 FPS 监控模块泄漏了一个实时模式 ETW 会话 `ETWSessionRecorder`**：无消费者、缓冲区永久写满，且订阅了 `DxgKrnl` + `Dwm-Core` 的全部关键字 —— 与 NVIDIA 采集帧率所需的事件源完全重叠，导致其大量丢事件 |
| **元凶文件** | `E:\wegame\wegame_environment\wegame_env.exe`（产品名 *WeGame Environmental Center* v1.0.0.1）、`E:\wegame\wegame_environment\PerfOverlay.exe` |
| **起始时间** | `PerfOverlay.exe` 于 **2026-09-08 12:03:50** 落盘，内核日志首条 `ETWSessionRecorder` 报错为 **2026-09-08 13:15:46** |
| **立即恢复** | 管理员：`logman stop "ETWSessionRecorder" -ets` |
| **长期解决** | 关闭 WeGame 的帧率显示/性能监控；或每次用完 WeGame 后清理该会话（可做定时任务） |
| **注意** | `logman delete "ETWSessionRecorder"` **必然报"找不到数据收集器集"** —— 它是运行时会话，没有保存的定义可删 |

---

## 1. 现象

- NVIDIA App（统计浮窗 / 信息浮窗）里 **`FPS avg` 与 `1% Low` 恒为 `9999`**；偶尔**所有指标**（含 GPU/CPU）一起变 `N/A`（后者是另一个独立问题，见 §7.3）。
- 与游戏无关：绝区零（DX12）、彩虹六号、极限竞速：地平线 6、CS2、泰拉瑞亚都一样。
- 与 DLSS/帧生成无关：关闭 DLSS 与帧生成后仍然 `9999`。
- 曾经正常，**大约从 2026 年 9 月上旬开始不正常**。
- **关键可复现动作**：`logman stop "ETWSessionRecorder" -ets` → 帧率立即恢复正常。

## 2. 环境

| 项目 | 值 |
|---|---|
| 操作系统 | Windows 11 25H2，内部版本 `26200.8457` |
| 显卡 / 驱动 | NVIDIA GeForce RTX 5060，驱动 `617.14`（2026-09-29 更新，此前 `596.49`） |
| NVIDIA App | `11.0.9.251`（2026-09-04 安装，自更新报告"已是最新"） |
| FrameView | SDK `1.9.12728` + 独立版 `FrameView 2.0.0`（2026-09-30 安装） |
| 显示器 | `27G5K-PRO+`，DisplayPort 直连显卡，200 Hz |
| 主板 / 固件 | MAXSUN MS-Terminator B650M，固件 `B4.2D` |
| **元凶** | WeGame「环境检测中心」：`wegame_env.exe` + `PerfOverlay.exe` |

## 3. 根因（一句话）

> WeGame 为显示 FPS 创建了一个**实时模式（`EVENT_TRACE_REAL_TIME_MODE`）且没有消费者**的 ETW 会话
> `ETWSessionRecorder`，订阅 `Microsoft-Windows-DxgKrnl` 与 `Microsoft-Windows-Dwm-Core`
> 的**全部关键字**；其停止逻辑失败后会话永久残留、缓冲区永久写满，拖垮了同一条事件通路，
> 于是 NVIDIA 的帧率采集拿不到"显示时刻"，浮窗只能显示无效占位值 `9999`。

## 4. 证据链

### 4.1 采集侧：PresentMon 对"几乎每一帧"都报 N/A

`C:\ProgramData\NVIDIA Corporation\FrameViewSDK\PresentMon_Consumer.log`

```
#2(E)[2026-09-30 17:43:59,681]{00005D68}UpdateCsv Animation Error: msBetweenDisplayChange is zero (msBetweenSimulationStart=8.253) - reporting N/A
#3(E)[2026-09-30 17:43:59,681]{00005D68}UpdateCsv Delta PCL: Both current (0.000) and reference PCL values are invalid - reporting N/A
```

统计（同一条链路，同一台机器）：

| 场景 | 日志规模 | `msBetweenDisplayChange is zero` | 失败速率 |
|---|---|---|---|
| 故障期（R6，9 分 25 秒） | 72,718 行 / 10.3 MB | **36,356 次** | **25.7 次/秒**（几乎每帧） |
| 故障期（绝区零 DX12，35 秒） | 10,788 行 | 5,392 次 | 154 次/秒（约全部帧） |
| 正常期（清理泄漏会话后） | 1,162 行 / 61 秒 | 257 次 | **4.2 次/秒**（仅零星噪声） |

含义：提交事件（`msBetweenSimulationStart` ≈ 6–8 ms）**正常**，但"这一帧何时真正显示到屏幕上"（`msBetweenDisplayChange`）**整列为 0**，而 FPS 平均 / 1% Low 完全由后者计算 → 指标无有效输入。

### 4.2 同期 NVIDIA 采集大量丢事件

| 时间 | PresentMon 报告 |
|---|---|
| 2026-09-29 15:16 | `PresentMonMain warning: 248878 ETW events were lost.` |
| 2026-09-30 17:30 | `PresentMonMain warning: 89352 ETW events were lost.` |
| 2026-09-30 17:51–17:54 | `PresentMonMain warning: 1830859 ETW events were lost.` |

### 4.3 系统侧：一个"实时模式、无消费者"的 ETW 会话

`Microsoft-Windows-Kernel-EventTracing/Admin` 中，自 **2026-09-08 13:15:46** 起，每约 2 分钟一条：

```
实时会话"ETWSessionRecorder"的备份文件已达到其最大大小。因此，在有可用空间之前，
无法将新事件记录到此会话。以实时模式启动跟踪会话，但没有任何实时使用者时，通常会导致此错误。
```

累计 **519 条**（9/8 → 10/3，几乎每天都有）。

会话的活体信息（`logman query "ETWSessionRecorder" -ets`，管理员）：

```
名称: ETWSessionRecorder          状态: 正在运行          文件模式: 实时
根路径: %systemdrive%\PerfLogs\Admin
缓存大小: 64        缓冲区刷新计时器: 1        缓冲区已写入: 8022

提供程序: Microsoft-Windows-DxgKrnl   KeywordsAny: 0xffffffffffffffff（全部关键字）
提供程序: Microsoft-Windows-Dwm-Core  KeywordsAny: 0xffffffffffffffff（全部关键字）
```

同时，**没有保存的"数据收集器集"定义**：

```
> logman query "ETWSessionRecorder"
错误: 找不到数据收集器集。
```

→ 因此 `logman delete` 必然失败；它只能被 `stop`（临时）或被创建者正确释放。

### 4.4 会话"重复创建失败"事件给出了创建者特征

`Kernel-EventTracing` 事件 ID 2（4 条）：

```
会话"ETWSessionRecorder"未能启动，存在以下错误: 0xC0000035
SessionName=ETWSessionRecorder   LoggingMode=256（REAL_TIME）
Security UserID = S-1-5-21-…-1001（登录用户，非 SYSTEM）
时间：2026-09-10 18:55:49 / 2026-09-15 22:31:58 / 2026-09-15 22:52:24 / 2026-09-30 16:24:06
```

`0xC0000035 = STATUS_OBJECT_NAME_COLLISION`：**程序反复尝试创建同名会话，却被自己那个残留实例占用了名字** —— 即"创建后不释放"的直接证据。

### 4.5 元凶定位：WeGame 的 FPS 监控模块

对 `E:\wegame\wegame_environment\wegame_env.exe`（UTF-16 字符串）取证：

```
ETWSessionRecorder
Microsoft-Windows-DxgKrnl
e:\wegame_env\wegame_env\wegame_env\fps_monitor.cpp     ← 源码路径：帧率监控
[CFPSMonitor] user_trace start
[StopETW] name=%s failed: status=%s                     ← 它想停会话，而且会失败
ETW trace error: / QueryAllTracesW failed: status=%d (need admin)
No more trace sessions available. / The trace session has already been registered
IsD3DProcess pid:{} matched gfx module:{} / FPS window dwProcessId:{}
FPS window skip, not a D3D/DXGI/Vulkan process
d3d9.dll / d3d10.dll / vulkan-1.dll / D3DProxyWindow
FPS_Monitor_Game_Process_Exit / wegame_env_fps_msg_wnd
```

文件属性：产品名 **WeGame Environmental Center**，版本 `1.0.0.1`，修改时间 **2026-09-29 12:25:05**；
同目录 `PerfOverlay.exe`（性能覆盖层）修改时间 **2026-09-08 12:03:50**。

### 4.6 时间线完全对齐

```
2026-09-08 12:03:50  WeGame 推送 PerfOverlay.exe（性能覆盖层）落盘
2026-09-08 13:15:46  内核日志出现第一条 ETWSessionRecorder"缓冲区满/无消费者"报错   ← 故障起点
2026-09-10 18:55:49  首次"撞名失败"（创建者再次运行）
2026-09-15 22:31 / 22:52、2026-09-30 16:24  更多撞名失败
2026-09-29 12:25:05  wegame_env.exe 更新
2026-10-06 13:36:57  WmiPrvSE 被拉起（有程序在做 WMI 查询）
2026-10-06 13:36:59  ETWSessionRecorder 被创建  ← 用户此刻正在打开 WeGame 玩手游模拟器
2026-10-06 起        手动 logman stop 后，帧率立即恢复
```

## 5. 机制解释

1. NVIDIA App 统计浮窗的 `FPS avg` / `1% Low` 来自 FrameView/PresentMon 的 ETW 采集：需要
   `Microsoft-Windows-DxgKrnl` 的**提交/显示翻转**事件（"显示时刻"）以及 `Dwm-Core` 的合成信息。
2. WeGame 的 FPS 监控（`fps_monitor.cpp` / `CFPSMonitor`）为统计游戏帧率，创建了一个**实时模式** ETW
   会话 `ETWSessionRecorder`，并把上述两个提供程序的**全部关键字**打开（事件量最大）。
3. 它的 `[StopETW]` 停止逻辑失败（其字符串本身就写着 `failed`），会话**残留**：
   **实时模式 + 没有消费者 + 缓冲区写满**（Windows 每 2 分钟报错印证）。
4. 两个会话同时订阅同一批提供程序，残留会话长期处于"写不进去"的状态，导致这条路上海量事件被丢弃
   （NVIDIA 侧实测 `ETW events were lost` 达 89,352 / 248,878 / 1,830,859）。
5. "显示时刻"伴生事件丢失后，PresentMon 对几乎每一帧都判定
   `msBetweenDisplayChange = 0` → `reporting N/A`，两个帧率指标都没有有效输入。
6. NVIDIA App 对"无效值"的渲染就是占位数字 **`9999`**（这也是为什么它是个固定值、并且
   `FPS avg` 与 `1% Low` 同时出现）。
7. `logman stop` 移除残留会话 → 事件不再被吞噬 → 帧率立刻恢复，**因果闭环**。

> 附带现象：丢的伴生事件有时是另一个字段，日志里出现过
> `msBetweenSimulationStart is zero (msBetweenDisplayChange=55.014)` —— 与"随机丢失伴生事件"一致。

## 6. 解决方案

### 6.1 首选：关闭 WeGame 的帧率显示 / 性能监控

WeGame → 设置 → 查找 **帧数显示 / FPS 显示 / 性能监控 / 游戏覆盖层 / 环境检测** 一类开关并关闭。
（对应 `wegame_environment\wegame_env.exe` 的 FPS 监控与 `PerfOverlay.exe`。
WeGame 自身的日志与 `data\json\platform_monitor_control.js` 为加密/混淆内容，无法离线确认确切开关名。）

### 6.2 兜底 A：清理泄漏的会话（手动或定时）

管理员执行：

```cmd
logman stop "ETWSessionRecorder" -ets
logman query "ETWSessionRecorder" -ets
```

第二条应显示"找不到"或不再列出该会话。可做每 5 分钟自动清理（管理员 PowerShell）：

```powershell
$action    = New-ScheduledTaskAction -Execute 'cmd.exe' -Argument '/c logman stop "ETWSessionRecorder" -ets'
$trigger   = New-ScheduledTaskTrigger -Once -At (Get-Date).AddMinutes(1) `
               -RepetitionInterval (New-TimeSpan -Minutes 5) -RepetitionDuration (New-TimeSpan -Days 3650)
$principal = New-ScheduledTaskPrincipal -UserId 'SYSTEM' -LogonType ServiceAccount -RunLevel Highest
Register-ScheduledTask -TaskName 'StopETWSessionRecorder' -Action $action -Trigger $trigger -Principal $principal -Force
```

撤销：`Unregister-ScheduledTask -TaskName 'StopETWSessionRecorder' -Confirm:$false`

### 6.3 兜底 B（野路子，谨慎）

将 `E:\wegame\wegame_environment\PerfOverlay.exe` 改名（如 `PerfOverlay.exe.bak`）。
WeGame 可能校验并重新下载，因此优先使用 6.1 的正规开关。

### 6.4 验收标准

1. 关闭 WeGame 的帧率监控（或执行 6.2）后，玩一局游戏 → 浮窗出现**真实帧数**；
2. **WeGame 关闭后** `logman query "ETWSessionRecorder" -ets` → 找不到该会话；
3. `PresentMon_Consumer.log` 不再疯长：

```powershell
$f='C:\ProgramData\NVIDIA Corporation\FrameViewSDK\PresentMon_Consumer.log'
$l=Get-Content $f -Tail 3000
"含 N/A 行: " + ($l | Where-Object { $_ -match 'reporting N/A' }).Count + " / " + $l.Count
```
正常应为"少量"（< 5 行/秒量级），而不是满屏。

## 7. 已排除的原因（避免重复劳动）

| 假设 | 排除依据 |
|---|---|
| 驱动版本问题 | 596.49 → 617.14（含重装、更新）症状不变 |
| NVIDIA App 版本问题 | 自更新报告已是最新（11.0.9.251）；装 FrameView 2.0 亦无效 |
| DLSS / 帧生成（DLFG） | 关闭 DLSS 与帧生成后，绝区零/彩虹六号仍为 `9999`；日志中 `DLFG` 行数为 0 |
| 内核反作弊（ACE / BattlEye） | 反作弊驱动正在运行时帧率也可正常（重启后绝区零、R6 均正常） |
| UU远程 虚拟显示器 | 禁用 `GameViewer Virtual Display Adapter` + 停止 `GameViewerService` 后症状不变 |
| 壁纸引擎 / 浏览器 ETW 噪声 | 排除后无效；且丢事件量与它们无相关性 |
| 显示器/接口 | 确认显示器直连 RTX 5060（`display_active=Enabled`），DP 200 Hz |
| 系统 IP/DNS/网络 | 与帧率无关（另一独立故障已单独定位，见下） |

**7.3 排查过程中另外发现并已解决的三个独立问题**（与本故障无因果、但都表现为"浮窗数据异常"）：

1. **NVIDIA-DD-External ETW 清单未注册**：`System32\ddETWExternal.dll` 缺失、提供程序未注册。
   用 FrameView SDK 自带脚本修复：
   `cd /d "C:\Program Files\NVIDIA Corporation\FrameViewSDK\etw_nv" && nvinstall.cmd`。
2. **`FvSvc` 服务卡死在 `STOP_PENDING`**（在一次"关机被快速启动转为休眠/恢复"的过程中，
   `FrameViewSvc.log` 记录 `StopFvContainer Failed to run taskkill. Error: 1008`），
   导致浮窗**全部**指标 `N/A`，且 `FvKMDSvc` 服务项消失。
   解决：**完整重启**（注意：启用"快速启动"时，"关机→开机"是休眠/恢复，不会重建服务，
   必须用"重启"，或按住 Shift 点关机）。
3. **GPU 遥测定格**（利用率/温度停在待机值）：`nvtopps.log` 出现
   `IpcDispatcher::OnMessage: Message was rejected!` 与多条 `NvAPI_InitializeEx` 错误。
   解决：完全退出 NVIDIA App → 重启 `NvContainerLocalSystem` → 重开 App → 重进游戏。

## 8. 给厂商的反馈模板

### 8.1 腾讯 WeGame（责任方）

> **ETW 会话泄漏导致第三方性能叠加层失效**
>
> `wegame_environment\wegame_env.exe`（WeGame Environmental Center v1.0.0.1，含 `fps_monitor.cpp` /
> `CFPSMonitor`）会创建**实时模式** ETW 会话 `ETWSessionRecorder`，订阅
> `Microsoft-Windows-DxgKrnl` 与 `Microsoft-Windows-Dwm-Core` 的全部关键字。其 `[StopETW]`
> 停止逻辑失败后，该会话**永久残留**（Windows 事件日志
> `Microsoft-Windows-Kernel-EventTracing/Admin` 每约 2 分钟记录
> "实时会话 ETWSessionRecorder 的备份文件已达到其最大大小 … 没有任何实时使用者"），
> 并出现 `会话"ETWSessionRecorder"未能启动，错误 0xC0000035`（同名残留）。
>
> 后果：**NVIDIA App 统计浮窗的 FPS 平均 / 1% Low 恒显示 9999**（FrameView/PresentMon 日志中
> 每帧 `msBetweenDisplayChange is zero … reporting N/A`，且 `ETW events were lost` 达
> 89,352 / 248,878 / 1,830,859），必须手动 `logman stop "ETWSessionRecorder" -ets` 或重启系统才能恢复。
>
> 环境：Windows 11 25H2 (26200.8457) / RTX 5060 / 驱动 617.14 / NVIDIA App 11.0.9.251。
> 起因为 2026-09-08 `PerfOverlay.exe` 更新后出现。
> **请修复：退出或停止帧率监控时可靠释放 ETW 会话；如无法停止，至少不要以实时模式订阅全部关键字。**

### 8.2 NVIDIA

> 当系统存在其他程序遗留的"实时模式 + 无消费者"ETW 会话（且订阅 DxgKrnl / Dwm-Core）时，
> FrameView/PresentMon 会大量丢事件（`ETW events were lost` 达 89,352–1,830,859），
> 统计浮窗的 FPS / 1% Low 恒显示 `9999`，且**没有任何诊断提示**。
> 建议：检测到"显示时刻"事件缺失或大批丢事件时给出明确提示（例如提示存在冲突的 ETW 会话），
> 并考虑在会话被挤占时自动重试/降级，而不是静默显示 `9999`。

## 9. 附录

### 9.1 只读诊断命令清单（可用于同类问题）

```powershell
# 1) 浮窗数据是否在报"缺显示时刻"
$f='C:\ProgramData\NVIDIA Corporation\FrameViewSDK\PresentMon_Consumer.log'
$l=Get-Content $f -Tail 2000
"含 N/A: " + ($l | Where-Object { $_ -match 'reporting N/A' }).Count + " / " + $l.Count

# 2) 是否存在"实时但无消费者"的 ETW 会话（管理员）
logman query -ets
logman query "ETWSessionRecorder" -ets

# 3) 内核层面的证据（缓冲区满 / 撞名失败）
Get-WinEvent -LogName 'Microsoft-Windows-Kernel-EventTracing/Admin' -MaxEvents 200 |
  Where-Object { $_.Message -match 'ETWSessionRecorder' } |
  Select-Object TimeCreated, Id, @{n='Msg';e={($_.Message -replace "`r`n",' ')}}

# 4) FrameView 服务链路健康度
sc.exe query FvSvc ; sc.exe query FvKMDSvc

# 5) GPU 遥测代理是否在报错
Get-Content 'C:\ProgramData\NVIDIA Corporation\nvtopps\nvtopps.log' -Tail 200 | Select-String ERROR
```

### 9.2 关键路径

| 用途 | 路径 |
|---|---|
| 帧采集日志（NVIDIA App） | `C:\ProgramData\NVIDIA Corporation\FrameViewSDK\PresentMon_Consumer.log` |
| FrameView 服务日志 | `C:\ProgramData\NVIDIA Corporation\FrameViewSDK\FrameViewSvc.log`、`container*.log` |
| NVIDIA App 侧日志 | `%LOCALAPPDATA%\NVIDIA Corporation\NVIDIA App\CxNative_NVIDIA App.log` |
| GPU 遥测日志 | `C:\ProgramData\NVIDIA Corporation\nvtopps\nvtopps.log` |
| DD 提供程序清单脚本 | `C:\Program Files\NVIDIA Corporation\FrameViewSDK\etw_nv\nvinstall.cmd` |
| **元凶** | `E:\wegame\wegame_environment\wegame_env.exe`、`PerfOverlay.exe` |
| 内核 ETW 日志 | 事件查看器 → 应用程序和服务日志 → Microsoft → Windows → Kernel-EventTracing → Admin |

### 9.3 通用脚本：抓"泄漏 ETW 会话"的创建者

当会话没有保存定义、字符串又不在任何程序文件里时（本案例正是如此），只能"抓活的"。
思路：ETS 会话创建**成功**时不留 PID，但**撞名失败**时会写事件 ID 2 并带 `ProcessId`。
因此：**保持会话运行**，然后启动可疑程序，用下面的脚本捕获发起进程：

```powershell
$last = (Get-WinEvent -LogName 'Microsoft-Windows-Kernel-EventTracing/Admin' -MaxEvents 300 -ErrorAction SilentlyContinue |
         Where-Object { $_.Id -eq 2 -and $_.Message -match 'ETWSessionRecorder' } |
         Measure-Object RecordId -Maximum).Maximum
if (-not $last) { $last = 0 }
while ($true) {
  Get-WinEvent -LogName 'Microsoft-Windows-Kernel-EventTracing/Admin' -MaxEvents 60 -ErrorAction SilentlyContinue |
    Where-Object { $_.Id -eq 2 -and $_.Message -match 'ETWSessionRecorder' -and $_.RecordId -gt $last } |
    ForEach-Object {
      $last = $_.RecordId
      $p = Get-Process -Id $_.ProcessId -ErrorAction SilentlyContinue
      $path = try { $p.Path } catch { '<无权限>' }
      "★ {0}  PID={1}  进程={2}  路径={3}" -f $_.TimeCreated, $_.ProcessId, $p.ProcessName, $path
    }
  Start-Sleep -Seconds 3
}
```

补充手段：开机自启的监视任务（写日志）记录"会话出现的时刻 + 前台窗口 + 最近启动的进程 + 当时全部进程"，
用于定位"被用户手动启动"的创建者；对**常驻进程**型的创建者，还需记录"当时全部进程"而非只看最近启动的。

---

*文档由一次实际故障排查整理而成；所有结论均附可复现的命令与日志证据。*
