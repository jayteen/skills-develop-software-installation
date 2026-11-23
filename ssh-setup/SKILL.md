---
name: ssh-setup
description: 在新电脑上生成 SSH 密钥并配置 ~/.ssh/config，支持多平台（GitHub、GitLab 等）
disable-model-invocation: true
---

# SSH 密钥生成与多平台配置

在新电脑上从零配置 SSH 密钥，支持同时连接多个 Git 托管平台。

## 使用方式

用户调用时需提供平台信息，例如：`/ssh-setup github gitlab`

如果用户未提供信息，询问用户需要配置哪些平台（GitHub、GitLab、Gitee 等），以及对应的邮箱和 HostName。

## 步骤

### 1. 收集信息

为每个平台确认以下信息：

| 字段 | 说明 | 示例 |
|------|------|------|
| 平台名称 | 用作 Host 别名和文件名后缀 | `github`、`gitlab_company` |
| HostName | 域名或 IP（不带协议前缀） | `github.com`、`192.168.1.100` |
| Port | SSH 端口，默认 22 可省略 | `22`、`2222` |
| 邮箱 | 用于密钥注释 | `user@example.com` |

### 2. 生成密钥对

为每个平台生成独立密钥。**默认用 ed25519**（更短、更快、更安全，GitHub/GitLab/Gitee 均支持）：

```bash
ssh-keygen -t ed25519 -C "your_email@example.com" -f ~/.ssh/id_ed25519_github
ssh-keygen -t ed25519 -C "your_email@example.com" -f ~/.ssh/id_ed25519_gitlab_company
```

- `-t ed25519`：Ed25519 算法（推荐）
- `-C`：注释，通常填邮箱
- `-f`：指定输出路径，命名规则 `id_ed25519_<平台别名>`
- passphrase 提示时直接回车跳过（或按需设置）

> 只有在对方服务器**明确不支持 ed25519**（老版本 OpenSSH、部分企业 GitLab）时，才退回 RSA 4096：
>
> ```bash
> ssh-keygen -t rsa -b 4096 -C "your_email@example.com" -f ~/.ssh/id_rsa_<平台别名>
> ```

每次生成产出两个文件：

- `id_ed25519_<name>` — 私钥（不可泄露）
- `id_ed25519_<name>.pub` — 公钥（添加到平台）

### 3. 将公钥添加到平台

查看公钥：

```bash
cat ~/.ssh/id_ed25519_github.pub
```

复制全部输出，到对应平台添加：

- **GitHub**：Settings → SSH and GPG keys → New SSH key
- **GitLab**：Preferences → SSH Keys → Add new key
- **Gitee**：设置 → SSH公钥 → 添加公钥

### 4. 编写 ~/.ssh/config

有两种 Host 配置方式，根据使用习惯选择：

#### 方式 A：Host 使用自定义别名

`Host` 填自定义名称，克隆时需要将原始地址中的域名替换为别名。

```
# github
Host github
    HostName github.com
    User git
    IdentityFile C:/Users/<用户名>/.ssh/id_ed25519_github
    IdentitiesOnly yes

# gitlab company
Host gitlab_company
    HostName 192.168.1.100
    Port 2222
    User git
    IdentityFile C:/Users/<用户名>/.ssh/id_ed25519_gitlab_company
    IdentitiesOnly yes
```

克隆时需替换域名为别名：

```bash
# 原始地址
git clone git@github.com:user/repo.git
# 使用别名
git clone git@github:user/repo.git

# 原始地址（带端口）
git clone ssh://git@192.168.1.100:2222/group/repo.git
# 使用别名
git clone git@gitlab_company:group/repo.git
```

#### 方式 B：Host 使用实际域名/IP（推荐）

`Host` 直接填实际域名或 IP，克隆时直接使用平台给出的原始地址，无需任何修改。

