---
layout:     post
title:      OpenClaw小白手册：使用本地模型在Mac上养龙虾
subtitle:   OpenClaw小白手册：使用本地模型在Mac上养龙虾
date:       2026-03-04
author:     LXY
header-img: img/home1.jpeg
catalog: true
tags:
    - 微经验
---




# 前言

OpenClaw 是一个基于本地 AI 模型的智能助手，可以让你在电脑上直接运行各种 LLM（大型语言模型），实现问答、代码辅助、写作等功能，无需依赖云端。本文将带你一步步完成 OpenClaw 的安装和本地模型配置，让你快速开始本地 AI 体验。



# 硬件要求

在安装 OpenClaw 之前，请确保你的电脑满足以下条件：

- 操作系统：macOS / Windows / Linux

- CPU：推荐四核及以上

- 内存：至少 8GB（建议 16GB 或更多）

- Python：3.9+（部分功能依赖 Python）

- 网络：首次安装需要下载模型和依赖

⚠️ 如果你计划运行大型模型（如 LLaMA 3、Qwen 等），建议使用带 GPU 的电脑以获得更好的性能。




# Mac 版安装教程



## 🦞 OpenClaw 一键安装（目前最简单方法）
✅ 推荐方式

✅ 自动安装 Node / Gateway / Agent

✅ 比 npm 手动安装稳定很多

### 1.执行官方安装脚本

直接在 Terminal 运行：

```
curl -fsSL https://openclaw.ai/install.sh | bash
```


### 2.安装过程中会发生什么？

脚本会自动：

1️⃣ 检测 macOS

2️⃣ 安装 Homebrew（如果没有）

3️⃣ 安装 Node.js

4️⃣ 安装 OpenClaw

5️⃣ 初始化 Gateway

6️⃣ 创建默认 Agent



**如果你成功了，你将会看到以下日志：**

```

  🦞 OpenClaw Installer
  I'm the reason your shell history looks like a hacker-movie montage.

✓ Detected: macos

Install plan
OS: macos
Install method: npm
Requested version: latest
· Existing OpenClaw installation detected, upgrading

[1/3] Preparing environment
✓ Homebrew already installed
✓ Node.js v25.7.0 found
· Active Node.js: v25.7.0 (/opt/homebrew/bin/node)
· Active npm: 11.10.1 (/opt/homebrew/bin/npm)

[2/3] Installing OpenClaw
✓ Git already installed
· Installing OpenClaw v2026.3.2
✓ OpenClaw npm package installed
✓ OpenClaw installed

[3/3] Finalizing setup
· Refreshing loaded gateway service
✓ Gateway service metadata refreshed
✓ Gateway service restarted
✗ Probing gateway service failed — re-run with --verbose for details
error: unknown option '--probe'
(Did you mean one of --no-probe, --profile?)
· Running doctor to migrate settings
✓ Doctor complete

🦞 OpenClaw installed successfully (2026.3.2)!
Reborn from the boiling waters of npm. Stronger now.

· Upgrade complete
· Running openclaw doctor

🦞 OpenClaw 2026.3.2 (85377a2) — Half butler, half debugger, full crustacean.

▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄
██░▄▄▄░██░▄▄░██░▄▄▄██░▀██░██░▄▄▀██░████░▄▄▀██░███░██
██░███░██░▀▀░██░▄▄▄██░█░█░██░█████░████░▀▀░██░█░█░██
██░▀▀▀░██░█████░▀▀▀██░██▄░██░▀▀▄██░▀▀░█░██░██▄▀▄▀▄██
▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀
                  🦞 OPENCLAW 🦞

┌  OpenClaw doctor
│
◇  State integrity ───────────────────────────────────────────────────────────────╮
│                                                                                 │
│  - OAuth dir not present (~/.openclaw/credentials). Skipping create because no  │
│    WhatsApp/pairing channel config is active.                                   │
│                                                                                 │
├─────────────────────────────────────────────────────────────────────────────────╯
│
◇  Security ─────────────────────────────────╮
│                                            │
│  - No channel security warnings detected.  │
│  - Run: openclaw security audit --deep     │
│                                            │
├────────────────────────────────────────────╯
│
◇  Skills status ────────────╮
│                            │
│  Eligible: 4               │
│  Missing requirements: 47  │
│  Blocked by allowlist: 0   │
│                            │
├────────────────────────────╯
│
◇  Plugins ──────╮
│                │
│  Loaded: 4     │
│  Disabled: 34  │
│  Errors: 0     │
│                │
├────────────────╯
│
◇
Agents: main (default)
Heartbeat interval: 30m (main)
Session store (main): /Users/lixueyang/.openclaw/agents/main/sessions/sessions.json (1 entries)
- agent:main:main (10m ago)
Run "openclaw doctor --fix" to apply changes.
│
└  Doctor complete.

· Updating plugins
No npm-installed plugins to update.
· Gateway daemon detected; restarting
✓ Gateway restarted

🦞 OpenClaw 2026.3.2 (85377a2) — Finally, a use for that always-on Mac Mini under your desk.

Dashboard URL: http://127.0.0.1:18789/#token=840f873b418666ec37e4a4c62c71dced14f5e15b3713cc9f
Copied to clipboard.
Opened in your browser. Keep that tab to control OpenClaw.
```


