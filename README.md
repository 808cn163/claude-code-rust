# Claude Code Rust

> 本文为 README.md 的中文译本，并新增「从源码编译与部署」一节。命令、路径、代码与 URL 均保留原文。

Claude Code 的原生 Rust 终端界面。可直接替换 Anthropic 官方基于 Node.js/React Ink 的 TUI，为性能与更佳的使用体验而打造。

[![Version](https://img.shields.io/github/v/release/808cn163/claude-code-rust?label=version)](https://github.com/808cn163/claude-code-rust/releases/latest)
[![CI](https://github.com/808cn163/claude-code-rust/actions/workflows/pr.yml/badge.svg)](https://github.com/808cn163/claude-code-rust/actions/workflows/pr.yml)
[![Docs](https://img.shields.io/badge/docs-GitHub%20Pages-blue)](https://srothgan.github.io/claude-code-rust/)
[![License](https://img.shields.io/badge/license-Apache%202.0-blue)](https://www.apache.org/licenses/LICENSE-2.0)

<p align="center">
  <img src="assets/demo.gif" alt="Claude Code Rust explaining the terminal problems its native Rust interface solves" width="900">
</p>

## 简介

Claude Code Rust 用一个基于 [Ratatui](https://ratatui.rs/) 构建的原生 Rust 二进制，替换了官方 Claude Code 的终端界面。它通过本地 Agent SDK bridge 连接同一个 Claude API。Claude Code 的核心功能均照常工作，包括工具调用、文件编辑、终端命令与权限管理。

## 前置条件

- 必须安装 Claude Code CLI，用于在部分 SDK 尚不支持的功能上作为回退方案。

## 安装

### 安装脚本（推荐，v0.14.0+）

**macOS/Linux：**

```bash
curl -fsSL https://raw.githubusercontent.com/808cn163/claude-code-rust/main/scripts/install/install.sh | sh
```

**Windows PowerShell：**

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -Command "irm 'https://raw.githubusercontent.com/808cn163/claude-code-rust/main/scripts/install/install.ps1' | iex"
```

### npm（全局安装）

```bash
npm install -g claude-code-rust
```

关于锁定版本、自定义安装位置、切换安装方式、卸载与故障排查，请参阅[安装指南](https://srothgan.github.io/claude-code-rust/installation.html)。

## 从源码编译与部署

这一节介绍如何自行编译，并把编译产物手工部署到系统上。

### 架构：本项目的四个组成部件

运行 `claude-rs` 时，实际需要以下四个部件同时就位：

1. **Rust 二进制**（`claude-rs`）—— TUI 本体
2. **Bun 运行时**（文件名必须为 `claude-rs-bridge-bun`）—— 用来执行下面的 bridge 脚本
3. **bridge 脚本**（`agent-sdk/dist/bridge.js`）—— TypeScript 编译产物，由 `agent-sdk/src/bridge.ts` 经 `tsc` 生成
4. **Node 依赖**（`agent-sdk/node_modules/`）—— 其中含 `@anthropic-ai/claude-agent-sdk`，**版本必须严格为 `0.3.270`**

**关键点：release 二进制在查找前两个部件时，只看「可执行文件的祖先目录」。** 依据 `src/agent/bridge.rs`，`allow_dev_fallbacks = cfg!(debug_assertions)`，release 构建下该值为 false，此时：

- bridge 脚本只在可执行文件的祖先目录中查找 `agent-sdk/dist/bridge.js`
- Bun 运行时只在可执行文件的祖先目录中查找 `claude-rs-bridge-bun`
- 环境变量 `CLAUDE_RS_AGENT_BRIDGE`、`CLAUDE_RS_AGENT_BRIDGE_RUNTIME` **完全无效**（后者在 release 下连读都不读）
- **PATH 中的 `bun` 同样不认**（只认那个特定的文件名）

以安装在 `/usr/local/bin/claude-rs` 的情况为例，其祖先目录依次是 `/usr/local/bin`、`/usr/local`、`/usr`、`/`（向上最多追溯 8 层），因此相关文件需要放在 `/usr/local/bin/` 下。

（debug 构建 `cargo build` 则宽松得多：会回退到 `CARGO_MANIFEST_DIR`、当前工作目录、环境变量，以及 PATH 中的 `bun`。本地开发时用 debug 构建最省事。）

### 编译前置依赖

- **Rust 1.88+**（项目 `Cargo.toml` 中写的是 `rust-version = "1.88.0"`、`edition = "2024"`；`rust-toolchain.toml` 声明 `channel = "1.89"`，装了 rustup 的会自动使用 1.89）
- **Node.js >= 24**（`agent-sdk/package.json` 的 engines 要求；实测 22 也能构建成功，只会出现 EBADENGINE 警告）
- **Bun**（用于运行 bridge，实测 1.3.14）
- **静态编译额外需要**：`musl-tools`、`musl-dev`、`cmake`、`go`、`perl`、`pkg-config`
  - 原因：musl 构建中真正调用 C 编译器的只有两个 crate，且都从自带源码现场编译 —— `aws-lc-sys`（cmake 构建，由 rustls ← hyper-rustls ← reqwest 引入）与 `onig_sys`（cc 构建，由 syntect 引入）。它们会自动回退到 `musl-gcc`。
  - **不需要** `libx11-dev` / `libxcb1-dev` / `libdbus-1-dev`：`arboard`、`x11rb`、`zbus` 都是纯 Rust 实现，不链接系统库。

### 编译步骤

```bash
git clone https://github.com/808cn163/claude-code-rust
cd claude-code-rust

# 1) 编译 bridge 脚本（生成 agent-sdk/dist/bridge.js）
npm --prefix agent-sdk install
npm --prefix agent-sdk run build

# 2a) 编译动态链接（glibc）版本
cargo build --release --locked

# 2b) 或：编译静态链接（musl）版本
cargo build --release --locked --target x86_64-unknown-linux-musl
```

- `--locked` 保证不修改 `Cargo.lock`
- glibc 版约 27.6 MB，musl 静态版约 27.7 MB，两者体积相近；musl 版实测为 `static-pie linked`，`ldd` 显示 statically linked，`DT_NEEDED` 为 0，无 `INTERP` 段，在 `env -i` 清空环境下可正常运行
- musl 版产物路径：`target/x86_64-unknown-linux-musl/release/claude-rs`
- glibc 版产物路径：`target/release/claude-rs`

### 部署（发布目录布局）

由于查找只看可执行文件的祖先目录，**必须把四个部件放在同一个目录下**，形成如下布局：

```
~/.local/bin/
├── claude-rs                    # Rust 二进制
├── claude-rs-bridge-bun         # Bun 运行时，文件名必须是这个
└── agent-sdk/
    ├── dist/                    # bridge.js 及其它 tsc 产物
    └── node_modules/            # 含 @anthropic-ai/claude-agent-sdk 0.3.270
```

对应的部署命令（以安装到 `~/.local/bin` 为例）：

```bash
cd claude-code-rust

mkdir -p ~/.local/bin/agent-sdk

# 二进制
cp target/x86_64-unknown-linux-musl/release/claude-rs ~/.local/bin/claude-rs
# 或者用 glibc 版：cp target/release/claude-rs ~/.local/bin/claude-rs

# Bun 运行时（必须改名）
cp /usr/bin/bun ~/.local/bin/claude-rs-bridge-bun

chmod 755 ~/.local/bin/claude-rs ~/.local/bin/claude-rs-bridge-bun

# bridge 脚本
cp -r agent-sdk/dist ~/.local/bin/agent-sdk/dist

# Node 依赖（软链可省约 220 MB）
ln -sfn "$PWD/agent-sdk/node_modules" ~/.local/bin/agent-sdk/node_modules
```

**关于 `node_modules` 是复制还是软链**：软链省空间，但要求仓库路径长期不变，且不能删除 `agent-sdk/node_modules`。想彻底独立，就改用 `cp -r agent-sdk/node_modules ~/.local/bin/agent-sdk/node_modules`（注意整个 `node_modules` 约 220 MB，其中 `@anthropic-ai/claude-agent-sdk-linux-x64` 原生包就占了约 214 MB）。

如果 `~/.local/bin` 不在 PATH 里，需要自行加入。若要安装到 `/usr/local/bin`，把上面所有 `~/.local/bin` 替换为 `/usr/local/bin` 并加上 `sudo` 即可。

### 验证

```bash
claude-rs --no-update-check doctor
```

期望输出 `0 failures`，并且能看到：

```
[PASS]  Bridge runtime         resolved <路径>/claude-rs-bridge-bun
[PASS]  Bridge runtime version 1.3.14
[PASS]  Bridge script          resolved <路径>/agent-sdk/dist/bridge.js
```

加上 `--strict` 会让警告也导致非零退出码。`doctor` 是**非交互**命令，适合在没有终端的环境下做验证；直接跑裸 `claude-rs` 则会进入全屏 TUI。

### 常见问题

1. **报 `bridge script not found near the installed executable`** —— 四个部件没有放在同一个目录下。注意 release 构建**不认环境变量**，设置 `CLAUDE_RS_AGENT_BRIDGE` 也没有用，唯一的办法就是保持上述目录布局。
2. **SDK 版本不匹配导致握手失败** —— 如果目标目录缺少 `node_modules`，Bun 会**自动安装最新版** `@anthropic-ai/claude-agent-sdk`（实测拉到 0.3.278），而 `bridge.js` 会硬校验版本号，于是报错：`Unsupported @anthropic-ai/claude-agent-sdk version: expected 0.3.270, found 0.3.278`。因此必须显式部署 `node_modules`，不能只复制 `dist/`。
3. **不要直接把裸二进制拷到 `/usr/local/bin` 就完事** —— 那样脚本和运行时都找不到。必须把 `agent-sdk/` 目录与 `claude-rs-bridge-bun` 一并放过去。

另外提一句：项目自带的 npm 分发渠道**只提供 glibc 版本**，`bin/claude-rs.js` 检测到 musl 运行时会拒绝执行（提示 build from source），但这不影响你自行编译 musl 静态版本。

### 登录

首次运行需要登录 Claude，凭据文件位于 `~/.claude/.credentials.json`。`doctor` 报告该文件缺失属于正常情况（即尚未登录），与编译部署无关。

## 使用

```bash
claude-rs
```

完整文档见 [srothgan.github.io/claude-code-rust](https://srothgan.github.io/claude-code-rust/)。

> [!NOTE]
> **Agent SDK 计费方式未变。** Anthropic 已暂停此前公布的 Agent SDK 配额调整。就目前而言一切照旧：Claude Agent SDK 的使用量（包括 `claude -p` 以及像本项目这样的第三方应用）仍然从你正常的 Claude 订阅额度中扣减。参见 [Use the Claude Agent SDK with your Claude plan](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan)。

## 为什么

官方 Claude Code TUI 运行在 Node.js + React Ink 之上，其渲染方式是直接用原始 ANSI 转义码重绘整帧画面。这带来了一系列真实存在、且被广泛反馈的问题：

- **闪烁**：每次状态更新都会重绘整个视图，导致持续闪烁（严重时会让编辑器的集成终端在长时间会话中崩溃）
- **CPU**：即使空闲也持续高占用，还会出现失控循环，反复拉起多个后台进程
- **内存**：基线占用 200-400MB（且随对话变长而攀升），而原生二进制仅约 20-50MB
- **窗口缩放**：调整窗口大小会在回滚缓冲区里留下重复帧，缩小时丢行，还可能把界面弄乱码
- **输入延迟**：随着上下文变满，按键回显出现肉眼可见的延迟，在 Windows 上尤其明显
- **回滚**：劫持终端自身的回滚缓冲区，抹掉你再也翻不回去的历史
- **粘贴**：大量粘贴可能灌爆 stdout，把终端卡死

Claude Code Rust 用一个原生终端 UI 解决了这些问题：它通过 Crossterm 与 Ratatui 进行差分、直接的终端控制，不做整帧重绘，也没有 React Ink 的渲染循环。

## 文档

手册涵盖使用脚本与 npm 安装，以及帮助、斜杠命令、键盘快捷键、设置、诊断、故障排查、从源码构建、架构说明与更新日志：

- [Installation](https://srothgan.github.io/claude-code-rust/installation.html)
- [Usage](https://srothgan.github.io/claude-code-rust/usage.html)
- [Help](https://srothgan.github.io/claude-code-rust/help.html)
- [Slash commands](https://srothgan.github.io/claude-code-rust/commands.html)
- [Settings](https://srothgan.github.io/claude-code-rust/settings.html)
- [Troubleshooting](https://srothgan.github.io/claude-code-rust/troubleshooting.html)
- [Development](https://srothgan.github.io/claude-code-rust/development.html)

## 项目状态

本项目尚未发布 1.0，正在积极开发中。想参与贡献，请参阅 [CONTRIBUTING.md](CONTRIBUTING.md)。

## 已知限制

启动过程仍然受限于本 TUI 所封装的上游 Claude Agent SDK 运行时。Rust 界面本身很快，但从启动到会话完全可用，端到端仍需等待一段可观的时间。改进这一点仍是当前的重点工作方向。

## 许可证

本项目基于 [Apache License 2.0](LICENSE) 授权。选择 Apache-2.0 是为了让个人用户、下游打包者和商业采用者的使用与再分发都保持简单顺畅。

## 免责声明与法律声明

本项目与 Anthropic 无任何关联，未获其背书，也不受其支持。

鉴于我知道大家会对这类事情有顾虑，这里简要说明一下本项目的定位：claude-code-rust 是一个我用 Rust 从零开始编写的终端 UI。它不是最新 Claude Code 源码泄露事件的 fork、副本或移植版本。它是以运行时依赖的形式，与 Anthropic 官方的 [Agent SDK](https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/agent-sdk) 通信，方式与任何其它第三方工具无异。在开发过程中的任何阶段，均未阅读 Anthropic 的源代码，也未将其作为参考。

本项目通过 Agent SDK，借助你现有的 Claude Code 账号完成身份认证，而 Agent SDK 的条款允许在其之上进行构建。计费、额度、限制与超额行为均由 Anthropic 控制，包括 Anthropic 未来可能对 Agent SDK 用量计费方式作出的任何调整。其它社区项目也采用同样的做法。据我所知，使用本项目没有问题，但我只是一名个人维护者，不是律师。如果 Anthropic 方面有任何变化，我会更新本节内容并相应调整本项目。

本项目的源代码基于 [Apache-2.0](LICENSE) 授权。Agent SDK 本身是专有软件，受 [Anthropic 商业服务条款](https://www.anthropic.com/legal/commercial-terms)约束。

Claude 官方文档请见 [https://claude.ai/docs](https://claude.ai/docs)。
