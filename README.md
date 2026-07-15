# AI Lead Management System — Meridian Realty Group

An end-to-end AI-powered lead capture, categorization, and automated follow-up system built with n8n, Google Gemini, Airtable, and Gmail.

---

## Overview

Real estate agencies lose deals not because of bad leads — but because of slow follow-up. This system ensures every inquiry gets an instant, personalized response and is properly tracked in a CRM, without any manual effort from the team.

Built as a portfolio project simulating a real-world deployment for a small real estate agency.

---

## What It Does

- Captures leads from a web form (Tally)
- Checks if the lead already exists in Airtable (duplicate detection by email)
- Uses AI to classify each lead as **Hot Lead**, **Warm Lead**, **Just Browsing**, or **Unrelated** based on their message
- For new leads: creates a CRM record in Airtable and sends a personalized AI-written reply email
- For returning leads: appends the new message to the existing record, re-evaluates their category, updates status to **Has New Message**, and sends a follow-up email
- For unrelated/spam submissions: logs to a separate Spam/Review table (new contacts) or sends a Slack alert with no action taken (returning contacts)
- Notifies the team via Slack for every lead event
- Catches all workflow errors and sends a Slack alert with the failed node, error message, and a direct link to the execution log

---

## Workflow Architecture

### Pipeline 1 — Lead Intake (Triggered by Tally form submission)

```
Tally Form → Webhook → Parse Fields → Airtable (Search by email)
  → Gemini (Classify message)
      → If (returning or new?)
            ├── [Returning] Switch1
            │     ├── [Unrelated] Slack alert — no action taken
            │     └── [Not Unrelated] Update record → Gemini (Write email) → Slack → Gmail
            └── [New] Switch
                  ├── [Unrelated] Airtable (Spam/Review) → Slack spam-log
                  └── [Not Unrelated] Airtable (Create lead) → Gemini (Write email) → Slack → Gmail
```

### Pipeline 2 — Follow-up Sequence (Triggered daily by schedule)

```
Schedule Trigger → Airtable (Search uncontacted leads) → If (leads exist?)
  → Switch (Day 3 or Day 7?)
      → Gemini (Write follow-up) → Gmail (Send) → Airtable (Update status)
```

### Pipeline 3 — Error Handler (Linked via Workflow Settings)

```
Error Trigger → Slack (sys-errors)
```

Catches any failed execution across the main pipeline and sends an immediate Slack alert containing:
- The node that failed
- The error message
- A direct link to the failed execution log in n8n

---

## Tech Stack

| Tool | Role |
|------|------|
| n8n (self-hosted via Docker) | Workflow automation engine |
| Google Gemini Flash Lite | Lead classification + email generation |
| Airtable | Lead CRM, status tracking, spam review |
| Tally | Lead capture form |
| Gmail | Automated email delivery |
| Slack | Internal notifications and error monitoring |

---

## AI Prompt Design

### Lead Classification
Classifies each lead into exactly one category based on urgency signals in their message:

- **Hot Lead** — pre-approved, ready to act, ASAP urgency
- **Warm Lead** — interested with specific criteria but no immediate urgency
- **Just Browsing** — early-stage, exploratory, no criteria or timeline
- **Unrelated** — wrong numbers, casual chat, spam, or support issues unrelated to property

The prompt is constrained to return only the category name with no punctuation or explanation, making Switch node string matching reliable.

### Personalized Email Generation
Writes a unique reply for each lead using:
- Their name and specific message content
- Category-aware tone and call to action
- A context check: if the message field contains `--- New Message ---`, the LLM detects this is a returning lead and responds with a re-engagement message instead of a first-contact reply
- Realism guardrail: never claims specific inventory is available — uses phrases like "I can look into our current options" or "I'll check with our team"
- Hard limit: under 200 words, signed as "The Meridian Realty Team"

### Follow-up Emails
Two automated follow-ups with distinct tone per category and lead stage, designed to re-engage without feeling spammy. The Day 7 follow-up closes the sequence gracefully regardless of whether the lead has responded.

---

## Key Features

