---
name: "vps-ssh-hardening"
description: "加固新购 VPS 的 SSH：修改默认 SSH 端口、创建专用 sudo 用户、强制密钥登录（禁用密码、禁用 root）。支持 CentOS/RHEL、Ubuntu 和 Debian。新购 VPS 设置 SSH 或需要「SSH 加固」时使用。"
---

# VPS SSH 加固

## 用途

把刚买的 VPS 从不安全的默认状态（root + 密码，22 端口）加固为：

- SSH 使用非标准端口
- 禁止 root SSH 登录
- 禁用密码认证——只允许密钥登录
- 一个专用的 sudo 用户作为唯一入口
- 每个客户端设备一对密钥；私钥永不离开本机

**范围：** 仅 SSH 登录加固。代理面板、通用防火墙策略、TCP 调优不在范围内。

**可移植性：** 本文件为纯 Markdown，无 agent 专有语法。交给 Claude、Codex、Gemini、Muse、Grok 或任何能 shell 操作服务器的 agent 都一样可用。

## 使用方法

你是 agent。步骤带有标签：

- `[AGENT]` —— 你在服务器上通过 shell/SSH 执行。
- `[USER]` —— 用户在自己的机器上执行。把精确命令给他们；你自己绝不执行。
- `[ASK]` —— 停下来，先拿到用户的明确答复。绝不编造端口、用户名、密钥备注或任何值。

**铁律（绝不违反——每一条都是有人被锁在门外换来的）：**

1. 在**独立窗口**验证新密钥登录成功之前，**绝不关闭最初的工作会话**。它是唯一的救生通道。
2. 编辑前**备份** `/etc/ssh/sshd_config`。
3. 每次 `systemctl restart sshd` **之前**，先用 `sshd -t` 和 `sshd -T` **验证**。
4. **绝不在聊天中索要、接收或经手私钥、passphrase 或密码。** 公钥随便看；私钥材料不行。
5. **顺序不能错：** 先在服务端启用 `PubkeyAuthentication`，**再**测试密钥登录。顺序反了必失败——服务端根本就没在接受密钥。
6. 任何验证步骤失败，**停下来**。不许进入下一步。

## Step 0 — 收集信息 [ASK]

动手之前，把下面**全部**问清楚。不许靠猜。

1. **VPS 访问方式**：IP 地址 + 现在怎么登（云控制台？root 密码？服务商预置密钥？）。你需要一个能用的 shell 起手。
2. **操作系统**：连上后跑 `grep -E "^ID=" /etc/os-release`，对照下面的差异表。
3. **新 SSH 端口**（如 `1234`）：1024–65535 之间的空闲端口，避开常用端口。跟用户确认云服务商安全组已放行——不确定就让他们去控制台开。
4. **新 sudo 用户名**（如 `test`）：不能是 `root`、`admin`，不要好猜的。
5. **密钥 passphrase**：建议设。用户在 `ssh-keygen` 时自己设；你永远看不到。
6. **客户端软件**（WindTerm / Termius / PuTTY / 原生 `ssh`）：决定 Step 6 给哪套界面指引。
7. **sudo 密码策略**：保留 `NOPASSWD`（方便；个人 VPS 推荐）还是每次 `sudo` 都要输账号密码（更严）？先跑 `sudo grep -rn NOPASSWD /etc/sudoers /etc/sudoers.d/ 2>/dev/null` 看现状并告知用户——不许悄悄改。
8. 确认：**彻底禁用 root SSH 登录**（`PermitRootLogin no`）？默认：是。

## 系统差异

| | CentOS / RHEL 7/8/9 | Ubuntu 20.04/22.04/24.04 | Debian 11/12 |
|---|---|---|---|
| sudo 组 | `wheel` | `sudo` | `sudo` |
| SSH 服务名 | `sshd` | `ssh` | `ssh` |
| SSH 认证日志 | `/var/log/secure` | `/var/log/auth.log` | `/var/log/auth.log` |
| 主机防火墙 | 通常没有——走服务商安全组（有 firewalld 就用它） | `ufw` | 默认没有 |
| SELinux | 经常是 Enforcing——见 Step 2 | 无 | 无 |

下文命令中，`<SUDO_GROUP>` = CentOS/RHEL 用 `wheel`，Ubuntu/Debian 用 `sudo`。`systemctl restart sshd` = Ubuntu/Debian 用 `systemctl restart ssh`。`/var/log/secure` = Ubuntu/Debian 用 `/var/log/auth.log`。

## Step 1 — 创建 sudo 用户 [AGENT]

```bash
useradd -m -s /bin/bash <用户名>
usermod -aG <SUDO_GROUP> <用户名>
id <用户名>
# 期望：uid=...(<用户名>) gid=... groups=...(<wheel|sudo>)
```

验证 sudo 权限：

```bash
sudo -l -U <用户名>
# 期望看到类似：(ALL) ALL   ——带或不带 NOPASSWD:
```

