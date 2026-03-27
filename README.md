# n8n + Zoho Automation Platform

Production-grade n8n workflow templates for Zoho integrations. Built from a real enterprise automation ecosystem processing **thousands of operations weekly** across 16+ platforms.

## What's inside

**7 ready-to-import workflow templates**, **3 architecture guides**, and **2 real-world case studies** — all extracted and sanitized from a production environment running **50+ active workflows (1,200+ nodes)** across two n8n instances.

---

## Architecture Overview

```
                          ┌─────────────────────────────────┐
                          │       Zoho Ecosystem             │
                          │  Desk · CRM · Cliq · People      │
                          └──────────┬──────────────────────┘
                                     │ REST APIs + Webhooks
                    ┌────────────────┼────────────────┐
                    │                │                │
              ┌─────▼─────┐   ┌─────▼─────┐   ┌─────▼─────┐
              │  n8n       │   │  n8n       │   │  External  │
              │  Primary   │   │  Secondary │   │  Services  │
              │  Instance  │   │  Instance  │   │            │
              │            │   │  (Skynet)  │   │  OpenAI    │
              │  24 active │   │  40 active │   │  WhatsApp  │
              │  workflows │   │  workflows │   │  Google    │
              └─────┬──────┘   └─────┬──────┘   │  Qdrant   │
                    │                │           └───────────┘
                    └────────┬───────┘
                             │
                    ┌────────▼────────┐
                    │  Infrastructure  │
                    │  Docker/Portainer│
                    │  PostgreSQL      │
                    │  Redis           │
                    │  Real-time       │
                    │  Auditor         │
                    └─────────────────┘
```

### Key Numbers

| Metric | Value |
|--------|-------|
| Active workflows | 50+ |
| Total nodes across all workflows | 1,200+ |
| Platforms integrated | 16+ |
| n8n instances managed | 2 |
| Tickets auto-classified weekly | Hundreds (zero human intervention) |
| Uptime monitoring | Every 10 minutes, 24/7 |

---

## Templates

### AI Ticket Classifier

**67 nodes** · OpenAI + business rules · Zoho Desk

Fully automated ticket classification system that routes incoming support tickets by department, category, priority, and type — combining AI analysis with deterministic business rules for reliability.

**How it works:**
1. Webhook receives new ticket from Zoho Desk
2. Business rules check for keyword matches (portability, billing, etc.)
3. If no rule matches, OpenAI analyzes the ticket content
4. Ticket is updated with department, category, priority, and assigned team
5. Private comment added with classification reasoning
6. Cliq notification sent to the assigned team

**Patterns used:** Centralized token management, two-stage classification (rules-first, AI-fallback), Zoho Desk API with error-resilient HTTP calls, Static Data caching.

> [Download template](templates/zoho-desk-classifier.json)

---

### WhatsApp Business Auto-Assign (Round Robin)

**7 nodes** · WhatsApp Business API · Zoho Desk

Automatic round-robin assignment of incoming WhatsApp tickets to support agents. Detects bot conversation selections, classifies by keywords, and assigns to the correct team with an automatic comment.

**Key features:**
- Detects Guided Conversation bot selections from WhatsApp
- Keyword-based routing to specialized teams
- Round-robin distribution with agent availability checks
- Automatic private comment with assignment reasoning

> [Download template](templates/wa-auto-assign.json)

---

### ConexiaBot — Internal AI Support Bot

**25 nodes** · OpenAI + Qdrant + Google Drive · Zoho Cliq

AI-powered internal support bot for Zoho Cliq that answers agent questions using:
- **Conversational memory** — maintains context across messages per user
- **Vector search** (Qdrant) — searches internal knowledge base semantically
- **Google Drive search** — finds relevant documents and procedures
- **Smart routing** — escalates to human when confidence is low

**Architecture:**
```
Cliq message → Webhook → Context loader → AI Router
                                            ├─ Qdrant vector search
                                            ├─ Google Drive lookup
                                            └─ Direct AI response
                                          → Format → Cliq reply
```

