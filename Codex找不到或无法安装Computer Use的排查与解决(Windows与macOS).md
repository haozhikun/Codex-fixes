# Codex 插件商店搜不到 Computer Use 的排查与解决

> 适用系统：Windows、macOS  
> 更新日期：2026-08-25

## 一、本文处理的具体问题

本文针对的是：

- 打开 Codex/ChatGPT 桌面端的“插件”页面；
- 搜索 `Computer Use`；
- 结果中没有官方 Computer Use 插件，只出现其他第三方插件，或者显示“不可用”。

本文首先处理“插件条目根本搜不到”，同时覆盖一种后续容易误判的情况：插件已经可见、技能也已启用，但设置中没有“任意应用（Any App）”，新任务仍拿不到 Computer Use 后台工具。后者属于原生运行时/后台服务没有加载，不能用重复卸载插件解决。

如果插件条目根本搜不到，macOS 的“屏幕录制”和“辅助功能”权限通常不是原因，因为这些权限是在插件安装并开始使用后才需要。应优先检查可用范围、插件目录同步和本地 bundled plugin 缓存。

## 二、Computer Use 是否只支持 Windows

不是。

OpenAI 官方文档明确说明：在受支持地区，Computer Use 可在 ChatGPT 桌面端的 Windows 和 macOS 上使用，并可配合 ChatGPT Work 和 Codex。正常入口是：

```text
Plugins → Computer Use → Install plugin/Enable
```

因此，macOS 也可能遇到“插件搜不到”，但不能把所有 Windows 修复命令直接复制到 macOS。

## 三、先区分三类原因

### 1. 可用性或账号范围

官方使用了“在受支持地区”这一限定。插件条目缺失可能与以下因素有关：

- 当前地区尚未提供；
- 当前桌面端模式、账号或工作区没有获得该功能；
- 管理员策略禁止安装某些插件；
- 当前应用版本过旧，尚未包含对应入口。

如果同一账号在多台设备上都没有 Computer Use，并且本地日志没有插件目录错误，应先考虑可用性和工作区策略，而不是删除缓存。

### 2. 插件目录或远程目录没有同步

插件页可以打开，但搜索不到官方插件，可能是目录同步失败、网络请求失败或桌面端更新没有完成。此时应先：

1. 更新到当前桌面端版本；
2. 完全退出后重开；
3. 切换一次网络或节点；
4. 检查是否能正常看到其他 OpenAI 官方 bundled 插件。

### 3. 本地 `openai-bundled` 缓存或 marketplace 损坏

更新后出现以下组合现象时，应重点怀疑本地缓存：

- Computer Use 和 Chrome/Browser 同时消失或显示不可用；
- Chrome 扩展本身显示 Connected，但 Codex 仍说插件不可用；
- 日志出现 marketplace manifest、helper path、native pipe 或目录占用错误；
- `openai-bundled` 目录中缺少插件版本、`latest` 指针或 marketplace 清单。

本机这次属于第三类：桌面端更新后，本地 bundled plugin marketplace/缓存结构不完整，导致插件商店里搜不到 Computer Use。修复完整 marketplace 并重新注册后，Computer Use 技能重新出现。这个问题与之前代理导致的 `Reconnecting` 是两件事。

## 四、跨平台通用的安全排查

以下思路 Windows 和 macOS 都适用：

1. 确认当前地区、账号/工作区和应用版本符合官方要求。
2. 完全退出 ChatGPT/Codex，而不只是关闭窗口。
3. 结束残留的桌面端辅助进程。
4. 备份 `$CODEX_HOME` 中的配置和插件缓存。
5. 对损坏的 bundled 插件缓存采用“改名备份后重建”，不要一开始永久删除。
6. 重开应用，让它重新整理插件目录。
7. 回到插件页面搜索 `Computer Use`，确认官方插件条目出现。

默认情况下，`CODEX_HOME` 是：

Windows：

```text
%USERPROFILE%\.codex
```

macOS：

```text
~/.codex
```

常见缓存相对位置是：

```text
plugins/cache/openai-bundled
```

“备份并重建缓存”这一思路可以跨平台，但应用安装包位置、辅助程序格式、协议注册和系统权限并不通用。

## 五、Windows：轻量缓存重建方法

适用于更新后 Chrome/Browser/Computer Use 同时异常，但还没有确认 marketplace 结构严重损坏的情况。

### 原文快速解决方案（完整保留）

当以下目录已经存在，并且判断为旧缓存损坏或版本冲突时：

```text
C:\Users\[user]\.codex\plugins\cache\openai-bundled\
```

原文给出的操作顺序是：

1. 删除该目录下的 `browser` 和 `chrome` 文件夹；
2. 完全关闭 Codex；
3. 按 `Ctrl+Alt+Del`，进入任务管理器，结束 `extension-host.exe` 进程；
4. 重新打开 Codex，让应用重新生成插件缓存。

这套快速方案已经完整写入本文，但必须先确认 `browser`、`chrome` 目录确实存在。为防止误删后无法恢复，更稳妥的做法是先备份，或者将它们分别改名为 `browser.bak` 和 `chrome.bak`；确认重建成功后再删除备份。

如果整个 `openai-bundled` 根目录不存在，或者没有 `extension-host.exe` 进程，这套快速方案不适用，应转到后文的 marketplace 未部署/高级修复流程。

### 1. 先备份

建议至少备份：

```text
%USERPROFILE%\.codex\config.toml
%USERPROFILE%\.codex\.codex-global-state.json
%USERPROFILE%\.codex\plugins\cache\openai-bundled
```

如果还涉及 Chrome 集成，再备份：

```text
%USERPROFILE%\.codex\chrome-native-hosts.json
%LOCALAPPDATA%\OpenAI\extension\com.openai.codexextension.json
```

### 2. 完全退出应用

退出 Codex/ChatGPT 桌面端，然后打开任务管理器，结束残留的：

```text
extension-host.exe
```

如有多个 ChatGPT/Codex 辅助进程，也应先确认它们属于当前桌面端再结束。

### 3. 重建旧插件缓存

检查：

```text
%USERPROFILE%\.codex\plugins\cache\openai-bundled\browser
%USERPROFILE%\.codex\plugins\cache\openai-bundled\chrome
%USERPROFILE%\.codex\plugins\cache\openai-bundled\computer-use
```

