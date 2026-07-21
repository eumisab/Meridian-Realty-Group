# Meridian Lead Pipeline Workflow — Change Log

**Workflow:** Meridian Lead Pipeline Workflow  
**Session date:** 2025-07-15  
**Tool:** n8n (self-hosted via Docker)  
**Integrations:** Tally · Airtable · Google Gemini · Gmail · Slack

---

## Summary

Full audit, bug fix, and structural refactor of the Meridian Lead Pipeline. The workflow now correctly handles new leads, returning leads, and unrelated/spam submissions across two separate routing branches, with error monitoring via a dedicated error handler workflow.

---

## Flow Overview (Final)

```
Webhook (Tally)
  └── Edit Fields
        └── Search records (Airtable — Leads)
              └── Basic LLM Chain (Gemini — classify message)
                    └── If (returning or new lead?)
                          ├── [true] Returning lead
                          │     └── Switch1
                          │           ├── [Unrelated] Send a message3 (Slack — sys-spam-log, no action)
                          │           └── [Not Unrelated] Update record
                          │                 └── Basic LLM Chain1 (Gemini — write email)
                          │                       └── Send a message1 (Slack — sys-leads)
                          │                             └── Send a message (Gmail)
                          └── [false] New lead
                                └── Switch
                                      ├── [Unrelated] Create a record1 (Airtable — Spam/Review)
                                      │     └── Send a message2 (Slack — sys-spam-log)
                                      └── [Not Unrelated] Create a record (Airtable — Leads)
                                            └── Basic LLM Chain1 (Gemini — write email)
                                                  └── Send a message1 (Slack — sys-leads)
                                                        └── Send a message (Gmail)
```

---

## Bugs Fixed

### 1. Workflow execution order was wrong — classify before search
**Node:** `Edit Fields → Basic LLM Chain → Switch → Search records`  
**Problem:** The original flow classified the lead with the LLM before checking if they already existed in Airtable. This meant the Switch was routing based on category before the `If` (new vs. returning) check could happen, making the duplicate detection branch unreachable.  
**Fix:** Reordered the flow so `Search records` runs immediately after `Edit Fields`, then `Basic LLM Chain`, then `If`. The correct order is now: parse → search → classify → route.

---

### 2. `Switch` — stray `=` in `rightValue` caused both outputs to fire
**Node:** `Switch` (new lead path), `Switch1` (returning lead path)  
**Problem:** Rule 0 (Unrelated check) had `"rightValue": "=Unrelated"` — the leading `=` caused n8n to treat it as an expression. Since `=Unrelated` is not a valid expression, the condition evaluated unpredictably, causing both output 0 and output 1 to fire simultaneously for every lead regardless of category.  
**Fix:** Removed the stray `=`. Both Switch nodes now use plain string `"Unrelated"` as the right value. Also updated left value references from `$json.text` to `$('Basic LLM Chain').item.json.text` for explicit node scoping.

---

### 3. `Search records` — `returnAll` was an empty expression
**Node:** `Search records`  
**Problem:** `returnAll` was set to `"="` (empty expression string) instead of a boolean. This caused n8n to default to returning only 1 result, but in an undefined way — the field was not properly set.  
**Fix:** Set `returnAll` to `false` (boolean). One result is intentional since email is unique per lead.

---

### 4. `If` node — used implicit `$json.id` instead of explicit node reference
**Node:** `If`  
**Problem:** The condition used `$json.id` to check if a matching lead was found. Since the node directly upstream changed during the reorder, `$json` context was unreliable.  
**Fix:** Updated to `$('Search records').item.json.id` for explicit, stable node reference.

---