> [Download template](templates/zoho-cliq-bot.json)

---

### Centralized OAuth Token Manager

**5 nodes** · Zoho OAuth 2.0 · n8n Static Data

Production pattern for managing OAuth tokens across multiple Zoho services from a single workflow. Eliminates token refresh race conditions when multiple workflows need API access simultaneously.

**Problem solved:** When 20+ workflows all try to refresh the same OAuth token, you hit rate limits and get cascading failures. This workflow centralizes token refresh using n8n Static Data as a cache with TTL.

**How it works:**
```
Any workflow → Execute Workflow (Token Manager) → Returns valid token
                    │
                    ├─ Token cached & valid? → Return from cache
                    └─ Token expired? → Refresh via Zoho API → Cache → Return
```

Supports multiple services (Desk, CRM, People, Cliq) with independent token lifecycles.

> [Download template](templates/zoho-token-manager.json)

---

### Auto-Close NoReply Tickets

**6 nodes** · Zoho Desk · Static Data (two-pass)

Automatically closes tickets where the customer hasn't responded after a configurable number of days. Uses a **two-pass pattern** with n8n Static Data:

1. **First pass:** Finds candidate tickets, stores them in Static Data with timestamp
2. **Second pass (next run):** Checks if candidates are still unresponsive → closes them

This prevents premature closures and gives customers a grace period.

> [Download template](templates/zoho-desk-noreply.json)

---

### Agent Mention Detector

**8 nodes** · Zoho Desk · Zoho Cliq

Detects when agent names appear in ticket comments and sends instant DM notifications via Cliq. Handles name variations (first name, full name, username) and only triggers on open tickets to avoid noise.

> [Download template](templates/zoho-desk-mentions.json)

---

### Scheduled Reminders (Cliq)

**3 nodes** · Zoho Cliq · Cron

Configurable scheduled reminders sent to Cliq channels or DMs. Used in production for shift reminders, check-in alerts, lunch break notifications, and end-of-day prompts.

> [Download template](templates/zoho-cliq-reminders.json)

---

## Guides

### Architecture Patterns

- [Infrastructure & Monitoring](guides/infrastructure-patterns.md) — Multi-instance n8n, Docker deployment, real-time auditing, automated backups
- [Zoho OAuth Setup (EU Region)](guides/zoho-oauth-setup.md) — Step-by-step OAuth2 configuration with Self Client
- [Zoho Desk API Patterns](guides/zoho-desk-api-patterns.md) — Error handling, pagination, rate limiting, webhook config

---

## Case Studies

### Case Study 1: AI Ticket Classifier (67 nodes)

**Challenge:** Support team manually classified 200+ tickets/week across 8 departments, 15 categories, and 4 priority levels. Average classification time: 3-5 minutes per ticket.

**Solution:** Built a hybrid classification system combining deterministic business rules with OpenAI fallback:
- **Phase 1 — Rules engine:** Pattern matching for known categories (portability requests, billing, hardware RMA). Catches ~40% of tickets instantly with 100% accuracy.
- **Phase 2 — AI classification:** OpenAI analyzes ticket subject + body, returns structured JSON with department, category, priority. Includes confidence score.
- **Phase 3 — Validation:** Business rules override AI when specific conditions are met (e.g., blocked categories like warranties during specific periods).
- **Phase 4 — Action:** Updates Zoho Desk ticket fields, adds private comment with reasoning, notifies team via Cliq.

**Result:** Zero manual classification needed. System processes tickets in <5 seconds. Business rules ensure compliance even when AI would suggest otherwise.

**Technical highlights:**
- `responseFormat: "text"` + `neverError: true` pattern for resilient Zoho API calls
- Static Data caching for department mappings (avoids API calls per ticket)
- Execute Workflow pattern for centralized token management
- Error handler workflow for automatic retry + Cliq alerts on failures

---

