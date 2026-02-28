# 💰 ExpenseTracker AI

**Email-driven personal expense tracking — zero manual entry, full financial visibility.**

Gmail label → n8n workflow → Gemini Flash parsing → PostgreSQL → Google Sheets dashboard

![Stack](https://img.shields.io/badge/n8n-Orchestration-FF6D5A?style=flat-square&logo=n8n&logoColor=white)
![Gemini](https://img.shields.io/badge/Gemini_Flash-LLM-4285F4?style=flat-square&logo=google&logoColor=white)
![Postgres](https://img.shields.io/badge/PostgreSQL-Database-4169E1?style=flat-square&logo=postgresql&logoColor=white)
![Sheets](https://img.shields.io/badge/Google_Sheets-Dashboard-0F9D58?style=flat-square&logo=google-sheets&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Deployed-2496ED?style=flat-square&logo=docker&logoColor=white)

---

## 🧠 What is this?

A self-hosted, automated expense tracker that reads ICICI Bank transaction emails from Gmail, extracts structured data using Gemini Flash, stores everything in PostgreSQL, and syncs to a Google Sheets dashboard with charts.

**No apps. No permissions. No manual entry.** Just your bank emails → clean spending data.

### Why not Walnut / CRED / Mint?

- They require excessive permissions (SMS, contacts, etc.)
- Bloated apps for a simple problem
- Don't work well with Indian bank email formats
- No data ownership — your financial data lives on their servers
- Not extensible — you can't build on top of them

---

## ⚡ Architecture

```
┌─────────────┐     ┌──────────────┐     ┌───────────────┐
│  Gmail API   │────▶│  n8n Workflow │────▶│ Gemini Flash  │
│ (label:txns) │     │  (Cron 1hr)  │     │ (JSON parse)  │
└─────────────┘     └──────┬───────┘     └───────┬───────┘
                           │                      │
                           ▼                      ▼
                    ┌──────────────┐     ┌───────────────┐
                    │  PostgreSQL  │◀────│   Validate &  │
                    │  (storage)   │     │    Enrich     │
                    └──────┬───────┘     └───────────────┘
                           │
                           ▼
                    ┌──────────────┐
                    │ Google Sheets│
                    │ (dashboard)  │
                    └──────────────┘
```

### Data Flow

1. **Trigger** — n8n cron fires every hour
2. **Fetch** — Gmail node pulls unread emails from `transactions` label
3. **Dedup** — Check `gmail_message_id` against PostgreSQL
4. **Extract** — Code node strips HTML → plain text
5. **Parse** — Gemini Flash extracts structured JSON (amount, merchant, category, etc.)
6. **Validate** — Schema validation + deterministic fallback rules
7. **Store** — PostgreSQL insert with `ON CONFLICT DO NOTHING`
8. **Sync** — Append row to Google Sheets
9. **Cleanup** — Mark email as read

---

## 🏗️ Tech Stack

| Component | Choice | Why |
|-----------|--------|-----|
| Orchestration | n8n (self-hosted) | Visual workflows, native connectors, RAG-ready |
| LLM | Gemini Flash | Cheap, fast, great at structured extraction |
| Database | PostgreSQL 16 | Shared with n8n, supports JSONB, RAG-ready |
| Dashboard | Google Sheets | Free, shareable, built-in charting |
| Email | Gmail API (OAuth2) | Native n8n connector, label filtering |
| Hosting | VPS (Docker) | Full data control, ~$5-7/month |

---

## 📂 Repo Structure

```
expense-tracker-ai/
├── docker-compose.yml          # n8n + PostgreSQL containers
├── .env.example                # Environment variables template
├── db/
│   └── init.sql                # PostgreSQL schema (transactions, failures, summaries)
├── n8n/
│   └── workflow-template.json  # Importable n8n workflow (MVP)
├── scripts/
│   ├── fallback-categories.js  # Deterministic category fallback rules
│   └── gemini-prompt.txt       # System prompt for Gemini Flash
├── docs/
│   ├── SETUP.md                # End-to-end VPS setup guide
│   ├── GOOGLE_OAUTH.md         # Google OAuth2 configuration
│   └── EMAIL_FORMATS.md        # ICICI Bank email parsing reference
└── README.md
```

---

## 🚀 Quick Start

### Prerequisites

- A VPS or local machine with Docker installed (2 vCPU, 4 GB RAM minimum)
- Gmail account with ICICI Bank transaction emails
- Gmail label `transactions` filtering bank emails
- Google Cloud project with Gmail API & Sheets API enabled

### 1. Clone the repo

```bash
git clone https://github.com/yourusername/expense-tracker-ai.git
cd expense-tracker-ai
```

### 2. Configure environment

```bash
cp .env.example .env
# Edit .env with your passwords
nano .env
```

### 3. Start containers

```bash
docker compose up -d
```

### 4. Initialize the database

```bash
docker compose exec postgres psql -U n8n -d n8n -f /docker-entrypoint-initdb.d/init.sql
```

Or manually:

```bash
docker compose exec postgres psql -U n8n -d n8n
# Paste contents of db/init.sql
```

### 5. Access n8n

Open `http://YOUR_SERVER_IP:5678` and log in with credentials from `.env`.

### 6. Set up Google OAuth2

Follow the guide in [`docs/GOOGLE_OAUTH.md`](docs/GOOGLE_OAUTH.md).

### 7. Build the workflow

Import `n8n/workflow-template.json` or build from scratch following the workflow design in the docs.

### 8. Test & enable

- Run workflow manually with a single email
- Verify parsed output in PostgreSQL and Google Sheets
- Enable the cron trigger for automatic hourly processing

---

## 📊 Categories

| Category | Examples |
|----------|----------|
| Food Delivery | Swiggy, Zomato, Dunzo |
| Groceries | BigBasket, Blinkit, DMart, Zepto |
| Dining Out | Restaurants, cafes, bars |
| Transport | Uber, Ola, Rapido, Metro |
| Shopping (Online) | Amazon, Flipkart, Myntra |
| Subscriptions | Netflix, Spotify, YouTube Premium |
| Utilities | Electricity, water, gas, broadband |
| Rent | Rent payments |
| EMI / Loan | EMI debits, loan payments |
| Health | Pharmacy, hospital, doctor |
| Travel | Hotels, flights, MakeMyTrip |
| Transfers | UPI to friends/family, NEFT |
| Income | Salary, freelance credits |
| *...and more* | See `scripts/fallback-categories.js` |

---

## 📈 Google Sheets Dashboard

The dashboard auto-populates with three sheets:

- **Raw Transactions** — Every parsed transaction as a row
- **Dashboard** — Summary metrics (total spent, income, net flow, daily average, category breakdown, top merchants)
- **Charts** — Pie (spending by category), bar (daily trend), line (cumulative monthly), horizontal bar (top merchants)

All formulas auto-recalculate when new rows are appended. Append-only — no rows are ever deleted or modified.

---

## 🔒 Error Handling

| Scenario | Handling |
|----------|----------|
| Gmail API rate limit | n8n retry with exponential backoff |
| Gemini returns invalid JSON | Try-parse → route to error branch |
| Gemini returns `parsed: false` | Log to `parse_failures` table |
| Duplicate email | PostgreSQL `UNIQUE` constraint catches it |
| Sheets API failure | `synced_to_sheets = false`, retry next run |
| Gemini API down | Retry 3x with backoff → log failure → continue |
| Token expiry | n8n auto-refreshes OAuth tokens |

---

## 💸 Cost

| Component | Cost |
|-----------|------|
| Gemini Flash | Free tier (~100 emails/month) |
| Gmail API | Free |
| Google Sheets API | Free |
| PostgreSQL | Self-hosted (free) |
| n8n | Self-hosted (free) |
| **VPS** | **~$5-7/month** |

**Total: ~$5-7/month** for a fully automated expense tracker.

---

## 🗺️ Roadmap

### ✅ MVP (Current)
- [x] Gmail → n8n → Gemini Flash → PostgreSQL → Google Sheets
- [x] ICICI Bank email parsing
- [x] 19-category taxonomy with fallback rules
- [x] Deduplication and error handling
- [x] Docker Compose deployment

### 🔜 Phase 2: RAG Foundation
- [ ] Generate text chunks from transactions
- [ ] Embed with Gemini Embedding API
- [ ] Store in Qdrant (self-hosted Docker)
- [ ] Nightly indexing workflow in n8n

### 🔮 Phase 3: Conversational Interface
- [ ] Telegram bot → n8n webhook
- [ ] Natural language queries over transaction history
- [ ] Query embedding → Qdrant search → Gemini response

### 🚀 Phase 4: Smart Features
- [ ] Budget alerts via Telegram
- [ ] Anomaly detection (amount outliers, new merchants)
- [ ] Multi-bank support (HDFC, SBI)
- [ ] Receipt OCR with Gemini Vision
- [ ] Web dashboard (Next.js) replacing Google Sheets

---

## 📄 License

MIT

---

## 🙏 Acknowledgments

Built with [n8n](https://n8n.io), [Gemini Flash](https://ai.google.dev/), and a healthy dislike for manual expense tracking.
