# openclaw-vps-2-mac-bridge

[English](README.md) | 中文

通过 OpenClaw node host，从 Linux VPS 远程控制 Mac 上的 Apple Notes、Calendar 和 Reminders。

## 为什么需要这个 Skill

OpenClaw 内置的 `apple-notes` 和 `apple-reminders` skill 假设 gateway 跑在 macOS 上。如果你的 gateway 在 Linux VPS 上（常见部署方式），这些 skill 用不了。

这个 skill 通过 **osascript + node host** 实现——VPS gateway 把 AppleScript 命令发送到作为远程 node 运行的 Mac 上执行。Mac 端零依赖，不需要装额外 CLI 工具。

## 架构

```
┌─────────────┐    WebSocket     ┌─────────────┐    osascript    ┌──────────────┐
│  VPS Gateway │ ──(SSH 隧道)────▶│  Node Host  │ ──────────────▶│  macOS 应用  │
│  (Linux)     │                  │  (Mac)      │                │  备忘录/日历/ │
│              │◀─── 返回结果 ────│             │◀───────────────│  提醒事项     │
└─────────────┘                  └─────────────┘                └──────────────┘
```

## 前置条件

- Linux VPS 上运行的 OpenClaw gateway
- 一台 Mac（macOS 12+），装有备忘录、日历、提醒事项
- Mac 到 VPS 的 SSH 访问（建议密钥认证）
- Mac 上安装 Node.js 18+

## 设置步骤

### 1. 在 Mac 上安装 OpenClaw CLI

```bash
npm install -g openclaw
```

### 2. 建立 SSH 隧道

Gateway 默认绑定 loopback，需要从 Mac 建隧道到 VPS：

```bash
ssh -N -L 18790:127.0.0.1:18789 -i ~/.ssh/your-private-key user@your-vps-ip
```

推荐用 `autossh` 实现断线自动重连：

```bash
brew install autossh
autossh -M 0 -N \
  -o ServerAliveInterval=30 \
  -o ServerAliveCountMax=3 \
  -o ExitOnForwardFailure=yes \
  -L 18790:127.0.0.1:18789 \
  -i ~/.ssh/your-private-key user@your-vps-ip
```

### 3. 启动 node host

```bash
export OPENCLAW_GATEWAY_TOKEN="<your-gateway-token>"
openclaw node run --host 127.0.0.1 --port 18790 --display-name "MacBook"
```

查找 gateway token：

```bash
# 在 VPS 上执行
grep -A2 '"auth"' ~/.openclaw/openclaw.json
# 或
openclaw config get gateway.auth.token
```

### 查找本地路径

```bash
# Node.js 路径
which node
# 例如 /Users/you/.nvm/versions/node/v22.12.0/bin/node

# openclaw 路径
which openclaw

# autossh 路径
which autossh
# Intel Mac: /usr/local/bin/autossh
# Apple Silicon: /opt/homebrew/bin/autossh
```

### 4. 配置 exec approvals

在 Mac 上设置 exec 安全模式：

```bash
openclaw config set tools.exec.security allowlist
```

编辑 `~/.openclaw/exec-approvals.json`，确保 `defaults.security` 已设置：

```json
{
  "version": 1,
  "defaults": {
    "security": "allowlist"
  },
  "agents": {
    "*": {
      "allowlist": [
        {"pattern": "/usr/bin/osascript"},
        {"pattern": "/usr/bin/find"},
        {"pattern": "/bin/cat"},
        {"pattern": "/usr/bin/grep"}
      ]
    }
  }
}
```

也可以从 VPS 端推送 approvals：

```bash
openclaw approvals set --node MacBook --file approvals.json
```

### 5. 验证

在 VPS 上执行：

```bash
openclaw nodes run --node MacBook -- /usr/bin/osascript -e 'return (current date) as string'
```

看到当前日期时间就说明配通了。

### 6. 设为后台服务（推荐）

在 Mac 上创建两个 launchd 服务，开机自动启动。