## 使用Ollama安装：本地模型

### 🧩 第一步：安装

去官网下载安装：

👉 https://ollama.com

安装完成后，终端输入：

```
ollama --version
```	

能看到版本号说明安装成功。


### 🧠 第二步：下载运行模型

**这里需要找一个可以被工具调用的本地模型，作者综合对比，使用qwen2.5:7b 最为本地模型**

比如运行 qwen2.5:7b：

```
ollama run qwen2.5:7b
```

第一次会自动下载模型（几GB）。

下载完直接进入聊天模式：


```
>>> 你好
你好！很高兴见到你。我可以帮助你解答问题、提供信息或者进行交流，请告诉我你需
要什么帮助。

>>> 你是谁
我是Qwen，由阿里云开发的AI助手。我可以帮助解答问题、提供信息、进行对话等。有
什么我可以帮助你的吗？
```


就可以本地对话了 🎉





## 🦞 OpenClaw 配置

当安装完成后，执行命令：

```
openclaw onboard
```

控制台会显示如下：

```
 OpenClaw 2026.3.2 (85377a2) — I'm the reason your shell history looks like a hacker-movie montage.

▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄
██░▄▄▄░██░▄▄░██░▄▄▄██░▀██░██░▄▄▀██░████░▄▄▀██░███░██
██░███░██░▀▀░██░▄▄▄██░█░█░██░█████░████░▀▀░██░█░█░██
██░▀▀▀░██░█████░▀▀▀██░██▄░██░▀▀▄██░▀▀░█░██░██▄▀▄▀▄██
▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀
                  🦞 OPENCLAW 🦞

┌  OpenClaw onboarding
│
◇  Security ─────────────────────────────────────────────────────────────────────────────────╮
│                                                                                            │
│  Security warning — please read.                                                           │
│                                                                                            │
│  OpenClaw is a hobby project and still in beta. Expect sharp edges.                        │
│  By default, OpenClaw is a personal agent: one trusted operator boundary.                  │
│  This bot can read files and run actions if tools are enabled.                             │
│  A bad prompt can trick it into doing unsafe things.                                       │
│                                                                                            │
│  OpenClaw is not a hostile multi-tenant boundary by default.                               │
│  If multiple users can message one tool-enabled agent, they share that delegated tool      │
│  authority.                                                                                │
│                                                                                            │
│  If you’re not comfortable with security hardening and access control, don’t run           │
│  OpenClaw.                                                                                 │
│  Ask someone experienced to help before enabling tools or exposing it to the internet.     │
│                                                                                            │
│  Recommended baseline:                                                                     │
│  - Pairing/allowlists + mention gating.                                                    │
│  - Multi-user/shared inbox: split trust boundaries (separate gateway/credentials, ideally  │
│    separate OS users/hosts).                                                               │
│  - Sandbox + least-privilege tools.                                                        │
│  - Shared inboxes: isolate DM sessions (`session.dmScope: per-channel-peer`) and keep      │
│    tool access minimal.                                                                    │
│  - Keep secrets out of the agent’s reachable filesystem.                                   │
│  - Use the strongest available model for any bot with tools or untrusted inboxes.          │
│                                                                                            │
│  Run regularly:                                                                            │
│  openclaw security audit --deep                                                            │
│  openclaw security audit --fix                                                             │
│                                                                                            │
│  Must read: https://docs.openclaw.ai/gateway/security                                      │
│                                                                                            │
├────────────────────────────────────────────────────────────────────────────────────────────╯
│

```

