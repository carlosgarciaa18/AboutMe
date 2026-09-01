# TL;DR

A weekly Claude Routine that archives advertising out of your Gmail and emails you a
round-up. It **archives, never deletes**. It only touches senders you approved.

## Right now

| | |
|---|---|
| Routine | **Live** — Mondays 8:00am Pacific |
| Mode | **Dry run** — reports what it *would* do, changes nothing |
| Archiving | 43 senders approved |
| Unsubscribing | **0 senders** — nothing is being unsubscribed yet |

## Commands

| Command | When |
|---|---|
| `/inbox-sweep` | Run a pass right now |
| `/inbox-sync` | **After any edit to `config/rules.json`** — publishes it to the Routine |
| `/inbox-unsubscribe` | Process approved unsubscribes (one-way, always confirms) |
| `/inbox-onboard` | First-time setup, or reconfigure from scratch |

## The one rule

The Routine does **not** read `config/rules.json` at run time — it runs from a prompt
compiled out of it.

```
edit config/rules.json  →  /inbox-sync  →  paste routine-prompt.txt into the Routine
```

Skip that and the Routine keeps using the old sender list, including archiving a sender
you just protected. **Every config edit ends with a paste.**

## Create the Routine yourself

Do not let Claude create it. A Claude-created Routine runs in **Auto mode**, where a
classifier checks every connector call — a permission prompt per Gmail call, dozens per
run. One you create yourself runs silently.

When you create it, grant exactly:

```
mcp__Gmail__search_threads
mcp__Gmail__unlabel_thread
mcp__Gmail__send_message
```

## Sender buckets

Each sender sits in exactly one, in `config/rules.json`:

- **`archive`** — swept out of the inbox weekly
- **`unsubscribe_approved`** — queued for `/inbox-unsubscribe`; one-way
- **`unsubscribed`** — already done, with date and method
- **`protected`** — never touched, even if it looks like an ad
- **`keep_in_inbox`** — marketing you want to see
- **`pending_review`** — new senders awaiting your decision

New senders always land in `pending_review`. The system counts and asks; it never
promotes a sender to `archive` on its own.

## Your weekly 30 seconds

The digest lands Monday. Subject tells you the outcome:

```
[Inbox Sweep] Sep 8 — 71 archived, 3 new senders
```

Read two sections:

1. **Held back** — a sender appearing here week after week is misclassified. Usually an
   advertiser that also sends receipts. Move it to `protected`.
2. **New senders** — decide: `archive`, `keep_in_inbox`, or `protected`.

Then edit `rules.json` and run `/inbox-sync`.

## Going live

```
1. Set  "dry_run": false  in config/rules.json
2. Run  /inbox-sync
3. Next Monday it archives for real
```

## Turning on unsubscribes

```
1. Move senders into  senders.unsubscribe_approved  in config/rules.json
2. Run  /inbox-unsubscribe
3. Run  /inbox-sync
```

Expect a mix of results. Some senders publish a `mailto:` opt-out (done automatically
from your account, traceable in Sent). Some publish only an HTTPS One-Click endpoint —
you get a link to click, because that requires a POST the sandbox can't make. Some
publish nothing and you get the in-body link. Roughly half will need a click.

## Undoing a bad archive

Nothing is ever deleted, so this always works:

```
1. Find the digest email for that run — search: subject:"[Inbox Sweep]"
2. In Gmail: in:archive from:<sender> newer_than:14d
3. Select all → Move to Inbox
4. Move the sender to `protected` in rules.json → /inbox-sync
```

## Never happens

- Never touches `category:primary` — personal mail is out of scope entirely
- Never deletes or marks spam — those tools are denied outright
- Never touches a starred thread, or one you replied to
- Never unsubscribes from a sender you didn't name
- Never archives receipts, security alerts, bank, health or job-application mail —
  protected by pattern regardless of sender

## Pausing

`update_trigger` with `enabled: false`, or pause it in the Routines UI on claude.ai.
Your config is untouched; re-enabling picks up where it left off.

---

Full detail: [README.md](README.md) · [docs/SAFETY.md](docs/SAFETY.md) · [docs/ROUTINE.md](docs/ROUTINE.md)
