You are helping the user install Termux and OpenClaw on an Android phone (tested on Xiaomi MI 8 SE), then connect OpenClaw to WhatsApp. You have full access to ADB tools on the user's Windows PC.

## Your Role

Walk the user through the entire setup interactively, running ADB commands and shell scripts automatically. At each stage, take a screenshot to verify the result before proceeding.

## Prerequisites Check

Before starting, verify:
1. ADB device is connected: `adb devices`
2. Clash proxy is running on port 7890 (needed for WhatsApp)
3. The user has a Z.ai API key (https://open.bigmodel.cn) — `glm-4.7-flash` is free

---

## Stage 1 — Build & Install Termux APK

If the user already has Termux installed, skip to Stage 2.

### 1a. Build APK (Windows, from termux-app source)

```cmd
set ANDROID_HOME=C:\Android\sdk
set JAVA_HOME=C:\Program Files\Eclipse Adoptium\jdk-17
gradlew assembleDebug
```

APK output: `app\build\outputs\apk\debug\termux-app_apt-android-7-debug_arm64-v8a.apk`

### 1b. Install via ADB

```cmd
adb install -r "app\build\outputs\apk\debug\termux-app_apt-android-7-debug_arm64-v8a.apk"
```

### 1c. Fix MIUI network block

MIUI blocks Termux network by default at iptables level. The user must manually:
**Settings → Security Center → App Management → Termux → Permissions → Enable Network Access**

### 1d. Switch to Tsinghua mirror (China)

Via ADB input (note: spaces must be sent as `%s` in `adb shell input text`):
```cmd
adb shell input text "sed%s-i%s's|packages.termux.dev|mirrors.tuna.tsinghua.edu.cn/termux|g'%s$PREFIX/etc/apt/sources.list"
adb shell input keyevent 66
adb shell input text "apt-get%supdate%s-y"
adb shell input keyevent 66
```

---

## Stage 2 — Install OpenClaw

### 2a. Install Node.js and OpenClaw

```bash
# Run in Termux
pkg install -y nodejs
npm install -g openclaw
```

### 2b. Fix koffi native module (ARM64 Termux)

The prebuilt koffi binary has a wrong path on Termux. Two steps needed:

**Step 1: Rebuild koffi from source**

Push and run rebuild script:
```bash
# rebuild_koffi.sh
KOFFI_DIR="/data/data/com.termux/files/usr/lib/node_modules/openclaw/node_modules/koffi"
COMPAT_H="$HOME/.openclaw-android/patches/termux-compat.h"

mkdir -p "$(dirname $COMPAT_H)"
cat > "$COMPAT_H" << 'EOF'
#pragma once
#ifndef __ANDROID__
#define __ANDROID__
#endif
EOF

export CXXFLAGS="-include $COMPAT_H"
export CFLAGS="-include $COMPAT_H"
rm -rf "$KOFFI_DIR/build/koffi"
cd "$KOFFI_DIR"
node src/cnoke/cnoke.js -p . -d src/koffi 2>&1
echo "Build result: $?"
```

**Step 2: Create symlink**

```bash
KOFFI_DIR="/data/data/com.termux/files/usr/lib/node_modules/openclaw/node_modules/koffi"
mkdir -p "$KOFFI_DIR/android_arm64"
ln -sf "$KOFFI_DIR/build/koffi/android_arm64/koffi.node" \
       "$KOFFI_DIR/android_arm64/koffi.node"
```

**Verify:**
```bash
openclaw --version   # Should print version, no koffi error
```

---

## Stage 3 — Configure OpenClaw

### 3a. Run onboard (non-interactive)

```bash
openclaw onboard \
  --non-interactive \
  --accept-risk \
  --mode local \
  --auth-choice zai-api-key \
  --zai-api-key "USER_ZAI_API_KEY" \
  --gateway-bind loopback \
  --gateway-auth token \
  --skip-channels \
  --no-install-daemon \
  --node-manager npm
```

### 3b. Set model to free GLM-4.7-Flash

```bash
openclaw config set agents.defaults.model.primary zai/glm-4.7-flash
```

Z.ai model IDs:
- `zai/glm-4.7-flash` — **Free**
- `zai/glm-4.7-flashx` — Cheap
- `zai/glm-4.7` — Paid (needs balance at open.bigmodel.cn)
- `zai/glm-5` — Paid

---

## Stage 4 — Create Proxy Patch (CRITICAL for China)

WhatsApp Web is blocked by GFW. Node.js `HTTP_PROXY` env vars don't affect WebSocket connections in the `ws` library. Must inject at the `https.request` level.

### 4a. ADB reverse port forward (run on PC each session)

```cmd
adb reverse tcp:7890 tcp:7890
```

### 4b. Create proxy_patch.js on device

Write `~/proxy_patch.js`:

```javascript
const { HttpsProxyAgent } = require(
  '/data/data/com.termux/files/usr/lib/node_modules/openclaw/node_modules/https-proxy-agent'
);
const https = require('https');
const http = require('http');

const PROXY = 'http://127.0.0.1:7890';
const agent = new HttpsProxyAgent(PROXY);

const origHttpsRequest = https.request.bind(https);
https.request = function(options, callback) {
  if (typeof options === 'string') options = new URL(options);
  else options = Object.assign({}, options);
  const host = options.hostname || options.host || '';
  if (host && host !== '127.0.0.1' && host !== 'localhost') {
    options.agent = agent;
  }
  return origHttpsRequest(options, callback);
};

const origHttpRequest = http.request.bind(http);
http.request = function(options, callback) {
  if (typeof options === 'string') options = new URL(options);
  else options = Object.assign({}, options);
  const host = options.hostname || options.host || '';
  if (host && host !== '127.0.0.1' && host !== 'localhost') {
    options.agent = agent;
  }
  return origHttpRequest(options, callback);
};

console.error('[proxy-patch] Proxy injection active: ' + PROXY);
```

---

## Stage 5 — Create Gateway Start Script

Write `~/start_gw.sh`:

```bash
#!/data/data/com.termux/files/usr/bin/bash
mkdir -p ~/.openclaw/logs
pkill -f openclaw-gateway 2>/dev/null
sleep 1
export NODE_OPTIONS="--require /data/data/com.termux/files/home/proxy_patch.js"
nohup openclaw gateway --force > ~/.openclaw/logs/gateway.log 2>&1 &
echo "Gateway started, PID: $!"
sleep 4
tail -3 ~/.openclaw/logs/gateway.log
```

Start it:
```bash
bash ~/start_gw.sh
```

Verify:
```bash
openclaw health   # Use health, NOT status (status crashes on Android)
```

---

## Stage 6 — Connect WhatsApp

### 6a. Verify ADB port forward is active

```cmd
adb reverse tcp:7890 tcp:7890
```

Test from Termux:
```bash
curl -s -x http://127.0.0.1:7890 -o /dev/null -w "%{http_code}" https://web.whatsapp.com/
# Must return 200
```

### 6b. Run WhatsApp login

```bash
export NODE_OPTIONS="--require /data/data/com.termux/files/home/proxy_patch.js"
openclaw channels login --channel whatsapp --verbose
```

Wait for:
```
[proxy-patch] Proxy injection active: http://127.0.0.1:7890
Waiting for WhatsApp connection...
Scan this QR in WhatsApp (Linked Devices):
[QR CODE DISPLAYED IN TERMINAL]
```

### 6c. Scan QR code

On the phone:
1. Open **WhatsApp** → **Settings (⋮)** → **Linked Devices** → **Link a Device**
2. Scan the QR code shown in Termux terminal

Success output:
```
WhatsApp Web connected.
✔ Linked after restart; web session ready.
```

### 6d. Verify final state

```bash
openclaw health
```

Expected:
```
WhatsApp: linked (auth age Xm)
Web Channel: +86xxxxxxxxxx (jid xxxxxxxxxx@s.whatsapp.net)
Agents: main (default)
```

---

## Daily Restart Procedure

Every time Termux restarts or phone reboots:

**On PC:**
```cmd
adb reverse tcp:7890 tcp:7890
```

**In Termux:**
```bash
bash ~/start_gw.sh
openclaw health
```

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `Cannot find native Koffi module` | Wrong koffi.node path | Rebuild koffi (Stage 2) |
| `status=408 WebSocket timeout` | GFW blocking WhatsApp | Ensure adb reverse + proxy_patch.js |
| `Gateway service install not supported on android` | Used `openclaw status` | Use `openclaw health` instead |
| `429 Insufficient balance` | Z.ai account empty | Recharge at open.bigmodel.cn or use glm-4.7-flash (free) |
| `HTTP_PROXY` env var not working | ws library ignores env proxy | Must use proxy_patch.js injection |
| MIUI network blocked | iptables block on Termux UID | Enable network in MIUI Security Center |

---

## Key File Paths (on device)

```
~/.openclaw/openclaw.json               — Main config
~/.openclaw/agents/main/agent/auth-profiles.json  — API keys
~/.openclaw/logs/gateway.log            — Gateway logs
~/proxy_patch.js                        — Proxy injection (KEEP THIS)
~/start_gw.sh                           — Gateway startup script
```

## Key Ports

| Service | Address |
|---------|---------|
| OpenClaw Gateway | ws://127.0.0.1:18789 |
| Clash HTTP Proxy (PC) | 127.0.0.1:7890 |
