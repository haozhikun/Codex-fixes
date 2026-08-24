# Codex 使用代理时反复 Reconnecting 的排查与解决.

> 适用系统：Windows、macOS  
> 更新日期：2026-08-21

## 一、典型现象

发送任务后，Codex 长时间显示：

```text
Reconnecting 1/5
Reconnecting 2/5
...
Reconnecting 5/5
```

随后才进入 `Thinking` 或开始输出。

这里的 `Reconnecting` 通常表示流式网络连接中断、超时或重新建立，并不代表模型正在重新思考。代理、防火墙、DNS、TLS 检查、WebSocket 支持和节点质量都可能导致同类现象，因此不能仅凭 `Reconnecting 5/5` 就断定一定是某一个原因。

## 二、代理场景下的常见根因

当电脑需要通过代理访问 OpenAI 服务时，常见问题是：

1. 代理软件已经启动，但没有开启“系统代理”，Codex 仍尝试直连。
2. Codex 没有正确读取操作系统的代理配置。
3. HTTPS 可以通过代理，但流式连接或 WebSocket 被代理、防火墙或公司网关中断。
4. `.env` 中写死的端口已经失效，例如仍指向 `127.0.0.1:7890`，但代理软件已经换端口。
5. 同时存在系统代理、TUN、`.env` 和系统级环境变量，多个配置互相冲突。

“前五次一定直连，第六次才自动切换代理”不是可以普遍确认的固定机制。更准确的描述是：连接层失败后发生多次重试，代理配置不正确只是常见原因之一。

## 三、首选方案：让 Codex 跟随系统代理

本机曾通过以下设置解决代理导致的重连问题：

```toml
[features]
respect_system_proxy = true
```

### 配置文件位置

Windows：

```text
%USERPROFILE%\.codex\config.toml
```

通常对应：

```text
C:\Users\用户名\.codex\config.toml
```

macOS：

```text
~/.codex/config.toml
```

### 修改方法

**推荐将这一段放在 `config.toml` 的最开头。** 随着 Codex 使用时间变长，文件会自动增加项目路径、插件、界面等配置；把系统代理开关固定放在顶部，之后打开文件即可立刻确认是否仍启用，避免在较长的配置中反复查找。

推荐结构：

```toml
# 系统代理：用于减少代理环境下的 Reconnecting
[features]
respect_system_proxy = true

# 其余已有配置从这里继续，例如 model、desktop、plugins、projects 等
```

这里的“放在最上面”是为了维护方便，**不会提高它的配置优先级**。同一份 `config.toml` 内只要 `[features]` 区块合法且不重复，写在开头或末尾的效果相同；真正影响优先级的是命令行参数、项目配置、配置 profile、用户级配置等不同配置层。

如果文件中没有 `[features]`，添加：

```toml
[features]
respect_system_proxy = true
```

如果已经存在 `[features]`，只在该区块中增加：

```toml
respect_system_proxy = true
```

不要创建两个重复的 `[features]` 区块。

保存后必须完全退出 Codex/ChatGPT 桌面端再重新打开。仅关闭窗口可能仍保留后台进程和旧的网络配置。

### 使用这一方案的前提

代理软件必须满足至少一项：

- 已开启“系统代理”；
- 使用 TUN/VPN 模式接管系统流量；
- 系统已经正确设置 PAC/WPAD 或企业代理。

`respect_system_proxy = true` 的含义是跟随操作系统当前代理，它不会自动扫描 Clash、V2RayN 等软件正在监听哪个端口。如果代理软件只启动本地端口、却没有修改系统代理，也没有启用 TUN，Codex 仍可能无法连接。

### 版本说明

当前 OpenAI 官方配置文档说明用户级持久配置位于 `~/.codex/config.toml`，但截至本文日期，公开的配置参考中没有列出 `respect_system_proxy`。它在本机当前版本中有效，但应视为可能随版本变化的配置项。Codex 更新后如果再次出现问题，应先核对最新官方配置参考，而不是盲目叠加更多代理设置。

## 四、`config.toml` 与 `.env` 的区别

| 项目 | `config.toml` 跟随系统代理 | `.env` 明确指定代理 |
|---|---|---|
| 工作方式 | 读取操作系统当前代理 | 把代理地址和端口固定写入变量 |
| 更换代理软件 | 通常不用改 Codex，只要新软件设置系统代理 | 端口或协议改变后必须同步修改 |
| 适合场景 | 日常使用、频繁切换代理软件 | 系统代理识别失败时的兼容或诊断手段 |
| 主要风险 | 代理软件没有开启系统代理时无效 | 旧端口会让 Codex 持续连接到不存在的代理 |
| 对子进程的影响 | 通常较小 | 可能影响 Codex 启动的 Git、npm、pip 等进程 |
| 维护成本 | 低 | 高 |

