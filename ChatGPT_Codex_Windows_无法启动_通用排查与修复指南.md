# ChatGPT / Codex Windows 客户端无法启动：通用排查与修复指南

> **版本：2026-09-01 GitHub Issue 综合版**  
> **适用对象：** Microsoft Store / MSIX 安装的 Windows 版 ChatGPT / Codex Desktop。  
> **内容来源：** 一次已经验证成功的真实排查过程，加上 OpenAI 官方 `openai/codex` GitHub 仓库中公开报告的其他 Windows 启动问题。

---

## 0. 使用说明与可信度标记

本文把处理方式分为三类：

- **【本机已验证】**：在实际故障电脑上执行后恢复正常。
- **【GitHub 已报告】**：来自 `openai/codex` 官方仓库中的用户复现或 workaround，但不代表 OpenAI 已正式发布修复说明。
- **【诊断建议】**：用于缩小范围，不应被理解为已确认根因。

GitHub Issue 的状态、适用版本和 workaround 可能随客户端更新发生变化。执行前先确认当前客户端版本，不要把某一个版本的固定目录或 runtime ID 永久写死。

---

# 1. 先看结论：最常见的启动故障类型

| 表现 | 高概率方向 | 首先检查 |
|---|---|---|
| 弹窗提示 `Unable to locate the Codex CLI binary` | CLI 定位失败、MSIX 受保护文件复制失败、杀毒软件隔离、WSL 状态残留 | `CODEX_CLI_PATH`、CLI 版本、启动日志 |
| 设置路径后出现 `spawn EINVAL` | `CODEX_CLI_PATH` 指向了 `codex.cmd` / `.bat` 包装脚本 | 必须改为原生 `codex.exe` |
| 双击后完全无反应，10 秒后也没有进程 | 初始化早期退出、runtime 不完整、原生模块崩溃、安全策略阻止 | 最新日志、Crashpad、`cua_node` staging |
| 后台有多个 ChatGPT/Codex 进程，但没有窗口 | updater 等待、首次 runtime 释放很慢、隐藏 Chromium 窗口、renderer/app-server 未启动 | `MainWindowHandle`、CPU、runtime 文件数是否增长 |
| 启动要等 5～8 分钟 | 更新策略超时、数千个 runtime 文件正在静默复制 | 不要立即杀进程；监测 staging 文件数量 |
| 日志只有 `Launching app` 和 `Appshot hotkey inactive` | 启动卡在 renderer/app-server 之前 | updater、`cua_node`、原生模块、用户路径 |
| 出现 `Couldn't update Agent sandbox` | `sandbox = "elevated"` 配置、Windows sandbox 初始化失败 | `config.toml` 的 `[windows]` 段 |
| WSL 切回 Windows 后持续异常 | WSL app-server 状态残留、Windows/WSL 路径混用 | `runCodexInWindowsSubsystemForLinux` |
| 添加第二个项目文件夹后永久 `Oops` | 项目全局状态中的多 `rootPaths` 触发 renderer 问题 | `.codex-global-state.json` |
| 切换账号后无法启动 | 重复启动、认证文件切换、第三方工具改写配置；也可能只是重新触发已有 runtime bug | 先退出所有进程，再区分认证问题和客户端问题 |
| 只有某个 Windows 用户无法启动 | 非 ASCII 用户目录、权限、WDAC/AppLocker、损坏的用户级状态 | 新建 ASCII 用户做对照测试 |

---

# 2. 安全原则：先不要做什么

除非已经完成针对性排查，否则不要一上来就做：

```text
Repair
Reset
卸载并安装旧版
删除整个 .codex
删除 auth.json
删除所有 LocalAppData\OpenAI\Codex
修改 WindowsApps 所有权
使用 takeown / icacls 给 WindowsApps 完全控制
关闭企业 AppLocker / WDAC
随意把 CODEX_CLI_PATH 指向旧 codex.exe
```

这些操作会引入新的变量，并可能造成：

```text
原问题没有解决
+ 本地设置丢失
+ CLI 版本回退
+ 模型列表异常
+ Store/MSIX 更新失效
```

### WindowsApps 无权限通常是正常现象

以下目录由 `TrustedInstaller` 保护：

```text
C:\Program Files\WindowsApps
D:\WindowsApps
```

普通管理员无法直接浏览，不等于 ChatGPT 权限损坏。需要读取应用包路径时，使用：

```powershell
Get-AppxPackage OpenAI.Codex |
Select-Object Name,Version,Status,PackageFamilyName,InstallLocation
```

---

# 3. 一键只读诊断脚本

下面脚本**不修改配置，不读取 token 内容**，会把报告保存到桌面：

