# Windows 开发环境安装 Skills（Claude Code）

> [English](README.md)

一组面向 **[Claude Code](https://claude.com/claude-code) 的安装技能（Skills）**，用于在**新 Windows 电脑**上
从零安装并配置常用开发工具，并且把程序、版本、缓存、配置**全部收敛到你指定的一个根目录**下。

调用技能、给它一个路径，Claude 会按「下载 → 配置 → 安装 → 验证」走完全流程，自动处理需要 UAC 提权的步骤，
并把各版本的特有坑绕开。

## 为什么做这个

Windows 安装器习惯把文件撒在 `C:\Program Files`、`%APPDATA%`、`%LOCALAPPDATA` 和注册表里。这些技能反其道而行：

- **一个工具一个根目录**：`exe`、版本、缓存、配置、数据都在 `D:\Programs\<工具>` 下——备份、迁移、删除都干净利落
- **环境变量直接指向根目录**，不再依赖用户目录下的隐藏文件夹
- **版本感知**：上游打包方式变过的项目，文档会写清哪个版本仍支持便携布局（见下方说明）
- **能用镜像就用镜像**：Node/npm 走 npmmirror，Adoptium 走清华 TUNA
- **权限说明诚实**：每一步是否必须管理员/UAC 都标注清楚
- **装完就验证**：每个技能末尾都有可直接粘贴的验证清单

## 技能列表

| 技能 | 安装内容 | 调用示例 |
|------|----------|----------|
| `jvms-setup` | [JVMS](https://github.com/ystyle/jvms) v2.1.x + Temurin JDK（默认 17） | `/jvms-setup D:\Programs\JVMS` |
| `nvm-setup` | [nvm-windows](https://github.com/coreybutler/nvm-windows) 1.2.2 + Node.js | `/nvm-setup D:\Programs\Nvm` |
| `volta-setup` | [Volta](https://volta.sh/) + Node.js 工具链 | `/volta-setup D:\Programs\Volta` |
| `uv-setup` | [uv](https://docs.astral.sh/uv/) + Python | `/uv-setup D:\Programs\UV` |
| `mysql-setup` | MySQL 8.0（zip 版）并注册为 Windows 服务 | `/mysql-setup D:\Programs\Mysql` |
| `redis-setup` | [Redis for Windows](https://github.com/tporadowski/redis)（5.0.14）并注册为服务 | `/redis-setup D:\Programs\Redis` |
| `ssh-setup` | ed25519 SSH 密钥 + 多平台 `~/.ssh/config` | `/ssh-setup github gitlab` |

> 技能文档正文为**中文**；工作流程与语言无关，Claude 读文档、执行命令即可。

## 环境要求

- **Windows 10 / 11**（这些技能以 Windows 为目标；`ssh-setup` 稍作路径调整也可用于 Linux/macOS）
- **PowerShell 5.1+**（Windows PowerShell 或 PowerShell 7）
- **[Claude Code](https://claude.com/claude-code)**（CLI、桌面端或 IDE 插件均可）
- **管理员权限**：设置机器级环境变量、安装 Windows 服务、创建符号链接时需要（Claude 会触发 UAC 弹窗）
- **可访问网络**：GitHub Releases、npmmirror、清华 TUNA、dev.mysql.com 等

## 安装技能

Claude Code 从 `~/.claude/skills/<技能名>/SKILL.md` 加载技能，把技能目录复制过去即可：

```powershell
git clone https://github.com/jayteen/skills-develop-software-installation.git
$dst = "$env:USERPROFILE\.claude\skills"
Get-ChildItem .\skills-develop-software-installation -Directory |
    Where-Object { $_.Name -notmatch '^\.' } |
    ForEach-Object { Copy-Item $_.FullName $dst -Recurse -Force }
```

只装其中几个也行：

```powershell
Copy-Item .\skills-develop-software-installation\jvms-setup "$env:USERPROFILE\.claude\skills\" -Recurse -Force
```

> ⚠️ 直接把仓库 clone 到 `~/.claude/skills` **不会生效**——技能会多嵌套一层目录，Claude 扫描不到。
> 请按上面的方式复制子目录。

重启 Claude Code，输入 `/jvms-setup`（或其他技能名）确认已加载。

## 使用方式

每个技能都是一个斜杠命令，参数是目标根目录：

```
/jvms-setup D:\Programs\JVMS
/nvm-setup  D:\Programs\Nvm
/uv-setup   D:\Programs\UV
```

不传路径时，技能会先问你要装到哪。之后 Claude 会：

1. 创建目录结构
2. 设置机器级/用户级环境变量（需要时通过 UAC 提权）
3. 下载并解压程序
4. 安装/激活所需版本
5. 给出验证清单

**装完请新开一个终端**——环境变量不会自动刷新到已打开的窗口。

所有技能都带 `disable-model-invocation: true`，Claude 不会自己触发，必须由你显式调用。

## 仓库结构

```
.
├── jvms-setup/SKILL.md          # JDK 版本管理 + JDK
├── nvm-setup/SKILL.md           # nvm-windows + Node.js
├── volta-setup/
│   ├── SKILL.md                 # Volta + Node.js 工具链
│   └── references/环境变量官方文档.md
├── uv-setup/SKILL.md            # uv + Python
├── mysql-setup/SKILL.md         # MySQL 安装为 Windows 服务
├── redis-setup/SKILL.md         # Redis（Windows 移植版）安装为服务
├── ssh-setup/SKILL.md           # SSH 密钥 + ~/.ssh/config
├── README.md                    # 英文说明
├── README.zh-CN.md              # 本文件
└── LICENSE
```

## 自己写一个技能

技能就是一个带 `SKILL.md` 的目录：

```markdown
---
name: mytool-setup
description: 一句话说明它做什么、什么时候用
disable-model-invocation: true
---

# MyTool 安装与配置

## 使用方式          <!-- 斜杠命令 + 参数说明 -->
## 步骤              <!-- 分步、可直接复制的命令 -->
## 最终目录结构
## 常见问题
```

建议沿用现有技能的结构：确认根目录 → 前置条件/提权 → 下载 → 配置 → 安装 → 验证 → 故障排查，
并标注每一步是否需要管理员权限。

## 注意事项与限制

- **版本会变**：运行前先看上游 Release 页；大版本之间的路径和参数可能不同。只在便携安装依赖特定版本时才锁版本：
  - **nvm-windows 1.2.2**：最后一个提供 `noinstall.zip` 的版本，2.x 只发安装包
  - **jvms 2.1.20**：store/download 路径相对 `jvms.exe`，网上大量老教程与当前版本不符
- **Windows 版 Redis** 是社区移植（5.0.14），非官方构建；生产环境请用 WSL2/Linux 官方 Redis
- **MySQL / Redis** 会注册为 Windows 服务（`MySQL80`、`Redis`）并自动启动
- 技能按**干净机器**编写。如果工具已安装，请先通读步骤——有的步骤是安全幂等的（如清理旧 PATH），
  有的不是（如重新初始化数据目录）
- 未在非 Windows 系统上验证

## 许可证

[MIT](LICENSE) © 2026 jayteen
