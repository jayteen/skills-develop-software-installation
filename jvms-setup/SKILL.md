---
name: jvms-setup
description: 在新电脑上安装并配置 JVMS，将所有 JDK 版本统一到一个根目录下管理
disable-model-invocation: true
---

# JVMS 全局安装与配置

在新电脑上从零配置 JVMS（JDK Version Manager for Windows），将所有 JVMS 文件和 JDK 版本统一到一个根目录下。

## 使用方式

用户调用时需提供根目录路径，例如：`/jvms-setup C:\Programs\JVMS`

如果用户未提供路径，询问用户希望将 JVMS 安装到哪个目录。

## 步骤

### 1. 确认根目录

将用户提供的路径作为 `$JVMS_ROOT`（以下步骤中用 `$JVMS_ROOT` 代替实际路径）。

### 2. 设置系统环境变量（管理员 PowerShell，安装前先配好）

```powershell
[Environment]::SetEnvironmentVariable("JVMS_HOME", "$JVMS_ROOT", "Machine")
[Environment]::SetEnvironmentVariable("JVMS_CONFIG", "$JVMS_ROOT\config", "Machine")
[Environment]::SetEnvironmentVariable("JVMS_STORE", "$JVMS_ROOT\store", "Machine")
[Environment]::SetEnvironmentVariable("JAVA_HOME", "$JVMS_ROOT\jdk", "Machine")
```

### 3. 创建目录结构（管理员 PowerShell）

```powershell
mkdir "$JVMS_ROOT\bin" -Force
mkdir "$JVMS_ROOT\store" -Force
mkdir "$JVMS_ROOT\config" -Force
```

### 4. 下载并解压 JVMS（管理员 PowerShell）

```powershell
# 下载最新版本（去 https://github.com/ystyle/jvms/releases 确认版本号）
$jvmsVersion = "v2.1.14"
$url = "https://github.com/ystyle/jvms/releases/download/$jvmsVersion/jvms_$jvmsVersion`_amd64.zip"
$zipPath = "$env:TEMP\jvms.zip"
Invoke-WebRequest -Uri $url -OutFile $zipPath
Expand-Archive -Path $zipPath -DestinationPath "$JVMS_ROOT\bin" -Force
Remove-Item $zipPath
```

解压后确认 `$JVMS_ROOT\bin\jvms.exe` 存在。

### 5. 配置 PATH（管理员 PowerShell）

```powershell
# 系统 PATH 添加 jvms.exe 所在目录和 JDK bin
$path = [Environment]::GetEnvironmentVariable("Path", "Machine")
if ($path -notlike "*$JVMS_ROOT\bin*") { $path = "$JVMS_ROOT\bin;$path" }
if ($path -notlike "*$JVMS_ROOT\jdk\bin*") { $path = "$JVMS_ROOT\jdk\bin;$path" }
[Environment]::SetEnvironmentVariable("Path", $path, "Machine")

# 用户 PATH 也加一份（兜底）
$userPath = [Environment]::GetEnvironmentVariable("Path", "User")
if ($userPath -notlike "*$JVMS_ROOT\bin*") { $userPath = "$JVMS_ROOT\bin;$userPath" }
[Environment]::SetEnvironmentVariable("Path", $userPath, "User")
```

### 6. 初始化 jvms（管理员 PowerShell）

```powershell
# 先让当前会话能访问 jvms
$env:PATH = "$JVMS_ROOT\bin;$env:PATH"

# 初始化，指定 JAVA_HOME 符号链接位置
jvms init --java_home "$JVMS_ROOT\jdk"
```

这会创建 `$JVMS_ROOT\jdk` 符号链接。

### 7. 创建配置文件

覆盖默认配置，写入 `$JVMS_ROOT\config\jvms.conf`：

```
# JVMS 配置文件
# JDK 存储目录
store=$JVMS_ROOT\store

# JAVA_HOME 符号链接路径
java_home=$JVMS_ROOT\jdk

# 使用国内镜像（解决网络问题）
mirror=https://mirrors.tuna.tsinghua.edu.cn/Adoptium/

# 超时时间（秒）
timeout=300

# 是否自动更新索引
auto_update=true

# 日志级别
log_level=info
```

```powershell
$configContent = @"
# JVMS 配置文件
# JDK 存储目录
store=$JVMS_ROOT\store

# JAVA_HOME 符号链接路径
java_home=$JVMS_ROOT\jdk

# 使用国内镜像（解决网络问题）
mirror=https://mirrors.tuna.tsinghua.edu.cn/Adoptium/

# 超时时间（秒）
timeout=300

# 是否自动更新索引
auto_update=true