```powershell
$ErrorActionPreference = "SilentlyContinue"
$stamp = Get-Date -Format "yyyyMMdd-HHmmss"
$desktop = [Environment]::GetFolderPath("Desktop")
$outFile = Join-Path $desktop "Codex_Diagnostics_$stamp.txt"
$report = [System.Collections.Generic.List[string]]::new()

function Add-Block {
    param(
        [string]$Title,
        [object]$Value
    )

    $report.Add("==== $Title ====")

    if ($null -eq $Value) {
        $report.Add("(no result)")
    } else {
        $text = ($Value | Out-String).TrimEnd()
        if ([string]::IsNullOrWhiteSpace($text)) {
            $report.Add("(no result)")
        } else {
            $report.Add($text)
        }
    }

    $report.Add("")
}

$pkg = Get-AppxPackage -Name OpenAI.Codex
Add-Block "AppX package" ($pkg | Select-Object Name,Version,Status,PackageFamilyName,InstallLocation)

$cli = [Environment]::GetEnvironmentVariable("CODEX_CLI_PATH", "User")
$cliVersion = $null
$cliSignature = $null
if ($cli -and (Test-Path $cli)) {
    $cliVersion = (& $cli --version 2>$null) -join " "
    $cliSignature = Get-AuthenticodeSignature $cli |
        Select-Object Status,@{N="Signer";E={$_.SignerCertificate.Subject}}
}

Add-Block "CODEX_CLI_PATH" ([PSCustomObject]@{
    Path = $cli
    Exists = if ($cli) { Test-Path $cli } else { $false }
    Version = $cliVersion
})
Add-Block "CLI signature" $cliSignature

Add-Block "Environment" ([PSCustomObject]@{
    CODEX_SPARKLE_ENABLED_User = [Environment]::GetEnvironmentVariable("CODEX_SPARKLE_ENABLED", "User")
    CODEX_SPARKLE_ENABLED_Process = $env:CODEX_SPARKLE_ENABLED
    USERPROFILE = $env:USERPROFILE
    TEMP = $env:TEMP
})

$config = "$env:USERPROFILE\.codex\config.toml"
$safeConfig = Select-String `
    -Path $config `
    -Pattern "model_provider|base_url|openai_base_url|runCodexInWindowsSubsystemForLinux|sandbox|approval" `
    -ErrorAction SilentlyContinue |
    ForEach-Object { $_.Line }
Add-Block "Safe config lines" $safeConfig

$authInfo = Get-Item `
    "$env:USERPROFILE\.codex\auth.json", `
    "$env:USERPROFILE\.codex\cap_sid" `
    -ErrorAction SilentlyContinue |
    Select-Object Name,Length,LastWriteTime
Add-Block "Auth file metadata only" $authInfo

$processes = Get-Process ChatGPT,Codex -ErrorAction SilentlyContinue |
    Select-Object ProcessName,Id,CPU,Responding,MainWindowHandle,StartTime,Path
Add-Block "Processes" $processes

$runtimeRoot = "$env:LOCALAPPDATA\OpenAI\Codex\runtimes\cua_node"
$stagingInfo = foreach ($dir in (Get-ChildItem $runtimeRoot -Directory -Force -ErrorAction SilentlyContinue |
    Where-Object { $_.Name -like ".staging-*" } |
    Sort-Object LastWriteTime -Descending |
    Select-Object -First 8)) {

    [PSCustomObject]@{
        Path = $dir.FullName
        Files = (Get-ChildItem $dir.FullName -Recurse -File -Force -ErrorAction SilentlyContinue).Count
        NodeExe = Test-Path (Join-Path $dir.FullName "bin\node.exe")
        NodeRepl = Test-Path (Join-Path $dir.FullName "bin\node_repl.exe")
        Updated = $dir.LastWriteTime
    }
}
Add-Block "CUA staging" $stagingInfo

$completedRuntime = foreach ($dir in (Get-ChildItem $runtimeRoot -Directory -Force -ErrorAction SilentlyContinue |
    Where-Object { $_.Name -notlike ".staging-*" } |
    Sort-Object LastWriteTime -Descending)) {

    [PSCustomObject]@{
        Path = $dir.FullName
        Files = (Get-ChildItem $dir.FullName -Recurse -File -Force -ErrorAction SilentlyContinue).Count
        NodeExe = Test-Path (Join-Path $dir.FullName "bin\node.exe")
        NodeRepl = Test-Path (Join-Path $dir.FullName "bin\node_repl.exe")
        Updated = $dir.LastWriteTime
    }
}
Add-Block "Completed CUA runtimes" $completedRuntime

$logRoot = "$env:LOCALAPPDATA\Packages\OpenAI.Codex_2p2nqsd0c76g0\LocalCache\Local\Codex\Logs"
$latestLog = Get-ChildItem $logRoot -Recurse -Filter "*.log" -ErrorAction SilentlyContinue |
    Sort-Object LastWriteTime -Descending |
    Select-Object -First 1

Add-Block "Latest desktop log" ($latestLog | Select-Object FullName,Length,LastWriteTime)
if ($latestLog) {
    Add-Block "Latest desktop log tail" (Get-Content $latestLog.FullName -Tail 120)
}

$crashRoot = "$env:LOCALAPPDATA\Packages\OpenAI.Codex_2p2nqsd0c76g0\LocalCache\Roaming\Codex\web\Codex\Crashpad"
$crashFiles = Get-ChildItem $crashRoot -Recurse -File -ErrorAction SilentlyContinue |
    Sort-Object LastWriteTime -Descending |
    Select-Object -First 15 Name,Length,LastWriteTime,FullName
Add-Block "Crashpad" $crashFiles

$report | Set-Content -Path $outFile -Encoding UTF8
Write-Host "诊断报告已保存：$outFile"
```

> 分享诊断报告前，仍建议检查并遮挡用户名、项目路径和公司目录。脚本不会输出 `auth.json` 内容。