**SSH 隧道** — `~/Library/LaunchAgents/com.openclaw.ssh-tunnel.plist`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.openclaw.ssh-tunnel</string>
    <key>ProgramArguments</key>
    <array>
        <string>/opt/homebrew/bin/autossh</string>
        <string>-M</string>
        <string>0</string>
        <string>-N</string>
        <string>-o</string>
        <string>ServerAliveInterval=30</string>
        <string>-o</string>
        <string>ServerAliveCountMax=3</string>
        <string>-o</string>
        <string>ExitOnForwardFailure=yes</string>
        <string>-L</string>
        <string>18790:127.0.0.1:18789</string>
        <string>-i</string>
        <string>/Users/YOU/.ssh/your-private-key</string>
        <string>user@your-vps-ip</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/tmp/openclaw-ssh-tunnel.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/openclaw-ssh-tunnel.log</string>
</dict>
</plist>
```

**Node host** — `~/Library/LaunchAgents/com.openclaw.node-host.plist`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.openclaw.node-host</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/YOU/.nvm/versions/node/vXX.XX.X/bin/node</string>
        <string>/Users/YOU/.nvm/versions/node/vXX.XX.X/bin/openclaw</string>
        <string>node</string>
        <string>run</string>
        <string>--host</string>
        <string>127.0.0.1</string>
        <string>--port</string>
        <string>18790</string>
        <string>--display-name</string>
        <string>MacBook</string>
    </array>
    <key>EnvironmentVariables</key>
    <dict>
        <key>OPENCLAW_GATEWAY_TOKEN</key>
        <string>YOUR_GATEWAY_TOKEN</string>
        <key>PATH</key>
        <string>/Users/YOU/.nvm/versions/node/vXX.XX.X/bin:/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
    </dict>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>ThrottleInterval</key>
    <integer>5</integer>
    <key>StandardOutPath</key>
    <string>/tmp/openclaw-node-host.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/openclaw-node-host.log</string>
</dict>
</plist>
```

把 `YOU`、`vXX.XX.X`、密钥路径、VPS IP 和 gateway token 替换成你的实际值。

加载服务：

```bash
launchctl load ~/Library/LaunchAgents/com.openclaw.ssh-tunnel.plist
sleep 5
launchctl load ~/Library/LaunchAgents/com.openclaw.node-host.plist
```

验证：

```bash
launchctl list | grep openclaw
```

### 首次运行 — macOS 权限

osascript 首次访问备忘录、日历、提醒事项时，macOS 会弹出权限确认对话框。需要有人在 Mac 前点击**允许**。之后的调用不再需要确认。

预触发三个应用的权限：

```bash
openclaw nodes run --node MacBook -- /usr/bin/osascript -e 'tell application "Notes" to get name of note 1'
openclaw nodes run --node MacBook -- /usr/bin/osascript -e 'tell application "Calendar" to get name of calendar 1'
openclaw nodes run --node MacBook -- /usr/bin/osascript -e 'tell application "Reminders" to get name of list 1'
```

## 常见问题

| 现象 | 解决方法 |
|---|---|
| `node not connected` | 检查 SSH 隧道：`cat /tmp/openclaw-ssh-tunnel.log` |
| `SYSTEM_RUN_DENIED: approval required` | 确保 `exec-approvals.json` 中 `defaults.security` 为 `allowlist`，然后重启 node host |
| `allowlist miss (shell wrappers)` | 使用 `--` 分隔符：`openclaw nodes run --node MacBook -- /usr/bin/osascript -e '...'` |
| macOS 权限弹窗 | 在 Mac 上点击允许 Notes/Calendar/Reminders 的自动化权限 |
| `invalid system.run.prepare response` | 版本不匹配——确保 VPS 和 Mac 上的 OpenClaw 版本一致 |

## 安装 Skill

将 `apple-apps` 文件夹复制到 OpenClaw skills 目录：

```bash
cp -r apple-apps ~/.openclaw/skills/
```

或放到 workspace 级别（所有 agent 共享）：

```bash
cp -r apple-apps ~/.openclaw/workspace/skills/
```

## License

MIT
