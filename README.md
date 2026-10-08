# AI Barbershop Booking Agent (n8n)

A conversational AI agent that books appointments in a barbershop. A customer writes in plain language – *"I need a haircut with Alex tomorrow after 5 pm"* – and the agent finds the right service and barber, checks **real free time slots**, and creates the event in the barber's **Google Calendar**.

The project is built as **one agent + three small workflows (tools)** that talk to each other. The tools are exposed through an **MCP server** and consumed by the agent through an **MCP client**, so the same tools could be reused by any other MCP-compatible agent.

## What it does

| Customer says | What the agent does |
|---|---|
| "What services do you have?" | Loads services and barbers from the database and answers |
| "When is Alex free for a haircut on Friday?" | Maps "haircut" → service ID, "Friday" → ISO dates, calls the free-slots tool, shows the options |
| "Book the 5 pm one" | Calculates the end time (start + service duration), creates a calendar event for the chosen barber |

The agent keeps the last 10 messages in Postgres memory, so the conversation can continue naturally. It always **checks availability before booking**.

## Project structure

Four separate n8n workflows. Each one does one job and can be tested on its own.

| File | Workflow | Role |
|---|---|---|
| [`barbershop-booking-agent.json`](workflows/barbershop-booking-agent.json) | **AI Booking Agent** | Chat trigger, AI Agent, memory, MCP server (exposes the tools) and MCP client (gives them to the agent) |
| [`barbershop-get-calendars-services.json`](workflows/barbershop-get-calendars-services.json) | **Get Calendars & Services** | Reads barbers (calendars) and services from NocoDB, returns them as JSON |
| [`barbershop-free-slots.json`](workflows/barbershop-free-slots.json) | **Free Slots** | Finds free time windows for a service across all barbers who offer it |
| [`barbershop-book-appointment.json`](workflows/barbershop-book-appointment.json) | **Book Appointment** | Creates the Google Calendar event |

```
                         ┌───────────────────────────── AI Booking Agent ─────────────────────────────┐
 Customer ─► Chat Trigger ─► Booking Agent ◄── OpenAI Chat Model                                       │
                              │         ◄── Chat Memory (Postgres, last 10 messages)                    │
                              │         ◄── MCP Client ──► MCP Server Trigger                           │
                              │                                  ├─ tool: Get Calendars and Services ─┼─► [Get Calendars & Services] ─► NocoDB
                              │                                  ├─ tool: Check Free Slots ───────────┼─► [Free Slots]
                              │                                  └─ tool: Book Appointment ───────────┼─► [Book Appointment] ─► Google Calendar
                         └────────────────────────────────────────────────────────────────────────────┘
```

### 1. AI Booking Agent
- **Model:** OpenAI `gpt-5-mini`
- **System prompt:** a booking assistant that converts relative dates ("tomorrow", "next Monday") to ISO dates using the current time, and always checks availability first
- **Memory:** Postgres Chat Memory, window of 10 messages
- **Tools:** reached through the MCP client; the MCP server in the same workflow exposes three workflow tools

### 2. Tools (sub-workflows)

**Get Calendars and Services** – "Call this at the start of every conversation." Returns `{ Calendars: [...], Service: [...] }` from two NocoDB tables so the agent knows which barbers, calendar IDs, services and durations exist.

**Check Free Slots** – inputs `start`, `end` (ISO 8601 with offset) and `service` (service ID). Step by step:
1. Loads the barbers from NocoDB and keeps only those who offer the requested service (`Service IDs` column).
2. Reads the service duration from the Services table.
3. Asks the Google Calendar **freeBusy** API for the busy periods of all those calendars in one request.
4. A Code node walks through each working day (08:00–20:00 by default), merges overlapping events and returns the free windows that are long enough for the service.

Output: `[{ calendar_id, slots: [{ "slot start", "slot end", duration }] }]` – one item per barber. To check one specific barber, the agent just reads that barber's entry.