---

# 4. 问题一：找不到 Codex CLI

## 4.1 症状

```text
ChatGPT failed to start.
Unable to locate the Codex CLI binary.
Set CODEX_CLI_PATH or ensure the Electron resources include bin/codex.
```

## 4.2 先搜索本机所有原生 CLI

```powershell
$patterns = @(
    "$env:LOCALAPPDATA\Programs\OpenAI\Codex\bin\codex.exe",
    "$env:USERPROFILE\.codex\packages\standalone\releases\*\bin\codex.exe",
    "$env:LOCALAPPDATA\OpenAI\Codex\bin\*\codex.exe"
)

$candidates = foreach ($pattern in $patterns) {
    Get-ChildItem $pattern -ErrorAction SilentlyContinue | ForEach-Object {
        $versionText = (& $_.FullName --version 2>$null) -join " "
        $signature = Get-AuthenticodeSignature $_.FullName

        [PSCustomObject]@{
            Path = $_.FullName
            Version = $versionText
            Signature = $signature.Status
            Updated = $_.LastWriteTime
        }
    }
}

$candidates |
Sort-Object Updated -Descending |
Format-List
```

### 必须注意

`CODEX_CLI_PATH` 必须指向：

```text
codex.exe
```

不要指向：

```text
codex.cmd
codex.bat
Linux / WSL 内的 codex
```

GitHub Issue #40752 报告，Windows Electron 直接 spawn `codex.cmd` 会出现：

```text
spawn EINVAL
```

## 4.3 设置正确路径

```powershell
$goodCli = "C:\完整路径\codex.exe"

Test-Path $goodCli
& $goodCli --version

[Environment]::SetEnvironmentVariable(
    "CODEX_CLI_PATH",
    $goodCli,
    "User"
)
```

结束客户端并重启：

```powershell
Get-Process ChatGPT,Codex -ErrorAction SilentlyContinue |
Stop-Process -Force
```

---

# 5. 问题二：MSIX/Application Protected 文件复制失败

## 5.1 典型特征

虽然安装包中的 `codex.exe` 存在，但启动仍提示找不到 CLI；日志出现：

```text
bundled_executable_relocation_failed
ERROR_ENCRYPTION_FAILED
0x80071770
The specified file could not be encrypted
```

这是 GitHub #40700、#40791、#40843 报告的故障类型：Store/MSIX 包中的文件带有 `Application Protected` 属性，普通复制方法无法把它们正确重定位到用户目录。

## 5.2 检查

```powershell
$pkg = Get-AppxPackage OpenAI.Codex
$bundledCli = Join-Path $pkg.InstallLocation "app\resources\codex.exe"

Test-Path $bundledCli
cipher /c $bundledCli
```

不要修改 WindowsApps ACL，也不要尝试直接取得所有权。

## 5.3 低风险临时方案

优先使用已经安装在用户目录、签名有效并且版本合适的原生 `codex.exe`，通过 `CODEX_CLI_PATH` 绕过失败的 bundled relocation。

如果使用外部 CLI，验证：

```powershell
& $goodCli --version
Get-AuthenticodeSignature $goodCli |
Format-List Status,SignerCertificate
```

---

# 6. 问题三：cua_node Runtime staging 不完整

## 6.1 症状

```text
双击无反应
→ ChatGPT.exe 短暂启动或后台驻留
→ 无 renderer
→ 无 codex.exe app-server
→ 无窗口
```

日志可能只剩：

```text
Launching app ...
Appshot hotkey inactive ...
```

同时：

```text
%LOCALAPPDATA%\OpenAI\Codex\runtimes\cua_node
```

出现多个：

```text
.staging-<runtime-id>-<随机串>
```

其中：

```text
node.exe      = True
node_repl.exe = False
```

这是本次故障的**最终已验证根因**，也与 GitHub #41540、#41654、#41850 等报告一致。

## 6.2 只读检查

```powershell
$runtimeRoot = "$env:LOCALAPPDATA\OpenAI\Codex\runtimes\cua_node"

Get-ChildItem $runtimeRoot -Directory -Force -ErrorAction SilentlyContinue |
Where-Object { $_.Name -like ".staging-*" } |
Sort-Object LastWriteTime -Descending |
ForEach-Object {
    [PSCustomObject]@{
        Path = $_.FullName
        Files = (Get-ChildItem $_.FullName -Recurse -File -Force -ErrorAction SilentlyContinue).Count
        NodeExe = Test-Path (Join-Path $_.FullName "bin\node.exe")
        NodeRepl = Test-Path (Join-Path $_.FullName "bin\node_repl.exe")
        Updated = $_.LastWriteTime
    }
} | Format-List
```

## 6.3 通用版修复脚本

该脚本会从**最新 staging 名称自动提取当前 runtime ID**，避免写死旧版本 ID。

```powershell
Get-Process ChatGPT,Codex -ErrorAction SilentlyContinue |
Stop-Process -Force

$pkg = Get-AppxPackage OpenAI.Codex
$src = Join-Path $pkg.InstallLocation "app\resources\cua_node"
$runtimeRoot = "$env:LOCALAPPDATA\OpenAI\Codex\runtimes\cua_node"

$latestStaging = Get-ChildItem $runtimeRoot `
    -Directory `
    -Force `
    -ErrorAction SilentlyContinue |
