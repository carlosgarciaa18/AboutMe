---
name: inbox-sweep
description: Run one pass of the Gmail advertising sweep — group recent Promotions/Updates mail by sender, archive mail from senders the owner already approved, surface new senders for review, and write a run report. This is what the scheduled Routine invokes. Use when asked to sweep, triage, clean up, or run the inbox routine.
---

# Inbox sweep

One pass of the recurring mail triage. Reads `config/rules.json`, archives only what the
owner has already approved, and leaves everything else alone.

**This is the interactive path** — run in a session that has the repo checked out. The
scheduled Routine does NOT use this file: it runs with no checkout, from a self-contained
prompt compiled out of the same config by `scripts/compile-routine-prompt.py`. Keep the
two in step. If you change the procedure here, mirror it in the compiler, then run
`/inbox-sync`.

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
  "new_senders": ["..."],
  "digest_sent": true,
  "commit": "f4a52c1"
}
```

Write a human report to `reports/YYYY-MM-DD.md` containing: total scanned, total archived,
a per-sender table, anything held back and why, new senders awaiting a decision, and any
sender in `unsubscribe_approved` that has not yet been processed.

Commit both files (`config/rules.json` too, if `pending_review` changed).

Interactive runs can commit; the scheduled Routine cannot — the fired session has no push
access, and the git proxy rejects it with a 403. For scheduled runs the **digest email is
the record**: it stays in the mailbox and is searchable. Do not treat a failed push as a
failed run.

## Step 6 — Email the digest

If `digest.enabled` is true, email the round-up. This is the primary output of a scheduled
run — nobody is watching the terminal when the Routine fires at 8am on a Monday.

Skip it only when `digest.send_when` is `"changes_only"` **and** nothing was archived and
no new senders appeared. A dry run always sends, since the whole point is to show the plan.

Send with `mcp__Gmail__send_message` to `digest.to`, providing **both** `htmlBody` and
`body`. The plain-text `body` is the fallback for clients that do not render HTML — write
it as a real summary, not a "please enable HTML" stub.

**Subject** — put the outcome in the subject, so the inbox list alone is enough on a quiet
week:

```
[Inbox Sweep] Sep 8 — 71 archived, 3 new senders
[Inbox Sweep] Sep 8 — dry run, 71 would be archived
[Inbox Sweep] Sep 8 — nothing to do
[Inbox Sweep] Sep 8 — FAILED: config/rules.json missing
```

**Body** — in this order, because it is read on a phone:

1. One line: what happened. `Archived 71 threads from 12 senders.`
2. **Held back**, if any — sender, count, and which rule fired. This is the most
   important section: a repeated hold-back means a sender is misclassified.
3. **New senders** awaiting a decision, with counts and how many sightings so far.
   Say plainly that nothing was archived from them.
4. **Per-sender table**, if `digest.include_sender_table` is true.
5. **Unsubscribe queue** — anyone in `unsubscribe_approved` not yet processed, and any
   sender in `unsubscribed` still arriving 30+ days later.
6. A footer line: the window swept, whether it was a dry run, and the commit SHA.

Use inline styles only. Email clients strip `<style>` blocks, and many strip `class`
attributes. Do not rely on a dark-mode media query; pick colours that read on both, or
leave the background unset and set only text colour. Keep the table under six columns so
it does not overflow on a phone.

Lead the HTML with the same one-line summary as the subject — many clients show the first
line as the preview text.

**If sending fails**, do not fail the run. The archive work is already done and committed.
Report the send failure in the terminal summary and leave the report file in `reports/`.

## Step 7 — Report to the terminal

Whether or not the digest sent, print the same summary for anyone watching:

```
Swept 96 threads from 14 senders. Archived 71. Nothing unsubscribed.
Held back: 1 World Market (starred).
New senders (3 sightings, awaiting your call): patagonia@..., rei@...
Digest emailed to carlosgarciaa97@gmail.com.
```

If `dry_run` was on, say so in the first line and give the plan instead.

## What this skill never does

- Never unsubscribes. That is `/inbox-unsubscribe`, and it needs per-sender approval.
- Never touches `category:primary`.
- Never adds a sender to `archive` on its own.
- Never deletes mail.
- Never emails anyone but the addresses in `digest.to`. The digest is a self-addressed
  report, not correspondence — it is never sent to a sender, and never replies to mail.
