# NextPhone Tools

Give your AI receptionist superpowers. Connect any API as a tool your NextPhone agent can use during live calls.

## How it works

NextPhone agents can call **custom tools** mid-conversation. When a caller asks to book an appointment, look up their account, or report an issue — the agent triggers your tool, gets the result, and continues the conversation naturally.

No code required. Define a JSON config, and your agent gets a new capability.

```
Caller: "I'd like to book a furnace inspection for next week"
         ↓
Agent checks calendar availability → finds 3 open slots
         ↓
Agent: "I have Tuesday at 10am, Wednesday at 2pm, or Thursday at 9am. Which works?"
         ↓
Caller: "Tuesday at 10"
         ↓
Agent books appointment → sends SMS confirmation
         ↓
Agent: "You're all set for Tuesday at 10am. You'll get a text confirmation shortly."
```

## Built-in tools

These work out of the box with NextPhone — no configuration needed.

### Calendar availability

The agent checks your calendar for open slots, respects business hours, and presents options to the caller.

- Connects to Google Calendar, Calendly, Cal.com, Acuity
- Configurable business hours and timezone
- Adjustable appointment duration and lookahead window
- Avoids double-booking automatically

### Appointment booking

Once the caller picks a time, the agent books it directly.

- Creates calendar event with caller details
- Sends calendar invite to both parties
- Captures caller name, email, phone, and reason for visit

### Call transfer

The agent routes calls to your team when needed.

- Instant transfer to any phone number
- AI decides when to transfer based on your rules (e.g., "transfer if caller mentions legal emergency")
- Fallback if transfer fails — agent takes a message instead

### SMS messaging

Send texts during or after calls.

- **Mid-call SMS** — Send booking links, directions, rate sheets while the caller is still on the line
- **Follow-up SMS** — Confirmation texts after appointments are booked
- **Missed call text-back** — Automatically text callers you couldn't reach

## Custom tools

Beyond the built-ins, you can connect **any REST API** as a tool. The agent calls it mid-conversation, gets the result, and keeps talking.

### Quick example

Notify your Slack channel when a call is missed:

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

### Tool config schema

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

### Available placeholders

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

## Example tools

### Search CRM before answering

Look up the caller in HubSpot so the agent can greet them by name and reference their history.

```json
{
  "type": "http",
  "tool_name": "search_contacts",
  "description": "Search for caller in CRM to personalize the conversation",
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

**Result:** Agent says "Hi Sarah, I see you're a current customer. How can I help?" instead of "Can I get your name?"

### Send SMS with booking link

During the call, text the caller a link to self-book if they want to pick their own time.

```json
{
  "type": "http",
  "tool_name": "send_booking_link",
  "description": "Send the caller a text with the booking link when they want to schedule online",
  "url": "https://api.twilio.com/2010-04-01/Accounts/{sid}/Messages.json",
  "http_method": "POST",
  "headers": {
    "Authorization": "Basic {your_twilio_credentials}"
  },
  "body_template": {
    "To": "{{caller_number}}",
    "From": "{{receiving_number}}",
    "Body": "Here's the link to book your appointment: {{booking_url}}"
  },
  "parameters": [],
  "return_response": false,
  "success_message": "Booking link sent"
}
```

**Result:** Agent says "I just texted you a link to book online. You should see it now."

### Log call to Google Sheets

Keep a simple call log for businesses that don't use a CRM.

```json
{
  "type": "http",
  "tool_name": "log_to_sheets",
  "description": "Log call details to Google Sheets after every call",
  "url": "https://script.google.com/macros/s/{your_script_id}/exec",
  "http_method": "POST",
  "body_template": {
    "caller": "{{caller_number}}",
    "name": "{{caller_name}}",
    "summary": "{{summary}}",
    "outcome": "{{outcome}}"
  },
  "parameters": [],
  "post_call": true
}
```

### Create support ticket

When a caller reports an issue, the agent creates a ticket in your helpdesk.

```json
{
  "type": "http",
  "tool_name": "create_ticket",
  "description": "Create a support ticket when caller reports a problem or issue",
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

### Webhook — trigger anything

Fire a generic webhook to Zapier, Make, or your own backend after specific call outcomes.

```json
{
  "type": "http",
  "tool_name": "webhook_new_lead",
  "description": "Send new lead data to webhook after qualifying calls",
  "url": "https://hooks.zapier.com/hooks/catch/{your_zap_id}",
  "http_method": "POST",
  "body_template": {
    "event": "new_lead",
    "caller_number": "{{caller_number}}",
    "caller_name": "{{caller_name}}",
    "summary": "{{summary}}",
    "business_number": "{{receiving_number}}"
  },
  "parameters": [],
  "post_call": true,
  "when": "BOOKED,TRANSFERRED"
}
```

## How the agent uses tools

1. Caller says something that matches a tool's `description`
2. Agent extracts the required `parameters` from the conversation
3. `{{placeholders}}` in `body_template` are resolved with call context + extracted params
4. HTTP request fires
5. Response (or `success_message`) is returned to the agent
6. Agent continues the conversation with the result

For `post_call` tools, the tool fires automatically after the call ends based on the `when` condition.

## Links

- [NextPhone](https://www.getnextphone.com) — AI answering service for small businesses
- [Blog](https://www.getnextphone.com/blog)