如果 `browser` 和 `chrome` 是更新前留下的残缺缓存，可先把它们改名为：

```text
browser.bak
chrome.bak
```

然后重新打开桌面端，让 Codex 重建。相比直接删除，改名更容易恢复。

网上流传的“删除 `browser` 和 `chrome`，结束 `extension-host.exe` 后重开”解决方案，本质上就是清除被占用或版本不一致的插件缓存。它只在缓存确实损坏时有效，不能解决地区、账号、管理员策略或官方目录未开放的问题。

## 六、Windows：marketplace 严重损坏的高级修复

如果轻量方法无效，并且日志或 CLI 明确出现以下错误：

```text
marketplace root does not contain a supported manifest
bundled_plugins_marketplace_resolve_failed
Windows Computer Use helper paths are unavailable
computer-use native pipe startup failed
EBUSY: resource busy or locked
```

则应检查完整结构，而不是继续反复安装 Chrome 扩展。

### 1. 检查配置中的 marketplace

检查 `config.toml` 是否包含错误的临时目录，例如：

```text
%USERPROFILE%\.codex\.tmp\bundled-marketplaces\openai-bundled
```

如果使用本地 marketplace，稳定位置应指向完整目录，例如：

```toml
[marketplaces.openai-bundled]
source_type = "local"
source = '\\?\C:\Users\用户名\.codex\plugins\cache\openai-bundled\marketplace-source'
```

不要照抄用户名；不要在不确认目录完整时添加这段配置。部分客户端版本会把 `openai-bundled` 识别为保留市场。如果命令明确返回 `openai-bundled is reserved`，单纯手写这段配置并不能证明市场已经被客户端核心注册，应继续执行后文“保留市场未挂载时的兼容性绕过方案”。

### 2. 检查 marketplace 必需结构

至少应有：

```text
marketplace-source
├─ .agents
│  └─ plugins
│     └─ marketplace.json
└─ plugins
   ├─ browser
   ├─ chrome
   └─ computer-use
```

只复制 `plugins` 而遗漏 `.agents\plugins\marketplace.json`，目录仍然不会被识别为有效 marketplace。

### 3. 从当前安装包恢复时的注意事项

Microsoft Store/AppX 版本的资源通常位于类似路径：

```text
C:\Program Files\WindowsApps\OpenAI.Codex_版本号_x64__发布者ID\app\resources\plugins\openai-bundled
```

版本号会变化，而且 WindowsApps 有访问限制。只有在确认当前安装包资源完整、做好备份并清楚目标目录后，才把完整 `openai-bundled` 结构复制到稳定的 `marketplace-source`。不要混用旧版本与新版本的插件文件，也不要只复制个别 EXE。

### 4. Windows 独有的关联项

以下问题只属于 Windows 路径，不适用于 macOS：

- `C:\Program Files\WindowsApps` 安装路径和权限；
- `extension-host.exe`；
- Chrome Native Messaging Host 注册表；
- `codex://` 的 AppX/DelegateExecute 注册；
- Electron 错误仍指向已经卸载的旧 WindowsApps 版本。

如果点击插件设置时提示找不到旧版 Electron 应用，应另外检查 `codex://` 协议注册和 AppX 安装状态。它与“插件条目搜不到”可能同时出现，但不是同一个层面。

## 七、macOS：可以通用的部分与不能照搬的部分

### 可以通用

如果 macOS 也出现更新后官方 Computer Use 插件条目消失，可以尝试：

1. 完全退出 ChatGPT/Codex；
2. 在“活动监视器”中结束残留的相关辅助进程；
3. 备份：

   ```text
   ~/.codex/config.toml
   ~/.codex/plugins/cache/openai-bundled
   ```

4. 如果 `browser`、`chrome` 或 `computer-use` 缓存目录明显残缺或存在多个冲突版本，先改名为 `.bak`；
5. 重新打开应用，让桌面端重建/重新下载；
6. 再进入 Plugins 搜索 `Computer Use`。

这是“重建用户目录下插件缓存”的通用思路。OpenAI 官方目前没有说明该缓存损坏是 macOS 的已知问题，因此只能在本地证据指向缓存异常时使用，不能把它当作 macOS 的固定必做步骤。

### 不能照搬 Windows 方法

macOS 没有以下机制：

- WindowsApps；
- Windows 注册表；
- AppX DelegateExecute；
- Windows 版 `extension-host.exe`；
- Windows x64 helper 路径。

macOS 使用 `.app` 应用包、不同格式的辅助程序、代码签名以及系统隐私权限。直接把 Windows 的 EXE、注册表命令或 WindowsApps 路径套过去没有意义，也可能破坏安装。

如果缓存重建后仍搜不到，macOS 更稳妥的下一步是更新或重装官方桌面端，然后重新登录并检查插件页，而不是手动从 `.app` 包里拼接内部插件文件。

## 八、macOS 权限什么时候才相关

只有当 Computer Use 已经出现并安装，但无法看到或操作应用时，才重点检查：

```text
系统设置 → 隐私与安全性 → 屏幕录制
系统设置 → 隐私与安全性 → 辅助功能
```

官方说明：

- 屏幕录制权限用于让 ChatGPT 看见目标应用；
- 辅助功能权限用于点击、输入和导航。

这两项权限通常不会让 Computer Use 从插件搜索结果中消失，所以不要在“插件搜不到”阶段把权限当成首要原因。

## 九、参考案例 A：bundled marketplace 完全没有部署

本机 Windows 电脑已经按照本文的 marketplace 高级修复方法恢复，结果不是推测，而是实际验证成功。

### 1. 修复前的精确检查结果

修复前逐项检查得到：

- 已安装的 Codex 桌面端版本为 `26.818.2441.0`；
- 该版本的 WindowsApps 安装包内确实包含完整的 `browser`、`chrome`、`computer-use` 插件和 `.agents\plugins\marketplace.json`；
- 但用户目录中的以下目录当时整个不存在：

  ```text
  C:\Users\chikwan\.codex\plugins\cache\openai-bundled
  ```

