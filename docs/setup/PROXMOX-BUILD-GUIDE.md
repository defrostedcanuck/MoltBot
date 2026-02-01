# MoltBot Complete Build Guide - Proxmox Mini PC

A comprehensive guide to setting up MoltBot as your personal AI assistant with multiple operational domains: business (BattleShirts), personal productivity, home automation, and finance tracking.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [What You'll Need](#what-youll-need)
3. [Build Phases](#build-phases)
4. [Phase 1: Proxmox Foundation](#phase-1-proxmox-foundation)
5. [Phase 2: Home Assistant VM](#phase-2-home-assistant-vm)
6. [Phase 3: MoltBot Gateway VM](#phase-3-moltbot-gateway-vm)
7. [Phase 4: Agent Configuration](#phase-4-agent-configuration)
8. [Phase 5: Communication Channels](#phase-5-communication-channels)
9. [Phase 6: Integration Setup](#phase-6-integration-setup)
10. [Security Model](#security-model)
11. [Maintenance & Backup](#maintenance--backup)
12. [Troubleshooting](#troubleshooting)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  PROXMOX MINI PC                                                            │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  VM 100: moltbot-gateway (Ubuntu 22.04)                              │   │
│  │  RAM: 4GB | CPU: 2 cores | Disk: 32GB                                │   │
│  │                                                                      │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐    │   │
│  │  │   Agent:    │ │   Agent:    │ │   Agent:    │ │   Agent:    │    │   │
│  │  │ battleshirts│ │  personal   │ │    home     │ │   finance   │    │   │
│  │  │             │ │             │ │             │ │             │    │   │
│  │  │ Shopify     │ │ Calendar    │ │ HA Control  │ │ CSV Import  │    │   │
│  │  │ Printful    │ │ Email       │ │ Devices     │ │ Balances    │    │   │
│  │  │ Vercel      │ │ Meetings    │ │ Routines    │ │ Tracking    │    │   │
│  │  │ Analytics   │ │ Reminders   │ │ Voice Cmds  │ │ Reports     │    │   │
│  │  │             │ │             │ │             │ │             │    │   │
│  │  │ LOW GATES   │ │ MED GATES   │ │ MED GATES   │ │ HIGH GATES  │    │   │
│  │  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘    │   │
│  │                                                                      │   │
│  │  Communication Channels:                                             │   │
│  │  [WhatsApp] [Telegram] [Voice/HA] [Web UI]                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│           │                                                                 │
│           │ Local API (REST)                                               │
│           ▼                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  VM 101: home-assistant (HAOS)                                       │   │
│  │  RAM: 2GB | CPU: 2 cores | Disk: 32GB                                │   │
│  │                                                                      │   │
│  │  • SmartThings Integration (migrate existing devices)                │   │
│  │  • Alexa Integration (voice commands)                                │   │
│  │  • Wyoming/Whisper (local voice processing)                          │   │
│  │  • Automations & Scenes                                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  Network: 192.168.x.0/24 (or your local subnet)                            │
│  Remote Access: Tailscale VPN                                              │
└─────────────────────────────────────────────────────────────────────────────┘
          │
          │ Outbound API Calls
          ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  EXTERNAL SERVICES                                                          │
│                                                                             │
│  Business:              Personal:              Finance:                     │
│  • Shopify Admin API    • Google Calendar      • CSV/OFX imports           │
│  • Printful API         • Google/MS Email      • (Future: Plaid)           │
│  • Vercel API           • Telegram Bot API                                 │
│  • Anthropic API        • WhatsApp Cloud API                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## What You'll Need

### Hardware
- Mini PC with Proxmox installed (8GB+ RAM recommended, 16GB ideal)
- At least 128GB storage (256GB+ preferred)

### Accounts & API Keys (gather these before starting)

| Service | Purpose | Where to Get |
|---------|---------|--------------|
| Anthropic | AI provider | console.anthropic.com |
| Tailscale | Secure remote access | tailscale.com |
| Telegram | Chat interface | Create bot via @BotFather |
| WhatsApp Business | Chat interface | developers.facebook.com/apps |
| Shopify | BattleShirts orders | Partners dashboard |
| Printful | Fulfillment status | developers.printful.com |
| Vercel | Deployment status | vercel.com/account/tokens |
| Google/Microsoft | Calendar & Email | Cloud console / Azure portal |

### Time Estimate
- Phase 1 (Proxmox): 30 min
- Phase 2 (Home Assistant): 1-2 hours
- Phase 3 (MoltBot): 1-2 hours
- Phase 4-6 (Config & Integration): 2-3 hours
- **Total: ~1 day for basic setup**

---

## Build Phases

```
PHASE 1          PHASE 2          PHASE 3          PHASE 4          PHASE 5          PHASE 6
────────────────────────────────────────────────────────────────────────────────────────────►

[Proxmox    ] → [Home        ] → [MoltBot     ] → [Agent       ] → [Channels    ] → [Integrate  ]
[Foundation ]   [Assistant   ]   [Gateway     ]   [Config      ]   [Setup       ]   [Services   ]
                [VM          ]   [VM          ]   [             ]   [             ]   [           ]

• Network       • HAOS install   • Ubuntu VM      • battleshirts  • Telegram bot  • BattleShirts
• Storage       • SmartThings    • Node.js        • personal      • WhatsApp      • Calendar
• Tailscale     • Basic config   • PostgreSQL     • home          • Web UI        • Home Assist
                • Alexa link     • MoltBot code   • finance       • Voice (HA)    • Finance CSV
```

---

## Phase 1: Proxmox Foundation

### 1.1 Initial Proxmox Setup

If Proxmox isn't installed yet:
1. Download Proxmox VE ISO from proxmox.com
2. Flash to USB with Balena Etcher
3. Boot mini PC from USB, follow installer
4. Access web UI at `https://<mini-pc-ip>:8006`

### 1.2 Network Configuration

In Proxmox shell or via web UI (Datacenter → Node → Network):

```bash
# Verify network bridge exists
cat /etc/network/interfaces
```

Should have something like:
```
auto vmbr0
iface vmbr0 inet static
    address 192.168.1.100/24
    gateway 192.168.1.1
    bridge-ports enp1s0
    bridge-stp off
    bridge-fd 0
```

### 1.3 Storage Setup

For VM images and backups:

```bash
# Create directories for ISOs and backups
mkdir -p /var/lib/vz/template/iso
mkdir -p /var/lib/vz/dump
```

Upload ISOs via web UI: Datacenter → Storage → local → ISO Images → Upload

Download these ISOs:
- Ubuntu 22.04 Server: https://ubuntu.com/download/server
- Home Assistant OS: https://www.home-assistant.io/installation/linux

### 1.4 Install Tailscale on Proxmox Host

```bash
# On Proxmox host
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up

# Enable IP forwarding for VMs to use Tailscale
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
sysctl -p
```

Follow the auth URL to connect to your Tailnet.

---

## Phase 2: Home Assistant VM

### 2.1 Create HAOS VM

Home Assistant OS requires specific VM settings. In Proxmox:

**Option A: Use the official script (recommended)**
```bash
# On Proxmox host shell
bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/vm/haos-vm.sh)"
```

**Option B: Manual creation**

1. Download HAOS image:
```bash
cd /var/lib/vz/template/iso
wget https://github.com/home-assistant/operating-system/releases/download/11.4/haos_ova-11.4.qcow2.xz
xz -d haos_ova-11.4.qcow2.xz
```

2. Create VM (ID 101):
```bash
qm create 101 --name home-assistant --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0
qm importdisk 101 /var/lib/vz/template/iso/haos_ova-11.4.qcow2 local-lvm
qm set 101 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-101-disk-0
qm set 101 --boot c --bootdisk scsi0
qm set 101 --machine q35
qm set 101 --bios ovmf
qm set 101 --efidisk0 local-lvm:1,format=qcow2,efitype=4m,pre-enrolled-keys=1
qm start 101
```

### 2.2 Initial Home Assistant Setup

1. Find HA IP: Check Proxmox console or router DHCP
2. Access: `http://<ha-ip>:8123`
3. Create account, name your home
4. Set location (for sunrise/sunset automations)

### 2.3 Install Essential Add-ons

Go to Settings → Add-ons → Add-on Store:

- **File Editor** - Edit config files
- **Terminal & SSH** - Command line access
- **Samba Share** - Easy file access from laptop
- **Mosquitto Broker** - MQTT for device communication

### 2.4 SmartThings Integration

1. Settings → Devices & Services → Add Integration
2. Search "SmartThings"
3. You'll need a Personal Access Token from https://account.smartthings.com/tokens
4. Create token with all device permissions
5. Enter token, select location, import devices

Your existing SmartThings devices will appear in Home Assistant.

### 2.5 Alexa Integration

1. Install "Alexa Media Player" via HACS:
   - First install HACS: https://hacs.xyz/docs/setup/download
   - Then: HACS → Integrations → Search "Alexa Media Player" → Install

2. Configure:
   - Settings → Devices & Services → Add Integration → Alexa Media Player
   - Login with Amazon credentials
   - Devices appear as media players

### 2.6 Local Voice Processing (Optional but Recommended)

For privacy, process voice locally instead of cloud:

1. Install add-ons:
   - **Whisper** (speech-to-text)
   - **Piper** (text-to-speech)
   - **openWakeWord** (wake word detection)

2. Settings → Voice Assistants → Add Assistant
3. Configure with local Whisper/Piper

### 2.7 Create Long-Lived Access Token

For MoltBot to control Home Assistant:

1. Click your profile (bottom left)
2. Scroll to "Long-Lived Access Tokens"
3. Create Token → Name: "MoltBot"
4. **Save this token** - you'll need it for MoltBot config

---

## Phase 3: MoltBot Gateway VM

### 3.1 Create Ubuntu VM

In Proxmox web UI:

1. **Create VM** (ID 100)
   - Name: `moltbot-gateway`
   - ISO: Ubuntu 22.04 Server

2. **System**
   - Machine: q35
   - BIOS: SeaBIOS (simpler than UEFI)

3. **Disks**
   - Size: 32GB
   - Bus: VirtIO Block

4. **CPU**
   - Cores: 2

5. **Memory**
   - RAM: 4096 MB (4GB)

6. **Network**
   - Bridge: vmbr0
   - Model: VirtIO

### 3.2 Install Ubuntu

Boot VM, complete installation:
- Hostname: `moltbot`
- Username: `moltbot`
- Install OpenSSH: Yes

After install:
```bash
ip addr show  # Note the IP
```

### 3.3 Base System Setup

SSH into VM:
```bash
ssh moltbot@<vm-ip>
```

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install essentials
sudo apt install -y curl git build-essential postgresql jq ffmpeg

# Install Node.js 20
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Install pnpm
npm install -g pnpm

# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

### 3.4 Setup PostgreSQL

```bash
# Start PostgreSQL
sudo systemctl enable postgresql
sudo systemctl start postgresql

# Create database
sudo -u postgres psql << EOF
CREATE USER moltbot WITH PASSWORD 'choose-a-secure-password';
CREATE DATABASE moltbot OWNER moltbot;
GRANT ALL PRIVILEGES ON DATABASE moltbot TO moltbot;
EOF
```

### 3.5 Clone and Build MoltBot

```bash
cd ~
git clone https://github.com/defrostedcanuck/MoltBot.git
cd MoltBot
pnpm install
pnpm build
```

### 3.6 Create Directory Structure

```bash
mkdir -p ~/.moltbot/{config,logs,data,skills,imports}
mkdir -p ~/.moltbot/imports/finance  # For CSV imports
```

---

## Phase 4: Agent Configuration

### 4.1 Main Configuration File

Create `~/.moltbot/config/config.json5`:

```json5
{
  // ===========================================
  // GATEWAY SETTINGS
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
  // AGENT DEFAULTS
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

    // ===========================================
    // AGENT DEFINITIONS
    // ===========================================
    list: [
      // -----------------------------------------
      // BATTLESHIRTS AGENT
      // Business operations - lower security gates
      // -----------------------------------------
      {
        id: "battleshirts",
        name: "BattleShirts Ops",
        description: "Manages BattleShirts e-commerce operations",
        tools: {
          profile: "standard",
          allow: ["group:fs", "group:runtime", "group:memory", "message"],
        },
        context: {
          system: "You manage BattleShirts operations. You can check orders, monitor fulfillment, review analytics, and help with customer issues. Be proactive about flagging problems.",
        },
      },

      // -----------------------------------------
      // PERSONAL AGENT
      // Calendar, email, tasks - medium gates
      // -----------------------------------------
      {
        id: "personal",
        name: "Personal Assistant",
        description: "Calendar, email, meetings, reminders",
        tools: {
          profile: "standard",
          allow: ["group:fs", "group:memory", "message"],
          deny: ["group:runtime"],  // No exec by default
        },
        context: {
          system: "You help manage personal productivity - calendar, email summaries, meeting prep, reminders. Be concise and actionable.",
        },
      },

      // -----------------------------------------
      // HOME AGENT
      // Home automation - medium gates
      // -----------------------------------------
      {
        id: "home",
        name: "Home Control",
        description: "Home Assistant integration and automation",
        tools: {
          profile: "minimal",
          allow: ["group:memory", "message", "homeassistant"],
        },
        context: {
          system: "You control smart home devices via Home Assistant. For safety-critical actions (locks, alarms, garage), always confirm before acting.",
        },
      },

      // -----------------------------------------
      // FINANCE AGENT
      // Banking, investments - HIGH GATES, READ ONLY
      // -----------------------------------------
      {
        id: "finance",
        name: "Finance Tracker",
        description: "Read-only financial tracking and analysis",
        tools: {
          profile: "minimal",
          allow: ["group:memory", "message"],
          deny: ["group:runtime", "group:fs"],  // No exec, no file write
        },
        context: {
          system: "You analyze financial data from CSV imports. You NEVER initiate transactions, transfers, or trades. You provide summaries, trends, and insights. All data is read-only.",
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
      safeBins: ["jq", "grep", "curl", "git", "cat", "ls"],
    },
    elevated: {
      enabled: false,
    },
  },

  // ===========================================
  // INTEGRATIONS
  // ===========================================
  integrations: {
    homeAssistant: {
      url: "http://192.168.1.x:8123",  // Update with your HA IP
      token: "${HOME_ASSISTANT_TOKEN}",
    },
  },

  // ===========================================
  // DATABASE
  // ===========================================
  database: {
    url: "postgresql://moltbot:your-password@localhost:5432/moltbot",
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

### 4.2 Environment Variables

Create `~/.moltbot/.env`:

```bash
# AI Provider
ANTHROPIC_API_KEY=sk-ant-api03-xxxxx

# Home Assistant
HOME_ASSISTANT_TOKEN=eyJ0eXAiOiJKV1QiLCJhbGciOi...

# BattleShirts
SHOPIFY_STORE_DOMAIN=battleshirts.myshopify.com
SHOPIFY_ADMIN_TOKEN=shpat_xxxxx
PRINTFUL_API_KEY=xxxxx
VERCEL_TOKEN=xxxxx
BATTLESHIRTS_DB_URL=postgresql://readonly:xxx@ep-xxx.us-east-1.aws.neon.tech/battleshirts

# Communication Channels
TELEGRAM_BOT_TOKEN=123456789:ABCdefGHI...
WHATSAPP_PHONE_ID=123456789
WHATSAPP_ACCESS_TOKEN=EAAxxxxx

# Database (local)
DATABASE_URL=postgresql://moltbot:your-password@localhost:5432/moltbot
```

### 4.3 Exec Approvals Configuration

Create `~/.moltbot/exec-approvals.json`:

```json
{
  "version": 1,
  "defaults": {
    "security": "allowlist",
    "ask": "on-miss",
    "askFallback": "deny",
    "autoAllowSkills": false
  },
  "agents": {
    "battleshirts": {
      "security": "allowlist",
      "ask": "on-miss",
      "askFallback": "deny",
      "allowlist": [
        {"pattern": "/usr/bin/curl", "comment": "API calls"},
        {"pattern": "/usr/bin/jq", "comment": "JSON processing"},
        {"pattern": "/usr/bin/git", "comment": "Repo operations"}
      ]
    },
    "personal": {
      "security": "deny",
      "ask": "always",
      "askFallback": "deny",
      "allowlist": []
    },
    "home": {
      "security": "allowlist",
      "ask": "on-miss",
      "askFallback": "deny",
      "allowlist": [
        {"pattern": "/usr/bin/curl", "comment": "HA API only"}
      ]
    },
    "finance": {
      "security": "deny",
      "ask": "always",
      "askFallback": "deny",
      "allowlist": []
    }
  }
}
```

---

## Phase 5: Communication Channels

### 5.1 Telegram Bot Setup

1. Message @BotFather on Telegram
2. `/newbot` → Name: "MoltBot" → Username: `your_moltbot_bot`
3. Copy the token to your `.env`

Configure in MoltBot (add to config.json5):

```json5
channels: {
  telegram: {
    enabled: true,
    token: "${TELEGRAM_BOT_TOKEN}",
    allowedUsers: ["your_telegram_user_id"],  // Get via @userinfobot
    agentRouting: {
      default: "personal",
      prefixes: {
        "/biz": "battleshirts",
        "/home": "home",
        "/money": "finance",
      },
    },
  },
},
```

### 5.2 WhatsApp Business Setup

1. Go to developers.facebook.com → Create App → Business
2. Add WhatsApp product
3. Set up phone number (can use test number initially)
4. Get Phone Number ID and Access Token

Configure webhook to point to: `https://moltbot.your-tailnet.ts.net:3100/webhooks/whatsapp`

Add to config.json5:

```json5
channels: {
  // ... telegram config ...

  whatsapp: {
    enabled: true,
    phoneNumberId: "${WHATSAPP_PHONE_ID}",
    accessToken: "${WHATSAPP_ACCESS_TOKEN}",
    verifyToken: "your-verify-token",
    allowedNumbers: ["+1234567890"],  // Your phone number
    agentRouting: {
      default: "personal",
    },
  },
},
```

### 5.3 Web UI

The web UI is enabled by default on port 3101. Access via:
- Local: `http://192.168.1.x:3101`
- Tailscale: `http://moltbot.tailnet.ts.net:3101`

### 5.4 Voice via Home Assistant

MoltBot can receive voice commands routed through Home Assistant:

1. In Home Assistant, create an automation:
   - Trigger: Voice command containing "moltbot" or "assistant"
   - Action: REST command to MoltBot API

2. Add to HA `configuration.yaml`:
```yaml
rest_command:
  moltbot_command:
    url: "http://192.168.1.x:3100/api/message"
    method: POST
    headers:
      Content-Type: application/json
    payload: '{"text": "{{ command }}", "agent": "{{ agent }}"}'
```

---

## Phase 6: Integration Setup

### 6.1 BattleShirts Integration

Create skill file `~/.moltbot/skills/battleshirts.json5`:

```json5
{
  id: "battleshirts-ops",
  name: "BattleShirts Operations",
  agent: "battleshirts",

  tools: [
    {
      name: "get-recent-orders",
      description: "Fetch recent Shopify orders",
      type: "http",
      config: {
        method: "GET",
        url: "https://${SHOPIFY_STORE_DOMAIN}/admin/api/2025-01/orders.json",
        headers: {
          "X-Shopify-Access-Token": "${SHOPIFY_ADMIN_TOKEN}",
        },
        params: {
          status: "any",
          limit: 20,
        },
      },
    },
    {
      name: "get-printful-orders",
      description: "Check Printful fulfillment status",
      type: "http",
      config: {
        method: "GET",
        url: "https://api.printful.com/orders",
        headers: {
          "Authorization": "Bearer ${PRINTFUL_API_KEY}",
        },
      },
    },
    {
      name: "query-generations",
      description: "Query battle generations from database",
      type: "sql",
      config: {
        connectionString: "${BATTLESHIRTS_DB_URL}",
        readOnly: true,
      },
      permissions: {
        requireApproval: true,
      },
    },
  ],
}
```

### 6.2 Home Assistant Integration

Create skill file `~/.moltbot/skills/home.json5`:

```json5
{
  id: "home-control",
  name: "Home Control",
  agent: "home",

  tools: [
    {
      name: "get-states",
      description: "Get all device states",
      type: "http",
      config: {
        method: "GET",
        url: "${HOME_ASSISTANT_URL}/api/states",
        headers: {
          "Authorization": "Bearer ${HOME_ASSISTANT_TOKEN}",
        },
      },
    },
    {
      name: "control-device",
      description: "Turn device on/off or set state",
      type: "http",
      config: {
        method: "POST",
        url: "${HOME_ASSISTANT_URL}/api/services/{{domain}}/{{service}}",
        headers: {
          "Authorization": "Bearer ${HOME_ASSISTANT_TOKEN}",
          "Content-Type": "application/json",
        },
      },
      permissions: {
        // Safe actions auto-allowed
        autoAllow: ["light.turn_on", "light.turn_off", "switch.turn_on", "switch.turn_off"],
        // Security actions require approval
        requireApproval: ["lock.*", "alarm_control_panel.*", "cover.open_cover"],
      },
    },
    {
      name: "run-scene",
      description: "Activate a scene",
      type: "http",
      config: {
        method: "POST",
        url: "${HOME_ASSISTANT_URL}/api/services/scene/turn_on",
        headers: {
          "Authorization": "Bearer ${HOME_ASSISTANT_TOKEN}",
        },
      },
    },
  ],
}
```

### 6.3 Finance Import Setup

Create directory and import script:

```bash
mkdir -p ~/.moltbot/imports/finance
```

Create `~/.moltbot/skills/finance.json5`:

```json5
{
  id: "finance-tracker",
  name: "Finance Tracker",
  agent: "finance",

  dataSources: [
    {
      id: "bank-transactions",
      type: "csv",
      path: "~/.moltbot/imports/finance/bank-*.csv",
      schema: {
        date: "Date",
        description: "Description",
        amount: "Amount",
        category: "Category",
      },
      refreshOn: "file-change",
    },
    {
      id: "investments",
      type: "csv",
      path: "~/.moltbot/imports/finance/portfolio-*.csv",
      schema: {
        symbol: "Symbol",
        shares: "Quantity",
        costBasis: "Cost Basis",
        currentValue: "Market Value",
      },
      refreshOn: "file-change",
    },
  ],

  tools: [
    {
      name: "analyze-spending",
      description: "Analyze spending patterns from imported transactions",
      type: "query",
      dataSource: "bank-transactions",
    },
    {
      name: "portfolio-summary",
      description: "Summarize investment portfolio from imported data",
      type: "query",
      dataSource: "investments",
    },
  ],

  // CRITICAL: No write/action capabilities
  readOnly: true,
}
```

**To import data:**
1. Export CSV from your bank/broker
2. Copy to `~/.moltbot/imports/finance/`
3. MoltBot will auto-detect and parse

---

## Security Model

### Permission Matrix

| Agent | Read Data | Write Data | Exec Commands | External API | Approval Level |
|-------|-----------|------------|---------------|--------------|----------------|
| battleshirts | Yes | Limited | Allowlist | Yes | Low |
| personal | Yes | Approval | No | Yes | Medium |
| home | HA states | HA control | Curl only | HA only | Medium (high for security) |
| finance | Imports | **NEVER** | **NEVER** | **NEVER** | Maximum |

### Approval Escalation

```
Action Type               → Required Approval
─────────────────────────────────────────────
Read public data          → None
Read business data        → None
Read personal data        → None (it's your data)
Read financial data       → None (it's your data)

Write business data       → Auto (battleshirts agent)
Write personal data       → Confirm via chat
Write home state          → Auto for lights, confirm for locks
Write financial data      → BLOCKED

Exec allowlisted cmd      → Auto
Exec unknown cmd          → Confirm via Telegram/WhatsApp
Exec any cmd (finance)    → BLOCKED

Send notification         → Auto
Send email                → Confirm
Make purchase             → BLOCKED (you do it yourself)
```

### Channel Security

Ensure only YOU can message MoltBot:

- **Telegram**: Allowlist your user ID
- **WhatsApp**: Allowlist your phone number
- **Web UI**: Tailscale-only access (no public exposure)
- **Voice**: Only responds to wake word from your HA

---

## Running MoltBot

### Start Manually (for testing)

```bash
cd ~/MoltBot
source ~/.moltbot/.env
pnpm start
```

### Run as Systemd Service

Create `/etc/systemd/system/moltbot.service`:

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

# Security
NoNewPrivileges=true
ProtectSystem=strict
ReadWritePaths=/home/moltbot/.moltbot

[Install]
WantedBy=multi-user.target
```

Enable:
```bash
sudo systemctl daemon-reload
sudo systemctl enable moltbot
sudo systemctl start moltbot
sudo systemctl status moltbot
```

---

## Maintenance & Backup

### Proxmox Snapshots

Before any major change:
```bash
# On Proxmox host
qm snapshot 100 before-update --description "Pre-update snapshot"
qm snapshot 101 ha-working --description "HA working config"
```

### MoltBot Updates

```bash
cd ~/MoltBot
git pull
pnpm install
pnpm build
sudo systemctl restart moltbot
```

### Log Rotation

Create `/etc/logrotate.d/moltbot`:
```
/home/moltbot/.moltbot/logs/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
}
```

---

## Troubleshooting

### MoltBot won't start
```bash
sudo systemctl status moltbot
journalctl -u moltbot -f
cd ~/MoltBot && pnpm start  # Run manually to see errors
```

### Can't connect via Tailscale
```bash
tailscale status
sudo tailscale up --reset
```

### Home Assistant API errors
```bash
# Test HA connection
curl -H "Authorization: Bearer $HOME_ASSISTANT_TOKEN" http://192.168.1.x:8123/api/
```

### Telegram bot not responding
```bash
# Check webhook
curl https://api.telegram.org/bot$TELEGRAM_BOT_TOKEN/getWebhookInfo
```

---

## What To Do First (Priority Order)

### Day 1: Foundation
1. ✅ Proxmox network and storage configured
2. ✅ Tailscale installed on Proxmox host
3. ⬜ Create Home Assistant VM, complete initial setup
4. ⬜ Create MoltBot VM, install Ubuntu + dependencies

### Day 2: Core Setup
5. ⬜ Clone MoltBot, build, create config structure
6. ⬜ Set up PostgreSQL database
7. ⬜ Configure basic agents (start with battleshirts only)
8. ⬜ Test MoltBot starts and responds locally

### Day 3: Communication
9. ⬜ Set up Telegram bot
10. ⬜ Configure Telegram channel in MoltBot
11. ⬜ Test end-to-end: message Telegram → MoltBot responds

### Week 1: Integration
12. ⬜ Connect BattleShirts APIs (Shopify, Printful)
13. ⬜ SmartThings → Home Assistant migration
14. ⬜ Home Assistant ↔ MoltBot integration
15. ⬜ WhatsApp setup (if needed)

### Week 2+: Expansion
16. ⬜ Add personal agent (calendar, email)
17. ⬜ Set up finance CSV import workflow
18. ⬜ Voice command integration
19. ⬜ Tune approval thresholds based on experience

---

## Quick Reference Commands

```bash
# SSH to MoltBot
ssh moltbot@moltbot.tailnet.ts.net

# View logs
journalctl -u moltbot -f
tail -f ~/.moltbot/logs/gateway.log

# Restart services
sudo systemctl restart moltbot
sudo systemctl restart postgresql

# Proxmox snapshots
qm snapshot 100 snapshot-name
qm rollback 100 snapshot-name

# Test APIs
curl -s http://localhost:3100/health
curl -s http://192.168.1.x:8123/api/ -H "Authorization: Bearer $HA_TOKEN"
```

---

*Last updated: February 2026*
