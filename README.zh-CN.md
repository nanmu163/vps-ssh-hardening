[English](README.md)

# VPS SSH 加固 Skill

一个可被 AI agent 直接执行的 skill：把新购 VPS 的 SSH 加固为非标准端口 + 专用 sudo 用户 + 纯密钥登录——无密码、无 root。

## 能做什么

把 VPS 从不安全的默认状态（root + 密码，22 端口）加固为：

- SSH 监听在你指定的**非标准端口**
- **禁止 root SSH 登录**
- **禁用密码认证**——只认 SSH 密钥
- 一个**专用 sudo 用户**作为唯一入口
- **一机一密钥对**；私钥永不离开本机

## 支持的系统

| 系统 | 版本 |
|---|---|
| CentOS / RHEL | 7、8、9 |
| Ubuntu | 20.04、22.04、24.04 |
| Debian | 11、12 |

Skill 会自动适配各系统差异（sudo 组名、SSH 服务名、认证日志路径、SELinux、防火墙）。

## 使用方法

1. 复制你语言版本的 skill 文件 raw 链接：
   - English: `SKILL.md`
   - 中文: `SKILL.zh-CN.md`

   （GitHub 上：打开文件 → **Raw** → 复制链接，例如
   `https://raw.githubusercontent.com/nanmu163/vps-ssh-hardening/main/SKILL.zh-CN.md`）
2. 把链接丢给你的 AI agent（Claude、Codex、Gemini、Muse、Grok……），说你要给新 VPS 做 SSH 加固。
3. Agent 会逐项问你需要的输入（IP、端口、用户名、客户端软件……）——如实回答，它不会自己编。
4. 按 agent 的指引完成你本机上的步骤（生成密钥、配置客户端）。服务端步骤由 agent 执行。

## 流程概览

| 步骤 | 内容 | 谁执行 |
|---|---|---|
| 0 | 收集信息（端口、用户名、系统、客户端软件……） | Agent 问，你答 |
| 1 | 创建 sudo 用户 | Agent |
| 2 | 修改 SSH 端口（含 SELinux / 防火墙） | Agent |
| 3 | 生成 ed25519 密钥对 | 你，在本机 |
| 4 | 把公钥装到服务器 | Agent |
| 5 | 在服务端启用公钥认证 | Agent |
| 6 | 配置 SSH 客户端（WindTerm / Termius / ssh） | 你，在本机 |
| 7 | 独立窗口验证密钥登录 | 你 + Agent |
| 8 | 禁用密码和 root 登录 | Agent |

每一步都带验证命令；任何检查失败 skill 都会停下来。

## 安全说明

- Agent **绝不**索要私钥、passphrase 或账号密码。公钥随便看。
- 最初的工作会话会一直开到新登录验证成功——这是内置的救生通道。
- 每次改 `sshd_config` 前都备份，重启前都用 `sshd -t` / `sshd -T` 验证。

## 文件说明

| 文件 | 说明 |
|---|---|
| `SKILL.md` | Skill 本体（英文） |
| `SKILL.zh-CN.md` | Skill 本体（中文） |
| `README.md` | 本文件（英文） |
| `README.zh-CN.md` | 本文件（中文） |