- `config.toml` 中没有 `[marketplaces.openai-bundled]`；
- Chrome Native Messaging Host 文件不存在；
- 当时没有正在运行的 `extension-host.exe`。

插件搜索仍能显示 Vera 和 TRIGGERcmd 等远程插件。这组现象说明远程插件目录至少能够返回结果，而本机随 Codex 安装包提供的 `openai-bundled` marketplace 没有正确部署或挂载。它比“账号或地区完全不支持 Computer Use”更符合本机证据。

### 2. 为什么常见的删除缓存办法不适用

常见方案是删除：

```text
%USERPROFILE%\.codex\plugins\cache\openai-bundled\browser
%USERPROFILE%\.codex\plugins\cache\openai-bundled\chrome
```

然后结束 `extension-host.exe` 并重启 Codex。

这个方案适用于“缓存已经生成，但内容损坏、版本冲突或文件被占用”的情况。本机当时的 `openai-bundled` 根目录都不存在，也没有 `extension-host.exe`，因此没有目录可删、没有相关进程可结束。继续套用删除法不能补回缺失的 bundled marketplace。

本机需要解决的不是“清理旧缓存”，而是“部署并挂载当前版本的完整 bundled marketplace”。

当时的表现是：

- 插件商店搜索 `Computer Use` 时没有 OpenAI 官方条目；
- Chrome/Computer Use bundled 插件状态异常；
- 用户目录下缺少可正常使用的完整 `openai-bundled` marketplace。

### 3. 本机实际采用的正确修复过程

1. 备份 `config.toml`、全局状态、`.env` 和 `codex://` 协议相关状态；
2. 从本机当前 `26.818.2441.0` 安装包的 WindowsApps 资源中取得完整 `openai-bundled`，没有使用文章中的旧版 `26.527.31326` 插件；
3. 将完整结构复制到：

   ```text
   C:\Users\chikwan\.codex\plugins\cache\openai-bundled\marketplace-source
   ```

4. 确认没有遗漏：

   ```text
   .agents\plugins\marketplace.json
   plugins\browser
   plugins\chrome
   plugins\computer-use
   ```

5. 在 `config.toml` 中让 `openai-bundled` 指向这个稳定的本地 marketplace；
6. 完全退出并重新启动桌面端，使 bundled 插件重新安装和注册；
7. 重启后让 Codex 生成 Chrome Native Messaging Host，再从 `Plugins → Computer Use` 安装并启用 `computer-use@openai-bundled`；
8. 只有点击插件或深链时出现 `codex://`/Electron 仍指向旧 WindowsApps 版本的错误，才继续修复协议注册表。没有该错误时不应修改注册表。

这里必须使用当前已安装 Codex 版本自带的完整资源。不能机械照抄旧文章中的版本号，也不能把旧版 `browser`、`chrome` 或 `computer-use` 与新版 marketplace 清单混用。

阶段性验证结果：

- Computer Use 技能文件已经存在；
- 会话能够识别 `computer-use:computer-use` 技能；
- 这只证明“市场可发现、技能可加载”这一层已经恢复，不足以证明 Computer Use 原生后台已经可用。

后续检查发现，只有技能而没有后台服务器、设置中没有“任意应用（Any App）”时，仍不能算完整修复。完整成功标准见后文“插件可见但设置中没有 Any App”的章节。

官方正常安装流程仍然是：

```text
Plugins → Computer Use → Install plugin/Enable
```

上面的手动复制和本地 marketplace 配置属于官方插件入口因本地 bundled marketplace 未部署而消失时的非官方修复手段，不应作为每台电脑的默认安装方式。

这也证明本机故障根因是本地插件 marketplace/缓存状态，而不是代理端口。为排查网络而临时创建的 `.env` 后来已经移除，保留原来的 `config.toml` 系统代理方案。

备份仍保存在：

```text
C:\Users\chikwan\.codex\repair-backups\computer-use-20260821-014254
```

该案例只证明这台 Windows 电脑的具体根因和修复有效。其他电脑仍应先检查地区、账号、应用版本与日志，不应在没有证据时直接覆盖插件目录。

## 十、如何判断问题是否已经修好

修复后至少满足：

1. 插件页面搜索 `Computer Use` 能看到 OpenAI 官方条目；
2. 可以选择 `Install plugin` 或 `Enable`；
3. 插件详情中同时出现 Computer Use server/MCP 和 skill 开关；
4. 重启后插件仍存在，不会再次消失；
5. 本地 `openai-bundled/computer-use` 目录包含完整版本和技能文件；
6. 日志不再持续出现 marketplace manifest 或 helper path 错误。

本机修复完成后，Computer Use 技能路径重新存在，并能被当前 Codex 会话识别，说明“搜不到插件”的本地 bundled marketplace 问题已经解除。

## 十一、参考案例 B：客户端更新后再次丢失（2026-08-25）

本机更新 Codex 桌面端后，Computer Use 再次从当前会话中消失。此次不是 `openai-bundled` 根目录不存在，而是更新后的客户端与之前固定的本地 marketplace 版本不一致。

### 本次检查结果

- 当前 Codex 客户端：`26.818.8289.0`；
- `config.toml` 仍将 `openai-bundled` 指向：

  ```text
  C:\Users\chikwan\.codex\plugins\cache\openai-bundled\marketplace-source
  ```

- 该本地 marketplace 仍提供 `computer-use 26.818.21641`；
- 更新后的客户端安装包内提供 `computer-use 26.818.61809`；
- 旧插件缓存中也只有 `26.818.21641`。

这说明手工固定的本地 marketplace 路径可能阻止插件随客户端自动更新：客户端升级了，但插件源和已安装缓存仍停留在旧版本。它不是代理问题，也不是简单删除 `browser`/`chrome` 可以解决的情况。

### 本次修复

1. 备份更新前的 marketplace 和配置到：

   ```text
   C:\Users\chikwan\.codex\repair-backups\computer-use-update-20260825-001454
   ```

2. 从当前客户端安装包的以下位置取得完整 `openai-bundled`：

   ```text
   C:\Program Files\WindowsApps\OpenAI.Codex_26.818.8289.0_x64__2p2nqsd0c76g0\app\resources\plugins\openai-bundled
   ```

