---
name: inbox-sync
description: Recompile config/rules.json into the scheduled Routine's prompt and publish it with update_trigger. Run this after editing the sender lists, or the Routine will keep acting on a stale roster. Use when asked to sync, publish, or push config changes to the Routine.
---

# Sync the config into the Routine

The scheduled Routine runs in a fresh session with **no repo checkout**, so it cannot
read `config/rules.json` at run time. The config is compiled into the Routine's prompt
instead, and this skill is what does the compiling.

```
config/rules.json  ──compile──▶  Routine prompt  ──update_trigger──▶  live
```

The repo is the source of truth a human edits. The prompt is a build artifact. They drift
the moment someone edits one without the other, and a drifted Routine acts on a stale
sender list — which is how a sender the owner just protected gets archived anyway.

**Run this every time `config/rules.json` changes.**

## Steps

1. **Compile.**

   ```
   python3 scripts/compile-routine-prompt.py -o /tmp/routine-prompt.txt
   ```

   The script fails loudly if a sender appears in both `archive` and `protected`. Fix the
   config rather than working around it.

2. **Read the output** and check it looks right — particularly the three sender counts and
   the `DRY RUN IS CURRENTLY:` line. That last one is the difference between a run that
   reports and a run that acts.

3. **Find the Routine.** `list_triggers` — it is normally named "Weekly inbox sweep".
   If there is more than one, ask rather than guessing.

4. **Publish.** `update_trigger(trigger_id, prompt: <the compiled text>)`.

   Use `update_trigger`, never delete-and-recreate: recreating loses the run history and
   drops the Gmail connector, which then has to be reattached by hand in the claude.ai
   Routines UI.

5. **Confirm** to the user what changed — the counts, and whether dry run is on or off.

## Going live

Turning off the dry run is the one change worth calling out explicitly, because it is the
moment the system starts modifying the mailbox unattended. To do it: set
`settings.dry_run` to `false` in `config/rules.json`, commit, then run this skill. Say
plainly in the confirmation that the next scheduled run will archive for real.

## What this skill never does

- Never edits the Routine prompt by hand. Edit `rules.json` and recompile.
- Never adds senders to `archive` on its own — it only publishes what the owner wrote.
- Never changes the cron or the connector; it touches the prompt only.