## Step 2 — 修改 SSH 端口 [AGENT]

```bash
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
sed -i 's/^#*Port.*/Port <新端口>/' /etc/ssh/sshd_config
sshd -t && echo "语法OK"
sshd -T | grep -E "^port"
# 期望：port <新端口>
```

**SELinux（仅 CentOS/RHEL）：** `getenforce` 如果输出 `Enforcing`，新端口必须打标签，否则 sshd 绑定不上：

```bash
semanage port -a -t ssh_port_t -p tcp <新端口>
# （没有 semanage 就装：yum install -y policycoreutils-python-utils）
```

**防火墙：** 在所有有过滤的地方放行——
服务商安全组（让用户确认），本机有防火墙的话再加一条：

```bash
firewall-cmd --permanent --add-port=<新端口>/tcp && firewall-cmd --reload   # firewalld
ufw allow <新端口>/tcp                                                       # Ubuntu ufw
```

然后 `systemctl restart sshd`（上面已验证过），**立刻用新端口重连当前会话**，确认通了再继续。这个会话保持开着。

## Step 3 — 生成密钥对 [USER]

在**用户自己的机器上**（绝不在服务器上）：

```bash
ssh-keygen -t ed25519 -C "<用户名>-<设备>" -f ~/.ssh/<用户名>_vps
```

- `-C` 是备注（比如 `test-mac`），以后靠它认出哪把钥匙是谁的。一定要写。
- passphrase：建议设（见 Step 0 第 5 条）。输入时屏幕上什么都不显示——正常。
- 结果：`~/.ssh/<用户名>_vps`（**私钥——永不离开本机，绝不发给任何人包括你**），`~/.ssh/<用户名>_vps.pub`（公钥——随便看）。

## Step 4 — 安装公钥 [AGENT]

在最初的（还能用的）会话里，以 root 执行：

```bash
mkdir -p /home/<用户名>/.ssh
cat >> /home/<用户名>/.ssh/authorized_keys
# 粘贴 .pub 内容，回车，Ctrl+D 结束
chmod 700 /home/<用户名>/.ssh
chmod 600 /home/<用户名>/.ssh/authorized_keys
chown -R <用户名>:<用户名> /home/<用户名>/.ssh
```

验证（属主和权限才是真正决定登录成败的）：

```bash
ls -la /home/<用户名>/.ssh/
# 期望：drwx------ <用户名>  和  -rw------- <用户名> authorized_keys
ssh-keygen -l -f /home/<用户名>/.ssh/authorized_keys
# 期望：每把钥匙一行，能看到 -C 备注
```

## Step 5 — 在服务端启用公钥认证 [AGENT]

**测试登录之前**必须先做这步：

```bash
sed -i 's/^#*PubkeyAuthentication.*/PubkeyAuthentication yes/' /etc/ssh/sshd_config
sshd -t && echo "语法OK"
sshd -T | grep -i pubkeyauthentication
# 期望：pubkeyauthentication yes
systemctl restart sshd
```

> 这步不许跳过、不许调换顺序。服务端的 `pubkeyauthentication` 还是 `no` 时，钥匙再对也登不进去。

## Step 6 — 配置客户端 [USER]

按用户的客户端软件给指引：

**WindTerm（macOS / Windows）：**
1. 新建会话 → 主机填 VPS IP，端口填新 SSH 端口，用户名填 `<用户名>`。
2. SSH → Authentication → 身份验证文件 → `...` → 选**私钥**（*没有* `.pub` 后缀的那个）。
   - macOS 文件选择框默认隐藏 `~/.ssh`：按 `Cmd+Shift+.` 显示隐藏文件，或 `Cmd+Shift+G` 输入 `~/.ssh` 跳转。
   - 路径必须是**绝对路径**（以 `/` 开头）。少了开头的 `/` 会悄无声息地认证失败。
3. 测试阶段：暂时**取消勾选**"尝试密码认证"，只留公钥认证——这样失败就是真失败，不会被密码兜底掩盖。
4. 连接 → 提示输入密码时，输的是**密钥 passphrase**（Step 3 设的那个），*不是*服务器账号密码。

**Termius（iOS / Android）：** 在 App 的钥匙串里*本机*生成或导入密钥对，绑定到主机条目。

**原生 ssh：** `ssh -i ~/.ssh/<用户名>_vps -p <新端口> <用户名>@<IP>`

**要讲清的概念：** *passphrase* 解锁的是私钥文件；*账号密码* 登录的是服务器账号。两码事。加固完成后，账号密码不再用于 SSH。

## Step 7 — 独立窗口验证密钥登录 [USER] + [AGENT]

用户新开一个客户端窗口，用密钥以 `<用户名>` 登录。**最初的会话保持开着。**

然后 [AGENT] 在服务端确认：

