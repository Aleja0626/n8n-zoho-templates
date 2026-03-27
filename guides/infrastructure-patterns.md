# Infrastructure & Monitoring Patterns

Patterns for running n8n at scale in production with high reliability.

## Multi-Instance Architecture

Running all workflows on a single n8n instance creates a single point of failure. Heavy workflows (like a 67-node AI classifier) can slow down lightweight webhook-based flows.

### Solution: Two-Instance Setup

```
┌──────────────────────────┐     ┌──────────────────────────┐
│   Primary Instance        │     │   Secondary Instance      │
│                           │     │   ("Skynet")              │
│   - Webhook-heavy flows   │     │   - AI Classifier (67n)   │
│   - Cal.com integrations  │     │   - All Zoho People flows │
│   - Quick-response APIs   │     │   - Monitoring & auditing │
│                           │     │   - Backup workflows      │
│   ~24 active workflows    │     │   - Sarenet monitoring    │
│                           │     │   ~40 active workflows    │
└───────────┬──────────────┘     └───────────┬──────────────┘
            │                                 │
            └──────────┬──────────────────────┘
                       │
              ┌────────▼────────┐
              │  Docker Host     │
              │  Portainer       │
              │  PostgreSQL      │
              │  Redis           │
              └─────────────────┘
```

### Benefits
- Heavy AI workflows don't block webhook responses
- Independent restarts and updates
- Cross-instance monitoring (each watches the other)
- Separate databases = isolated failure domains

### Docker Stack Configuration

Each n8n instance runs as a Docker stack with:
- `TZ=Europe/Madrid` (critical for cron-based workflows)
- Dedicated PostgreSQL database
- Persistent volume for file storage
- Portainer for management UI

**Important:** Always set timezone explicitly. Default UTC causes cron triggers to fire at wrong times for business-hour workflows.

---

## Real-Time Auditor

A dedicated monitoring workflow that runs every 10 minutes and validates system health.

### What It Checks

```
Every 10 min → List all workflows via n8n API
                    │
                    ├─ Compare active count vs expected
                    ├─ Check for unexpected deactivations
                    ├─ Query recent executions for error spikes
                    └─ Validate critical workflows are running
                          │
                     Any anomaly?
                     ├─ YES → Send Cliq DM to admin
                     └─ NO  → Silent (no noise)
```

### Implementation Notes

- Uses n8n's own API (`/api/v1/workflows`, `/api/v1/executions`)
- Stores "known good" state in Static Data
- Only alerts on *changes* (avoids alert fatigue)
- Runs on the secondary instance (monitors primary) and vice versa

---

## Automated Backup Strategy

### Weekly Full Export

Every Monday at 8:00 AM:
1. Fetch all workflows via n8n API
2. Export each as JSON
3. Upload to Google Drive (organized by date)
4. Send confirmation to Cliq

### Daily GitHub Sync

Critical workflows are synced to a private GitHub repo on change:
1. Detect workflow modifications via execution history
2. Export modified workflow JSON
3. Commit and push to GitHub with descriptive message

### Backup of Backups

The auditor monitors both backup workflows. If the weekly backup fails or the GitHub sync stops working, admin gets notified within 10 minutes.

---

## Centralized Token Management

### The Problem

With 50+ workflows hitting Zoho APIs:
- Multiple workflows refresh tokens simultaneously
- Zoho rate-limits token refresh endpoint
- Cascading auth failures across all workflows
- Each workflow maintains its own token state (inconsistent)

### The Solution: Token Manager Workflow

```
┌─────────────┐     ┌─────────────────────────┐
│ Workflow A   │──→  │                           │
│ Workflow B   │──→  │  Token Manager Workflow   │
│ Workflow C   │──→  │                           │
│ ...          │──→  │  1. Check Static Data     │
│ Workflow N   │──→  │  2. Token valid? Return   │
└──────────────┘     │  3. Expired? Refresh once │
                     │  4. Cache new token       │
                     │  5. Return to caller      │
                     └─────────────────────────┘
```

### Key Design Decisions

- **50-min TTL** (Zoho tokens last 60 min) — refresh before expiry
- **Static Data as cache** — survives workflow restarts
- **Execute Workflow pattern** — called by any workflow that needs a token
- **Per-service tokens** — Desk, CRM, People, Cliq each have independent refresh cycles
- **Single point of refresh** — eliminates race conditions

### Execute Workflow Configuration (n8n v1.2+)

```javascript
// Caller workflow - Execute Workflow node
workflowId: { __rl: true, value: "TOKEN_MANAGER_ID", mode: "id" }
workflowInputs: {
  mappingMode: "defineBelow",
  value: { service: "desk" }  // or "crm", "people", "cliq"
}
```

---

## Zoho Desk API — Resilient HTTP Pattern

### The Problem

Zoho Desk API occasionally returns:
- HTML error pages instead of JSON
- Empty responses on timeout
- 500 errors during maintenance windows

Standard n8n HTTP Request node crashes the workflow on these responses.

### The Solution

```
HTTP Request Node:
  responseFormat: "text"     ← Don't parse as JSON
  neverError: true           ← Don't throw on 4xx/5xx
      │
      ▼
Code Node:
  try {
    const data = JSON.parse($input.first().json.data);
    if (data.errorCode) { /* handle Zoho error */ }
    return [{ json: data }];
  } catch (e) {
    /* response was HTML/empty — handle gracefully */
  }
```

This pattern is used in **every** workflow that calls Zoho APIs. It eliminates silent failures and makes debugging much easier.

---

## Cron Workflow Patterns

### Timezone-Aware Scheduling

Business workflows need to respect local time. Examples:
- Ticket classifier: Mon-Thu 8:00-16:00, Fri 8:00-13:00
- Clock-in reminders: 7:50 AM (before shift starts)
- Auto-checkout: Based on each employee's shift end time

**Critical:** Set `TZ=Europe/Madrid` (or your timezone) in the Docker environment. Don't rely on n8n's default — it may not match your server's timezone.

### Holiday-Aware Scheduling

Workflows that shouldn't run on holidays:
1. Maintain a holiday calendar (CSV or Google Calendar)
2. First node after cron trigger: check if today is a holiday
3. If holiday → stop execution (no-op)
4. If workday → continue normally

We sync Barcelona holidays from Google Calendar to n8n weekly, stored in Static Data for fast lookup.
