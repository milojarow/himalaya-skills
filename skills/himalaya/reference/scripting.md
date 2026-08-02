# Scripting himalaya — exit codes, stdout, unattended runs

Everything below matters the moment a himalaya call stops being interactive and
starts running under a wrapper, a timer or a cron job. Interactively a broken
command is obvious — a human sees the error. Unattended, the same failure is
silent unless the script is built to make noise.

## 1. himalaya writes to stdout even when the command FAILS

On failure the error goes to stderr — but **stdout is not empty**. It carries
the help block (`Usage:`, `Suggestions: - Run with --log-level to enable more
verbose logs`, …). Any wrapper that slices `stdout` without looking at the exit
code will happily treat that help text as message content.

The broken shape — no `returncode` check:

```python
out = run(["himalaya", "message", "read", "-a", acct, str(mid)]).stdout
i = out.find("\n\n")
body = out[i+2:] if i != -1 else out          # ⚠️ may be the error help block
```

This does not crash and does not log. It forwards garbage that looks like an
email. A wrapper doing auto-forwarding will relay the CLI's own usage text to
the recipient.

The fix — read `returncode` **before** touching `stdout`, and return `None` so
the caller decides (retrying beats sending the error):

```python
r = run([...], capture_output=True, text=True)
if r.returncode != 0:
    log(f"read {mid} FAIL: {r.stderr.strip()[:160]}")
    return None
```

**Test helpers against a failing input, not by reading them.** Call the wrapper
with an id that does not exist. Code review does not surface this class of bug;
one deliberate failed call does, immediately.

## 2. A wrapper that swallows errors and exits 0 makes the outage invisible

The failure mode: a script under a systemd timer catches himalaya's errors,
writes them to its own `.log`, and returns without propagating anything. `main()`
never yields a non-zero code, so the process always exits 0.

What the operator sees after an upgrade breaks every call to the binary:

- systemd: `Finished …` on every cycle, green.
- `systemctl --failed`: clean.
- Any down-unit monitor: nothing to report.
- The only trace: hundreds of `FAIL` lines in a log nobody reads.

An outage can run for many hours this way. With a correct exit code the alert
fires on the second cycle.

Propagate — one broken rule should not abort the others, but it **must** show up
in the exit status:

```python
def main():
    ok = True
    for rule in rules:
        try:
            if not run_rule(rule):     # run_rule returns False on failure
                ok = False
        except Exception as ex:
            log(f"EXCEPTION: {ex!r}")
            ok = False
    return 0 if ok else 1

if __name__ == "__main__":
    sys.exit(main())
```

**The general rule:** if a himalaya script runs under systemd/cron and its only
health signal is its own log file, it has no health signal. Detailed logging is
good; the *exit code* is what the monitor reads. Every `except` that does not
end up influencing the exit status is a blind spot with an expiry date — the day
the CLI changes, it fails silently.

## 3. Checklist for any unattended himalaya wrapper

- Check `returncode` before parsing `stdout` on **every** invocation.
- Return a sentinel (`None`) on failure instead of a best-effort string.
- Aggregate per-item failures into a single non-zero exit at the end.
- Log stderr (truncated) so the *why* survives, but never rely on the log as the
  alerting mechanism.
- Pin/verify the CLI version the wrapper was written against — a major upgrade
  changes flags, subcommands and JSON shape.
- Exercise the wrapper's failure path in tests: bad id, bad mailbox name, bad
  account.
