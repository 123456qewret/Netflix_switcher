# Netflix Switcher

[English](#english) · [中文](#中文)

A single-file Windows batch script that toggles your PC between **daily use** and **movie watching** with one double-click.

单文件 Windows 批处理脚本，双击一次即可在**日常模式**和**观影模式**之间切换。

---

<a name="english"></a>

## English

No install, no dependencies, no background service. Just a `.bat` file with an embedded PowerShell payload.

### Why this exists

Watching 24p/30p content on a high-refresh monitor causes judder, because 24 fps does not divide evenly into 120/144/160 Hz. The usual fix is a manual chore every single time:

1. Open display settings, drop the refresh rate to 60 Hz
2. Turn HDR on
3. Switch to single-display mode so the player does not get confused
4. Disable any virtual display adapter that streaming DRM refuses to trust
5. Undo all of it afterwards

This script does all of that in one click, and does it in both directions.

### What it does

The script is a **toggle**. It reads the current refresh rate of your primary display and picks the opposite mode.

#### Movie mode (`~60 Hz`)

| Step | Action |
| --- | --- |
| 1 | *(optional, prompted)* Close Edge and reset its PlayReady DRM state |
| 2 | Disable every virtual / indirect display adapter |
| 3 | `DisplaySwitch /internal` — PC screen only |
| 4 | Set refresh rate to the nearest 58–62 Hz mode |
| 5 | Turn **HDR on** for all active displays |

#### Daily mode (max Hz)

| Step | Action |
| --- | --- |
| 1 | Turn **HDR off** |
| 2 | Set refresh rate to the **highest** mode available at the current resolution |
| 3 | `DisplaySwitch /extend` — restore multi-monitor |
| 4 | Re-enable the virtual display adapters |
| 5 | Turn HDR off again (the second display only becomes active after Extend) |

### Usage

Download `Netflix_switcher.bat` and double-click it.

That is the whole workflow. The script requests administrator rights automatically (needed to enable/disable PnP devices) and closes itself when done.

You will see a prompt like this when entering movie mode:

```
Reset Edge DRM state (CDM store + origin_data + disabled_times)?
This forces PlayReady to re-provision, which is what restores the 4K tier.
Netflix sign-in is kept.
Type Y for yes, anything else (or Enter) for no [y/N]:
```

**Pressing Enter does nothing to Edge.** Only an explicit `y` / `yes` triggers the reset.

> **Tip:** pin it to your taskbar, or assign a shortcut key via a desktop shortcut's *Properties → Shortcut key*.

### The Edge DRM reset (optional)

<details>
<summary><b>Why streaming services silently drop to 1080p — and what this option fixes</b></summary>

#### The mechanism

Chromium (and therefore Edge) ships a component called `MediaFoundationServiceMonitor`. It scores hardware-DRM failures: CDM errors, MediaFoundation service crashes, and unexpected hardware-context resets.

From [`media_foundation_service_monitor.cc`](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/chrome/browser/media/media_foundation_service_monitor.cc):

```cpp
constexpr int kMaxNumberOfSamples = 20;
constexpr double kMaxAverageFailureScore = 0.1;  // Two failures out of 20.
```

**Two failures out of twenty** trips the threshold. Chromium then writes a timestamped penalty to:

- `Local State` → `media.hardware_secure_decryption.disabled_times`
- `Default\Preferences` → `media_cdm.origin_data` (per-site)

While that penalty is active, hardware DRM (PlayReady SL3000) is **silently disabled for days**, and the browser falls back to software DRM (SL2000). Streaming services cap software DRM at 1080p. Nothing errors out — the 4K tier just quietly disappears.

The penalty compounds: the shorter the gap between two disable events, the longer the next lockout.

#### Why this script can trigger it

The monitor listens for exactly the things this script does:

```cpp
void OnDisplayAdded(...)          { OnPowerOrDisplayChange(); }
void OnDisplaysRemoved(...)       { OnPowerOrDisplayChange(); }
void OnDisplayMetricsChanged(...) { OnPowerOrDisplayChange(); }
void OnSuspend() / OnResume()     { OnPowerOrDisplayChange(); }
```

There is a 5-second grace period, but topology switches and refresh-rate changes while a browser is running can still register as failures.

#### Why clearing cookies does not help

The per-site **origin ID** lives in `media_cdm.origin_data`. Clearing cookies leaves it intact, so the browser reuses the same origin — and therefore the same software-tier credential. You can sign out and back in forever without changing anything.

You can verify which tier you are on by looking at the CDM store:

```
%LOCALAPPDATA%\Microsoft\Edge\User Data\Default\MediaFoundationCdmStore\x86_64\<origin-id>\
```

| File | Meaning |
| --- | --- |
| `...msprhw.hds` | **Hardware** SL3000 — 4K available (`hw` = hardware) |
| `mspr.hds` | **Software** SL2000 — capped at 1080p |

#### What the option actually removes

| Target | Purpose |
| --- | --- |
| `MediaFoundationCdmStore\` | The PlayReady credential itself |
| `media.hardware_secure_decryption` *(Local State)* | Global penalty timer |
| `media_cdm.origin_data` *(Preferences)* | Origin ID map + per-site penalties |
| `CdmStorage.db` | EME persistent sessions bound to those origin IDs |

**Not touched:** bookmarks, passwords, history, extensions, settings, cookies. Your streaming sign-in survives.

</details>

**When to answer `y`:** you are stuck at 1080p on a machine that should support 4K.

**When to answer `n` (default):** every other time.

### Requirements

- Windows 10 / 11
- Windows PowerShell 5.1 (built in — no PowerShell 7 needed)
- Administrator rights (auto-requested via UAC)

For 4K HDR streaming specifically, you also need what the service requires anyway: a compatible GPU, an HDCP 2.2 display, the HEVC Video Extensions from the Microsoft Store, and PlayReady SL3000 support.

### Portability notes

The script contains no hardcoded paths, usernames, or device IDs. Everything resolves at runtime:

```powershell
Join-Path $env:SystemRoot   'System32\DisplaySwitch.exe'
Join-Path $env:LOCALAPPDATA 'Microsoft\Edge\User Data'
```

Browser profiles are enumerated dynamically (`Default`, `Profile 1`, …), so multi-profile setups work.

Virtual displays are detected **by bus, not by name**:

```powershell
$_.InstanceId -match '^(ROOT|SWD)\\'
```

A real GPU is always enumerated under `PCI\`, so a physical adapter can never be disabled by accident. Software display drivers — GameViewer, Parsec, IddSampleDriver, Sunshine, spacedesk — all enumerate under `ROOT\` or `SWD\` and are matched automatically.

#### Known limitation

Mode detection uses the refresh rate as its signal:

```powershell
if ($hz -ge 58 -and $hz -le 62) { # treat as movie mode, switch to daily }
```

**If your daily refresh rate is already 60 Hz, the script will pick the wrong direction.** It suits setups where daily use runs at 100 Hz or higher. Open an issue if you need a different detection strategy.

### Safety

- **Fail-soft:** a failed device toggle or file deletion emits a warning and continues; it never aborts halfway and leaves your displays in a broken state.
- **Idempotent:** already-correct devices are skipped. Re-running the DRM reset on clean prefs leaves the files **byte-for-byte identical**.
- **Surgical JSON edits:** the pref files are modified by exact text excision, not by `ConvertTo-Json`. PowerShell 5.1's JSON round-trip was verified to corrupt Edge's `Local State` into an unparseable file, so it is deliberately avoided.
- **Validated:** the JSON editing was tested against real Edge pref files using `JavaScriptSerializer`, confirming valid output, complete key counts, and preserved sibling keys.

### How it works

The file is a batch/PowerShell polyglot. The batch header self-elevates and then feeds everything after the `#PSSTART` marker to PowerShell:

```bat
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "$c=[IO.File]::ReadAllText($env:SELF);..."
```

Display control uses P/Invoke against `user32.dll`:

| API | Used for |
| --- | --- |
| `EnumDisplayDevices` / `EnumDisplaySettings` | Enumerate adapters and modes |
| `ChangeDisplaySettingsEx` | Apply refresh rate (tested with `CDS_TEST` first) |
| `QueryDisplayConfig` | Enumerate active paths |
| `DisplayConfigSetDeviceInfo` | Toggle HDR |

HDR uses `DISPLAYCONFIG_DEVICE_INFO_SET_HDR_STATE` (type 16) on Windows 11 24H2+, falling back to `SET_ADVANCED_COLOR_STATE` (type 10) on older builds.

### Disclaimer

This script changes display settings and deletes browser DRM state files. It does not bypass, circumvent, or tamper with any DRM protection — it only clears local cache so the browser re-runs its **normal, official** provisioning handshake. Resolution tiers remain entirely at the service provider's discretion.

Provided as-is. Review the source before running it — it is a single readable file.

---

<a name="中文"></a>

## 中文

无需安装，无依赖，无后台服务。只有一个 `.bat` 文件，内嵌 PowerShell 代码。

### 为什么需要它

24p/30p 的影视内容在高刷新率显示器上会出现抖动，因为 24 fps 无法被 120/144/160 Hz 整除。常规做法是每次手动折腾一遍：

1. 打开显示设置，把刷新率降到 60 Hz
2. 打开 HDR
3. 切换到单屏模式，避免播放器识别混乱
4. 禁用流媒体 DRM 不信任的虚拟显示适配器
5. 看完之后再全部改回来

这个脚本把上述操作合并成一次点击，且双向可逆。

### 它做了什么

脚本是**开关式**的：读取主显示器当前刷新率，自动切换到相反的模式。

#### 观影模式（`~60 Hz`）

| 步骤 | 操作 |
| --- | --- |
| 1 | *（可选，会询问）* 关闭 Edge 并重置其 PlayReady DRM 状态 |
| 2 | 禁用所有虚拟 / 间接显示适配器 |
| 3 | `DisplaySwitch /internal` —— 仅电脑屏幕 |
| 4 | 将刷新率设为最接近的 58–62 Hz 模式 |
| 5 | 为所有活动显示器**开启 HDR** |

#### 日常模式（最高刷新率）

| 步骤 | 操作 |
| --- | --- |
| 1 | **关闭 HDR** |
| 2 | 在当前分辨率下设为**最高**可用刷新率 |
| 3 | `DisplaySwitch /extend` —— 恢复多显示器 |
| 4 | 重新启用虚拟显示适配器 |
| 5 | 再次关闭 HDR（第二块屏在扩展之后才会激活） |

### 使用方法

下载 `Netflix_switcher.bat`，双击运行。

就这么简单。脚本会自动申请管理员权限（启用/禁用 PnP 设备需要），执行完毕后自动退出。

进入观影模式时会出现如下提示：

```
Reset Edge DRM state (CDM store + origin_data + disabled_times)?
This forces PlayReady to re-provision, which is what restores the 4K tier.
Netflix sign-in is kept.
Type Y for yes, anything else (or Enter) for no [y/N]:
```

**直接回车不会对 Edge 做任何操作。** 只有明确输入 `y` / `yes` 才会执行重置。

> **提示：** 可以把它固定到任务栏，或者建一个桌面快捷方式，在*属性 → 快捷键*里设置一个热键。

### Edge DRM 重置（可选）

<details>
<summary><b>为什么流媒体会悄悄掉到 1080p —— 以及这个选项修复了什么</b></summary>

#### 机制

Chromium（因此也包括 Edge）内置了一个叫 `MediaFoundationServiceMonitor` 的组件，它会对硬件 DRM 失败打分：CDM 错误、MediaFoundation 服务崩溃、以及意外的硬件上下文重置。

摘自 [`media_foundation_service_monitor.cc`](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/chrome/browser/media/media_foundation_service_monitor.cc)：

```cpp
constexpr int kMaxNumberOfSamples = 20;
constexpr double kMaxAverageFailureScore = 0.1;  // 20 次里错 2 次
```

**20 次采样里错 2 次**就会触发阈值。随后 Chromium 会把一个时间戳惩罚记录写入：

- `Local State` → `media.hardware_secure_decryption.disabled_times`
- `Default\Preferences` → `media_cdm.origin_data`（按站点）

在惩罚生效期间，硬件 DRM（PlayReady SL3000）会被**静默禁用数天**，浏览器降级到软件 DRM（SL2000）。流媒体对软件 DRM 的上限是 1080p。全程不报任何错误 —— 4K 档位就这样悄无声息地消失了。

而且惩罚会累加：两次禁用之间间隔越短，下一次锁定时间越长。

#### 为什么这个脚本自身会触发它

监控器监听的恰好就是这个脚本所做的事情：

```cpp
void OnDisplayAdded(...)          { OnPowerOrDisplayChange(); }
void OnDisplaysRemoved(...)       { OnPowerOrDisplayChange(); }
void OnDisplayMetricsChanged(...) { OnPowerOrDisplayChange(); }
void OnSuspend() / OnResume()     { OnPowerOrDisplayChange(); }
```

虽然有 5 秒宽限期，但在浏览器运行期间切换拓扑、改变刷新率，仍然可能被记为失败。

#### 为什么清 cookie 没用

按站点的 **origin ID** 存放在 `media_cdm.origin_data` 中。清除 cookie 不会动它，浏览器于是继续复用同一个 origin —— 也就继续复用同一份软件档凭据。你可以反复退出登录再登录，结果毫无变化。

可以通过查看 CDM 存储来确认当前处于哪一档：

```
%LOCALAPPDATA%\Microsoft\Edge\User Data\Default\MediaFoundationCdmStore\x86_64\<origin-id>\
```

| 文件 | 含义 |
| --- | --- |
| `...msprhw.hds` | **硬件** SL3000 —— 4K 可用（`hw` 即 hardware） |
| `mspr.hds` | **软件** SL2000 —— 上限 1080p |

#### 这个选项实际删除了什么

| 目标 | 用途 |
| --- | --- |
| `MediaFoundationCdmStore\` | PlayReady 凭据本身 |
| `media.hardware_secure_decryption`（Local State） | 全局惩罚计时器 |
| `media_cdm.origin_data`（Preferences） | origin ID 映射 + 按站点惩罚 |
| `CdmStorage.db` | 绑定到这些 origin ID 的 EME 持久会话 |

**不会动：** 书签、密码、历史记录、扩展、设置、cookie。流媒体的登录状态得以保留。

</details>

**什么时候选 `y`：** 机器本应支持 4K，却一直卡在 1080p。

**什么时候选 `n`（默认）：** 其余所有情况。

### 环境要求

- Windows 10 / 11
- Windows PowerShell 5.1（系统自带，无需 PowerShell 7）
- 管理员权限（通过 UAC 自动申请）

如果目标是 4K HDR 流媒体，还需要满足服务商本来就要求的条件：兼容的 GPU、支持 HDCP 2.2 的显示器、Microsoft Store 的 HEVC 视频扩展，以及 PlayReady SL3000 支持。

### 可移植性说明

脚本内不含任何硬编码路径、用户名或设备 ID，全部在运行时解析：

```powershell
Join-Path $env:SystemRoot   'System32\DisplaySwitch.exe'
Join-Path $env:LOCALAPPDATA 'Microsoft\Edge\User Data'
```

浏览器配置文件是动态枚举的（`Default`、`Profile 1`……），因此多配置文件的环境同样适用。

虚拟显示器**按总线识别，而非按名称**：

```powershell
$_.InstanceId -match '^(ROOT|SWD)\\'
```

真实 GPU 一定枚举在 `PCI\` 下，所以物理显卡不可能被误禁用。软件显示驱动 —— GameViewer、Parsec、IddSampleDriver、Sunshine、spacedesk —— 均枚举在 `ROOT\` 或 `SWD\` 下，会被自动匹配。

#### 已知限制

模式判定以刷新率作为依据：

```powershell
if ($hz -ge 58 -and $hz -le 62) { # 视为观影模式，切换回日常 }
```

**如果你的日常刷新率本来就是 60 Hz，脚本会判断错方向。** 它适用于日常运行在 100 Hz 以上的配置。如果你需要其他判定策略，欢迎提 issue。

### 安全性

- **失败不中断：** 设备开关或文件删除失败时只发出警告并继续，不会中途退出而让显示设置停在错乱状态。
- **幂等：** 已处于正确状态的设备会被跳过。在已清理干净的配置上重复执行 DRM 重置，文件保持**逐字节一致**。
- **精确的 JSON 编辑：** 配置文件通过文本精确切除来修改，而非 `ConvertTo-Json`。经验证，PowerShell 5.1 的 JSON 往返会把 Edge 的 `Local State` 破坏成无法解析的文件，因此刻意避开。
- **已验证：** JSON 编辑功能使用 `JavaScriptSerializer` 在真实 Edge 配置文件上测试通过，确认输出有效、键数完整、兄弟键保留。

### 工作原理

该文件是 batch/PowerShell 混合体。批处理头部先自提权，然后把 `#PSSTART` 标记之后的所有内容交给 PowerShell 执行：

```bat
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "$c=[IO.File]::ReadAllText($env:SELF);..."
```

显示控制通过 P/Invoke 调用 `user32.dll`：

| API | 用途 |
| --- | --- |
| `EnumDisplayDevices` / `EnumDisplaySettings` | 枚举适配器与显示模式 |
| `ChangeDisplaySettingsEx` | 应用刷新率（先用 `CDS_TEST` 试探） |
| `QueryDisplayConfig` | 枚举活动路径 |
| `DisplayConfigSetDeviceInfo` | 切换 HDR |

HDR 在 Windows 11 24H2+ 上使用 `DISPLAYCONFIG_DEVICE_INFO_SET_HDR_STATE`（类型 16），在旧版本上回退到 `SET_ADVANCED_COLOR_STATE`（类型 10）。

### 免责声明

本脚本会修改显示设置并删除浏览器的 DRM 状态文件。它**不会**绕过、规避或篡改任何 DRM 保护 —— 仅清除本地缓存，使浏览器重新执行其**正常的、官方的**置备握手流程。分辨率档位完全由服务提供商决定。

按现状提供，不作任何担保。运行前请先审阅源码 —— 它只有一个可读的文件。

---

## License

MIT
