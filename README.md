# WhatsApp AI Sales Assistant for Restaurants

An automated WhatsApp ordering and customer engagement system built for restaurants, designed from the ground up as a **reusable template** rather than a one-off bot for a single business. The architecture assumes multiple restaurant tenants will run on the same infrastructure, each with isolated data, menus, and conversation state.

This repository documents the infrastructure, workflow logic, and design decisions behind the system. It does not include production secrets, customer data, or restaurant-specific business logic beyond sanitized examples.

## Why this exists

Most "AI WhatsApp bot" projects fail in production for three reasons: they burn through LLM tokens on every message, they hallucinate menu items or prices, and they have no recovery path when something breaks mid-conversation. This project was built specifically to solve those three problems before adding any other feature.

### 1. Token cost control
Not every incoming message needs an LLM call. The system uses a hybrid UX pattern: prefixed/structured buttons for menu navigation and standard flows (zero LLM cost, instant response), and LLM calls reserved only for free-text input that requires interpretation (customizations, questions, ambiguous requests). This keeps per-conversation cost predictable and low, which matters once you're running this across many restaurants instead of one.

### 2. Zero hallucination on menu and pricing
The AI agent never generates menu items or prices from memory or general knowledge. Menu data (items, prices, weekly promotions, sauce/add-on options) is fetched live from Supabase on every relevant turn and injected into the agent's context as structured data. The agent's job is to interpret customer intent and map it to real catalog data, not to invent or recall it. This eliminates the most common and most damaging failure mode of restaurant bots: telling a customer a price or item that doesn't exist.

### 3. Error handling as a first-class workflow, not an afterthought
A dedicated error-handling workflow (`workflows/manejo_error_automatizacion_restaurante.example.json`) catches failures anywhere in the main flow and does two things simultaneously:
- Notifies staff in real time via Telegram with the failing node, error message, and affected customer chat ID.
- Sends the customer a graceful fallback WhatsApp message instead of leaving them with a dead conversation or a raw error.

The goal is not "handle errors gracefully" as a nice-to-have — it's driving customer-facing failures toward zero, since a broken checkout conversation directly costs the restaurant a sale.

## Multi-tenant design intent

This is built to eventually host multiple restaurants, not just one:
- Client/customer data is keyed by WhatsApp ID (`clientes` table, WhatsApp ID as primary key) rather than by a single restaurant's internal customer ID scheme.
- Menu, pricing, and promotions live in Supabase as data, not hardcoded in workflow logic — swapping a restaurant means swapping rows, not rebuilding workflows.
- API keys, server URLs, and instance identifiers are never hardcoded in workflow JSON; they're pulled from environment variables or from the incoming webhook payload itself, so the same workflow file can be imported for a different tenant without code changes.

Per-tenant conversation memory and Supabase Row Level Security (RLS) are the next steps toward true multi-tenant isolation and are not yet implemented — see Roadmap.

## Architecture

```
WhatsApp Customer Message
        │
        ▼
Evolution API (self-hosted WhatsApp gateway)
        │
        ▼
n8n Webhook ──> Extract Data (Code) ──> Lookup Customer (Supabase)
        │
        ▼
   IF customer exists?
      │            │
   New Customer   Existing Customer
   AI Agent        AI Agent
      │            │
      └─────┬──────┘
            ▼
   Fetch Menu + Promos (Supabase, live data)
            ▼
   AI Agent (interprets intent, maps to real catalog)
            ▼
   HTTP Request ──> Evolution API ──> WhatsApp reply

   [Error anywhere above] ──> Error Trigger ──> Telegram alert to staff
                                            └──> Fallback WhatsApp message to customer
```

## Tech stack

| Component | Purpose |
|---|---|
| **n8n** (self-hosted, Docker) | Workflow orchestration and AI Agent execution |
| **Evolution API** | Self-hosted WhatsApp connectivity layer (Baileys-based) |
| **Supabase (PostgreSQL)** | Customer data, menu/pricing catalog, order notes |
| **Redis** | Evolution API session/cache backing store |
| **nginx + certbot** | Reverse proxy and SSL termination |
| **Google Gemini / OpenAI** (via n8n LangChain nodes) | LLM backend for the conversational agent |
| **Telegram** | Real-time staff error alerts and order confirmations |

## Repository structure

