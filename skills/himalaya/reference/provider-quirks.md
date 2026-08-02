# Provider quirks & IMAP gotchas

Per-provider behaviour that bites when sweeping a mailbox with himalaya. Read
this before scripting bulk reads/searches against an account you don't control.

## Yahoo Mail — aggressive anti-abuse IP block

Yahoo's IMAP has a strict anti-abuse rate limiter that escalates to a hard,
**IP-level** block. It is the most severe of the common providers.

### What triggers it

- Several IMAP commands fired in quick succession (parallel, or a tight loop).
- **The specific worst offender: chained `SEARCH`/fetch across NON-INBOX
  folders** (`Archive`, `Sent`, `Junk`, `Bulk`). This pattern trips the block
  fastest — far faster than the same volume of activity against `INBOX`.

### What the block looks like

- The server starts rejecting even `LOGIN`, returning something like
  `IMAP4rev1 Server logging out`, despite correct credentials.
- It lasts an extended window (**25+ minutes**).
- **Aggressive retries reset/extend the block** — each attempt during the
  window can push the timer further out.

### Rules when listing/reading a Yahoo account

1. **One command at a time.** Never run multiple Yahoo IMAP queries in parallel
   or in a tight loop.
2. **Sleep generously between commands** — 30s minimum, not a couple of seconds.
3. **Avoid chained `SEARCH` across non-INBOX folders** — that exact pattern is
   the trigger. Prefer `INBOX`; touch other folders sparingly with long pauses.
4. **If you get blocked, STOP and wait out the full window.** Do not retry
   aggressively.
5. **For bulk work, snapshot once and operate on the cache.** Pull a slice with
   `himalaya envelope list --json --page-size 50` (`--output json` on 1.x) and
   work from that output instead of re-querying the server.

### Severity is Yahoo-specific

This level of severity is particular to Yahoo. Own-domain IMAP, Gmail, iCloud
and Outlook do **not** block this hard — they apply softer rate limits. Keep
all providers civilized, but reserve the heavy pacing above for Yahoo.

## Bulk sweeps: leave himalaya, use ONE raw IMAP session with `FETCH`

**Structural limitation to know before any sweep:** himalaya opens **one IMAP login per command**. That's the right design for interactive use, but against a provider with an aggressive rate limiter a burst of commands — listing several folders, reading N messages, moving mail — triggers an **IP block**, not a mild throttle. Chaining `SEARCH` across non-INBOX folders is the fastest way there (see Yahoo above).

The rule:

- **A single himalaya command against INBOX is safe** (`himalaya envelope list -a <account>`). Nothing special needed.
- **For anything bulk** (sweeping several folders, reading many bodies, moving N messages) → step out of himalaya and do it in **one IMAP session**, using **`FETCH` over a sequence range** and **zero `SEARCH`**.

### Validated pattern (no block)

```python
import imaplib, email, time

M = imaplib.IMAP4_SSL(HOST, 993)
M.login(USER, APP_PASSWORD)                     # ONE login

def sweep(folder, n=70):
    typ, data = M.select(folder, readonly=True)  # readonly: doesn't mark as seen
    total = int(data[0])
    lo = max(1, total - n + 1)
    # ONE fetch over a range. No SEARCH.
    typ, msgs = M.fetch(f"{lo}:{total}", "(BODY.PEEK[HEADER.FIELDS (DATE FROM SUBJECT)])")
    ...  # filter by date client-side, not server-side

sweep("INBOX")
time.sleep(35)          # breathe BEFORE touching a non-INBOX folder
sweep("Bulk", 40)

M.logout()
```

Keys to the pattern:

- **`BODY.PEEK[...]` instead of `BODY[...]`** — `PEEK` doesn't set `\Seen`. Without it, a read-only sweep marks mail the user has never opened as read.
- **Filter by date client-side.** Pulling the last N by sequence number and discarding the old ones locally is cheaper than a `SEARCH SINCE`, and avoids the code path that trips the block.
- **~30s sleep before switching to a non-INBOX folder.**
- **For full bodies, a second session with a single `fetch("1:N", "(BODY.PEEK[])")`** — not one fetch per message. On a small mailbox (tens of messages), pulling everything at once and filtering locally is safer than being selective.
- **Reuse the app password himalaya already has configured** (its config points at the secret file) rather than provisioning new credentials.

**Sign you're already blocked:** logins from that host start failing or hanging, while the same account connects fine from a different IP.

## Other providers

- **Gmail / iCloud / Outlook** — permissive but still rate-limited. Don't
  hammer them; space out scripted loops. No known 25-minute IP block.
- **Self-hosted / own-domain IMAP** — behaviour depends on the server
  (Dovecot, etc.) and its configured limits; usually the most forgiving.
