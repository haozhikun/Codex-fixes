# Codex 插件商店搜不到 Computer Use 的排查与解决.

> 适用系统：Windows、macOS  
> 更新日期：2026-08-21

## 一、本文处理的具体问题

本文针对的是：

- 打开 Codex/ChatGPT 桌面端的“插件”页面；
- 搜索 `Computer Use`；
- 结果中没有官方 Computer Use 插件，只出现其他第三方插件，或者显示“不可用”。

这与“已经安装 Computer Use，但无法点击、看不到画面或不能控制应用”不是同一个问题。

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

不要照抄用户名；不要在不确认目录完整时添加这段配置。

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

## 九、本机已验证的修复记录

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

验证结果：

- 当前 Computer Use 技能文件已经存在；
- 当前会话已经能识别 `computer-use:computer-use` 技能；
- 本机“搜不到/不能使用 Computer Use”的问题已经解决。

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

## 十一、推荐排查顺序

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

## 十二、参考资料

- [OpenAI：Computer Use](https://learn.chatgpt.com/docs/computer-use)
- [OpenAI：Codex Config basics](https://learn.chatgpt.com/docs/config-file/config-basic)
- [OpenAI：Codex Environment variables](https://learn.chatgpt.com/docs/config-file/environment-variables)
