# MoltBot Complete Build Guide
## From Blank Beelink to Working AI Assistant

**Print this document and follow step-by-step.**

---

# Pre-Flight Checklist

## Hardware Required
- [ ] Beelink mini PC (blank, no OS)
- [ ] USB flash drive (8GB+)
- [ ] Monitor + keyboard for initial setup
- [ ] Ethernet cable (recommended) or WiFi credentials
- [ ] Your laptop for ongoing management

## Accounts to Create/Gather Before Starting

| Service | Purpose | Sign Up URL | Your Info |
|---------|---------|-------------|-----------|
| Tailscale | Secure remote access | tailscale.com | Email: _____________ |
| Anthropic | AI provider | console.anthropic.com | API Key: _____________ |
| Telegram | Chat interface | telegram.org + @BotFather | Bot Token: _____________ |
| Meta/WhatsApp | Chat interface | developers.facebook.com | Phone ID: _____________ |

## BattleShirts Credentials (gather from existing setup)

| Service | Value |
|---------|-------|
| Shopify Store Domain | __________________________ |
| Shopify Admin Token | __________________________ |
| Printful API Key | __________________________ |
| Vercel Token | __________________________ |
| Database URL (read-only) | __________________________ |

## Google Workspace Environments

List each Google Workspace you want to integrate:

| Environment Name | Domain | Primary Use |
|-----------------|--------|-------------|
| 1. _____________ | _____________ | _____________ |
| 2. _____________ | _____________ | _____________ |
| 3. _____________ | _____________ | _____________ |

---

# PART 1: PROXMOX INSTALLATION
**Time: 30-45 minutes**

## Step 1.1: Download Proxmox ISO

On your laptop:
1. Go to: https://www.proxmox.com/en/downloads
2. Download: **Proxmox VE ISO Installer** (latest version)
3. Note the filename: `proxmox-ve_8.x-x.iso`

## Step 1.2: Create Bootable USB

**Option A: Using Balena Etcher (easiest)**
1. Download Etcher: https://etcher.balena.io
2. Insert USB drive
3. Open Etcher → Select Proxmox ISO → Select USB → Flash

**Option B: Using Rufus (Windows)**
1. Download Rufus: https://rufus.ie
2. Select USB, select ISO, click Start
3. Choose "Write in DD Image mode" if prompted

## Step 1.3: Boot Beelink from USB

1. Connect USB to Beelink
2. Connect monitor, keyboard, ethernet
3. Power on Beelink
4. Press **F7** or **DEL** repeatedly to enter boot menu
   - (Different Beelink models may use F2, F10, or F12)
5. Select USB drive from boot menu

## Step 1.4: Proxmox Installation

Follow the installer:

```
┌─────────────────────────────────────────────────────────┐
│ Proxmox VE Installer                                    │
├─────────────────────────────────────────────────────────┤
│ 1. Accept EULA                                          │
│ 2. Target disk: Select Beelink's internal drive        │
│ 3. Location: Select your country/timezone              │
│ 4. Password: _____________________ (WRITE THIS DOWN)   │
│    Email: _____________________                         │
│ 5. Network:                                             │
│    Hostname: pve.local                                  │
│    IP: Use DHCP or set static (recommended):            │
│        IP: 192.168.___.100                              │
│        Netmask: 255.255.255.0                          │
│        Gateway: 192.168.___.1                          │
│        DNS: 8.8.8.8                                    │
│ 6. Install                                              │
└─────────────────────────────────────────────────────────┘
```

**Write down your Proxmox IP: ____________________**

## Step 1.5: Remove USB and Reboot

1. When installer completes, remove USB
2. Press Enter to reboot
3. Wait for Proxmox to boot (shows IP address on screen)

## Step 1.6: Access Proxmox Web UI

On your laptop browser:
```
https://192.168.x.100:8006
```
- Accept the security warning (self-signed cert)
- Login: `root` / `<password you set>`

**Checkpoint: You should see the Proxmox dashboard.**

---

# PART 2: PROXMOX CONFIGURATION
**Time: 15 minutes**

