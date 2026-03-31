[English](README.md) | [中文](README_CN.md)

# CVM - Claude Code Version Manager

Manage multiple versions of [Claude Code](https://docs.anthropic.com/en/docs/claude-code) with ease. Install, switch, and patch — like [nvm](https://github.com/nvm-sh/nvm) for Claude Code.

## Why CVM?

Claude Code releases frequently — sometimes multiple times a day. You might need to:

- **Pin a known-good version** while a new release is validated
- **Test across versions** to verify behavior changes
- **Patch the CLI** to route API traffic through a reverse proxy (for restricted network environments)

CVM makes all of this a single command.

## Quick Start

```bash
# Install CVM
npm install -g @wei-shaw/cvm

# Initialize
cvm setup

# Add to your shell profile (~/.bashrc or ~/.zshrc)
export PATH="$HOME/.cvm/bin:$PATH"

# Install and use Claude Code
cvm install latest
cvm use latest
claude --version
```

## Installation

### Prerequisites

- **Node.js** >= 18.0.0
- **npm** (comes with Node.js)

### From npm (recommended)

```bash
npm install -g @wei-shaw/cvm
```

### From Source

```bash
git clone https://github.com/Wei-Shaw/cvm.git && cd cvm
pnpm install
pnpm build
npm link        # makes `cvm` available globally
```

### Verify

```bash
cvm --version
cvm setup
```

After running `setup`, add the PATH line it prints to your shell profile and restart your shell (or `source ~/.bashrc`).

## Commands

### `cvm setup`

Initialize the CVM directory structure and install the `claude` shim.

```bash
cvm setup
```

Creates `~/.cvm/` with the required directory layout and generates platform-specific shim scripts that intercept the `claude` command.

---

### `cvm install <version>`

Install a Claude Code version.

```bash
cvm install latest          # latest release
cvm install stable          # stable release
cvm install 2.1.87          # exact version
cvm install 2.1.87 --force  # reinstall even if exists
```

The first version installed is automatically activated. Each version is isolated in its own directory under `~/.cvm/versions/`.

---

### `cvm uninstall <version>`

Remove an installed version.

```bash
cvm uninstall 2.1.81
cvm uninstall 2.1.81 --force  # remove even if active or patched
```

---

### `cvm use <version>`

Switch the active Claude Code version.

```bash
cvm use 2.1.87
```

This updates the `~/.cvm/active` symlink and regenerates the `claude` shim with the correct entry point for that version.

---

### `cvm current`

Print the active version. Useful in scripts.

```bash
$ cvm current
2.1.87
```

---

### `cvm list`

List installed versions. The active version is marked with `*`.

```bash
$ cvm list
 * 2.1.81
   2.1.87 [patched]
```

List available versions from the npm registry:

```bash
$ cvm list --remote
$ cvm list --remote --last 50   # show last 50 versions (default: 20)
```

---

### `cvm patch proxy <url>`

Replace all Anthropic API domains in the active version's CLI bundle with a custom proxy URL. This is the core feature for environments that cannot reach Anthropic's API directly.

```bash
cvm patch proxy https://your-proxy.example.com
```

What gets replaced:

| Original Domain | Description |
|---|---|
| `https://api.anthropic.com` | Main API endpoint |
| `https://api-staging.anthropic.com` | Staging API endpoint |
| `https://platform.claude.com` | OAuth / platform endpoint |
| `https://mcp-proxy.anthropic.com` | MCP proxy endpoint |

Additionally, an internal domain validation check is bypassed so the SDK treats the proxy URL as a first-party endpoint.

**Options:**

```bash
cvm patch proxy <url> -V 2.1.81    # patch a specific version (default: active)
```

**Key behaviors:**

- **Idempotent** — re-running always patches from the original backup, so you can change the URL freely
- **Safe** — a `.bak` file is created before the first patch; the original is never lost
- **Version-specific** — patches are tracked per version; switching versions does not carry patches over

---

### `cvm patch revert`

Restore the original, unpatched CLI.

```bash
cvm patch revert
cvm patch revert -V 2.1.81
```

---

### `cvm patch status`

Check patch state and verify effectiveness.

```bash
$ cvm patch status
v2.1.81: patched → https://your-proxy.example.com (2026-03-31T06:44:48.141Z)

  Remaining original domains:
    api-staging.anthropic.com        0 (clean)
    api.anthropic.com                0 (clean)
    platform.claude.com              0 (clean)
    mcp-proxy.anthropic.com          0 (clean)
```

## How It Works

### Directory Layout

```
~/.cvm/
├── versions/                  # isolated version installs
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

### Shim Mechanism

When you run `claude`, the shim script in `~/.cvm/bin/` resolves the `active` symlink and executes `node <active-version>/cli.js` with all arguments forwarded. This adds zero overhead — no config parsing or version resolution at runtime.

The entry point (`cli.js` vs `start.js`) is read from each version's `package.json` at `cvm use` time and baked into the shim, so it adapts to structural changes across Claude Code versions.

### Proxy Patching

Claude Code ships as a minified JavaScript bundle (`cli.js`, ~7 MB). The proxy patcher performs literal string replacement on the bundled file — replacing hardcoded Anthropic domain strings with your proxy URL. This approach is:

- **Version-stable** — URL strings don't change with minification; they're the same across all versions
- **Non-destructive** — the original file is backed up and can be restored at any time
- **Idempotent** — patches are always applied from the pristine backup, never stacked

## Platform Support

| Platform | Status | Notes |
|---|---|---|
| Linux | Fully supported | Bash shim, symlinks |
| macOS | Fully supported | Bash shim, symlinks |
| Windows | Supported | CMD + PowerShell shims, NTFS junctions (no admin required) |
| WSL/WSL2 | Fully supported | Treated as Linux |

## Configuration

CVM respects the following environment variable:

| Variable | Default | Description |
|---|---|---|
| `CVM_DIR` | `~/.cvm` | Override the CVM home directory |

### Registry

CVM auto-detects your npm registry from `npm config get registry`. To override:

```bash
# In ~/.cvm/config.json
{
  "registry": "https://registry.npmmirror.com"
}
```

## Development

```bash
git clone https://github.com/Wei-Shaw/cvm.git && cd cvm
pnpm install
pnpm build          # one-time build
pnpm dev            # watch mode
```

### Project Structure

```
src/
├── bin.ts           # entry point
├── cli.ts           # command definitions (commander)
├── types.ts         # shared interfaces
├── util.ts          # spawn, semver, colors, spinner
└── core/
    ├── config.ts    # config.json read/write
    ├── paths.ts     # directory constants
    ├── patcher.ts   # proxy patch engine
    ├── registry.ts  # npm registry client
    ├── shim.ts      # cross-platform shim generation
    └── versions.ts  # install/uninstall/switch logic
```

**Design principles:**

- Single runtime dependency (`commander`)
- 20 KB compiled output
- Cross-platform from day one (POSIX + Windows)
- No abstractions beyond what the feature set requires

## Contributing

Contributions are welcome. Please open an issue first to discuss what you'd like to change.

## License

MIT
