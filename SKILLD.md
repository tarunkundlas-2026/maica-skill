## MAICA — AI Voice Batch Calling Skill

MAICA lets your Mind place automated voice calls to any list of people or businesses. You provide the contacts and the goal — MAICA calls everyone, conducts the conversation using a natural AI voice, and returns a summary for each call.

---

## What This Skill Does

- Calls any list of phone numbers using an AI voice agent
- Personalises each call with dynamic variables (names, roles, context)
- Returns a per-call AI-generated summary when all calls are done
- Works for any use case: recruiting, sales, research, reservations, reminders

---

## Credentials

This skill requires one credential:

- `MAICA_API_KEY` — your API key for the MAICA platform

Check your MEMORY.md first. If the key is saved there, use it without asking. If not, ask the user once and save it immediately to memory.

Every MCP tool call must include this key. The MCP server handles authentication automatically once the key is configured.

---

## MCP Server

Connect your Mind to the MAICA MCP server:

- **Endpoint**: `https://app-staging-backend-mcp.maica24.com/mcp`
- **Transport**: StreamableHTTP
- **Health check**: `https://app-staging-backend-mcp.maica24.com/health`
- **Auth header**: `Authorization: Bearer YOUR_MAICA_API_KEY`

Once connected, four tools are available to your Mind.

---

## Tools

### 1. list_phone_numbers

Lists all outbound phone numbers available in the MAICA workspace.

No parameters required.

Call this first to get the phone number ID needed for placing calls. Save the `_id` of the number you want to use.

Example response:
```
_id: 6847a1c2f3d0a90012b4e3f1
phoneNumber: +14155552671
label: US Outbound — Sales
status: active
```

---

### 2. list_maica_agents

Lists all voice agents available in the MAICA workspace.

No parameters required.

A voice agent defines the personality, tone, and goal of the calls. Each agent has a system prompt with `{{placeholder}}` slots that get filled per call using dynamic variables. Save the `_id` of the agent you want to use.

Example response:
```
_id: 6847b3e1a2c0b10034d9f2a0
name: Friendly Outreach Agent
systemPrompt: You are calling {{contact_name}} regarding {{purpose}}...
```

---

### 3. submit_batch_calls

Submits a batch of outbound calls to a list of contacts.

Required parameters:
- `call_name` — a label for this batch run (e.g. "Hotel Availability Check — June 2025")
- `agent_id` — the `_id` from list_maica_agents
- `phone_number_id` — the `_id` from list_phone_numbers
- `candidates` — array of contact objects

Each candidate object:
- `phone_number` — international format, e.g. `+919876543210` (required)
- `id` — your own reference ID, returned in status responses (optional)
- `dynamic_variables` — key-value pairs injected into the agent's prompt for this specific call (optional)

Save the `_id` returned in the response. This is your `batch_call_id` — you need it to check status.

---

### 4. get_batch_call_status

Returns the status and results for a submitted batch job.

Required parameters:
- `batch_call_id` — the `_id` returned by submit_batch_calls
- `include_summary` — always pass `true` to get AI-generated call summaries

Call this after submitting a batch. Poll every 30–60 seconds until the batch status is `completed`.

The response includes a `batch_summary` array with one AI-generated summary per completed call. Each summary entry is matched to a recipient by phone number.

Recipient statuses:
- `pending` — not yet dialled
- `in_progress` — call currently active
- `completed` — call finished
- `failed` — could not connect (busy, no answer, invalid number)

---

## Workflow

Every MAICA batch calling job follows this sequence:

**Step 1 — Get available resources**
Call `list_phone_numbers` and `list_maica_agents`. Confirm with the user which phone number and agent to use, or pick the most appropriate ones automatically if the user does not specify.

**Step 2 — Build the candidate list**
Take the contacts the user provides (from a message, a sheet, a search result, or any source). Format each one as a candidate object with `phone_number` and `dynamic_variables` matching the agent's prompt placeholders.

**Step 3 — Submit the batch**
Call `submit_batch_calls` with the call name, agent ID, phone number ID, and candidate list. Save the returned `_id` as `batch_call_id`.

