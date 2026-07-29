# Composing email with himalaya v2 (+ MML reference)

For reading, searching, organizing, managing flags/attachments: see `SKILL.md` and the quick-reference table there.

> **v2 changed this area the most.** The `$EDITOR` flow, `template send`, and the built-in MML composer are all gone. `message compose` is now a flag-driven composer that **prints RFC 5322 to stdout**, and rich MIME is produced by the standalone [`mml`](https://github.com/pimalaya/mml) tool piped into `message send`. If you are on v1.x, pin this plugin to `0.1.4`.

## The v2 model

Three composable pieces, each doing one thing:

| Piece | Does |
|---|---|
| `message compose` / `reply` / `forward` | Builds a message from flags. Prints it to stdout. |
| `mml` (separate binary) | Compiles MML markup into real MIME. |
| `message send` / `message add` | Consumes a raw RFC 5322 message (stdin, file path, or inline) and sends / stores it. |

Because `compose` writes to stdout and `send` reads stdin, they chain:

```bash
himalaya message compose -t you@example.com -s "Hi" --body "text" | himalaya message send
```

…or skip the pipe with `--send`.

## Compose

```bash
himalaya message compose \
  -t recipient@example.com \
  -s "Quick message" \
  --body "Message body here"
```

Prints the built message. Add `--send` to actually send it, `--save <MAILBOX>` to file a copy:

```bash
himalaya message compose -t recipient@example.com -s "Report" \
  --body "See attached." --attach ~/report.pdf \
  --send --save Sent
```

| Flag | Purpose |
|---|---|
| `--from <ADDR>` | Override the account's From. |
| `-t, --to <ADDR>` | Recipient. Repeatable. |
| `--cc <ADDR>` / `--bcc <ADDR>` | Copy / blind copy. Repeatable. |
| `-s, --subject <TEXT>` | Subject. |
| `--body <TEXT>` | Body text inline. |
| `--body-file <PATH>` | Body from a file — use for anything multi-line. |
| `--attach <PATH>` | Attach a file. Repeatable. |
| `--signature <TEXT>` / `--signature-file <PATH>` | Append a signature. |
| `--save <MAILBOX>` | Save a copy to that mailbox. |
| `--send` | Send it. Without this, the message is only printed. |

**Nothing is sent unless you pass `--send`** (or pipe into `message send`). That makes dry runs trivial — drop the flag and read the output.

## Reply and forward

Same flag set as `compose`, plus a source message ID:

```bash
himalaya message reply 42 --body "Sounds good." --send
himalaya message forward 42 -t colleague@example.com --body "FYI" --send
himalaya message reply 42 -m Archive --body "…"     # source lives in another mailbox
```

**There is no `--all` / reply-all flag in v2's shared API.** To reply to everyone, read the original's recipients and pass them explicitly:

```bash
himalaya message read 42 --json | jq -r '…'   # inspect To/Cc
himalaya message reply 42 --cc "a@x.com" --cc "b@x.com" --body "…" --send
```

## Send a raw message (automation path)

`message send` takes a complete RFC 5322 message from stdin, a file path, or an inline string:

```bash
cat <<'EOF' | himalaya message send
From: you@example.com
To: recipient@example.com
Subject: Test Message

Hello from himalaya!
EOF
```

```bash
himalaya message send ~/drafts/announcement.eml --save Sent
```

To store without sending (e.g. a draft), use `message add`:

```bash
himalaya message add -m Drafts < draft.eml
himalaya message add -m Drafts -f seen -f draft < draft.eml
himalaya message add -m Sent --send < msg.eml     # store and send
```

## MML — MIME Meta Language

MML is a tiny XML-ish syntax that compiles into proper MIME. **In v2 it is no longer built into himalaya** — install [`mml`](https://github.com/pimalaya/mml) and pipe its output into `message send` or `message add`:

```bash
mml < message.mml | himalaya message send
```

The markup below is `mml`'s input format. himalaya just transports the compiled result.

### Basic structure

Headers, blank line, body:

```
From: you@example.com
To: recipient@example.com
Subject: Hello

This is the body. No MML needed for a plain-text mail.
```

### Address formats

```
To: user@example.com
To: John Doe <john@example.com>
To: "John Doe" <john@example.com>
To: user1@example.com, user2@example.com, "Jane" <jane@example.com>
```

Common headers:

| Header | Purpose |
|---|---|
| `From` | Sender |
| `To` | Primary recipient(s) |
| `Cc` | Carbon copy |
| `Bcc` | Blind carbon copy |
| `Subject` | Subject |
| `Reply-To` | Different reply address |
| `In-Reply-To` | Message-ID being replied to |

### Plain text + HTML alternative

The receiver picks whichever they can render:

```
From: you@example.com
To: recipient@example.com
Subject: Multipart Example

<#multipart type=alternative>
This is the plain text version.
<#part type=text/html>
<html><body><h1>This is the HTML version</h1></body></html>
<#/multipart>
```

### Attachments

Single attachment:

```
From: you@example.com
To: recipient@example.com
Subject: Document

Here is the file you requested.

<#part filename=/path/to/document.pdf><#/part>
```

Attachment with a custom display name:

```
<#part filename=/path/to/file.pdf name=report-q4.pdf><#/part>
```

Multiple attachments — just stack `<#part>` tags:

```
<#part filename=/path/to/doc1.pdf><#/part>
<#part filename=/path/to/doc2.zip><#/part>
```

For plain attachments with no MIME nesting, `message compose --attach` is simpler than MML.

### Inline images (HTML email with embedded pictures)

```
From: you@example.com
To: recipient@example.com
Subject: With Inline Image

<#multipart type=related>
<#part type=text/html>
<html><body>
<p>Check out this image:</p>
<img src="cid:image1">
</body></html>
<#part disposition=inline id=image1 filename=/path/to/image.png><#/part>
<#/multipart>
```

The HTML `<img src="cid:…">` and the inline part's `id=…` must match — that's how the renderer associates them.

### Mixed content (text body + attachments)

```
From: you@example.com
To: recipient@example.com
Subject: Files attached

<#multipart type=mixed>
<#part type=text/plain>
Please find the attached files.

Best,
You
<#part filename=/path/to/file1.pdf><#/part>
<#part filename=/path/to/file2.zip><#/part>
<#/multipart>
```

### Tag reference

#### `<#multipart>`

Groups multiple parts. Close with `<#/multipart>`.

| Attribute | Purpose |
|---|---|
| `type=alternative` | Different representations of the same content (plain + HTML) — the receiver picks one. |
| `type=mixed` | Independent parts shown together (body + attachments). |
| `type=related` | Parts that reference each other (HTML body + embedded images via `cid:`). |

#### `<#part>`

A single MIME part. Close with `<#/part>` (some forms don't need a closing tag — keep it for clarity).

| Attribute | Purpose |
|---|---|
| `type=<mime-type>` | Content type (`text/html`, `application/pdf`, `image/png`, …). |
| `filename=<path>` | Absolute path to the file to attach. |
| `name=<display-name>` | What the receiver sees as the filename (defaults to the basename of `filename`). |
| `disposition=inline` | Render inline (for `type=related` images), not as a download. |
| `id=<cid>` | Content-ID — referenced from HTML via `<img src="cid:<cid>">`. |

## Tips

- **Dry-run everything**: omit `--send` and read the printed message before committing to a send.
- For automation, prefer piping a raw message into `message send` — it's deterministic and has no flag-parsing surprises.
- Inspect raw MIME of a received message with `himalaya message read <id> --raw` to learn how senders structured a multipart you want to imitate.
- HTML email with attachments + inline images: nest `<#multipart type=mixed>` containing a `<#multipart type=related>` for the HTML-with-images part, plus the attachment parts as siblings.
- `himalaya smtp raw NOOP` verifies SMTP connectivity and auth without sending anything — useful before a bulk send.
