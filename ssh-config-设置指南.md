# SSH Config 配置文件指南

## SSH 密钥认证原理

SSH 密钥认证基于**非对称加密**（公钥密码学），使用一对密钥：公钥和私钥。

### 核心概念

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   客户端（你的电脑）                    服务器（远程主机）    │
│                                                             │
│   ┌──────────────┐                   ┌──────────────┐      │
│   │   私钥       │                   │   公钥       │      │
│   │ (private key)│                   │ (public key) │      │
│   │   保密！     │                   │   可以公开   │      │
│   └──────────────┘                   └──────────────┘      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

| 密钥类型 | 保存位置 | 特点 |
|----------|----------|------|
| 私钥 | 客户端 `~/.ssh/id_ed25519` | ⚠️ 绝对不能泄露 |
| 公钥 | 服务器 `~/.ssh/authorized_keys` | ✅ 可以公开 |

### 认证流程

```
第1步：客户端发起连接请求
       │
       ▼
第2步：服务器发送一个随机挑战（challenge）
       │
       ▼
第3步：客户端用【私钥】对挑战进行签名
       │
       ▼
第4步：服务器用【公钥】验证签名
       │
       ▼
第5步：验证成功 → 允许登录
```

### 为什么安全？

| 问题 | 答案 |
|------|------|
| 私钥泄露会怎样？ | 攻击者可以冒充你登录所有存有你公钥的服务器 |
| 公钥泄露会怎样？ | 没关系，公钥本来就是公开的 |
| 为什么不能反向伪造？ | 用公钥无法推导出私钥（数学上的单向函数） |

### 与密码登录的对比

| 特性 | 密码登录 | 密钥登录 |
|------|----------|----------|
| 认证方式 | 知道密码即可 | 必须持有私钥文件 |
| 暴力破解难度 | 较低（容易尝试） | 几乎不可能 |
| 中间人攻击风险 | 存在 | 较低 |
| 便利性 | 每次输入密码 | 配置后无需输入 |

### 密钥生成算法

| 算法 | 特点 | 推荐度 |
|------|------|--------|
| **ED25519** | 速度快，密钥短，安全性高 | ⭐⭐⭐⭐⭐ |
| RSA | 兼容性好，但密钥较长 | ⭐⭐⭐ |
| ECDSA | 安全性好，但依赖随机数生成器 | ⭐⭐⭐⭐ |

**推荐命令**：
```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519
```

---

## 文件位置

Windows: `C:\Users\<用户名>\.ssh\config`
Linux/Mac: `~/.ssh/config`

---

## 基本格式规范

```ssh
# 注释行以 # 开头

Host <模式>              # 主机匹配模式（可以是别名、域名、IP或通配符 * ?）
  指令名 值              # 配置项使用缩进（通常是2个空格）
  指令名 值              # 每行一个配置项

Host 另一个主机
  指令名 值
```

---

## 常用配置指令

| 指令 | 作用 | 示例 |
|------|------|------|
| `Host` | 主机匹配模式（别名/通配符） | `myserver` 或 `*.example.com` |
| `HostName` | 实际主机名或IP | `192.168.1.100` |
| `User` | 登录用户名 | `ubuntu` |
| `Port` | SSH端口（默认22） | `2222` |
| `IdentityFile` | 私钥文件路径 | `~/.ssh/my_key` |
| `ServerAliveInterval` | 心跳间隔（秒），防止断连 | `60` |

---

## 完整配置示例

### 基础配置（密码登录）

```ssh
# 云服务器 - 生产环境
Host prod-server
  HostName 124.223.114.33
  User ubuntu
  Port 22
  ServerAliveInterval 60
```

### 密钥登录配置

```ssh
# 云服务器 - 生产环境
Host prod-server
  HostName 124.223.114.33
  User ubuntu
  Port 22
  IdentityFile ~/.ssh/id_rsa
  ServerAliveInterval 60
```

---

## IdentityFile 配置说明

### 是否必须配置？

取决于你的认证方式：

| 认证方式 | 是否需要配置 |
|----------|--------------|
| 密码登录 | ❌ 不需要 |
| 密钥登录 | ✅ 建议配置 |

### 如何判断当前使用哪种认证方式？

运行以下命令查看详细日志：