```bash
grep "Accepted" /var/log/secure | tail -3    # CentOS/RHEL
grep "Accepted" /var/log/auth.log | tail -3  # Ubuntu/Debian
# 期望：Accepted publickey for <用户名> ... ED25519 SHA256:<指纹>
```

**只有看到 `Accepted publickey` 才能进下一步。** 失败就看故障排查——不许进 Step 8。

## Step 8 — 禁用密码和 root 登录 [AGENT]

Step 7 成功之后才能做：

```bash
sed -i 's/^#*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sed -i 's/^#*ChallengeResponseAuthentication.*/ChallengeResponseAuthentication no/' /etc/ssh/sshd_config
sshd -t && echo "语法OK"
sshd -T | grep -Ei "permitrootlogin|passwordauthentication|pubkeyauthentication"
# 期望：no / no / yes
systemctl restart sshd
```

最终验证（最初会话还开着，用新窗口测）：
- `<用户名>` + 密钥 → ✅ 能进
- `<用户名>` + 密码 → ❌ 被拒（预期）
- `root` + 任何方式 → ❌ 被拒（预期）

然后最初会话就可以关了。日常管理：以 `<用户名>` 登录，`sudo -i` 拿 root shell。

**回滚**（用还开着的最初会话，万一出问题）：

```bash
cp /etc/ssh/sshd_config.bak /etc/ssh/sshd_config && systemctl restart sshd
```

## 日常密钥管理

- **新设备：** 在*那台设备本机*生成自己的密钥对（`ssh-keygen -t ed25519 -C "<用户名>-<设备>"`）。把**公钥**追加为 `/home/<用户名>/.ssh/authorized_keys` 的新一行。**绝不在设备间复制私钥**——丢一台就得全换；一机一钥匙的话，删一行就行。
- **列出钥匙：** `ssh-keygen -l -f /home/<用户名>/.ssh/authorized_keys`（靠 `-C` 备注认出每把是谁的）。
- **吊销钥匙：** `sed -i '/<备注或key-id>/d' /home/<用户名>/.ssh/authorized_keys`，再列一遍确认。
- **备份私钥：** 加密的异地备份（密码管理器 / 加密 U 盘）。同盘复制只能防误删。绝不明文存聊天记录、邮件、网盘。

## 故障排查

| 现象 | 原因 | 修法 |
|---|---|---|
| 密钥登录失败，客户端回落到密码框 | 服务端 `PubkeyAuthentication` 还是 `no`（Step 5 没做） | 做 Step 5，重启 sshd，重试 |
| 选了密钥仍"验证失败" | 选错文件（选了 `.pub`）或路径缺了开头的 `/` | 重选私钥；确认绝对路径 |
| 服务端日志 `Failed publickey` | `authorized_keys` 内容断行/损坏，或属主权限不对 | 客户端跑 `ssh-keygen -y -f <私钥>`，输出必须跟服务端某一行完全一致；权限改回 700/600 + 正确属主 |
| 日志只有 `Failed password`，没有 `publickey` 尝试 | 客户端根本没出示钥匙 | 检查客户端认证设置：方法勾了没、钥匙文件挂了没、用户名对不对 |
| 新端口 `Connection refused/timed out` | 安全组/主机防火墙没放行，或 SELinux 拦截（CentOS） | 控制台放行 `<新端口>`；Enforcing 的话 `semanage port -a -t ssh_port_t -p tcp <新端口>` |
| 彻底锁死 | 验证密钥登录成功之前就关了密码 | 服务商控制台/VNC 进去 → 恢复 `/etc/ssh/sshd_config.bak` → 重启 sshd。**这就是最初会话要开到最后的原因。** |

## 操作规范

1. 绝不在聊天中索要、接收私钥、passphrase、账号密码。公钥随便看。
2. 端口、用户名、密钥备注、系统版本——没问清不许编，先 `[ASK]` / 先检测。
3. 验证永远用独立会话，绝不关闭或复用最初会话。
4. 改 `sshd_config` 必先备份 `.bak`，重启前必 `sshd -t` + `sshd -T` 验证。
5. 每次 `sed` 完都 `grep` 对行，再用 `sshd -T` 确认——`sed` 没匹配到是悄无声息的。
6. 任何验证失败就停，不许进下一步。
7. 改 sudoers 用 `visudo`（带语法检查）。绝不用 `sed` 手改 sudoers。
8. 查到 `NOPASSWD` 要告知用户，不许悄悄改 sudo 密码策略。

## 输出约定

Skill 完成时报告：

- 新 SSH 端口、登录用户名、已授权钥匙的指纹和备注
- 生效配置：`sshd -T` 里 `port`、`permitrootlogin`、`pubkeyauthentication`、`passwordauthentication` 的值
- 最后一条 `Accepted publickey` 日志作为证据
- 私钥在哪，确认已建议加密异地备份
- 刻意没动的东西（如保留了 `NOPASSWD`、保留了某台旧设备钥匙）及原因
