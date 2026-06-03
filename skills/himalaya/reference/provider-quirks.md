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
   `himalaya envelope list --output json --page-size 50` and work from that
   output instead of re-querying the server.

### Severity is Yahoo-specific

This level of severity is particular to Yahoo. Own-domain IMAP, Gmail, iCloud
and Outlook do **not** block this hard — they apply softer rate limits. Keep
all providers civilized, but reserve the heavy pacing above for Yahoo.

## Other providers

- **Gmail / iCloud / Outlook** — permissive but still rate-limited. Don't
  hammer them; space out scripted loops. No known 25-minute IP block.
- **Self-hosted / own-domain IMAP** — behaviour depends on the server
  (Dovecot, etc.) and its configured limits; usually the most forgiving.
