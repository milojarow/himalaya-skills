# Composing email with himalaya (+ MML reference)

For reading, searching, organizing, managing flags/attachments: see `SKILL.md` and the quick-reference table there.

## Compose, reply, forward

### Write a new message (interactive — opens `$EDITOR`)

```bash
himalaya message write
himalaya message write -a personal       # use a specific account
```

Save and exit the editor to send. Exit without saving to cancel.

### Prefill headers from the CLI

```bash
himalaya message write \
  -H "To:recipient@example.com" \
  -H "Subject:Quick Message" \
  "Message body here"
```

`-H` is repeatable — one per header. The trailing positional argument is the body.

### Reply

```bash
himalaya message reply 42                # reply to sender only
himalaya message reply 42 --all          # reply-all
himalaya message reply -a personal 42    # specific account
```

Reply opens an editor with the quoted original message pre-populated.

### Forward

```bash
himalaya message forward 42
```

### Send from stdin (non-interactive — for scripts)

```bash
cat <<'EOF' | himalaya template send
From: you@example.com
To: recipient@example.com
Subject: Test Message

Hello from himalaya!
EOF
```

This is the path for automation. The body is a complete RFC-822-ish message (headers, blank line, body); MML markup in the body compiles to MIME.

## MML — MIME Meta Language

MML is a tiny XML-ish syntax that himalaya compiles into proper MIME parts. You write it inside the body of a message; `himalaya template send` (and the interactive flow) handles the rest.

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

- The editor opens with a template; fill in headers and the body, save to send.
- For automation, prefer `template send` from stdin — it's deterministic and scriptable.
- Inspect raw MIME of a received message with `himalaya message export <id> --full` to learn how senders structured a multipart you want to imitate.
- HTML email with attachments + inline images: nest `<#multipart type=mixed>` containing a `<#multipart type=related>` for the HTML-with-images part, plus the attachment parts as siblings.
