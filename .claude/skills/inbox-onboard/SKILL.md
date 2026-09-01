---
name: inbox-onboard
description: First-run setup for the Gmail advertising sweep. Checks prerequisites, surveys the user's mailbox, walks them through classifying their own top senders, writes config/rules.json, runs a dry sweep, and offers to create the scheduled Routine. Use when someone is installing or setting up this system for the first time, or asks to reconfigure it.
---

# Onboarding

Gets a brand-new person from "I cloned this repo" to "a Routine sweeps my inbox weekly."

Run it conversationally. The person on the other end has not read the code and should not
have to. Your job is to do the scanning and present *their* mail back to them as a short
set of decisions.

**The one rule:** you are configuring a system that will act on someone's email on a
schedule, unattended. Everything defaults to inert. They opt in, item by item.

## Step 0 — Check prerequisites

1. **Gmail connector.** Try `mcp__Gmail__list_labels`. If the tool is unavailable or
   errors, stop and tell them to connect Gmail before continuing. Everything else depends
   on it.
2. **Existing install.** If `config/rules.json` already exists, do not overwrite it. Say
   what is already configured and ask whether they want to review it, add senders, or
   start over.

## Step 1 — Explain, briefly

Before touching their mail, tell them in three or four sentences what this does:

- It scans only Promotions and Updates. Never your Primary inbox, never personal mail.
- It archives — it never deletes. Archived mail is still searchable in All Mail.
- It only archives senders **you** approve, one by one.
- Unsubscribing is separate, one-way, and never happens without you saying yes to that
  specific sender.

Then ask permission to scan.

## Step 2 — Survey their mailbox

Scan a 90-day window so the volumes are meaningful:

```
category:promotions in:inbox newer_than:90d
category:updates in:inbox newer_than:90d
```

Use `mcp__Gmail__search_threads` with `view: "THREAD_VIEW_METADATA_ONLY"`, `pageSize: 50`,
paging through. Metadata view keeps this cheap; you need sender and date, not bodies.

**Get the counts right.** `resultCountEstimate` is only meaningful when the full result
set fits in the page you asked for. Ask for `pageSize: 50` and count returned threads; if
a `nextPageToken` comes back, page again and add. A small `pageSize` returns a
placeholder number (commonly `201`) that is not a count — never report it as one.

Group by lowercased sender address and rank by volume.

## Step 3 — Separate the obvious cases yourself

Do not make them adjudicate 60 senders. Pre-sort, then confirm:

- **Clearly protected** — banks, receipts, security alerts, job applications, anything
  matching `config/protected-defaults.json`. Put these straight into `protected` and just
  tell them you did.
- **Clearly advertising** — retail blasts, daily sale mail, travel promos. Propose these
  for `archive`.
- **Genuinely ambiguous** — mixed transactional/marketing streams, anything financial,
  anything that looks like a real person, subscriptions they may rely on. These go to
  them as questions.

Watch for these traps specifically, all of which show up in real mailboxes:

- **Marketing subdomains that also carry receipts.** A retailer sends both ads and
  purchase receipts from lookalike addresses. Check the subjects before proposing.
- **Mixed streams from one address.** Genetics/health services and finance apps commonly
  mix account notices with upsells at the same address. Unsubscribing kills both — flag
  the tradeoff rather than deciding for them.
- **Named humans in Promotions.** Admissions officers, recruiters and salespeople get
  filtered into Promotions. If the local part looks like a person's name rather than a
  role, protect it and point it out.
- **Job alerts.** Usually the largest volume in a job-seeker's mailbox and usually wanted.
  Suggest throttling frequency in the platform's own settings instead of unsubscribing.

## Step 4 — Walk them through the decisions

Present the ranked list with counts. Use `AskUserQuestion` for the ambiguous ones, in
small batches — never a wall of 20 questions. For each sender the buckets are:

- **archive** — sweep it out of the inbox every run
- **unsubscribe** — archive *and* ask the sender to stop (one-way; confirm explicitly)
- **keep** — leave in the inbox
- **protect** — never touch, even if it looks like an ad

Anything they do not decide goes to `pending_review` and gets raised again next run. That
is the intended path — the roster is meant to fill in over weeks, not in one sitting.

## Step 5 — Write the config

Copy `config/rules.example.json` to `config/rules.json` and fill in their decisions.
Set `owner` to their address and `timezone` to theirs — ask if you cannot infer it.

Set up the **digest** while you are here. Ask where the weekly round-up should go, and
default to their own address. Ask whether they want one every week (`send_when: "always"`)
or only when something actually happened (`"changes_only"`). Default to `"always"` — a
silent week is otherwise indistinguishable from a broken Routine.

Mention that the digest is sent from their own account to themselves, and that a hard stop
keeps the sweep from ever archiving its own reports.

**Leave `dry_run: true`.** The first scheduled run should report, not act. Say this
explicitly so they know the first report will change nothing.

Commit the file. This repo is the system's only durable memory across sessions.

## Step 6 — Dry run

Invoke `/inbox-sweep` immediately. It reports what it *would* archive. This is the moment
mistakes surface cheaply — a protected sender in the archive list is obvious here and
expensive later.

Walk them through the output and fix anything wrong. Then ask whether to flip
`dry_run` to false.

## Step 7 — Offer the Routine

Ask for a cadence, and default to **weekly, Monday morning**. Daily is rarely worth it;
these senders arrive steadily and a weekly batch reads better as a report.

Convert their local time to UTC before writing the cron — `create_trigger` evaluates cron
in UTC, and if the conversion crosses midnight the day-of-week field shifts too. For
8am Monday US Pacific (UTC-7) that is `0 15 * * 1`.

Create it with `create_trigger`:

- `create_new_session_on_fire: true` — each run starts clean
- `initiation: "human_request"`
- `notifications: { "push": true }`
- prompt: standalone, since a fresh session has no context. See `docs/ROUTINE.md`.

**Then check the connector warning.** `create_trigger` may return a Routine that stores no
connectors, with a warning rather than an error — or reject the `connectors` parameter
outright depending on the organisation. A Routine in that state fires on schedule, has no
Gmail tools, and fails.

If that happens, **disable the Routine immediately** with
`update_trigger(enabled: false)` and tell them plainly: the Routine exists but is paused,
and they need to attach the Gmail connector in the Routines UI on claude.ai and enable it
there. Do not leave a broken Routine armed — a weekly failing push notification is worse
than no Routine.

Confirm what you created, whether it is enabled, and how to pause or delete it.

## Step 8 — Hand off

Close with the four things they need to know:

1. Their config lives in `config/rules.json` and can be edited by hand.
2. Reports land in `reports/`.
3. `/inbox-sweep` runs a pass on demand.
4. `/inbox-unsubscribe` handles unsubscribes, and always asks first.
