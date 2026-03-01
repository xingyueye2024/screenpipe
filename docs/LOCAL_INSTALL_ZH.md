# Screenpipe macOS 本地编译安装指南

> 本文档介绍如何从源码在 macOS 上免费编译和运行 Screenpipe，无需购买任何授权。

---

## 目录

1. [系统要求](#1-系统要求)
2. [安装前置依赖](#2-安装前置依赖)
3. [克隆仓库](#3-克隆仓库)
4. [方式一：只运行后端 CLI](#4-方式一只运行后端-cli)
5. [方式二：编译桌面应用（推荐）](#5-方式二编译桌面应用推荐)
6. [方式三：开发模式（热重载）](#6-方式三开发模式热重载)
7. [方式四：npx 快速体验（无需编译）](#7-方式四npx-快速体验无需编译)
8. [macOS 权限设置](#8-macos-权限设置)
9. [验证安装](#9-验证安装)
10. [日常使用](#10-日常使用)
11. [数据存储与清理](#11-数据存储与清理)
12. [常见问题](#12-常见问题)

---

## 1. 系统要求

| 项目 | 要求 |
|------|------|
| 操作系统 | macOS 10.15 (Catalina) 及以上 |
| 处理器 | Apple Silicon (M1/M2/M3/M4) 或 Intel |
| 内存 | 建议 8GB 及以上 |
| 磁盘 | 编译约需 5GB，录制数据约 5-10GB/月 |
| Xcode | 需要**完整版 Xcode**（不能只装 Command Line Tools） |
| Rust | 1.92.0（项目自动管理） |

---

## 2. 安装前置依赖

### 2.1 安装 Xcode

从 Mac App Store 安装 Xcode，然后运行：

```bash
sudo xcodebuild -license
xcodebuild -runFirstLaunch
```

### 2.2 安装 Homebrew 依赖

```bash
brew install pkg-config ffmpeg jq cmake wget git-lfs
```

各依赖用途：

| 依赖 | 用途 |
|------|------|
| `pkg-config` | 检测系统库路径 |
| `ffmpeg` | 音视频处理 |
| `cmake` | 编译 Whisper/ONNX 等 C++ 依赖 |
| `wget` | 构建脚本下载 ffmpeg sidecar |
| `jq` | JSON 处理工具 |
| `git-lfs` | 大文件支持 |

### 2.3 安装 Rust

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env
```

项目根目录的 `rust-toolchain.toml` 会自动安装正确的 Rust 版本（1.92.0），无需手动指定。

### 2.4 安装 Bun

```bash
curl -fsSL https://bun.sh/install | bash
```

> 本项目强制使用 `bun`，不支持 npm 或 pnpm。

---

## 3. 克隆仓库

```bash
git clone https://github.com/mediar-ai/screenpipe.git
cd screenpipe
```

---

## 4. 方式一：只运行后端 CLI

如果你只需要录制和 API 功能，不需要图形界面：

```bash
# Apple Silicon（M 系列芯片）或 Intel Mac 均使用 metal 特性
cargo build --release --features metal
```

> 首次编译约需 10-20 分钟，取决于机器性能。

编译完成后运行：

```bash
./target/release/screenpipe
```

服务启动后会在 `http://localhost:3030` 提供 REST API。

### 可选：快速编译模式（开发测试用）

```bash
cargo build --profile release-dev --features metal
```

比完整 release 快 3-5 倍，适合本地测试。

---

## 5. 方式二：编译桌面应用（推荐）

完整的桌面应用包含图形界面、系统托盘、时间轴回溯等功能。

```bash
# 进入桌面应用目录
cd apps/screenpipe-app-tauri

# 安装 JavaScript 依赖
bun install

# 编译桌面应用（会自动运行 pre_build.js 下载 ffmpeg 等依赖）
bun tauri build
```

编译完成后，生成的应用位于：

```
apps/screenpipe-app-tauri/src-tauri/target/release/bundle/macos/
```

将 `.app` 文件拖到「应用程序」文件夹即可使用。

### 关于签名

自己编译的应用没有签名，macOS 会弹出安全提示。解决方法：

- 打开应用时按住 `Control` 键点击，选择「打开」
- 或者在「系统设置 → 隐私与安全性」中允许打开
- 没有签名的应用每次重新编译后需要重新授权 macOS 权限（屏幕录制、麦克风等）

---

## 6. 方式三：开发模式（热重载）

适合开发和调试，前端代码修改后自动刷新：

```bash
cd apps/screenpipe-app-tauri

# 安装依赖（首次执行）
bun install

# 启动开发模式
bun tauri dev
```

这会同时启动：
- Next.js 开发服务器（端口 1420）
- Rust Tauri 后端

---

## 7. 方式四：npx 快速体验（无需编译）

如果不想自己编译，可以直接用 npx 下载预编译的 CLI 二进制：

```bash
npx screenpipe@latest record
```

会自动下载对应平台的二进制文件和 ffmpeg，开箱即用。

> 注意：官方已标记此方式为过时，推荐使用桌面应用。

---

## 8. macOS 权限设置

Screenpipe 需要以下 macOS 权限才能正常工作：

| 权限 | 路径 | 用途 |
|------|------|------|
| 屏幕录制 | 系统设置 → 隐私与安全性 → 屏幕录制 | 捕获屏幕内容 |
| 麦克风 | 系统设置 → 隐私与安全性 → 麦克风 | 采集音频 |
| 辅助功能 | 系统设置 → 隐私与安全性 → 辅助功能 | 读取 UI 元素、键盘/鼠标事件 |

首次运行时应用会自动弹出权限请求。如果错过了，需手动到系统设置中开启。

---

## 9. 验证安装

### 检查服务是否正常运行

```bash
# 健康检查
curl http://localhost:3030/health

# 查看 API 文档
open http://localhost:3030/openapi.yaml
```

### 测试搜索功能

```bash
# 搜索 OCR 文本
curl "http://localhost:3030/search?q=hello&content_type=ocr"

# 搜索音频转录
curl "http://localhost:3030/search?q=meeting&content_type=audio"
```

### 查看已连接的音频设备

```bash
curl http://localhost:3030/audio/devices
```

---

## 10. 日常使用

### 桌面应用

- **系统托盘图标**：显示录制状态，点击可打开主窗口
- **全局快捷键**：
  - 打开搜索/聊天面板
  - 打开时间轴回溯
  - 打开浮动覆盖层
- **时间轴**：拖动查看任意时间点的屏幕截图和对应文本
- **AI 聊天**：基于你的屏幕和音频历史进行提问

### CLI 模式

```bash
# 默认启动（端口 3030，数据存储在 ~/.screenpipe/）
./target/release/screenpipe

# 自定义端口和数据目录
./target/release/screenpipe --port 3035 --data-dir ~/my-screenpipe-data
```

### 插件（Pipes）

通过 API 安装和管理插件：

```bash
# 查看已安装插件
curl http://localhost:3030/pipes/list

# 安装插件
curl -X POST http://localhost:3030/pipes/install -d '{"url": "..."}'
```

---

## 11. 数据存储与清理

默认数据目录：`~/.screenpipe/`

```
~/.screenpipe/
├── db.sqlite              # 主数据库（OCR 文本、音频转录、元数据）
├── data/                  # 屏幕截图（JPEG）和音频文件
└── pipes/                 # 已安装的插件
```

清理数据：

```bash
# 查看数据目录大小
du -sh ~/.screenpipe/

# 删除所有数据（谨慎操作）
rm -rf ~/.screenpipe/
```

---

## 12. 常见问题

### Q: 编译时报错找不到 ffmpeg

确保已通过 Homebrew 安装 ffmpeg：

```bash
brew install ffmpeg
```

如果仍然报错，检查 `pkg-config` 能否找到：

```bash
pkg-config --libs libavformat
```

### Q: 编译时报错 Whisper 相关

确保已安装 `cmake`：

```bash
brew install cmake
```

### Q: 没有签名导致每次编译都要重新授权权限

这是 macOS TCC 机制导致的。解决方案：

1. 获取一个 Apple 开发者证书（免费的 Apple ID 即可创建）
2. 在 `apps/screenpipe-app-tauri/src-tauri/tauri.conf.json` 中配置 `bundle.macOS.signingIdentity`
3. 之后编译的应用签名一致，权限不会丢失

### Q: Apple Silicon 和 Intel Mac 有什么区别？

两者都使用 `--features metal` 编译。Apple Silicon 的 Metal GPU 加速效果更好，语音识别和 OCR 更快。

### Q: 运行后 CPU 占用很高

Screenpipe 采用事件驱动采集，正常情况下 CPU 占用应低于 20%。如果偏高：

- 检查是否有大量屏幕内容变化（如播放视频）
- 尝试在设置中降低视频质量
- 关闭不需要的音频设备

### Q: 如何只编译某个 crate 进行测试？

```bash
# 只测试音频模块
cargo test -p screenpipe-audio

# 只测试 OCR 模块
cargo test -p screenpipe-vision

# 测试 macOS 原生 OCR
cargo test -p screenpipe-vision test_apple_native_ocr
```

### Q: 如何更新到最新版本？

```bash
cd screenpipe
git pull
cargo build --release --features metal
```

桌面应用：

```bash
cd apps/screenpipe-app-tauri
git pull
bun install
bun tauri build
```
