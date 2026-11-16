---
name: volta-setup
description: 在新电脑上安装并配置 Volta，将所有内容统一到一个根目录下管理
disable-model-invocation: true
---

# Volta 全局安装与配置

在新电脑上从零配置 Volta，将所有 Volta 文件（程序 + 数据）统一到一个根目录下。

## 使用方式

用户调用时需提供根目录路径，例如：`/volta-setup C:\Programs\Volta`

如果用户未提供路径，询问用户希望将 Volta 安装到哪个目录。

## 步骤

### 1. 确认根目录

将用户提供的路径作为 `$VOLTA_ROOT`（以下步骤中用 `$VOLTA_ROOT` 代替实际路径）。

### 2. 设置 VOLTA_HOME（系统环境变量，管理员 PowerShell）

```powershell
[Environment]::SetEnvironmentVariable("VOLTA_HOME", "$VOLTA_ROOT", "Machine")
```

### 3. 安装 Volta（管理员 PowerShell）

```powershell
# 确保 VOLTA_HOME 在当前会话生效
$env:VOLTA_HOME = "$VOLTA_ROOT"
mkdir "$VOLTA_ROOT\bin" -Force

# 通过 winget 安装（会装到 C:\Program Files\Volta）
winget install --id Volta.Volta --silent --accept-package-agreements

# 将二进制文件复制到 VOLTA_HOME\bin
Copy-Item "C:\Program Files\Volta\*" "$VOLTA_ROOT\bin\" -Force

# 卸载 MSI 清理注册表
$installer = New-Object -ComObject WindowsInstaller.Installer
$installer.ConfigureProduct("{73341A09-B693-496D-9FFE-0F2A86F3C922}", 0, 2)
```

### 4. 配置 PATH（用户环境变量）

```powershell
# 移除旧路径，添加 VOLTA_HOME\bin
$path = [Environment]::GetEnvironmentVariable("Path", "User")
$path = ($path -split ';' | Where-Object {
    $_ -ne "$env:USERPROFILE\.volta\bin" -and
    $_ -ne "$env:USERPROFILE\.local\bin" -and
    $_ -ne "C:\Program Files\Volta"
}) -join ';'
if ($path -notlike "*$VOLTA_ROOT\bin*") { $path = "$VOLTA_ROOT\bin;$path" }
[Environment]::SetEnvironmentVariable("Path", $path, "User")
```

### 5. 重启终端后安装 Node.js

```bash
volta install node@20
```

### 6. 安装常用全局工具

```bash
volta install pnpm
volta install yarn
volta install eslint
volta install prettier
volta install typescript
volta install nodemon
volta install nrm
volta install @anthropic-ai/claude-code
```

### 7. 验证

```powershell
$env:VOLTA_HOME            # $VOLTA_ROOT
volta --version            # 版本号
volta list                 # 已安装的工具链
where.exe node             # $VOLTA_ROOT\bin\node.exe
node --version             # v20.x
```

## 最终目录结构

```
$VOLTA_ROOT\
├── bin\                     ← 所有二进制文件
│   ├── volta.exe            ← Volta 主程序
│   ├── volta-shim.exe       ← Shim 代理
│   ├── volta-migrate.exe    ← 迁移工具
│   ├── node.exe
│   ├── npm.exe / npm.cmd
│   ├── npx.exe / npx.cmd
│   ├── pnpm.exe / pnpm.cmd
│   ├── yarn.exe / yarn.cmd
│   ├── claude / claude.cmd
│   ├── eslint / eslint.cmd
│   └── ...
├── tools\                   ← 工具版本数据
│   └── image\node\20.x.x
├── cache\                   ← 下载缓存
├── log\                     ← 日志
├── tmp\                     ← 临时文件
├── layout.v4
└── volta.lock
```

## 环境变量

| 变量 | 值 | 用途 |
|---|---|---|
| `VOLTA_HOME` | `$VOLTA_ROOT` | Volta 根目录（程序 + 数据全部在此） |
| `Path`（用户） | `$VOLTA_ROOT\bin` | 找到所有命令 |

## 参考文档

- Volta 官网：https://www.voltajs.com/
- 环境变量官方文档：https://www.voltajs.com/config/env.html
- 参考文件：`references/环境变量官方文档.md`

## 注意事项

- `VOLTA_HOME` 需在安装前设置，否则 Volta 会用默认的 `%USERPROFILE%\.volta`
- `VOLTA_NODE_MIRROR` 不是官方环境变量，不要使用
- winget 安装的 MSI 会自动注册，需卸载以避免注册表冲突
- `C:\Program Files\Volta` 只是临时中转，最终会被删除
- 设置完环境变量必须重启终端
