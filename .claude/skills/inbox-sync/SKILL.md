---
name: inbox-sync
description: Recompile config/rules.json into routine-prompt.txt, the text the owner pastes into their scheduled Routine. Run after editing the sender lists, or the Routine keeps acting on a stale roster. Use when asked to sync, publish, or push config changes to the Routine.
---

# Sync the config into the Routine

The scheduled Routine runs in a fresh session with **no repo checkout**, so it cannot
read `config/rules.json` at run time. The config is compiled into the Routine's prompt
instead, and this skill is what does the compiling.

```
config/rules.json  ──compile──▶  routine-prompt.txt  ──owner pastes──▶  live
```

**The owner pastes it; you cannot publish it.** A Routine created by Claude runs in
Auto mode, where every connector call is checked by a classifier — which means a
permission prompt per Gmail call, dozens per run. The fix is that the *owner* creates the
Routine themselves, and a Routine you did not create is not yours to update. So this skill
regenerates the file and hands it over; the last step is manual by design.

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

3. **Hand it over.** Send `routine-prompt.txt` to the user with `SendUserFile` and tell
   them to paste it into their Routine's prompt field, replacing what is there.

4. **Confirm** what changed — the three sender counts, and whether dry run is on or off.
   Say plainly that the change is not live until they paste it.

## Going live

Turning off the dry run is the one change worth calling out explicitly, because it is the
moment the system starts modifying the mailbox unattended. To do it: set
`settings.dry_run` to `false` in `config/rules.json`, commit, run this skill, and have the
owner paste the result. Say plainly that the next scheduled run will archive for real.

## What this skill never does

- Never edits the Routine prompt by hand. Edit `rules.json` and recompile.
- Never adds senders to `archive` on its own — it only compiles what the owner wrote.
- Never claims the change is live. It is live when the owner has pasted it.