3. 重新建立 `marketplace-source`，并逐项验证：

   - `.agents\plugins\marketplace.json` 存在；
   - 文件数和总字节数与当前客户端安装包一致；
   - `computer-use` 清单版本为 `26.818.61809`。

4. 保留旧缓存 `26.818.21641` 不动，同时新增当前版本缓存：

   ```text
   C:\Users\chikwan\.codex\plugins\cache\openai-bundled\computer-use\26.818.61809
   ```

5. 完全退出并重新打开 Codex，再到插件页面确认 Computer Use 出现且可以启用。

### 第二阶段：文件已匹配但插件页仍搜不到

完成版本同步并重启后，插件页仍然搜不到 Computer Use。继续检查得到：

- 客户端进程确实已在修复后重新启动，不是单纯忘记重启；
- 安装包与本地 `marketplace-source` 的 `marketplace.json`、`computer-use`、`browser`、`chrome` 清单 SHA-256 完全一致；
- 安装包和本地 marketplace 中的 Computer Use 版本一致；
- `config.toml` 仍把 `computer-use@openai-bundled` 标记为已安装并启用；
- 已安装缓存中同时残留旧版和新版目录；
- Codex 功能开关包含 `ComputerUse`，自动安装也没有被禁用；
- 插件日志没有显示 marketplace 文件损坏，但当前会话没有加载 Computer Use 技能。

这类现象说明问题已从“版本失配”变成“配置显示已安装，但实际没有加载”的安装状态不一致。继续覆盖相同文件通常没有意义，应先清除“已安装”状态，再判断当前客户端能否自行部署 bundled marketplace。

参考处理过程：

1. 先备份 `config.toml`、全局状态、`.bundle-id` 和整个 `openai-bundled` 缓存；
2. 将已安装的 Computer Use 缓存移动到备份，不直接删除；
3. 从 `config.toml` 移除手工添加的 `[plugins."computer-use@openai-bundled"]`；
4. 暂时移除 `[marketplaces.openai-bundled]`，用于验证当前客户端能否自行部署 bundled marketplace；
5. 将用户目录下的 `openai-bundled` 整体移动到备份；
6. 完全结束桌面端、`codex.exe` 和可能残留的 `extension-host.exe`，再重新启动；
7. 让当前客户端尝试从自己安装包内的 bundled 资源自动部署和索引；
8. 重启后检查 `openai-bundled` 是否重新生成；如果没有生成，说明该版本不能依赖自动部署，继续执行第三阶段。

### 第三阶段：确认当前版本没有自动部署 bundled marketplace

完成第二阶段并自动重启后，检查结果为：

```text
bundled_root_regenerated = false
marketplace_manifest_regenerated = false
computer_use_manifest_regenerated = false
installed_computer_use_cache_regenerated = false
marketplace_config_added_by_client = false
plugin_config_added_by_client = false
```

这组结果推翻了“只要移除手工配置，当前客户端就会自动修复”的假设。至少在该参考版本和该 Windows 环境中，完全移除本地 marketplace 后，客户端没有重新部署 `openai-bundled`，插件搜索仍然只有远程市场内容。

正确的下一步是把“市场可发现”和“插件已安装”分开处理：

1. 恢复与当前客户端版本完全一致的完整 `marketplace-source`；
2. 在 `config.toml` 中只添加 marketplace：

   ```toml
   [marketplaces.openai-bundled]
   source_type = "local"
   source = '\\?\C:\Users\用户名\.codex\plugins\cache\openai-bundled\marketplace-source'
   ```

3. 此时不要手工添加下面这段：

   ```toml
   [plugins."computer-use@openai-bundled"]
   enabled = true
   ```

4. 完全重启 Codex 后进入 Plugins 搜索。只有用户在界面中真正完成安装，或安装流程已经实际成功后，才应出现插件启用项；
5. 如果提前手工写入 `enabled = true`，插件页可能把它视为已安装而从搜索结果中隐藏，但插件缓存、技能或服务实际上又没有完成安装，从而形成“搜不到也用不了”的假安装状态。

另外，直接从 WindowsApps 复制时可能遇到“无法加密指定的文件”。只要出现该错误，就不能把生成的目录当作完整副本，必须比较文件数、总字节数和关键文件哈希；残缺副本应移动到备份，不能继续用于插件市场。

该参考电脑的可恢复备份保存在：

```text
C:\Users\chikwan\.codex\repair-backups\computer-use-reinstall-20260825-002944
```

备份路径和版本号只用于说明本次实际操作，其他电脑应按自己的用户名、客户端版本和时间生成备份目录。

## 十二、如何尽量做到更新后不再重复修复

### 推荐的长期状态

应优先区分两种客户端行为。

如果完全退出重启后，客户端能够自行生成 `openai-bundled`，就不要长期手工固定下面两类配置：

```toml
[marketplaces.openai-bundled]
source_type = "local"
source = "某个用户目录下的固定副本"

[plugins."computer-use@openai-bundled"]
enabled = true
```

第一段会把 bundled marketplace 固定到一个不会随客户端自动更新的副本；第二段可能在缓存实际缺失时仍让配置显示“已安装”。

自动部署正常时，更耐更新的状态是：

- 客户端安装包自带 `openai-bundled`；
- bundled 自动安装没有被禁用；
- 用户目录中的插件缓存由当前客户端自行生成；
- `config.toml` 不手工固定安装包版本或旧 marketplace 路径；
- 更新后完整退出并重开一次应用，让新版本完成迁移。

如果已经通过实际重启验证客户端不会自动生成 `openai-bundled`，则不能机械套用上面的状态。此时可保留 `[marketplaces.openai-bundled]` 作为本地市场挂载，但不要在安装流程完成前手工添加 `[plugins."computer-use@openai-bundled"]`。

### 能否保证以后永远不再出现

不能做绝对保证。原因有三类：

1. 某个新版本可能自身存在 bundled 插件部署或迁移回归；
2. 安全软件、ACL、磁盘清理工具可能阻止缓存写入或隔离辅助程序；
3. 企业策略、地区/账号可用性或管理员配置可能改变插件可见性。

因此，不能承诺“一次修改以后所有版本永久正常”。如果客户端能够自动部署，恢复自动管理通常可以降低复发概率；如果客户端不能自动部署，只能保留本地市场挂载，并在大版本更新后检查它是否仍与安装包一致。

