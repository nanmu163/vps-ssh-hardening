---
name: "vps-ssh-hardening"
description: "Harden SSH on a newly purchased VPS: change the default SSH port, create a dedicated sudo user, and enforce key-only SSH login (no passwords, no root). Covers CentOS/RHEL, Ubuntu and Debian. Use when setting up or securing SSH access on a new VPS, or when asked to 'secure SSH' / 'SSH 加固' on a server."
---

# VPS SSH Hardening

## Purpose

Take a freshly purchased VPS from its insecure defaults (root + password, port 22) to a hardened state:

- SSH on a non-standard port
- No root SSH login
- No password authentication — key-only
- One dedicated sudo user as the single entry point
- One SSH keypair per client device; private keys never leave their device

**Scope:** SSH login hardening only. Proxy panels, general firewall policy, and TCP tuning are out of scope.

**Portability:** this file is plain Markdown with no agent-specific syntax. It works the same when handed to Claude, Codex, Gemini, Muse, Grok, or any other agent with shell access to the server.

## How to use this skill

You are the agent. Steps are tagged:

- `[AGENT]` — you execute on the server via your shell/SSH access.
- `[USER]` — the user executes on their own machine. Give them the exact commands; never run these yourself.
- `[ASK]` — stop and get the user's explicit answer first. Never invent a port, username, key label, or any other value.

**Golden rules (never violate — each one exists because someone got locked out learning it):**

1. **Never close the initial working session** until the new key login is verified in a *separate* session. It is the only rescue path.
2. **Back up** `/etc/ssh/sshd_config` before editing it.
3. **Validate** with `sshd -t` and `sshd -T` *before* every `systemctl restart sshd`.
4. **Never ask for, receive, or handle private keys, passphrases, or passwords.** Public keys are safe to see; private key material is not.
5. **Order matters:** enable `PubkeyAuthentication` on the server *before* testing key login. Testing before enabling always fails — the server simply wasn't accepting keys.
6. If any verification step fails, **stop**. Do not proceed to the next step.

## Step 0 — Collect inputs [ASK]

Get ALL of the following before touching the server. Do not proceed on assumptions.

1. **VPS access**: IP address and current login method (cloud console? root password? provider-injected key?). You need one working shell to start.
2. **OS**: run `grep -E "^ID=" /etc/os-release` once connected, then use the differences table below.
3. **New SSH port** (e.g. `1234`): any free port 1024–65535; avoid well-known ones. Confirm with the user that the cloud provider's security group allows it — have them open it in the provider console if unsure.
4. **New sudo username** (e.g. `test`): not `root`, not `admin`, nothing guessable.
5. **Key passphrase**: recommend YES. The user sets it themselves during `ssh-keygen`; you never see it.
6. **Client software** (WindTerm / Termius / PuTTY / plain `ssh`): determines the UI guidance in Step 6.
7. **sudo password policy**: keep `NOPASSWD` (convenient; recommended for a personal VPS) or require the account password on every `sudo` (stricter)? Check the current state with `sudo grep -rn NOPASSWD /etc/sudoers /etc/sudoers.d/ 2>/dev/null` and report it — don't silently change it.
8. Confirm: **disable root SSH login entirely** (`PermitRootLogin no`)? Default: yes.

## OS differences

| | CentOS / RHEL 7/8/9 | Ubuntu 20.04/22.04/24.04 | Debian 11/12 |
|---|---|---|---|
| sudo group | `wheel` | `sudo` | `sudo` |
| SSH service name | `sshd` | `ssh` | `ssh` |
| SSH auth log | `/var/log/secure` | `/var/log/auth.log` | `/var/log/auth.log` |
| host firewall | often none — use provider security group (`firewalld` if present) | `ufw` | none by default |
| SELinux | often Enforcing — see Step 2 | n/a | n/a |

In the commands below, `<SUDO_GROUP>` = `wheel` on CentOS/RHEL, `sudo` on Ubuntu/Debian. `systemctl restart sshd` = `systemctl restart ssh` on Ubuntu/Debian. `/var/log/secure` = `/var/log/auth.log` on Ubuntu/Debian.

## Step 1 — Create the sudo user [AGENT]

```bash
useradd -m -s /bin/bash <username>
usermod -aG <SUDO_GROUP> <username>
id <username>
# expect: uid=...( <username>) gid=... groups=...(<wheel|sudo>)
```

