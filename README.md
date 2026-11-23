# Windows Software Setup Skills for Claude Code

> [中文说明](README.zh-CN.md)

A collection of [Claude Code](https://claude.com/claude-code) skills that install and configure common
developer tools on a **new Windows machine** — and keep every program, version, cache and config file
under **one root directory** you choose.

Invoke a skill, give it a path, and Claude walks through download → configure → install → verify,
handling the admin-elevation steps and the version-specific gotchas along the way.

## Why these skills

Most Windows installers scatter files across `C:\Program Files`, `%APPDATA%`, `%LOCALAPPDATA%` and the
registry. These skills instead:

- **One root per tool.** Everything a tool needs (`exe`, versions, cache, config, data) lives under
  `D:\Programs\<Tool>` (or wherever you point it) — easy to back up, migrate, or delete cleanly.
- **Environment variables point into that root**, never into user-profile hidden folders.
- **Version-aware.** Some upstream projects changed their packaging; the skill documents exactly which
  version still supports a portable layout (see notes below).
- **Mirrors where they help** — npmmirror for Node/npm, TUNA for Adoptium, and so on.
- **Honest about privileges.** Every step that needs Administrator/UAC is marked as such.
- **Verify at the end.** Each skill finishes with a checklist you can paste into a terminal.

## Available skills

| Skill | Installs | Example invocation |
|-------|----------|--------------------|
| `jvms-setup` | [JVMS](https://github.com/ystyle/jvms) v2.1.x + Temurin JDK (17 by default) | `/jvms-setup D:\Programs\JVMS` |
| `nvm-setup` | [nvm-windows](https://github.com/coreybutler/nvm-windows) 1.2.2 + Node.js | `/nvm-setup D:\Programs\Nvm` |
| `volta-setup` | [Volta](https://volta.sh/) + Node.js toolchain | `/volta-setup D:\Programs\Volta` |
| `uv-setup` | [uv](https://docs.astral.sh/uv/) + Python | `/uv-setup D:\Programs\UV` |
| `mysql-setup` | MySQL 8.0 (zip distribution) as a Windows service | `/mysql-setup D:\Programs\Mysql` |
| `redis-setup` | [Redis for Windows](https://github.com/tporadowski/redis) (5.0.14) as a service | `/redis-setup D:\Programs\Redis` |
| `ssh-setup` | ed25519 SSH keys + multi-platform `~/.ssh/config` | `/ssh-setup github gitlab` |

> Skill instructions are written in **Chinese**. The workflow itself is language-independent — Claude
> reads the doc and runs the commands.

## Requirements

- **Windows 10 / 11** (these skills target Windows; `ssh-setup` also applies to Linux/macOS with minor
  path changes)
- **PowerShell 5.1+** (Windows PowerShell or PowerShell 7)
- **[Claude Code](https://claude.com/claude-code)** CLI, desktop app, or IDE extension
- **Administrator access** for the steps that set machine-level environment variables, install Windows
  services, or create symlinks (Claude will trigger a UAC prompt)
- **Internet access** to GitHub Releases, npmmirror, TUNA, dev.mysql.com, etc.

## Installation

Claude Code discovers skills at `~/.claude/skills/<skill-name>/SKILL.md`. Copy the skill folders there:

```powershell
git clone https://github.com/jayteen/skills-develop-software-installation.git
$dst = "$env:USERPROFILE\.claude\skills"
Get-ChildItem .\skills-develop-software-installation -Directory |
    Where-Object { $_.Name -notmatch '^\.' } |
    ForEach-Object { Copy-Item $_.FullName $dst -Recurse -Force }
```

Or copy just the ones you need:

```powershell
Copy-Item .\skills-develop-software-installation\jvms-setup "$env:USERPROFILE\.claude\skills\" -Recurse -Force
```

> Cloning the whole repo directly into `~/.claude/skills` will **not** work — skills would end up one
> level too deep to be discovered. Copy the folders as shown above.

Restart Claude Code, then type `/jvms-setup` (or another skill name) to confirm it is loaded.

## Usage

Each skill is a slash command that takes the target root directory as its argument:

```
/jvms-setup D:\Programs\JVMS
/nvm-setup  D:\Programs\Nvm
/uv-setup   D:\Programs\UV
```

If you omit the path, the skill asks you for one first. Then Claude:

1. creates the directory layout,
2. sets machine/user environment variables (elevating via UAC when needed),
3. downloads and unpacks the tool,
4. installs/activates versions,
5. prints a verification checklist.

**Open a new terminal afterwards** — environment-variable changes do not apply to already-running shells.

All skills ship with `disable-model-invocation: true`, so Claude will not trigger them on its own;
you must call them explicitly.

## Repository structure

```
.
├── jvms-setup/SKILL.md          # JDK version manager + JDK
├── nvm-setup/SKILL.md           # nvm-windows + Node.js
├── volta-setup/
│   ├── SKILL.md                 # Volta + Node.js toolchain
│   └── references/环境变量官方文档.md
├── uv-setup/SKILL.md            # uv + Python
├── mysql-setup/SKILL.md         # MySQL as a Windows service
├── redis-setup/SKILL.md         # Redis (Windows port) as a service
├── ssh-setup/SKILL.md           # SSH keys + ~/.ssh/config
├── README.md                    # this file
├── README.zh-CN.md              # Chinese README
└── LICENSE
```

## Writing your own skill

A skill is a folder with a `SKILL.md`:

```markdown
---
name: mytool-setup
description: One line explaining what it does and when to use it
disable-model-invocation: true
---

# MyTool setup

## 使用方式          <!-- / usage -->
## 步骤              <!-- numbered steps with copy-pasteable commands -->
## 最终目录结构       <!-- resulting layout -->
## 常见问题          <!-- troubleshooting -->
```

Keep the same shape as the existing skills: confirm root directory → prerequisites/admin → download →
configure → install → verify → troubleshooting. Mark every step that needs elevation.

## Notes and limitations

- **Versions drift.** Before running, check the release page for the tool; paths and flags change
  between major versions. The skills pin versions only where portable installs depend on them:
  - **nvm-windows 1.2.2** — the last release that ships `noinstall.zip`; 2.x only provides installers.
  - **jvms 2.1.20** — store/download/cache locations are relative to `jvms.exe`; older tutorials differ.
- **Redis on Windows** is a community port (5.0.14), not an official build. Use WSL2/Linux for
  production.
- **MySQL/Redis services** are registered as Windows services (`MySQL80`, `Redis`) and auto-start.
- Skills assume a **clean machine**. If a tool is already installed, read the skill first — some steps
  (e.g. removing old PATH entries) are safe and idempotent, others (re-initializing a data directory)
  are not.
- Not tested on non-Windows systems.

## License

[MIT](LICENSE) © 2026 jayteen
