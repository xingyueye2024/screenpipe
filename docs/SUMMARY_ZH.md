# Screenpipe 仓库中文总结

> **一句话概述**：Screenpipe 是一个开源的本地优先 AI 上下文基础设施，持续捕获你在屏幕上看到的、麦克风听到的一切，并通过 API 和插件系统让 AI 智能体能够基于完整的用户上下文自主工作。

---

## 1. 产品愿景

屏幕是人类与数字世界交互的通用接口，每秒传输约 1000 万比特的信息。Screenpipe 把这些信息在本地完整捕获并结构化，让 AI 无需用户手动输入 prompt 就能理解用户正在做什么，从而实现真正的无感自动化。

**三个发展阶段**：

| 阶段 | 目标 | 说明 |
|------|------|------|
| **现在** | 桌面记忆 | 录制、回溯、提问 —— 三个动词，简单到不可或缺 |
| **下一步** | AI 智能体上下文层 | 开放 API，任何 AI agent 都可查询屏幕历史 |
| **未来** | 全传感器 | 屏幕只是第一个传感器，之后是摄像头、空间记忆等 |

**核心原则**：
- 稳定性优先于新功能
- 不做功能膨胀（每个功能必须服务于 录制/回溯/提问）
- 尊重用户机器资源（目标：<20% CPU、<3GB RAM）
- 本地优先，数据不出设备（除非用户主动开启云同步）
- 跨平台（macOS / Windows / Linux）

---

## 2. 技术栈

### 后端（Rust）

| 技术 | 用途 |
|------|------|
| Tokio | 异步运行时 |
| Axum 0.7 | HTTP/WebSocket 服务器（端口 3030） |
| SQLx + SQLite | 数据库，FTS5 全文搜索 |
| sqlite-vec | 向量嵌入搜索 |
| Whisper-rs / Qwen3-ASR | 本地语音转文字 |
| Silero VAD v5/v6 | 语音活动检测 |
| cpal | 跨平台音频采集 |
| ScreenCaptureKit / DXGI / xcap | 屏幕捕获（分平台） |
| Apple Vision / Windows OCR / Tesseract | OCR 文字识别 |
| OnnxRuntime | 说话人分离等 ML 推理 |
| Tauri v2 | 桌面应用框架 |

### 前端（TypeScript/JavaScript）

| 技术 | 用途 |
|------|------|
| Next.js 15 | Tauri 应用前端（App Router） |
| React 18 + Tailwind CSS | UI 组件 |
| shadcn/ui (Radix) | 组件库 |
| Zustand | 状态管理 |
| Vercel AI SDK | AI 流式响应 |
| Bun | 包管理器和运行时 |

---

## 3. 项目目录结构

```
screenpipe/
├── apps/
│   └── screenpipe-app-tauri/       # 桌面应用（Tauri + Next.js）
├── crates/                          # Rust 核心库
│   ├── screenpipe-server/           # 核心服务器 & 采集编排
│   ├── screenpipe-audio/            # 音频采集 & 语音转文字
│   ├── screenpipe-vision/           # 屏幕截图 & OCR
│   ├── screenpipe-db/               # SQLite 数据库层
│   ├── screenpipe-core/             # 共享基础类型、PII 脱敏、插件管理、云同步
│   ├── screenpipe-accessibility/    # 辅助功能树 / 键盘 / 鼠标 / 剪贴板捕获
│   ├── screenpipe-integrations/     # macOS 提醒事项、Windows 日历
│   ├── screenpipe-events/           # 内部发布/订阅事件总线
│   └── screenpipe-apple-intelligence/ # Apple 基础模型集成（macOS 26+）
├── packages/                        # TypeScript/JS 包
│   ├── screenpipe-js/               # SDK（Node.js + 浏览器共用）
│   ├── agent/                       # @screenpipe/agent（AI 智能体 CLI）
│   ├── ai-gateway/                  # Cloudflare Worker AI 代理网关
│   ├── cli/                         # npx screenpipe 命令行工具
│   ├── skills/                      # 内置技能模板（搜索、回忆、摘要、上下文）
│   └── sync/                        # @screenpipe/sync（每日摘要同步）
├── docs/                            # 架构设计文档
├── openspec/                        # AI 智能体工作流规范
└── docker/                          # Docker 支持
```