```bash
ssh -v ubuntu@124.223.114.33
```

查看输出中的关键信息：

| 日志内容 | 认证方式 |
|----------|----------|
| `Authenticated ... using "password"` | 密码登录 |
| `Authentication succeeded (publickey)` | 密钥登录 |

### 默认私钥文件

SSH 会自动尝试以下默认私钥文件（无需配置）：
- `~/.ssh/id_rsa`
- `~/.ssh/id_ed25519`
- `~/.ssh/id_ecdsa`
- `~/.ssh/id_dsa`

如果你的私钥文件名不是以上默认名称，**必须**在配置中指定：

```ssh
IdentityFile C:/Users/29580/.ssh/aliyun_key
```

---

## ServerAliveInterval 配置说明

### 作用

防止长时间无操作时 SSH 连接被防火墙或服务器断开。

### 是否必须？

❌ 非必须，但**强烈推荐**

### 推荐配置

```ssh
ServerAliveInterval 60    # 每60秒发送一次心跳
```

### 适用场景

- 长时间运行的命令
- 使用 screen/tmux 会话
- 网络不稳定环境

---

## 从密码登录切换到密钥登录

### 完整操作步骤

#### 步骤 1：检查现有密钥

首先检查是否已经有SSH密钥：

```bash
# Windows (Git Bash)
ls -la ~/.ssh

# Linux/Mac
ls -la ~/.ssh
```

如果看到 `id_ed25519`、`id_rsa` 等文件，说明已有密钥，可以跳过生成步骤。

#### 步骤 2：生成密钥对

```bash
# 生成 ED25519 密钥对（推荐）
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N "" -C "your_email@example.com"

# 参数说明：
# -t ed25519     : 使用 ED25519 算法
# -f             : 指定文件名
# -N ""          : 不设置密码（空密码）
# -C             : 添加注释（通常是邮箱）
```

生成的文件：
| 文件 | 说明 |
|------|------|
| `~/.ssh/id_ed25519` | 私钥（保密！） |
| `~/.ssh/id_ed25519.pub` | 公钥（可以公开） |

#### 步骤 3：复制公钥到服务器

**方法一：使用 ssh-copy-id（Linux/Mac）**

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@124.223.114.33
```

**方法二：手动复制（Windows）**

```bash
# 一行命令完成（推荐）
cat ~/.ssh/id_ed25519.pub | ssh ubuntu@124.223.114.33 'mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && echo "公钥已添加"'
```

**方法三：手动复制（分步操作）**

```bash
# 1. 查看公钥内容
cat ~/.ssh/id_ed25519.pub

# 2. 登录服务器
ssh ubuntu@124.223.114.33

# 3. 在服务器上执行
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
# 将公钥内容粘贴进去，保存退出
chmod 600 ~/.ssh/authorized_keys
```

#### 步骤 4：更新 SSH 配置文件

在 `~/.ssh/config` 中添加 `IdentityFile` 配置：

```ssh
Host 124.223.114.33
  HostName 124.223.114.33
  User ubuntu
  Port 22
  IdentityFile ~/.ssh/id_ed25519
  ServerAliveInterval 60
```

#### 步骤 5：测试密钥登录

```bash
# 方式1：使用 BatchMode 测试（不使用密码）
ssh -o BatchMode=yes -o ConnectTimeout=10 ubuntu@124.223.114.33 "echo '密钥登录成功！'"

# 方式2：直接连接测试
ssh ubuntu@124.223.114.33
```

如果不需要输入密码即可登录，说明配置成功！

#### 步骤 6：禁用密码登录（可选，提高安全性）

⚠️ **注意**：确保密钥登录成功后再执行此步骤！

登录到服务器，编辑 SSH 配置：

```bash
sudo nano /etc/ssh/sshd_config
```

修改以下配置：

```ssh
PasswordAuthentication no    # 禁用密码登录
PubkeyAuthentication yes     # 启用公钥登录
```

重启 SSH 服务：

```bash
# Ubuntu/Debian
sudo systemctl restart sshd

# CentOS/RHEL
sudo systemctl restart sshd
```

---

## 使用别名简化连接

配置别名后，可以用更短的命令连接：

```bash
# 不使用配置
ssh ubuntu@124.223.114.33

# 使用别名
ssh prod-server
```