### 5. `Update record` — only updated Message field, lost other field data
**Node:** `Update record`  
**Problem:** The original update only wrote the new `Message` value. It did not update `Category`, `Status`, `Name`, or `Phone` — meaning returning leads never had their category re-evaluated or their status updated in Airtable.  
**Fix:** Updated to write:
- `Message` — appended with `\n\n--- New Message ---\n` separator so the email LLM can detect returning leads
- `Category` — re-evaluated by the LLM based on new message
- `Status` — set to `Has New Message`
- `Name` and `Phone` — preserved from the existing Airtable record (not overwritten with new form data)

---

### 6. `Edit Fields` — Email, Phone, Message used positional index instead of label lookup
**Node:** `Edit Fields`  
**Problem:** Email was mapped as `$json.body.data.fields[2].value`, Phone as `[3]`, and Message as `[5]`. Positional indexing breaks silently if Tally ever reorders fields or a new field is added.  
**Fix:** Switched all fields to label-based lookup using `.find(f => f.label === "...")?.value`. Also removed the `Company name` field which was no longer needed.

---

### 7. `Basic LLM Chain` — prompt referenced `$json.Message` instead of explicit node
**Node:** `Basic LLM Chain`  
**Problem:** The classification prompt used `{{ $json.Message }}` which depended on implicit context. After the flow reorder, `$json` no longer pointed to `Edit Fields` output reliably.  
**Fix:** Updated to `{{ $('Edit Fields').item.json.Message }}` for explicit reference.

---

### 8. `Send a message (Gmail)` — message body used `$json.text` instead of explicit node
**Node:** `Send a message` (Gmail)  
**Problem:** The email body was set to `{{ $json.text }}`, which relied on `$json` implicitly pointing to `Basic LLM Chain1`. This is fragile — if any node is inserted between them, it silently sends a blank email.  
**Fix:** Updated to `{{ $('Basic LLM Chain1').item.json.text }}` for explicit, stable reference.

---

### 9. Returning lead classified as Unrelated was incorrectly written to Spam/Review
**Node:** `Switch1` (returning lead path)  
**Problem:** The original flow had no separate handling for returning leads. When a returning lead's new message was classified as Unrelated, they were written to the `Spam/Review` Airtable table — polluting it with known contacts and losing the update context.  
**Fix:** Removed the `Create a record` connection from `Switch1` output 0. Replaced with `Send a message3` (Slack → `sys-spam-log`) with the message: `⚠️ Returning lead [Name] sent an unrelated message: [Message] — no action taken.` No Airtable write occurs for returning Unrelated leads.

---

### 10. `Categorize` node was misnamed and caused confusion
**Node:** `Categorize` (Google Gemini Chat Model — sub-model for Basic LLM Chain)  
**Problem:** The Gemini model sub-node attached to `Basic LLM Chain` was named `Categorize` instead of a standard model name, making it unclear which LLM Chain it belonged to.  
**Fix:** Renamed to `Google Gemini Chat Model` to match naming convention used by `Google Gemini Chat Model1`.

---

## Other Changes

### Gemini model updated
**Node:** `Google Gemini Chat Model1` (email writer)  
**Change:** Model changed from `models/gemini-3.5-flash` to `models/gemini-3.1-flash-lite` to match the classification model and reduce API cost.

### Email word limit increased
**Node:** `Basic LLM Chain1` (email writer prompt)  
**Change:** Word limit increased from 100 to 200 words to give the LLM more room for warm, natural responses without truncation.

### Slack `includeLinkToWorkflow` disabled on all Slack nodes
**Nodes:** `Send a message1`, `Send a message2`, `Send a message3`  
**Change:** Set `"includeLinkToWorkflow": false` in `otherOptions` on all Slack nodes to remove the auto-appended n8n attribution footer from Slack messages.

### `Has New Message` status added to Leads table schema
**Nodes:** `Create a record`, `Update record`  
**Change:** Added `Has New Message` as a valid `Status` option in the Airtable schema for both nodes to match the Airtable field configuration.

### Airtable — Email field set to unique (database level)
**Tool:** Airtable  
**Change:** Enabled "Prevent duplicate values" on the Email field in the Leads table to guard against duplicate records caused by manual entry or bulk imports. The workflow itself returns only 1 search result by design (`returnAll: false`).

