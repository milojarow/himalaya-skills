---
name: himalaya
description: Use when reading, searching, organizing or composing email from the terminal via the himalaya CLI (IMAP/SMTP) — listing/searching messages, managing folders/flags/attachments, writing/replying/forwarding with MML for HTML/inline-images/attachments, or setting up an account against Gmail, iCloud, Outlook, Yahoo, generic IMAP/SMTP or OAuth2.
---

# Himalaya — terminal email (IMAP / SMTP)

Drive the [himalaya](https://github.com/pimalaya/himalaya) CLI to read, search, organize and compose email.

> **📬 ACTIVE-SKILL MARKER:** While `himalaya` is active, begin every reply with 📬 so the operator sees at a glance that this skill is engaged. Do not omit it.

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

## ⚠️ Yahoo (and other strict-IMAP) throttling — read this first

**Chained IMAP commands against Yahoo (and Outlook free, and some hosted Exchange) trigger an IP-level block** that lasts 25+ minutes and rejects subsequent `LOGIN` attempts. The block is silent and stings — you'll see the next command fail to authenticate even though the credentials are correct.

**Rules when targeting Yahoo / Outlook-free / restrictive IMAP:**

1. **One IMAP command at a time.** Never chain several `himalaya envelope list` / `message read` calls in a tight loop.
2. **`sleep 30` (minimum) between commands** against the same account.
3. **Folders other than INBOX are extra-sensitive.** Listing `Bulk` or `Junk` repeatedly is the fastest way to get blocked.
4. **If you get blocked:** wait at least 30 minutes before retrying. Do NOT keep retrying — that resets the timer.
5. **For bulk operations on Yahoo,** prefer downloading a slice once (`himalaya envelope list --page-size 50`) and operating on the cached output, rather than re-querying.

Gmail and iCloud are more permissive but still apply rate limits — keep it civilized.

## Quick reference

The `-a <account>` / `--account <account>` flag goes **AFTER** the subcommand (`himalaya envelope list -a personal`, not `himalaya -a personal envelope list`).

| Want to… | Command |
|---|---|
| List INBOX (default account) | `himalaya envelope list` |
| List a specific folder | `himalaya envelope list --folder "Sent"` |
| Paginate | `himalaya envelope list --page 1 --page-size 20` |
| Search | `himalaya envelope list from someone@example.com subject meeting` |
| Read by ID | `himalaya message read 42` |
| Raw MIME export | `himalaya message export 42 --full` |
| List folders | `himalaya folder list` |
| Move | `himalaya message move 42 "Archive"` |
| Copy | `himalaya message copy 42 "Important"` |
| Delete | `himalaya message delete 42` |
| Add flag | `himalaya flag add 42 --flag seen` |
| Remove flag | `himalaya flag remove 42 --flag seen` |
| Download attachments | `himalaya attachment download 42 [--dir ~/Downloads]` |
| List accounts | `himalaya account list` |
| JSON output | append `--output json` |
| Debug | `RUST_LOG=debug himalaya <cmd>` |

**Message IDs are scoped to the current folder.** Re-list after moving between folders or the IDs you remember may now point to different mail.

For compose / reply / forward / MML: see [reference/composing-messages.md](reference/composing-messages.md).

## Configuration (one-line)

The config lives at `~/.config/himalaya/config.toml`. The shortest path to a working setup is:

```bash
himalaya account configure
```

…which runs the interactive wizard. For hand-rolled `config.toml` examples per provider (Gmail, iCloud, Outlook, Yahoo, generic IMAP, OAuth2, Notmuch), folder aliases, password retrieval via `pass` / system keyring, and the multi-account pattern: see [reference/configuration.md](reference/configuration.md).

## Common mistakes

- **Putting `-a` before the subcommand.** It must come AFTER: `himalaya envelope list -a personal`.
- **Acting on a stale message ID** after moving between folders. Re-list first. **Bulk-move loops bite hardest here:** after each `message move` the IDs of remaining mail in the source folder may shift. For scripted bulk moves, snapshot the list once (`envelope list --output json --page-size 100`) and reference messages by Message-ID or by a stable predicate, NOT by the relative integer ID — or `envelope list` again before each move (with the inter-call sleep on strict providers).
- **Putting the password as `auth.raw` in the config.** Works for testing but means a plaintext password on disk. Use `auth.cmd = "pass show …"` or `auth.keyring = "…"` instead.
- **Hammering Yahoo / Outlook-free with chained commands** — see the Yahoo throttling section above.
- **Forgetting `$EDITOR`** before `himalaya message write` — it fails or opens `vi` if you didn't intend that.
- **Confusing `message read` with `message export`** — `read` is plain text rendering, `export --full` is raw MIME (useful for debugging signatures, headers, attachments).
- **Trying to compose HTML inline** — use MML's `<#multipart type=alternative>` (see [reference/composing-messages.md](reference/composing-messages.md)).