```
.
├── docker-compose.yml          # Full stack definition (n8n, Evolution API, Postgres, Redis, nginx, certbot)
├── .env.example                # Environment variable template — copy to .env and fill in real values
├── nginx.conf.template          # nginx reverse proxy config template — copy to nginx.conf and set your domain
├── workflows/
│   ├── automatizacion_restaurante.example.json           # Main conversation + ordering flow
│   └── manejo_error_automatizacion_restaurante.example.json  # Error handling + alerting flow
├── schema/                     # Supabase SQL schema (tables, RLS policies)
└── docs/                       # Design notes and architecture decisions
```

## Setup from scratch

### Prerequisites
- A Linux server (VPS or equivalent) with Docker and Docker Compose installed
- A domain name pointed to your server's IP (required for SSL via certbot)
- A Supabase project (free tier is sufficient to start)
- A Telegram bot token (for staff error alerts) — create one via [@BotFather](https://t.me/BotFather)
- API keys for your chosen LLM provider (Google Gemini and/or OpenAI)

### 1. Clone the repository
```bash
git clone https://github.com/Elcarlospantoja/whatsapp-assistant.git
cd whatsapp-assistant
```

### 2. Configure environment variables
```bash
cp .env.example .env
nano .env
```
Fill in real values for:
- `AUTHENTICATION_API_KEY` — generate a strong random string for Evolution API auth
- `POSTGRES_PASSWORD` — a strong password for the Evolution API's internal Postgres instance
- `N8N_ENCRYPTION_KEY` — generate with `openssl rand -hex 32`; **back this up separately**, losing it means losing access to all saved n8n credentials
- `DOMAIN` — your actual domain (e.g. `yourdomain.com`)

### 3. Configure nginx
```bash
cp nginx.conf.template nginx.conf
nano nginx.conf
```
Replace all instances of `YOUR_DOMAIN` with your actual domain.

### 4. Obtain SSL certificates
Before first launch, you'll need valid certificates for nginx to serve HTTPS. Run certbot in standalone mode once, or use the certbot container's webroot flow — see [certbot's Docker documentation](https://certbot.eff.org/instructions) for your exact setup.

### 5. Launch the stack
```bash
docker compose up -d
docker ps
```
Confirm `n8n`, `evolution-api`, `evolution_postgres`, `evolution_redis`, and `nginx` are all `Up`.

### 6. Connect WhatsApp
Access the Evolution API Manager at `https://YOUR_DOMAIN` (or the configured path), create an instance, and scan the QR code with the WhatsApp account you want to use for the bot.

### 7. Set up Supabase
- Create a `clientes` table with WhatsApp ID as a `TEXT` primary key.
- Create a menu/products table with items, prices, and promotion flags.
- Add your Supabase project URL and service role key as an n8n credential (do not hardcode these in any workflow).

### 8. Import the workflows
In n8n, go to **Workflows → Import from File** and import both files from `workflows/`. You'll need to:
- Reconnect the Supabase, Telegram, and LLM credentials (workflow exports never include real credential values, only references).
- Set the environment variables referenced by the error-handling workflow: `EVOLUTION_API_KEY`, `EVOLUTION_SERVER_URL`, `EVOLUTION_INSTANCE_NAME`, `TELEGRAM_ALERT_CHAT_ID`.

### 9. Activate and test
Activate both workflows, send a WhatsApp message to your connected number, and confirm the flow responds and that a forced error (e.g. temporarily breaking a Supabase credential) triggers the Telegram alert correctly.

## Roadmap

- [ ] Supabase Row Level Security (RLS) policies for true multi-tenant data isolation
- [ ] Per-restaurant configuration table (menu source, branding, business hours) to fully decouple workflow logic from any single tenant
- [ ] Migration path from the informal WhatsApp gateway (Evolution API / Baileys) to the official WhatsApp Business API once revenue justifies the cost
- [ ] POS integration (explicitly out of scope until the core ordering flow is validated across multiple real restaurants)

## Security notes

- Never commit `.env`, `.env.bak`, `nginx.conf` (the real one), or the `certbot/` directory — all are excluded via `.gitignore`.
- Workflow JSON files in `workflows/` are sanitized examples: credential values, API keys, and any captured customer data (`pinData`) have been stripped. Re-importing them requires reconnecting credentials and setting environment variables as described above.
- Rotate any key that has ever been pasted into a chat, ticket, or shared document, even temporarily.