### Case Study 2: GO2CONEXIA — Chrome Extension + n8n Backend

**Challenge:** Company needed employees to clock in/out via a web-based system (Zoho People), but had no way to enforce it or track compliance in real time.

**Solution:** Built a full-stack attendance control system:

**Chrome Extension (v3.0, Manifest V3):**
- Blocks access to work tools until employee clocks in
- Floating widget showing current status, shift info, break timers
- One-click clock in/out with Zoho People API integration
- Auto-deployed to all employees via group policy

**n8n Backend (12+ workflows):**
- Webhook-based status checks (real-time)
- Automatic clock-out at shift end (cron-based, per-shift schedules)
- Lunch break reminders (14:00) and end-of-day reminders (16:55)
- Daily compliance reports sent to management via Cliq
- Integration with Zoho People attendance API + Google Calendar for holidays

**Infrastructure:**
- Zoho People API for attendance records
- n8n webhooks for real-time extension ↔ backend communication
- Holiday calendar sync (Google Calendar → n8n)
- Shift-aware logic (morning/afternoon/split schedules)

---

## Production Patterns & Lessons Learned

### Token Management at Scale
When you have 50+ workflows hitting Zoho APIs, naive token refresh causes cascading failures. Solution: one Token Manager workflow called via Execute Workflow, with Static Data cache and 50-min TTL (Zoho tokens last 60 min).

### Zoho Desk HTTP Requests That Don't Break
Zoho Desk API sometimes returns HTML error pages instead of JSON. Always use:
```javascript
// HTTP Request node settings
responseFormat: "text"
neverError: true

// Then in Code node
const response = JSON.parse($input.first().json.data);
```

### Two-Instance Architecture
Running critical workflows (classifier, monitoring) on a separate n8n instance ("Skynet") with its own database. Benefits:
- Heavy workflows don't affect response times of webhook-based flows
- Independent deployment and updates
- Cross-instance monitoring (each instance watches the other)

### Real-Time Auditing
A dedicated workflow runs every 10 minutes, checking:
- All expected workflows are active
- No unexpected deactivations
- Execution error rates within thresholds
- Sends Cliq alert within 10 min of any anomaly

### Automated Backups
- Weekly full export of all workflows to Google Drive
- Daily GitHub sync of critical workflow changes
- Both backup workflows are themselves monitored by the auditor

---

## Tech Stack

| Layer | Technologies |
|-------|-------------|
| Automation | n8n (self-hosted, 2 instances) |
| AI/ML | OpenAI API, Qdrant (vector DB) |
| Zoho Suite | Desk, CRM, Cliq, People |
| Messaging | WhatsApp Business API |
| Infrastructure | Docker, Portainer, PostgreSQL, Redis |
| Monitoring | Custom n8n auditor + Cliq alerts |
| Browser | Chrome Extension (Manifest V3) |
| Scripting | JavaScript, PowerShell, Zoho Deluge |
| APIs | REST, OAuth 2.0, Webhooks |
| Version Control | Git, GitHub |

---

## How to Use These Templates

1. Download the `.json` template file
2. In n8n, go to **Workflows > Import from File**
3. Configure your Zoho OAuth credentials ([see guide](guides/zoho-oauth-setup.md))
4. Adjust parameters to match your Zoho organization
5. Activate the workflow

## Requirements

- n8n (self-hosted or cloud)
- Zoho account with API access
- Zoho OAuth Self Client credentials

---

## About the Author

IT Automation Specialist focused on building production systems that connect platforms. This repository represents patterns extracted from a real enterprise environment where these automations handle critical business operations daily.

**Core expertise:** n8n workflow architecture, Zoho ecosystem integration, AI-powered automation, WhatsApp Business API, Chrome extension development, Docker infrastructure management.

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-blue)](https://www.linkedin.com/in/alejandra-mendoza-5451a6219/)
[![GitHub](https://img.shields.io/badge/GitHub-Aleja0626-black)](https://github.com/Aleja0626)

## License

MIT
