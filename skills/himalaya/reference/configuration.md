# Configuring himalaya v2

Config file, first valid path wins:

- `$XDG_CONFIG_HOME/himalaya/config.toml`
- `$HOME/.config/himalaya/config.toml`
- `$HOME/.himalayarc`

Override with `-c <PATH>`. Multiple `:`-separated paths deep-merge onto the first. **`HIMALAYA_CONFIG` was removed in v2.**

The upstream schema reference is [`config.sample.toml`](https://github.com/pimalaya/himalaya/blob/master/config.sample.toml); the notes below are the practical subset.

> **v1 users:** this file documents the v2 schema, which is a full rewrite of the v1 `backend.*` / `message.send.backend.*` layout. See the migration table in `SKILL.md`, or pin this plugin to `0.1.4` for the v1 docs.

## Quickstart — the wizard

```bash
himalaya > ~/.config/himalaya/config.toml
```

Bare `himalaya` (no subcommand) runs the wizard. It discovers IMAP/SMTP/JMAP settings via PACC, Thunderbird Autoconfiguration and RFC 6186 SRV, **tests the connection**, then prints a ready-to-save account to stdout (prompts go to stderr, so redirecting stdout is safe).

Re-run it to generate another account and append:

```bash
himalaya >> ~/.config/himalaya/config.toml
```

**`>` truncates.** Never point it at a config that already holds accounts you want to keep — generate to a temp file and merge instead.

The account name is derived from the domain; rename it by editing the printed `[accounts.<name>]` key. `himalaya account configure` no longer exists.

## Minimal IMAP + SMTP

```toml
[accounts.personal]
default = true

imap.server = "imaps://imap.example.com:993"
imap.sasl.plain.username = "you@example.com"
imap.sasl.plain.password.command = "pass show email/personal-imap"

smtp.server = "smtps://smtp.example.com:465"
smtp.sasl.plain.username = "you@example.com"
smtp.sasl.plain.password.command = "pass show email/personal-smtp"

mailbox.alias.inbox = "INBOX"
mailbox.alias.sent = "Sent"
mailbox.alias.drafts = "Drafts"
mailbox.alias.trash = "Trash"
```

### No `email` / `display-name` keys in v2

Both were removed — composition left the CLI, so himalaya no longer knows your address. **Consequence: `message compose` emits no `From` header unless you pass `--from`.** Either pass it per-call, or let your SMTP server stamp the sender:

```bash
himalaya message compose --from "You <you@example.com>" -t someone@example.com -s "Hi" --body "…"
```

A leftover `email = …` / `display-name = …` from a v1 config is silently ignored rather than rejected — easy to miss.

### Server URL forms

`imap.server` / `smtp.server` accept a bare authority or a full URL:

| Value | Meaning |
|---|---|
| `"example.com"` | Bare authority — treated as `imaps://` (implicit TLS) |
| `"imap.example.com:143"` | Bare authority with port, still implicit TLS |
| `"imap://example.com:143"` | Cleartext, optionally upgraded with `imap.starttls = true` |
| `"imaps://example.com:993"` | Explicit implicit-TLS |
| `"unix:///path/to/sock"` | Pre-authenticated session proxy (e.g. [sirup](https://github.com/pimalaya/sirup)); no SASL negotiated |

TLS knobs: `imap.tls.provider` (`rustls` / `native-tls`), `imap.tls.rustls.crypto` (`ring` / `aws`), `imap.tls.cert` (extra PEM root), `imap.alpn` (defaults `["imap"]`, `[]` to skip). Same keys under `smtp.*`.

## Authentication

Pick exactly one SASL mechanism: `anonymous`, `login`, `plain`, `oauthbearer`, `xoauth2`, `scram-sha-256`. Omit the whole `imap.sasl` table to skip authentication entirely.

```toml
imap.sasl.plain.username = "you@example.com"
imap.sasl.plain.password.command = "pass show email/personal"
```

### Secrets

Every password/token field takes either a literal or a shell command:

| Form | Use when |
|---|---|
| `password.command = "pass show <entry>"` | **Recommended.** Any command printing the secret to stdout. |
| `password.command = ["pass", "show", "foo"]` | Array form — avoids shell quoting problems. |
| `password.raw = "<password>"` | **Testing only.** Plaintext on disk; never commit. |

**Native keyring support was removed in v2.** Use a password-manager CLI instead:

```toml
# macOS Keychain
imap.sasl.plain.password.command = "security find-generic-password -a you@example.com -s himalaya-imap -w"

# Secret Service (GNOME/KDE)
imap.sasl.plain.password.command = "secret-tool lookup service himalaya user you@example.com"

# 1Password / Bitwarden / gopass
imap.sasl.plain.password.command = "op read 'op://Vault/Item/password'"
```

### OAuth 2.0

OAuth moved out of himalaya into an external broker — [pimalaya/ortie](https://github.com/pimalaya/ortie), `pizauth` or `oama`. himalaya just consumes the token the same way it consumes a password:

```toml
imap.sasl.oauthbearer.username = "you@example.com"
imap.sasl.oauthbearer.token.command = "ortie token get gmail"
```

`xoauth2` works the same for providers that require it. Host/port for the OAUTHBEARER GS2 header are derived from the server URL at connect time.

## Provider examples

### Gmail — IMAP with App Password

```toml
[accounts.gmail]
imap.server = "imaps://imap.gmail.com:993"
imap.sasl.plain.username = "you@gmail.com"
imap.sasl.plain.password.command = "pass show google/app-password"

smtp.server = "smtps://smtp.gmail.com:465"
smtp.sasl.plain.username = "you@gmail.com"
smtp.sasl.plain.password.command = "pass show google/app-password"

mailbox.alias.inbox = "INBOX"
mailbox.alias.sent = "[Gmail]/Sent Mail"
mailbox.alias.drafts = "[Gmail]/Drafts"
mailbox.alias.trash = "[Gmail]/Trash"
```

App Password: Account → Security → 2-Step Verification → App Passwords.

**v2 alternative:** the `gmail` REST backend (`--backend gmail`) avoids IMAP throttling entirely, but requires an OAuth 2.0 bearer token — `gmail.auth.token.raw` / `.command` — which is the only authorization Gmail's REST API accepts.

### iCloud — App-Specific Password

```toml
[accounts.icloud]
imap.server = "imaps://imap.mail.me.com:993"
imap.sasl.plain.username = "you@icloud.com"
imap.sasl.plain.password.command = "pass show icloud/app-password"

smtp.server = "smtp://smtp.mail.me.com:587"
smtp.starttls = true
smtp.sasl.plain.username = "you@icloud.com"
smtp.sasl.plain.password.command = "pass show icloud/app-password"
```

Generate at https://appleid.apple.com → Sign-In and Security → App-Specific Passwords.

### Outlook / Hotmail

```toml
[accounts.outlook]
imap.server = "imaps://outlook.office365.com:993"
imap.sasl.plain.username = "you@outlook.com"
imap.sasl.plain.password.command = "pass show microsoft/app-password"

smtp.server = "smtp://smtp.office365.com:587"
smtp.starttls = true
smtp.sasl.plain.username = "you@outlook.com"
smtp.sasl.plain.password.command = "pass show microsoft/app-password"
```

Microsoft is retiring basic auth on many tenants. If it fails, use the **`msgraph` REST backend** added in v2 (`--backend msgraph`) with an OAuth 2.0 bearer token, or `oauthbearer` over IMAP.

### Yahoo — App Password (read the throttling warning first)

```toml
[accounts.yahoo]
imap.server = "imaps://imap.mail.yahoo.com:993"
imap.sasl.plain.username = "you@yahoo.com"
imap.sasl.plain.password.command = "pass show yahoo/app-password"

smtp.server = "smtp://smtp.mail.yahoo.com:587"
smtp.starttls = true
smtp.sasl.plain.username = "you@yahoo.com"
smtp.sasl.plain.password.command = "pass show yahoo/app-password"
```

Yahoo IMAP is far stricter than the rest — read the throttling warning in `SKILL.md` and `reference/provider-quirks.md` before scripting anything against it.

### Self-hosted / generic IMAP

```toml
[accounts.self_hosted]
imap.server = "imaps://mail.your-domain.example:993"
imap.sasl.plain.username = "you@your-domain.example"
imap.sasl.plain.password.command = "pass show self-hosted/imap"

smtp.server = "smtps://mail.your-domain.example:465"
smtp.sasl.plain.username = "you@your-domain.example"
smtp.sasl.plain.password.command = "pass show self-hosted/smtp"
```

### Local mail — Maildir / M2dir

```toml
[accounts.local]
maildir.root = "~/Mail/local"
```

Notmuch and Sendmail backends were **removed in v2**. `m2dir` is the newer layout and now has CLI parity with `maildir` for messages, flags and envelopes (mailbox `rename` and message `copy`/`move` still pending upstream).

## Multiple accounts

Stack `[accounts.<name>]` blocks. Mark **one** `default = true`.

```toml
[accounts.personal]
default = true
# … imap/smtp config …

[accounts.work]
# … imap/smtp config …
```

`-a/--account` is a **global** option in v2 — it works before or after the subcommand:

```bash
himalaya envelope list -a work
himalaya -a work envelope list
himalaya account list
himalaya account check -a work     # validate config + connectivity
```

## Mailbox aliases

The v1 `[folder.alias]` block is now `[mailbox.alias]`:

```toml
[mailbox.alias]
inbox = "INBOX"
sent = "Sent"
drafts = "Drafts"
trash = "Trash"
```

Two behaviours worth knowing:

- **Alias names are case-insensitive** on both lookup and storage — `INBOX`, `Inbox` and `inbox` are the same entry.
- **The `inbox` alias is the implicit default mailbox.** Shared commands fall back to it when `-m/--mailbox` is omitted. There is no separate `default-mailbox` key.

Account-level `[accounts.<name>.mailbox.alias]` entries override same-named global `[mailbox.alias]` entries. The wizard pre-fills these from the server where the protocol exposes them (JMAP reads RFC 8621 roles; Gmail and MS Graph map their fixed system labels; IMAP pins only the reserved `INBOX`, pending `LIST RETURN (SPECIAL-USE)` support upstream).

## Listing and table rendering

```toml
# Global or per-account
downloads-dir = "~/downloads/himalaya"     # attachment download default; falls back to $TMPDIR
envelope.list.page-size = 50               # -s/--page-size still wins; hard fallback 25
envelope.list.datetime-fmt = "%F %R%:z"
envelope.list.datetime-local-tz = false    # true converts to system tz
table.preset = "││──╞═╪╡┆    ┬┴┌┐└┘"
table.arrangement = "dynamic"              # or dynamic-full-width, disabled
```

Per-column colors live under `envelope.list.table.*-color` (e.g. `id-color`, `flags-color`, `from-color`). Note `sender-color` was renamed `from-color` in v2, and the per-type `folder.list.table.*` keys collapsed into a single global/per-account `table.*` plus `mailbox.list.table.*`.

## Removed in v2 — don't carry these over

| Key | Status |
|---|---|
| `email`, `display-name` | Removed. Pass `--from` at compose time. |
| `signature`, `signature-delim` | Removed. Use `compose --signature` / `--signature-file`. |
| `[message.*]`, `[template.*]`, `[pgp.*]` | Removed. Composition lives in [`mml`](https://github.com/pimalaya/mml). |
| `backend.*`, `message.send.backend.*` | Replaced by `imap.*` / `smtp.*`. |
| `[folder.alias]` | Renamed `[mailbox.alias]`. |
| `auth.keyring` | Removed. Use `password.command` with a password-manager CLI. |
| `auth.type`, `backend.type` | Gone — the protocol table you populate selects the backend. |
| `HIMALAYA_CONFIG` env var | Removed. Use `-c`. |
| Notmuch, Sendmail backends | Removed. |

## Editor

v2 needs no `$EDITOR` — `compose`, `reply` and `forward` are flag-driven and non-interactive. For an editor-based flow, write a draft yourself and pipe it in:

```bash
$EDITOR /tmp/draft.eml && himalaya message send /tmp/draft.eml
```
