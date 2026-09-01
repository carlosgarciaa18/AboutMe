---
name: inbox-sweep
description: Run one pass of the Gmail advertising sweep — group recent Promotions/Updates mail by sender, archive mail from senders the owner already approved, surface new senders for review, and write a run report. This is what the scheduled Routine invokes. Use when asked to sweep, triage, clean up, or run the inbox routine.
---

# Inbox sweep

One pass of the recurring mail triage. Reads `config/rules.json`, archives only what the
owner has already approved, and leaves everything else alone.

## Before anything else

1. Read `config/protected-defaults.json`. Its `hard_stops` are absolute — user config
   cannot override them. If a message trips a hard stop, it is out of scope, full stop.
2. Read `config/rules.json`. If it does not exist, **stop** and tell the user to run
   `/inbox-onboard` first. Do not improvise a sender list.
3. Read `state/ledger.json` for what previous runs did.
4. Note `settings.dry_run`. If true, you will compute and report the whole plan but
   **call no mutating tool**. Say clearly in the report that it was a dry run.

## Step 1 — Scan

For each category in `settings.categories`, search the window:

```
category:promotions in:inbox newer_than:{scan_window_days}d
```

Use `mcp__Gmail__search_threads` with `view: "THREAD_VIEW_METADATA_ONLY"` and
`pageSize: 50`. Metadata view is much cheaper than minimal view and gives you sender,
date, labels and `sizeEstimate`, which is all the grouping needs.

**Counting caveat:** `resultCountEstimate` is only trustworthy when the whole result set
fits in one page (no `nextPageToken`). With a small `pageSize` it returns a meaningless
placeholder such as `201`. To get a real count, page with `pageSize: 50` and count the
threads you actually receive.

Page until exhausted or until you hit `max_archive_per_run`.

## Step 2 — Group and classify

Group threads by sender address (lowercased — senders vary the case, e.g.
`Noreply@bitunix.com`). Classify each sender:

| Bucket | Action |
|---|---|
| in `senders.protected`, or matches a protected pattern | skip, never report as archivable |
| in `senders.keep_in_inbox` | skip |
| in `senders.archive` | archive, subject to the per-thread checks below |
| anything else | new sender → `pending_review` |

Then, for every thread you intend to archive, verify the per-thread hard stops:

- no message in the thread is `STARRED`
- the owner has not replied (no message in the thread is in `SENT`)
- the subject matches no `protected_subject_patterns` entry
- the sender matches no `protected_sender_patterns` entry

A thread failing any check is dropped from the archive set and noted in the report under
"held back". Do not silently skip — the owner needs to see when a rule fired, because a
recurring hold-back usually means a sender belongs in `protected` rather than `archive`.

## Step 3 — Archive

Only if `dry_run` is false. Archive by removing the `INBOX` label:

```
mcp__Gmail__unlabel_thread(threadId, labelIds: ["INBOX"])
```

Never trash, never delete, never mark spam. Archiving is reversible; those are not.

Work sender by sender so a failure is isolated and the ledger stays accurate. Record the
running total and stop at `max_archive_per_run`, reporting that you hit the cap.

## Step 4 — Handle new senders

For each sender not in any list, add or increment an entry in `senders.pending_review`
with the address, display name, count this run, and cumulative `sightings`. Once
`sightings` reaches `settings.promote_after_sightings`, flag it in the report as ready
for a decision — but **never move it into `archive` yourself**. Classification is the
owner's call; the routine only counts and asks.

## Step 5 — Write state and report

Append a run record to `state/ledger.json`:

```json
{
  "run": "2026-09-08T15:00:00Z",
  "dry_run": false,
  "window_days": 7,
  "scanned": 96,
  "archived": 71,
  "held_back": [{"sender": "...", "reason": "starred"}],
  "by_sender": {"info@mail.levi.com": 6},
  "new_senders": ["..."]
}
```

Write a human report to `reports/YYYY-MM-DD.md` containing: total scanned, total archived,
a per-sender table, anything held back and why, new senders awaiting a decision, and any
sender in `unsubscribe_approved` that has not yet been processed.

Commit both files (`config/rules.json` too, if `pending_review` changed) so the state
survives the container. This repo is the system's only durable memory — an uncommitted
run effectively did not happen.

## Step 6 — Report to the owner

Lead with what changed. Keep it short:

```
Swept 96 threads from 14 senders. Archived 71. Nothing unsubscribed.
Held back: 1 World Market (starred).
New senders (3 sightings, awaiting your call): patagonia@..., rei@...
```

If `dry_run` was on, say so in the first line and give the plan instead.

## What this skill never does

- Never unsubscribes. That is `/inbox-unsubscribe`, and it needs per-sender approval.
- Never touches `category:primary`.
- Never adds a sender to `archive` on its own.
- Never deletes mail.
