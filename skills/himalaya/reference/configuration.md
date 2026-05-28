# Configuring himalaya

Config file: `~/.config/himalaya/config.toml`.

## Quickstart — interactive wizard

```bash
himalaya account configure
```

The wizard walks you through one account and writes the file. For everything beyond the most basic case (multi-account, OAuth2, folder aliases, custom auth), hand-edit the file with the patterns below.

## Minimal IMAP + SMTP

```toml
[accounts.personal]
email = "you@example.com"
display-name = "Your Name"
default = true

# IMAP — receive
backend.type = "imap"
backend.host = "imap.example.com"
backend.port = 993
backend.encryption.type = "tls"
backend.login = "you@example.com"
backend.auth.type = "password"
backend.auth.cmd = "pass show email/personal-imap"

# SMTP — send
message.send.backend.type = "smtp"
message.send.backend.host = "smtp.example.com"
message.send.backend.port = 587
message.send.backend.encryption.type = "start-tls"
message.send.backend.login = "you@example.com"
message.send.backend.auth.type = "password"
message.send.backend.auth.cmd = "pass show email/personal-smtp"
```

## Password options

| Option | Use when |
|---|---|
| `auth.cmd = "pass show <entry>"` | **Recommended** — `pass` is the standard UNIX password store. |
| `auth.cmd = "security find-generic-password -a <user> -s <service> -w"` | macOS Keychain on the CLI. |
| `auth.keyring = "<entry-name>"` | System keyring (Secret Service / KWallet / etc.). After this, run `himalaya account configure <account>` to store the password into the keyring. |
| `auth.raw = "<password>"` | **Testing only.** Plaintext on disk; never commit. |

The `cmd` form runs any shell command that prints the password to stdout — so you can also use 1Password CLI (`op read "op://Vault/Item/password"`), Bitwarden CLI, `gopass`, etc.

## Provider examples

### Gmail (with App Password)

Gmail blocks ordinary password auth — you need an **App Password** (Account → Security → 2-Step Verification → App Passwords).

```toml
[accounts.gmail]
email = "you@gmail.com"
display-name = "Your Name"

backend.type = "imap"
backend.host = "imap.gmail.com"
backend.port = 993
backend.encryption.type = "tls"
backend.login = "you@gmail.com"
backend.auth.type = "password"
backend.auth.cmd = "pass show google/app-password"

message.send.backend.type = "smtp"
message.send.backend.host = "smtp.gmail.com"
message.send.backend.port = 587
message.send.backend.encryption.type = "start-tls"
message.send.backend.login = "you@gmail.com"
message.send.backend.auth.type = "password"
message.send.backend.auth.cmd = "pass show google/app-password"
```

### iCloud (with App-Specific Password)

Generate the password at https://appleid.apple.com → Sign-In and Security → App-Specific Passwords.

```toml
[accounts.icloud]
email = "you@icloud.com"
display-name = "Your Name"

backend.type = "imap"
backend.host = "imap.mail.me.com"
backend.port = 993
backend.encryption.type = "tls"
backend.login = "you@icloud.com"
backend.auth.type = "password"
backend.auth.cmd = "pass show icloud/app-password"

message.send.backend.type = "smtp"
message.send.backend.host = "smtp.mail.me.com"
message.send.backend.port = 587
message.send.backend.encryption.type = "start-tls"
message.send.backend.login = "you@icloud.com"
message.send.backend.auth.type = "password"
message.send.backend.auth.cmd = "pass show icloud/app-password"
```

### Outlook / Hotmail (Microsoft personal)

```toml
[accounts.outlook]
email = "you@outlook.com"
display-name = "Your Name"

backend.type = "imap"
backend.host = "outlook.office365.com"
backend.port = 993
backend.encryption.type = "tls"
backend.login = "you@outlook.com"
backend.auth.type = "password"
backend.auth.cmd = "pass show microsoft/app-password"

message.send.backend.type = "smtp"
message.send.backend.host = "smtp.office365.com"
message.send.backend.port = 587
message.send.backend.encryption.type = "start-tls"
message.send.backend.login = "you@outlook.com"
message.send.backend.auth.type = "password"
message.send.backend.auth.cmd = "pass show microsoft/app-password"
```

**Note:** Microsoft is phasing out basic auth for some Outlook tenants. If basic auth fails, use OAuth2 (see below) or `microsoft-membrane-skills` for a different angle.

### Yahoo (with App Password) — see also the throttling warning in SKILL.md

Yahoo requires an **App Password** (Account Security → Generate app password). Yahoo IMAP is also stricter than most — read the throttling warning at the top of `SKILL.md` before scripting against it.

```toml
[accounts.yahoo]
email = "you@yahoo.com"
display-name = "Your Name"

backend.type = "imap"
backend.host = "imap.mail.yahoo.com"
backend.port = 993
backend.encryption.type = "tls"
backend.login = "you@yahoo.com"
backend.auth.type = "password"
backend.auth.cmd = "pass show yahoo/app-password"

message.send.backend.type = "smtp"
message.send.backend.host = "smtp.mail.yahoo.com"
message.send.backend.port = 587
message.send.backend.encryption.type = "start-tls"
message.send.backend.login = "you@yahoo.com"
message.send.backend.auth.type = "password"
message.send.backend.auth.cmd = "pass show yahoo/app-password"
```

### Self-hosted / generic IMAP server

```toml
[accounts.self_hosted]
email = "you@your-domain.example"
display-name = "Your Name"

backend.type = "imap"
backend.host = "mail.your-domain.example"
backend.port = 993
backend.encryption.type = "tls"
backend.login = "you@your-domain.example"
backend.auth.type = "password"
backend.auth.cmd = "pass show self-hosted/imap"

message.send.backend.type = "smtp"
message.send.backend.host = "mail.your-domain.example"
message.send.backend.port = 587
message.send.backend.encryption.type = "start-tls"
message.send.backend.login = "you@your-domain.example"
message.send.backend.auth.type = "password"
message.send.backend.auth.cmd = "pass show self-hosted/smtp"
```

### OAuth2 (for providers that support it)

```toml
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

### Local mail via Notmuch

```toml
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
# … backend config …

[accounts.work]
email = "you@company.example"
# … backend config …
```

Use them with `-a <name>` (note: AFTER the subcommand):

```bash
himalaya envelope list -a work
himalaya message read -a work 42
```

List configured accounts:

```bash
himalaya account list
```

## Folder aliases

If your server uses non-standard folder names, map them:

```toml
[accounts.personal.folder.alias]
inbox = "INBOX"
sent = "Sent"
drafts = "Drafts"
trash = "Trash"
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

…controls where `himalaya attachment download` writes files when `--dir` isn't passed.

## Editor

The interactive compose / reply / forward flow uses `$EDITOR`. Set it in your shell init:

```bash
export EDITOR="vim"   # or nvim, nano, "code -w", etc.
```
