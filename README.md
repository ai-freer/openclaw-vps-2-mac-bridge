# apple-apps

Control Apple Notes, Calendar, and Reminders on a remote Mac via OpenClaw node host.

## Why This Skill

OpenClaw's built-in `apple-notes` and `apple-reminders` skills assume the gateway runs on macOS. If your gateway is on a Linux VPS (the common setup), those skills won't work.

This skill uses **osascript over node host** — your VPS gateway sends AppleScript commands to a Mac running as a remote node. Zero CLI dependencies on the Mac side.

## Architecture

```
┌─────────────┐    WebSocket     ┌─────────────┐    osascript    ┌──────────────┐
│  VPS Gateway │ ──(SSH tunnel)──▶│  Node Host  │ ──────────────▶│  macOS Apps  │
│  (Linux)     │                  │  (Mac)      │                │  Notes/Cal/  │
│              │◀─── results ─────│             │◀───────────────│  Reminders   │
└─────────────┘                  └─────────────┘                └──────────────┘
```

## Prerequisites

- OpenClaw gateway running on a Linux VPS
- A Mac (macOS 12+) with Apple Notes, Calendar, and Reminders
- SSH access from Mac to VPS (key-based auth recommended)
- Node.js 18+ on the Mac

## Setup

### 1. Install OpenClaw CLI on Mac

```bash
npm install -g openclaw
```

### 2. Create SSH tunnel

The gateway typically binds to loopback. Create a tunnel from Mac to VPS:

```bash
ssh -N -L 18790:127.0.0.1:18789 -i ~/.ssh/your-private-key user@your-vps-ip
```

For persistent connections, install `autossh`:

```bash
brew install autossh
autossh -M 0 -N \
  -o ServerAliveInterval=30 \
  -o ServerAliveCountMax=3 \
  -o ExitOnForwardFailure=yes \
  -L 18790:127.0.0.1:18789 \
  -i ~/.ssh/your-private-key user@your-vps-ip
```

### 3. Start node host

```bash
export OPENCLAW_GATEWAY_TOKEN="<your-gateway-token>"
openclaw node run --host 127.0.0.1 --port 18790 --display-name "MacBook"
```

Find your gateway token on the VPS:

```bash
grep -A2 '"auth"' ~/.openclaw/openclaw.json
# or
openclaw config get gateway.auth.token
```

### Tips for filling in paths

```bash
# Find your Node.js path (on Mac)
which node
# e.g. /Users/you/.nvm/versions/node/v22.12.0/bin/node

# Find openclaw path
which openclaw

# Find autossh path
which autossh
# Intel Mac: /usr/local/bin/autossh
# Apple Silicon: /opt/homebrew/bin/autossh
```

### 4. Configure exec approvals

On the Mac, set the exec security mode:

```bash
openclaw config set tools.exec.security allowlist
```

Edit `~/.openclaw/exec-approvals.json` to ensure `defaults.security` is set:

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

You can also push approvals from the VPS:

```bash
openclaw approvals set --node MacBook --file approvals.json
```

### 5. Verify

From the VPS:

```bash
openclaw nodes run --node MacBook -- /usr/bin/osascript -e 'return (current date) as string'
```

If you see the current date, you're good.

### 6. Run as background service (recommended)

Create two launchd services on the Mac so everything starts automatically on boot.

**SSH tunnel** — `~/Library/LaunchAgents/com.openclaw.ssh-tunnel.plist`:

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

**Node host** — `~/Library/LaunchAgents/com.openclaw.node-host.plist`:

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

Replace `YOU`, `vXX.XX.X`, key path, VPS IP, and gateway token with your values.

Load the services:

```bash
launchctl load ~/Library/LaunchAgents/com.openclaw.ssh-tunnel.plist
sleep 5
launchctl load ~/Library/LaunchAgents/com.openclaw.node-host.plist
```

Verify:

```bash
launchctl list | grep openclaw
```

### Troubleshooting

| Symptom | Fix |
|---|---|
| `node not connected` | Check SSH tunnel: `cat /tmp/openclaw-ssh-tunnel.log` |
| `SYSTEM_RUN_DENIED: approval required` | Ensure `defaults.security` is `allowlist` in `exec-approvals.json`, then restart node host |
| `allowlist miss (shell wrappers)` | Use `--` separator: `openclaw nodes run --node MacBook -- /usr/bin/osascript -e '...'` |
| macOS permission dialog | Click Allow on the Mac when prompted for Notes/Calendar/Reminders automation |
| `invalid system.run.prepare response` | Version mismatch — ensure VPS and Mac run the same OpenClaw version |

### First run — macOS permissions

The first time osascript accesses Notes, Calendar, or Reminders, macOS will show a permission dialog. Someone must be at the Mac to click **Allow**. After that, subsequent calls work unattended.

To pre-trigger all three:

```bash
openclaw nodes run --node MacBook -- /usr/bin/osascript -e 'tell application "Notes" to get name of note 1'
openclaw nodes run --node MacBook -- /usr/bin/osascript -e 'tell application "Calendar" to get name of calendar 1'
openclaw nodes run --node MacBook -- /usr/bin/osascript -e 'tell application "Reminders" to get name of list 1'
```

## Install the Skill

Copy the `apple-apps` folder to your OpenClaw skills directory:

```bash
cp -r apple-apps ~/.openclaw/skills/
```

Or for workspace-level (shared across agents):

```bash
cp -r apple-apps ~/.openclaw/workspace/skills/
```

## License

MIT