Where-Object { $_.Name -like ".staging-*" } |
Sort-Object LastWriteTime -Descending |
Select-Object -First 1

if (-not $latestStaging) {
    throw "没有发现 cua_node staging 目录，不能自动判断 runtime ID。请先查看日志，不要盲目复制。"
}

if ($latestStaging.Name -notmatch '^\.staging-([0-9a-fA-F]{16})-') {
    throw "无法从 staging 名称解析 runtime ID：$($latestStaging.Name)"
}

$runtimeId = $Matches[1]
$dst = Join-Path $runtimeRoot $runtimeId

if (-not (Test-Path $src)) {
    throw "官方包内没有找到 cua_node：$src"
}

New-Item -ItemType Directory -Force -Path $dst | Out-Null

Write-Host "源目录：$src"
Write-Host "目标目录：$dst"
Write-Host "runtime ID：$runtimeId"

cmd /c "xcopy `"$src\*`" `"$dst\`" /E /I /H /K /G /Y"

$sourceCount = (Get-ChildItem $src -Recurse -File -Force -ErrorAction SilentlyContinue).Count
$destCount = (Get-ChildItem $dst -Recurse -File -Force -ErrorAction SilentlyContinue).Count

[PSCustomObject]@{
    SourceFiles = $sourceCount
    DestinationFiles = $destCount
    NodeExe = Test-Path (Join-Path $dst "bin\node.exe")
    NodeRepl = Test-Path (Join-Path $dst "bin\node_repl.exe")
    Destination = $dst
} | Format-List
```

理想结果：

```text
NodeExe  : True
NodeRepl : True
SourceFiles 与 DestinationFiles 接近或一致
```

### 为什么使用 `xcopy /G`

`/G` 允许把受 EFS/Application Protected 保护的源文件复制到不支持相同保护属性的目标位置。普通 `Copy-Item` 在相关故障机器上可能报 `0x80071770`。

### 不要只补一个文件

GitHub 中有个别案例只补 `node_repl.exe` 即可恢复，但另一些案例下次启动会重新创建新的 staging 并再次失败。更稳妥的方式是复制**完整官方 `cua_node` runtime**。

---

# 7. 问题四：其实不是卡死，而是在静默释放数千个文件

GitHub #41073、#41339、#41523、#41822 等报告显示，更新后的首次启动可能需要数分钟：

```text
后台进程存在
MainWindowHandle = 0
CPU 或磁盘仍有活动
staging 文件数量持续增长
最终窗口自动出现
```

## 7.1 不要立即终止

如果 CPU、磁盘或 staging 文件数仍在变化，先观察 2～8 分钟。

监测：

```powershell
$runtimeRoot = "$env:LOCALAPPDATA\OpenAI\Codex\runtimes\cua_node"

1..20 | ForEach-Object {
    $latest = Get-ChildItem $runtimeRoot -Directory -Force -ErrorAction SilentlyContinue |
        Where-Object { $_.Name -like ".staging-*" } |
        Sort-Object LastWriteTime -Descending |
        Select-Object -First 1

    $count = if ($latest) {
        (Get-ChildItem $latest.FullName -Recurse -File -Force -ErrorAction SilentlyContinue).Count
    } else {
        0
    }

    [PSCustomObject]@{
        Time = Get-Date
        Staging = $latest.Name
        Files = $count
    }

    Start-Sleep -Seconds 15
}
```

如果文件数不断增加，不要重复双击或频繁杀进程。

如果文件数长时间不变，并且 `node_repl.exe=False`，转到第 6 章。

---

# 8. 问题五：Updater / 更新策略阻塞启动

## 8.1 典型症状

```text
后台进程存在
无窗口
日志显示 enableUpdater=true
数分钟后才继续
```

GitHub #41073 报告，在部分版本中临时禁用 updater 并从 MSIX 包上下文启动可以恢复。

## 8.2 只对当前 PowerShell 会话禁用

```powershell
Get-Process ChatGPT,Codex -ErrorAction SilentlyContinue |
Stop-Process -Force

Start-Sleep -Seconds 3

$env:CODEX_SPARKLE_ENABLED = "false"

$pkg = Get-AppxPackage -Name OpenAI.Codex
$exe = Join-Path $pkg.InstallLocation "app\ChatGPT.exe"

Invoke-CommandInDesktopPackage `
    -PackageFamilyName $pkg.PackageFamilyName `
    -AppId "App" `
    -Command $exe
```

## 8.3 持久禁用

仅在当前版本确认 updater 会触发问题时使用：

```powershell
[Environment]::SetEnvironmentVariable(
    "CODEX_SPARKLE_ENABLED",
    "false",
    "User"
)
```

## 8.4 恢复更新

```powershell
[Environment]::SetEnvironmentVariable(
    "CODEX_SPARKLE_ENABLED",
    $null,
    "User"
)

Remove-Item Env:CODEX_SPARKLE_ENABLED -ErrorAction SilentlyContinue
```

> 临时禁用 updater 不等于卸载或降级模型；但长期关闭后需要手动通过 Microsoft Store 检查新版本。

---

# 9. 问题六：隐藏窗口与无 renderer/app-server

## 9.1 进程存在不代表 UI 已正常创建

