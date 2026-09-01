# Inbox Sweep

A Claude Routine that keeps advertising out of your Gmail inbox.

Once a week it groups your Promotions and Updates mail by sender, archives the senders
you have approved, surfaces new ones for a decision, and writes you a report. It archives —
it never deletes. It unsubscribes only from senders you have named, one by one.

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
