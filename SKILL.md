---
name: gmail-morning-briefing
description: >
  Alex's daily Gmail briefing. Runs weekdays at ~8am CET via claude.ai/code routines.
  Inbox-only (skip-inbox emails excluded by design). 3 sections: Act Now, Invoices &
  Bills (7-day window, auto-labeled when actioned), Digest. Displays full results in
  the session, sends a 1-line Slack DM with a link to the run, then stays interactive
  for drafting replies, labeling, and skill edits committed to GitHub.
---

# Gmail Morning Briefing

## Identity & Config

- **Alex's email:** `alex.ruppel@applied.co`
- **Slack user ID:** `U08C621K1J4`
- **Slack DM channel (self):** `D08B2MM7NUF`
- **Timezone:** CET (UTC+1) / CEST (UTC+2)
- **Repo:** `alex-ruppel-ai/gmail-assistant` (SKILL.md, config.json live here)

Load `config.json` at the repo root at the start of each run. Use it for
`priority_senders`, `invoice_labels`, `invoice_handled_labels`, `invoice_keywords`,
and section emojis throughout.

---

## Step 0: Time Window & Label Bootstrap

### 0a — Compute time window

```bash
python3 -c "
from datetime import datetime, timedelta
import zoneinfo
tz = zoneinfo.ZoneInfo('Europe/Berlin')
now = datetime.now(tz)
dow = now.weekday()  # 0=Mon, 5=Sat, 6=Sun
hour_min = now.hour * 60 + now.minute
is_weekend_window = (
    (dow == 5 and hour_min >= 510) or
    (dow == 6) or
    (dow == 0 and hour_min < 720)
)
if is_weekend_window:
    days_since_friday = (dow - 4) % 7
    last_friday = now - timedelta(days=days_since_friday)
    anchor = last_friday.replace(hour=8, minute=30, second=0, microsecond=0)
else:
    anchor = now - timedelta(hours=24)
# Gmail query date format
print(anchor.strftime('%Y/%m/%d'))
"
```

Store as `OLDEST_DATE` (e.g. `2026/05/12`). Also compute `OLDEST_DATE_7_DAYS` the
same way but always `now - 7 days` (used for the invoice section only).

Store the current time as `NOW_CET` (formatted: `Wed 13 May · 8:03am CET`) for the
briefing header.

### 0b — Ensure "first follow up done" label exists

Call `list_labels`. Check whether a label named `"first follow up done"` (case-insensitive)
exists. If not, call `create_label` with `name: "first follow up done"`. Store its
`id` as `HANDLED_LABEL_ID` for use in Step 5.

---

## Step 1: Parallel Email Gathering

**Launch all 3 agents simultaneously in a single Agent tool call turn.**

All inbox agents use `in:inbox` — this is the mechanism that excludes emails routed
away by Gmail "Skip Inbox" filters. Agent 2 (invoices) is the only exception.

---

### Agent 1 — Inbox: Act Now candidates

```
Search Gmail using search_threads with query: "in:inbox after:{OLDEST_DATE}"

For each thread returned, call get_thread to fetch the full message list.

For each thread, determine:
- SENDER: the From address of the most recent message
- LAST_FROM_ALEX: whether the most recent message in the thread is FROM alex.ruppel@applied.co
- ADDRESSEE: whether alex.ruppel@applied.co appears in TO or CC of any message
- HAS_QUESTION_OR_ACTION: whether the email body contains a direct question, request, or
  clear action expected of Alex (look for question marks, "please", "can you", "could you",
  "let me know", "your approval", "waiting on you", "action required", etc.)
- UNREAD: whether any message in the thread is unread (labelIds includes "UNREAD")

Classify each thread:
- ACT_NOW if: LAST_FROM_ALEX=false AND (UNREAD=true OR HAS_QUESTION_OR_ACTION=true) AND ADDRESSEE=true
- DIGEST otherwise

Return structured list with fields: thread_id, subject, sender_name, sender_email,
snippet (first ~100 chars of latest message body), classification, unread (bool),
has_question (bool), last_message_ts (Unix timestamp of most recent message).
```

---

### Agent 2 — Invoices, Bills & Orders

