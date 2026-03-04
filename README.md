# NextPhone Tools

Give your AI receptionist superpowers. Connect any API as a tool your NextPhone agent can use during live calls.

## How it works

NextPhone agents can call **custom tools** mid-conversation. When a caller asks something that requires an external action — looking up a contact, sending an SMS, creating a ticket — the agent triggers your tool automatically.

No code required. Define a JSON config, and your agent gets a new capability.

## Quick example

Send an SMS confirmation after every booked appointment:

```json
{
  "type": "http",
  "tool_name": "send_confirmation_sms",
  "description": "Send SMS confirmation to caller after booking",
  "url": "https://api.twilio.com/2010-04-01/Accounts/{sid}/Messages.json",
  "http_method": "POST",
  "headers": {
    "Authorization": "Basic {your_twilio_credentials}"
  },
  "body_template": {
    "To": "{{caller_number}}",
    "From": "+15551234567",
    "Body": "Your appointment with {{owner_name}} is confirmed for {{date}} at {{time}}."
  },
  "parameters": [
    { "name": "date", "type": "string", "description": "Appointment date", "required": true },
    { "name": "time", "type": "string", "description": "Appointment time", "required": true }
  ],
  "post_call": true,
  "when": "BOOKED"
}
```

The agent extracts `date` and `time` from the conversation. `{{caller_number}}` and `{{owner_name}}` are injected automatically from the call context.

## Tool config schema

```json
{
  "type": "http",
  "tool_name": "unique_name",
  "description": "Tell the agent when to use this tool",
  "url": "https://your-api.com/endpoint",
  "http_method": "POST",
  "headers": {},
  "body_template": {},
  "parameters": [],
  "return_response": true,
  "response_extract": "data.id",
  "post_call": false,
  "when": "BOOKED,TRANSFERRED"
}
```

| Field | Required | Description |
|---|---|---|
| `type` | Yes | Always `"http"` |
| `tool_name` | Yes | Unique name the agent calls (e.g. `search_contacts`) |
| `description` | Yes | Tells the agent when to use this tool |
| `url` | Yes | API endpoint. Supports `{{placeholders}}` |
| `http_method` | No | `POST`, `GET`, `PUT`, `DELETE`. Default: `POST` |
| `headers` | No | Request headers (auth tokens, content type) |
| `body_template` | No | Request body with `{{placeholders}}` resolved at runtime |
| `parameters` | Yes | What the agent extracts from the conversation |
| `return_response` | No | Return API response to the agent. Default: `true` |
| `response_extract` | No | Dot-path to extract from response (e.g. `results.0.name`) |
| `post_call` | No | Fire after call ends instead of during. Default: `false` |
| `when` | No | Post-call only: trigger condition (`BOOKED`, `TRANSFERRED`, `MISSED`, `SPAM`) |

## Available placeholders

These are automatically available in any `body_template` or `url`:

| Placeholder | Source | Description |
|---|---|---|
| `{{caller_number}}` | Call | Caller's phone number |
| `{{receiving_number}}` | Call | Your NextPhone number |
| `{{owner_name}}` | Account | Business owner name |
| `{{website}}` | Account | Business website URL |
| `{{booking_url}}` | Account | Calendar booking link |
| `{{message}}` | Agent | Message composed by agent |
| `{{summary}}` | Post-call | Full call summary |
| `{{caller_name}}` | Post-call | Caller's name (if captured) |

Plus any custom `parameters` you define — the agent extracts these from the conversation automatically.

## Examples

### Search CRM before answering

```json
{
  "type": "http",
  "tool_name": "search_contacts",
  "description": "Search for caller in CRM when they provide their name",
  "url": "https://api.hubapi.com/crm/v3/objects/contacts/search",
  "http_method": "POST",
  "headers": {
    "Authorization": "Bearer {your_hubspot_key}"
  },
  "body_template": {
    "filterGroups": [{
      "filters": [{
        "propertyName": "phone",
        "operator": "EQ",
        "value": "{{caller_number}}"
      }]
    }]
  },
  "parameters": [],
  "return_response": true,
  "response_extract": "results.0.properties"
}
```

The agent searches HubSpot by caller's phone number, gets their record, and can reference their name, company, and history in the conversation.

### Post to Slack on missed calls

```json
{
  "type": "http",
  "tool_name": "slack_missed_call",
  "description": "Notify team in Slack when a call is missed",
  "url": "https://hooks.slack.com/services/{your_webhook_url}",
  "http_method": "POST",
  "body_template": {
    "text": "Missed call from {{caller_number}} ({{caller_name}})\n\nSummary: {{summary}}"
  },
  "parameters": [],
  "post_call": true,
  "when": "MISSED"
}
```

### Create a support ticket

```json
{
  "type": "http",
  "tool_name": "create_ticket",
  "description": "Create a support ticket when caller reports an issue",
  "url": "https://your-helpdesk.com/api/tickets",
  "http_method": "POST",
  "headers": {
    "Authorization": "Bearer {your_api_key}"
  },
  "body_template": {
    "subject": "Phone inquiry from {{caller_name}}",
    "description": "{{issue_description}}",
    "phone": "{{caller_number}}",
    "priority": "{{priority}}"
  },
  "parameters": [
    { "name": "caller_name", "type": "string", "description": "Caller's name", "required": true },
    { "name": "issue_description", "type": "string", "description": "Description of the issue", "required": true },
    { "name": "priority", "type": "string", "description": "Priority: low, medium, high", "required": false }
  ],
  "return_response": true,
  "response_extract": "ticket.id",
  "success_message": "Ticket created"
}
```

## How the agent uses tools

1. Caller says something that matches a tool's `description`
2. Agent extracts the required `parameters` from the conversation
3. `{{placeholders}}` in `body_template` are resolved with call context + extracted params
4. HTTP request fires
5. Response (or `success_message`) is returned to the agent
6. Agent continues the conversation with the result

For `post_call` tools, steps 1-2 are skipped — the tool fires automatically after the call ends based on the `when` condition.

## Links

- [NextPhone](https://www.getnextphone.com) — AI answering service for small businesses
- [Live demo](https://www.getnextphone.com/demo) — Call and talk to an AI receptionist
