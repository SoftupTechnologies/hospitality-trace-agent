# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an **n8n workflow automation system** (not a traditional code repository) that processes Apaleo hotel PMS webhooks to generate operational tasks in Sweeply using AI (GPT-5). Business rules are defined externally in Google Sheets, not hardcoded.

## Repository Structure

```
├── booking_workflow/
│   └── Booking Traces.json          — Main entry workflow for booking webhooks
├── reservation_workflow/
│   └── Reservation Traces.json      — Main entry workflow for reservation webhooks
├── ai_agent_workflow/
│   └── Sub Trace Ai Agent.json      — Core AI agent logic (GPT-5 powered task generation)
├── lock_workflow/
│   ├── Manage Lock Create and Wait.json  — Concurrency control (lock acquisition)
│   └── Release Lock n8n.json             — Lock release sub-workflow
├── reporting_workflow/
│   └── Daily Traces Email.json      — Scheduled daily report workflow
├── deprecated_workflow/
│   └── Traces Agent - Deprecated.json   — Legacy, do not use
├── system.md                        — System prompt for the AI agent
├── user.md                          — User prompt template
├── System prompt documentation.md   — Detailed scenario documentation and rule examples
├── Rulebook.xlsx                    — Excel version of the business rules (canonical reference)
├── readme.md                        — Full setup guide, architecture docs, and troubleshooting
└── CLAUDE.md                        — This file
```

## Architecture

```
Apaleo Webhook → [Booking/Reservation Traces] → Lock Acquisition → Data Extraction
    → [Sub Trace AI Agent] → Rule Book Lookup → Duplicate Check → Task Generation → Sweeply POST
    → [Release Lock] → Cleanup
```

**Data flow phases:**
1. **Webhook reception** — Apaleo sends `booking/changed`, `booking/created`, `reservation/changed`, or `reservation/created`
2. **Lock acquisition** — Database lock via Supabase `webhook_event_queue` table (key: `{topic}:{propertyId}:{entityId}`) with 10-second polling
3. **Data extraction** — Fetch full booking/reservation from Apaleo API with expansions
4. **AI processing** — GPT-5 agent loads rule books from Google Sheets, checks existing traces in `trace_logs`, generates structured task JSON
5. **Task posting** — POST to Sweeply API (Basic Auth), log to Supabase `trace_logs`
6. **Lock release** — Delete lock entry, unblock queued webhooks

**AI Agent tool call order** (mandatory sequence in `user.md`):
1. Get System Rule Book (Google Sheets)
2. Get Property Rule Book (Google Sheets)
3. Get Trace Logs (Supabase)

## Technology Stack

| Service | Purpose | Auth Method |
|---------|---------|-------------|
| n8n | Workflow orchestration | Cloud/self-hosted instance |
| OpenAI GPT-5 | AI task generation | API Key (n8n credential) |
| Apaleo | Hotel PMS webhooks & API | OAuth2 (n8n credential) |
| Google Sheets | Dynamic rule book storage | OAuth2 (n8n credential) |
| Supabase/PostgreSQL | Trace logs & locking | Service Role Secret |
| Sweeply | Task management | Basic Auth |
| Resend | Email delivery | Bearer Token |

## Database Schema (Supabase)

**`trace_logs`** — Records of all generated tasks with `apaleo_property_id`, `booking_id`, `webhook_topic`, `trace` (JSONB), `sweeply_status`, `sweeply_trace_id`

**`webhook_event_queue`** — Concurrency locks with unique `event_identifier` constraint

## Working with This Repo

There are **no build, test, or lint commands** — this is a workflow configuration project. Changes are made by:

1. Editing workflow JSON files (exported from n8n)
2. Editing AI prompt markdown files (`system.md`, `user.md`)
3. Updating documentation (`readme.md`, `System prompt documentation.md`)

Workflows are deployed by importing the JSON files into an n8n instance. Credentials and placeholders (e.g., `PROPERTY_ID`, `ACCOUNT_ID`, `REDACTED:REDACTED`) must be configured per the setup guide in `readme.md`.

## Key Business Logic

- **Rule evaluation**: Rules from Google Sheets define which guest comments trigger task creation. Rules have hard-stop domains (e.g., payment/invoice intents are always skipped) and enabled rule matching.
- **Lifecycle decisions**: The AI determines create vs update vs skip based on existing traces and operational intent (meaning-based, not text-based).
- **Scope isolation**: Booking-level tasks vs reservation-level tasks are handled separately.
- **Department routing**: Tasks auto-route to Reception, Housekeeping, or Technik based on rules.
- **Canonical integrity**: Task titles are fixed per operational meaning — not generated freely by the AI.

## Important Notes

- The `.env` file contains Supabase credentials and is gitignored
- Workflow JSON files are n8n exports — edit via n8n UI and re-export, or edit JSON directly with care (node IDs and connections must stay consistent)
- `system.md` and `user.md` are the prompts used inside the `Sub Trace Ai Agent.json` workflow — changes here directly affect AI behavior