若希望在客户端自动部署缺失的情况下做到无人值守，需要另外建立一个受控的自愈脚本或 Windows 计划任务：检测当前安装版本、校验 bundled 清单、备份旧副本、同步新版本并重启插件宿主。这属于额外的系统自动化，不是 Codex 官方默认机制，也不应在未验证脚本和备份策略前直接部署。

### 未来更新后的最小检查

更新后先完全重启并检查：

```text
Plugins 中是否出现 Computer Use
→ config.toml 是否指向旧 marketplace
→ openai-bundled 是否由客户端自动生成，或是否仍需本地挂载
→ 安装包与缓存中的 Computer Use 版本是否一致
→ 日志是否出现权限、manifest、安装或 helper 错误
```

只有自动部署失败时，才使用“从当前安装包恢复完整 marketplace”的高级修复。若该版本经实际验证不会自动部署，可以保留 marketplace 挂载；但插件的安装/启用状态应由真实安装流程生成，不能只靠手工写 `enabled = true`。

### 如果仍使用固定路径兜底

只要 `config.toml` 把 marketplace 固定在本地 `marketplace-source`，每次大版本客户端更新后都应检查 Computer Use 插件版本是否与新安装包一致。若客户端更新后 Computer Use 消失，优先比较这两个文件中的 `version`：

```text
%USERPROFILE%\.codex\plugins\cache\openai-bundled\marketplace-source\plugins\computer-use\.codex-plugin\plugin.json
C:\Program Files\WindowsApps\OpenAI.Codex_当前版本_x64__2p2nqsd0c76g0\app\resources\plugins\openai-bundled\plugins\computer-use\.codex-plugin\plugin.json
```

版本不一致时，应使用当前客户端自带的完整 `openai-bundled` 覆盖重建本地 marketplace，并补齐对应版本的 `computer-use` 缓存；不要把旧版插件文件继续带到新版客户端中。

## 十三、保留市场未挂载时的兼容性绕过方案

下面的方法只适用于已经确认以下条件同时成立的情况：

- 当前安装包内确实存在完整 `openai-bundled`；
- 本地清单和插件源完整，Computer Use 版本与客户端一致；
- `codex plugin marketplace list` 不显示 `openai-bundled`；
- 手工执行 `codex plugin marketplace add <openai-bundled路径>` 返回“`openai-bundled` is reserved”；
- 执行 `codex plugin add computer-use@openai-bundled` 又返回“plugin not found”；
- 插件页和日志只显示/请求远程市场。

这组结果意味着：新版核心把 `openai-bundled` 设为保留市场，不允许用户注册；但桌面端没有把保留市场交给核心。手工写同名 marketplace 会被忽略，属于客户端内部挂载状态异常，而不是代理、账号或插件文件问题。

### 绕过原理

从当前客户端自带的 bundled 资源创建一个只供本机使用的副本，把 marketplace 名称改为非保留名称，例如：

```text
openai-bundled-local
```

然后用当前 Codex 核心正式注册这个本地市场，再通过插件命令安装 Computer Use。这样绕过的是“保留市场名称没有挂载”的问题，不是破解插件本身。

### 操作要点

1. 所有文件必须来自本机当前安装的 Codex 客户端，不使用网上下载的旧插件包；
2. 复制完整 marketplace 到独立目录，例如：

   ```text
   %USERPROFILE%\.codex\plugins\cache\openai-bundled-local\marketplace-source
   ```

3. 修改副本中的：

   ```text
   .agents\plugins\marketplace.json
   ```

   将：

   ```json
   "name": "openai-bundled"
   ```

   改为：

   ```json
   "name": "openai-bundled-local"
   ```

4. 用当前客户端核心注册并检查：

   ```powershell
   codex plugin marketplace add "$env:USERPROFILE\.codex\plugins\cache\openai-bundled-local\marketplace-source" --json
   codex plugin marketplace list
   codex plugin list --available --json
   ```

5. 确认列表中出现 `computer-use@openai-bundled-local` 后再安装：

   ```powershell
   codex plugin add computer-use@openai-bundled-local --json
   codex plugin list --json
   ```

6. 完全退出并重新打开桌面端，让新安装的技能和服务进入新会话。

### 验证标准

底层验证不以插件搜索页为唯一标准。至少确认：

```text
plugin marketplace list 能看到 openai-bundled-local
plugin list --json 显示 computer-use 已安装、enabled=true
安装缓存中存在当前版本目录和完整 skills
重启后的新任务能加载 Computer Use 技能/工具
Settings → Computer Use 能显示控制配置
```

### 限制和维护

- 这是针对客户端保留市场挂载异常的本地兼容方案，不是 OpenAI 官方文档中的标准安装流程；
- 客户端更新后，本地副本不会自动跟随安装包升级，需要重新比较版本和清单；
- 一旦新客户端恢复正常的 `openai-bundled` 挂载，应卸载本地兼容插件并删除兼容市场配置，回到官方市场；
- 如果直接从 WindowsApps 复制出现“无法加密指定的文件”，必须停止并校验，不能使用残缺副本；
- 不要把本机安装包中的专有插件文件上传或分发给其他人，每台电脑应从自己的当前安装包恢复。

### 参考验证结果

在参考 Windows 环境的客户端 `26.818.8289.0` 中：

```text
openai-bundled 被核心判定为 reserved，无法手工注册
computer-use@openai-bundled 被判定为 not found
改名后的 openai-bundled-local 能被核心列出
computer-use 26.818.61809 能通过 plugin add 成功安装
安装结果显示 installed=true、enabled=true
```

这足以证明该参考故障由保留市场挂载异常触发。是否属于该版本的普遍缺陷，仍需更多设备或官方发布说明才能确认；文档中只能写成“高度支持客户端回归”，不能写成已经由 OpenAI 官方确认的已知问题。

## 十四、连续多次修复未成功的复盘与经验总结

这一节解释一种容易误判的情况：文件已经复制、配置也已经写入、客户端也重启过，但插件页仍然搜不到 Computer Use。以下记录来自参考 Windows 环境；其他电脑可以复用判断方法，但不能机械照抄版本号和路径。

