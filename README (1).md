<div align="center">

# 📈 Server Pulse

**Live vitals and a year of history for your Ubuntu server — from anywhere, on free tiers.**

btop + Netdata on the server · a Next.js dashboard on Vercel · history in Neon Postgres · Telegram & email alerts

[![Next.js](https://img.shields.io/badge/Next.js-App_Router-000000?logo=nextdotjs&logoColor=white)](https://nextjs.org)
[![TypeScript](https://img.shields.io/badge/TypeScript-strict-3178C6?logo=typescript&logoColor=white)](https://www.typescriptlang.org)
[![Vercel](https://img.shields.io/badge/Vercel-Hobby-000000?logo=vercel&logoColor=white)](https://vercel.com)
[![Neon](https://img.shields.io/badge/Neon-Postgres-00E599?logo=postgresql&logoColor=white)](https://neon.com)
[![Netdata](https://img.shields.io/badge/Netdata-metrics-00AB44?logo=netdata&logoColor=white)](https://www.netdata.cloud)
[![Tailscale](https://img.shields.io/badge/Tailscale-Funnel-242424?logo=tailscale&logoColor=white)](https://tailscale.com/kb/1223/funnel)
[![Cost](https://img.shields.io/badge/cost-%240%2Fmonth-brightgreen)](#-free-tier-budget)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue)](#-license)

</div>

---

## 📑 Table of contents

- [What it does](#-what-it-does)
- [Architecture](#-architecture)
- [Tech stack](#-tech-stack)
- [Free-tier budget](#-free-tier-budget)
- [Project structure](#-project-structure)
- [Quick start](#-quick-start)
- [Part 1 — Server kit (Ubuntu)](#-part-1--server-kit-ubuntu)
- [Part 2 — Dashboard (Vercel)](#-part-2--dashboard-vercel)
- [Local development](#-local-development)
- [Configuration reference](#-configuration-reference)
- [How history works](#-how-history-works)
- [Data model](#-data-model)
- [API reference](#-api-reference)
- [Alerts](#-alerts)
- [30-day email report](#-30-day-email-report)
- [Security](#-security)
- [Troubleshooting](#-troubleshooting)
- [Roadmap](#-roadmap)
- [License](#-license)

---

## ✨ What it does

Server Pulse has two parts that only talk over one HTTPS link:

| Part | Runs on | What you get |
| --- | --- | --- |
| **Server kit** | Your Ubuntu machine | btop live in the browser, Netdata metrics every second with a 30-day local store, one login via Caddy, a free tunnel (Tailscale Funnel or ngrok), real-time Telegram/email alerts, and a 30-day usage report by email |
| **Dashboard** | Vercel (free) | Live KPI cards and charts, history from 1 hour to 1 year, monthly reports, CSV export, owner + viewer passwords, installable as a phone app |

**Highlights**

- 🔗 **One-link setup** — paste `SERVER_URL` into Vercel and the dashboard works.
- ⚡ **Live** — CPU, RAM, load, disk, network and temperature refresh every 2 s.
- 🗄️ **History that outlives the server** — copied nightly into Postgres, 1-minute detail for 30 days and hourly for a year or more.
- 📊 **Reports** — monthly averages and peaks to decide when to upgrade.
- 🔔 **Alerts** — Telegram and email the moment a threshold is crossed.
- 🔒 **Read-only by design** — nothing in the dashboard can change the server.
- 💸 **$0/month** — every service stays inside its free tier.
- 🧪 **Demo mode** — runs locally with generated data before you connect anything.

---

## 🏗️ Architecture

```mermaid
flowchart LR
  B["👤 Browser or installed PWA"]

  subgraph S["🖥️ Ubuntu server — server kit"]
    TU["Tunnel agent<br/>Tailscale Funnel or ngrok<br/>outbound only, no open ports"]
    CA["Caddy :8080<br/>basic auth<br/>/ → btop · /history → Netdata"]
    TT["btop + ttyd :7681<br/>read-only live terminal"]
    ND["Netdata :19999<br/>1 s metrics · 30-day local DB"]
    AL["Netdata health alarms"]
    RP["30-day report timer<br/>Python → SMTP"]
    TU --> CA
    CA --> TT
    CA --> ND
    ND --> AL
    ND --> RP
  end

  subgraph C["☁️ Vercel — cloud dashboard"]
    UI["Next.js pages<br/>Live · History · Reports · Settings"]
    API["Route handlers<br/>/api/live · /api/history · /api/cron/sync"]
    CR["Vercel Cron<br/>daily 00:15 UTC"]
    DB[("Neon Postgres<br/>samples · sync_runs · settings")]
    UI --> API
    CR --> API
    API --> DB
  end

  B --> UI
  B -. "btop at /" .-> TU
  API -- "HTTPS + basic auth" --> TU
  AL --> TG["📱 Telegram / email"]
  RP --> M["📬 Mailbox"]
```

**Key ideas**

1. **The server never accepts inbound connections.** The tunnel agent dials out; every request (browser or Vercel) arrives through it and must pass Caddy's login.
2. **Netdata is the single source of truth.** It keeps 30 days of per-second data on the server, so the dashboard can copy history once a day without missing anything.
3. **The browser never talks to the server.** All server calls run in Vercel route handlers, so credentials never reach the client.
4. **The two sides are independent.** Alerts and the monthly email run on the server and keep working if Vercel is down; the dashboard keeps its history if the server dies.

### Live data flow

```mermaid
sequenceDiagram
  participant B as Browser
  participant V as Vercel /api/live
  participant T as Tunnel
  participant N as Netdata
  loop Every 2 s (10 s in ngrok budget mode), only while the tab is visible
    B->>V: GET /api/live
    alt Snapshot younger than the refresh interval
      V-->>B: Shared cached snapshot
    else
      V->>T: GET /history/api/v1/allmetrics (basic auth)
      T->>N: Proxied by Caddy
      N-->>V: Latest values (JSON)
      V-->>B: Normalized KPIs
    end
  end
```

### Nightly history sync

```mermaid
sequenceDiagram
  participant C as Vercel Cron (00:15 UTC)
  participant S as /api/cron/sync
  participant N as Netdata (via tunnel)
  participant D as Neon Postgres
  C->>S: GET with Bearer CRON_SECRET
  loop Each tracked chart
    S->>N: /api/v1/data — last 26 h at 60 s
    N-->>S: About 1,560 rows
    S->>D: Batched upsert, ON CONFLICT DO NOTHING
  end
  S->>D: Roll up rows older than 30 days to hourly
  S->>D: Record the run in sync_runs
  opt Server unreachable for 1 day or more
    S-->>S: Email the owner once via SMTP
  end
```

---

## 🧰 Tech stack

### Server kit (Ubuntu 22.04 / 24.04)

| Layer | Choice | Why | Alternative |
| --- | --- | --- | --- |
| Metrics + local DB | **Netdata** | Per-second metrics with zero config, tiered on-disk database, REST API | Prometheus + node_exporter |
| Live terminal view | **btop** via **ttyd 1.7.7** | Best-looking live process view; ttyd is read-only by default | Glances web mode |
| Login + routing | **Caddy** | One basic-auth login for `/` and `/history/`, separate user for the app | Nginx |
| Public access | **Tailscale Funnel** (default) | Free, stable `*.ts.net` HTTPS URL, no open ports, no published request quota | ngrok free in budget mode |
| Supervision | **systemd** services + timers | Starts at boot, restarts on crash | Docker Compose |
| Alerts | **Netdata health alarms** → Telegram + email | Real-time, runs on the server | ntfy.sh |
| Monthly email | **Python 3** standard library | Nothing to install, any SMTP | msmtp |

### Dashboard (Vercel)

| Layer | Choice | Why | Alternative |
| --- | --- | --- | --- |
| Framework | **Next.js** (App Router) + **TypeScript** | Pages and API routes in one repo, first-class on Vercel | Remix, SvelteKit |
| UI | **Tailwind CSS** + **shadcn/ui** | Fast, consistent, dark mode built in | Mantine |
| Charts | **Recharts** | Sparklines and line charts up to a few thousand points | Apache ECharts, uPlot |
| Data fetching | **TanStack Query** | Polling, pause on hidden tab, retries | SWR |
| Database | **Neon Postgres** (Vercel Marketplace) | Free tier, one-click connect, scales to zero | Supabase |
| ORM + migrations | **Drizzle ORM** + drizzle-kit | Typed SQL, migrations run at build | Prisma |
| Validation | **Zod** | Validates Netdata responses and API input | Valibot |
| Auth | Signed httpOnly cookie (**jose**) | Owner + viewer passwords, no user table | Auth.js |
| Scheduling | **Vercel Cron** (daily) | Free on Hobby | GitHub Actions schedule |
| Email (offline alert) | **Nodemailer** + Gmail App Password | Free SMTP | Resend free tier |
| PWA | Web manifest + service worker | Installable on Android and iPhone | — |
| CI/CD | **GitHub → Vercel** Git integration | Every push to `main` deploys | Vercel CLI |

---

## 💸 Free-tier budget

| Service | Free limit | How Server Pulse stays inside it |
| --- | --- | --- |
| Vercel Hobby cron | [Once per day, ±59 min](https://vercel.com/docs/cron-jobs/usage-and-pricing) | Nightly job pulls 26 h from Netdata's own 30-day store |
| Vercel functions | Per-request time cap | Backfill runs one chart per request; every route stays well under 60 s |
| Tailscale Funnel | Free on the Personal plan; unpublished bandwidth limits | Only small JSON passes through it |
| ngrok free | [20,000 requests + 1 GB out per month](https://ngrok.com/docs/pricing-limits/free-plan-limits) | **Budget mode**: 10 s shared polling, usage meter, live view pauses at 80 % (≈ 54 h of live viewing a month) |
| Neon free | [100 CU-hours + 0.5 GB per project](https://neon.com/pricing), suspends after 5 min idle | Live view never touches the DB; hourly rollup keeps a year near 100 MB |
| Telegram / Gmail SMTP | Free | Alerts fire only on state changes; offline email once per outage |

> **Personal use.** Vercel Hobby and Tailscale Personal are for non-commercial use. For business use, keep ngrok budget mode for the tunnel and host the app on a commercial-friendly free tier (e.g. Cloudflare Workers via OpenNext).

---

## 📁 Project structure

```text
server-pulse/
├── app/
│   ├── login/page.tsx            # Password login (owner or viewer)
│   ├── page.tsx                  # Live
│   ├── history/page.tsx          # History charts 1h → 1y
│   ├── reports/page.tsx          # Monthly table
│   ├── settings/page.tsx         # Link, sync, backfill, usage (owner only)
│   └── api/
│       ├── live/route.ts         # One allmetrics call, shared cache
│       ├── history/route.ts      # Bucketed series (Postgres or Netdata)
│       ├── reports/route.ts      # Monthly aggregates
│       ├── backfill/route.ts     # 30-day import, one chart per call
│       ├── cron/sync/route.ts    # Nightly 26 h sync + rollup + offline email
│       ├── login/route.ts
│       ├── logout/route.ts
│       └── settings/
│           ├── link/route.ts     # Save server link (masked)
│           └── test/route.ts     # Connection test
├── components/                   # KPI cards, charts, range picker, ui/ (shadcn)
├── lib/
│   ├── netdata.ts                # The ONLY module that talks to the server
│   ├── config.ts                 # SERVER_URL parsing, tunnel detection, limits
│   ├── auth.ts                   # Sessions, owner/viewer roles
│   ├── sync.ts                   # Sync, backfill, rollup logic
│   ├── mail.ts                   # Offline email (Nodemailer)
│   ├── demo.ts                   # Generated demo data
│   └── db/
│       ├── schema.ts             # Drizzle schema
│       └── index.ts              # Neon client
├── drizzle/                      # Generated SQL migrations
├── scripts/migrate.mjs           # Runs migrations at build (skips without a DB)
├── public/                       # manifest.webmanifest, icons, sw.js
├── middleware.ts                 # Protects every route except login + cron
├── vercel.json                   # Cron schedule
├── .env.example
└── README.md
```

---

## 🚀 Quick start

1. **Server** — run [Part 1](#-part-1--server-kit-ubuntu) on your Ubuntu machine. You end up with a link like `https://dashboard:PASS@my-server.tailnet-name.ts.net/history`.
2. **Dashboard** — click **Import** on [vercel.com/new](https://vercel.com/new), add that link as `SERVER_URL`, connect Neon, deploy ([Part 2](#-part-2--dashboard-vercel)).
3. **Open the site**, log in, and the 30-day backfill starts automatically.

---

## 🖥️ Part 1 — Server kit (Ubuntu)

Run every block as your normal sudo user. Change only values marked `CHANGE_ME`. Use passwords with letters and numbers only (they go inside a URL later).

### 1.1 btop + ttyd (live terminal in the browser)

```bash
sudo apt update && sudo apt install -y btop curl wget
sudo wget -O /usr/local/bin/ttyd "https://github.com/tsl0922/ttyd/releases/download/1.7.7/ttyd.$(uname -m)"
sudo chmod +x /usr/local/bin/ttyd

sudo tee /etc/systemd/system/btop-web.service > /dev/null <<EOF
[Unit]
Description=btop live in browser (ttyd)
After=network-online.target
Wants=network-online.target

[Service]
User=$USER
Environment=TERM=xterm-256color
ExecStart=/usr/local/bin/ttyd -p 7681 -i lo -t titleFixed=btop-live -t fontSize=14 -t disableLeaveAlert=true btop
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload && sudo systemctl enable --now btop-web
```

### 1.2 Netdata (metrics + 30-day local database)

```bash
wget -O /tmp/netdata-kickstart.sh https://get.netdata.cloud/kickstart.sh
sh /tmp/netdata-kickstart.sh --stable-channel --disable-telemetry --non-interactive

sudo tee -a /etc/netdata/netdata.conf > /dev/null <<'EOF'

[web]
    bind to = 127.0.0.1

[db]
    mode = dbengine
    dbengine tier 0 retention time = 30d
EOF

sudo systemctl restart netdata
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:19999/api/v1/info   # → 200
```

### 1.3 Caddy (one login, two users)

`owner` is for you in the browser; `dashboard` is a separate read-only credential used only by the Vercel app, so you can rotate it without touching your own.

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https gnupg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update && sudo apt install -y caddy

OWNER_HASH=$(caddy hash-password --plaintext 'CHANGE_ME_owner_password')
APP_HASH=$(caddy hash-password --plaintext 'CHANGE_ME_app_password')

sudo tee /etc/caddy/Caddyfile > /dev/null <<EOF
:8080 {
    bind 127.0.0.1
    basicauth {
        owner ${OWNER_HASH}
        dashboard ${APP_HASH}
    }
    redir /history /history/
    handle_path /history/* {
        reverse_proxy 127.0.0.1:19999
    }
    handle {
        reverse_proxy 127.0.0.1:7681
    }
}
EOF

caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
sudo systemctl enable --now caddy && sudo systemctl restart caddy
```

### 1.4 Tunnel — option A: Tailscale Funnel (recommended)

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up                 # log in with the link it prints
sudo tailscale funnel --bg 8080   # first run prints a link to enable HTTPS + Funnel; open it, then re-run
tailscale funnel status           # shows https://<machine>.<tailnet>.ts.net
```

`--bg` makes the Funnel persist across reboots. Your server link is:

```text
https://dashboard:CHANGE_ME_app_password@<machine>.<tailnet>.ts.net/history
```

### 1.4 Tunnel — option B: ngrok (budget mode)

<details>
<summary>Show ngrok setup</summary>

```bash
curl -sSL https://ngrok-agent.s3.amazonaws.com/ngrok.asc | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null
echo "deb https://ngrok-agent.s3.amazonaws.com bookworm main" | sudo tee /etc/apt/sources.list.d/ngrok.list
sudo apt update && sudo apt install -y ngrok
ngrok config add-authtoken CHANGE_ME_YOUR_AUTHTOKEN

NGROK_DOMAIN='CHANGE_ME.ngrok-free.app'   # your free dev domain from the ngrok dashboard
sudo tee /etc/systemd/system/ngrok-pulse.service > /dev/null <<EOF
[Unit]
Description=ngrok tunnel for Server Pulse
After=network-online.target caddy.service
Wants=network-online.target

[Service]
User=$USER
ExecStart=$(command -v ngrok) http 8080 --url=https://${NGROK_DOMAIN} --log=stdout
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload && sudo systemctl enable --now ngrok-pulse
```

Server link: `https://dashboard:CHANGE_ME_app_password@CHANGE_ME.ngrok-free.app/history`. The dashboard detects ngrok from the host and switches to budget mode automatically.

</details>

### 1.5 Verify

```bash
systemctl is-active btop-web netdata caddy            # all "active"
curl -s -o /dev/null -w '%{http_code}\n' -u dashboard:CHANGE_ME_app_password \
  https://<your-public-host>/history/api/v1/info        # → 200
sudo reboot                                             # then re-run the checks
```

Open `https://<your-public-host>/` in a browser and log in as `owner` to see btop live.

---

## ☁️ Part 2 — Dashboard (Vercel)

### 2.1 Push to GitHub

```bash
git init && git add -A && git commit -m "Initial commit: Server Pulse"
git branch -M main
git remote add origin https://github.com/YOUR_GITHUB_USERNAME/server-pulse.git
git push -u origin main
```

### 2.2 Deploy

1. Go to [vercel.com/new](https://vercel.com/new) → pick `server-pulse` → **Import**.
2. Under **Environment Variables** add `SERVER_URL` (the link from Part 1). Mark it **Sensitive**.
3. Click **Deploy**.
4. Open the project → **Storage** → **Create Database** → **Neon** → connect it to this project (adds `DATABASE_URL`).
5. **Deployments** → ⋯ on the latest → **Redeploy** (creates the tables).
6. Open your `*.vercel.app` URL and log in with the password inside `SERVER_URL`. The 30-day backfill starts on its own.

Every later `git push` to `main` redeploys automatically.

---

## 💻 Local development

```bash
npm install
npm run dev              # http://localhost:3000 — demo mode, password "demo"
```

Use real data locally:

```bash
npm i -g vercel
vercel link
vercel env pull .env.local
npm run dev
```

| Script | What it does |
| --- | --- |
| `npm run dev` | Start the dev server |
| `npm run build` | Run migrations (if a DB is set), then `next build` |
| `npm run lint` | ESLint |
| `npm run db:generate` | Generate a migration from `lib/db/schema.ts` |

---

## ⚙️ Configuration reference

| Variable | Required | Default | Description |
| --- | --- | --- | --- |
| `SERVER_URL` | ✅ | — | `https://USER:PASS@host` (root or `/history`). Alias: `NGROK_URL`. Can be overridden in Settings. |
| `DATABASE_URL` | Auto | — | Added by the Neon integration (also reads `POSTGRES_URL`). |
| `DASHBOARD_PASSWORD` | — | Password inside `SERVER_URL` | Owner login. |
| `VIEWER_PASSWORD` | — | Off | Read-only login: Live, History, Reports. |
| `SESSION_SECRET` | — | Derived from the link | Signs session cookies. Set it to keep sessions across link changes. |
| `CRON_SECRET` | — | Off | If set, `/api/cron/sync` requires `Authorization: Bearer …`. If unset, the job is throttled to once per 10 min. |
| `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`, `ALERT_EMAIL_TO` | — | Off | Enables the "server offline" email. |

`vercel.json`:

```json
{
  "crons": [{ "path": "/api/cron/sync", "schedule": "15 0 * * *" }]
}
```

---

## 🕰️ How history works

| Stage | Where | Resolution | Kept for |
| --- | --- | --- | --- |
| Collected | Netdata on the server | 1 s | 30 days |
| Backfill (first connect) | Neon | 5 min | until rolled up |
| Nightly sync (last 26 h) | Neon | 1 min | 30 days |
| Rollup | Neon | 1 hour | 1 year + |

- The 26-hour window overlaps the previous run by 2 hours, covering Hobby cron's ±59-minute timing and occasional missed runs.
- If the dashboard is down for days, the next sync and a manual **Backfill** fill the gap from Netdata (up to 30 days back).
- History queries bucket with `date_bin` + `avg` to about 500 points per chart; ranges inside the last 24 h read Netdata directly for full detail.

**Tracked charts**

| Netdata chart | Shown as | Unit |
| --- | --- | --- |
| `system.cpu` | CPU % (sum of dimensions) | % |
| `system.ram` | used ÷ total | %, GiB |
| `system.load` | load1 / load5 / load15 | — |
| `system.io` | read / write (abs) | KiB/s |
| `system.net` | in / out (abs) | kbit/s, GB totals |
| `disk_space./` (fallback `disk_space._`) | used ÷ (used + avail) | %, GiB |
| context `sensors.temperature` | hottest sensor | °C |

---

## 🗃️ Data model

```sql
CREATE TABLE samples (
  id            BIGSERIAL PRIMARY KEY,
  chart         TEXT        NOT NULL,
  ts            TIMESTAMPTZ NOT NULL,
  values        JSONB       NOT NULL,        -- { "user": 3.1, "system": 1.2, ... }
  resolution_s  INT         NOT NULL,        -- 60, 300 or 3600
  UNIQUE (chart, ts, resolution_s)
);
CREATE INDEX samples_chart_ts ON samples (chart, ts DESC);

CREATE TABLE sync_runs (
  id             BIGSERIAL PRIMARY KEY,
  started_at     TIMESTAMPTZ NOT NULL,
  finished_at    TIMESTAMPTZ,
  rows_inserted  INT,
  status         TEXT,                       -- ok | partial | error | offline
  error          TEXT
);

CREATE TABLE settings (
  key    TEXT PRIMARY KEY,                   -- e.g. server_url
  value  TEXT NOT NULL
);
```

---

## 🔌 API reference

**App routes** (all require a session unless noted)

| Route | Method | Access | Purpose |
| --- | --- | --- | --- |
| `/api/login`, `/api/logout` | POST | Public / session | Start or end a session |
| `/api/live` | GET | Owner, viewer | Latest values; one shared server request per interval |
| `/api/history?chart=&from=&to=` | GET | Owner, viewer | Bucketed series |
| `/api/reports` | GET | Owner, viewer | 12 months of aggregates |
| `/api/backfill?chart=` | POST | Owner | 30-day import for one chart |
| `/api/settings/link`, `/api/settings/test` | POST | Owner | Save / test the server link |
| `/api/cron/sync` | GET | `CRON_SECRET` or throttled | Nightly sync, rollup, offline check |

**Netdata endpoints used** (server, read-only)

| Endpoint | Used for |
| --- | --- |
| `GET /api/v1/info` | Hostname, OS, cores, RAM, uptime, connection test |
| `GET /api/v1/allmetrics?format=json` | All live values in one call |
| `GET /api/v1/data?chart=&after=&before=&points=&group=average&format=json&options=seconds` | Seeding charts, sync, backfill |
| `GET /api/v1/charts` | Discovering temperature and disk charts |

Every server call sends basic auth from the link plus `ngrok-skip-browser-warning: 1`, and times out after 8 s.

---

## 🔔 Alerts

Netdata's built-in health alarms watch CPU, RAM, disk and temperature and notify you in real time — even when Vercel is down.

1. Create a Telegram bot with [@BotFather](https://t.me/BotFather) and copy its token.
2. Send your bot a message, then get your chat ID from `https://api.telegram.org/bot<TOKEN>/getUpdates`.
3. Edit the notification config:

```bash
cd /etc/netdata && sudo ./edit-config health_alarm_notify.conf
```

Set:

```bash
SEND_TELEGRAM="YES"
TELEGRAM_BOT_TOKEN="CHANGE_ME_bot_token"
DEFAULT_RECIPIENT_TELEGRAM="CHANGE_ME_chat_id"
```

4. Send a test alert:

```bash
sudo su -s /bin/bash netdata -c '/usr/libexec/netdata/plugins.d/alarm-notify.sh test'
```

The dashboard also emails you once if the server has been unreachable for a day (set the `SMTP_*` variables in Vercel).

---

## 📬 30-day email report

Every 30 days the server emails a summary of the last 30 days plus a zip of CSVs (one row per 5 minutes).

<details>
<summary>1. SMTP settings (Gmail shown — use an <a href="https://myaccount.google.com/apppasswords">App Password</a>)</summary>

```bash
sudo tee /etc/btop-report.env > /dev/null <<'EOF'
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=CHANGE_ME@gmail.com
SMTP_PASS=CHANGE_ME_app_password
MAIL_FROM=CHANGE_ME@gmail.com
MAIL_TO=CHANGE_ME@gmail.com
EOF
sudo chmod 600 /etc/btop-report.env
```

</details>

<details>
<summary>2. Report script — <code>/usr/local/bin/btop-report.py</code></summary>

```bash
sudo tee /usr/local/bin/btop-report.py > /dev/null <<'EOF'
#!/usr/bin/env python3
"""Every 30 days: export the last 30 days from Netdata as CSV and email it via SMTP.
   --now  send immediately (test) without changing the 30-day countdown"""
import csv, io, os, sys, time, socket, ssl, smtplib, zipfile
import urllib.request, urllib.parse, urllib.error
from datetime import datetime
from email.message import EmailMessage

NETDATA = "http://127.0.0.1:19999"
DAYS = 30
POINTS = DAYS * 24 * 12              # one row per 5 minutes
ENV_FILE = "/etc/btop-report.env"
STAMP = "/var/lib/btop-report/last_sent"
CHARTS = {
    "cpu": ["system.cpu"],
    "ram": ["system.ram"],
    "load": ["system.load"],
    "disk_io": ["system.io"],
    "network": ["system.net"],
    "disk_space_root": ["disk_space./", "disk_space._"],
}

def load_env():
    cfg = {}
    with open(ENV_FILE) as f:
        for line in f:
            line = line.strip()
            if line and not line.startswith("#") and "=" in line:
                k, v = line.split("=", 1)
                cfg[k.strip()] = v.strip().strip('"').strip("'")
    return cfg

def fetch(chart):
    q = urllib.parse.urlencode({"chart": chart, "after": -DAYS * 86400, "points": POINTS,
                                "group": "average", "format": "csv"})
    with urllib.request.urlopen(f"{NETDATA}/api/v1/data?{q}", timeout=120) as r:
        return r.read().decode()

def num(x):
    try:
        return float(x)
    except (TypeError, ValueError):
        return 0.0

def parse(text):
    rows = list(csv.reader(io.StringIO(text)))
    if len(rows) < 2:
        return [], []
    head = [h.strip() for h in rows[0][1:]]
    data = [[num(x) for x in r[1:]] for r in rows[1:] if r]
    return head, data

def col(head, name):
    return head.index(name) if name in head else None

def summary(results):
    out = []
    def line(label, vals, unit="%", fmt="{:.1f}"):
        if vals:
            out.append(f"{label:<14} avg {fmt.format(sum(vals)/len(vals))} {unit:<3} peak {fmt.format(max(vals))} {unit}")
    if "cpu" in results:
        h, d = results["cpu"]; line("CPU usage", [sum(r) for r in d])
    if "ram" in results:
        h, d = results["ram"]; u = col(h, "used")
        if u is not None:
            line("RAM usage", [100 * r[u] / sum(r) for r in d if sum(r) > 0])
    if "load" in results:
        h, d = results["load"]; i = col(h, "load1")
        if i is not None:
            line("Load (1 min)", [r[i] for r in d], unit="", fmt="{:.2f}")
    if "network" in results:
        h, d = results["network"]
        if d:
            step = DAYS * 86400 / len(d)
            gb = lambda i: sum(abs(r[i]) for r in d) * step / 8 / 1024 / 1024 if i is not None else 0
            out.append(f"{'Network':<14} in {gb(col(h, 'received')):.1f} GB   out {gb(col(h, 'sent')):.1f} GB")
    if "disk_space_root" in results:
        h, d = results["disk_space_root"]; u, a = col(h, "used"), col(h, "avail")
        if u is not None and a is not None:
            vals = [100 * r[u] / (r[u] + r[a]) for r in d if r[u] + r[a] > 0]
            if vals:
                out.append(f"{'Disk / used':<14} peak {max(vals):.1f} %")
    return "\n".join(out)

def send(cfg):
    results, zbuf, skipped = {}, io.BytesIO(), []
    with zipfile.ZipFile(zbuf, "w", zipfile.ZIP_DEFLATED) as z:
        for name, ids in CHARTS.items():
            for cid in ids:
                try:
                    text = fetch(cid)
                except urllib.error.URLError:
                    continue
                z.writestr(f"{name}.csv", text)
                results[name] = parse(text)
                break
            else:
                skipped.append(name)
    if not results:
        sys.exit("Netdata returned no data - is it running on 127.0.0.1:19999?")

    host = socket.gethostname()
    end = datetime.now()
    start = datetime.fromtimestamp(end.timestamp() - DAYS * 86400)
    period = f"{start:%Y-%m-%d} -> {end:%Y-%m-%d} ({DAYS} days)"
    body = f"Server: {host}   Period: {period}\n\n{summary(results)}\n\n"
    body += "Attached: CSV files, one row per 5 minutes (averages).\n"
    if skipped:
        body += f"Not available on this server: {', '.join(skipped)}\n"

    msg = EmailMessage()
    msg["Subject"] = f"[{host}] 30-day utilization report {start:%d %b} - {end:%d %b %Y}"
    msg["From"] = cfg["MAIL_FROM"]
    msg["To"] = cfg["MAIL_TO"]
    msg.set_content(body)
    msg.add_attachment(zbuf.getvalue(), maintype="application", subtype="zip",
                       filename=f"{host}-usage-{end:%Y-%m-%d}.zip")

    port = int(cfg.get("SMTP_PORT", "587"))
    ctx = ssl.create_default_context()
    if port == 465:
        s = smtplib.SMTP_SSL(cfg["SMTP_HOST"], port, context=ctx, timeout=60)
    else:
        s = smtplib.SMTP(cfg["SMTP_HOST"], port, timeout=60)
        s.starttls(context=ctx)
    s.login(cfg["SMTP_USER"], cfg["SMTP_PASS"])
    s.send_message(msg)
    s.quit()
    print(f"Report for {period} sent to {cfg['MAIL_TO']}")

def main():
    cfg = load_env()
    if "--now" in sys.argv:
        send(cfg)
        return
    os.makedirs(os.path.dirname(STAMP), exist_ok=True)
    now = time.time()
    if not os.path.exists(STAMP):
        open(STAMP, "w").write(str(now))
        print(f"30-day countdown started; first report on {datetime.fromtimestamp(now + DAYS*86400):%Y-%m-%d}")
        return
    last = num(open(STAMP).read().strip())
    if now - last < DAYS * 86400 - 3600:
        print(f"Next report in {(last + DAYS*86400 - now) / 86400:.1f} days")
        return
    send(cfg)
    open(STAMP, "w").write(str(now))

if __name__ == "__main__":
    main()
EOF
sudo chmod 755 /usr/local/bin/btop-report.py
```

</details>

<details>
<summary>3. Daily timer (sends only when 30 days have passed)</summary>

```bash
sudo tee /etc/systemd/system/btop-report.service > /dev/null <<'EOF'
[Unit]
Description=Email 30-day utilization report
After=network-online.target netdata.service
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/bin/python3 /usr/local/bin/btop-report.py
EOF

sudo tee /etc/systemd/system/btop-report.timer > /dev/null <<'EOF'
[Unit]
Description=Daily check for the 30-day utilization report

[Timer]
OnCalendar=*-*-* 09:00:00
Persistent=true

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now btop-report.timer
sudo /usr/local/bin/btop-report.py --now     # send a test email now
```

</details>

---

## 🔒 Security

- **Read-only everywhere.** ttyd runs without `-W`; Netdata's API is read-only; the dashboard has no write path to the server.
- **Nothing listens publicly.** ttyd, Netdata and Caddy bind to `127.0.0.1`; only the outbound tunnel reaches them.
- **Two credentials.** `owner` for you, `dashboard` for the app — rotate either independently by re-running step 1.3.
- **Secrets stay server-side.** `SERVER_URL` lives in Vercel as a Sensitive env var; it never appears in client bundles, logs or API responses, and is masked in Settings.
- **Sessions** are httpOnly, Secure, SameSite=Lax cookies; viewers get 403 on Settings, Sync and Backfill.
- **Cron** is protected by `CRON_SECRET`, or throttled to once per 10 minutes when unset (the job is idempotent).

---

## 🩺 Troubleshooting

| Symptom | Fix |
| --- | --- |
| Dashboard says **Server offline** | `systemctl is-active caddy netdata`; `tailscale funnel status` (or `systemctl status ngrok-pulse`); run **Settings → Test connection** to see the exact error |
| Connection test returns **401** | Wrong user/password in `SERVER_URL`; re-run step 1.3 or fix the link in Settings |
| Live view **paused** | ngrok budget mode hit 80 % of the monthly quota; switch to Tailscale Funnel or wait for the reset date shown in Settings |
| History is **empty** | Database not connected: Vercel → Storage → Neon → Redeploy; then **Settings → Backfill 30 days** |
| Sync shows **errors** | Check `sync_runs` in Settings; the next nightly run re-fills the last 26 h |
| btop page **garbled** | Make sure the service has `Environment=TERM=xterm-256color`; zoom the browser out |
| No **Telegram** alerts | Re-check the bot token and chat ID; run the test command in [Alerts](#-alerts) |
| Report email **not sent** | `journalctl -u btop-report -n 30`; Gmail needs an App Password, not your normal password |

**Useful commands**

```bash
systemctl status btop-web netdata caddy --no-pager
journalctl -u caddy -u netdata -f
sudo du -sh /var/cache/netdata          # local history size
sudo /usr/local/bin/btop-report.py      # days until the next report
```

---

## 🗺️ Roadmap

- [x] Server kit: btop + ttyd, Netdata, Caddy, tunnel, systemd
- [x] 30-day email report
- [ ] Dashboard MVP: one-link setup, login, demo mode, Live page
- [ ] History + reports: Neon, nightly sync, backfill, rollup, Reports page
- [ ] Alerts (Telegram + email), viewer password, installable PWA
- [ ] v2: several servers, one link each, server switcher

---

## 📄 License

MIT © YOUR_NAME

---

<div align="center">

Built with [Netdata](https://www.netdata.cloud) · [btop](https://github.com/aristocratos/btop) · [ttyd](https://github.com/tsl0922/ttyd) · [Caddy](https://caddyserver.com) · [Tailscale](https://tailscale.com) · [Next.js](https://nextjs.org) · [Neon](https://neon.com) · [Vercel](https://vercel.com)

</div>