---

## 4. 核心功能

### 4.1 屏幕捕获与 OCR

- **事件驱动采集**：不再定时轮询，而是由用户事件（应用切换、窗口聚焦、点击、输入停顿、滚动停止、剪贴板变化、视觉变化、空闲回退）触发截图
- **多显示器同时采集**
- **OCR 引擎**：macOS 用 Apple Vision，Windows 用原生 OCR，其余用 Tesseract
- **辅助功能树**：通过 Accessibility API 获取结构化 UI 文本，比纯 OCR 更精准

### 4.2 音频捕获与语音识别

- 同时采集麦克风输入和系统音频输出
- **语音活动检测（VAD）**：Silero v5/v6，支持 6000+ 种语言
- **语音转文字**：本地 Whisper 模型 / Qwen3-ASR（0.6B GGML，GPU 加速）/ 云端 Deepgram
- **说话人分离和识别**
- **会议自动检测**

### 4.3 搜索与检索

- **全文搜索**（FTS5）：对 OCR 文本和音频转录内容进行搜索
- **向量语义搜索**（sqlite-vec）
- **过滤条件**：内容类型、应用名称、窗口标题、浏览器 URL、时间范围、说话人等

### 4.4 REST API

在 `localhost:3030` 提供完整的 HTTP API：

| 端点 | 功能 |
|------|------|
| `/search` | 全文 + 语义搜索 |
| `/frames` | 帧数据和 OCR 结果 |
| `/audio` | 音频控制和设备管理 |
| `/speakers` | 说话人管理（列表、合并、重新分配） |
| `/meetings` | 自动检测的会议记录 |
| `/elements` | 辅助功能树元素 |
| `/pipes` | 插件生命周期管理 |
| `/health` | 系统健康状态 |
| `/ws/events` | WebSocket 实时事件流 |
| `/activity-summary` | AI 生成的每日活动摘要 |

### 4.5 插件系统（Pipes）

- 插件以 Markdown 定义，存放在 `~/.screenpipe/pipes/`
- 可通过 UI 或 API 安装、启用、配置
- 内置技能：搜索（search）、回忆（recall）、摘要（digest）、上下文（context）
- 支持定时调度执行

### 4.6 桌面应用

- **系统托盘**图标，实时显示健康状态
- **全局快捷键**：覆盖层、聊天、回溯
- **浮动覆盖面板**（macOS 上使用 NSPanel，可悬浮在全屏应用之上）
- **时间轴 / 回溯视图**
- **AI 聊天面板**
- **引导式入门流程**（权限申请）
- **自动更新**
- **深度链接**

### 4.7 MCP 服务器

通过 `npx screenpipe-mcp` 暴露 MCP（Model Context Protocol）服务，可与 Claude Desktop、Cursor、VS Code 等 MCP 兼容客户端集成。

### 4.8 Apple Intelligence 集成

- `screenpipe-apple-intelligence` crate 封装 Apple Foundation Models
- 独立的 `fm-server` 二进制：在端口 5273 提供 OpenAI 兼容 API
- 设备端 TODO 提取、每日摘要 —— 零成本、零云端

### 4.9 云功能（可选，Pro 版）

- **端到端加密同步**：Argon2 密钥派生 + ChaCha20-Poly1305 加密，零知识架构
- **跨设备云搜索**
- **AI 网关**：Cloudflare Worker 路由到 OpenAI / Anthropic / Google / Deepgram

---

## 5. 架构概览

