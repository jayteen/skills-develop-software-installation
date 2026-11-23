---
name: jvms-setup
description: 在新电脑上安装并配置 JVMS（v2.1.x），将所有 JDK 版本统一到一个根目录下管理
disable-model-invocation: true
---

# JVMS 全局安装与配置

在新电脑上从零配置 JVMS（JDK Version Manager for Windows），将所有 JVMS 文件和 JDK 版本统一到一个根目录下。

> 本文基于 **jvms v2.1.20 实测**。v2.1.x 与 2.0 以前的行为差异很大（没有 `add`、没有 `--mirror`、没有 `jvms.conf`），
> 网上大量老教程不适用于当前版本。

## 使用方式

用户调用时需提供根目录路径，例如：`/jvms-setup D:\Programs\JVMS`

如果用户未提供路径，询问用户希望将 JVMS 安装到哪个目录。

## 前置说明（v2.1.20 的关键事实）

| 事实 | 说明 |
|------|------|
| **store / download 相对 exe 目录** | 不受配置控制。`jvms.exe` 放在哪，`store\`、`download\` 就建在哪。 |
| **配置是 JSON** | `%APPDATA%\jvms\jvms.json`，不是 `config\jvms.conf` |
| **`JVMS_CONFIG` / `JVMS_STORE` / `JVMS_HOME` 不被读取** | v2.1.20 只认 `JAVA_HOME`；这三个变量可设置但无效果 |
| **`jvms init` 需要管理员** | 创建符号链接 + 写系统环境变量 |
| **`jvms switch` 需要管理员** | 切换 `jdk` 符号链接 |
| **`jvms install` 不需要管理员** | 只下载解压到 `store\` |
| **没有 `jvms add`** | 手动解压的 JDK 用 `jvms switch -p <路径>` 直接指向 |
| **没有 `--mirror`** | 镜像机制改为 `jvms config --originalpath`（索引地址）+ `--proxy` |
| **init 会重复追加 PATH** | 每次 `jvms init` 都往 PATH 追加条目，跑完要清理重复项 |

## 步骤

以下 `$JVMS_ROOT` 代表用户提供的根目录（如 `D:\Programs\JVMS`）。

### 1. 提权方式

当前会话不是管理员时，把步骤写成脚本再提权执行：

```powershell
$script = "$env:TEMP\jvms-setup.ps1"
# ……把要执行的内容写入 $script，脚本里用 Add-Content 写日志……
Start-Process powershell.exe -Verb RunAs `
  -ArgumentList '-NoProfile','-ExecutionPolicy','Bypass','-File',$script `
  -Wait
```

> `Start-Process -Verb RunAs` 会弹 UAC。脚本自己拿不到交互输出，建议把内容写到 `$JVMS_ROOT\setup.log` 再读回。
> 不要用 `-Wait` 的窗口做长下载（看不到进度），长下载命令单独跑。

### 2. 设置环境变量（管理员）

```powershell
[Environment]::SetEnvironmentVariable("JAVA_HOME",  "$JVMS_ROOT\jdk",    "Machine")
[Environment]::SetEnvironmentVariable("JVMS_HOME",  "$JVMS_ROOT",        "Machine")
[Environment]::SetEnvironmentVariable("JVMS_CONFIG","$JVMS_ROOT\config", "Machine")
[Environment]::SetEnvironmentVariable("JVMS_STORE", "$JVMS_ROOT\store",  "Machine")
```

后三个对 v2.1.20 无效，仅作约定/兼容保留。

### 3. 创建目录

```powershell
foreach ($d in "store","config","download") {
    New-Item -ItemType Directory -Path (Join-Path "$JVMS_ROOT" $d) -Force | Out-Null
}
```

**不要创建 `bin\`**——见下一步。

### 4. 下载 jvms 并放到根目录（不是 bin\）

```powershell
# 先看 https://github.com/ystyle/jvms/releases 确认最新版本号
$ver = "v2.1.20"
$url = "https://github.com/ystyle/jvms/releases/download/$ver/jvms_v$($ver.TrimStart('v'))_amd64.zip"
$tmp = "$env:TEMP\jvms-dl"
New-Item -ItemType Directory -Path $tmp -Force | Out-Null
Invoke-WebRequest -Uri $url -OutFile "$tmp\jvms.zip" -TimeoutSec 300
Expand-Archive -Path "$tmp\jvms.zip" -DestinationPath $tmp -Force
Copy-Item "$tmp\jvms.exe" "$JVMS_ROOT\jvms.exe" -Force
```

压缩包里只有一个 `jvms.exe`（约 11.8 MB，无 dll）。

> **为什么放根目录**：`store\` 和 `download\` 固定相对 exe 所在目录。放 `$JVMS_ROOT\bin\` 会让 JDK 落到 `bin\store\`。
> 放根目录可让 store/download 天然位于根下，比额外做目录联接稳定。

### 5. 配置 PATH（管理员）

```powershell
$root = "$JVMS_ROOT"
$mp = [Environment]::GetEnvironmentVariable("Path","Machine")
foreach ($d in @("$root\jdk\bin", $root)) {
    if ($mp -notlike "*$d*") { $mp = "$d;$mp" }
}
[Environment]::SetEnvironmentVariable("Path", $mp, "Machine")

