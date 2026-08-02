# himalaya 2.x — breaking changes vs 1.x (config + CLI + JSON)

himalaya 2.0.0 broke the **config schema**, several **CLI flags**, a few
**subcommands** and the **JSON output shape**. Nothing warns you: the upgraded
binary starts fine and only fails once a command touches an account.

The diagnostic error is not self-explanatory:

```
Error: No backend matching `auto` is configured for this account
```

It means: v2 resolves the backend by looking for an `imap.*` / `smtp.*` /
`jmap.*` / `gmail.*` / `msgraph.*` / `maildir.*` section, and a v1 config has
none — only `backend.*`, which no longer means anything.

**Confirm the diagnosis fast:** `himalaya account check -a <account>` reproduces
the error against a single account.

## 1. Config schema

Transport moves from loose fields to **a URL whose SCHEME carries the TLS mode**,
and auth hangs off `sasl.plain`:

```toml
# v1 — no longer works
backend.type = "imap"
backend.host = "imap.example.com"
backend.port = 993
backend.encryption.type = "tls"
backend.login = "you@example.com"
backend.auth.type = "password"
backend.auth.cmd = "pass show email/personal-imap"

message.send.backend.type = "smtp"
message.send.backend.host = "smtp.example.com"
message.send.backend.port = 587
message.send.backend.encryption.type = "start-tls"
message.send.backend.login = "you@example.com"
message.send.backend.auth.type = "password"
message.send.backend.auth.cmd = "pass show email/personal-smtp"

# v2
imap.server = "imaps://imap.example.com:993"
imap.sasl.plain.username = "you@example.com"
imap.sasl.plain.password.command = "pass show email/personal-imap"

smtp.server = "smtp://smtp.example.com:587"
smtp.starttls = true
smtp.sasl.plain.username = "you@example.com"
smtp.sasl.plain.password.command = "pass show email/personal-smtp"
```

Scheme table — the port alone is not enough, STARTTLS needs its own flag:

| Port / TLS | v2 |
|---|---|
| IMAP 993, implicit TLS | `imaps://host:993` |
| IMAP 143, plain or STARTTLS | `imap://host:143` (+ `imap.starttls = true`) |
| SMTP 465, implicit TLS | `smtps://host:465` |
| SMTP 587, STARTTLS | `smtp://host:587` **+ `smtp.starttls = true`** |

Account-level `email`, `display-name` and `default = true` are **still valid** —
`himalaya account list` renders the DEFAULT column correctly. Don't strip them.

### ⚠️ The config rewrite is the most dangerous part of the upgrade — it doesn't crash

A CLI break at least produces an error. A **third-party reader of
`config.toml`** — a watcher, a dispatcher, anything that enumerates accounts
itself — hits the v2 shape, fails to recognise the accounts it doesn't
understand, and **carries on with the ones it does**. No exception, no log line.

In a mail watcher that means starting up `active`, printing
`Starting mail watcher for: …` with a *shorter* list than yesterday, and going
blind on the missing mailboxes without a single alarm.

Rules for anything that parses the config itself:

- **Accept both shapes** (`backend.*` for v1, `<backend>.server` for v2), or
- **Alarm when the account count drops** below the previous run / a configured
  expected count — and refuse to start rather than run degraded.
- Never treat "account not recognised" as "account not present". Log it loudly
  and make it affect the exit status ([scripting.md](scripting.md) §2).

## 2. ⚠️ v2 no longer assumes INBOX — declare the alias

In v1, omitting the folder fell back to INBOX. In v2 the command **fails**:

```
Error: Mailbox is required: pass the mailbox name or alias, or set
`mailbox.alias.inbox = "<id>"` in your configuration.
```

Fix — add to every account:

```toml
mailbox.alias.inbox = "Inbox"
```

**Mind the exact id.** Run `himalaya mailbox list -a <account>` and copy the id
verbatim — on many IMAP servers it is `Inbox`, not `INBOX`.

**Nasty trap:** passing a mailbox name that does not exist **is not an error —
it returns an empty list.** `-m INBOX` against a server whose real id is `Inbox`
exits 0 with zero envelopes, indistinguishable from a genuinely empty mailbox.
Before concluding "there's no mail", verify the name against `mailbox list` or
cross-check with a one-line `imaplib` session. This is a prime cause of silent
failure in scripts — see [scripting.md](scripting.md).

## 3. CLI flags

| v1 | v2 |
|---|---|
| `-f` / `--folder` | `-m` / `--mailbox` |
| `--output json` | `--json` |
| `--page-size N` | `-s N` (short form) |

