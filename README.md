# AI Lead Management System — Meridian Realty Group

An end-to-end AI-powered lead capture, categorization, automated follow-up, and reply detection system built with n8n, Google Gemini, Airtable, and Gmail.

---

## Overview

Real estate agencies lose deals not because of bad leads — but because of slow follow-up. This system ensures every inquiry gets an instant, personalized response, is properly tracked in a CRM, and is followed up automatically — without any manual effort from the team.

Built as a portfolio project simulating a real-world deployment for a small real estate agency.

---

## What It Does

- Captures leads from a web form (Tally)
- Checks if the lead already exists in Airtable (duplicate detection by email)
- Uses AI to classify each lead as **Hot Lead**, **Warm Lead**, **Just Browsing**, or **Unrelated** based on their message
- For new leads: creates a CRM record in Airtable and sends a personalized AI-written reply email
- For returning leads: appends the new message to the existing record, re-evaluates their category, updates status to **Has New Message**, stamps the `Last Message Date`, and sends a follow-up email
- For unrelated/spam submissions: logs to a separate Spam/Review table (new contacts) or sends a Slack alert with no action taken (returning contacts)
- Follows up automatically at Day 3 and Day 7 if the lead hasn't responded — based on first submission date for new leads, or last message date for returning leads
- Detects when a lead replies to any email and immediately updates their status to **Responded**, stops the follow-up sequence, and alerts the team via Slack
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
Schedule Trigger → Airtable (Search eligible leads) → If (leads exist?)
  → Switch (Day 3 or Day 7?)
      ├── [Day 3] Gemini (Write follow-up 1) → Gmail → Airtable (Update status + Follow-up 1 Sent)
      └── [Day 7] Gemini (Write follow-up 2) → Gmail → Airtable (Update status + Follow-up 2 Sent)
```

### Pipeline 3 — Reply Detection (Triggered by Gmail)

```
Gmail Trigger (new reply) → Airtable (Search by sender email)
  → If (lead found?)
      ├── [true] Airtable (Status → Responded) → Slack (sys-leads alert)
      └── [false] No Operation
```

### Pipeline 4 — Error Handler (Linked via Workflow Settings)

```
Error Trigger → Slack (sys-errors)
```

Catches any failed execution across all pipelines and sends an immediate Slack alert containing:
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
| Gmail | Automated email delivery and reply detection |
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
- Their name (first name only) and specific message content
- Category-aware tone and call to action — always uses "we", "our", "us" (never "I" or "my")
- A context check: if the message field contains `--- New Message ---`, the LLM detects this is a returning lead and responds accordingly
- Realism guardrail: never claims specific properties are available, never makes promises or guarantees
- No pressure tactics: banned phrases include "properties move fast", "before it disappears", "don't miss out"
- Hard limit: under 200 words, returned as simple HTML using `<p>` tags, signed as "The Meridian Realty Team"

### Follow-up Emails
Two automated follow-ups, each with category-aware tone:

- **Day 3** — first follow-up, briefly acknowledges previous unanswered email, offers to set up a call this week
- **Day 7** — final follow-up, acknowledges previous outreach without counting attempts or implying the lead is unresponsive, closes the sequence gracefully with one clear offer

Both follow-ups: written in first-person plural, no clichés ("touching base", "circling back"), no pressure tactics, HTML formatted.

---

## Key Features

- **Duplicate detection** — checks by email before any write operation; returning leads have their existing record updated, never duplicated
- **Conversation history** — new messages from returning leads are appended to the existing message field with a `--- New Message ---` separator, giving the LLM full context without a separate memory system
- **Dynamic re-categorization** — returning leads are re-classified based on their latest message, so a browsing lead can automatically be upgraded to a hot lead
- **Last Message Date tracking** — follow-up sequence resets based on the date of the most recent message for returning leads, not the original submission date
- **Reply detection** — Gmail trigger automatically detects when a lead replies, marks them as Responded, and stops the follow-up sequence
- **Resilient field parsing** — all Tally fields parsed by label name using `.find()`, not by array index, so form reordering never silently breaks the workflow
- **Error monitoring** — a dedicated error handler workflow catches any failed execution in production and sends a Slack alert with node name, error message, and direct execution link
- **Clean Slack notifications** — n8n attribution footer disabled on all Slack nodes
- **Full audit trail** — every lead, status change, category update, follow-up date, and reply logged in Airtable

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
| Date Received | DateTime | ISO format, set on first submission — never overwritten |
| Last Message Date | DateTime | Updated every time a returning lead sends a new message |
| Follow-up 1 Sent | DateTime | Stamped when Day 3 follow-up is sent |
| Follow-up 2 Sent | DateTime | Stamped when Day 7 follow-up is sent |
| Days Since Received | Formula | Auto-calculated by Airtable |

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
| `sys-leads` | New lead notifications and reply alerts with name, email, category, and message preview |
| `sys-spam-log` | Unrelated/spam alerts for both new and returning contacts |
| `sys-errors` | Execution failure alerts with node name, error message, and execution link |

---

## Files

| File | Description |
|------|-------------|
| `Meridian Lead Pipeline Workflow.json` | Main intake workflow — form capture, deduplication, classification, CRM logging, AI email, Slack notifications |
| `Meridian follow-up.json` | Daily follow-up sequence — Day 3 and Day 7 automated emails |
| `Meridian Reply Detection.json` | Reply detection workflow — detects lead replies, updates status, alerts team via Slack |
| `Meridian Error Handler.json` | Error handler workflow — catches failed executions and sends Slack alert to sys-errors |
| `CHANGELOG_meridian-lead-pipeline.md` | Full change log across all sessions — bugs found, fixes applied, and decisions made |

---

## How to Import

1. In n8n, go to **Workflows → Import from File**
2. Import all four JSON files separately
3. Reconnect credentials on each workflow:
   - Gemini API key (LangChain Gemini nodes)
   - Airtable personal access token
   - Gmail OAuth2
   - Slack API token
4. Update Airtable base and table IDs in all Airtable nodes to match your own base
5. Update the Slack channel IDs in all Slack nodes to match your own workspace
6. In the main Lead Pipeline workflow → **Settings → Error Workflow** → select **Meridian Error Handler**
7. Do the same for the Follow-up and Reply Detection workflows
8. Activate all four workflows
9. Submit a test form via Tally to verify the full intake flow end to end
10. To test the error handler: temporarily corrupt an Airtable base ID, submit a test form, confirm the Slack alert fires, then revert the ID
11. To test reply detection: reply to any automated email from a test lead account and confirm Status updates to Responded and Slack alert fires

---

## What I Learned

- Designing multi-branch conditional workflows with Switch and If logic in n8n
- Integrating Google Gemini via LangChain nodes for both classification and generation tasks
- Debugging n8n expression context — when to use `$json` vs explicit `$('Node name').item.json` references, and why implicit context breaks after flow reordering
- Building duplicate detection using Airtable search + If node before any write operations
- Structuring conversation history for returning leads so the LLM has full context without a separate memory system
- Designing follow-up sequences that reset based on last activity date, not just first submission date
- Building a reply detection workflow using Gmail trigger with Simplify off to access full email payload
- Prompt engineering for tone consistency — enforcing first-person plural, banning clichés and pressure tactics, controlling email formatting via HTML output
- Setting up and testing error handler workflows in production mode (editor test runs bypass the error trigger)
- Managing Git branches and GitHub remotes for n8n workflow version control

---

## About

Built by Eumi Sabueto — AI Automation Developer specializing in n8n, Make, and Airtable, with a focus on integrating LLMs like Gemini, Claude, and OpenAI into practical business workflows.