$up = [Environment]::GetEnvironmentVariable("Path","User")
if ($up -notlike "*$root*") {
    [Environment]::SetEnvironmentVariable("Path", "$root;$up", "User")
}
```

系统 PATH 优先放 `$JVMS_ROOT\jdk\bin`（`java`/`javac`），再放 `$JVMS_ROOT`（`jvms` 本体）。

### 6. 把配置目录收进根目录（管理员，可选但推荐）

v2.1.20 固定从 `%APPDATA%\jvms\` 读写配置和索引缓存。用一个目录联接把它指到根目录：

```powershell
$appCfg = "$env:APPDATA\jvms"
$tgtCfg = "$JVMS_ROOT\config"
if ((Test-Path $appCfg) -and -not (Get-Item $appCfg -Force).LinkType) {
    Get-ChildItem $appCfg -Force | Copy-Item -Destination $tgtCfg -Recurse -Force
    Remove-Item $appCfg -Recurse -Force
}
if (-not (Test-Path $appCfg)) {
    cmd /c "mklink /J `"$appCfg`" `"$tgtCfg`""
}
```

> 这一步只是整洁性。**注意**：如果清理工具删除了这个联接，jvms 会在 AppData 下重建真实目录，配置会“分叉”（JDK 不受影响）。

### 7. 初始化（管理员）

```powershell
& "$JVMS_ROOT\jvms.exe" init --java_home "$JVMS_ROOT\jdk"
```

正常输出：

```
set `JAVA_HOME` Environment variable to  <root>\jdk
add jvms.exe to `path` Environment variable
JVMS native Windows provider is installed and working.
```

`init` 会创建 `$JVMS_ROOT\jdk` 符号链接（此时可能指向空/不存在，切换 JDK 后才有内容）。

### 8. 安装 JDK（普通权限即可）

```powershell
# 查看可安装版本（含序号）
& "$JVMS_ROOT\jvms.exe" rls -a

