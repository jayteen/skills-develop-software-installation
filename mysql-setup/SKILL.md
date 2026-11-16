---
name: mysql-setup
description: 在新电脑上安装并配置 MySQL，将所有数据统一到一个根目录下管理
disable-model-invocation: true
---

# MySQL 全局安装与配置

在新电脑上从零配置 MySQL，将所有 MySQL 相关文件（程序 + 数据 + 配置）统一到一个根目录下。

## 使用方式

用户调用时需提供根目录路径，例如：`/mysql-setup C:\Programs\Mysql`

如果用户未提供路径，询问用户希望将 MySQL 安装到哪个目录。

## 步骤

### 1. 确认根目录

将用户提供的路径作为 `$MYSQL_ROOT`（以下步骤中用 `$MYSQL_ROOT` 代替实际路径）。

### 2. 下载并解压 MySQL ZIP

```bash
mkdir -p "$MYSQL_ROOT"
cd "$MYSQL_ROOT"
curl -L -o mysql.zip "https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-8.0.40-winx64.zip"
unzip -q mysql.zip
# 将解压内容上移一层，删除中间目录
mv mysql-8.0.*-winx64/* .
rmdir mysql-8.0.*-winx64
rm mysql.zip
```

### 3. 创建配置文件 my.ini

在 `$MYSQL_ROOT\my.ini` 中写入：

```ini
[mysqld]
basedir=$MYSQL_ROOT
datadir=$MYSQL_ROOT/data
port=3306
character-set-server=utf8mb4
default-storage-engine=INNODB
max_connections=200
max_connect_errors=10

[mysql]
default-character-set=utf8mb4

[client]
port=3306
default-character-set=utf8mb4
```

注意：`basedir` 和 `datadir` 使用正斜杠 `/`，Windows 路径盘符需写为 `C:/` 格式。

### 4. 初始化数据库

```bash
mkdir -p "$MYSQL_ROOT/data"
$MYSQL_ROOT/bin/mysqld --initialize --console
```

**务必记录输出的临时 root 密码**，例如：`A temporary password is generated for root@localhost: xxxxxx`

### 5. 安装 Windows 服务（管理员 PowerShell）

```powershell
Start-Process powershell -ArgumentList '-Command `
    $MYSQL_ROOT\bin\mysqld --install MySQL80 --defaults-file="$MYSQL_ROOT\my.ini"' `
    -Verb RunAs -Wait
```

### 6. 启动服务（管理员 PowerShell）

```powershell
Start-Process powershell -ArgumentList '-Command sc.exe start MySQL80' -Verb RunAs -Wait
```

验证服务状态：

```powershell
Get-Service MySQL80
```

### 7. 修改 root 密码

```bash
# 使用临时密码登录并修改密码
$MYSQL_ROOT/bin/mysql -u root -p'临时密码' --connect-expired-password -e "ALTER USER 'root'@'localhost' IDENTIFIED BY '新密码'; FLUSH PRIVILEGES;"
```

### 8. 保存凭证

将连接信息保存到 `$MYSQL_ROOT\credentials.txt`：

```
MySQL 连接信息
==============
主机: localhost
端口: 3306
用户名: root
密码: <新密码>
```

### 9. 配置系统 PATH（管理员 PowerShell）

```powershell
$path = [Environment]::GetEnvironmentVariable("Path", "Machine")
if ($path -notlike "*$MYSQL_ROOT\bin*") {
    $newPath = "$MYSQL_ROOT\bin;$path"
    [Environment]::SetEnvironmentVariable("Path", $newPath, "Machine")
}
```

重启终端后，`mysql` 命令即可全局使用。

### 10. 验证

```bash
mysql -u root -p -e "SELECT VERSION(); SHOW DATABASES;"
```

## 最终目录结构

```
$MYSQL_ROOT\
├── bin\                      ← MySQL 可执行文件（加 PATH）
├── lib\                      ← 库文件
├── share\                    ← 共享文件
├── include\                  ← 头文件
├── docs\                     ← 文档
├── my.ini                    ← 配置文件
├── data\                     ← 数据库数据文件
└── credentials.txt           ← 连接凭证
```

## 常用管理命令

```bash
# 启动服务
net start MySQL80

# 停止服务
net stop MySQL80

# 查看服务状态
Get-Service MySQL80

# 移除服务
mysqld --remove MySQL80
```

## 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| `Install/Remove of the Service Denied!` | 权限不足 | 用管理员 PowerShell 安装服务 |
| `Public Key Retrieval is not allowed` | MySQL 8.0 默认 `caching_sha2_password` | DBeaver 驱动属性中 `allowPublicKeyRetrieval=true`，`useSSL=false` |
| 服务无法启动 | 数据目录损坏或端口占用 | 检查 `$MYSQL_ROOT/data/*.err` 错误日志；`netstat -ano \| findstr 3306` 查端口 |
| `Access denied` | 密码错误或已过期 | 使用 `--connect-expired-password` 参数或重新初始化 |

## 注意事项

- `my.ini` 中路径使用正斜杠 `/`，不要用反斜杠，否则可能导致启动失败
- 安装服务和修改 Machine PATH 需要管理员权限
- 初始化时必须在终端记录临时密码，错过只能重新初始化（删除 data 目录重来）
- MySQL 8.0 使用 `caching_sha2_password` 认证，部分旧客户端需要调整驱动属性
- 下载 URL 中的版本号（如 8.0.40）可能更新，去 https://dev.mysql.com/downloads/mysql/ 确认最新版本
