# The scheduled Routine

The Routine is a Claude scheduled trigger that fires a fresh session on a cadence and has
it run `/inbox-sweep`.

## How it is created

**The owner creates it, in the Routines UI on claude.ai.** Not Claude — see "The owner
must create the Routine, not Claude" below, which is the whole reason this page exists.

```
name:           Weekly inbox sweep
schedule:       0 15 * * 1        (Mondays 8am US Pacific — cron is UTC, see below)
prompt:         paste routine-prompt.txt
connector:      Gmail
allowed tools:  mcp__Gmail__search_threads
                mcp__Gmail__unlabel_thread
                mcp__Gmail__send_message
```

`/inbox-onboard` walks through this and generates the prompt, but hands the last step to
the owner.

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

## The prompt is compiled, not hand-written

The fired session has **no repo checkout and no push access**, so it cannot read
`config/rules.json`. The config is compiled into the prompt instead:

```
config/rules.json  --compile-routine-prompt.py-->  routine-prompt.txt  --owner pastes-->  live
```

Run `/inbox-sync` after every config change. Never hand-edit the Routine prompt — the repo
and the Routine drift apart the moment you do, and a drifted Routine acts on a stale sender
list. That is how a sender you just protected gets archived anyway.

The compiled prompt carries the full archive / protected / keep rosters, the hard stops,
the dry-run flag and the digest instructions. It is self-contained by design: no clone
step, no credentials, nothing to go wrong before the sweep starts.

### Why not have the Routine use git?

It was tried, and both halves failed in different ways:

- **Push: blocked.** `403 — not in this session's authorized repository set`. So
  `reports/` and `state/ledger.json` never reached the repo.
- **Clone: worked, but pointless.** Once the write half is gone, git is only delivering a
  config file that fits comfortably in the prompt.

There is also nothing that genuinely *needs* to persist between runs. The sweep recomputes
new senders from the mailbox every time, and the emailed digest is a durable, searchable
record. The only real loss is the `pending_review` sightings counter, which resets weekly.

Dropping git removed a whole class of failure — a wrong clone URL, a missing checkout, a
403 — in exchange for one compile step the owner runs when they edit the config.

## The owner must create the Routine, not Claude

**This is the single most important thing on this page.**

A Routine created by Claude runs in **Auto mode**, where every connector call is checked by
a classifier. In practice that means a permission prompt for every Gmail call — dozens per
run — which makes an unattended Routine useless. The claude.ai UI states it plainly on any
such Routine:

> Claude created this routine, so it runs in Auto mode — connector calls are checked by a
> classifier

A Routine the **owner** creates does not run in Auto mode and can carry explicit tool
grants, so it runs silently. Nothing about the prompt, the connector or the schedule
changes this — only who created it.

So the setup is: Claude compiles the prompt, the owner creates the Routine and pastes it.

    Routine created by:  Claude  ->  Auto mode  ->  classifier  ->  a prompt per call
    Routine created by:  owner   ->  normal     ->  tool grants ->  silent

Allowed tools to grant when creating it:

    mcp__Gmail__search_threads
    mcp__Gmail__unlabel_thread
    mcp__Gmail__send_message

That is the complete set — scan, archive, send digest — which also keeps the grant tight.

A consequence worth stating: because the Routine belongs to the owner, Claude cannot update
its prompt. `/inbox-sync` regenerates `routine-prompt.txt` and the owner re-pastes it.

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

A scheduled run has no audience — nobody is watching a terminal at 8am on a Monday, and
nothing is written to disk. The emailed digest is therefore the *only* output of the
Routine, and the prompt says so explicitly.

Push notifications stay on as well (`notifications: {push: true}`), but they are a
heads-up, not the report. If you would rather have only push, set `digest.enabled` to
`false` in `config/rules.json`.

A send failure must never fail the run. By the time the digest is sent, the archiving is
already done and committed — losing the email is an annoyance, but re-running a sweep that
already ran is worse.

## Managing it

- `list_triggers` — inspect it, including whether it is enabled and what tools it grants
- The owner edits schedule, prompt and tool grants in the Routines UI
- Pause it there, or with `update_trigger(enabled: false)`

Claude cannot change the prompt of a Routine the owner created — regenerate
`routine-prompt.txt` with `/inbox-sync` and have them paste it.

Pausing is the right move if you are travelling or want to stop the sweep temporarily.
The config and ledger are untouched, so re-enabling picks up where it left off.

## Why a fresh session per run

The alternative — binding to one long-lived session — accumulates context across runs and
eventually degrades. A fresh session driven by a compiled prompt is reproducible: the same
config produces the same behaviour whether it runs next week or in a year.