```powershell
Get-Process ChatGPT,Codex -ErrorAction SilentlyContinue |
Select-Object ProcessName,Id,CPU,Responding,MainWindowHandle,StartTime
```

```text
MainWindowHandle = 0
```

可能代表：

1. 主窗口还没创建；
2. Chromium 顶层窗口已创建但隐藏；
3. renderer 没启动；
4. UI 线程被 updater/runtime materializer 阻塞。

GitHub #41696 有隐藏 `Chrome_WidgetWin_0` 的案例，但 `ShowWindow` 并非所有机器都有效，甚至可能让调用者长时间阻塞。因此不要把强制 ShowWindow 当作通用修复。

## 9.2 检查 renderer 与 app-server

```powershell
Get-CimInstance Win32_Process |
Where-Object {
    $_.Name -in @("ChatGPT.exe", "codex.exe")
} |
Select-Object Name,ProcessId,CommandLine
```

如果有主进程、GPU、utility，但没有：

```text
ChatGPT.exe --type=renderer
codex.exe app-server
```

优先检查第 6 章 runtime staging 和第 8 章 updater。

---

# 10. 问题七：WSL 与 Windows Native 状态残留

GitHub #23894、#40776、#29881 报告过：

```text
WSL 模式开启
→ 更新或切回 Windows Native
→ app-server 仍由 wsl.exe 启动
→ Windows sandbox 重复初始化或 CLI 定位失败
```

## 10.1 检查配置

```powershell
Select-String `
    -Path "$env:USERPROFILE\.codex\config.toml" `
    -Pattern "runCodexInWindowsSubsystemForLinux|sandbox" `
    -ErrorAction SilentlyContinue
```

如果存在：

```toml
runCodexInWindowsSubsystemForLinux = true
```

并且当前想使用 Windows Native，可先备份：

```powershell
$config = "$env:USERPROFILE\.codex\config.toml"
Copy-Item $config "$config.bak-$(Get-Date -Format 'yyyyMMdd-HHmmss')"
```

然后手动改为：

```toml
runCodexInWindowsSubsystemForLinux = false
```

完全结束 ChatGPT/Codex 后再启动。

如果 App 能打开但 Windows sandbox 未初始化，可尝试在设置里：

```text
Windows Native → WSL → Windows Native
```

用来重新触发 sandbox setup。该方法是社区报告的恢复方式，不是永久修复。

---

# 11. 问题八：Windows sandbox 配置阻塞启动

GitHub #29622 报告：

```toml
[windows]
sandbox = "elevated"
```

可能使客户端卡在：

```text
Couldn't update Agent sandbox
Retry the update to continue
```

可在备份后暂时改为：

```toml
[windows]
sandbox = "unelevated"
```

不要混淆：

```toml
sandbox_mode = "danger-full-access"
```

与：

```toml
[windows]
sandbox = "elevated"
```

两者不是同一个配置层级。

如果出现 COM+ 注册表、`ShellExecuteExW 1223`、模块找不到等 sandbox setup 错误，先检查：

- UAC 是否正常启用；
- Windows 是否为 Insider/Beta 特殊版本；
- 是否有企业策略阻止 setup helper；
- 是否有第三方代理/环境文件影响；
- `codex-windows-sandbox-setup.exe` 是否存在。

不要通过关闭所有系统安全机制来解决。

---

# 12. 问题九：config.toml 无效或被第三方工具改写

无效的配置键或值可能让 CLI、扩展或桌面端无法继续初始化。GitHub #4860 中，错误的 `approval_policy = "auto"` 导致启动失败。

## 最安全的对照测试

```powershell
Get-Process ChatGPT,Codex -ErrorAction SilentlyContinue |
Stop-Process -Force

$config = "$env:USERPROFILE\.codex\config.toml"
$backup = "$config.disabled-$(Get-Date -Format 'yyyyMMdd-HHmmss')"

if (Test-Path $config) {
    Move-Item $config $backup
    Write-Host "配置已临时移走：$backup"
}
```

重新启动客户端。

如果恢复，说明问题位于配置文件。不要把整个 `.codex` 删除；应逐项恢复配置。

恢复：

```powershell
Move-Item "备份文件完整路径" "$env:USERPROFILE\.codex\config.toml"
```

---

# 13. 问题十：多账号切换工具触发异常

即使所有账号都是官方 ChatGPT 账号，第三方切换工具仍可能：

- 替换 `auth.json`；
- 改写 `config.toml`；
- 自动启动客户端；
- 在客户端还没完全退出时产生多个启动请求。

## 推荐流程

```text
1. 完全退出 ChatGPT/Codex
2. 结束所有残留进程
3. 使用工具只切换账号，不自动 launch
4. 退出工具托盘进程
5. 等待 3～5 秒
6. 手动只启动一次客户端
```

结束残留：

```powershell
Get-Process ChatGPT,Codex -ErrorAction SilentlyContinue |
Stop-Process -Force
```

验证 JSON 格式，但不要输出 token：

```powershell
$auth = "$env:USERPROFILE\.codex\auth.json"

try {
    Get-Content $auth -Raw | ConvertFrom-Json | Out-Null
    Write-Host "auth.json 格式正常"
} catch {
    Write-Host "auth.json 格式异常"
}