**第一个问题是询问你是否是个人模式，选择Yes**

```
◆  I understand this is personal-by-default and shared/multi-user use requires lock-down. Continue?
│  ●  Yes / ● No

```

**运行模式选择QuickStart，按回车**

```
◆  Onboarding mode
│  ● QuickStart (Configure details later via openclaw configure.)
│  ○ Manual
```


**模型选择**

```
◆  Model/auth provider
│  ...
│  ○ Volcano Engine
│  ○ BytePlus
│  ○ OpenRouter
│  ○ Kilo Gateway
│  ○ Qwen
│  ○ Z.AI
│  ○ Qianfan
│  ○ Copilot
│  ○ Vercel AI Gateway
│  ○ OpenCode Zen
│  ○ Xiaomi
│  ○ Synthetic
│  ○ Together AI
│  ○ Hugging Face
│  ○ Venice AI
│  ○ LiteLLM
│  ○ Cloudflare AI Gateway
│  ● Custom Provider (Any OpenAI or Anthropic compatible endpoint)
│  ○ Skip for now
```

因为要使用本地模型，因此选择Custom Provider 按回车


**配置Base URL**

```
◇  API Base URL
│  http://127.0.0.1:11434/v1
```
🎯 Ollama 本身就是服务端
“关联”的本质就是让 OpenClaw 连接：
```
http://localhost:11434
```

**其他配置及模型关联**

```
◇  How do you want to provide this API key?
│  Paste API key now
│
◇  API Key (leave blank if not required)
│  ollama
│
◇  Endpoint compatibility
│  OpenAI-compatible
│
◇  Model ID
│  qwen2.5:7b
│
◇  Verification successful.
│
◆  Endpoint ID
│  custom-127-0-0-1-11434█
```

- API Key 随便写一个字符串就可以，这里输入ollama
-  Model ID需要输入本地模型名称，这里输入qwen2.5:7b
- 信息输入完成后，会自动进行模型可用性校验，看到Verification successful就表示成功了

**选择使用渠道**

```
◆  Select channel (QuickStart)
│  ● Telegram (Bot API) (recommended · newcomer-friendly)
│  ○ WhatsApp (QR link)
│  ○ Discord (Bot API)
│  ○ IRC (Server + Nick)
│  ○ Google Chat (Chat API)
│  ○ Slack (Socket Mode)
│  ○ Signal (signal-cli)
│  ○ iMessage (imsg)
│  ○ Feishu/Lark (飞书)
│  ○ Nostr (NIP-04 DMs)
│  ○ Microsoft Teams (Bot Framework)
│  ○ Mattermost (plugin)
│  ○ Nextcloud Talk (self-hosted)
│  ○ Matrix (plugin)
│  ○ BlueBubbles (macOS app)
│  ○ LINE (Messaging API)
│  ○ Zalo (Bot API)
│  ○ Zalo (Personal Account)
│  ○ Synology Chat (Webhook)
│  ○ Tlon (Urbit)
│  ○ Skip for now
```

这里可以选择关联飞书、WhatsApp等聊天App，这里选择跳过；关联方式这里就不做介绍了，感兴趣的同学可以在官网查找配置方式


**最后配置选择**

