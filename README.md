# Inbox Sweep

A Claude Routine that keeps advertising out of your Gmail inbox.

Once a week it groups your Promotions and Updates mail by sender, archives the senders
you have approved, surfaces new ones for a decision, and **emails you a round-up of what
it did**. It archives — it never deletes. It unsubscribes only from senders you have
named, one by one.

## Install

You need a Gmail connector enabled in your Claude session.

```
git clone https://github.com/carlosgarciaa18/AboutMe
cd AboutMe
claude
```

Then run:

```
/inbox-onboard
```

The wizard scans your actual mailbox, shows you your own top senders with 90-day counts,
walks you through classifying them, writes your config, runs a dry sweep so you can see
what it *would* do, and offers to create the weekly Routine.

It takes about ten minutes and asks before touching anything.

### One manual step

The scheduled Routine can only use connectors that are stored **on the Routine itself**,
and those cannot always be attached from a Claude Code session. After onboarding creates
the Routine, open it in the **Routines UI on claude.ai**, attach the **Gmail** connector,
and enable it.

Until you do, the Routine will fire with no Gmail tools and fail. Onboarding therefore
leaves it disabled and tells you so. `/inbox-sweep` works on demand regardless — this
only affects the unattended schedule. Details in [docs/ROUTINE.md](docs/ROUTINE.md).

## Commands

| Command | What it does |
|---|---|
| `/inbox-onboard` | First-run setup. Also used to reconfigure. |
| `/inbox-sweep` | Run one pass now. This is what the Routine invokes. |
| `/inbox-unsubscribe` | Process approved unsubscribes. Always confirms first. |

## How it is laid out

```
config/
  protected-defaults.json   Hard stops. Ship with the system, apply to every install.
  rules.example.json        Template for a new install.
  rules.json                Your sender roster. Created by /inbox-onboard.
state/
  ledger.json               Append-only run history.
reports/
  YYYY-MM-DD.md             One report per run.
docs/
  SAFETY.md                 The safety model and how to undo a bad run.
  ROUTINE.md                Creating, scheduling and managing the Routine.
```

Your config and history live in the repo because the sessions that run the sweep are
ephemeral — the repo is the system's only durable memory. The sweep commits after every
run.

## The weekly digest

Every run emails a round-up to the addresses in `digest.to` (your own, by default). The
subject carries the outcome, so the inbox list alone tells you what happened:

```
[Inbox Sweep] Sep 8 — 71 archived, 3 new senders
[Inbox Sweep] Sep 8 — dry run, 71 would be archived
[Inbox Sweep] Sep 8 — nothing to do
```

The body leads with a one-line summary, then anything **held back** and which rule fired,
then **new senders** awaiting your decision, then the per-sender table and the unsubscribe
queue.

The held-back section is the one worth reading. A sender showing up there week after week
is misclassified — usually an advertiser that also sends receipts, and it belongs in
`protected` instead.

Configure it under `digest` in `config/rules.json`:

| Key | Meaning |
|---|---|
| `enabled` | Set `false` to fall back to push notifications only |
| `to` | Recipients. Defaults to you |
| `send_when` | `always`, or `changes_only` to skip quiet weeks |
| `include_sender_table` | Per-sender breakdown in the body |

The digest is sent from your account to yourself, and a hard stop keeps the sweep from
ever archiving its own reports.

## Sender buckets

Every sender sits in exactly one bucket in `config/rules.json`:

- **`archive`** — swept out of the inbox every run
- **`unsubscribe_approved`** — queued for `/inbox-unsubscribe`; one-way, needs your sign-off
- **`unsubscribed`** — already done, with the date and method
- **`protected`** — never touched, even if it looks like an ad
- **`keep_in_inbox`** — marketing you actually want to see
- **`pending_review`** — new senders the sweep noticed, waiting on your decision

New senders always land in `pending_review`. The system counts and asks; it never
promotes a sender to `archive` on its own.

## Safety

The short version:

- Only Promotions and Updates are ever scanned. Never Primary, never personal mail.
- Archiving is the only removal action. Trash and spam tools are denied outright.
- Starred threads and threads you have replied to are never touched.
- Receipts, security alerts, job applications, health and finance mail are protected by
  pattern, regardless of sender.
- Unsubscribing needs explicit per-sender approval and is never inferred from archiving.

The long version, including how to undo a bad run, is in [docs/SAFETY.md](docs/SAFETY.md).

## A note on unsubscribing

Roughly three things happen when you try to unsubscribe from a sender:

1. It publishes a `mailto:` opt-out — automated, sent from your account, traceable in Sent.
2. It publishes only an HTTPS One-Click opt-out — you get the link and click it. This is
   not automated: RFC 8058 requires a POST, outbound POST is blocked by the sandbox
   network policy, and a GET is not equivalent.
3. It publishes no `List-Unsubscribe` header at all — some platforms put the opt-out only
   in the message body, and it is often per-list rather than global. You get that link.

So expect a mix of "done" and "here are four links to click." That is the honest ceiling,
not a limitation of the setup.