Get-Item $auth | Select-Object Length,LastWriteTime
```

### 重要判断

如果账号切换后不启动，但最终通过补齐 `cua_node` 恢复，则账号工具只是**触发重新启动的动作**，不一定是根因。

---

# 14. 问题十一：杀毒软件隔离 bundled codex.exe

GitHub #32463 报告，安全软件隔离 WindowsApps 中的 `codex.exe` 后，即使用户目录已有可用 CLI，客户端仍可能在 bootstrap 阶段失败。

## 检查

```powershell
$pkg = Get-AppxPackage OpenAI.Codex
$bundledCli = Join-Path $pkg.InstallLocation "app\resources\codex.exe"

Test-Path $bundledCli
```

同时查看：

```text
Windows 安全中心 → 病毒和威胁防护 → 保护历史记录
第三方杀毒软件 → 隔离区 / 检测历史
```

恢复文件前必须确认：

- 文件来自官方 Microsoft Store/MSIX 包；
- Authenticode 签名有效；
- 不是从不明网站下载的同名文件。

不要为了启动而全局关闭杀毒软件。企业电脑应由 IT 添加针对官方签名或官方包路径的规则。

---

# 15. 问题十二：AppLocker / WDAC / Code Integrity 阻止运行

如果 `codex.exe`、`node.exe`、`node_repl.exe` 存在但进程立即退出，可能被企业安全策略阻止。

检查事件日志：

```powershell
Get-WinEvent `
    -LogName "Microsoft-Windows-AppLocker/EXE and DLL" `
    -MaxEvents 50 `
    -ErrorAction SilentlyContinue |
Select-Object TimeCreated,Id,LevelDisplayName,Message
```

```powershell
Get-WinEvent `
    -LogName "Microsoft-Windows-CodeIntegrity/Operational" `
    -MaxEvents 50 `
    -ErrorAction SilentlyContinue |
Select-Object TimeCreated,Id,LevelDisplayName,Message
```

不要自行关闭公司策略。把命中的官方文件路径、签名状态和客户端版本提交给 IT。

---

# 16. 问题十三：非 ASCII Windows 用户目录导致原生模块崩溃

GitHub #27780 等旧版本报告：Windows 用户目录包含韩文、中文或其他非 ASCII 字符时，部分原生模块可能在启动早期崩溃，并生成 Crashpad `.dmp`。

例如：

```text
C:\Users\민지
C:\Users\张三
```

## 对照测试

不要直接重命名现有 `C:\Users\...` 目录。

安全方法是：

1. 新建一个仅含 ASCII 字符的本地 Windows 用户；
2. 登录该用户；
3. 安装/启动同版本客户端；
4. 对比是否能正常启动。

如果只有原用户失败，优先检查用户路径和用户级缓存，不要误判为整机安装损坏。

---

# 17. 问题十四：项目全局状态损坏，启动后永久 Oops

GitHub #35137 报告，向已有项目添加第二个文件夹后，`rootPaths` 保存为多个路径，可能导致应用每次启动都显示 `Oops, an error occurred`。

相关文件：

```text
%USERPROFILE%\.codex\.codex-global-state.json
%USERPROFILE%\.codex\.codex-global-state.json.bak
```

## 只在完全匹配该症状时处理

先备份：

```powershell
$state = "$env:USERPROFILE\.codex\.codex-global-state.json"
$stateBak = "$env:USERPROFILE\.codex\.codex-global-state.json.bak"

Copy-Item $state "$state.manual-backup-$(Get-Date -Format 'yyyyMMdd-HHmmss')" -ErrorAction SilentlyContinue
Copy-Item $stateBak "$stateBak.manual-backup-$(Get-Date -Format 'yyyyMMdd-HHmmss')" -ErrorAction SilentlyContinue
```

然后使用文本编辑器，仅删除受影响项目 `rootPaths` 中多余的第二路径，保留原项目路径。

不要清空整个状态文件，否则可能丢失项目列表和本地 UI 状态。

---

# 18. 问题十五：更新后仍引用旧 AppX 路径

GitHub #41753 报告，当前包已经升级，但日志仍尝试访问旧版本 WindowsApps 路径：

```text
OpenAI.Codex_旧版本_...\app\resources\codex.exe
```

## 检查日志

```powershell
$logRoot = "$env:LOCALAPPDATA\Packages\OpenAI.Codex_2p2nqsd0c76g0\LocalCache\Local\Codex\Logs"
$latest = Get-ChildItem $logRoot -Recurse -Filter "*.log" -ErrorAction SilentlyContinue |
    Sort-Object LastWriteTime -Descending |
    Select-Object -First 1

Select-String `
    -Path $latest.FullName `
    -Pattern "bundled_executable_relocation_failed|sourcePath=.*OpenAI.Codex_" `
    -ErrorAction SilentlyContinue
```

如果只看到旧版本缺失路径：

1. 先确认 Store 是否还有待完成更新；
2. 重启 Windows；
3. 再测试；
4. 只有低风险检查均无效时，才考虑 Windows 设置中的 **Repair**；
5. 不要先用 Reset。

Repair 在个别报告中缩短了启动时间，但没有完全清除旧路径告警，因此不是通用根治方案。

---

# 19. 问题十六：安装在非系统 AppX 卷

部分 Issue 在 `D:\WindowsApps` 等次要 AppX 卷上复现 protected-file relocation 问题，但相同问题在 C 盘也可能发生。因此：