**Book Appointment** – inputs `start`, `end`, `Calendar` (barber's calendar ID), `event_title`, `description` (format: `{Service name} at {Barber name}`). Creates the event with the Google Calendar node.

Both the Get Calendars & Services and Book Appointment workflows also have a **Webhook** trigger, so you can test them with a plain HTTP call without the agent.

## Tech stack

- [n8n](https://n8n.io) – AI Agent, Tool Workflow, MCP Server Trigger, MCP Client Tool, Postgres Chat Memory, Execute Workflow Trigger, Webhook
- OpenAI (`gpt-5-mini`)
- NocoDB – barbers and services database
- Google Calendar – one calendar per barber
- Postgres – chat memory

## Setup

### Requirements
- An n8n instance reachable from the internet (the MCP endpoint must be callable by the agent)
- NocoDB account with an API token
- Google account with one calendar per barber (OAuth2)
- A Postgres database (e.g. Supabase, Neon) for chat memory
- OpenAI API key

### 1. NocoDB tables

Create a base with two tables:

| Table | Columns used by the workflows |
|---|---|
| **Calendars** | barber name, `Calendar ID` (the barber's Google Calendar ID), `Service IDs` (list of IDs of the services this barber performs) |
| **Services** | service name, `Duration, min` (number), price. The NocoDB record ID is used as the service ID |

The workflows read `Calendar ID`, `Service IDs` and `Duration, min` by exact name – keep these names or update the Free Slots nodes.

In Google Calendar, create **one calendar per barber** and copy each calendar's ID (Calendar settings → *Integrate calendar*).

### 2. Import the workflows (in this order)

1. `barbershop-get-calendars-services.json`
2. `barbershop-free-slots.json`
3. `barbershop-book-appointment.json`
4. `barbershop-booking-agent.json`

Create credentials (NocoDB, Google Calendar OAuth2, OpenAI, Postgres) and select them in the matching nodes.

### 3. Replace the placeholders

- **Get Calendars & Services** and **Free Slots:** `YOUR_NOCODB_WORKSPACE_ID`, `YOUR_NOCODB_BASE_ID`, `YOUR_NOCODB_CALENDARS_TABLE_ID`, `YOUR_NOCODB_SERVICES_TABLE_ID`
- **Free Slots:** working hours (`workStart`, `workEnd`) and time zone in the `Set Parameters` node
- **AI Booking Agent – tool nodes:** in `Book Appointment`, `Check Free Slots`, `Get Calendars and Services` select the matching imported workflow (replaces `YOUR_..._WORKFLOW_ID`)
- **AI Booking Agent – MCP Client:** replace `YOUR_N8N_DOMAIN` with your n8n domain. The path (`/mcp/barbershop-booking`) must match the **MCP Server Trigger** path.

### 4. Activate and test

Activate the three sub-workflows and the agent workflow, then open the chat of **Chat Trigger** and write: *"Book me a haircut tomorrow afternoon."*

> 🔐 Keep all tokens in n8n credentials only – never inside node parameters.

## Security notes

- The MCP server and the webhook test entries are **unauthenticated by default**. Anyone who knows the URL could list services or create calendar events. For production, add authentication to the MCP Server Trigger and the webhooks (Bearer / header auth), or remove the webhook triggers.
- Calendar IDs are private data – never commit real ones to a repository.

## Limitations

- The timezone (Europe/Lisbon) and the example ISO format are hard-coded in the tool descriptions and in `Set Parameters`.
- Free Slots returns free **windows** (e.g. 14:00–17:30), not fixed start times; the agent picks a start inside the window. Working hours are the same for every barber and every day.
- The UTC offset is taken from the `start` value, so a search range that crosses a daylight-saving change can be off by one hour.
- The agent trusts the customer's name/phone only as free text – there is no customer database.
- No cancelling or rescheduling yet.
- Chat memory is shared per chat session; there is no user identification.

## Possible improvements

- Cancel and reschedule tools
- Telegram or WhatsApp as the customer channel
- Confirmation messages and reminders (e.g. 24 h before the appointment)
- Customer records in NocoDB (name, phone, history)
- Authentication on MCP and webhooks, timezone as a setting

## Author

Built by [@Anatolii33](https://github.com/Anatolii33). Available for n8n automation and AI agent projects.
