[中文版](README.zh-CN.md)

# VPS SSH Hardening Skill

An agent-executable skill that hardens SSH on a freshly purchased VPS: non-standard port, dedicated sudo user, key-only login — no passwords, no root.

## What it does

Takes a VPS from its insecure defaults (root + password on port 22) to a hardened state:

- SSH listening on a **non-standard port** of your choice
- **No root SSH login**
- **No password authentication** — SSH keys only
- One **dedicated sudo user** as the single entry point
- **One keypair per device**; private keys never leave their device

## Supported systems

| OS | Versions |
|---|---|
| CentOS / RHEL | 7, 8, 9 |
| Ubuntu | 20.04, 22.04, 24.04 |
| Debian | 11, 12 |

The skill adapts automatically to OS differences (sudo group name, SSH service name, auth log path, SELinux, firewall).

## How to use

1. Copy the raw URL of the skill file in your language:
   - English: `SKILL.md`
   - 中文: `SKILL.zh-CN.md`

   (On GitHub: open the file → **Raw** → copy the URL, e.g.
   `https://raw.githubusercontent.com/nanmu163/vps-ssh-hardening/main/SKILL.md`)
2. Paste the URL to your AI agent (Claude, Codex, Gemini, Muse, Grok, …) and say you want to harden SSH on a new VPS.
3. The agent will ask for the required inputs (IP, port, username, client software, …) — answer them; it never invents values on its own.
4. Follow the agent's instructions for steps on your own machine (key generation, client config). Server-side steps are executed by the agent.

## Workflow overview

| Step | What happens | Who |
|---|---|---|
| 0 | Collect inputs (port, username, OS, client software…) | Agent asks, you answer |
| 1 | Create the sudo user | Agent |
| 2 | Change the SSH port (incl. SELinux / firewall) | Agent |
| 3 | Generate the ed25519 keypair | You, on your machine |
| 4 | Install the public key on the server | Agent |
| 5 | Enable public-key auth on the server | Agent |
| 6 | Configure your SSH client (WindTerm / Termius / ssh) | You, on your machine |
| 7 | Verify key login in a separate session | You + Agent |
| 8 | Disable passwords and root login | Agent |

Every step ships with a verification command; the skill stops if any check fails.

## Safety notes

- The agent **never** asks for private keys, passphrases, or account passwords. Public keys are safe to share.
- Your initial session stays open until the new login is verified — the built-in rescue path if anything goes wrong.
- `sshd_config` is backed up before every change and validated (`sshd -t` / `sshd -T`) before every restart.

## Files

| File | Description |
|---|---|
| `SKILL.md` | The skill itself (English) |
| `SKILL.zh-CN.md` | The skill itself (Chinese) |
| `README.md` | This file (English) |
| `README.zh-CN.md` | This file (Chinese) |