- 次要卷是风险因素，不是唯一根因；
- 不要修改 D:\WindowsApps 权限；
- 如果长期反复发生，并且 Store 允许选择安装位置，可把**未来新版**安装到系统盘做对照；
- 移动安装位置不保证解决 EFS/Application Protected 复制逻辑缺陷。

---

# 20. 应用能打开，但 Browser / Computer Use / Node REPL 不可用

这不一定是“无法启动”，但与同一类 runtime relocation 问题有关。

GitHub #32732、#36260 报告：

- `cua_node` 无法从 WindowsApps 正确 staging；
- Browser / Computer Use 显示不可用；
- read-only sandbox 下 `node_repl` 生成的 TEMP 文件对 sandbox 用户不可读；
- 出现 `MODULE_NOT_FOUND`。

如果客户端能打开但工具不可用：

1. 检查 `cua_node` 是否完整；
2. 检查 `node_repl.exe`；
3. 区分 full access 与 read-only 是否只有后者失败；
4. 不要通过给整个 TEMP 目录 Everyone 完全控制来修复；
5. 等待官方修复最小权限 staging。

---

# 21. 日志与事件位置

## 21.1 Desktop 日志

```text
%LOCALAPPDATA%\Packages\OpenAI.Codex_2p2nqsd0c76g0\
LocalCache\Local\Codex\Logs
```

读取最新日志：

```powershell
$logRoot = "$env:LOCALAPPDATA\Packages\OpenAI.Codex_2p2nqsd0c76g0\LocalCache\Local\Codex\Logs"

$latest = Get-ChildItem $logRoot `
    -Recurse `
    -Filter "*.log" `
    -ErrorAction SilentlyContinue |
Sort-Object LastWriteTime -Descending |
Select-Object -First 1

$latest | Select-Object FullName,Length,LastWriteTime
Get-Content $latest.FullName -Tail 150
```

## 21.2 Crashpad

```text
%LOCALAPPDATA%\Packages\OpenAI.Codex_2p2nqsd0c76g0\
LocalCache\Roaming\Codex\web\Codex\Crashpad
```

```powershell
$crashRoot = "$env:LOCALAPPDATA\Packages\OpenAI.Codex_2p2nqsd0c76g0\LocalCache\Roaming\Codex\web\Codex\Crashpad"

Get-ChildItem $crashRoot `
    -Recurse `
    -File `
    -ErrorAction SilentlyContinue |
Sort-Object LastWriteTime -Descending |
Select-Object -First 15 Name,Length,LastWriteTime,FullName
```

只有 `settings.dat`、没有 `.dmp`，通常更像初始化早期退出，而不是传统 crash。

## 21.3 AppModel Runtime

```powershell
Get-WinEvent `
    -LogName "Microsoft-Windows-AppModel-Runtime/Admin" `
    -MaxEvents 50 `
    -ErrorAction SilentlyContinue |
Select-Object TimeCreated,Id,LevelDisplayName,Message
```

## 21.4 AppX 部署

```powershell
Get-WinEvent `
    -LogName "Microsoft-Windows-AppXDeploymentServer/Operational" `
    -MaxEvents 50 `
    -ErrorAction SilentlyContinue |
Select-Object TimeCreated,Id,LevelDisplayName,Message
```

---

# 22. 推荐的标准排查顺序

```text
第 1 步：只点击一次，观察 30～60 秒
↓
第 2 步：检查进程、CPU、MainWindowHandle
↓
第 3 步：读取客户端版本与 AppX Status
↓
第 4 步：检查 CODEX_CLI_PATH 和 codex.exe --version
↓
第 5 步：确认不是 codex.cmd / WSL 路径
↓
第 6 步：读取最新 Desktop 日志
↓
第 7 步：查 cua_node 下的新 .staging-*
↓
第 8 步：判断 staging 文件数是增长还是停止
↓
第 9 步：检查 node_repl.exe
↓
第 10 步：检查 updater / WSL / sandbox / config
↓
第 11 步：检查 AV、AppLocker、非 ASCII 用户路径
↓
第 12 步：完全匹配症状时再处理项目状态或 AppX Repair
↓
最后：才考虑 Reset、卸载、重新安装
```

---

# 23. 其他电脑上的最短应急流程

当另一台电脑出现“双击无反应”时，可先依次执行：

```powershell
# 1. 当前版本
Get-AppxPackage OpenAI.Codex |
Select-Object Name,Version,Status,InstallLocation

# 2. 当前 CLI
$cli = [Environment]::GetEnvironmentVariable("CODEX_CLI_PATH", "User")
$cli
Test-Path $cli
if ($cli -and (Test-Path $cli)) { & $cli --version }

# 3. 进程
Get-Process ChatGPT,Codex -ErrorAction SilentlyContinue |
Select-Object ProcessName,Id,CPU,Responding,MainWindowHandle,StartTime

# 4. 最新 staging
$root = "$env:LOCALAPPDATA\OpenAI\Codex\runtimes\cua_node"
Get-ChildItem $root -Directory -Force -ErrorAction SilentlyContinue |
Where-Object { $_.Name -like ".staging-*" } |
Sort-Object LastWriteTime -Descending |
Select-Object -First 5 Name,LastWriteTime

# 5. 最新日志
$logRoot = "$env:LOCALAPPDATA\Packages\OpenAI.Codex_2p2nqsd0c76g0\LocalCache\Local\Codex\Logs"
$latest = Get-ChildItem $logRoot -Recurse -Filter "*.log" -ErrorAction SilentlyContinue |
    Sort-Object LastWriteTime -Descending |
    Select-Object -First 1
Get-Content $latest.FullName -Tail 100
```

