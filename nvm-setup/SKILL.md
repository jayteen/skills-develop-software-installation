---
name: nvm-setup
description: 在新电脑上安装并配置 nvm-windows，将所有内容（nvm 程序 + Node 版本 + 全局包 + 缓存）统一到一个根目录下管理
disable-model-invocation: true
---

# nvm-windows 安装与配置

在新电脑上从零配置 nvm-windows，所有文件统一到一个根目录下。

## 使用方式

用户调用时需提供根目录路径，例如：`/nvm-setup C:\Programs\Nvm`

如果用户未提供路径，询问用户希望将 nvm 安装到哪个目录。建议使用首字母大写（如 `Nvm`）。

## 步骤

### 1. 确认根目录

将用户提供的路径作为 `$NVM_ROOT`（以下步骤中用 `$NVM_ROOT` 代替实际路径）。

```
$NVM_ROOT  = C:\Programs\Nvm        ← NVM_HOME（nvm.exe + Node 版本目录）
$NVM_SYMLINK = $NVM_ROOT\nodejs     ← 符号链接，指向当前激活的 Node 版本
```

### 2. 下载并解压 nvm-windows（管理员 PowerShell）

```powershell
# 下载 noinstall 版本
$url = "https://github.com/coreybutler/nvm-windows/releases/download/1.2.2/nvm-noinstall.zip"
$zip = "$env:USERPROFILE\Downloads\nvm-noinstall.zip"
Invoke-WebRequest -Uri $url -OutFile $zip -UseBasicParsing

# 创建根目录并解压
New-Item -ItemType Directory -Path "$NVM_ROOT" -Force
Expand-Archive -Path $zip -DestinationPath "$NVM_ROOT" -Force
Remove-Item $zip
```

> 如果 GitHub 下载失败，提示用户手动下载 `nvm-noinstall.zip` 放到 Downloads 目录，再用上述命令解压。

> **为什么固定 1.2.2**：`noinstall.zip`（免安装、纯解压版）最后一个发布版本就是 **1.2.2**。
> 2.x（当前 latest 为 v2.0.1 / v2.1.0-beta）只提供 `setup.exe` 安装包，没有 zip 包，无法做到“全部文件收进一个根目录”。
> 需要便携布局就**必须**用 1.2.2，不要盲目升级。

### 3. 配置 settings.txt

在 `$NVM_ROOT\settings.txt` 写入：

```
root: $NVM_ROOT
path: $NVM_ROOT\nodejs
node_mirror: https://npmmirror.com/mirrors/node/
npm_mirror: https://npmmirror.com/mirrors/npm/
```

用实际路径替换 `$NVM_ROOT`。PowerShell 命令：

```powershell
@"
root: $NVM_ROOT
path: $NVM_ROOT\nodejs
node_mirror: https://npmmirror.com/mirrors/node/
npm_mirror: https://npmmirror.com/mirrors/npm/
"@ | Out-File -FilePath "$NVM_ROOT\settings.txt" -Encoding ascii
```

### 4. 设置环境变量（管理员 PowerShell）

```powershell
# NVM_HOME 和 NVM_SYMLINK（系统变量，Machine 级别）
[Environment]::SetEnvironmentVariable("NVM_HOME", "$NVM_ROOT", "Machine")
[Environment]::SetEnvironmentVariable("NVM_SYMLINK", "$NVM_ROOT\nodejs", "Machine")

# NODE_PATH（系统变量）
[Environment]::SetEnvironmentVariable("NODE_PATH", "$NVM_ROOT\nodejs\node_modules", "Machine")
```

### 5. 配置 PATH（用户环境变量）

将 `%NVM_HOME%`、`%NVM_SYMLINK%`、以及全局包目录放在 PATH **最前面**：

```powershell
$path = [Environment]::GetEnvironmentVariable("Path", "User")
# 移除可能存在的旧 nvm 路径
$path = ($path -split ';' | Where-Object {
    $_ -notmatch 'nvm|NVM' -and $_ -notmatch 'Nvm'
}) -join ';'
# 在前面插入 nvm 路径（全局包目录放最后，让 node/npm 优先级更高）
$newPath = "%NVM_HOME%;%NVM_SYMLINK%;$NVM_ROOT\node_global;$path"
[Environment]::SetEnvironmentVariable("Path", $newPath, "User")
```

PATH 优先级：
```
1. %NVM_HOME%                ← nvm 命令
2. %NVM_SYMLINK%             ← node / npm（当前激活版本）
3. $NVM_ROOT\node_global     ← 全局包可执行文件
```

### 6. 重启终端后安装 Node.js

**关闭所有终端窗口，重新打开**（让环境变量生效），然后：

```bash
nvm install 22          # 安装 Node 22 LTS 的最新版
nvm list                # 看实际装到的版本号（如 22.20.0）
nvm use 22              # 激活 22.x（也可写完整版本号，如 nvm use 22.20.0）
```

> ⚠️ **`nvm use` 需要管理员权限**：它要把 `nodejs` 目录换成符号链接。要么以管理员身份开终端，
> 要么在系统设置里启用 **开发者模式**（设置 → 系统 → 开发者选项 → 开发人员模式）。
> 否则会报 “Access is denied” 或静默不生效。

> 如果 `nvm` 命令找不到，检查 PATH 中 `%NVM_HOME%` 是否正确展开。可以 `echo $env:NVM_HOME` 确认。
> 新装的 Node 版本号以 `nvm list` 实际输出为准，别把示例版本号写死。