```
◇  Configure skills now? (recommended)
│  No
│
◇  Hooks ──────────────────────────────────────────────────────────────────╮
│                                                                          │
│  Hooks let you automate actions when agent commands are issued.          │
│  Example: Save session context to memory when you issue /new or /reset.  │
│                                                                          │
│  Learn more: https://docs.openclaw.ai/automation/hooks                   │
│                                                                          │
├──────────────────────────────────────────────────────────────────────────╯
│
◆  Enable hooks?
│  ◻ Skip for now
│  ◻ 🚀 boot-md
│  ◻ 📎 bootstrap-extra-files
│  ◻ 📝 command-logger
│  ◼ 💾  session-memory (Save session context to memory when /new or /reset command is issued)

 Gateway service already installed
│  Reinstall
│
```
Enable hooks选择💾  session-memory，是为了让回话记录在本地，使会话连贯，最后的选项选择重新安装Reinstall

```

Moved LaunchAgent to Trash: /Users/lixueyang/.Trash/ai.openclaw.gateway.plist
◇  Gateway service uninstalled.
│
◐  Installing Gateway service….
Installed LaunchAgent: /Users/lixueyang/Library/LaunchAgents/ai.openclaw.gateway.plist
Logs: /Users/lixueyang/.openclaw/logs/gateway.log
◇  Gateway service installed.
│
◇
Agents: main (default)
Heartbeat interval: 30m (main)
Session store (main): /Users/lixueyang/.openclaw/agents/main/sessions/sessions.json (1 entries)
- agent:main:main (22m ago)
│
◇  Optional apps ────────────────────────╮
│                                        │
│  Add nodes for extra features:         │
│  - macOS app (system + notifications)  │
│  - iOS app (camera/canvas)             │
│  - Android app (camera/canvas)         │
│                                        │
├────────────────────────────────────────╯
│
◇  Control UI ─────────────────────────────────────────────────────────────────────╮
│                                                                                  │
│  Web UI: http://127.0.0.1:18789/                                                 │
│  Web UI (with token):                                                            │
│  http://127.0.0.1:18789/#token=b1f855f6138d20e69f99765377b72ca31e51bbc844dd7610  │
│  Gateway WS: ws://127.0.0.1:18789                                                │
│  Gateway: reachable                                                              │
│  Docs: https://docs.openclaw.ai/web/control-ui                                   │
│                                                                                  │
├──────────────────────────────────────────────────────────────────────────────────╯
│
◇  Start TUI (best option!) ─────────────────────────────────╮
│                                                            │
│  This is the defining action that makes your agent you.    │
│  Please take your time.                                    │
│  The more you tell it, the better the experience will be.  │
│  We will send: "Wake up, my friend!"                       │
│                                                            │
├────────────────────────────────────────────────────────────╯
│
◇  Token ─────────────────────────────────────────────────────────────────────────────────╮
│                                                                                         │
│  Gateway token: shared auth for the Gateway + Control UI.                               │
│  Stored in: ~/.openclaw/openclaw.json (gateway.auth.token) or OPENCLAW_GATEWAY_TOKEN.   │
│  View token: openclaw config get gateway.auth.token                                     │
│  Generate token: openclaw doctor --generate-gateway-token                               │
│  Web UI stores a copy in this browser's localStorage (openclaw.control.settings.v1).    │
│  Open the dashboard anytime: openclaw dashboard --no-open                               │
│  If prompted: paste the token into Control UI settings (or use the tokenized dashboard  │
│  URL).                                                                                  │
│                                                                                         │
├─────────────────────────────────────────────────────────────────────────────────────────╯
│
◆  How do you want to hatch your bot?
│  ○ Hatch in TUI (recommended)
│  ● Open the Web UI
│  ○ Do this later
└

```

到这里恭喜你，你已经完成了你的小🦞，使用Open the Web UI，就可以直接在浏览器里面调戏你的小🦞了

# 相关网站

- 官网：https://openclaw.ai/
- GitHub：https://github.com/openclaw/openclaw
- 官方文档：https://docs.openclaw.ai/zh-CN/