OpenAI 官方说明：`config.toml` 用于持久设置，环境变量更适合当前 Shell 的覆盖、自动化机密、安装行为或诊断。官方当前公开的稳定环境变量列表没有列出 `HTTP_PROXY`、`HTTPS_PROXY`、`ALL_PROXY`，也没有把 `$CODEX_HOME/.env` 描述为稳定公共接口。因此 `.env` 方案应当作为版本相关的兼容性后备方案，而不是默认首选。

## 五、什么情况下才考虑使用 `.env`

只有满足以下条件之一时才建议测试：

1. 系统代理已经开启，但当前 Codex 版本仍持续重连。
2. 代理软件不会设置系统代理，也不能使用 TUN，只提供固定的本地代理端口。
3. 需要短时间明确指定某个代理，以确认故障究竟来自系统代理识别还是网络本身。
4. 企业网络要求使用固定上游代理，并且已确认所用 Codex 版本会读取这些变量。

不建议使用 `.env` 的情况：

- `config.toml` 配合系统代理已经正常工作；
- 经常切换 Clash、V2RayN 或其他代理软件，端口可能变化；
- 不清楚本地端口是 HTTP、SOCKS5 还是 mixed；
- 代理软件当前没有运行；
- 只是 Computer Use 插件在商店里搜不到——这通常是插件目录或可用性问题，不应先改代理文件。

## 六、`.env` 示例与注意事项

某些 Codex 桌面端构建可能读取以下文件：

Windows：

```text
%USERPROFILE%\.codex\.env
```

macOS：

```text
~/.codex/.env
```

示例：

```env
HTTP_PROXY=http://127.0.0.1:7890
HTTPS_PROXY=http://127.0.0.1:7890
ALL_PROXY=http://127.0.0.1:7890
NO_PROXY=localhost,127.0.0.1,::1
```

注意：

- `7890` 只是示例，必须使用代理软件当前真实端口。
- URL 协议必须与端口类型匹配，不能把 SOCKS5 端口写成 HTTP。
- 修改或删除 `.env` 后必须完全重启桌面端。
- 不要让 `.env` 指向一个没有程序监听的端口。
- 如果 `.env` 与系统代理指向不同节点，排查会变得困难。
- 使用前先备份；测试失败后应恢复或删除，避免留下永久隐患。

本机曾临时创建指向 `127.0.0.1:7890` 的 `.env`，但后来发现该端口已无人监听，而 `config.toml` 的系统代理方案仍然存在，因此已经删除 `.env`，恢复为只跟随系统代理的方式。

## 七、Windows 排查步骤

1. 检查代理软件是否开启“系统代理”或 TUN。
2. 打开“设置 → 网络和 Internet → 代理”，确认当前代理状态。
3. 检查 `C:\Users\用户名\.codex\config.toml` 是否只有一个 `[features]` 区块。
4. 检查是否存在过期的 `C:\Users\用户名\.codex\.env`。
5. 检查代理软件显示的 HTTP/mixed 端口是否真的在监听。
6. 完全退出 Codex/ChatGPT，包括托盘和后台进程，再重新打开。
7. 仍然重连时，换一个明确支持 HTTPS 长连接/WebSocket 的节点测试。

## 八、macOS 排查步骤

1. 检查代理软件是否启用“设置为系统代理”或 TUN/VPN 模式。
2. 打开“系统设置 → 网络 → 当前网络 → 详细信息 → 代理”，核对系统代理。
3. 检查 `~/.codex/config.toml`。
4. 检查是否存在过期的 `~/.codex/.env` 或由 Shell 启动脚本设置的代理变量。
5. 使用“活动监视器”完全退出 ChatGPT/Codex 及其辅助进程，再重新打开。
6. 仍然重连时，测试其他节点，并确认代理没有提前关闭长连接。

## 九、建议的排查顺序

```text
确认 OpenAI 服务状态
→ 确认普通网络和代理节点可用
→ 开启代理软件的系统代理或 TUN
→ 使用 config.toml 跟随系统代理
→ 完全重启 Codex
→ 排除旧 .env/系统环境变量冲突
→ 最后才用固定 .env 做诊断
```

如果删除 `.env` 后 Codex 无法联网，而 Windows/macOS 系统代理也是关闭状态，这并不说明必须恢复 `.env`；应先让当前代理软件正确设置系统代理，或者启用 TUN。

## 十、参考资料

- [OpenAI：Codex Config basics](https://learn.chatgpt.com/docs/config-file/config-basic)
- [OpenAI：Codex Configuration Reference](https://learn.chatgpt.com/docs/config-file/config-reference)
- [OpenAI：Codex Environment variables](https://learn.chatgpt.com/docs/config-file/environment-variables)
