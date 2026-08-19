# Multi-Project Standup Agent

An n8n automation that reduces the cost of switching between parallel projects by turning short, terse daily log entries into (1) a scheduled daily digest across all active projects, and (2) a detailed on-demand explanation for one specific project, delivered over Telegram.

## Why

Working across several parallel projects means constantly losing context every time you switch back to one after a few days away. This agent removes that "where was I?" tax without adding a new logging habit — you keep doing what you already do (a quick note in a Google Sheet after a work session), and the agent turns that into something genuinely useful to read later.

## What it does

**Workflow A — Daily Digest**
Fires automatically every morning at a scheduled time. Reads the single most recent log entry per project, and sends one Telegram message with a short 2–3 sentence summary per project, each ending in a concrete next action.

**Workflow B — On-Demand Query**
Message the bot `where did I stop on <project name>` any time. It pulls the *full* history for that one project and replies with a detailed 100–180 word reconstruction: what's been done overall, where things stand, the last blocker or decision, and the single next step.

Both workflows read from the same Google Sheet — no database, no auto-logging, no multi-user support, by design.

## Architecture

```
Google Sheet ("Project Log")
        │
        ├── Workflow A (Schedule Trigger, daily)
        │     → group by project, keep latest row per project
        │     → Gemini: short digest per project
        │     → Telegram: send to fixed chat
        │
        └── Workflow B (Telegram Trigger, on message)
              → parse project name from message
              → filter sheet to that project's full history
              → Gemini: detailed reconstruction
              → Telegram: reply in same chat
```

## Stack

- [n8n](https://n8n.io/) (Cloud) — workflow orchestration
- Google Sheets — sole data store, manually updated
- [Gemini API](https://aistudio.google.com/) (free tier) — summarization/reconstruction
- Telegram Bot API — delivery and on-demand querying

## Setup

See [`SETUP.md`](./SETUP.md) for full step-by-step instructions, including:
- Google Sheets OAuth2 credential setup
- Gemini API key setup
- Telegram bot creation via BotFather
- Required n8n variables
- Sheet column structure
- Known gotchas (e.g. Gemini's response-path shape vs. OpenAI-style APIs)

## Files

| File | Description |
|---|---|
| `workflow-a-digest.json` | n8n export — scheduled daily digest |
| `workflow-b-query.json` | n8n export — on-demand query workflow |
| `SETUP.md` | Full setup guide and credential instructions |

## Status

Both workflows are running in production (n8n Cloud), triggered on schedule and via live Telegram messages.

## Security note

Workflow JSON exports in this repo do **not** contain credential secrets (n8n excludes credential values from exports by default). You'll still need to create your own credentials per `SETUP.md` — nothing here is usable out of the box without your own Google, Gemini, and Telegram bot tokens.
