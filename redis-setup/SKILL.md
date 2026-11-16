---
name: redis-setup
description: 在新电脑上安装并配置 Redis（Windows 原生移植版），将所有数据统一到一个根目录下管理
disable-model-invocation: true
---

# Redis 全局安装与配置（Windows）

在新电脑上从零配置 Redis，将所有 Redis 相关文件（程序 + 数据 + 配置）统一到一个根目录下。

> **注意：** Redis 官方不支持 Windows。本技能使用 [tporadowski/redis](https://github.com/tporadowski/redis) 维护的 Windows 原生移植版本（基于 Redis 5.0.14），是目前最成熟的 Windows Redis 方案。

## 使用方式

用户调用时需提供根目录路径，例如：`/redis-setup C:\Programs\Redis`

如果用户未提供路径，询问用户希望将 Redis 安装到哪个目录。

## 步骤

### 1. 确认根目录

将用户提供的路径作为 `$REDIS_ROOT`（以下步骤中用 `$REDIS_ROOT` 代替实际路径）。

### 2. 下载并解压 Redis ZIP

```bash
mkdir -p "$REDIS_ROOT"
cd "$REDIS_ROOT"
curl -L -o redis.zip "https://github.com/tporadowski/redis/releases/download/v5.0.14.1/Redis-x64-5.0.14.1.zip"
unzip -q redis.zip
rm redis.zip
```

### 3. 创建配置文件

编辑 `$REDIS_ROOT\redis.windows-service.conf`，找到以下行并修改：

```bash
# 设置访问密码（取消注释并修改）
# requirepass foobared  →  requirepass <密码>

# 建议同时修改：
# bind 127.0.0.1        ← 默认只监听本地，如需远程访问请注释此行
# port 6379             ← 默认端口
# logfile "server_log.txt"  ← 日志文件名
```

与 MySQL 不同，Redis ZIP 已自带配置文件，无需从零创建。

### 4. 安装 Windows 服务（管理员 PowerShell）

```bash
# 使用 redis-server.exe 自带的服务安装功能
powershell.exe -Command "Start-Process powershell -ArgumentList '-Command \"$REDIS_ROOT\redis-server.exe\" --service-install \"$REDIS_ROOT\redis.windows-service.conf\" --service-name Redis' -Verb RunAs -Wait"
```

注意：`--service-install` 后面紧跟的是配置文件路径（作为普通参数，不是 `--config` 选项）。

### 5. 启动服务（管理员 PowerShell）

```bash
powershell.exe -Command "Start-Process powershell -ArgumentList '-Command sc.exe start Redis' -Verb RunAs -Wait"
```

验证服务状态：

```bash
powershell.exe -Command "Get-Service Redis"
```

### 6. 测试连接

```bash
# 使用密码连接并测试
$REDIS_ROOT/redis-cli.exe -a <密码> ping
# 期望输出: PONG

# 查看服务器信息
$REDIS_ROOT/redis-cli.exe -a <密码> INFO SERVER
```

### 7. 保存凭证

将连接信息保存到 `$REDIS_ROOT\credentials.txt`：

```
Redis 连接信息
==============
主机: localhost
端口: 6379
密码: <密码>

连接命令: redis-cli -a <密码>
```

### 8. 配置系统 PATH（管理员 PowerShell）

```bash
powershell.exe -Command "Start-Process powershell -ArgumentList '-Command `$path = [Environment]::GetEnvironmentVariable(\"Path\", \"Machine\"); if (`$path -notlike \"*$REDIS_ROOT*\" ) { `$newPath = \"$REDIS_ROOT;`$path\"; [Environment]::SetEnvironmentVariable(\"Path\", `$newPath, \"Machine\") }' -Verb RunAs -Wait"
```

重启终端后，`redis-cli`、`redis-server` 等命令即可全局使用。

### 9. 验证

```bash
redis-cli -a <密码> INFO SERVER
redis-cli -a <密码> SET test "hello redis"
redis-cli -a <密码> GET test
```

## 最终目录结构

```
$REDIS_ROOT\
├── redis-server.exe          ← Redis 服务端
├── redis-cli.exe             ← Redis 客户端
├── redis-benchmark.exe       ← 性能测试工具
├── redis-check-aof.exe       ← AOF 修复工具
├── redis-check-rdb.exe       ← RDB 修复工具
├── redis.windows.conf        ← 普通运行配置
├── redis.windows-service.conf ← 服务运行配置（主要使用）
├── EventLog.dll              ← Windows 事件日志支持
├── credentials.txt           ← 连接凭证
├── 00-RELEASENOTES
├── README.txt
└── RELEASENOTES.txt
```

## 常用管理命令

```bash
# 启动服务
net start Redis

# 停止服务
net stop Redis

# 查看服务状态
powershell.exe -Command "Get-Service Redis"

# 卸载服务
redis-server --service-uninstall --service-name Redis

# 命令行连接
redis-cli -a <密码>

# 查看所有 key
redis-cli -a <密码> KEYS "*"

# 查看内存使用
redis-cli -a <密码> INFO MEMORY

# 查看客户端连接
redis-cli -a <密码> CLIENT LIST
```

## 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| `Install/Remove of the Service Denied!` | 权限不足 | 用管理员 PowerShell 安装服务 |
| `Could not connect to Redis at 127.0.0.1:6379` | 服务未启动 | `sc.exe start Redis` |
| `NOAUTH Authentication required` | 需要密码 | 使用 `-a <密码>` 参数或在连接后执行 `AUTH <密码>` |
| `Invalid argument during startup: unknown argument: config` | `--service-install` 后的配置文件参数格式不对 | 直接写配置文件路径，不要加 `--config` 前缀 |
| 服务启动后立即停止 | 配置文件路径错误或端口被占用 | 检查配置文件中路径是否正确；`netstat -ano \| findstr 6379` 查端口 |

## 与 MySQL 安装的区别

| 项目 | MySQL | Redis |
|------|-------|-------|
| 官方 Windows 支持 | 有 | 无（使用第三方移植） |
| 初始化步骤 | `mysqld --initialize` 生成临时密码 | 无需初始化 |
| 配置文件 | 手动创建 `my.ini` | ZIP 自带，直接修改 |
| 服务安装 | `mysqld --install` + `--defaults-file` | `redis-server --service-install` 直接跟配置文件路径 |
| 密码设置 | 初始化后 ALTER USER 修改 | 配置文件中 `requirepass` |
| 数据目录 | `data/`（需手动创建） | 无需额外创建，RDB/AOF 文件在根目录 |

## 注意事项

- Redis Windows 版不支持 fork 子进程，`bgsave` 和 `bgrewriteaof` 会阻塞主线程，大数据量时谨慎使用
- 生产环境建议在 WSL2 或 Linux 上运行官方 Redis
- Windows 版默认最大内存无限制，建议在配置文件中设置 `maxmemory` 限制
- 安装服务和修改 Machine PATH 需要管理员权限
- `redis.windows-service.conf` 和 `redis.windows.conf` 的区别：前者用于 Windows 服务模式，后者用于手动前台运行；建议统一使用 service 版本
