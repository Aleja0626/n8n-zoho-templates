# n8n + Zoho Desk API Patterns

Common patterns and best practices for working with the Zoho Desk API in n8n workflows.

## HTTP Request Node Configuration

When calling Zoho Desk API from n8n HTTP Request nodes, use this configuration to avoid silent failures:

```
Response Format: text (not JSON)
Never Error: true (in options)
```

Then parse the response in a Code node:

```javascript
const response = JSON.parse($input.first().json.data);
// Now work with the parsed response safely
```

**Why?** Zoho Desk API sometimes returns non-JSON error messages or HTML error pages. Setting `responseFormat: "text"` + `neverError: true` ensures your workflow doesn't crash on unexpected responses.

## Common Endpoints

### Tickets

```
GET    /api/v1/tickets                    # List tickets
GET    /api/v1/tickets/{ticketId}         # Get single ticket
PATCH  /api/v1/tickets/{ticketId}         # Update ticket
POST   /api/v1/tickets                    # Create ticket
```

### Search Tickets

```
GET /api/v1/tickets/search?departmentId={deptId}&status=open&limit=100
```

Useful query parameters:
- `status`: open, closed, on hold
- `departmentId`: filter by department
- `assignee`: filter by agent email
- `limit`: max 100 per request
- `from`: pagination offset

### Ticket Comments/Threads

```
GET  /api/v1/tickets/{ticketId}/threads         # Get all threads
GET  /api/v1/tickets/{ticketId}/comments        # Get comments only
POST /api/v1/tickets/{ticketId}/comments        # Add comment
```

**Private comment** (visible only to agents):

```json
{
  "content": "Internal note here",
  "isPublic": false,
  "contentType": "html"
}
```

### Contacts

```
GET /api/v1/contacts/search?email={email}
GET /api/v1/contacts/{contactId}
```

## Pagination Pattern

Zoho Desk returns max 100 items per request. For full data extraction:

```javascript
// In a Code node after HTTP Request
const items = $input.first().json;
const hasMore = items.length === 100;

return [{
  json: {
    data: items,
    hasMore,
    nextFrom: hasMore ? items.length : 0
  }
}];
```

Use a Loop node or recursive Execute Workflow to paginate.

## Rate Limiting

Zoho Desk API limits:
- **Free**: 100 requests / minute
- **Standard+**: 200 requests / minute

### n8n Batching Configuration

In HTTP Request node options:

```
Batching:
  Batch Size: 5
  Batch Interval: 2000 (ms)
```

This sends 5 requests, waits 2 seconds, then sends the next batch.

## Update Ticket Fields

### Standard fields

```json
{
  "status": "Open",
  "priority": "High",
  "assigneeId": "136810000000012345",
  "departmentId": "136810000000007061"
}
```

### Custom fields

```json
{
  "cf": {
    "cf_custom_field_name": "value"
  }
}
```

## Error Handling Pattern

Wrap Zoho API calls with proper error handling:

```javascript
try {
  const response = JSON.parse($input.first().json.data);

  if (response.errorCode) {
    // Zoho returned an error
    return [{ json: { error: true, code: response.errorCode, message: response.message } }];
  }

  return [{ json: { error: false, data: response } }];
} catch (e) {
  // Response wasn't JSON (HTML error page, etc.)
  return [{ json: { error: true, code: 'PARSE_ERROR', raw: $input.first().json.data } }];
}
```

## Webhook Configuration

To receive real-time ticket updates:

1. In Zoho Desk > Setup > Developer Space > Webhooks
2. Set URL to your n8n webhook endpoint
3. Select events (ticket created, updated, etc.)
4. Zoho sends POST with ticket data to your n8n webhook

## Department IDs

Get all departments:
```
GET /api/v1/departments
```

Cache department IDs in n8n Static Data to avoid repeated API calls.