- **Duplicate detection** — checks by email before any write operation; returning leads have their existing record updated, never duplicated
- **Conversation history** — new messages from returning leads are appended to the existing message field with a `--- New Message ---` separator, giving the LLM full context without a separate memory system
- **Dynamic re-categorization** — returning leads are re-classified based on their latest message, so a browsing lead can automatically be upgraded to a hot lead
- **Resilient field parsing** — all Tally fields parsed by label name using `.find()`, not by array index, so form reordering never silently breaks the workflow
- **Error monitoring** — a dedicated error handler workflow catches any failed execution in production and sends a Slack alert with node name, error message, and direct execution link
- **Clean Slack notifications** — n8n attribution footer disabled on all Slack nodes
- **Full audit trail** — every lead, status change, category update, and follow-up date logged in Airtable

---

## Airtable Structure

### Leads Table
| Field | Type | Notes |
|-------|------|-------|
| Name | Text | Full name parsed from Tally form |
| Email | Text | Unique field — duplicates prevented at DB level |
| Phone | Text | From form |
| Message | Long text | Appended on return visits with `--- New Message ---` separator |
| Category | Select | Hot Lead, Warm Lead, Just Browsing |
| Status | Select | New, Contacted, Responded, Has New Message |
| Date Received | DateTime | ISO format, set on first submission |

### Spam/Review Table
| Field | Type | Notes |
|-------|------|-------|
| Name | Text | |
| Email | Text | |
| Phone | Text | |
| Message | Text | |
| Category | Select | Unrelated |
| Status | Select | Pending Review, Confirmed Spam, Approved (Move to Leads) |
| Date Received | Date | |

---

## Slack Channels

| Channel | Purpose |
|---------|---------|
| `sys-leads` | New lead notifications with name, email, phone, category, message preview, and Airtable link |
| `sys-spam-log` | Unrelated/spam alerts for both new and returning contacts |
| `sys-errors` | Execution failure alerts with node name, error message, and execution link |

---

## Files

| File | Description |
|------|-------------|
| `Meridian Lead Pipeline Workflow.json` | Main intake workflow — form capture, deduplication, classification, CRM logging, AI email, Slack notifications |
| `Meridian follow-up.json` | Daily follow-up sequence — Day 3 and Day 7 automated emails |
| `Meridian Error Handler.json` | Error handler workflow — catches failed executions and sends Slack alert to sys-errors |
| `CHANGELOG_meridian-lead-pipeline.md` | Full change log from initial build to final version — bugs found, fixes applied, and decisions made |

---

## How to Import

1. In n8n, go to **Workflows → Import from File**
2. Import all three JSON files separately
3. Reconnect credentials on each workflow:
   - Gemini API key (LangChain Gemini nodes)
   - Airtable personal access token
   - Gmail OAuth2
   - Slack API token
4. Update Airtable base and table IDs in all Airtable nodes to match your own base
5. Update the Slack channel IDs in all Slack nodes to match your own workspace
6. In the main Lead Pipeline workflow → **Settings → Error Workflow** → select **Meridian Error Handler**
7. Activate all three workflows
8. Submit a test form via Tally to verify the full flow end to end
9. To test the error handler: temporarily corrupt an Airtable base ID, submit a test form, confirm the Slack alert fires, then revert the ID

---

## What I Learned

- Designing multi-branch conditional workflows with Switch and If logic in n8n
- Integrating Google Gemini via LangChain nodes for both classification and generation tasks
- Debugging n8n expression context — when to use `$json` vs explicit `$('Node name').item.json` references, and why implicit context breaks after flow reordering
- Building duplicate detection using Airtable search + If node before any write operations
- Structuring conversation history for returning leads so the LLM has full context without a separate memory system
- Setting up and testing error handler workflows in production mode (editor test runs bypass the error trigger)
- Managing Git branches and GitHub remotes for n8n workflow version control

---

## About

Built by Eumi Sabueto — AI Automation Developer specializing in n8n, Make, and Airtable, with a focus on integrating LLMs like Gemini, Claude, and OpenAI into practical business workflows.
