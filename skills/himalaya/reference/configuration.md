# Configuring himalaya

Config file: `~/.config/himalaya/config.toml`.

> **⚠️ Check your version first: `himalaya --version`.** The config schema
> changed in **2.0.0** — transport is now a URL whose scheme carries the TLS
> mode, auth lives under `sasl.plain`, and INBOX is no longer implicit. The
> examples below are **v2 (2.x)**. If you are on 1.x, or you are migrating an
> existing 1.x config, read [v2-migration.md](v2-migration.md) for the full
> v1 → v2 mapping.

## Quickstart — interactive wizard

```bash
himalaya account configure
```

The wizard walks you through one account and writes the file in the schema of
whichever version you have installed. For everything beyond the most basic case
(multi-account, OAuth2, mailbox aliases, custom auth), hand-edit the file with
the patterns below.

## Minimal IMAP + SMTP (v2)

```toml
[accounts.personal]
email = "you@example.com"
display-name = "Your Name"
default = true

# v2 requires an explicit inbox alias — see the note below
mailbox.alias.inbox = "Inbox"

# IMAP — receive
imap.server = "imaps://imap.example.com:993"
imap.sasl.plain.username = "you@example.com"
imap.sasl.plain.password.command = "pass show email/personal-imap"

# SMTP — send
smtp.server = "smtp://smtp.example.com:587"
smtp.starttls = true
smtp.sasl.plain.username = "you@example.com"
smtp.sasl.plain.password.command = "pass show email/personal-smtp"
```

**Server URL scheme carries the TLS mode:**

| Port / TLS | URL |
|---|---|
| IMAP 993, implicit TLS | `imaps://host:993` |
| IMAP 143, plain or STARTTLS | `imap://host:143` (+ `imap.starttls = true`) |
| SMTP 465, implicit TLS | `smtps://host:465` |
| SMTP 587, STARTTLS | `smtp://host:587` **+ `smtp.starttls = true`** |

## ⚠️ The inbox alias is mandatory in v2

Omitting the mailbox no longer falls back to INBOX — the command fails with
`Mailbox is required: … or set 'mailbox.alias.inbox = "<id>"'`. Add
`mailbox.alias.inbox` to every account, using the id exactly as the server
reports it:

```bash
himalaya mailbox list -a <account>     # copy the id verbatim — often `Inbox`, not `INBOX`
```

A wrong mailbox name does **not** error — it returns an empty list with exit 0.
See [v2-migration.md](v2-migration.md) §2.

## Password options

| Option | Use when |
|---|---|
| `…sasl.plain.password.command = "pass show <entry>"` | **Recommended** — `pass` is the standard UNIX password store. |
| `…sasl.plain.password.command = "security find-generic-password -a <user> -s <service> -w"` | macOS Keychain on the CLI. |
| `…sasl.plain.password.command = "op read 'op://Vault/Item/password'"` | 1Password CLI; likewise Bitwarden CLI, `gopass`, etc. |

The `command` form runs any shell command that prints the password to stdout,
which covers every secret manager. **Never store the password in plaintext in
the config** — it is readable by anything running as your user and easy to
commit by accident. For system-keyring storage, run `himalaya account configure
<account>` and let the wizard write the keyring section for your version.

## Provider examples (v2 schema)

### Gmail (with App Password)

Gmail blocks ordinary password auth — you need an **App Password** (Account →
Security → 2-Step Verification → App Passwords).

```toml
[accounts.gmail]
email = "you@gmail.com"
display-name = "Your Name"
mailbox.alias.inbox = "INBOX"

imap.server = "imaps://imap.gmail.com:993"
imap.sasl.plain.username = "you@gmail.com"
imap.sasl.plain.password.command = "pass show google/app-password"

smtp.server = "smtp://smtp.gmail.com:587"
smtp.starttls = true
smtp.sasl.plain.username = "you@gmail.com"
smtp.sasl.plain.password.command = "pass show google/app-password"
```

### iCloud (with App-Specific Password)

Generate the password at https://appleid.apple.com → Sign-In and Security →
App-Specific Passwords.

```toml
[accounts.icloud]
email = "you@icloud.com"
display-name = "Your Name"
mailbox.alias.inbox = "INBOX"

imap.server = "imaps://imap.mail.me.com:993"
imap.sasl.plain.username = "you@icloud.com"
imap.sasl.plain.password.command = "pass show icloud/app-password"

smtp.server = "smtp://smtp.mail.me.com:587"
smtp.starttls = true
smtp.sasl.plain.username = "you@icloud.com"
smtp.sasl.plain.password.command = "pass show icloud/app-password"
```

### Outlook / Hotmail (Microsoft personal)