### 1. 为什么前几次看似正确却没有生效

| 尝试 | 当时为什么看起来合理 | 没有成功的实际原因 | 适用前提 |
| --- | --- | --- | --- |
| 删除 `browser`、`chrome`，结束 `extension-host.exe` | 常见方案用于清理更新后的旧缓存 | 当时整个 `openai-bundled` 根目录不存在，也没有相关进程，没有损坏缓存可清理 | 缓存已经生成，但内容损坏、版本冲突或文件被占用 |
| 从当前安装包同步完整 `openai-bundled` | 可以补齐缺失文件并避免混用旧版本 | 只解决了“文件是否完整”，没有解决“市场是否被客户端核心挂载” | marketplace 文件缺失、残缺或版本不一致 |
| 在 `config.toml` 手写 `[marketplaces.openai-bundled]` | 配置中已经出现市场路径 | 当前核心把 `openai-bundled` 视为保留名称，手写同名配置可能被忽略；配置存在不等于注册成功 | 客户端允许加载该名称，并且 `marketplace list` 能验证成功 |
| 移除手工配置和缓存，等待客户端自动部署 | 正常情况下客户端应从安装包恢复 bundled 资源 | 参考版本重启后没有重新生成目录、清单、插件缓存或配置 | 自动部署机制正常、只是旧缓存阻塞时 |
| 提前手写 `[plugins."computer-use@openai-bundled"] enabled = true` | 表面上可以把插件标记为启用 | 容易形成“配置显示已安装，实际缓存、技能和服务没有完成安装”的假安装状态 | 真实安装流程已经成功后用于启停，不能代替安装 |
| 反复重启 Codex | 可以刷新索引并释放旧进程 | 重启只能重新加载已经注册的市场，不能注册一个被核心忽略的保留市场 | 市场已经注册，只是宿主或索引保留旧状态 |

### 2. 为什么兼容市场能让插件重新出现，但不等于完整成功

最终方案没有继续覆盖同一个 `openai-bundled` 目录，而是把当前客户端自带的完整 marketplace 复制为独立兼容副本，并把市场名称改为：

```text
openai-bundled-local
```

随后通过当前客户端核心正式注册该市场，再明确安装：

```text
computer-use@openai-bundled-local
```

这个名称不是保留名称，因此能够依次通过以下“插件发现与技能安装”验证：

```text
plugin marketplace list 能列出 openai-bundled-local
plugin list --available 能列出 computer-use@openai-bundled-local
plugin add 返回安装成功
plugin list 显示 installed=true、enabled=true
完全重启后的新任务能加载 Computer Use 技能
```

该阶段成功的关键不是“又复制了一遍文件”，而是完成了前三个层次：

1. **资源层**：插件文件来自当前客户端版本并且结构完整；
2. **市场层**：使用非保留名称，让客户端核心真正注册并能列出市场；
3. **安装层**：通过真实安装命令生成安装状态，而不是仅在 `config.toml` 中手写 `enabled = true`。

后来进一步确认：`openai-bundled-local` 只能解决插件可见性和技能加载。Windows 客户端的 Computer Use 原生后台会识别官方 ID `computer-use@openai-bundled`，并依赖 `cua_node/node_repl.exe` 与 native pipe。若官方运行时没有部署成功，兼容市场即使显示“立即试用”，设置中仍不会出现“任意应用”，新任务也不会获得完整 Computer Use 工具。

### 3. 如何证明主要原因不是代理、账号或地区

参考环境中的证据包括：

- 插件页能显示 Vera、TRIGGERcmd 等远程插件，说明远程插件目录至少可以返回结果；
- 当前客户端安装包内确实包含完整的 Computer Use；
- `codex plugin marketplace list` 当时不显示 `openai-bundled`；
- 注册同名市场返回 `openai-bundled is reserved`；
- 安装 `computer-use@openai-bundled` 返回 `plugin not found`；
- 改为 `openai-bundled-local` 后，同一批当前版本文件可以被列出、安装并在重启后加载。

因此，这台参考电脑的主要故障高度支持“客户端没有把保留的 bundled marketplace 正确交给插件核心”，而不是代理端口、账号或地区限制。这个结论来自本地命令和最终对照结果；OpenAI 官方尚未把它确认成该版本的普遍已知缺陷，所以其他电脑仍要先检查自己的账号、地区、管理员策略和网络日志。

### 4. 可复用的排障经验

以后遇到“插件搜不到”，应按层次验证，不要把“目录存在”直接等同于“插件可用”：

```text
安装包里是否有插件
→ 本地 marketplace 是否完整
→ marketplace list 是否真的列出市场
→ available list 是否能发现插件
→ 插件是否通过真实安装流程安装
→ 完全重启后新任务是否加载技能和服务
```

每一步只解决一个层次的问题：

- 删除缓存解决旧文件或占用问题；
- 同步安装包解决文件缺失和版本失配；
- 注册 marketplace 解决市场可发现性；
- `plugin add` 解决真实安装状态；
- 重启解决宿主重新加载，不能代替前四步。

只要某一步没有通过验证，就不要继续用重启、重复复制或手写 `enabled = true` 掩盖它。

### 5. 当前方案是否永久

`openai-bundled-local` 注册信息和插件安装状态会保存在用户配置与插件缓存中，因此普通关闭应用、重新打开以及重启电脑后通常仍然有效。但它不是跨所有未来版本的永久修复：

- 本地兼容市场是当前客户端资源的版本快照，不会自动跟随客户端升级；
- 新客户端可能修复官方 `openai-bundled` 挂载，此时应回到官方市场；
- 新客户端也可能改变插件版本、接口、目录结构或迁移逻辑，使旧兼容副本过期；
- 更新不一定每次都会导致失效，只有版本或挂载状态发生不兼容时才需要重新同步。

未来更新后应先检查官方 `openai-bundled` 是否恢复正常。若已恢复，优先卸载本地兼容插件并移除兼容市场，使用官方流程；若仍未恢复，再从新客户端自己的安装包重新生成 `openai-bundled-local`，不能长期沿用旧版本插件文件。

### 6. “立即试用”不代表限时试用

