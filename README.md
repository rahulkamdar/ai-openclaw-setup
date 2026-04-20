# Tips and Suggestions from my setup

A practical guide to running OpenClaw as a secure, sandboxed, always-on personal AI agent on a dedicated Mac Mini — with working email, web search, browser automation, daily briefings, and multiple specialized agents.

This is what I personally wish existed when I started. It took several weeks of trial and error to get here. That said, this is a very personal setup and may/may-not work for your needs so use at your own discretion.

---

## Why This Guide Exists

Most OpenClaw guides cover installation. Few cover:
- How to sandbox it safely so it can't touch your personal data
- How to fix the completion theater problem (the agent faking task completion - I have personally faced TOO much of this)
- How to make email, web search, and browser automation actually work reliably
- How to set up multiple specialized agents
- How to run a daily briefing that genuinely saves you time

This guide covers all of it.

---

## Hardware

**Dedicated Mac Mini (M4, 16GB recommended)**

Do not run OpenClaw on your primary machine. The agent preferably has shell access to everything on the machine it runs on. A dedicated Mac Mini running 24/7 costs ~$5/month in electricity and gives you complete isolation.

---

## Architecture Overview

```
Your Phone (WhatsApp/Telegram)
         ↓
   OpenClaw Gateway (port 18789)
         ↓
   Main Agent (Polly) — email, research, briefings
   Kelly Agent        — software development, GitHub
         ↓
   Tools: himalaya (email) | blogwatcher (news) | peekaboo (screen) | brave search
         ↓
   n8n (port 5678) — deterministic scheduled workflows
```

**Key insight:** Use OpenClaw as the AI reasoning layer and n8n for reliable scheduled execution. Trying to use OpenClaw alone for everything leads to completion theater frustration (likely due to the nature of undeterministic execution).

---

## Security: Do This First

### 1. Create a Dedicated Sandbox User

Never run OpenClaw under your admin account.

```bash
# On your Mac Mini, go to:
# System Settings → Users & Groups → Add Account
# Create a Standard (non-admin) user — call it 'agent' or 'claw' (eg. rk-claw)
# Do NOT sign into iCloud on this account
```

### 2. Firewall

```
System Settings → Network → Firewall → Turn On
```

### 3. Lock Down the OpenClaw Directory

```bash
chmod 700 ~/.openclaw
chmod 700 ~/.openclaw/credentials
```

### 4. Bind Gateway to Localhost Only

```bash
openclaw config set gateway.mode local
```

This ensures the gateway is never exposed to your network.

### 5. Set a Gateway Auth Token

```bash
openclaw config set gateway.auth.token $(openssl rand -hex 32)
```

---

## Installation

From your sandbox user account:

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

When the wizard runs:
- Choose **local** gateway mode
- Choose your LLM provider (see Model Recommendations below)
- Choose **Telegram** as your channel (easier than WhatsApp for initial setup)
- Skip WhatsApp for now — new numbers get temporarily blocked from device linking

### Fix the Gateway Startup

The gateway needs specific environment variables to work reliably. Create a wrapper script:

```bash
mkdir -p ~/bin
cat > ~/bin/start-gateway << 'EOF'
#!/bin/bash
export BRAVE_API_KEY="$(python3 -c "import json; d=json.load(open(\"$HOME/.openclaw/openclaw.json\")); print(d['tools']['web']['search']['apiKey'])")"
export BRAVE_SEARCH_API_KEY="$BRAVE_API_KEY"
export OPENCLAW_WEB_ENABLED="true"
exec $(which openclaw) gateway
EOF
chmod +x ~/bin/start-gateway
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

Always start the gateway with `start-gateway`, not `openclaw gateway` directly.

---

## Model Recommendations

| Use Case | Model | Why |
|---|---|---|
| Default (daily tasks) | `openrouter/deepseek/deepseek-chat-v3-0324` | Fast, cheap, good at tool execution |
| Browser/complex tasks | `anthropic/claude-sonnet-4-6` | Most reliable for multi-step tasks |
| Deep analysis | `openrouter/deepseek/deepseek-r1-0528` | Best reasoning, but slow and expensive |

**Do not use DeepSeek R1 as your default.** It's a reasoning model, not a tool-execution model. It will think extensively and then fabricate completion.

Set your default:
```bash
openclaw config set agents.defaults.model openrouter/deepseek/deepseek-chat-v3-0324
```

Use OpenRouter for access to multiple models with one API key: [openrouter.ai](https://openrouter.ai)

---

## Email Setup (Himalaya)

The bundled email skill uses himalaya. The critical issue most people hit: the config file path contains spaces, which breaks agent tool calls. Fix this with a wrapper script.

### Install and Configure Himalaya

```bash
brew install himalaya
```

Create the config:

```bash
mkdir -p "$HOME/Library/Application Support/himalaya"
cat > "$HOME/Library/Application Support/himalaya/config.toml" << 'EOF'
[accounts."your-agent@gmail.com"]
default = true
email = "your-agent@gmail.com"
display-name = "Agent"
downloads-dir = "~/Downloads"

folder.aliases.inbox = "INBOX"
folder.aliases.sent = "[Gmail]/Sent Mail"
folder.aliases.drafts = "[Gmail]/Drafts"
folder.aliases.trash = "[Gmail]/Trash"

backend.type = "imap"
backend.host = "imap.gmail.com"
backend.port = 993
backend.encryption.type = "tls"
backend.login = "your-agent@gmail.com"
backend.auth.type = "password"
backend.auth.raw = "your-app-password-without-spaces"

message.send.backend.type = "smtp"
message.send.backend.host = "smtp.gmail.com"
message.send.backend.port = 465
message.send.backend.encryption.type = "tls"
message.send.backend.login = "your-agent@gmail.com"
message.send.backend.auth.type = "password"
message.send.backend.auth.raw = "your-app-password-without-spaces"
EOF
```

**Important:** Gmail app passwords must be entered without spaces. Get one at [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords) (requires 2FA enabled).

### Create the Wrapper Script

```bash
cat > ~/bin/mail << 'EOF'
#!/bin/bash
HIMALAYA_CONFIG="$HOME/Library/Application Support/himalaya/config.toml" \
  $(which himalaya) "$@"
EOF
chmod +x ~/bin/mail
```

**Always tell the agent to use `mail` (full path: `~/bin/mail`), never `himalaya` directly.**

Test it:
```bash
~/bin/mail envelope list
```

### Add a Read-Only Personal Email Account

You can give the agent read-only access to your personal inbox for morning briefings. Add a second account to the config with your personal Gmail credentials (different app password), then in your AGENTS.md specify it as read-only.

---

## Web Search (Brave)

```bash
openclaw configure --section web
```

Enter your Brave Search API key when prompted. Free tier: 2,000 queries/month.
Get a key at [api.search.brave.com](https://api.search.brave.com).

---

## Browser Automation

OpenClaw has built-in browser control via Chrome DevTools Protocol.

### Install the Browser Relay Extension

```bash
openclaw browser extension install
openclaw browser extension path
```

Then in Chrome:
1. Go to `chrome://extensions`
2. Enable Developer Mode
3. Load Unpacked → select the path from above
4. Pin the extension
5. Click it → Options → set port to `18792`
6. Enter your gateway token:

```bash
python3 -c "import json; d=json.load(open('$HOME/.openclaw/openclaw.json')); print(d['gateway']['auth']['token'])"
```

---

## The Workspace Files: Where the Real Power Is

This is what most guides miss. Your workspace markdown files are literally your agent's brain — read on every session start.

