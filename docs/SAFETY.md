# Safety model

This system acts on a personal mailbox on a schedule, unattended. The design assumes that
a mistake will eventually happen and tries to make every mistake cheap and reversible.

## The three-layer gate

Every action passes through, in order:

1. **Hard stops** (`config/protected-defaults.json`) — ship with the system, apply to
   every install, cannot be overridden by user config. They are compiled verbatim into the
   Routine prompt, so the unattended path enforces exactly the same rules as the
   interactive one.
2. **User rules** (`config/rules.json`) — the owner's own sender roster. Only senders
   explicitly placed in `archive` are ever archived.
3. **Tool permissions** (`.claude/settings.json`) — destructive Gmail tools are denied
   outright; mutating ones prompt.

A sender must clear all three. Adding a sender to `archive` is not enough if a hard stop
catches the thread.

## Why `send_message` is allowed rather than prompted

The Routine runs unattended, so a tool that prompts would hang the run. `send_message` is
therefore in the `allow` list — it has to be, for the digest to send at 8am on a Monday
with nobody watching.

That same tool is what `/inbox-unsubscribe` uses for `mailto:` opt-outs, and the
permission layer cannot tell the two uses apart. So **the permission layer is not the
control for unsubscribing** — the `unsubscribe_approved` list is. A sender is only ever
unsubscribed if a human put its address on that list by name.

This is a deliberate trade: an unattended digest in exchange for moving the unsubscribe
gate up into the skill. If you would rather have the prompt, move `mcp__Gmail__send_message`
to `ask` in `.claude/settings.json` and set `digest.enabled` to `false` — you will get
push notifications from the Routine instead.

Two narrower limits keep the blast radius small either way: the sweep may only send to the
addresses in `digest.to`, and `forward` and `reply` are denied and prompted respectively,
so the system cannot send mail to a third party or answer correspondence.

## Archive, never delete

The only removal action is removing the `INBOX` label. Mail stays in All Mail, stays
searchable, and can be restored by re-adding the label. `trash_message`, `trash_thread`,
`mark_message_spam` and `mark_thread_spam` are in the `deny` list — not merely unused,
but blocked.

If a run archives something it should not have, the fix is
`in:archive from:sender` and re-labelling. Nothing is lost.

## Unsubscribing is treated as a separate, harder decision

Archiving is reversible; unsubscribing is not. So they are separate skills with separate
approval lists. Being in `archive` grants no permission to unsubscribe. A sender only
enters `unsubscribe_approved` when a human puts it there by name.

Even then, the system will not POST to an RFC 8058 One-Click endpoint from the sandbox —
outbound POST is blocked by network policy, and a GET is not equivalent. HTTPS opt-outs
come back to the human as links to click. Only `mailto:` unsubscribes are automated,
because they are sent from the subscribed account itself and leave a trace in Sent.

## What is never in scope

- `category:primary` — personal correspondence is never scanned, grouped, or acted on
- any thread containing a starred message
- any thread the owner has replied to
- any sender or subject matching the protected patterns

That last set exists because real mailboxes mix transactional and marketing mail at the
same address and on the same subdomains. Receipts arrive from marketing subdomains;
recruiters and admissions officers land in Promotions; health and finance services mix
account notices with upsells.

## Failure modes worth knowing about

**A sender changes character.** A retailer that only sent ads starts sending receipts from
the same address. The subject-pattern hard stops catch most of this; a thread showing up
repeatedly in "held back" is the signal to move that sender to `protected`.

**An unsubscribe kills wanted mail.** Mixed streams are the main risk. The skill flags
them rather than deciding, and `unsubscribed` entries keep a note of what was traded away.

**A sender ignores the unsubscribe.** Common. The sweep keeps archiving them, and a
follow-up run flags anyone still sending 30 days later.

**The container is reclaimed mid-run.** Nothing is lost that matters. The sweep is
idempotent — archiving an already-archived thread is a no-op — so the next run simply
recomputes from the mailbox. Scheduled runs keep no state at all; the digest email is the
record.

**The config and the Routine drift apart.** The most likely real-world failure. The
Routine acts on the roster compiled into its prompt, not on `config/rules.json`. Edit the
config without running `/inbox-sync` and it keeps using the old list — including archiving
a sender you just moved to `protected`. After any config change, run `/inbox-sync`.

## Recovering from a bad run

1. The digest email for that run lists exactly what was archived, by sender. Search
   `subject:"[Inbox Sweep]"`. Interactive runs also leave `reports/YYYY-MM-DD.md`.
2. Search `in:archive from:<sender> newer_than:Nd` in Gmail.
3. Select all, "Move to Inbox".
4. Move the sender from `archive` to `protected` or `keep_in_inbox` in `config/rules.json`
   and commit, so the next run does not repeat it.