## Step 2.1: Remove Subscription Nag (Optional)

In Proxmox shell (click node → Shell):

```bash
sed -Ezi.bak "s/(Ext\.Msg\.show\(\{.*?)title:/void\({title:/g" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
systemctl restart pveproxy.service
```

## Step 2.2: Update Proxmox

In Proxmox shell:

```bash
# Fix repository for non-subscription
cat > /etc/apt/sources.list.d/pve-no-subscription.list << 'EOF'
deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription
EOF

# Comment out enterprise repo
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/pve-enterprise.list

# Update
apt update && apt dist-upgrade -y
```

## Step 2.3: Install Tailscale on Proxmox Host

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```

A URL will appear - open it on your laptop to authenticate.

**Your Proxmox Tailscale hostname: _______________________**

## Step 2.4: Enable IP Forwarding

```bash
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
echo 'net.ipv6.conf.all.forwarding = 1' >> /etc/sysctl.conf
sysctl -p
```

## Step 2.5: Upload ISO Images

In Proxmox web UI:
1. Expand your node → local (storage) → ISO Images
2. Click "Download from URL"
3. Add these ISOs:

**Ubuntu 22.04 Server:**
```
https://releases.ubuntu.com/22.04/ubuntu-22.04.4-live-server-amd64.iso
```

**Checkpoint: Two ISOs visible in storage.**

---

# PART 3: HOME ASSISTANT VM
**Time: 30-45 minutes**

## Step 3.1: Create HAOS VM (Automated Script)

In Proxmox shell:

```bash
bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/vm/haos-vm.sh)"
```

When prompted:
- Use defaults for most options
- VM ID: `101`
- Cores: `2`
- RAM: `4096` (4GB)

Wait for script to complete and VM to start.

## Step 3.2: Find Home Assistant IP

In Proxmox:
1. Click VM 101 → Console
2. Wait for boot (may take 2-3 minutes)
3. Note the IP shown: `http://192.168.x.xxx:8123`

**Home Assistant IP: _______________________**

## Step 3.3: Initial Home Assistant Setup

In your laptop browser:
```
http://192.168.x.xxx:8123
```

Wait for "Preparing Home Assistant" (can take 5-20 minutes on first boot).

When ready:
1. Create account (this is your HA admin)
   - Name: _____________
   - Username: _____________
   - Password: _____________
2. Name your home
3. Set location (for weather/sun automations)
4. Skip analytics if desired
5. Finish setup

## Step 3.4: Install Essential Add-ons

Settings → Add-ons → Add-on Store → Install these:

- [ ] **File Editor** (click Install, then Start)
- [ ] **Terminal & SSH** (Install, Configure with password, Start)
- [ ] **Samba Share** (Install, set password, Start)

## Step 3.5: Install HACS (Home Assistant Community Store)

In HA Terminal add-on:

```bash
wget -O - https://get.hacs.xyz | bash -
```

Then:
1. Restart Home Assistant (Settings → System → Restart)
2. Settings → Devices & Services → Add Integration → Search "HACS"
3. Follow GitHub auth flow

## Step 3.6: SmartThings Integration

1. Go to https://account.smartthings.com/tokens
2. Generate new token with ALL permissions
3. Copy token: _______________________________
4. In HA: Settings → Devices & Services → Add Integration → SmartThings
5. Paste token, select your location

**Checkpoint: Your SmartThings devices appear in HA.**

## Step 3.7: Create MoltBot Access Token

1. Click your profile picture (bottom left)
2. Scroll to "Long-Lived Access Tokens"
3. Create Token → Name it "MoltBot"
4. **COPY AND SAVE THIS TOKEN:**

```
HA Token: ________________________________________________
```

---

# PART 4: MOLTBOT GATEWAY VM
**Time: 45-60 minutes**

## Step 4.1: Create Ubuntu VM

In Proxmox web UI:

1. Click "Create VM" (top right)

**General:**
- VM ID: `100`
- Name: `moltbot-gateway`