### 7. 创建全局包和缓存目录（⭐ 关键：放在 symlink 外面）

⚠️ **不要**把 `node_global` 和 `node_cache` 放在 `nodejs` 目录内——`nodejs` 是符号链接，切换 Node 版本后内容会丢失。

正确做法：放在 NVM 根目录下，与 symlink 平级。

```bash
mkdir "$NVM_ROOT\node_global"
mkdir "$NVM_ROOT\node_cache"
```

### 8. 配置 npm（prefix / cache / registry）

```bash
npm config set cache "$NVM_ROOT\node_cache"
npm config set prefix "$NVM_ROOT\node_global"
npm config set registry "https://registry.npmmirror.com/"
```

验证：`cat ~/.npmrc` 应输出：
```
cache=$NVM_ROOT\node_cache
prefix=$NVM_ROOT\node_global
registry=https://registry.npmmirror.com/
```

### 9. 安装常用全局工具

```bash
npm i -g pnpm yarn eslint prettier typescript nodemon nrm @anthropic-ai/claude-code
```

### 10. 验证

```powershell
# 环境变量
$env:NVM_HOME           # $NVM_ROOT
$env:NVM_SYMLINK        # $NVM_ROOT\nodejs
$env:NODE_PATH           # $NVM_ROOT\nodejs\node_modules

# nvm 和 node
nvm version              # 1.2.2
nvm list                 # 列出已装版本，* 为当前激活
node --version           # 与 nvm list 中激活版本一致（如 v22.20.0）
npm --version            # 随 Node 附带

# 全局包
pnpm --version           # 版本号
yarn --version           # 版本号
claude --version         # 版本号
```

## 最终目录结构

```
$NVM_ROOT\
├── nvm.exe                      ← nvm 本体
├── settings.txt                 ← nvm 配置
├── node_cache\                  ← npm 缓存（固定目录）
├── node_global\                 ← npm 全局包（固定目录，⭐ 跨版本保留）
│   ├── pnpm / pnpm.cmd
│   ├── yarn / yarn.cmd
│   ├── claude / claude.cmd
│   ├── tsc / tsserver
│   ├── eslint / eslint.cmd
│   ├── prettier / prettier.cmd
│   ├── nodemon / nodemon.cmd
│   ├── nrm / nrm.cmd
│   └── node_modules\
├── v22.20.0\                    ← Node 22 真实文件（版本号以 nvm list 为准）
├── v18.x.x\                     ← 其他 Node 版本（可选）
└── nodejs\  → v22.20.0\         ← NVM_SYMLINK（符号链接，nvm use 重建）
```

## 环境变量

| 变量 | 级别 | 值 | 用途 |
|------|------|-----|------|
| `NVM_HOME` | Machine | `$NVM_ROOT` | nvm.exe 和 Node 版本目录 |
| `NVM_SYMLINK` | Machine | `$NVM_ROOT\nodejs` | 指向当前激活版本的符号链接 |
| `NODE_PATH` | Machine | `$NVM_ROOT\nodejs\node_modules` | Node.js 模块搜索路径 |
| `Path`（用户） | User | `%NVM_HOME%;%NVM_SYMLINK%;...\node_global` | 命令行可执行文件查找 |

## 关键设计决策（为什么这样配置）

### node_global 不放在 symlink 内

文章的常规做法是把 `node_global` 放在 `nodejs` 目录里，但 `nodejs` 是符号链接，`nvm use` 切换版本时会改变指向，导致全局包丢失。

我们的方案把 `node_global` 和 `node_cache` 放在 NVM 根目录（symlink 外面），npm prefix 指向固定路径，切换 Node 版本后全局包完好无损。

### 使用绝对路径而非 %VAR% 格式

环境变量用 `%NVM_HOME%` 格式写入 PATH 更优雅，但部分 Windows 环境下展开可能失败。如果遇到 `nvm` 命令找不到的情况，可以改为绝对路径：

```
C:\Programs\Nvm           ← 替代 %NVM_HOME%
C:\Programs\Nvm\nodejs    ← 替代 %NVM_SYMLINK%
```

### 镜像源

所有下载走国内镜像：
- `node_mirror` / `npm_mirror` → `npmmirror.com/mirrors/`（nvm install 用）
- `registry` → `registry.npmmirror.com`（npm install 用）

## 注意事项

- **版本固定在 1.2.2**：2.x 只发 `setup.exe`，没有免安装 zip；想要“全部收进一个根目录”就别升级
- `nvm use` **需要管理员权限或开发者模式**（要重建 `nodejs` 符号链接）
- `NVM_HOME` 和 `NVM_SYMLINK` **不能**设为同一个目录——`NVM_SYMLINK` 会被 nvm 转为符号链接，与 nvm.exe 所在目录冲突
- 设置环境变量后**必须重启终端**才能生效
- 安装 Node 版本后 `nodejs` 目录才会出现（由 nvm 自动创建符号链接）
- `settings.txt` 的编码必须是 **ASCII 或 UTF-8 without BOM**，否则 nvm 可能读取失败
- nvm-windows 1.2.2 没有 `nvm alias` 命令，使用 `nvm use` 即为设置默认版本
- `nvm use 22` 与 `nvm use 22.20.0` 等价（前者选已装的最高 22.x），不要写死具体补丁号
