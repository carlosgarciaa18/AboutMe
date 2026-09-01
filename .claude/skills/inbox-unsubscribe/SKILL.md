---
name: inbox-unsubscribe
description: Process approved unsubscribes for the Gmail sweep — extract List-Unsubscribe headers, send mailto unsubscribes, and hand back HTTPS and in-body links the human must click. One-way and gated on explicit per-sender approval. Use when asked to unsubscribe from senders, or to process the unsubscribe queue.
---

# Unsubscribe

Unsubscribing is **one-way**. You cannot undo it, and for some senders re-subscribing
means re-registering an account. This skill is therefore gated harder than the sweep.

## The gate

Only process senders listed in `senders.unsubscribe_approved` in `config/rules.json`.

If the user asks you to unsubscribe from something not on that list, confirm the specific
sender with them first, add it to the list, and then proceed. Never infer approval from
a sender merely being in `archive` — archiving is reversible and unsubscribing is not, so
approval for one is not approval for the other.

Never unsubscribe from anything in `senders.protected`, even on request, without saying
plainly what the sender also carries (receipts, security notices, account status) and
getting a second confirmation.

## Step 1 — Find the unsubscribe route

For each approved sender, pick **one** recent message and read its headers:

```
mcp__Gmail__get_message(messageId, messageFormat: "RAW")
```

`RAW` returns the entire base64url-encoded MIME message and there is no header-only
option, so this is the expensive part of the job.

**Pick the smallest message that sender has.** `sizeEstimate` comes back in search
results — sort by it and use the smallest. A 200KB retail mail and an 8KB one carry the
same headers; there is no reason to pull the big one.

Do this **once per sender**, not once per message.

Decode the base64url payload and read only the header block (everything up to the first
blank line). Look for:

```
List-Unsubscribe: <mailto:...>, <https://...>
List-Unsubscribe-Post: List-Unsubscribe=One-Click
```

## Step 2 — Route by what you find

Three real cases, in order of preference:

### A. `mailto:` present — do it

Send an empty message to the mailto address with `mcp__Gmail__send_message`. Use the
subject and body from the URI if it carries `?subject=` or `?body=` parameters, otherwise
subject `unsubscribe` and an empty body.

This is the preferred route: it comes from the subscribed account, which is exactly what
the sender's system expects, it leaves an auditable trace in Sent, and it needs no
outbound network access beyond Gmail.

### B. HTTPS One-Click only — hand it to the human

RFC 8058 One-Click needs an HTTP POST of `List-Unsubscribe=One-Click` to that URL.
**Do not attempt this from the sandbox.** Outbound POST to arbitrary hosts is blocked by
the environment's network policy, and a GET against a One-Click endpoint is not equivalent
— it may do nothing, or land on a preference page that silently does nothing.

Collect the URL and give it to the user as a link to click.

### C. No `List-Unsubscribe` header — hand it to the human

Plenty of senders have none; some platforms put the opt-out only in the message body,
often per-artist or per-list rather than global. Extract the unsubscribe link from the
decoded body and give it to the user, noting what it actually unsubscribes from.

## Step 3 — Archive the backlog

Once a sender is unsubscribed, archive their remaining mail in the window with
`mcp__Gmail__unlabel_thread(threadId, labelIds: ["INBOX"])`, subject to the same hard
stops the sweep uses.

## Step 4 — Record it

Move the sender from `unsubscribe_approved` to `unsubscribed` in `config/rules.json`,
with the date and the method used:

```json
{ "address": "news@e.example.com", "method": "mailto", "date": "2026-09-08",
  "confirmed": false, "note": "Sent to unsub@bounce.example.com" }
```

Keep them in `archive` too. Unsubscribes routinely take days to take effect, and some are
simply ignored, so the sweep should keep clearing them out meanwhile.

Commit the change. Then report:

- unsubscribed via mailto (done)
- needs a click (with the links)
- no unsubscribe route found

## Step 5 — Verify later

On a subsequent run, check whether senders in `unsubscribed` are still arriving. If a
sender is still sending 30 days after a mailto unsubscribe, say so and offer the HTTPS
link or a Gmail filter as a fallback. Set `confirmed: true` on the ones that genuinely
stopped.

## Never

- Never unsubscribe from a sender not explicitly approved by the human.
- Never POST to a One-Click endpoint from the sandbox.
- Never treat "in the archive list" as approval to unsubscribe.
- Never unsubscribe from a mixed transactional/marketing stream without flagging that
  account notices will stop too.
