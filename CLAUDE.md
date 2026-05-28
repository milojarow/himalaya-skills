# CLAUDE.md

This file provides guidance to Claude Code when working with this repository.

## Project Overview

This is the **himalaya-skills** repository — a Claude Code marketplace that ships one skill for driving the [himalaya](https://github.com/pimalaya/himalaya) terminal email client over IMAP/SMTP.

**Repository**: https://github.com/milojarow/himalaya-skills

## Repository Structure

```
himalaya-skills/
├── .claude-plugin/          # marketplace.json + plugin.json
├── CLAUDE.md                # This file
├── README.md                # Project overview
├── LICENSE                  # MIT
├── evaluations/             # Test scenarios for the skill
└── skills/
    └── himalaya/
        ├── SKILL.md          # Entry point (lean) — includes the Yahoo throttling warning
        └── reference/        # installation.md, configuration.md, composing-messages.md
```

## The skill

### himalaya
Drive the himalaya CLI for email. `SKILL.md` is the lean entry point with a prominent Yahoo throttling warning, a quick-reference table, and cross-links to:
- `reference/installation.md` — how to get himalaya on a fresh box (cargo, brew, pacman, prebuilt).
- `reference/configuration.md` — `config.toml` layout for Gmail, iCloud, Outlook, Yahoo, generic IMAP, OAuth2, notmuch.
- `reference/composing-messages.md` — write/reply/forward + the full MML reference for HTML / attachments / inline images.

## Skill Activation

Activates when reading, searching, organizing or composing email from the terminal via the himalaya CLI — listing/searching messages, managing folders/flags/attachments, writing/replying/forwarding, account setup, MML composition.

## Updating this skill

After any session that discovers a new wall, gotcha or provider quirk. Keep entries **generic** — placeholder addresses, hostnames and paths only; never real accounts, hosts or credentials. The git log of this repo is the diary.