**Flags that used to be positional now need their name.** This is the sneakiest
class of break, because the command still *looks* right:

| v1 | v2 |
|---|---|
| `flag add <id> <flag>` | `flag add --flag <flag> <id>` |
| `message move <dst> <id>` | `message move --to <dst> <id>` (+ `-f`/`--from` for the source) |
| `message copy <dst> <id>` | `message copy --to <dst> <id>` |

The failure is `error: the following required arguments were not provided` — which
inside a `try/except` is a **no-op**. See [scripting.md](scripting.md).

## 4. Subcommands that disappeared

| v1 | v2 |
|---|---|
| `template send` (stdin) | `message send` (stdin) — the `template` group **no longer exists** |
| `message export <id> --full` | `message read <id> --raw` |
| `message read --preview <id>` | `message read <id>` — **`--preview` is gone**; no documented way to read without risking the Seen flag |
| `message read --header <H>` | **gone** → `message read --raw` and parse the header yourself |
| `folder list` | `mailbox list` (alias `mbox`) |
| `folder expunge <F>` | `imap expunge <F>` — moved to the IMAP-specific API |
| `message delete <id>` | **gone** → `flag add --flag deleted <ids>` then `imap expunge <F>` |

To read without marking as seen under v2, drop to a raw IMAP session with
`BODY.PEEK[...]` (pattern in [provider-quirks.md](provider-quirks.md)).

### Per-backend API groups are new in v2

Operations that aren't part of the shared API moved under a group named after the
backend: `imap`, `jmap`, `gmail`, `msgraph`, `maildir`, `smtp`. If a v1 verb
vanished from the top level, look for it there first — `expunge` is the common
case. `himalaya imap --help` lists what a given backend exposes.

## 5. JSON shape changed — it breaks every parser

```jsonc
// v1: bare array
[ { "id": 123, "from": {"addr": "someone@example.com"}, "date": "2026-01-16 00:36+00:00", ... } ]

// v2: object keyed by `envelopes`
{ "envelopes": [ {
    "id": "1227021",                                        // now a STRING, was an int
    "from": [ {"name": "...", "email": "someone@example.com"} ],  // now a LIST of objects with `email`
    "date": "2026-01-28T06:09:02Z",                         // RFC3339 with Z; was "2026-01-16 00:36+00:00"
    "message-id": "<...>",                                  // ⭐ NEW
    "subject": "...", "to": [...], "flags": [...],
    "size": 127182, "has-attachment": false
} ] }
```

**Every list command wraps now, each under its own key** — it is not just
envelopes:

```jsonc
{ "envelopes":  [ {"id": "52", "message-id": "...", "flags": [], "subject": "..."} ] }
{ "mailboxes":  [ {"id": "INBOX", "name": "INBOX", "total": null, "unread": null} ] }
```

A `json.loads(out)` that used to iterate the list directly now iterates the
**keys of a dict** — which either raises on attribute access or silently yields
one garbage item, depending on what the loop body does with it.

Migration notes for parsers:

- Unwrap `["envelopes"]` before iterating.
- `id` is a string — stop coercing to int, and stop comparing against ints.
- `from` is a list of `{name, email}`; there is no `addr` key anymore.
- `date` is RFC3339 with a trailing `Z`; `datetime.fromisoformat` parses it from
  Python 3.11 onward.

**The upside:** `message-id` now ships inside the envelope. Any code that made an
extra fetch (`message export --full`) purely to recover the Message-ID can drop
that call entirely — one IMAP round-trip less per message. That matters a lot on
rate-limited providers.

## 6. Migration path that works

1. Back up the config: `cp config.toml config.toml.bak-pre-v2`.
2. Rewrite each account with the §1 transport format plus the §2 inbox alias.
3. `himalaya account list` → accounts should show BACKENDS `imap, smtp`.
4. `himalaya account check -a <account>` → `imap: OK` per account.
5. Do a real read against a mailbox **known** to hold messages — not INBOX,
   which may legitimately be empty and proves nothing.
6. Test sending by mailing **your own account**, never a third party.

**Rolling back:** on distros that cache packages (e.g. pacman's
`/var/cache/pacman/pkg/`), the signed previous package is usually still there if
the cache-pruning job keeps more than one version. Reinstall it from the cache
and pin it (`IgnorePkg` or the equivalent hold) until the config is migrated.

## 7. Version-check first

Because the schema, flags and JSON all differ, any script or skill run should
establish the version before assuming a syntax:

```bash
himalaya --version
```

Treat `2.x` and `1.x` as different tools.
