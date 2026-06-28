# Hermes Agent Multi-Profile Deployment / 多 Profile 部署

> One server, multiple independent AI assistants. No interference.
> 一台服务器跑多个互不干扰的 AI 助手。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Hermes Agent](https://img.shields.io/badge/Hermes%20Agent-Compatible-blue)](https://github.com/NousResearch/hermes-agent)

---

## What Problem This Solves / 解决什么问题

You want to run **multiple Hermes AI assistants on one server** — for yourself, for team members, or for different purposes (work + investing + customer support). But sharing the same `~/.hermes/` directory means configs, memories, and platform credentials collide.

**Profile isolation** gives each assistant its own complete workspace — separate `.env`, memories, skills, crons, and platform tokens. They run side by side, never stepping on each other.

---

## Quick Start / 快速开始

### 1. Create a new Profile

```bash
hermes profile create <name>
# Creates ~/.hermes/profiles/<name>/
```

### 2. Configure your platform (e.g. Feishu Bot)

Edit `~/.hermes/profiles/<name>/.env`:

```bash
FEISHU_APP_ID=cli_xxxxx
FEISHU_APP_SECRET=xxxxx
FEISHU_VERIFICATION_TOKEN=xxxxx
```

**One Feishu Bot = One Profile.** Multiple bots require multiple profiles.

### 3. Create systemd service

`/etc/systemd/system/hermes-gateway-<name>.service`:

```ini
[Unit]
Description=Hermes Gateway (<Name> Profile)
After=network.target

[Service]
Type=simple
WorkingDirectory=/home/ubuntu
ExecStart=/home/ubuntu/.local/bin/hermes gateway -p <name>
Restart=always
RestartSec=10
Environment="HERMES_PROFILE=<name>"

[Install]
WantedBy=multi-user.target
```

The `-p <name>` in `ExecStart` is what tells Hermes which profile to use.

### 4. Enable and start

```bash
sudo systemctl daemon-reload
sudo systemctl enable hermes-gateway-<name>
sudo systemctl start hermes-gateway-<name>
```

---

## Pitfalls / 踩过的坑

### ⚠️ PID File Race

**Symptom:** Second gateway won't start. Log says:
```
ERROR gateway.run: PID file race lost to another gateway instance. Exiting.
```

**Fix:**
```bash
rm -f ~/.hermes/profiles/<name>/gateway.pid
sudo systemctl restart hermes-gateway-<name>
```

### ⚠️ Config Not Taking Effect

**Symptom:** Changed `.env` but old config still loaded after restart.

**Fix:** Restart the correct service, not the root one. Then verify:
```bash
cat ~/.hermes/profiles/<name>/.env   # check you edited the right file
sudo systemctl restart hermes-gateway-<name>
journalctl -u hermes-gateway-<name> -f   # verify
```

---

## Daily Ops / 日常运维

```bash
# Check service status
systemctl status hermes-gateway-root
systemctl status hermes-gateway-friend

# Tail logs (Ctrl+C to exit)
journalctl -u hermes-gateway-root -f
journalctl -u hermes-gateway-friend -f

# Restart one instance
sudo systemctl restart hermes-gateway-friend

# Check gateway connection state
cat ~/.hermes/gateway_state.json                   # root
cat ~/.hermes/profiles/friend/gateway_state.json   # friend
```

---

## Reference Architecture / 参考架构

Real-world setup by [wind](https://mp.weixin.qq.com/s/zgIGWDwXmL8XCsxwTsaquA) on a 4-core 4GB server:

| Instance | Platform | Profile |
|---|---|---|
| `hermes-gateway-root` | WeChat + Feishu | root |
| `hermes-gateway-friend` | Feishu ("Kitten") | friend |

---

## Installation / 安装

```bash
# Clone this skill into your Hermes skills directory
git clone https://github.com/asmao812/hermes-multi-profile.git
cp -r hermes-multi-profile ~/.hermes/skills/devops/hermes-multi-profile/
```

Or install via SkillHub:
```bash
skillhub install hermes-multi-profile --dir ~/.hermes/skills --force
```

---

## License

MIT — use it, modify it, share it.
