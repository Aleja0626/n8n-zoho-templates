# Setting up Zoho OAuth in n8n (EU Region)

Zoho's native OAuth node in n8n doesn't always support the EU region correctly. This guide shows how to set up a **generic OAuth2 credential** that works reliably with Zoho EU APIs.

## Prerequisites

- A Zoho account (EU: zoho.eu)
- Access to [Zoho API Console](https://api-console.zoho.eu/)
- n8n instance (self-hosted or cloud)

## Step 1: Create a Self Client in Zoho

1. Go to [api-console.zoho.eu](https://api-console.zoho.eu/)
2. Click **Add Client** > **Self Client**
3. Note down your **Client ID** and **Client Secret**

## Step 2: Generate a Code

1. In the Self Client, click **Generate Code**
2. Enter the scopes you need. Common scopes:

| Service | Scopes |
|---------|--------|
| Zoho Desk | `Desk.tickets.ALL, Desk.contacts.READ, Desk.settings.READ, Desk.basic.READ` |
| Zoho CRM | `ZohoCRM.modules.ALL, ZohoCRM.contacts.READ` |
| Zoho Cliq | `ZohoCliq.Webhooks.CREATE, ZohoCliq.Messages.ALL` |
| Zoho People | `ZohoPeople.forms.ALL, ZohoPeople.attendance.ALL` |

3. Set **Time Duration** to 10 minutes
4. Enter a description and click **Create**
5. Copy the generated **code** (valid for 10 minutes!)

## Step 3: Exchange Code for Refresh Token

Run this curl command (replace placeholders):

```bash
curl -X POST "https://accounts.zoho.eu/oauth/v2/token" \
  -d "grant_type=authorization_code" \
  -d "client_id=YOUR_CLIENT_ID" \
  -d "client_secret=YOUR_CLIENT_SECRET" \
  -d "code=YOUR_GENERATED_CODE"
```

Save the `refresh_token` from the response. This doesn't expire.

## Step 4: Create OAuth2 Credential in n8n

1. In n8n, go to **Credentials > New > OAuth2 API** (generic)
2. Fill in:

| Field | Value |
|-------|-------|
| Grant Type | Authorization Code |
| Authorization URL | `https://accounts.zoho.eu/oauth/v2/auth` |
| Access Token URL | `https://accounts.zoho.eu/oauth/v2/token` |
| Client ID | Your Client ID |
| Client Secret | Your Client Secret |
| Scope | Your scopes (space-separated) |
| Auth URI Query Parameters | `access_type=offline&prompt=consent` |
| Authentication | Body |

3. Instead of going through the OAuth flow, you can directly set the refresh token:
   - Save the credential
   - Edit it and add the refresh token in the credential data

## Step 5: Use in HTTP Request nodes

In any HTTP Request node:
- Authentication: **Generic Credential Type**
- Generic Auth Type: **OAuth2 API**
- Select your credential

The token will auto-refresh when it expires.

## Common API Base URLs (EU)

| Service | Base URL |
|---------|----------|
| Zoho Desk | `https://desk.zoho.eu/api/v1` |
| Zoho CRM | `https://www.zohoapis.eu/crm/v2` |
| Zoho Cliq | `https://cliq.zoho.eu/api/v2` |
| Zoho People | `https://people.zoho.eu/people/api` |

## Troubleshooting

- **INVALID_CODE**: The code expired (10 min limit). Generate a new one.
- **INVALID_SCOPE**: Check that scopes match exactly what the Self Client allows.
- **Rate limits**: Zoho allows ~100 requests/minute. Use n8n batching in HTTP Request nodes.
- **EU vs US**: Make sure ALL URLs use `.eu` domain, not `.com`.

## Token Refresh Pattern

For production workflows, consider using a **centralized Token Manager** workflow that refreshes tokens for all Zoho services and stores them in n8n Static Data. This avoids rate limits from multiple workflows refreshing simultaneously.

See the [Token Manager template](../templates/zoho-token-manager.json) for an implementation.