**OS:**
- ISO image: ubuntu-22.04-live-server
- Type: Linux
- Version: 6.x - 2.6 Kernel

**System:**
- Machine: q35
- BIOS: SeaBIOS
- [ ] Add EFI Disk (leave unchecked)

**Disks:**
- Bus: VirtIO Block
- Disk size: `32` GB
- [ ] SSD emulation (check)

**CPU:**
- Cores: `2`

**Memory:**
- Memory: `4096` MB

**Network:**
- Bridge: vmbr0
- Model: VirtIO

Click **Finish**, then **Start** the VM.

## Step 4.2: Install Ubuntu

Click VM 100 → Console, then follow installer:

```
┌─────────────────────────────────────────────────────────┐
│ Ubuntu Server Installer                                 │
├─────────────────────────────────────────────────────────┤
│ 1. Language: English                                    │
│ 2. Keyboard: Your layout                                │
│ 3. Installation: Ubuntu Server                          │
│ 4. Network: Accept DHCP (or set static)                │
│ 5. Proxy: Leave blank                                   │
│ 6. Mirror: Accept default                               │
│ 7. Storage: Use entire disk                             │
│ 8. Profile:                                             │
│    Your name: MoltBot Admin                            │
│    Server name: moltbot                                 │
│    Username: moltbot                                    │
│    Password: _____________________                      │
│ 9. SSH: [x] Install OpenSSH server                      │
│ 10. Snaps: Skip                                         │
│ 11. Wait for install, then Reboot                       │
└─────────────────────────────────────────────────────────┘
```

## Step 4.3: First Login and Get IP

After reboot, login in console:
- Username: `moltbot`
- Password: `<your password>`

Get IP:
```bash
ip addr show
```

**MoltBot VM IP: _______________________**

## Step 4.4: SSH from Your Laptop

From now on, work via SSH (easier to copy/paste):

```bash
ssh moltbot@192.168.x.xxx
```

## Step 4.5: System Updates and Dependencies

Run these commands in order:

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install essential packages
sudo apt install -y curl git build-essential postgresql jq ffmpeg unzip

# Install Node.js 20
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Verify Node
node --version   # Should show v20.x

# Install pnpm
sudo npm install -g pnpm
pnpm --version   # Should show 8.x or 9.x

# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Follow the Tailscale auth URL.

**MoltBot Tailscale hostname: _______________________**

## Step 4.6: Setup PostgreSQL

```bash
# Enable and start PostgreSQL
sudo systemctl enable postgresql
sudo systemctl start postgresql

# Create database and user
sudo -u postgres psql << 'EOF'
CREATE USER moltbot WITH PASSWORD 'change-this-password';
CREATE DATABASE moltbot OWNER moltbot;
GRANT ALL PRIVILEGES ON DATABASE moltbot TO moltbot;
\q
EOF
```

**Your local DB password: _______________________**

## Step 4.7: Clone MoltBot

```bash
cd ~
git clone https://github.com/defrostedcanuck/MoltBot.git
cd MoltBot
pnpm install
pnpm build
```

## Step 4.8: Create Directory Structure

```bash
mkdir -p ~/.moltbot/{config,logs,data,skills,imports/finance}
```

**Checkpoint: MoltBot code is cloned and built.**

---

# PART 5: CONFIGURATION FILES
**Time: 30 minutes**

## Step 5.1: Environment Variables

Create the environment file:

```bash
nano ~/.moltbot/.env
```

Paste and fill in your values:

```bash
# ==========================================
# AI PROVIDER
# ==========================================
ANTHROPIC_API_KEY=sk-ant-api03-xxxxx

# ==========================================
# HOME ASSISTANT
# ==========================================
HOME_ASSISTANT_URL=http://192.168.x.xxx:8123
HOME_ASSISTANT_TOKEN=eyJ0eXAiOiJKV1...paste-full-token

# ==========================================
# BATTLESHIRTS
# ==========================================
SHOPIFY_STORE_DOMAIN=battleshirts.myshopify.com
SHOPIFY_ADMIN_TOKEN=shpat_xxxxx
PRINTFUL_API_KEY=xxxxx
VERCEL_TOKEN=xxxxx
BATTLESHIRTS_DB_URL=postgresql://readonly:xxx@host/db

# ==========================================
# COMMUNICATION CHANNELS
# ==========================================
TELEGRAM_BOT_TOKEN=123456789:ABCdefGHIjklmnop

# WhatsApp (fill in when ready)
# WHATSAPP_PHONE_ID=
# WHATSAPP_ACCESS_TOKEN=

# ==========================================
# GOOGLE WORKSPACE (multiple environments)
# ==========================================
# Environment 1
GOOGLE_WS1_CLIENT_ID=xxxxx.apps.googleusercontent.com
GOOGLE_WS1_CLIENT_SECRET=xxxxx
GOOGLE_WS1_REFRESH_TOKEN=xxxxx
GOOGLE_WS1_DOMAIN=your-domain-1.com

# Environment 2
GOOGLE_WS2_CLIENT_ID=xxxxx.apps.googleusercontent.com
GOOGLE_WS2_CLIENT_SECRET=xxxxx
GOOGLE_WS2_REFRESH_TOKEN=xxxxx
GOOGLE_WS2_DOMAIN=your-domain-2.com

# Environment 3 (add if needed)
# GOOGLE_WS3_CLIENT_ID=
# GOOGLE_WS3_CLIENT_SECRET=
# GOOGLE_WS3_REFRESH_TOKEN=
# GOOGLE_WS3_DOMAIN=

# ==========================================
# LOCAL DATABASE
# ==========================================
DATABASE_URL=postgresql://moltbot:your-password@localhost:5432/moltbot
```

Save: `Ctrl+O`, `Enter`, `Ctrl+X`

## Step 5.2: Main Configuration

Create config file:

```bash
nano ~/.moltbot/config/config.json5
```

Paste this configuration:

```json5
{
  // ===========================================
  // GATEWAY
  // ===========================================
  gateway: {
    name: "MoltBot",
    host: "0.0.0.0",
    port: 3100,
    webUI: {
      enabled: true,
      port: 3101,
    },
  },

  // ===========================================
  // AI PROVIDERS
  // ===========================================
  providers: {
    anthropic: {
      apiKey: "${ANTHROPIC_API_KEY}",
      defaultModel: "claude-sonnet-4-20250514",
    },
  },

  // ===========================================
  // AGENTS
  // ===========================================
  agents: {
    defaults: {
      provider: "anthropic",
      model: "claude-sonnet-4-20250514",
      sandbox: {
        mode: "all",
        scope: "per-agent",
      },
    },

    list: [
      // BATTLESHIRTS - Business operations
      {
        id: "battleshirts",
        name: "BattleShirts Ops",
        description: "E-commerce operations for BattleShirts",
        tools: {
          allow: ["group:fs", "group:runtime", "group:memory", "message"],
        },
      },

      // PERSONAL - Calendar, email, productivity
      {
        id: "personal",
        name: "Personal Assistant",
        description: "Calendar, email, meetings across all workspaces",
        tools: {
          allow: ["group:fs", "group:memory", "message", "google-workspace"],
          deny: ["group:runtime"],
        },
      },

      // HOME - Home automation
      {
        id: "home",
        name: "Home Control",
        description: "Smart home via Home Assistant",
        tools: {
          allow: ["group:memory", "message", "homeassistant"],
        },
      },

      // FINANCE - Read-only financial tracking
      {
        id: "finance",
        name: "Finance Tracker",
        description: "Read-only financial analysis from imports",
        tools: {
          allow: ["group:memory", "message"],
          deny: ["group:runtime", "group:fs"],
        },
      },
    ],
  },

  // ===========================================
  // TOOL POLICIES
  // ===========================================
  tools: {
    exec: {
      security: "allowlist",
      timeout: 30000,
      safeBins: ["jq", "grep", "curl", "git"],
    },
    elevated: {
      enabled: false,
    },
  },

  // ===========================================
  // GOOGLE WORKSPACE (Multiple Environments)
  // ===========================================
  integrations: {
    googleWorkspace: {
      environments: [
        {
          id: "ws1",
          name: "Primary Workspace",
          domain: "${GOOGLE_WS1_DOMAIN}",
          credentials: {
            clientId: "${GOOGLE_WS1_CLIENT_ID}",
            clientSecret: "${GOOGLE_WS1_CLIENT_SECRET}",
            refreshToken: "${GOOGLE_WS1_REFRESH_TOKEN}",
          },
          services: ["calendar", "gmail", "drive"],
        },
        {
          id: "ws2",
          name: "Secondary Workspace",
          domain: "${GOOGLE_WS2_DOMAIN}",
          credentials: {
            clientId: "${GOOGLE_WS2_CLIENT_ID}",
            clientSecret: "${GOOGLE_WS2_CLIENT_SECRET}",
            refreshToken: "${GOOGLE_WS2_REFRESH_TOKEN}",
          },
          services: ["calendar", "gmail"],
        },
        // Add more as needed
      ],
    },

    homeAssistant: {
      url: "${HOME_ASSISTANT_URL}",
      token: "${HOME_ASSISTANT_TOKEN}",
    },
  },

  // ===========================================
  // COMMUNICATION CHANNELS
  // ===========================================
  channels: {
    telegram: {
      enabled: true,
      token: "${TELEGRAM_BOT_TOKEN}",
      allowedUsers: [],  // Add your Telegram user ID
      agentRouting: {
        default: "personal",
        prefixes: {
          "/biz": "battleshirts",
          "/home": "home",
          "/money": "finance",
        },
      },
    },

    // WhatsApp - enable when ready
    // whatsapp: {
    //   enabled: true,
    //   phoneNumberId: "${WHATSAPP_PHONE_ID}",
    //   accessToken: "${WHATSAPP_ACCESS_TOKEN}",
    // },
  },

  // ===========================================
  // DATABASE
  // ===========================================
  database: {
    url: "${DATABASE_URL}",
  },

  // ===========================================
  // LOGGING
  // ===========================================
  logging: {
    level: "info",
    file: "~/.moltbot/logs/gateway.log",
  },
}
```