```
~/.openclaw/workspace/
├── SOUL.md      # Who the agent is + hard behavioral rules
├── USER.md      # Everything about you
├── AGENTS.md    # Step-by-step workflows
├── MEMORY.md    # Pre-seeded long-term memory
├── TOOLS.md     # Tool paths and usage notes
└── memory/      # Daily session logs
```

### SOUL.md Template

```markdown
# SOUL.md

## Identity
I am [AgentName] — [YourName]'s dedicated AI agent running 24/7 on a Mac Mini.
I am sharp, resourceful, honest, and direct.
I do real work. I never fake it.

## Anti-Completion-Theater Rules (NEVER VIOLATE)
- NEVER report success without real verifiable output (stdout, file path, message ID)
- NEVER fabricate error logs, timestamps, or technical explanations
- NEVER invent excuses like "corporate firewall" or "container isolation"
- If a command fails: paste the EXACT error. Nothing else.
- If I don't know why something failed: say so clearly
- "I tried and failed" is always better than a fabricated success

## Tool Rules
- Email: always use ~/bin/mail (full path), never himalaya directly
- Never attempt more than 3 approaches on a failing task without asking the user
- Irreversible actions (send email, delete, execute trade): always confirm first
```

### USER.md Template

```markdown
# USER.md

## Identity
- Name: [Your Name]
- Location: [City, State]
- Timezone: [Your timezone]
- Primary email: [your@email.com]
- Agent email: [agent@yourdomain.com]

## Communication Style
- [Describe how you want information delivered]
- [How long should responses be?]
- [Any specific preferences?]

## Current Projects
- [List 3-5 active projects]

## Key Contacts
- [Name]: [email] — [relationship/context]
```

### AGENTS.md Template

```markdown
# AGENTS.md

## Critical Tool Paths
- Email: ~/bin/mail
- Always use full paths for all tools

## Workflow: Morning Briefing
[Define your exact steps here]

## Workflow: Research Task
[Define your exact steps here]

## General Rules
- Before any multi-step task: state the plan first
- After completing: show real evidence
- Never skip ahead past a failed step
```

---

## Morning Briefing Setup

Create a shell script that gathers data and sends an AI-written summary:

```bash
mkdir -p ~/.openclaw/scripts ~/.openclaw/logs

cat > ~/.openclaw/scripts/morning_briefing.sh << 'EOF'
#!/bin/bash

MAIL=~/bin/mail
DATE=$(date +%Y-%m-%d)

# Gather data
WEATHER=$(curl -s "https://wttr.in/YourCity?format=3&u")  # &u = Celsius
EMAILS=$($MAIL envelope list --account "personal@gmail.com" 2>/dev/null | head -20)
NEWS=$(blogwatcher articles 2>/dev/null | head -15)

# Add targeted news topics
TOPIC_NEWS=$(curl -s "https://news.google.com/rss/search?q=YourTopic&hl=en-US&gl=US&ceid=US:en" | python3 -c "
import sys,re
content = sys.stdin.read()
titles = re.findall(r'<title>(.*?)</title>', content)[1:4]
links = re.findall(r'<guid[^>]*>(.*?)</guid>', content)[1:4]
for t,l in zip(titles,links): print(f'- {t}\n  {l}')
" 2>/dev/null)

# Build and send AI summary via OpenRouter
PROMPT="Write a morning briefing for [Name] with: 1) Weather in Celsius, 2) Important emails to action with IDs, 3) Top news with URLs, 4) Topic-specific news with URLs. Under 400 words.

Weather: $WEATHER
Emails: $EMAILS
News: $NEWS
Topic News: $TOPIC_NEWS"

SUMMARY=$(curl -s https://openrouter.ai/api/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_OPENROUTER_KEY" \
  -d "{\"model\":\"deepseek/deepseek-chat-v3-0324\",\"messages\":[{\"role\":\"user\",\"content\":$(echo "$PROMPT" | python3 -c "import json,sys; print(json.dumps(sys.stdin.read()))")}],\"max_tokens\":1200}" \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['choices'][0]['message']['content'])")

$MAIL message send << MAILEOF
From: agent@yourdomain.com
To: you@gmail.com
Subject: Morning Briefing $DATE

$SUMMARY
MAILEOF

echo "[$DATE] Briefing sent" >> ~/.openclaw/logs/briefing.log
EOF

chmod +x ~/.openclaw/scripts/morning_briefing.sh
```