```
# github
Host github.com
    HostName github.com
    User git
    IdentityFile C:/Users/<用户名>/.ssh/id_ed25519_github
    IdentitiesOnly yes

# gitlab company
Host 192.168.1.100
    HostName 192.168.1.100
    Port 2222
    User git
    IdentityFile C:/Users/<用户名>/.ssh/id_ed25519_gitlab_company
    IdentitiesOnly yes
```

克隆时直接用原始地址：

```bash
git clone git@github.com:user/repo.git
git clone git@192.168.1.100:group/repo.git
```

### 5. config 格式规则

- 每个平台一个 `Host` 块
- `Host` — 匹配规则：SSH 连接时根据目标地址匹配对应的 Host 块
- `HostName` — 纯域名或 IP，**不带 `http://` 或 `https://`**
- `Port` — SSH 端口，默认 22 时可省略整行
- `User` — 固定写 `git`
- `IdentityFile` — 私钥绝对路径；`<用户名>` 要换成真实 Windows 用户名（中文用户名照写）
- `IdentitiesOnly yes` — **建议都加**：只用本块的私钥，避免 agent 里其它 key 抢先尝试导致 `Permission denied`
- 注释只能用 `#` 独占一行，**不支持 `//` 行内注释**
- 关键字和值之间用空格分隔，值不需要引号

#### 可选：Windows 启用 ssh-agent（私钥带 passphrase 时）

Windows 自带的 OpenSSH 客户端可启用 agent 缓存口令，免去每次输入：

```powershell
# 管理员 PowerShell
Set-Service ssh-agent -StartupType Automatic
Start-Service ssh-agent

# 普通用户
ssh-add "$env:USERPROFILE\.ssh\id_ed25519_github"
```

也可以在对应 Host 块里加 `AddKeysToAgent yes`，首次使用后自动加入 agent。

### 6. 验证连接

测试命令取决于 Host 的配置方式：

```bash
# 方式 A（自定义别名）
ssh -T git@github
ssh -T git@gitlab_company

# 方式 B（实际域名/IP）
ssh -T git@github.com
ssh -T git@192.168.1.100
```

规则：`ssh -T git@<Host值>`，即 config 中 `Host` 后面写的是什么，测试时就用什么。

成功输出示例：
```
Hi username! You've successfully authenticated, but GitHub does not provide shell access.
```

## 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| `extra arguments at end of line` | 行内写了 `//` 注释 | 删除行内注释，用 `#` 独占一行 |
| `Could not resolve hostname` | Host 别名拼写错误 | 检查拼写，注意下划线与横杠区别 |
| `Connection closed by port X` | 端口不是 SSH 端口 | 到平台页面 Clone with SSH 确认实际端口 |
| `Permission denied (publickey)` | 公钥未添加或私钥路径错误 | 检查 IdentityFile 路径；确认 .pub 已添加到平台 |
| `Too many authentication failures` | agent 里 key 太多，逐个尝试被服务器拒绝 | 给 Host 块加 `IdentitiesOnly yes` |
| `Host key verification failed` | 首次连接未确认指纹 | 手动 `ssh -T <host>` 输入 yes 确认 |

## 注意事项

- Windows 路径在 config 中使用正斜杠 `/` 或反斜杠 `\` 均可
- 如果同一平台有多个账号，用不同的 Host 别名区分（如 `github_work`、`github_personal`）
- 确认平台 SSH 端口的方法：打开项目页面 → Clone → SSH，查看给出的地址格式
- 私钥文件权限过大会被拒绝，Linux/Mac 需要 `chmod 600 ~/.ssh/id_ed25519_*`
- **新电脑用 ed25519**；已有 RSA 密钥的旧电脑不必重做，继续用即可
- 私钥路径里的用户名是**本机**用户名，别照抄教程里的 `C:/Users/xxx`
- 生成时如果 `~/.ssh` 不存在，`ssh-keygen` 会自动创建；权限建议 `700`
