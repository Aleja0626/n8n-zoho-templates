# n8n + Zoho Integration Templates

Ready-to-import n8n workflow templates for common Zoho integrations. Built from real production workflows handling thousands of tickets, messages, and automations.

## Templates

### Zoho Desk

| Template | Description | File |
|----------|-------------|------|
| Ticket Auto-Classifier | Classify tickets by category, priority and department using AI + business rules | [zoho-desk-classifier.json](templates/zoho-desk-classifier.json) |
| Auto-Close NoReply | Automatically close tickets with no customer response after X days | [zoho-desk-noreply.json](templates/zoho-desk-noreply.json) |
| Agent Mention Detector | Detect agent names in ticket comments and send DM notifications | [zoho-desk-mentions.json](templates/zoho-desk-mentions.json) |

### Zoho Cliq

| Template | Description | File |
|----------|-------------|------|
| Support Bot | Internal support bot that answers agent questions using AI + knowledge base | [zoho-cliq-bot.json](templates/zoho-cliq-bot.json) |
| Scheduled Reminders | Send recurring reminders to channels or DMs on a schedule | [zoho-cliq-reminders.json](templates/zoho-cliq-reminders.json) |

### WhatsApp Business + Zoho Desk

| Template | Description | File |
|----------|-------------|------|
| Round Robin Auto-Assign | Auto-assign incoming WhatsApp tickets to agents using round robin | [wa-auto-assign.json](templates/wa-auto-assign.json) |

### Zoho OAuth

| Template | Description | File |
|----------|-------------|------|
| Token Manager | Centralized OAuth token refresh for multiple Zoho services | [zoho-token-manager.json](templates/zoho-token-manager.json) |

## Guides

- [Setting up Zoho OAuth in n8n (EU region)](guides/zoho-oauth-setup.md) - Step by step OAuth2 configuration
- [n8n + Zoho Desk API patterns](guides/zoho-desk-api-patterns.md) - Common patterns and best practices

## How to use

1. Download the `.json` template file
2. In n8n, go to **Workflows > Import from File**
3. Configure your Zoho OAuth credentials (see [OAuth guide](guides/zoho-oauth-setup.md))
4. Adjust the parameters to match your Zoho organization
5. Activate the workflow

## Requirements

- n8n (self-hosted or cloud)
- Zoho account with API access
- Zoho OAuth Self Client credentials

## About

Templates extracted and sanitized from production workflows at [Conexia Telecom](https://conexiatec.com), processing 50+ automations daily across Zoho Desk, CRM, Cliq, People, and WhatsApp Business.

Built by [@Aleja0626](https://github.com/Aleja0626)

## License

MIT
