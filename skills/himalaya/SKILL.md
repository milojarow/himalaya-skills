---
name: himalaya
description: Use when reading, searching, organizing or composing email from the terminal via the himalaya CLI (IMAP/SMTP) — listing/searching messages, managing folders/flags/attachments, writing/replying/forwarding with MML for HTML/inline-images/attachments, or setting up an account against Gmail, iCloud, Outlook, Yahoo, generic IMAP/SMTP or OAuth2.
---

# Himalaya — terminal email (IMAP / SMTP)

Drive the [himalaya](https://github.com/pimalaya/himalaya) CLI to read, search, organize and compose email.

> **📬 ACTIVE-SKILL MARKER:** Prefix your reply with 📬 **only on turns where the work touches the `himalaya` / email-CLI domain** — listing, reading, searching, moving or flagging mail; composing/replying/forwarding or writing MML; account setup against an IMAP/SMTP/OAuth2 box. On turns that do NOT touch it (typecheck, build, git ops, unrelated edits or shell work in other domains), **omit 📬** even if the skill loaded earlier in the session. If other active skills also apply to the same turn, **stack their emojis** in the prefix.

## Overview

`himalaya` is a single-binary CLI email client. It speaks IMAP (read), SMTP (send), Notmuch (local) or Sendmail. One config file (`~/.config/himalaya/config.toml`) defines one or more named accounts; everything else is subcommands on those accounts.

## When to use

- Listing, searching, reading email from the terminal (any provider with IMAP).
- Managing folders, flags, attachments.
- Writing, replying, forwarding — interactively or from stdin.
- Composing rich email (HTML + attachments + inline images) via MML.
- Setting up an account for a fresh box (Gmail, iCloud, Outlook, Yahoo, generic IMAP, OAuth2).

**Not for:** running a mail server (that's a different skill — see e.g. `posteio-skills`). Himalaya is a *client*; it talks to someone else's IMAP/SMTP.

## Prerequisites

- The `himalaya` binary on `$PATH`. Verify with `himalaya --version`.
- A configured account (see Configuration below).
- A `$EDITOR` set if you'll use the interactive compose flow.

If himalaya isn't installed, see [reference/installation.md](reference/installation.md) for one-liners per OS.

## ⚠️ Check the major version before assuming any syntax

**`himalaya --version` first — 2.x and 1.x are effectively different tools.**
2.0.0 broke the config schema (transport is a URL whose scheme carries TLS, auth
under `sasl.plain`), renamed flags (`--folder` → `--mailbox`, `--output json` →
`--json`), dropped subcommands (`template`, `message export`, `--preview`), and
changed the JSON output shape (envelopes now nest under an `envelopes` key, `id`
is a string). An upgrade gives no warning: the binary starts fine and only fails
when a command touches an account, with
`No backend matching 'auto' is configured for this account`. Full v1 → v2 map and
migration path: [reference/v2-migration.md](reference/v2-migration.md).

The commands below use **2.x** syntax.

## ⚠️ Yahoo IMAP throttling — read this first

**Chained IMAP commands against Yahoo trigger a silent, IP-level block** that lasts 25+ minutes and starts rejecting even `LOGIN` (`IMAP4rev1 Server logging out`) despite correct credentials. The fastest trigger is **chained `SEARCH`/fetch across non-INBOX folders** (`Archive`, `Sent`, `Junk`, `Bulk`). Aggressive retries reset/extend the block.

**Minimum rules for Yahoo:** one command at a time (no loops/parallel), `sleep 30`+ between commands, avoid chained searches on non-INBOX folders, and if blocked **stop and wait out the full window** — don't retry. For bulk work, snapshot once (`himalaya envelope list --json --page-size 50`) and operate on the cached output.

This severity is Yahoo-specific; Gmail, iCloud, Outlook and own-domain IMAP apply softer limits — keep them civilized but reserve the heavy pacing for Yahoo. Full detail and per-provider notes: [reference/provider-quirks.md](reference/provider-quirks.md).

**himalaya opens one IMAP login per command** — fine interactively, dangerous in bulk. For anything massive against a rate-limited provider (several folders, many bodies, N moves), step out of himalaya and use **one raw IMAP session with `FETCH` by sequence range and no `SEARCH`**. Validated pattern: [reference/provider-quirks.md](reference/provider-quirks.md) §Bulk sweeps.

## Quick reference

The `-a <account>` / `--account <account>` flag goes **AFTER** the subcommand (`himalaya envelope list -a personal`, not `himalaya -a personal envelope list`).

| Want to… | Command |
|---|---|
| List inbox (default account) | `himalaya envelope list` (needs `mailbox.alias.inbox` on 2.x) |
| List a specific mailbox | `himalaya envelope list -m "Sent"` |
| Paginate | `himalaya envelope list --page 1 --page-size 20` |
| Search | `himalaya envelope list from someone@example.com subject meeting` |
| Read by ID | `himalaya message read 42` |
| Raw MIME | `himalaya message read 42 --raw` |
| List mailboxes | `himalaya mailbox list` (alias `mbox`) |
| Move | `himalaya message move --to "Archive" 42` |
| Copy | `himalaya message copy --to "Important" 42` |
| Delete | `himalaya flag add --flag deleted 42` + `himalaya imap expunge "INBOX"` |
| Add flag | `himalaya flag add --flag seen 42` |
| Remove flag | `himalaya flag remove --flag seen 42` |
| Download attachments | `himalaya attachment download 42 [--dir ~/Downloads]` |
| List accounts | `himalaya account list` |
| Check an account | `himalaya account check -a <account>` |
| JSON output | append `--json` |
| Debug | `RUST_LOG=debug himalaya <cmd>` |

On 1.x these differ: `-f`/`--folder` for `-m`, `--output json` for `--json`, `folder list` for `mailbox list`, `folder expunge` for `imap expunge`, `message export 42 --full` for `message read 42 --raw`, and `message delete 42` still exists. 1.x also took the flag/destination **positionally** (`flag add 42 seen`, `message move "Archive" 42`) — 2.x requires `--flag` / `--to`.

**`message delete` was removed in 2.x.** Deleting is now flag-then-expunge, and `imap expunge` purges **every** `\Deleted` message in that mailbox, not just yours — see [reference/provider-quirks.md](reference/provider-quirks.md).

Operations outside the shared API live under per-backend groups (`imap`, `jmap`, `gmail`, `msgraph`, `maildir`, `smtp`). If a 1.x verb disappeared, look there.

**Message IDs are scoped to the current mailbox.** Re-list after moving between mailboxes or the IDs you remember may now point to different mail.

**A mailbox name that doesn't exist returns an EMPTY LIST, not an error** (exit 0, zero envelopes) — indistinguishable from a genuinely empty mailbox. Confirm ids with `himalaya mailbox list` before concluding there's no mail.

For compose / reply / forward / MML: see [reference/composing-messages.md](reference/composing-messages.md).

## Scripting himalaya

**himalaya prints to stdout even when a command FAILS** (the `Usage:` /
`Suggestions:` help block) — a wrapper that slices `stdout` without checking the
exit code will forward the error text as if it were the message body. Always
read `returncode` first. And a wrapper that catches errors, logs them and still
exits 0 makes an outage invisible to systemd/cron monitoring: aggregate failures
into a non-zero exit. Full rules and patterns:
[reference/scripting.md](reference/scripting.md).

## Configuration (one-line)

The config lives at `~/.config/himalaya/config.toml`. The shortest path to a working setup is:

```bash
himalaya account configure
```

…which runs the interactive wizard. For hand-rolled `config.toml` examples per provider (Gmail, iCloud, Outlook, Yahoo, generic IMAP, OAuth2, Notmuch), mailbox aliases, password retrieval via `pass` / system keyring, and the multi-account pattern: see [reference/configuration.md](reference/configuration.md). On 2.x every account also needs `mailbox.alias.inbox = "<id>"` — omitting it is a hard failure, not a fallback to INBOX.

## Common mistakes

- **Putting `-a` before the subcommand.** It must come AFTER: `himalaya envelope list -a personal`.
- **Acting on a stale message ID** after moving between folders. Re-list first. **Bulk-move loops bite hardest here:** after each `message move` the IDs of remaining mail in the source folder may shift. For scripted bulk moves, snapshot the list once (`envelope list --json --page-size 100`) and reference messages by Message-ID or by a stable predicate, NOT by the relative integer ID — or `envelope list` again before each move (with the inter-call sleep on strict providers).
- **Assuming 1.x syntax on a 2.x binary** (or the reverse). Check `himalaya --version` first — see [reference/v2-migration.md](reference/v2-migration.md).
- **Storing the password in plaintext in the config.** Use a command that prints the secret (`…password.command = "pass show …"` on 2.x, `auth.cmd` on 1.x) or the system keyring instead.
- **Hammering Yahoo with chained commands** (especially searches across non-INBOX folders) — see the Yahoo throttling section above.
- **Forgetting `$EDITOR`** before `himalaya message write` — it fails or opens `vi` if you didn't intend that.
- **Confusing rendered text with raw MIME** — `message read` renders plain text; `message read --raw` (2.x) / `message export --full` (1.x) gives raw MIME, useful for debugging signatures, headers and attachments.
- **Making an extra fetch just to get the Message-ID on 2.x** — it now ships inside the envelope JSON (`message-id`). Drop the second call; it's one IMAP round-trip per message you don't need.
- **Parsing 2.x JSON as a bare array** — every list command wraps under its own key (`{"envelopes": [...]}`, `{"mailboxes": [...]}`), and `id` is a string now. Iterating the unwrapped result iterates the dict's *keys*.
- **Passing a flag or destination positionally on 2.x** — `flag add 42 seen` and `message move "Archive" 42` are 1.x. 2.x needs `--flag` / `--to`, and fails with `error: the following required arguments were not provided` — a no-op inside a bare `try/except`.
- **Grepping `message read --raw` for the first line with `@`** — `--raw` returns the full RFC 5322 message, so that line is a `Received:` header. Match the header by name and stop at the blank line. (`--header` existed on 1.x only.)
- **Comparing Message-IDs across sources without normalising** — `envelope list` gives it *without* `<>`, the raw header *with*. Strip both sides.
- **Trying to compose HTML inline** — use MML's `<#multipart type=alternative>` (see [reference/composing-messages.md](reference/composing-messages.md)).