```
┌──────────────── Tauri 桌面应用 ──────────────────┐
│  Next.js 15 前端（React + Tailwind）               │
│  时间轴 | AI 聊天 | 设置 | 入门引导               │
│         │  Tauri IPC 命令/事件                     │
│  ┌──────▼──────────────────────────────────────┐   │
│  │  Rust Tauri 后端                             │   │
│  │  托盘、快捷键、窗口管理、同步、内嵌服务器     │   │
│  └──────┬──────────────────────────────────────┘   │
└─────────│──────────────────────────────────────────┘
          │ 进程内链接
┌─────────▼──────────────────────────────────────────┐
│  screenpipe-server（Axum REST + WebSocket :3030）    │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │  路由层   │  │  应用状态 │  │  插件管理器       │  │
│  └────┬─────┘  └──────────┘  └──────────────────┘  │
│       │                                              │
│  ┌────▼─────────────────────────────────────────┐   │
│  │  事件驱动采集引擎                              │   │
│  │  触发器：应用切换|点击|输入暂停|滚动停止|空闲  │   │
│  └────┬──────────────────────┬──────────────────┘   │
└───────│──────────────────────│──────────────────────┘
        │                      │
┌───────▼──────────┐  ┌───────▼──────────────────────┐
│ screenpipe-vision │  │ screenpipe-audio              │
│ 屏幕截图 + OCR    │  │ 音频采集 + VAD + 语音转文字   │
│ 辅助功能树        │  │ 说话人分离 + 会议检测         │
└───────┬──────────┘  └───────┬──────────────────────┘
        │                      │
┌───────▼──────────────────────▼─────────────────────┐
│  screenpipe-db（SQLite + FTS5 + sqlite-vec）        │
│  帧、OCR 文本、音频块、转录、UI 事件、说话人、会议  │
└───────┬────────────────────────────────────────────┘
        │
┌───────▼────────────────────────────────────────────┐
│  screenpipe-core（共享基础层）                       │
│  语言枚举、PII 脱敏、FFmpeg、插件调度、加密云同步   │
└────────────────────────────────────────────────────┘
```

**数据流**：
1. 用户操作（切换应用、点击、输入等）触发采集事件
2. 事件驱动引擎同时截取屏幕帧 + 辅助功能树
3. 帧送入 `screenpipe-vision` 进行 OCR；音频块送入 `screenpipe-audio` 进行 VAD + 语音识别
4. 结果写入 SQLite 数据库
5. `screenpipe-server` 通过 REST API 暴露所有数据
6. 前端 / SDK / AI 智能体 通过 API 查询和使用数据

---

## 6. 平台支持

| 能力 | macOS | Windows | Linux |
|------|-------|---------|-------|
| 屏幕捕获 | ScreenCaptureKit | DXGI | xcap (X11/Wayland) |
| OCR | Apple Vision | Windows OCR | Tesseract |
| 辅助功能 | Accessibility API | UIAutomation | AT-SPI2 (D-Bus) |
| 音频 | cpal | cpal | PipeWire/PulseAudio |
| 输入监听 | Accessibility | UIAutomation | evdev |
| 系统集成 | 提醒事项 | 日历 | - |
| Apple Intelligence | macOS 26+ | - | - |

---

## 7. 开发指南

- **包管理器**：JS/TS 使用 `bun`，Rust 使用 `cargo`
- **测试**：`cargo test`（Rust）、`bun test`（JS/TS）
- **API 服务端口**：`localhost:3030`
- **开发构建签名**：macOS 开发版使用开发者证书签名，保证 TCC 权限持久
- **文件头要求**：每个源文件必须包含 screenpipe 标识注释

---

## 8. 关键数据

- 当前版本：**v0.3.159**
- 许可证：**MIT**
- 主要语言：**Rust + TypeScript**
- 支持平台：**macOS / Windows / Linux**
- 分发方式：桌面应用（Tauri）、CLI（npx screenpipe）、Docker
- SDK：`@screenpipe/js`（Node.js）、`@screenpipe/browser`（浏览器）