Verify sudo rights:

```bash
sudo -l -U <username>
# expect a line like: (ALL) ALL   — with or without NOPASSWD:
```

## Step 2 — Change the SSH port [AGENT]

```bash
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
sed -i 's/^#*Port.*/Port <newport>/' /etc/ssh/sshd_config
sshd -t && echo "syntax OK"
sshd -T | grep -E "^port"
# expect: port <newport>
```

**SELinux (CentOS/RHEL only):** if `getenforce` prints `Enforcing`, the new port must be labelled or sshd can't bind to it:

```bash
semanage port -a -t ssh_port_t -p tcp <newport>
# (if semanage is missing: yum install -y policycoreutils-python-utils)
```

**Firewall:** open the port wherever filtering happens —
provider security group (ask the user to confirm), plus on-host if present:

```bash
firewall-cmd --permanent --add-port=<newport>/tcp && firewall-cmd --reload   # firewalld
ufw allow <newport>/tcp                                                       # Ubuntu ufw
```

Then `systemctl restart sshd` (validated above), and **reconnect the current session on the new port immediately** to confirm it works before continuing. Keep this session open.

## Step 3 — Generate the keypair [USER]

On the **user's own machine** (never on the server):

```bash
ssh-keygen -t ed25519 -C "<username>-<device>" -f ~/.ssh/<username>_vps
```

- `-C` is a free-text label (e.g. `test-mac`) so keys are identifiable later. Always set it.
- Passphrase: recommended (see Step 0, item 5). The passphrase prompt shows nothing while typing — that's normal.
- Result: `~/.ssh/<username>_vps` (**PRIVATE — never leaves this machine, never sent to anyone including you**) and `~/.ssh/<username>_vps.pub` (public — safe to share).

## Step 4 — Install the public key [AGENT]

In the initial (still working) session, as root:

```bash
mkdir -p /home/<username>/.ssh
cat >> /home/<username>/.ssh/authorized_keys
# paste the .pub content, Enter, Ctrl+D
chmod 700 /home/<username>/.ssh
chmod 600 /home/<username>/.ssh/authorized_keys
chown -R <username>:<username> /home/<username>/.ssh
```

Verify (ownership and permissions are what actually break logins):

```bash
ls -la /home/<username>/.ssh/
# expect: drwx------ <username>  and  -rw------- <username> authorized_keys
ssh-keygen -l -f /home/<username>/.ssh/authorized_keys
# expect: one line per key, with the -C label visible
```

## Step 5 — Enable public-key auth on the server [AGENT]

Do this **before** any login test:

```bash
sed -i 's/^#*PubkeyAuthentication.*/PubkeyAuthentication yes/' /etc/ssh/sshd_config
sshd -t && echo "syntax OK"
sshd -T | grep -i pubkeyauthentication
# expect: pubkeyauthentication yes
systemctl restart sshd
```

> Do not skip or reorder this step. A key login test against a server with `pubkeyauthentication no` always fails, no matter how correct the key is.

## Step 6 — Configure the client [USER]

Adapt to their client software:

**WindTerm (macOS / Windows):**
1. New session → host = VPS IP, port = new SSH port, username = `<username>`.
2. SSH → Authentication → identity file → `...` → select the **private** key (the one *without* `.pub`).
   - macOS file picker hides `~/.ssh`: press `Cmd+Shift+.` to reveal hidden files, or `Cmd+Shift+G` and type `~/.ssh`.
   - The path must be **absolute** (start with `/`). A missing leading `/` silently breaks authentication.
3. For the test: temporarily **uncheck** "try password authentication", leaving only public-key — so a failure can't hide behind password fallback.
4. Connect → when prompted for a password, enter the **key passphrase** (set in Step 3). This is *not* the server account password.

**Termius (iOS / Android):** generate or import the keypair in the app's keychain *on that device*, attach it to the host entry.

**Plain ssh:** `ssh -i ~/.ssh/<username>_vps -p <newport> <username>@<ip>`

**Concept to convey:** the *passphrase* unlocks the private key file; the *account password* logs into the server account. Different secrets. After hardening, the account password is never used for SSH.

## Step 7 — Verify key login in a SEPARATE session [USER] + [AGENT]

The user opens a **new** client window and logs in as `<username>` with the key. The original session stays open.

Then [AGENT] confirms on the server:

```bash
grep "Accepted" /var/log/secure | tail -3    # CentOS/RHEL
grep "Accepted" /var/log/auth.log | tail -3  # Ubuntu/Debian
# expect: Accepted publickey for <username> ... ED25519 SHA256:<fingerprint>
```

**Only proceed when `Accepted publickey` is confirmed.** If it fails, see Troubleshooting — do not continue to Step 8.

## Step 8 — Disable passwords and root login [AGENT]

Only after Step 7 succeeds:

```bash
sed -i 's/^#*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sed -i 's/^#*ChallengeResponseAuthentication.*/ChallengeResponseAuthentication no/' /etc/ssh/sshd_config
sshd -t && echo "syntax OK"
sshd -T | grep -Ei "permitrootlogin|passwordauthentication|pubkeyauthentication"
# expect: no / no / yes
systemctl restart sshd
```

Final verification (original session still open, use new windows):
- `<username>` + key → ✅ connects
- `<username>` + password → ❌ rejected (expected)
- `root` + anything → ❌ rejected (expected)

Then the original session may be closed. Day-to-day admin: log in as `<username>`, `sudo -i` for a root shell.

**Rollback** (from the still-open original session, if anything went wrong):

```bash
cp /etc/ssh/sshd_config.bak /etc/ssh/sshd_config && systemctl restart sshd
```

## Ongoing key management

- **New device:** generate its own keypair *on that device* (`ssh-keygen -t ed25519 -C "<username>-<device>"`). Append the **public** key as a new line in `/home/<username>/.ssh/authorized_keys`. **Never copy a private key between devices** — a lost device then means rotating keys everywhere; with per-device keys you just delete one line.
- **List keys:** `ssh-keygen -l -f /home/<username>/.ssh/authorized_keys` (the `-C` label identifies each one).
- **Revoke a key:** `sed -i '/<label-or-key-id>/d' /home/<username>/.ssh/authorized_keys`, then re-list to confirm.
- **Back up private keys:** an encrypted off-device copy (password manager / encrypted USB). A same-disk copy only protects against accidental deletion. Never store unencrypted in chat history, email, or cloud drives.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Key login fails, client falls back to password prompt | `PubkeyAuthentication` still `no` (Step 5 skipped) | Do Step 5, restart sshd, retry |
| "Authentication failed" with key selected | Wrong file picked (`.pub` instead of private key), or relative path missing leading `/` | Reselect the private key; verify the path is absolute |
| `Failed publickey` in server log | `authorized_keys` content broken (pasted across lines) or wrong permissions/ownership | Compare `ssh-keygen -y -f <private>` output (on client) against the server line; fix perms to 700/600 with correct ownership |
| Only `Failed password` in log, no `publickey` attempt | Client never offered the key | Check client auth settings: method enabled, key file attached, correct username |
| `Connection refused/timed out` on new port | Security group / host firewall not opened, or SELinux blocking (CentOS) | Open `<newport>` in provider console; `semanage port -a -t ssh_port_t -p tcp <newport>` if Enforcing |
| Locked out entirely | Password disabled before key login was verified | Provider console/VNC → restore `/etc/ssh/sshd_config.bak` → restart sshd. **This is why the first session stays open until the end.** |

## Operating Rules

1. Never ask for or accept private keys, passphrases, or account passwords in chat. Public keys are safe.
2. Never invent the port, username, key labels, or OS — always `[ASK]` / detect first.
3. Never close or reuse the initial session for verification — always a separate session.
4. Never edit `sshd_config` without a `.bak` and without `sshd -t` + `sshd -T` validation before restart.
5. After every `sed`, `grep` the changed lines and confirm via `sshd -T` — `sed` fails silently when a pattern doesn't match.
6. If any verification step fails, stop. Do not proceed to the next step.
7. Prefer `visudo` for sudoers edits (syntax-checked). Never hand-edit sudoers with `sed`.
8. Report `NOPASSWD` findings; don't silently change sudo password policy.

## Output Contract

When the skill completes, report:

- New SSH port, login username, authorized key fingerprint(s) and labels
- Effective config: the `sshd -T` values for `port`, `permitrootlogin`, `pubkeyauthentication`, `passwordauthentication`
- The last `Accepted publickey` log line as proof
- Where the private key lives, and confirmation that an encrypted off-device backup was recommended
- Anything deliberately left unchanged (e.g. kept `NOPASSWD`, kept an old device key) and why