Save: `Ctrl+O`, `Enter`, `Ctrl+X`

## Step 5.3: Exec Approvals

Create approval rules:

```bash
nano ~/.moltbot/exec-approvals.json
```

```json
{
  "version": 1,
  "defaults": {
    "security": "allowlist",
    "ask": "on-miss",
    "askFallback": "deny"
  },
  "agents": {
    "battleshirts": {
      "security": "allowlist",
      "ask": "on-miss",
      "allowlist": [
        {"pattern": "/usr/bin/curl"},
        {"pattern": "/usr/bin/jq"},
        {"pattern": "/usr/bin/git"}
      ]
    },
    "personal": {
      "security": "deny",
      "ask": "always"
    },
    "home": {
      "security": "allowlist",
      "ask": "on-miss",
      "allowlist": [
        {"pattern": "/usr/bin/curl"}
      ]
    },
    "finance": {
      "security": "deny",
      "ask": "always"
    }
  }
}
```

Save and exit.

---

# PART 6: GOOGLE WORKSPACE SETUP
**Time: 20-30 minutes per workspace**

Repeat for each Google Workspace environment.

## Step 6.1: Create Google Cloud Project

1. Go to https://console.cloud.google.com
2. Create new project: "MoltBot-WS1" (or descriptive name)
3. Select the project

## Step 6.2: Enable APIs

Go to APIs & Services → Enable APIs:
- [ ] Google Calendar API
- [ ] Gmail API
- [ ] Google Drive API (optional)

## Step 6.3: Create OAuth Credentials

1. APIs & Services → Credentials → Create Credentials → OAuth Client ID
2. Configure consent screen first:
   - User Type: Internal (if Workspace) or External
   - App name: MoltBot
   - Support email: your email
   - Scopes: Add these:
     - `https://www.googleapis.com/auth/calendar.readonly`
     - `https://www.googleapis.com/auth/calendar.events`
     - `https://www.googleapis.com/auth/gmail.readonly`
     - `https://www.googleapis.com/auth/gmail.send`
