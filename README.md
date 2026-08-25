# Dead Lead Follow-up

> An AI-powered, human-in-the-loop automation that resurrects dormant leads:
> it drafts personalized follow-up emails with an LLM and files them in Gmail,
> so a human always approves before anything is sent.

![n8n](https://img.shields.io/badge/automation-n8n-EA4B71?logo=n8n&logoColor=white)
![ClickUp](https://img.shields.io/badge/crm-ClickUp-7B68EE?logo=clickup&logoColor=white)
![Gmail](https://img.shields.io/badge/email-Gmail%20Drafts-EA4335?logo=gmail&logoColor=white)
![OpenRouter](https://img.shields.io/badge/llm-OpenRouter-6467F2)
![Human in the loop](https://img.shields.io/badge/review-human--in--the--loop-00B8D9)

---

## Why this exists

Dormant leads rarely die because someone said "no" — they die because nobody
followed up. Manual follow-up is slow, inconsistent, and easy to deprioritize,
while fully automated outreach feels robotic and burns relationships.

**Dead Lead Follow-up** takes the middle path:

1. Every morning it scans a ClickUp list for leads still marked **TO DO**
2. An LLM writes a short, warm, personalized check-in email for each one
3. The email lands as a **Gmail draft** — never sent automatically
4. The task moves to **Draft Done**, waiting for human review
5. When you send it, the lead's journey ends at **Follow-up Sent**

The result: consistent outreach at machine speed, with human judgment
kept exactly where it matters most.

## Architecture

```mermaid
flowchart LR
    A["⏰ Schedule Trigger\n(daily 10:00)") --> B["📋 Get Dead Leads\nClickUp · filter: TO DO"]
    B --> C["✍️ Generate AI Follow-up\nLLM Agent"]
    C --> D["📨 Create a draft\nGmail · HTML + CTA button"]
    D --> E["🏷️ Set Draft Done\nClickUp update"]
```

**Status lifecycle** (single source of truth lives in your board):

```
TO DO ──(AI draft created)──► Draft Done ──(human sends)──► Follow-up Sent
```

The workflow treats your board as the database: nothing is sent without a
draft existing first, and every state transition is reflected back onto the
task itself.

## Features

| Feature | Detail |
|---|---|
| **Server-side status filtering** | Only `TO DO` leads are fetched — filtering happens inside the ClickUp API query, not in post-processing |
| **Personalization via custom fields** | The LLM receives the lead's name, company and original interest notes; task titles are only used as fallback |
| **Named-field resolution** | Custom fields are located by *name* (regex-matched), not array position — resilient to field reordering |
| **HTML call-to-action** | Each draft ends with a styled "Schedule a Call" button linking to your booking form |
| **Anti-hallucination guard** | Prompt forbids the model from inventing URLs; the only link in the email is yours |
| **Duplicate protection** | Status flips to `Draft Done` immediately after drafting, so tomorrow's run skips already-contacted leads |
| **Timezone-aware scheduling** | Trigger fires at 10:00 in the workflow's configured timezone |

## How each node works

| Node | Type | What it does |
|---|---|---|
| Schedule Trigger | `scheduleTrigger` | Fires daily at 10:00 (configurable) |
| Get Dead Leads | `clickUp.getAll` | Fetches all tasks whose status is `TO DO`; `returnAll` enabled |
| Generate AI Follow-up | LangChain **AI Agent** | Writes a 3-sentence check-in using lead name / company / interest notes; rules forbid subjects, URLs and corporate tone |
| OpenRouter Chat Model | `lmChatOpenRouter` | The LLM behind the agent — any OpenRouter-hosted model works |
| Create a draft | `gmail.draft` | Builds an HTML draft: AI copy converted from newlines, sign-off preserved, booking button appended |
| Set Draft Done | `clickUp.update` | Marks the task `Draft Done` so it exits tomorrow's queue |

## Setup

### Prerequisites

| Requirement | Notes |
|---|---|
| [n8n](https://n8n.io) ≥ 2.x | Self-hosted recommended; any instance you control |
| ClickUp account + list | With the schema below |
| Google account | Gmail for drafts; GCP project for OAuth |
| [OpenRouter](https://openrouter.ai) API key | One key unlocks hundreds of models |
| [Tally](https://tally.so) form (optional) | Or any booking link: Calendly, Cal.com, … |

### 1 · ClickUp list schema

Create statuses and custom fields with these exact names:

**Statuses:** `TO DO` → `Draft Done` → `Follow-up Sent`
*(status matching is case-insensitive but must be word-exact)*

**Custom fields:**

| Field name | Type | Used for |
|---|---|---|
| `Company Name` | Text | Mentioned in the AI copy |
| `Email Address` | Text | Draft recipient |
| `Name` | Text | The lead's actual person-name (greeting) |
| `Original Interest Notes` | Text | Context the AI references |

> 💡 Fields are matched by name (with regex), so extra fields are safe.
> The workflow falls back to the task title when `Name` is empty.

### 2 · Google OAuth credentials

In Google Cloud Console (same project for both):

1. Enable the **Gmail API**
2. OAuth consent screen → add yourself as a **test user**
3. Create an **OAuth client ID** (Web application) with redirect URI
   `http://localhost:5678/rest/oauth2-credential/callback`
   *(Google exempts localhost from HTTPS requirements)*
4. In n8n create a **Gmail OAuth2** credential → paste ID/secret → Connect

### 3 · Import & configure

1. n8n → *Workflows → Import from File* → `workflow.json`
2. Attach credentials to the two ClickUp nodes, Gmail node and OpenRouter node
3. Replace placeholders: `YOUR_CLICKUP_TEAM_ID`, `YOUR_CLICKUP_SPACE_ID`,
   `YOUR_CLICKUP_LIST_ID`, `YOUR_TALLY_FORM_SLUG`
4. Edit the prompt persona/sign-off to sound like *you*
5. Run once manually, then activate

## Configuration reference

| Setting | Where | Default | Notes |
|---|---|---|---|
| Fetch hour | Schedule Trigger | 10:00 | Workflow timezone (`Asia/Manila` by default) |
| Lead status filter | Get Dead Leads | `["to do"]` | Word-exact match required |
| LLM model | OpenRouter Chat Model | `google/gemini-2.5-flash-lite` | Any OpenRouter model ID |
| Email subject | Create a draft | *"🔍 Quick check-in…"* | Emoji-safe |
| CTA button URL | Create a draft message | Tally form slug | Swap for Calendly/Cal.com |
| Button style | Inline CSS in message | Blue, rounded | Fully editable inline styles |
| Persona & sign-off | AI prompt text | Neutral placeholder | Make it sound like you |

## Adapting it to other CRMs & integrations

The pipeline is deliberately boring — five linear nodes — which makes every
layer swappable:

| Layer | This repo uses | Drop-in alternatives |
|---|---|---|
| CRM / lead store | ClickUp | HubSpot, Pipedrive, Zoho CRM, Airtable, Notion, any REST API |
| Inbox | Gmail drafts | Outlook (Microsoft 365 node), SMTP node, SendGrid draft APIs |
| LLM | OpenRouter | Native OpenAI/Anthropic/Google nodes, Ollama (local), Azure OpenAI |
| Booking link | Tally form | Calendly, Cal.com, SavvyCal — any URL works |
| Scheduler | n8n Schedule Trigger | Cron, webhook, CMS-driven triggers |

Swapping the CRM means replacing one `getAll` and one `update` call and
pointing the personalization expressions at that system's field names —
the prompt, draft construction and lifecycle logic stay identical.

## Caveats & lessons learned

These are real bugs found while running this system in production —
each cost a debugging session, so they're documented here on purpose.

1. **Omitted parameters fall back to dangerous defaults.**
   n8n's ClickUp node defaults its operation to *create*. A `getAll` node
   missing an explicit `"operation": "getAll"` silently tried to *create*
   tasks instead of listing them. Always declare `resource` and `operation`.

2. **Status names must match word-exactly.**
   Case doesn't matter (`IN PROGRESS` → `in progress` is accepted), but words
   do: updating with `COMPLETED` when the board says `complete` returns a 400.
   Verify against the board's own status slugs before wiring updates.

3. **Positional custom-field access is a time bomb.**
   `custom_fields[1].value` breaks the moment someone reorders fields.
   Always resolve fields by name with a regex and provide a sane fallback.

4. **Task title ≠ lead name.**
   Boards often use descriptive task titles ("Leads Pipelines") while the
   person's name hides in a custom field. Greeting a lead by the pipeline
   name is how you get ignored — resolve the person-name field explicitly.

5. **Stale editor tabs overwrite API edits.**
   n8n keeps per-tab canvas state. If a workflow is modified via API (or
   another device) while an old tab holds it, saving that tab reverts
   everything — silently. Refresh before editing; prefer one writer at a time;
   keep JSON backups under version control.

6. **Polling triggers have undocumented default intervals.**
   A Gmail trigger with no explicit poll schedule defaults to roughly hourly
   polling. For latency-sensitive branches, either set the interval explicitly
   or replace polling with a scheduled batch sweep.

7. **Re-runs duplicate drafts.**
   If a run fails halfway (or you test twice), tasks that already received a
   draft will receive another unless their status advanced. That's why the
   status update is part of the same atomic pipeline run.

## Security notes

- All secrets live in n8n **credentials**, never in the workflow JSON
- The published `workflow.json` is sanitized: credential references and
  instance IDs removed; placeholders marked `YOUR_*`
- OAuth apps should stay in *Testing* mode with only your accounts as test users
- Rotate any credential that has ever been pasted into a chat window 🙂
- The draft-only design means even a runaway execution cannot email anyone

## Roadmap

- [ ] **Sent-detection branch**: sweep `Draft Done` tasks against the Gmail
      sent folder and auto-move matches to `Follow-up Sent`
- [ ] **Tally → Calendar**: auto-create Google Calendar events from booking
      form submissions
- [ ] Multi-list support via a config mapping
- [ ] Optional reply-detection to stop following up on answered threads

## Author & license

Built as a portfolio piece demonstrating practical, human-in-the-loop AI
automation. Released under the [MIT License](LICENSE).
