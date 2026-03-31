[English](README.md) | [中文](README_CN.md)

# CVM - Claude Code 版本管理器

轻松管理多个版本的 [Claude Code](https://docs.anthropic.com/en/docs/claude-code)。安装、切换和补丁 —— 就像 Claude Code 的 [nvm](https://github.com/nvm-sh/nvm)。

## 为什么选择 CVM？

Claude Code 发布频繁 —— 有时一天多次。你可能需要：

- **固定已知良好的版本**，同时验证新发布版本
- **跨版本测试**以验证行为变化
- **修补 CLI**以通过反向代理路由 API 流量（适用于受限网络环境）

CVM 让这一切只需一个命令即可完成。

## 先卸载官方 Claude Code

CVM 通过 shim 管理自己的 `claude` 二进制文件。为避免冲突，**在使用 CVM 之前必须先卸载官方 Claude Code**。

<details>
<summary><strong>npm（全局安装）</strong></summary>

```bash
npm uninstall -g @anthropic-ai/claude-code
```
</details>

<details>
<summary><strong>macOS (Homebrew)</strong></summary>

```bash
brew uninstall claude-code
```
</details>

<details>
<summary><strong>Linux（原生二进制 / 独立安装）</strong></summary>

```bash
# 如果通过官方安装脚本安装
rm -f /usr/local/bin/claude

# 如果安装到 ~/.local/bin
rm -f ~/.local/bin/claude
```

如果你不确定 `claude` 安装在哪里：

```bash
which claude
```
</details>

卸载后，验证 `claude` 已不再可用：

```bash
which claude  # 应该没有返回或显示 "not found"
```

## 快速开始

```bash
# 安装 CVM
npm install -g @wei-shaw/cvm

# 初始化
cvm setup

# 添加到你的 shell 配置文件 (~/.bashrc 或 ~/.zshrc)
export PATH="$HOME/.cvm/bin:$PATH"

# 安装并使用 Claude Code
cvm install latest
cvm use latest
claude --version
```

## 安装

### 前提条件

- **Node.js** >= 18.0.0
- **npm**（随 Node.js 一起提供）

### 从 npm 安装（推荐）

```bash
npm install -g @wei-shaw/cvm
```

### 从源码安装

```bash
git clone https://github.com/Wei-Shaw/cvm.git && cd cvm
pnpm install
pnpm build
npm link        # 使 `cvm` 全局可用
```

### 验证

```bash
cvm --version
cvm setup
```

运行 `setup` 后，将其输出的 PATH 行添加到你的 shell 配置文件并重新启动 shell（或 `source ~/.bashrc`）。

## 命令

### `cvm setup`

初始化 CVM 目录结构并安装 `claude` shim。

```bash
cvm setup
```

创建 `~/.cvm/` 目录及所需的目录布局，并生成拦截 `claude` 命令的特定于平台的 shim 脚本。

---

### `cvm install <version>`

安装 Claude Code 版本。

```bash
cvm install latest          # 最新发布版本
cvm install stable          # 稳定版本
cvm install 2.1.87          # 精确版本
cvm install 2.1.87 --force  # 即使已存在也重新安装
```

第一个安装的版本会自动激活。每个版本都隔离在 `~/.cvm/versions/` 下的自己的目录中。

---

### `cvm uninstall <version>`

移除已安装的版本。

```bash
cvm uninstall 2.1.81
cvm uninstall 2.1.81 --force  # 即使处于激活状态或已打补丁也移除
```

---

### `cvm use <version>`

切换激活的 Claude Code 版本。

```bash
cvm use 2.1.87
```

这会更新 `~/.cvm/active` 符号链接，并为该版本重新生成具有正确入口点的 `claude` shim。

---

### `cvm current`

打印当前激活的版本。在脚本中很有用。

```bash
$ cvm current
2.1.87
```

---

### `cvm list`

列出已安装的版本。激活的版本用 `*` 标记。

```bash
$ cvm list
 * 2.1.81
   2.1.87 [patched]
```

从 npm 仓库列出可用版本：

```bash
$ cvm list --remote
$ cvm list --remote --last 50   # 显示最近 50 个版本（默认：20）
```

---

### `cvm patch proxy <url>`

将活动版本 CLI 包中的所有 Anthropic API 域名替换为自定义代理 URL。这是用于无法直接访问 Anthropic API 的环境的核心功能。

```bash
cvm patch proxy https://your-proxy.example.com
```

替换内容：

| 原始域名 | 描述 |
|---|---|
| `https://api.anthropic.com` | 主 API 端点 |
| `https://api-staging.anthropic.com` | 预发布 API 端点 |
| `https://platform.claude.com` | OAuth / 平台端点 |
| `https://mcp-proxy.anthropic.com` | MCP 代理端点 |

此外，内部域名验证检查被绕过，以便 SDK 将代理 URL 视为第一方端点。

**选项：**

```bash
cvm patch proxy <url> -V 2.1.81    # 修补特定版本（默认：激活版本）
```

**关键行为：**

- **幂等** — 重新运行始终从原始备份进行修补，因此你可以自由更改 URL
- **安全** — 在首次打补丁前创建 `.bak` 文件；原始文件永远不会丢失
- **版本特定** — 补丁按版本跟踪；切换版本不会携带补丁

---

### `cvm patch revert`

恢复原始的、未打补丁的 CLI。

```bash
cvm patch revert
cvm patch revert -V 2.1.81
```

---

### `cvm patch status`

检查补丁状态并验证有效性。

```bash
$ cvm patch status
v2.1.81: patched → https://your-proxy.example.com (2026-03-31T06:44:48.141Z)

  剩余的原始域名：
    api-staging.anthropic.com        0 (clean)
    api.anthropic.com                0 (clean)
    platform.claude.com              0 (clean)
    mcp-proxy.anthropic.com          0 (clean)
```

## 工作原理

### 目录布局

```
~/.cvm/
├── versions/                  # 隔离的版本安装
│   ├── 2.1.81/
│   │   └── node_modules/@anthropic-ai/claude-code/
│   └── 2.1.87/
│       └── node_modules/@anthropic-ai/claude-code/
├── active → versions/2.1.87/node_modules/@anthropic-ai/claude-code
├── bin/
│   ├── claude                 # bash shim (Linux/macOS)
│   ├── claude.cmd             # CMD shim (Windows)
│   └── claude.ps1             # PowerShell shim (Windows)
└── config.json
```

### Shim 机制

当你运行 `claude` 时，`~/.cvm/bin/` 中的 shim 脚本会解析 `active` 符号链接并执行 `node <active-version>/cli.js`，所有参数都会转发。这不会增加任何开销 —— 运行时无需配置解析或版本解析。

入口点（`cli.js` 与 `start.js`）在 `cvm use` 时从每个版本的 `package.json` 中读取，并烘焙到 shim 中，因此它可以适应 Claude Code 版本之间的结构变化。

### 反向代理策略

在某些受限网络环境下（如企业防火墙、区域网络策略等），可能无法直接访问 Anthropic 的 API 端点。代理补丁功能提供了一种**反向代理策略**来应对这一问题 —— 它将 API 流量重定向到用户自行搭建的反向代理服务器，使 Claude Code 在这些受限环境下也能正常工作。

补丁通过对打包的 CLI 文件（`cli.js`，约 7 MB）执行字面字符串替换来实现，将硬编码的 Anthropic 域名字符串替换为你的反向代理 URL。这种方法：

- **版本稳定** — URL 字符串不会随压缩而改变；它们在所有版本中都是相同的
- **非破坏性** — 修改前会备份原始文件，可随时通过 `cvm patch revert` 恢复
- **幂等** — 补丁始终从原始备份应用，不会堆叠

## 平台支持

| 平台 | 状态 | 说明 |
|---|---|---|
| Linux | 完全支持 | Bash shim，符号链接 |
| macOS | 完全支持 | Bash shim，符号链接 |
| Windows | 支持 | CMD + PowerShell shims，NTFS 连接点（无需管理员权限） |
| WSL/WSL2 | 完全支持 | 视为 Linux |

## 配置

CVM 尊重以下环境变量：

| 变量 | 默认值 | 描述 |
|---|---|---|
| `CVM_DIR` | `~/.cvm` | 覆盖 CVM 主目录 |

### 仓库

CVM 从 `npm config get registry` 自动检测你的 npm 仓库。要覆盖：

```bash
# 在 ~/.cvm/config.json 中
{
  "registry": "https://registry.npmmirror.com"
}
```

## 开发

```bash
git clone https://github.com/Wei-Shaw/cvm.git && cd cvm
pnpm install
pnpm build          # 一次性构建
pnpm dev            # 监视模式
```

### 项目结构

```
src/
├── bin.ts           # 入口点
├── cli.ts           # 命令定义 (commander)
├── types.ts         # 共享接口
├── util.ts          # spawn, semver, colors, spinner
└── core/
    ├── config.ts    # config.json 读/写
    ├── paths.ts     # 目录常量
    ├── patcher.ts   # 代理补丁引擎
    ├── registry.ts  # npm 仓库客户端
    ├── shim.ts      # 跨平台 shim 生成
    └── versions.ts  # 安装/卸载/切换逻辑
```

**设计原则：**

- 单一运行时依赖（`commander`）
- 20 KB 编译输出
- 从一开始就跨平台（POSIX + Windows）
- 没有超出功能集所需的抽象

## 贡献

欢迎贡献。请先打开一个 issue 讨论你想要更改的内容。

## 许可证

MIT