```toml
[accounts.outlook]
email = "you@outlook.com"
display-name = "Your Name"
mailbox.alias.inbox = "Inbox"

imap.server = "imaps://outlook.office365.com:993"
imap.sasl.plain.username = "you@outlook.com"
imap.sasl.plain.password.command = "pass show microsoft/app-password"

smtp.server = "smtp://smtp.office365.com:587"
smtp.starttls = true
smtp.sasl.plain.username = "you@outlook.com"
smtp.sasl.plain.password.command = "pass show microsoft/app-password"
```

**Note:** Microsoft is phasing out basic auth for some Outlook tenants. If basic
auth fails, use OAuth2 (see below) or `microsoft-membrane-skills` for a different
angle.

### Yahoo (with App Password) — see also the throttling warning in SKILL.md

Yahoo requires an **App Password** (Account Security → Generate app password).
Yahoo IMAP is also stricter than most — read the throttling warning at the top of
`SKILL.md` before scripting against it.

```toml
[accounts.yahoo]
email = "you@yahoo.com"
display-name = "Your Name"
mailbox.alias.inbox = "Inbox"

imap.server = "imaps://imap.mail.yahoo.com:993"
imap.sasl.plain.username = "you@yahoo.com"
imap.sasl.plain.password.command = "pass show yahoo/app-password"

smtp.server = "smtp://smtp.mail.yahoo.com:587"
smtp.starttls = true
smtp.sasl.plain.username = "you@yahoo.com"
smtp.sasl.plain.password.command = "pass show yahoo/app-password"
```

### Self-hosted / generic IMAP server

```toml
[accounts.self_hosted]
email = "you@your-domain.example"
display-name = "Your Name"
mailbox.alias.inbox = "Inbox"

imap.server = "imaps://mail.your-domain.example:993"
imap.sasl.plain.username = "you@your-domain.example"
imap.sasl.plain.password.command = "pass show self-hosted/imap"

smtp.server = "smtp://mail.your-domain.example:587"
smtp.starttls = true
smtp.sasl.plain.username = "you@your-domain.example"
smtp.sasl.plain.password.command = "pass show self-hosted/smtp"
```

The mailbox ids on self-hosted servers vary — always confirm with
`himalaya mailbox list -a self_hosted` rather than assuming `INBOX`.

### OAuth2 and Notmuch — verify against your version

The OAuth2 and Notmuch backends were **not re-verified against the 2.x schema**;
the shapes below are the 1.x form and are kept for reference. On 2.x, run
`himalaya account configure <account>` and let the wizard emit the correct
section, then hand-edit from there.

```toml
# 1.x form — verify before reusing on 2.x
[accounts.oauth_example]
email = "you@example.com"

backend.type = "imap"
backend.host = "imap.example.com"
backend.port = 993
backend.encryption.type = "tls"
backend.login = "you@example.com"
backend.auth.type = "oauth2"
backend.auth.client-id = "<your-client-id>"
backend.auth.client-secret.cmd = "pass show oauth/client-secret"
backend.auth.access-token.cmd = "pass show oauth/access-token"
backend.auth.refresh-token.cmd = "pass show oauth/refresh-token"
backend.auth.auth-url = "https://provider.example/oauth/authorize"
backend.auth.token-url = "https://provider.example/oauth/token"
```

```toml
# 1.x form — verify before reusing on 2.x
[accounts.local]
email = "you@localhost"

backend.type = "notmuch"
backend.db-path = "~/.mail/.notmuch"
```

## Multiple accounts

Just stack `[accounts.<name>]` blocks. Mark **one** as `default = true`.

```toml
[accounts.personal]
email = "you@example.com"
default = true
# … imap/smtp config …

[accounts.work]
email = "you@company.example"
# … imap/smtp config …
```

Use them with `-a <name>` (note: AFTER the subcommand):

```bash
himalaya envelope list -a work
himalaya message read -a work 42
```

List configured accounts:

```bash
himalaya account list          # BACKENDS column should read `imap, smtp` on v2
himalaya account check -a work # per-account connectivity: expect `imap: OK`
```

## Mailbox aliases

Map server mailbox ids to the names himalaya uses. `inbox` is required on v2;
the rest live in the same `mailbox.alias.*` namespace — confirm each id against
`himalaya mailbox list` before writing it:

```toml
[accounts.personal]
mailbox.alias.inbox = "Inbox"
mailbox.alias.sent = "Sent"
mailbox.alias.drafts = "Drafts"
mailbox.alias.trash = "Trash"
```

## Signature

```toml
[accounts.personal]
signature = "Best regards,\nYour Name"
signature-delim = "-- \n"
```

## Downloads directory

```toml
[accounts.personal]
downloads-dir = "~/Downloads/himalaya"
```

…controls where `himalaya attachment download` writes files when `--dir` isn't
passed.

## Editor

The interactive compose / reply / forward flow uses `$EDITOR`. Set it in your
shell init:

```bash
export EDITOR="vim"   # or nvim, nano, "code -w", etc.
```
