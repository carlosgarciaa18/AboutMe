# The scheduled Routine

The Routine is a Claude scheduled trigger that fires a fresh session on a cadence and has
it run `/inbox-sweep`.

## How it is created

`/inbox-onboard` offers to create it at the end of setup. To create it by hand, use the
`create_trigger` tool:

```
name:  Weekly inbox sweep
cron:  0 15 * * 1
create_new_session_on_fire: true
initiation: human_request
notifications: { push: true }
```

## Cron is UTC

`create_trigger` evaluates cron in **UTC**. Convert from local time using the offset in
effect, and if the conversion crosses midnight, shift the day-of-week field too.

| Local intent | Offset | Cron (UTC) |
|---|---|---|
| Mon 8:00am US Pacific | UTC-7 (PDT) | `0 15 * * 1` |
| Mon 8:00am US Pacific | UTC-8 (PST) | `0 16 * * 1` |
| Mon 8:00am US Eastern | UTC-4 (EDT) | `0 12 * * 1` |
| Weekdays 5pm US Pacific | UTC-7 | `0 0 * * 2-6` |

Note the last row: 5pm Pacific is midnight UTC *the next day*, so the day-of-week field
shifts from `1-5` to `2-6`. This is the easiest thing to get wrong.

The offset changes with daylight saving. A Routine set in July will fire an hour off in
December. It is a weekly cleanup job, so this rarely matters — but if it does, update the
cron with `update_trigger` rather than deleting and recreating, which would lose the run
history.

## The prompt

Each firing starts a session with **no memory of any previous run**. The prompt has to be
self-contained, and all real state lives in the repo. Use this:

```
Run the weekly inbox sweep for the AboutMe repo.

Work on the branch `claude/gmail-ad-archival-unsubscribe-y4utn6` (or `main`
once it has been merged).

Read .claude/skills/inbox-sweep/SKILL.md and follow it exactly.

In short: read config/protected-defaults.json and config/rules.json, scan the
Promotions and Updates categories for the configured window, group threads by
lowercased sender, and archive only mail from senders listed in senders.archive
— subject to every hard stop in protected-defaults.json (no starred threads, no
threads the owner replied to, no protected sender or subject patterns, nothing
from the owner's own address, never category:primary).

Archive by removing the INBOX label with mcp__Gmail__unlabel_thread. Never
trash, never delete, never mark spam.

Do NOT unsubscribe from anything — that is a separate, human-approved step.
Do NOT add senders to the archive list on your own; new senders go to
senders.pending_review for the owner to decide.

Honour settings.dry_run. If it is true, compute and report the full plan but
call no mutating tool other than sending the digest, and say plainly in the
digest that it was a dry run.

Then EMAIL THE ROUND-UP. This is the main output of the run — nobody is watching
a terminal on Monday morning. Using mcp__Gmail__send_message, send to the
addresses in digest.to with both htmlBody and a real plain-text body. Put the
outcome in the subject, e.g. "[Inbox Sweep] Sep 8 — 71 archived, 3 new senders".
Cover, in this order: a one-line summary; anything held back and which rule
fired; new senders awaiting a decision; the per-sender table; the unsubscribe
queue; and a footer with the window, dry-run status and commit SHA. Inline
styles only — email clients strip style blocks. If the send fails, do not fail
the run: the archive work is already done, so just report the failure.

Two Gmail gotchas: resultCountEstimate is only meaningful when the whole result
set fits the page you requested (use pageSize 50 and count returned threads,
paging while nextPageToken is present); and use THREAD_VIEW_METADATA_ONLY for
scanning, which is far cheaper than minimal view.

Write the run report to reports/YYYY-MM-DD.md, append a record to
state/ledger.json, and commit and push both — the container is ephemeral and the
repo is this system's only durable memory, so an uncommitted run did not happen.

Then summarise in a few lines: how many threads were archived, anything held
back and why, any new senders awaiting a decision, and whether the digest sent.

If config/rules.json is missing, stop, say so, and email that as the digest
rather than guessing a sender list.
```

Keep this in sync with the live Routine. If you change the prompt, change it with
`update_trigger` rather than deleting and recreating — that preserves the run history.

## The Gmail connector has to be attached in the UI

**This is the one manual step, and the Routine does not work without it.**

A fired session only has the connectors stored on the Routine. `create_trigger` will
happily create a Routine with none, and it returns a warning rather than an error:

> this trigger stores no MCP connectors, so the sessions it fires will run without
> connector (`mcp__<server>__*`) tools

A Routine in that state fires on schedule, finds no Gmail tools, and fails — so a Routine
created from a Claude Code session should be left **disabled** until the connector is
attached.

Depending on the organisation, the `connectors` parameter may also be rejected outright:

> create_trigger: the connectors parameter is not available for this organization

To fix it, attach Gmail from the **Routines UI on claude.ai**: open the Routine, add the
Gmail connector, then enable it. After that it runs unattended.

To confirm a Routine is actually armed, check `list_triggers` for `enabled: true` and
verify the connector is listed in the UI. `next_run_at` is populated even on a disabled
Routine, so it is not evidence that the Routine will fire.

## The digest is the output

A scheduled run has no audience — nobody is watching a terminal at 8am on a Monday. The
emailed digest is therefore the real output of the Routine, and the prompt says so
explicitly.

Push notifications stay on as well (`notifications: {push: true}`), but they are a
heads-up, not the report. If you would rather have only push, set `digest.enabled` to
`false` in `config/rules.json`.

A send failure must never fail the run. By the time the digest is sent, the archiving is
already done and committed — losing the email is an annoyance, but re-running a sweep that
already ran is worse.

## Managing it

- `list_triggers` — find the trigger ID
- `update_trigger` — change cadence, prompt, or pause it with `enabled: false`
- `delete_trigger` — remove it permanently (loses run history; prefer disabling)

Pausing is the right move if you are travelling or want to stop the sweep temporarily.
The config and ledger are untouched, so re-enabling picks up where it left off.

## Why a fresh session per run

The alternative — binding to one long-lived session — accumulates context across runs and
eventually degrades. A fresh session reading committed state is reproducible: the same
config produces the same behaviour whether it is run one or fifty weeks from now.

This is also why the sweep commits. **An uncommitted run effectively did not happen** —
the container is reclaimed and the ledger goes with it.
