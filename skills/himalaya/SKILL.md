---
name: himalaya
description: Use when reading, searching, organizing or composing email from the terminal via the himalaya CLI v2 (IMAP/SMTP/JMAP/Gmail REST/MS Graph) — listing/searching messages, managing mailboxes/flags/attachments, writing/replying/forwarding, MML via the standalone mml tool for HTML/inline-images/attachments, raw IMAP/SMTP passthrough, or setting up an account against Gmail, iCloud, Outlook, Yahoo, generic IMAP/SMTP or OAuth2.
---

# Himalaya — terminal email (IMAP / SMTP)

Drive the [himalaya](https://github.com/pimalaya/himalaya) CLI to read, search, organize and compose email.

> **📬 ACTIVE-SKILL MARKER:** Prefix your reply with 📬 **only on turns where the work touches the `himalaya` / email-CLI domain** — listing, reading, searching, moving or flagging mail; composing/replying/forwarding or writing MML; account setup against an IMAP/SMTP/OAuth2 box. On turns that do NOT touch it (typecheck, build, git ops, unrelated edits or shell work in other domains), **omit 📬** even if the skill loaded earlier in the session. If other active skills also apply to the same turn, **stack their emojis** in the prefix.

## Overview

`himalaya` is a single-binary CLI email client. One config file (`~/.config/himalaya/config.toml`) defines one or more named accounts; everything else is subcommands on those accounts.

**This skill documents himalaya v2.x** (released 2026-07-26). v2 is a breaking release: several commands this skill previously covered (`folder`, `template`, `message delete`, `message export`, `account configure`) no longer exist. Check your version first — `himalaya --version` — and if you are on v1.x see [§v1 → v2 migration](#v1--v2-migration) at the bottom.

Backends in v2: `imap`, `jmap`, `gmail` (REST), `msgraph` (REST), `maildir`, `m2dir`, `smtp`. Notmuch and Sendmail were dropped.

## When to use

- Listing, searching, reading email from the terminal (any provider with IMAP).
- Managing mailboxes, flags, attachments.
- Writing, replying, forwarding — via the flag composer or a piped RFC 5322 message.
- Composing rich email (HTML + attachments + inline images) by piping [`mml`](https://github.com/pimalaya/mml) into `message send`.
- Raw protocol work via `imap raw` / `smtp raw` and the flat `imap` verb set.
- Setting up an account for a fresh box (Gmail, iCloud, Outlook, Yahoo, generic IMAP, OAuth2).

**Not for:** running a mail server (that's a different skill — see e.g. `posteio-skills`). Himalaya is a *client*; it talks to someone else's IMAP/SMTP.

## Prerequisites

- The `himalaya` binary on `$PATH`. Verify with `himalaya --version` — this skill assumes **v2.x**.
- A configured account (see Configuration below).
- Optionally [`mml`](https://github.com/pimalaya/mml) for rich (HTML / inline-image) composition. v2 removed the built-in MML composer; `mml` is now a standalone tool you pipe into `message send`.

No `$EDITOR` is required in v2 — `message compose` is a flag-driven composer, not an editor flow.

If himalaya isn't installed, see [reference/installation.md](reference/installation.md) for one-liners per OS.

## ⚠️ Yahoo IMAP throttling — read this first

**Chained IMAP commands against Yahoo trigger a silent, IP-level block** that lasts 25+ minutes and starts rejecting even `LOGIN` (`IMAP4rev1 Server logging out`) despite correct credentials. The fastest trigger is **chained `SEARCH`/fetch across non-INBOX folders** (`Archive`, `Sent`, `Junk`, `Bulk`). Aggressive retries reset/extend the block.

**Minimum rules for Yahoo:** one command at a time (no loops/parallel), `sleep 30`+ between commands, avoid chained searches on non-INBOX folders, and if blocked **stop and wait out the full window** — don't retry. For bulk work, snapshot once (`himalaya envelope list --json --page-size 50`) and operate on the cached output.

This severity is Yahoo-specific; Gmail, iCloud, Outlook and own-domain IMAP apply softer limits — keep them civilized but reserve the heavy pacing for Yahoo. Full detail and per-provider notes: [reference/provider-quirks.md](reference/provider-quirks.md).

**himalaya opens one IMAP login per command** — fine interactively, dangerous in bulk. For anything massive against a rate-limited provider (several folders, many bodies, N moves), step out of himalaya and use **one raw IMAP session with `FETCH` by sequence range and no `SEARCH`**. Validated pattern: [reference/provider-quirks.md](reference/provider-quirks.md) §Bulk sweeps.

## Quick reference

`-a <account>` / `--account <account>` is a **global** option in v2 — it works before or after the subcommand. Mailboxes are selected with `-m/--mailbox` (there is no `--folder`).

| Want to… | Command |
|---|---|
| List inbox (default account) | `himalaya envelope list` |
| List a specific mailbox | `himalaya envelope list -m "Sent"` |
| Paginate | `himalaya envelope list --page 1 --page-size 20` |
| Only mail with attachments | `himalaya envelope list --has-attachment` |
| Search | `himalaya envelope search from someone@example.com and subject meeting` |
| Read by ID | `himalaya message read 42` |
| Raw MIME | `himalaya message read 42 --raw` |
| List mailboxes | `himalaya mailbox list` |
| Mailboxes with counts | `himalaya mailbox list --counts` |
| Move | `himalaya message move 42 -f INBOX -t "Archive"` |
| Copy | `himalaya message copy 42 -f INBOX -t "Important"` |
| Delete | `himalaya flag add 42 -f deleted` then `himalaya imap expunge INBOX` |
| Add flag | `himalaya flag add 42 -f seen` |
| Remove flag | `himalaya flag remove 42 -f seen` |
| Replace all flags | `himalaya flag set 42 -f seen` |
| List attachments | `himalaya attachment list 42` |
| Download attachments | `himalaya attachment download 42 [--dir ~/Downloads]` |
| List accounts | `himalaya account list` |
| Validate an account | `himalaya account check -a personal` |
| JSON output | append `--json` |
| Debug | `himalaya <cmd> --log-level debug` |

**Message IDs are scoped to the current mailbox.** Re-list after moving between mailboxes or the IDs you remember may now point to different mail.

### Search query DSL

`envelope search` is its own subcommand in v2 (search is no longer positional args on `envelope list`):

- Conditions: `date <yyyy-mm-dd>`, `after <yyyy-mm-dd>`, `from <pattern>`, `to <pattern>`, `subject <pattern>`, `body <pattern>`, `flag <seen|answered|flagged|draft>`
- Combine with `and` / `or` / `not`, group with parentheses
- Sort with `order by <date|from|to|subject> [asc|desc]`

```bash
himalaya envelope search from boss@example.com and not flag seen order by date desc
himalaya envelope search "(subject invoice or subject receipt) and after 2026-01-01"
```

### Raw protocol access

v2 exposes the protocol directly — useful when the shared API has no equivalent:

```bash
himalaya imap raw "STATUS INBOX (MESSAGES UNSEEN)"
himalaya smtp raw NOOP          # verifies connect + AUTH without sending mail
himalaya imap fetch 1:50 --envelope --flags -m INBOX --json
```

`himalaya imap --help` lists the flat RFC 3501 verb set (`select`, `fetch`, `store`, `expunge`, `search`, `sort`, `thread`, `append`, …).

For compose / reply / forward / MML: see [reference/composing-messages.md](reference/composing-messages.md).

## Configuration (one-line)

The config lives at `~/.config/himalaya/config.toml`. In v2 the wizard runs on **bare `himalaya`** (no subcommand) and **prints TOML to stdout instead of writing it** — you redirect it yourself:

```bash
himalaya > ~/.config/himalaya/config.toml     # fresh setup
himalaya >> ~/.config/himalaya/config.toml    # append a second account
```

`himalaya account configure` was removed in v2. The wizard tests the IMAP connection (and then SMTP) before printing, so a bad credential fails the wizard rather than yielding a config that cannot connect. It also pre-fills `mailbox.alias.*` from the server and derives the account name from the domain — rename it by editing the printed `[accounts.<name>]` key.

**Redirect carefully:** `>` truncates. Writing straight into a config that already has accounts destroys them — generate to a temp file and merge if you are not starting clean.

For hand-rolled `config.toml` examples per provider (Gmail, iCloud, Outlook, Yahoo, generic IMAP, OAuth2), mailbox aliases, password retrieval via `pass` / system keyring, and the multi-account pattern: see [reference/configuration.md](reference/configuration.md).

## Common mistakes

- **Using v1 command names.** `folder`, `template`, `message delete`, `message export` and `account configure` are all gone in v2 and fail with `unrecognized subcommand`. See the migration table below.
- **Reaching for `--folder`.** It is `-m/--mailbox` in v2.
- **Searching via `envelope list`.** Search moved to `envelope search` with its own DSL; positional filter words on `list` are not parsed.
- **Acting on a stale message ID** after moving between mailboxes. Re-list first. **Bulk-move loops bite hardest here:** after each `message move` the IDs of remaining mail in the source mailbox may shift. For scripted bulk moves, snapshot the list once (`envelope list --json --page-size 100`) and reference messages by Message-ID or by a stable predicate, NOT by the relative integer ID — or `envelope list` again before each move (with the inter-call sleep on strict providers).
- **Putting the password as `raw` in the config.** Works for testing but means a plaintext password on disk. Use `password.command = "pass show …"` or a keyring entry instead.
- **Hammering Yahoo with chained commands** (especially searches across non-INBOX mailboxes) — see the Yahoo throttling section above.
- **Expecting `message compose` to open an editor.** v2's composer is flag-driven (`--to`, `--subject`, `--body`, `--attach`). Pipe a full RFC 5322 message into `message send` for anything richer.
- **Expecting `flag add … -f deleted` to remove mail.** It only sets `\Deleted`; the message stays until `himalaya imap expunge <MAILBOX>`. Moving to Trash is usually what you actually want.
- **`himalaya` with no subcommand no longer lists mail** — it launches the account wizard. Use `himalaya envelope list`.
- **Setting `HIMALAYA_CONFIG`** — dropped in v2. Use `-c/--config` (accepts `:`-separated paths).

## v1 → v2 migration

Quick translation for anything written against himalaya v1.x:

| v1.x | v2.x |
|---|---|
| `folder list` | `mailbox list` |
| `envelope list --folder X` | `envelope list -m X` |
| `envelope list from a subject b` | `envelope search from a and subject b` |
| `message write` | `message compose` (alias `write` retained) |
| `message export 42 --full` | `message read 42 --raw` |
| `message delete 42` | `flag add 42 -f deleted` + `imap expunge <MAILBOX>` |
| `template send < msg` | `message send < msg` |
| `account configure` | bare `himalaya` (prints TOML to stdout) |
| `--output json` | `--json` |
| `RUST_LOG=debug` | `--log-level debug` (`RUST_LOG` still honoured) |
| `HIMALAYA_CONFIG=<path>` | `-c <path>` |
| `[accounts.x] backend.host` | `[accounts.x] imap.server = "imaps://host:993"` |
| `[accounts.x.folder.alias]` | `mailbox.alias.*` |
| Notmuch / Sendmail backends | removed |

If you need the v1 docs, pin this plugin to `0.1.4`.
