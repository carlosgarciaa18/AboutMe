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
Run the weekly inbox sweep.

Read .claude/skills/inbox-sweep/SKILL.md and follow it exactly.

In short: read config/protected-defaults.json and config/rules.json, scan the
Promotions and Updates categories for the configured window, group by sender,
and archive only mail from senders listed in senders.archive — subject to every
hard stop in protected-defaults.json.

Do not unsubscribe from anything. Do not delete anything. Do not add senders to
the archive list on your own; new senders go to pending_review for the owner to
decide.

Write the run report to reports/, append to state/ledger.json, and commit both
so the state survives this container. Then summarise in a few lines: how many
threads were archived, anything held back and why, and any new senders awaiting
a decision.

If config/rules.json is missing, stop and say so rather than guessing.
```

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
