---
name: uv-setup
description: 在新电脑上安装并配置 uv，将所有数据统一到一个根目录下管理
disable-model-invocation: true
---

# UV 全局安装与配置

在新电脑上从零配置 uv，将所有 uv 相关文件统一到一个根目录下。

## 使用方式

用户调用时需提供根目录路径，例如：`/uv-setup C:\Programs\UV`

如果用户未提供路径，询问用户希望将 uv 安装到哪个目录。

## 步骤

### 1. 确认根目录

将用户提供的路径作为 `$UV_ROOT`（以下步骤中用 `$UV_ROOT` 代替实际路径）。

### 2. 安装 uv

```powershell
# 如果 uv 尚未安装，使用官方安装器
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

安装完成后将 `uv.exe` 移动到 `$UV_ROOT\uv.exe`（如果安装器没有直接放到该位置）。

### 3. 设置系统环境变量（管理员 PowerShell）

```powershell
[Environment]::SetEnvironmentVariable("UV_INSTALL_DIR", "$UV_ROOT", "Machine")
[Environment]::SetEnvironmentVariable("UV_CACHE_DIR", "$UV_ROOT\Cache", "Machine")
[Environment]::SetEnvironmentVariable("UV_PYTHON_CACHE_DIR", "$UV_ROOT\Cache\Python", "Machine")
[Environment]::SetEnvironmentVariable("UV_TOOL_DIR", "$UV_ROOT\Tools", "Machine")
[Environment]::SetEnvironmentVariable("UV_TOOL_BIN_DIR", "$UV_ROOT\bin", "Machine")
[Environment]::SetEnvironmentVariable("UV_PYTHON_INSTALL_DIR", "$UV_ROOT\Python", "Machine")
[Environment]::SetEnvironmentVariable("UV_PYTHON_BIN_DIR", "$UV_ROOT\bin", "Machine")
```

### 4. 配置 PATH

确保系统或用户 PATH 中包含：
- `$UV_ROOT`（uv.exe 本体）
- `$UV_ROOT\bin`（工具和 Python shim）

移除旧的 `.local\bin` 路径（如果存在）：

```powershell
# 更新用户 PATH
$path = [Environment]::GetEnvironmentVariable("Path", "User")
$path = ($path -split ';' | Where-Object { $_ -ne "$env:USERPROFILE\.local\bin" }) -join ';'
if ($path -notlike "*$UV_ROOT\bin*") { $path = "$UV_ROOT\bin;$path" }
if ($path -notlike "*$UV_ROOT*") { $path = "$UV_ROOT;$path" }
[Environment]::SetEnvironmentVariable("Path", $path, "User")
```

### 5. 创建目录结构

```powershell
mkdir "$UV_ROOT\bin" -Force
mkdir "$UV_ROOT\Cache" -Force
mkdir "$UV_ROOT\Cache\Python" -Force
mkdir "$UV_ROOT\Python" -Force
mkdir "$UV_ROOT\Tools" -Force
```

### 6. 安装 Python（重启终端后）

```powershell
uv python install 3.14
```

### 7. 验证

```powershell
uv cache dir            # $UV_ROOT\Cache
uv tool dir             # $UV_ROOT\Tools
uv tool dir --bin       # $UV_ROOT\bin
uv python dir           # $UV_ROOT\Python
uv python dir --bin     # $UV_ROOT\bin
where.exe uv            # $UV_ROOT\uv.exe
where.exe python3.14    # $UV_ROOT\bin\python3.14.exe
```

### 8. 清理旧数据（如有）

```powershell
Remove-Item "$env:LOCALAPPDATA\uv" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item "$env:USERPROFILE\.local" -Recurse -Force -ErrorAction SilentlyContinue
```

## 最终目录结构

```
$UV_ROOT\
├── uv.exe                    ← UV_INSTALL_DIR
├── bin\                      ← UV_TOOL_BIN_DIR + UV_PYTHON_BIN_DIR
│   └── python3.14.exe
├── Cache\                    ← UV_CACHE_DIR
│   └── Python\              ← UV_PYTHON_CACHE_DIR
├── Python\                   ← UV_PYTHON_INSTALL_DIR
│   └── cpython-3.14.x-...
└── Tools\                    ← UV_TOOL_DIR
```

## 环境变量速查

| 变量 | 值 | 用途 |
|---|---|---|
| `UV_INSTALL_DIR` | `$UV_ROOT` | uv.exe 自身（self update 用） |
| `UV_CACHE_DIR` | `$UV_ROOT\Cache` | 包缓存 |
| `UV_PYTHON_CACHE_DIR` | `$UV_ROOT\Cache\Python` | Python 安装包归档缓存 |
| `UV_TOOL_DIR` | `$UV_ROOT\Tools` | 工具虚拟环境 |
| `UV_TOOL_BIN_DIR` | `$UV_ROOT\bin` | 工具可执行文件 |
| `UV_PYTHON_INSTALL_DIR` | `$UV_ROOT\Python` | Python 版本安装 |
| `UV_PYTHON_BIN_DIR` | `$UV_ROOT\bin` | Python 可执行文件链接 |

## 注意事项

- 所有变量均为官方文档确认存在的变量（https://uv.doczh.com/configuration/environment/）
- `UV_BIN_DIR`、`UV_STATE_DIR`、`UV_HOME` 不存在，不要使用
- 需要管理员权限设置 Machine 级别环境变量
- 设置完环境变量后必须重启终端才能生效