```
Load invoice_keywords and invoice_labels from config.json.

Run the following searches simultaneously (all calls in one turn):
1. search_threads q="{invoice_keywords joined with ' OR '} after:{OLDEST_DATE_7_DAYS}"
2. For each label ID in invoice_labels: search_threads q="label:{label_id} after:{OLDEST_DATE_7_DAYS}"

Deduplicate results by thread_id. For each thread call get_thread.

For each thread, determine:
- HAS_ALEX_REPLY: whether any message in the thread is FROM alex.ruppel@applied.co
  AND has a timestamp AFTER the first invoice/bill message in the thread
- HAS_HANDLED_LABEL: whether the thread's labelIds contains any label matching
  invoice_handled_labels from config.json (compare by label name, case-insensitive)

Exclude threads where HAS_ALEX_REPLY=true OR HAS_HANDLED_LABEL=true.

Classify remaining threads by subtype:
- "invoice" if subject or body contains: invoice, bill, due, payment due, amount due
- "order" if subject or body contains: order, shipped, delivery, tracking, arrives
- "subscription" if subject or body contains: subscription, renewal, renews, plan, charged

For each included thread return: thread_id, subject, sender_name, sender_email,
subtype, amount (extract if visible, else null), due_date (extract if visible, else null),
snippet, last_message_ts.
```

---

### Agent 3 — Digest sweep

```
Search Gmail using search_threads with query: "in:inbox after:{OLDEST_DATE}"

For each thread call get_thread.

Classify each thread:
- DIGEST for all threads

Categorize each DIGEST thread:
- "work" if sender domain is applied.co or a known work contact
- "newsletter" if sender is a mailing list, marketing platform, or body contains
  "unsubscribe" / "view in browser"
- "personal" if sender domain is gmail.com, icloud.com, or similar personal domain
- "other" for everything else

Return: thread_id, subject, sender_name, sender_email, category, snippet, last_message_ts.
```

---

## Step 2: Merge & Deduplicate

After all 3 agents return:

1. **Dedup by thread_id** across Agent 1 and Agent 3 (Agent 2 may overlap — keep invoice
   classification if a thread appears in both invoice and inbox results).

2. **Act Now final filter:** Remove any thread where `LAST_FROM_ALEX=true` (Alex's message
   is the most recent — he already replied, no new incoming message since).

3. **Act Now ordering:** Sort by `last_message_ts` descending. Move threads whose
   `sender_email` matches any entry in `priority_senders` to the top.

4. **Digest dedup:** Exclude from Digest any thread_id already in Act Now.

5. **Invoice ordering:** Sort by `last_message_ts` descending, grouped by subtype:
   invoices/bills first, then orders, then subscriptions.

6. Assign sequential item numbers across all sections: Act Now items 1…N, then
   Invoices N+1…M, then Digest M+1…Z. This lets Alex reference any item by a single
   number throughout the session.

---

## Step 3: Display Results in Session

Output the full briefing as formatted text in the session. This is what Alex sees
when he opens the run link.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
:email: GMAIL BRIEFING — {NOW_CET}
{window description} · inbox-only · {N} threads scanned
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

:email: 1. ACT NOW ({N} items)
━━━━━━━━━━━━━━━━━━━━
*1. Sender Name* <sender@example.com> — Subject line
Summary: What they're asking or what action is needed (2–3 sentences).
→ Say "reply to 1" to draft a response.

─────

*2. Sender Name* <sender@example.com> — Subject line
Summary: ...
→ Say "reply to 2" to draft a response.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
:receipt: 2. INVOICES, BILLS & ORDERS ({N} items · 7-day window)
━━━━━━━━━━━━━━━━━━━━
Invoices & Bills
  • {N+1}. Vendor Name — Invoice #1234 — $450.00 — due May 20
  • {N+2}. Vendor Name — Bill for services — $120.00

Orders & Shipping
  • {N+3}. Amazon — Order #123-456 — shipped · arrives May 15

Subscriptions
  • {N+4}. Stripe — Monthly renewal — $99.00 — charged May 12

→ Say "forward invoice {N}" or "reply to {N}" — I'll apply "first follow up done" automatically.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
:inbox_tray: 3. DIGEST ({N} items)
━━━━━━━━━━━━━━━━━━━━
Work
  • {M+1}. Sender — Subject — one-line summary