**Step 4 — Monitor until complete**
Call `get_batch_call_status` with `include_summary: true`. Repeat every 30–60 seconds until all recipients show `completed` or `failed`. Report progress to the user as you poll.

**Step 5 — Return results**
Present the AI-generated summary for each contact. If the user wants results saved to a Google Sheet or sent to Telegram, do that now using the summaries from `batch_summary`.

---

## Dynamic Variables

Dynamic variables personalise each call at the individual level.

The agent's system prompt is written with `{{placeholder}}` syntax:

```
You are calling {{contact_name}} from {{company}}.
Your goal is {{call_goal}}.
```

Each candidate in the batch passes matching key-value pairs:

```
dynamic_variables: {
  "contact_name": "Priya",
  "company": "Sunrise Hotel",
  "call_goal": "check table availability between 7 PM and 10 PM tonight"
}
```

Rules:
- Keys must exactly match placeholder names (without the `{{ }}` braces)
- Missing keys cause the placeholder to be spoken out loud verbatim
- Variables can be anything: names, purposes, dates, locations, questions

---

## Example Use Cases

These examples show how to trigger a batch calling campaign from a Telegram message or chat.

---

**Restaurant / Hotel Availability**

```
Find all non-veg restaurants in Mohali with a rating above 4.5.
Call each one and ask if they have table availability tonight between
7 PM and 10 PM for a group of 4. Save the summaries.
```

Agent prompt:
```
You are calling {{restaurant_name}}, a restaurant in {{city}}.
Ask if they have table availability between 7 PM and 10 PM tonight
for a group of 4 people. Ask if a prior reservation is needed.
Be polite and brief.
```

---

**Recruiter Outreach**

```
Here are 5 candidates:
- Priya (+919876543210), Backend Engineer at TechCorp
- Raj (+918765432109), Designer at DesignCo
Call them all using the Recruiter Agent.
Label this batch "June Outreach".
```

Agent prompt:
```
You are a recruiter calling {{candidate_name}}, a {{role}} at {{company}}.
Introduce yourself and ask if they are open to a new opportunity.
Keep it under 2 minutes.
```

---

**Sales Follow-up**

```
Call these leads and ask if they received our proposal and
have any questions. Here is the list: [phone numbers]
```

Agent prompt:
```
You are following up with {{lead_name}} from {{company}}.
Ask if they received the proposal sent on {{sent_date}} and
whether they have any questions or need clarification.
```

---

**Appointment Reminders**

```
Remind these patients about their appointment tomorrow at {{time}}.
Ask them to confirm attendance.
```

Agent prompt:
```
You are calling {{patient_name}} to remind them of their appointment
tomorrow at {{time}} with {{doctor_name}}.
Ask them to confirm: reply yes to confirm, or tell you if they need to reschedule.
```

---

## Using MAICA from Telegram

Once this skill is added to your Mind and your Mind is connected to Telegram:

1. Open your Mind's Telegram chat
2. Send a message describing who to call and what the goal is
3. Optionally paste a list of names and phone numbers directly in the message
4. Your Mind will handle the rest — pick the agent, build the candidate list, submit the batch, and report back with summaries

You do not need to mention tool names or know the API. Describe the task in plain language — your Mind figures out the steps.

---

## Error Reference

| Error | Cause | Fix |
|-------|-------|-----|
| 401 Unauthorized | Missing or invalid API key | Check MAICA_API_KEY in memory |
| 404 Not Found | Invalid agent or batch ID | Re-run list_maica_agents to get a fresh ID |
| 422 Unprocessable Entity | Bad phone format or missing field | Use international format: +countrycode followed by number |
| 500 Internal Server Error | MAICA backend error | Wait 30 seconds and retry |
| API unreachable | Network or server down | Check health endpoint first |

---

## Notes

- Phone numbers must be in international format: `+` followed by country code and number
- A batch job persists on the server — you can always re-check its status later using the saved `batch_call_id`
- Failed recipients (busy, no answer) are noted in the status response; a human can follow up manually
- Always use `include_summary: true` when fetching status if you plan to share or save results
