# Working in this repo

This repo is a Claude Routine that archives advertising out of a personal Gmail inbox.
It acts on someone's real mail on a schedule, unattended.

## Read these first

- `config/protected-defaults.json` — hard stops. They apply to every install and user
  config cannot override them.
- `docs/SAFETY.md` — the three-layer gate and how to undo a bad run.

## Rules for any session touching this repo

**Never widen the blast radius.** Do not add senders to `config/rules.json` `archive` on
your own, do not relax a hard stop, do not add Primary to the scanned categories, and do
not remove entries from `protected`. Those are the owner's decisions.

**Archive, never delete.** Removing the `INBOX` label is the only permitted removal.
Trash and spam tools are in the `deny` list in `.claude/settings.json`; leave them there.

**Unsubscribing needs named approval.** A sender being in `archive` is not approval to
unsubscribe from it. Only `senders.unsubscribe_approved` counts.

**Keep the config and the Routine in step.** The scheduled Routine has no checkout — its
sender lists are compiled into its prompt by `scripts/compile-routine-prompt.py`. After
editing `config/rules.json`, run `/inbox-sync` to republish, or the Routine keeps acting
on a stale roster. Never hand-edit the Routine prompt.

**The Routine cannot push.** The git proxy rejects it (403, repo not in the fired
session's authorized set), which is why the runtime path uses no git at all. For scheduled
runs the digest email is the record. Interactive runs, which do have the repo, still write
`reports/` and `state/ledger.json`.

## Gmail tool notes, learned the hard way

**Thread counts.** `resultCountEstimate` is only meaningful when the whole result set fits
in the page you requested. With `pageSize: 1` it returns a placeholder — commonly `201` —
that is not a count. To count reliably, request `pageSize: 50` and count the threads
returned; page while `nextPageToken` is present.

**Use metadata view for scanning.** `THREAD_VIEW_METADATA_ONLY` returns sender, date,
labels and `sizeEstimate` — everything grouping needs — at a fraction of the context cost
of `THREAD_VIEW_MINIMAL`, which adds subject and snippet for every thread.

**Headers require RAW.** There is no header-only message format. `messageFormat: "RAW"`
returns the entire base64url-encoded MIME message, so reading `List-Unsubscribe` is
expensive. Do it once per sender, not once per message, and pick that sender's smallest
message using `sizeEstimate` from the search results.

**Not every sender has `List-Unsubscribe`.** Verified: the San Diego Union-Tribune
publishes both a `mailto:` and an RFC 8058 One-Click HTTPS endpoint; Bandcamp publishes
neither and puts a per-artist opt-out in the body only. Handle all three cases.

**Sender case varies.** Addresses arrive with inconsistent capitalisation
(`Noreply@bitunix.com`, `CARLOSGARCIAA97@gmail.com`). Lowercase before grouping or
matching.

**Outbound POST is blocked.** Do not try to complete an HTTPS One-Click unsubscribe from
the sandbox. Hand the link to the human.

## Cron is UTC

`create_trigger` evaluates cron in UTC. Convert from local time using the offset in
effect, and shift the day-of-week field if the conversion crosses midnight. See
`docs/ROUTINE.md` for a conversion table.