# 按目录名安装（推荐，名字即 store 下的文件夹名）
& "$JVMS_ROOT\jvms.exe" install "OpenJDK17U-jdk_x64_windows_hotspot_17.0.21.1_2"
```

下载包缓存在 `$JVMS_ROOT\download\`，解压到 `$JVMS_ROOT\store\<目录名>\`。

**实测速度**：GitHub Releases 直连约 10 MB/s（191 MB 用 22 秒）。如果被墙/很慢：

```powershell
# 设置下载代理
& "$JVMS_ROOT\jvms.exe" proxy --set "http://127.0.0.1:7890"
# 或换索引地址（jdkdlindex.json 的镜像）
& "$JVMS_ROOT\jvms.exe" config --originalpath "<可访问的索引 URL>"
```

手动下载的 JDK（不经过 install）用路径直接激活：

```powershell
& "$JVMS_ROOT\jvms.exe" switch -p "D:\SomeWhere\jdk-17"
```

### 9. 切换版本（管理员）

```powershell
& "$JVMS_ROOT\jvms.exe" switch "OpenJDK17U-jdk_x64_windows_hotspot_17.0.21.1_2"
# 或按序号
& "$JVMS_ROOT\jvms.exe" switch "#1"
```

`switch` 会重建 `$JVMS_ROOT\jdk` 符号链接并写入 `current_jdk_version`。也可以一步到位：

```powershell
& "$JVMS_ROOT\jvms.exe" use 17      # 未安装则自动安装（= install + switch）
```

### 10. 清理 PATH 重复项（管理员）

`init`（以及某些命令）会往 PATH 追加条目，可能产生重复或失效的 `...\bin` 路径：

```powershell
function Clean-Path([string]$scope) {
    $raw = [Environment]::GetEnvironmentVariable("Path", $scope)
    $out = New-Object System.Collections.Generic.List[string]
    foreach ($e in ($raw -split ';')) {
        $t = $e.Trim()
        if ($t -eq '') { continue }
        if ($t -ieq "$JVMS_ROOT\jdk\bin" -or $t -ieq "$JVMS_ROOT\jdk/bin") { $t = "$JVMS_ROOT\jdk\bin" }
        if (-not ($out | Where-Object { $_ -ieq $t })) { $out.Add($t) }
    }
    [Environment]::SetEnvironmentVariable("Path", ($out -join ';'), $scope)
}
Clean-Path 'Machine'
Clean-Path 'User'
```

### 11. 验证（新终端）

```powershell
echo $env:JAVA_HOME            # $JVMS_ROOT\jdk
echo $env:JVMS_HOME            # $JVMS_ROOT
jvms --version                 # v2.1.20
jvms ls                        # * 标记当前版本
java -version                  # 已切换的 JDK 版本
javac -version
where.exe java                 # $JVMS_ROOT\jdk\bin\java.exe
Test-Path "$JVMS_ROOT\jdk"     # True
(Get-Item "$JVMS_ROOT\jdk").Target   # store\<目录名>
```

## 最终目录结构

```
$JVMS_ROOT\
├── jvms.exe                  ← 主程序（必须在根目录）
├── store\                    ← JDK 存放（相对 exe 目录）
│   └── OpenJDK17U-jdk_x64_windows_hotspot_17.0.21.1_2\
│       ├── bin\  lib\  conf\  ...
├── download\                 ← 安装包缓存（*.zip）
├── config\                   ← ← %APPDATA%\jvms 目录联接指向这里
│   ├── jvms.json             ← 配置文件
│   └── jdk_versions.json     ← 版本索引缓存
├── jdk  →  store\<目录名>     ← JAVA_HOME 符号链接（switch 时重建）
└── setup.log                 ← （可选）安装日志
```

## 配置文件 jvms.json

位于 `%APPDATA%\jvms\jvms.json`（做了第 6 步后即 `$JVMS_ROOT\config\jvms.json`）：

```json
{
    "java_home": "D:\\Programs\\JVMS\\jdk",
    "current_jdk_version": "OpenJDK17U-jdk_x64_windows_hotspot_17.0.21.1_2",
    "original_path": "https://raw.githubusercontent.com/ystyle/jvms/new/jdkdlindex.json",
    "proxy": "",
    "cache_enabled": true,
    "cache_ttl": "24h"
}
```

**优先用 `jvms config` 修改**（直接编辑 JSON 会在下次 `init` 时被重写，不认识的键会被丢弃）：

```powershell
jvms config                                              # 查看
jvms config --java_home "$JVMS_ROOT\jdk"                 # JAVA_HOME 符号链接位置
jvms config --originalpath "<索引 URL>"                  # 版本索引地址
jvms config --proxy "http://127.0.0.1:7890"              # 下载代理（空值清除）
jvms config --cache_toggle false                         # 关闭索引缓存
jvms config --cache_ttl 30m                              # 缓存有效期
```

## 环境变量速查

| 变量 | 值 | 级别 | v2.1.20 是否生效 |
|------|-----|------|------------------|
| `JAVA_HOME` | `$JVMS_ROOT\jdk` | Machine | ✅ 核心 |
| `JVMS_HOME` | `$JVMS_ROOT` | Machine | ❌ 约定保留 |
| `JVMS_CONFIG` | `$JVMS_ROOT\config` | Machine | ❌ 约定保留 |
| `JVMS_STORE` | `$JVMS_ROOT\store` | Machine | ❌ 约定保留 |
| `Path` + `$JVMS_ROOT` | | Machine/User | ✅ jvms.exe |
| `Path` + `$JVMS_ROOT\jdk\bin` | | Machine | ✅ java/javac |

## 常用命令速查

```bash
jvms tui                 # 交互式界面
jvms ls                  # 已安装（* 为当前）
jvms rls -a              # 全部可安装版本（带序号）
jvms install <名称|#N>   # 安装（无需管理员）
jvms switch <名称|#N>    # 切换（需管理员）
jvms switch -p <路径>    # 直接指向某个 JDK 目录（需管理员）
jvms use <名称|#N>       # 未安装则安装，然后切换
jvms remove <名称>       # 删除版本
jvms init --java_home <路径>   # 初始化（需管理员）
jvms config [--flag=值]  # 查看/修改配置
jvms proxy --set <URL>   # 设置下载代理
jvms proxy --show        # 查看代理
```

> v2.1.20 **没有** `jvms current`、`jvms add`、`install --mirror`。

## 注意事项

- **exe 必须在根目录**：`store\`、`download\` 跟着 exe 走，放 `bin\` 就全跑偏了
- `JAVA_HOME` 指向符号链接 `$JVMS_ROOT\jdk`，不要指向 `store\<版本>` 具体目录
- `switch` / `init` / `use` 需要管理员；`install` / `ls` / `rls` / `config` 不需要
- 切换后必须**新开终端**，已有终端的环境变量不刷新
- `jvms init` 会重复追加 PATH，装完顺手清理
- 直接编辑 `jvms.json` 的未知键会被 `init` 丢弃，用 `jvms config` 改
- 版本索引来自 `raw.githubusercontent.com`，国内可能慢；可用 `--proxy` 或换 `--originalpath`
- GitHub Releases：https://github.com/ystyle/jvms/releases
- 下载缓存（`download\*.zip`，约 190 MB/版本）可安全删除，不影响已安装 JDK