3. Create OAuth Client ID:
   - Application type: Desktop app
   - Name: MoltBot

4. Download JSON, note:
   - Client ID: _______________________
   - Client Secret: _______________________

## Step 6.4: Get Refresh Token

On MoltBot server, create helper script:

```bash
nano ~/get-google-token.js
```

```javascript
const http = require('http');
const url = require('url');

const CLIENT_ID = 'YOUR_CLIENT_ID';
const CLIENT_SECRET = 'YOUR_CLIENT_SECRET';
const REDIRECT_URI = 'http://localhost:3000/callback';
const SCOPES = [
  'https://www.googleapis.com/auth/calendar.readonly',
  'https://www.googleapis.com/auth/calendar.events',
  'https://www.googleapis.com/auth/gmail.readonly',
  'https://www.googleapis.com/auth/gmail.send',
].join(' ');

const authUrl = `https://accounts.google.com/o/oauth2/v2/auth?` +
  `client_id=${CLIENT_ID}&` +
  `redirect_uri=${encodeURIComponent(REDIRECT_URI)}&` +
  `response_type=code&` +
  `scope=${encodeURIComponent(SCOPES)}&` +
  `access_type=offline&` +
  `prompt=consent`;

console.log('\n1. Open this URL in your browser:\n');
console.log(authUrl);
console.log('\n2. Waiting for callback...\n');

const server = http.createServer(async (req, res) => {
  const query = url.parse(req.url, true).query;
  if (query.code) {
    const tokenResponse = await fetch('https://oauth2.googleapis.com/token', {
      method: 'POST',
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      body: new URLSearchParams({
        client_id: CLIENT_ID,
        client_secret: CLIENT_SECRET,
        code: query.code,
        grant_type: 'authorization_code',
        redirect_uri: REDIRECT_URI,
      }),
    });
    const tokens = await tokenResponse.json();
    console.log('\n=== TOKENS ===\n');
    console.log('Refresh Token:', tokens.refresh_token);
    console.log('\nAdd this to your .env file!\n');
    res.end('Success! Check terminal for tokens.');
    server.close();
  }
});