插件详情页顶部显示的“立即试用”对应英文 `Try now`，含义是“立即启动一次 Computer Use 任务”，不是订阅试用、限时授权或尚未安装。判断是否已经安装，应看技能/服务开关、插件版本、安装列表以及新任务能否加载 Computer Use，而不是根据按钮文案判断。

“立即试用”只说明插件页面可以启动示例任务，不证明原生后台已经加载。详情页显示版本、技能开关开启，也只能证明插件/技能层已安装；还必须检查后台服务器、设置中的“任意应用”和新任务中的实际工具。

## 十五、插件可见但设置中没有 Any App：Windows 原生运行时部署失败

### 1. 典型症状

以下组合表示问题已经不是“搜不到插件”，而是“插件外壳存在、后台没有加载”：

- 插件详情页能看到 Computer Use，但来源写着 `OpenAI Bundled (Local Compatibility)`；
- 插件详情只有 `Skills 1`，没有 Computer Use server/MCP；
- `设置 → 电脑操控` 只有 Chrome、Excel 等独立集成，没有“任意应用（Any App）”；
- 顶部显示“立即试用”，但新任务实际没有 Computer Use 工具；
- 反复在插件页卸载、安装没有变化。

Chrome 和 Excel 是单独应用集成，不等于完整 Computer Use。“任意应用”缺失通常意味着 Windows 原生管道没有就绪。

### 2. 日志中的直接证据

在 Windows 客户端日志中重点搜索：

```text
BundledPluginsMarketplace
bundled_plugins_marketplace_resolve_failed
bundled_executable_relocation_failed
computer-use-native-pipe
computer_use_native_pipe_thread_config_skipped
browser_use_setup_failed
node-repl-missing
```

参考环境中同时出现两类错误：

```text
copyfile ...\resources\plugins\openai-bundled\plugins\sites\.app.json
    -> ...\.codex\.tmp\bundled-marketplaces\openai-bundled.staging-*\plugins\sites\.app.json

copyfile ...\resources\cua_node\bin\corepack
    -> ...\AppData\Local\OpenAI\Codex\runtimes\cua_node\.staging-<fingerprint>-*\bin\corepack
```

随后日志明确记录：

```text
computer-use native pipe helper paths unavailable
computer_use_native_pipe_thread_config_skipped reason=not-ready
browser_use_setup_failed backend=node_repl reason=node-repl-missing
```

这比“可能是代理、账号或地区”更直接：客户端安装包内资源存在，但从 WindowsApps 复制到用户运行时缓存失败。

### 3. Windows EFS/AppX 复制故障

Microsoft Store/AppX 安装包中的资源可能带有 `Encrypted`/Application Protected 属性。Electron/Node 的普通 `copyfile` 无法把某些文件复制到用户目录，于是出现“指定文件无法加密”或 `UNKNOWN copyfile`。该故障可能同时破坏两层：

1. 官方 `openai-bundled` marketplace 没有完成暂存和切换；
2. `cua_node` 没有迁移成功，`node_repl.exe` 和 native pipe helper 路径不可用。

只修复 marketplace 会让插件重新出现，但仍没有 Any App；只修复运行时而没有以官方市场 ID 安装插件，也不能完整加载后台。

### 4. 通用修复原则

以下属于高级修复，应先备份并以当前安装版本的日志和路径为准：

1. 完全退出桌面端，确认当前 AppX 安装版本和安装目录；
2. 从当前版本安装包复制完整 `openai-bundled`，不能混用旧客户端资源；
3. 将完整官方市场放到客户端运行时市场目录，例如：

   ```text
   %USERPROFILE%\.codex\.tmp\bundled-marketplaces\openai-bundled
   ```

4. 比较源和目标的文件数、总字节数及关键清单 SHA-256；
5. 清除目标副本继承的 EFS 属性，确认目标文件和目录均不再是 `Encrypted`；
6. 根据当前启动日志中的 `<fingerprint>`，将完整 `cua_node` 部署到：

   ```text
   %LOCALAPPDATA%\OpenAI\Codex\runtimes\cua_node\<fingerprint>
   ```

   `<fingerprint>` 由当前客户端计算，不能从其他电脑或旧版本照抄；
7. 校验 `manifest.json`、`bin\node.exe`、`bin\node_repl.exe`、文件总数、总字节数和关键哈希，并确认 `node.exe --version` 可以运行；
8. 以精确市场名 `openai-bundled` 挂载修复后的官方市场，使最终插件 ID 为：

   ```text
   computer-use@openai-bundled
   ```

9. 让客户端生成官方安装缓存后，完全退出并重新打开；必须新建任务测试，因为已经打开的旧任务不会动态获得新工具。

参考配置形态如下，实际路径应使用当前用户目录：

```toml
[marketplaces.openai-bundled]
source_type = "local"
source = '\\?\C:\Users\用户名\.codex\.tmp\bundled-marketplaces\openai-bundled'

[plugins."computer-use@openai-bundled"]
enabled = true
```

只有在官方市场文件已经完整、客户端已经实际生成官方插件缓存时才添加启用项。不能用 `enabled = true` 代替资源部署和真实安装。

### 5. 为什么 Local Compatibility 不能作为最终方案

`openai-bundled-local` 是在官方保留市场无法挂载时创建的兼容名称，可以帮助确认插件资源完整并恢复搜索/技能入口。但客户端的 Computer Use 原生后台按官方 ID 和 `node_repl` 运行时初始化，因此：

```text
computer-use@openai-bundled-local
≠ 完整的 computer-use@openai-bundled 原生后台
```

在官方版本验证成功前可以暂时保留兼容市场作为回退；确认官方 Computer Use 正常后，应备份并移除兼容插件和兼容市场配置，避免两个同名条目混淆。

### 6. 完整成功标准

必须同时满足：

1. 插件来源是官方 `openai-bundled`，不再只是 `Local Compatibility`；
2. 官方插件安装缓存存在且版本与当前客户端一致；
3. `cua_node` 当前指纹目录完整，`node_repl.exe` 可用；
4. 日志不再出现 `node-repl-missing`、native pipe `not-ready`；
5. `设置 → 电脑操控` 出现“任意应用（Any App）”；
6. 完全重启后新建任务，实际获得并成功调用 Computer Use 工具。