---

## New Additions

### `Switch1` node — returning lead routing
**Type:** Switch  
**Purpose:** Routes returning leads after the `If` check. Output 0 (Unrelated) → Slack alert only. Output 1 (not Unrelated) → Update record → email flow.

### `Send a message3` node — returning Unrelated Slack alert
**Type:** Slack → `sys-spam-log`  
**Purpose:** Notifies the team when a known lead sends an unrelated follow-up message. No Airtable write is made. Message format: `⚠️ Returning lead [Name] sent an unrelated message: [Message] — no action taken.`

### Error Handler Workflow (separate workflow)
**Nodes:** `Error Trigger → Slack`  
**Purpose:** Catches any failed execution in the main pipeline and sends a Slack alert to `sys-errors` with the failed node name, error message, and a direct link to the execution log.  
**Message format:**
```
❌ Meridian Pipeline Error

Node: [lastNodeExecuted]
Error: [error.message]

http://localhost:5678/workflow/[workflow.id]/executions/[execution.id]
```
**Setup:** Linked under main workflow → Settings → Error Workflow.  
**Tested:** Confirmed working by temporarily corrupting the `Search records` Airtable base ID and submitting a live test form. Slack alert fired correctly with node name `Search records` and error `Forbidden`.

---

## Known Limitations (Not Fixed — By Design)

| Item | Reason |
|---|---|
| One email per Tally submission — no batch processing | Tally fires one webhook per submit; batching not needed |
| Tally field label changes will silently break `Edit Fields` | Accepted risk; field labels are documented and stable |
| No email sent to returning leads classified as Unrelated | Intentional — no response is the correct action for known contacts sending off-topic messages |
| LLM re-classifies based on new message only, not full history | Intentional — category should reflect the lead's current intent, not their first message |
| Error handler uses localhost URL (not publicly accessible from Slack) | Acceptable for local Docker setup; update URL if migrating to cloud-hosted n8n |

---

# Session 2 — Follow-up Sequence, Reply Detection & Prompt Refinement

**Session date:** 2026-07-21
**Tool:** n8n (self-hosted via Docker)
**Integrations:** Airtable · Google Gemini · Gmail · Slack

---

## Summary

Extended the system with returning lead follow-up support, a reply detection workflow, and significant email prompt refinement. All three workflows are now production-ready and linked to the error handler.

---

## Airtable Changes

### Added `Last Message Date` field
**Type:** DateTime
**Purpose:** Tracks the date of the most recent message from a returning lead. Used by the follow-up workflow to reset the Day 3 / Day 7 sequence based on last activity, not original submission date.

### Enabled `Has New Message` as valid Status option
**Table:** Leads
**Purpose:** Added to Status field options to support the returning lead flow. Follow-up workflow now recognizes this status alongside `New` and `Contacted`.

---

## Main Pipeline Changes

### `Update record` — added `Last Message Date` field
**Node:** `Update record`
**Change:** Added `Last Message Date` → `$now.toISO()` so every returning lead submission stamps the date of their latest message. This is the reference date the follow-up workflow uses for the `Has New Message` condition.

### Email writer prompt overhauled
**Node:** `Basic LLM Chain1`
**Changes:**
- Switched from "senior real estate assistant writing on behalf of a CEO" to "friendly real estate agent responding directly"
- Enforced first-person plural throughout — always "we", "our", "us", never "I" or "my"
- Added first name only greeting — "Hi [First Name]!" — never "Dear"
- Added realism guardrail against pressure tactics: banned "properties move fast", "before it disappears", "don't miss out"
- Added guardrail against false actions: never claim to have updated search criteria or prepared listings
- Added guardrail against surveillance language: banned "we noticed", "we saw", "we observed"
- Added HTML output formatting: returns email body using `<p>` tags for consistent rendering across devices
- Banned clichés: "touching base", "circling back", "as soon as possible"