server.listen(3000);
```

Run it:
```bash
node ~/get-google-token.js
```

Open the URL, authorize, copy the refresh token.

**WS1 Refresh Token: _______________________**

Repeat for each workspace.

---

# PART 7: TELEGRAM BOT SETUP
**Time: 10 minutes**

## Step 7.1: Create Bot

1. Open Telegram, search for `@BotFather`
2. Send `/newbot`
3. Name: `MoltBot` (or your preference)
4. Username: `your_moltbot_bot` (must end in `bot`)
5. Copy the token

**Bot Token: _______________________**

## Step 7.2: Get Your User ID

1. Search for `@userinfobot` on Telegram
2. Send `/start`
3. Note your ID

**Your Telegram User ID: _______________________**

## Step 7.3: Update Config

Edit `~/.moltbot/config/config.json5` and add your user ID to allowedUsers:

```json5
telegram: {
  enabled: true,
  token: "${TELEGRAM_BOT_TOKEN}",
  allowedUsers: ["YOUR_USER_ID_HERE"],  // <-- Add this
```

---

# PART 8: START MOLTBOT
**Time: 10 minutes**

## Step 8.1: Test Run

```bash
cd ~/MoltBot
source ~/.moltbot/.env
pnpm start
```

Watch for errors. If it starts successfully, you'll see:
```
MoltBot Gateway started on port 3100
Web UI available on port 3101
```

Press `Ctrl+C` to stop.

## Step 8.2: Create Systemd Service

```bash
sudo nano /etc/systemd/system/moltbot.service
```

```ini
[Unit]
Description=MoltBot Gateway
After=network.target postgresql.service

[Service]
Type=simple
User=moltbot
WorkingDirectory=/home/moltbot/MoltBot
EnvironmentFile=/home/moltbot/.moltbot/.env
ExecStart=/usr/bin/node dist/gateway/index.js
Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

Save and enable:

```bash
sudo systemctl daemon-reload
sudo systemctl enable moltbot
sudo systemctl start moltbot
sudo systemctl status moltbot
```

## Step 8.3: Verify Running

```bash
# Check status
sudo systemctl status moltbot

# Check logs
journalctl -u moltbot -f

# Test API
curl http://localhost:3100/health
```

---

# PART 9: TEST EVERYTHING
**Time: 15 minutes**

## Test Checklist

### Telegram
- [ ] Open Telegram, find your bot
- [ ] Send: `Hello`
- [ ] Should get a response

### Agent Routing
- [ ] Send: `/biz what orders came in today?`
- [ ] Send: `/home turn on living room lights`
- [ ] Send: `/money` (should say no data yet)
- [ ] Send: `what's on my calendar tomorrow?` (personal agent)

### Web UI
- [ ] Open: `http://moltbot.tailnet.ts.net:3101`
- [ ] Can see dashboard
- [ ] Can send messages

### Home Assistant
- [ ] Send via Telegram: `/home what devices are on?`
- [ ] Should list device states

---

# PART 10: MAINTENANCE

## Daily

Nothing required - runs automatically.

## Weekly

```bash
# Check logs for errors
journalctl -u moltbot --since "1 week ago" | grep -i error
```

## Monthly

```bash
# Update MoltBot
cd ~/MoltBot
git pull
pnpm install
pnpm build
sudo systemctl restart moltbot

# Update system
sudo apt update && sudo apt upgrade -y
```

## Backup (Before Major Changes)

On Proxmox host:
```bash
# Snapshot MoltBot VM
qm snapshot 100 pre-update

# Snapshot Home Assistant VM
qm snapshot 101 pre-update
```

---

# QUICK REFERENCE CARD

Cut out and keep near your workstation:

```
╔═══════════════════════════════════════════════════════════╗
║  MOLTBOT QUICK REFERENCE                                  ║
╠═══════════════════════════════════════════════════════════╣
║                                                           ║
║  URLS                                                     ║
║  Proxmox:     https://________:8006                       ║
║  Home Assist: http://________:8123                        ║
║  MoltBot UI:  http://________:3101                        ║
║                                                           ║
║  SSH ACCESS                                               ║
║  ssh moltbot@________.tailnet.ts.net                      ║
║                                                           ║
║  TELEGRAM COMMANDS                                        ║
║  /biz    → BattleShirts operations                        ║
║  /home   → Home automation                                ║
║  /money  → Finance (read-only)                            ║
║  (none)  → Personal assistant                             ║
║                                                           ║
║  SERVICE COMMANDS                                         ║
║  sudo systemctl status moltbot                            ║
║  sudo systemctl restart moltbot                           ║
║  journalctl -u moltbot -f                                 ║
║                                                           ║
║  EMERGENCY                                                ║
║  sudo systemctl stop moltbot   ← Stop immediately         ║
║                                                           ║
╚═══════════════════════════════════════════════════════════╝
```

---

# TROUBLESHOOTING

## MoltBot Won't Start

```bash
# Check status
sudo systemctl status moltbot

# See detailed logs
journalctl -u moltbot -n 100

# Try manual start to see errors
cd ~/MoltBot
source ~/.moltbot/.env
pnpm start
```

## Telegram Bot Not Responding

1. Verify bot token in `.env`
2. Check your user ID is in allowedUsers
3. Check logs: `journalctl -u moltbot | grep -i telegram`

## Home Assistant Connection Failed

```bash
# Test from MoltBot server
curl -H "Authorization: Bearer $HOME_ASSISTANT_TOKEN" \
  http://192.168.x.xxx:8123/api/
```

## Google Calendar Not Working

1. Verify refresh token is valid
2. Check API is enabled in Google Cloud Console
3. Re-run token generation if expired

---

**Build Complete!**

Your MoltBot system is now ready. Start with basic commands via Telegram and gradually expand capabilities as you build confidence.

*Document Version: 1.0 - February 2026*