Newsletter
  • {M+2}. Sender — Subject — one-line summary

Personal
  • {M+3}. Sender — Subject — one-line summary

Other
  • {M+4}. Sender — Subject — one-line summary
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

If any section is empty, show: `Nothing here.`

---

## Step 4: Send 1-Line Slack DM

After displaying results in-session, send a single Slack DM to `D08B2MM7NUF`:

```
:email: *Gmail · {date} {time} CET* — {N_act} act now · {N_inv} invoices · {N_dig} digest | <{RUN_URL}|Open briefing →>
```

Compute `RUN_URL`:
```bash
python3 -c "
import os
url = os.environ.get('CLAUDE_SESSION_URL') or os.environ.get('CLAUDE_RUN_URL') or 'https://claude.ai/code/routines'
print(url)
"
```

---

## Step 5: Interactive Mode (always-on after briefing)

The session stays live after the briefing. Handle these commands:

### Replies & Drafts
- **`"reply to [N]"`** — Fetch thread N, draft a reply, show it to Alex for review.
  After Alex confirms: call `create_draft` with the reply body and the correct `thread_id`.
  Tell Alex: "Draft saved — open Gmail to review and send."
  **If thread N is an invoice item: also call `label_thread` with `HANDLED_LABEL_ID` after creating the draft.**

- **`"forward invoice [N] to [address]"`** — Fetch thread N, draft a forward to the specified
  address (default: `accounts-payable@applied.co` if no address given), show for review.
  After Alex confirms: call `create_draft`.
  **Automatically call `label_thread` with `HANDLED_LABEL_ID` after creating the draft.**

### Labels & Filing
- **`"label [N] as [label name]"`** — Call `label_thread` with the named label (create it first
  via `create_label` if it doesn't exist).
- **`"archive [N]"`** — Call `label_thread` to remove INBOX label (apply `label:archive` pattern).
- **`"mark [N] as done"`** / **`"done with [N]"`** — For invoice items: apply `HANDLED_LABEL_ID`.

### Any follow-up action on an invoice thread
Whenever Alex takes ANY action on a thread classified as an invoice (reply, forward, label,
archive, upload trigger), automatically apply `HANDLED_LABEL_ID` via `label_thread` unless
the label is already present. Confirm: "Marked as 'first follow up done'."

### Config edits
- **`"add [name/email] to priority senders"`** — Edit `config.json`, add to `priority_senders`.
- **`"add label [X] to invoice labels"`** — Edit `config.json`, add label name to `invoice_labels`.
- **`"add [X] to handled labels"`** — Edit `config.json`, add to `invoice_handled_labels`.

### Skill edits
- **`"update skill to [rule]"`** / any instruction to change briefing behavior — Edit `SKILL.md`
  directly with the specified change.

### Auto-commit after any config or skill edit
After any edit to `SKILL.md` or `config.json`:
```bash
git add SKILL.md config.json
git commit -m "skill: [short description of change]"
git push origin main
```
Confirm: "Committed and pushed — change is live for the next run."

---

## Edge Cases

- **Thread fetch fails:** Include in output with note "thread unreadable — check Gmail directly."
- **No Act Now items:** Section 1 shows "Nothing here — inbox is clear."
- **No invoices in 7-day window:** Section 2 shows "No pending invoices."
- **`create_label` fails (label already exists with different casing):** Use the existing label ID.
- **`create_draft` for a reply:** Always set `thread_id` so Gmail threads it correctly.
- **Slack send fails:** Print the DM text inline in the session so Alex can copy it manually.
- **Weekend / holiday run:** Time window logic in Step 0 handles Friday→Monday automatically.
  On a manually triggered run mid-day, the 24h window still applies.

---

## Priority Rules

Within Act Now, surface first:
- Senders in `priority_senders`
- External parties (non-applied.co domains) asking for something
- Threads with explicit deadlines or "urgent" / "ASAP" in subject or body

Within Digest, surface work category before newsletter/personal.

Bot / automated messages with no question or action: Digest only, or omit if clearly
pure-notification (GitHub CI pass, calendar echo, etc.).