# 日志级别
log_level=info
"@
$configContent | Out-File -FilePath "$JVMS_ROOT\config\jvms.conf" -Encoding UTF8
```

### 8. 安装 JDK（推荐手动方式）

在线安装在国内可能很慢或失败，推荐手动下载 JDK zip 包，解压到 `$JVMS_ROOT\store\` 下。

**下载 JDK：**

| 版本 | 推荐来源 | 文件名示例 |
|------|---------|-----------|
| JDK 17（Oracle） | https://www.oracle.com/java/technologies/downloads/ | `jdk-17.0.12_windows-x64_bin.zip` |
| JDK 11/17/21（Adoptium） | https://mirrors.tuna.tsinghua.edu.cn/Adoptium/ | `OpenJDK17U-jdk_x64_windows_hotspot_17.0.10_7.zip` |

**解压并注册：**

```powershell
# 示例：解压 JDK 17
Expand-Archive -Path "下载路径\jdk-17.0.12_windows-x64_bin.zip" -DestinationPath "$JVMS_ROOT\store\jdk-17.0.12" -Force

# 用 jvms add 注册本地 JDK
jvms add "$JVMS_ROOT\store\jdk-17.0.12"
```

也可以用 `jvms install` 在线安装（走清华镜像）：

```powershell
jvms install 17.0.12 --mirror https://mirrors.tuna.tsinghua.edu.cn/Adoptium/
```

### 9. 切换默认版本

```powershell
# 列出已安装的 JDK
jvms ls

# 切换到指定版本（会更新 $JVMS_ROOT\jdk 符号链接）
jvms switch 17.0.12
```

**重要：** 切换版本后必须**新开一个终端窗口**才能生效。

### 10. 验证（新终端中执行）

```powershell
echo $env:JAVA_HOME           # $JVMS_ROOT\jdk
echo $env:JVMS_HOME           # $JVMS_ROOT
jvms --version                # 版本号
jvms ls                       # 已安装的 JDK 列表
java -version                 # 当前 JDK 版本
javac -version                # 编译器版本
where.exe java                # $JVMS_ROOT\jdk\bin\java.exe
Test-Path "$JVMS_ROOT\jdk"    # True（符号链接存在）
```

## 最终目录结构

```
$JVMS_ROOT\
├── bin\                      ← jvms 主程序（加 PATH）
│   ├── jvms.exe
│   └── jvms.dll
├── store\                    ← JVMS_STORE，JDK 存储目录
│   ├── jdk-17.0.12\
│   │   ├── bin\
│   │   ├── lib\
│   │   └── ...
│   └── jdk-11.0.31\
│       ├── bin\
│       ├── lib\
│       └── ...
├── jdk                       ← JAVA_HOME，符号链接 → store/jdk-xx
│   ├── bin\
│   ├── lib\
│   └── ...
├── config\                   ← JVMS_CONFIG
│   └── jvms.conf
└── download\                 ← JDK 下载缓存（jvms init 自动创建）
```

## 环境变量速查

| 变量 | 值 | 用途 |
|------|-----|------|
| `JVMS_HOME` | `$JVMS_ROOT` | JVMS 根目录 |
| `JVMS_CONFIG` | `$JVMS_ROOT\config` | 配置文件目录 |
| `JVMS_STORE` | `$JVMS_ROOT\store` | JDK 版本存放目录 |
| `JAVA_HOME` | `$JVMS_ROOT\jdk` | 当前 JDK（符号链接，切换版本自动更新） |
| `Path`（系统） | `$JVMS_ROOT\bin` | `jvms.exe` 命令 |
| `Path`（系统） | `$JVMS_ROOT\jdk\bin` | `java`、`javac` 等命令 |

## 常用命令速查

```bash
# 版本管理
jvms ls                  # 列出已安装的 JDK
jvms rls                 # 列出远程可安装的 JDK
jvms current             # 查看当前使用的版本

# 安装与切换
jvms install 17.0.12     # 在线安装（走 mirror）
jvms add <本地路径>       # 注册本地已解压的 JDK
jvms switch 17.0.12      # 切换版本
jvms use 17              # 切换（自动匹配版本号）

# 卸载
jvms remove 17.0.12      # 删除指定版本

# 配置
jvms init --java_home <路径>  # 重新初始化
jvms proxy <代理地址>         # 设置下载代理
```

## 注意事项

- 环境变量必须在 `jvms init` **之前**设置，这样 init 才能识别自定义路径
- `jvms init --java_home` 指定的是符号链接路径，非真实 JDK 目录
- 切换 JDK 后必须**新开终端**，当前终端的环境变量不会自动刷新
- 手动下载的 JDK 解压到 `store\` 后，需用 `jvms add` 注册才能被 `jvms ls` 识别和切换
- `JAVA_HOME` 指向符号链接 `$JVMS_ROOT\jdk`，不要指向具体版本目录
- 清华镜像源地址：https://mirrors.tuna.tsinghua.edu.cn/Adoptium/
- GitHub Releases：https://github.com/ystyle/jvms/releases
- 如果在线安装失败，优先用手动下载 + `jvms add` 的方式