根据结果进入对应章节，不要一次性执行所有修复。

---

# 24. 本次已经验证成功的案例

实际故障状态：

```text
客户端：26.825.6671.0
CLI：0.150.1
model_provider：openai
无反代
AppX Status：Ok
```

失败 runtime：

```text
多个 .staging-415ffebf3d576e9b-*
文件数量：194 / 784 / 909 等
node.exe：True
node_repl.exe：False
```

完整复制官方 `cua_node` 后：

```text
文件数量：4679
node.exe：True
node_repl.exe：True
```

随后客户端恢复启动。

因此该案例的最终根因是：

```text
MSIX/Application Protected cua_node runtime
→ 复制到 LocalAppData 时中断
→ staging 半成品无法 finalization
→ renderer/app-server 未启动
→ 客户端无窗口或早期退出
```

---

# 25. GitHub Issue 来源索引

> 检索日期：2026-09-01。Issue 的 Open/Closed 状态以后可能变化。

| Issue | 主题 |
|---|---|
| [#40752](https://github.com/openai/codex/issues/40752) | `Unable to locate CLI`；`CODEX_CLI_PATH` 指向 `.cmd` 时 `spawn EINVAL` |
| [#40700](https://github.com/openai/codex/issues/40700) | WindowsApps 中 bundled CLI relocation 失败 |
| [#40791](https://github.com/openai/codex/issues/40791) | `ERROR_ENCRYPTION_FAILED` / Application Protected CLI |
| [#40843](https://github.com/openai/codex/issues/40843) | MSIX protected 文件复制失败导致 CLI 缺失假象 |
| [#41120](https://github.com/openai/codex/issues/41120) | 后台进程存在但无 GUI |
| [#41073](https://github.com/openai/codex/issues/41073) | 禁用 updater 后可启动的 headless 案例 |
| [#41339](https://github.com/openai/codex/issues/41339) | 更新策略等待约 5 分钟阻塞启动 |
| [#41523](https://github.com/openai/codex/issues/41523) | 自动更新后首次启动 MainWindowHandle=0 |
| [#41540](https://github.com/openai/codex/issues/41540) | `node_repl.exe` relocation 失败造成 headless |
| [#41654](https://github.com/openai/codex/issues/41654) | `cua_node` staging 不完整；完整 `xcopy /G` workaround |
| [#41752](https://github.com/openai/codex/issues/41752) | 无 renderer、无 app-server、无窗口 |
| [#41850](https://github.com/openai/codex/issues/41850) | staging 无法 finalization 的持续复现 |
| [#23894](https://github.com/openai/codex/issues/23894) | WSL app-server 状态残留破坏 Windows sandbox |
| [#40776](https://github.com/openai/codex/issues/40776) | WSL 模式更新后无法定位 CLI |
| [#29881](https://github.com/openai/codex/issues/29881) | WSL → Native 切换触发 sandbox 初始化 |
| [#29622](https://github.com/openai/codex/issues/29622) | `sandbox = elevated` 阻塞启动 |
| [#32463](https://github.com/openai/codex/issues/32463) | 杀毒软件隔离 bundled `codex.exe` |
| [#27780](https://github.com/openai/codex/issues/27780) | 非 ASCII Windows 用户路径下原生模块崩溃 |
| [#35137](https://github.com/openai/codex/issues/35137) | 添加第二项目目录后状态文件导致永久 Oops |
| [#41753](https://github.com/openai/codex/issues/41753) | 更新后仍探测旧 AppX 路径 |
| [#32732](https://github.com/openai/codex/issues/32732) | Browser/Computer Use runtime staging 失败 |
| [#36260](https://github.com/openai/codex/issues/36260) | read-only sandbox 无法读取 TEMP 中 node_repl kernel |
| [#4860](https://github.com/openai/codex/issues/4860) | 无效 `config.toml` 值阻止启动 |

---

# 26. 最终建议

1. **先诊断，再修复。** 不同“无窗口”现象可能对应 updater、runtime、配置、WSL、安全策略或 renderer 问题。
2. **不要连续双击。** 重复启动会制造多个进程，让判断更困难。
3. **更新后首次启动先观察。** 如果 staging 文件数持续增长，客户端可能只是在静默释放数千个文件。
4. **runtime 修复必须使用当前包与当前 runtime ID。** 不要复制其他电脑或旧版本的二进制。
5. **不要修改 WindowsApps 权限。** 使用 `Get-AppxPackage` 获取源路径，并用支持受保护文件的复制方式。
6. **第三方账号切换工具应只负责切换，不自动启动客户端。** 避免认证替换与多实例启动叠加。
7. **企业安全策略问题交给 IT。** 不要为了运行 Codex 关闭 WDAC/AppLocker。
8. **新版本发布后重新验证 workaround。** 今天有效的环境变量或 runtime 目录，下一版本可能不再适用。
