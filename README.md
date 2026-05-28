# himalaya-skills

**Drive [himalaya](https://github.com/pimalaya/himalaya), the terminal email client, from Claude Code.**

## What is this?

A Claude Code marketplace with one skill, `himalaya`. It teaches Claude how to use the `himalaya` CLI to read, search, organize and compose email over IMAP/SMTP — including the MML syntax for HTML, attachments and inline images — and how to install + configure himalaya on a fresh box if it's missing.

### Why this skill exists

- **Yahoo (and other strict-IMAP) servers will hard-block your IP** if you fire chained IMAP commands at non-INBOX folders. The block rejects subsequent LOGIN attempts and persists 25+ minutes. The skill encodes the "1 IMAP command at a time, sleep between calls" discipline as a top-of-page warning.
- **MML is a tiny XML-ish syntax that compiles to MIME** — once you know the three multipart types (`alternative`/`mixed`/`related`) you can compose HTML mail, attachments and inline images from the shell. The reference file has copy-pasteable templates.
- **The `--account` (`-a`) flag goes AFTER the subcommand**, not before. Easy to get wrong; the skill spells it out.
- **Message IDs are scoped to the current folder** — re-list after moving between folders or you'll act on the wrong message.
- **Passwords come from a `cmd` (or system keyring), not raw text** in the config. The skill nudges toward `pass` / keyring even though `raw` is technically allowed.

## The skill

| Skill | Description |
|-------|-------------|
| **himalaya** | Read, search, organize and compose email via the himalaya CLI (IMAP/SMTP). Covers install, multi-provider configuration (Gmail/iCloud/Outlook/Yahoo/OAuth2), folder/flag/attachment operations, and MML composition. |

## Installation

Add this marketplace in Claude Code:

```
/plugin → Marketplaces → Add Marketplace → milojarow/himalaya-skills
```

Then install:

```
/plugin → Discover → himalaya-skills → Install
```

## Requirements

- The **himalaya** CLI on `$PATH` (see `skills/himalaya/reference/installation.md` for Linux / macOS / Windows install paths). The skill explains how to install it if missing.
- An IMAP/SMTP account (any provider). For Gmail, iCloud, Outlook, Yahoo: an app-specific password is usually required.
- A text editor (`$EDITOR`) for the interactive compose flow.

## License

MIT
