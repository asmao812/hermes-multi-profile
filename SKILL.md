---
name: hermes-multi-profile
description: Hermes Agent 多 Profile 部署——一台服务器跑多个独立 AI 助手。systemd 管理、PID 冲突解决、飞书多 Bot 绑定。
version: 1.0.0
source: wind 微信公众号文章 + 实战验证
---

# Hermes Agent 多 Profile 部署

> 一台服务器跑多个互不干扰的 AI 助手。

## 什么时候用

- 要给不同人开独立 AI 助手（团队成员、家人、客户）
- 一个 Bot 只能绑一个 Profile，多飞书 Bot 必须多 Profile
- 自己需要多个"人格"：主用 + 投资专用 + 副业客服

## Profile 本质

每个 Profile = 独立工作间，有自己的一套：
- 配置（`.env`、`config.yaml`）
- 记忆（`memories/`）
- 技能（`skills/`）
- 定时任务（`cron/`）
- 平台凭证（飞书 App ID/Secret 等）

目录结构：
```
~/.hermes/                    # root profile
~/.hermes/profiles/friend/    # friend profile
~/.hermes/profiles/invest/    # invest profile
```

## 部署步骤

### 1. 创建 Profile

```bash
# 初始化新 profile
hermes profile create <name>

# 会生成 ~/.hermes/profiles/<name>/ 目录
```

### 2. 配置新 Profile 的飞书 Bot

在 `~/.hermes/profiles/<name>/.env` 里填新 Bot 的凭证：
```bash
FEISHU_APP_ID=cli_xxxxx
FEISHU_APP_SECRET=xxxxx
FEISHU_VERIFICATION_TOKEN=xxxxx
```

**一个飞书 Bot 只能绑一个 Profile。** 多 Bot = 多 Profile。

### 3. 创建 systemd 服务

每个 Profile 对应一个 systemd 服务，实现**开机自启 + 崩溃自动拉起**。

`/etc/systemd/system/hermes-gateway-<name>.service`：
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

关键：`ExecStart` 里的 `-p <name>` 指定使用哪个 Profile。

### 4. 启用并启动

```bash
sudo systemctl daemon-reload
sudo systemctl enable hermes-gateway-<name>
sudo systemctl start hermes-gateway-<name>
```

---

## ⚠️ 踩坑清单

### 坑 1：PID 文件冲突

**症状**：第二个 Gateway 起不来，日志报：
```
ERROR gateway.run: PID file race lost to another gateway instance. Exiting.
```

**原因**：两个 Gateway 实例抢同一个 PID 记录文件。

**修复**：
```bash
# 删掉残留的 PID 记录
rm -f ~/.hermes/profiles/<name>/gateway.pid

# 重新启动
sudo systemctl restart hermes-gateway-<name>
```

### 坑 2：改配置不生效

**症状**：改了 Profile 的 `.env`，重启后还是旧配置。

**原因**：`systemctl restart` 不会自动重载环境变量，或者只重启了一个实例。

**修法**：
```bash
# 1. 确认改的是对的 Profile 目录
cat ~/.hermes/profiles/<name>/.env

# 2. 重启对应服务（不是 root 那个）
sudo systemctl restart hermes-gateway-<name>

# 3. 验证日志
journalctl -u hermes-gateway-<name> -f
```

---

## 日常运维

```bash
# 看所有服务状态
systemctl status hermes-gateway-root
systemctl status hermes-gateway-friend

# 看实时日志（Ctrl+C 退出）
journalctl -u hermes-gateway-root -f
journalctl -u hermes-gateway-friend -f

# 重启某个
sudo systemctl restart hermes-gateway-friend

# 看连接状态
cat ~/.hermes/gateway_state.json                    # root
cat ~/.hermes/profiles/friend/gateway_state.json    # friend
```

---

## 参考架构（wind 的实战配置）

| 实例 | 平台 | Profile |
|---|---|---|
| hermes-gateway-root | 微信 + 飞书 | root |
| hermes-gateway-friend | 飞书（"小猫"） | friend |

服务器：4 核 4G，两个 Gateway 实例互不干扰。