---

## Follow-up Workflow Changes

### `Search records` — filter formula updated
**Node:** `Search records`
**Problem:** Original formula only searched for `Status = "New"`, missing leads with `Has New Message` or `Contacted` status. Leads who received Follow-up 1 (which sets Status to `Contacted`) were never picked up for Follow-up 2.
**Fix:** Updated formula to four conditions:
- `New` leads → Day 3 based on `Date Received`
- `New` leads → Day 7 based on `Date Received`
- `Has New Message` leads → Day 3 based on `Last Message Date`
- `Has New Message` or `Contacted` leads → Day 7 based on `Date Received` or `Last Message Date`

### `Search records` — returnAll updated
**Change:** Set limit to 100 records per run (previously undefined). Sufficient for current scale.

### `Google Gemini Chat Model1` — model updated
**Change:** Updated from `models/gemini-3.5-flash` to `models/gemini-3.1-flash-lite` to match the rest of the system and reduce API cost.

### `Has New Message` added to Update record schema
**Nodes:** `Update record`, `Update record1`
**Change:** Added `Has New Message` as a recognized Status option in both update nodes so n8n doesn't throw a type validation error when reading leads with that status.

### Day 3 follow-up prompt overhauled
**Node:** `Basic LLM Chain`
**Changes:**
- Enforced first-person plural — always "we", "our", "us"
- First name only in greeting
- Explicitly acknowledges previous unanswered email without being pushy
- Banned clichés: "touching base", "circling back", "reaching out again", "as soon as possible"
- Banned pressure tactics and false promises
- Banned surveillance language: "we noticed", "we saw"
- Suggests specific timeframe ("this week") instead of vague urgency
- HTML output with `<p>` tags

### Day 7 follow-up prompt overhauled
**Node:** `Basic LLM Chain1`
**Changes:**
- Same guardrails as Day 3
- Context updated to reflect this is the final message after two previous unanswered emails
- Banned counting attempts: no "we've followed up a couple of times" or similar
- Banned referencing or comparing previous search criteria the lead may have had
- Example rewritten to demonstrate tone and length only — LLM instructed not to copy its structure

---

## New Workflow — Reply Detection

### Overview
Detects when a lead replies to any automated email, updates their Airtable status to `Responded`, and alerts the team via Slack. Effectively stops the follow-up sequence from firing on leads who have already replied.

### Flow
```
Gmail Trigger → Search records (Airtable) → If (lead found?)
  ├── [true] Update record (Status → Responded) → Slack (sys-leads)
  └── [false] No Operation
```

### Nodes

**Gmail Trigger**
- Simplify: off (required to access full email payload including `from.value[0].address` and `text`)
- Filter: Search = `subject:Re:` to catch replies only
- Poll interval: every minute

**Search records**
- Searches Leads table by sender email using `$json.from.value[0].address` (clean address, no name wrapper)
- returnAll: false

**If**
- Checks `$('Search records').item.json.id` is not empty

**Update record**
- Sets Status to `Responded`

**Slack — sys-leads**
- Sends alert with lead name, email, category, and reply snippet
- Snippet cleaned with `.split('\n\n>')[0]` to remove Gmail quoted thread
- n8n attribution footer disabled

**Error Workflow**
- Linked to `Meridian Error Handler`

---

## Known Limitations (Not Fixed — By Design)

| Item | Reason |
|---|---|
| Follow-up sequence uses exact day matching (3 or 7) not ≥ | Intentional — late follow-ups would be confusing and unprofessional |
| Reply detection uses subject filter `Re:` not a label | Acceptable for portfolio/testing; use Gmail labels on a business account in production |
| Gmail trigger snippet cuts off at quoted thread | Full body available via `$json.text` but snippet is sufficient for Slack alert context |
| Reply detection does not send a response email | Human in the loop takes over after a lead replies — automation hands off via Slack alert |