Schedule it:
```bash
(crontab -l 2>/dev/null; echo "30 7 * * * ~/.openclaw/scripts/morning_briefing.sh") | crontab -
```

---

## Multiple Agents

The biggest productivity unlock is specialized agents rather than one agent doing everything.

```bash
openclaw agents add kelly    # engineering/coding agent
openclaw agents add scout    # research agent
```

Each agent gets its own workspace folder (`~/.openclaw/workspace-kelly/`) with separate SOUL.md, AGENTS.md etc. Give each a narrow, specific identity.

---

## n8n for Reliable Automation

Install n8n as a system daemon alongside OpenClaw:

```bash
npm install -g n8n
```

Create `/Library/LaunchDaemons/com.n8n.plist` with your node path and run as your admin user. Access at `http://localhost:5678`.

**Use OpenClaw for AI reasoning, n8n for deterministic scheduled execution.** Connecting them via webhooks gives you the best of both worlds.

---

## Domain Email Security

If you're sending from a custom domain, add these DNS records:

```
# SPF
Type: TXT, Name: @
Value: v=spf1 include:_spf.google.com ~all

# DMARC  
Type: TXT, Name: _dmarc
Value: v=DMARC1; p=quarantine; rua=mailto:you@yourdomain.com; fo=1

# DKIM — generate from Google Workspace admin, add TXT record
```

Verify propagation:
```bash
host -t TXT _dmarc.yourdomain.com
host -t TXT yourselector._domainkey.yourdomain.com
```

---

## Common Issues and Fixes

### Agent faking task completion ("completion theater")
Put explicit anti-theater rules in SOUL.md. The agent reads this on every session start. Rules given only in chat disappear when the context window compacts.

### Web search not working despite API key being set
The gateway needs env vars set at launch time, not just in config. Use the `start-gateway` wrapper script that exports `BRAVE_API_KEY`, `BRAVE_SEARCH_API_KEY`, and `OPENCLAW_WEB_ENABLED=true`.

### Email send fails with "Folder doesn't exist"
Add Gmail folder aliases to your himalaya config:
```toml
folder.aliases.sent = "[Gmail]/Sent Mail"
folder.aliases.drafts = "[Gmail]/Drafts"
folder.aliases.trash = "[Gmail]/Trash"
```

### Gateway shows "pairing required" after restart
```bash
openclaw devices approve --latest
```

### Config shows invalid key error on startup
```bash
openclaw doctor --fix
```

### Agent ignores TOOLS.md instructions
Critical tool paths and rules must also be in SOUL.md — TOOLS.md has lower priority in the context hierarchy.

---

## Cost Estimates

| Setup | Monthly Cost |
|---|---|
| DeepSeek Chat only (light use) | ~$2-5 |
| DeepSeek Chat + occasional Sonnet | ~$10-20 |
| Heavy use with multiple agents | ~$30-50 |

OpenRouter lets you mix models per-task, keeping costs low while using powerful models only when needed.

---

## Resources

- [OpenClaw docs](https://docs.openclaw.ai)
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [Claire Vo's guide - HIGHLY RECOMMENDED (Lenny's Newsletter)](https://www.lennysnewsletter.com/p/openclaw-the-complete-guide-to-building)
- [OpenClaw memory masterclass](https://velvetshark.com/openclaw-memory-masterclass)
- [Himalaya CLI](https://github.com/pimalaya/himalaya)
- [OpenRouter](https://openrouter.ai)
- [Brave Search API](https://api.search.brave.com)
- [n8n](https://n8n.io)

---