“插件搜得到”“技能开关开启”“显示立即试用”都不是单独的完成标准。

### 7. macOS 是否适用

“区分插件/技能层和原生后台层”“以实际工具和设置项验证”的判断方式适用于 macOS；但 WindowsApps、EFS、`cua_node` Windows 路径、`node_repl.exe` 和上述复制修复是 Windows 专属，不能照搬到 macOS。

macOS 若插件已经安装但不能控制应用，应优先检查当前客户端日志、应用包完整性、屏幕录制与辅助功能权限。只有本地证据证明 bundled 缓存损坏时，才备份后重建缓存；不要使用 Windows 的 EFS、注册表或 EXE 命令。

### 8. 修复后出现两个 Computer Use 的清理方法

当官方市场恢复后，插件页可能同时显示两个同名条目：

- `computer-use@openai-bundled`：官方版本，应保留；
- `computer-use@openai-bundled-local`：此前为解决市场不可见而创建的 `Local Compatibility` 兼容版本，应在官方后台验证成功后清理。

先确认以下条件：

```text
设置中已经出现“任意应用（Any App）”
官方 computer-use@openai-bundled 缓存存在
cua_node/node_repl.exe 已部署并实际运行
```

确认后，从 `config.toml` 移除：

```toml
[marketplaces.openai-bundled-local]
source_type = "local"
source = "兼容市场路径"

[plugins."computer-use@openai-bundled-local"]
enabled = true
```

保留：

```toml
[marketplaces.openai-bundled]
source_type = "local"
source = "修复后的官方市场路径"

[plugins."computer-use@openai-bundled"]
enabled = true
```

完全退出并重新打开桌面端后，插件页应只剩一个官方 Computer Use。确认官方版本仍可使用后，再把以下兼容缓存改名或移动到备份；不要在验证前直接永久删除：

```text
%USERPROFILE%\.codex\plugins\cache\openai-bundled-local
```

如果删除配置后当前旧任务出现 `setup refresh` 错误，这是因为任务启动时已经加载了兼容技能路径；完全重启并新建任务后再验证，不能用旧任务的工具列表判断清理结果。

### 9. 当前修复能否跨客户端更新长期保持

结论应分两种情况：

- **同一客户端版本内**：普通退出、重新打开和重启电脑后通常可以保持，因为修复后的官方 marketplace、插件缓存和 `cua_node` 指纹目录仍在用户目录中；
- **升级到新的客户端版本后**：不能保证，且在自动复制错误仍存在时有较高复发风险。

参考环境当前版本为 `26.818.8289.0`。修复后虽然“任意应用”已经出现、官方 Computer Use 已安装、`node_repl.exe` 也在运行，但同一次正常启动的日志仍反复出现：

```text
plugin_marketplace_folder_write_failed
bundled_plugins_marketplace_resolve_failed
```

失败文件仍然是安装包中的：

```text
plugins\openai-bundled\plugins\sites\.app.json
```

同时，安装包中的 `.app.json`、`cua_node\bin\corepack` 和 `node_repl.exe` 仍带 `Encrypted` 属性。这说明当前能用是因为手工修复后的缓存在兜底，不是客户端自动部署机制已经恢复。

当前修复包含三类版本相关状态：

1. marketplace 的 `.bundle-id`；
2. `computer-use` 插件版本；
3. `cua_node` 的运行时版本和由客户端计算的指纹目录。

客户端升级后，只要其中任意一项变化，旧缓存就可能不再匹配：

- bundle ID 或插件版本变化：固定的 marketplace 可能继续提供旧 Computer Use，导致条目消失、版本不兼容或假安装；
- `cua_node` 版本/指纹变化：客户端会寻找新的运行时目录，旧 `node_repl.exe` 即使仍在也不能代替新指纹，设置中的 Any App 可能再次消失；
- 更新或清理过程移除 `.codex\.tmp`：当前固定在临时目录的 marketplace 可能直接丢失；
- 如果新版本已经修复自动复制：应改回客户端自动管理，并移除手工固定的旧 marketplace，避免旧副本阻止升级。

因此，不能把当前方案描述为“以后所有版本永久有效”。更准确的表述是：

```text
对当前版本是稳定兜底；对未来版本是需要版本校验的兼容修复。
```

客户端更新后的最小检查顺序：

```text
完全退出并重开
→ 插件页只保留一个官方 Computer Use
→ 设置中仍有 Any App
→ 新任务能加载 Computer Use
→ node_repl.exe 来自当前运行时指纹目录
→ 比较安装包与缓存的 bundle ID、插件版本、cua_node 版本
→ 检查日志是否仍有 copyfile / relocation 错误
```

如果版本标识没有变化，旧修复通常可以继续使用；如果任一标识变化且自动复制仍失败，就必须从**新版本自己的安装包**重新同步和解密，不能继续沿用旧版文件。

可以进一步制作受控的“更新后自检/自愈脚本”，自动读取当前安装版本、比较三个版本标识、校验文件并在需要时重建缓存。它能减少人工操作，但仍属于非官方补偿机制，不能保证应对未来所有目录结构变化。

不建议仅为了避免复发而长期停留在旧客户端。OpenAI 官方更新说明指出，桌面端默认自动更新，旧版本不会获得独立安全补丁或长期支持。更合理的做法是更新后做上述检查，或在受管理环境中先用小范围设备验证新版本。

## 十六、推荐排查顺序

```text
确认地区/账号/工作区可用性
→ 更新桌面端并完全重启
→ 判断是否只有 Computer Use 消失，还是多个官方 bundled 插件一起消失
→ 检查日志与 openai-bundled 缓存
→ 先备份并改名重建缓存
→ Windows 有明确 manifest/helper 错误时再做 marketplace 高级修复
→ macOS 不使用 WindowsApps、注册表或 EXE 修复
→ 插件出现后，再处理各系统的控制权限
```

## 十七、参考资料

- [OpenAI：Computer Use](https://learn.chatgpt.com/docs/computer-use)
- [OpenAI：Codex Config basics](https://learn.chatgpt.com/docs/config-file/config-basic)
- [OpenAI：Codex Environment variables](https://learn.chatgpt.com/docs/config-file/environment-variables)
