---
name: mcp-npx-resilience
description: 修复因 npm 缓存被清理（如 360 安全卫士）导致 npx 启动的 MCP 服务器 ENOTCACHED 崩溃，并做预防配置。适用于任何安装了 Reasonix 的 Windows 电脑。
---

# MCP npx 缓存韧性修复

## 问题描述

360 安全卫士/电脑管家等清理垃圾时，会清空 Reasonix 的 MCP npm 缓存（`mcp-state/.../cache/npm/`）。
用 `npx -y <pkg>` 启动的 MCP 服务器因缓存缺失且无法回退到网络 → `ENOTCACHED`（`cache mode is 'only-if-cached' but no cached response is available`）→ 进程崩溃 EOF → Reasonix 标记该 MCP 为 `failed`。

典型报错：
```
npm error code ENOTCACHED
npm error request to https://registry.npmmirror.com/<pkg> failed: cache mode is 'only-if-cached' but no cached response is available.
```

## 修复流程（4 步）

### Step 1：诊断 — 找出使用 npx 启动的 MCP

```bash
# 列出所有插件配置，找出 command="npx" 的 MCP
grep -n -B2 -A2 'command = "npx"' "<reasonix_config>"
```

`<reasonix_config>` 按优先级检查：
1. `./reasonix.toml`（项目级，当前工作目录）
2. `~/AppData/Roaming/reasonix/config.toml`（用户级）
3. `~/AppData/Roaming/reasonix/global-workspace/reasonix.toml`（workspace 级）

对于每个 MCP，记录：`name`（插件名）和 `pkg`（args 中的包名，去掉 `@version` 后缀）。

**包名 → 命令名映射速查：**

| npm 包名 | 全局安装后的命令 |
|----------|----------------|
| `firecrawl-mcp` | `firecrawl-mcp` |
| `@modelcontextprotocol/server-github` | `mcp-server-github` |
| `@next-ai-drawio/mcp-server` | `next-ai-drawio-mcp` |
| `anycrawl-mcp` | `anycrawl-mcp` |

### Step 2：全局安装缺失的包（绕过 npx 缓存依赖）

```bash
# 对每个受影响的 MCP 包执行
npm install -g <包名> --prefer-online
```

关键参数 `--prefer-online`：强制走网络下载，避免因缓存问题安装失败。
安装后验证命令在 PATH 中：
```bash
which <命令名>   # 应返回路径，如 /c/Users/<user>/AppData/Roaming/npm/<cmd>
```

### Step 3：改配置 — 从 npx 改为直接调用

在 `reasonix.toml` / `config.toml` 中，将：

```toml
[[plugins]]
name    = "xxx"
command = "npx"
args    = ["-y", "<pkg>[@version]"]
```

改为：

```toml
[[plugins]]
name    = "xxx"
command = "<命令名>"
args    = []
```

> 如果命令不在 PATH 中，用完整路径，如：
> ```toml
> command = "C:\\Users\\<用户名>\\AppData\\Roaming\\npm\\<命令名>.cmd"
> ```

### Step 4：预防 — 将 Reasonix 目录加入 360 信任区

在 **360 安全卫士** 中操作：
1. 打开 **木马查杀 → 信任区**
2. **添加目录**：`C:\Users\<用户名>\AppData\Roaming\reasonix`
3. 确认添加

此后 360 清理垃圾时会跳过该目录，保护所有 MCP 缓存。推荐使用外层 `reasonix` 目录而非仅 `mcp-state`，同时保护其他可能被清理的运行时文件。

## 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| `npm install -g` 很慢 | 国内网络连 npm registry 慢 | 可先用 `npm config set registry https://registry.npmmirror.com` |
| 安装后 `which` 找不到命令 | 包名和命令名不同 | 查表映射，或到 `%APPDATA%\npm\` 目录下查看 .cmd 文件 |
| 改完配置 MCP 仍然 failed | Reasonix 内存缓存未刷新 | 重启 Reasonix 会话，或用 `install_source` 重新注册 |

## 反合理化表

| 想法 | 现实 |
|------|------|
|「把 registry 从 npmmirror 改回 npmjs」| 国内网络直连 npmjs 太慢/可能失败 |
|「关掉 360 就行了」| 不现实，下次还会忘或其他人用电脑时又打开 |
|「加 --prefer-online 就行」| npx 的缓存行为由调用方环境变量控制，不一定生效 |
|「只修 anycrawl 就行」| firecrawl/drawio/github 也用的 npx，迟早会炸 |
|「下次清完垃圾重新 npx 就行」| 每次都要修，不如一次到位 |

## 跨电脑移植

将此 skill 目录复制到目标电脑的 `.reasonix/skills/` 下即可：

```
.reasonix/skills/mcp-npx-resilience/
└── SKILL.md
```

首次运行 skill 时 agent 会自动执行 4 步修复流程。
