# Multi-Project Standup Agent — Setup Guide

## Files

| File | Description |
|------|-------------|
| `workflow-a-digest.json` | Scheduled daily digest → Telegram |
| `workflow-b-query.json`  | On-demand "where did I stop on X" → Telegram reply |

> [!IMPORTANT]
> After importing each workflow, update: **(1)** the Google Sheets node — select your "Project Log" spreadsheet and sheet from the dropdowns; **(2)** the Schedule Trigger time in Workflow A (default 08:00); **(3)** the n8n variable `MY_TELEGRAM_CHAT_ID` (see below).

---

## 1. Google Sheets OAuth2 Credential

**n8n credential type:** `Google Sheets OAuth2 API`
**n8n credential name:** `Google Sheets OAuth2`

### Steps

1. Go to [Google Cloud Console](https://console.cloud.google.com/) → create or select a project.
2. **APIs & Services → Library** → enable **Google Sheets API** and **Google Drive API**.
3. **APIs & Services → Credentials → Create Credentials → OAuth client ID**.
   - Application type: **Web application**.
   - Authorized redirect URI: `https://<your-n8n-domain>/rest/oauth2-credential/callback` (n8n Cloud users: use your `*.app.n8n.cloud` domain).
4. Copy the **Client ID** and **Client Secret**.
5. In n8n → **Settings → Credentials → New → Google Sheets OAuth2 API**:
   - Paste Client ID & Client Secret.
   - Click **Sign in with Google** and authorize the account that owns "Project Log".
6. **Scopes granted automatically:** `https://www.googleapis.com/auth/spreadsheets.readonly` and `https://www.googleapis.com/auth/drive.readonly` (n8n requests these by default; no manual scope configuration is needed).

---

## 2. Gemini API Key Credential

**n8n credential type:** `Header Auth`
**n8n credential name:** `Gemini API`

This is the LLM used by both workflows. (Groq was originally used, but its free-tier model `llama-3.3-70b-versatile` was deprecated by Groq — this project now runs entirely on Gemini.)

### Steps

1. Go to [aistudio.google.com](https://aistudio.google.com/) → sign in with Google.
2. Click **Get API key → Create API key** → select/create a project → copy the key.
3. In n8n → **Settings → Credentials → New → Header Auth**:
   - **Name:** `Gemini API`
   - **Header Name:** `x-goog-api-key`
   - **Header Value:** your Gemini API key (paste directly — **no** `Bearer` prefix, that's an OpenAI/Groq convention, not Gemini's).
4. On both workflows' LLM node (**Groq LLM – Digest** in Workflow A, **Groq LLM – Query** in Workflow B — names are historical, both call Gemini now):
   - **Authentication:** Generic Credential Type
   - **Generic Auth Type:** Header Auth
   - **Credential:** select the `Gemini API` credential created above.

**Model used:** `gemini-3.1-flash-lite` (free tier). Model names on Gemini's free tier change periodically — if you get a "model does not exist" error in the future, check [ai.google.dev/gemini-api/docs/pricing](https://ai.google.dev/gemini-api/docs/pricing) for the current free-tier Flash model and update the URL below.

**Endpoint URL (both LLM nodes):**
```
https://generativelanguage.googleapis.com/v1beta/models/gemini-3.1-flash-lite:generateContent
```

**Request body shape** (Content-Type: JSON) — only the `system_instruction` text differs between the two workflows:
```
{{ JSON.stringify({
  system_instruction: { parts: [{ text: "<system prompt for this workflow>" }] },
  contents: [ { role: "user", parts: [{ text: $json.formattedText }] } ]
}) }}
```

> [!WARNING]
> **Response path.** Gemini's response shape is different from Groq/OpenAI-style APIs. Every node that reads the LLM's output — including both Telegram send nodes — must use:
> ```
> {{ $json.candidates[0].content.parts[0].text }}
> ```
> **Not** `{{ $json.choices[0].message.content }}` (that's the old Groq path). Leaving the old path in place doesn't error — it silently resolves to `undefined`, which then gets sent as the literal text of your Telegram message. If you ever see a message that just says "undefined," this is almost always the cause — check every downstream node for the old path.

---

## 3. Telegram Bot Credential

**n8n credential type:** `Telegram API`
**n8n credential name:** `Telegram Bot`

### Steps

1. Open Telegram and message [@BotFather](https://t.me/BotFather).
2. Send `/newbot`, follow the prompts to choose a name and username.
3. BotFather replies with a **bot token** (e.g. `123456:ABC-DEF...`). Copy it.
4. In n8n → **Settings → Credentials → New → Telegram API**:
   - **Name:** `Telegram Bot`
   - **Access Token:** paste the bot token from step 3.

### Get Your Chat ID

1. Start a conversation with your new bot in Telegram (send it any message like `/start`).
2. Open `https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates` in a browser.
3. Find `"chat":{"id": 123456789, ...}` in the JSON response — that number is your chat ID.
4. Alternatively, message [@userinfobot](https://t.me/userinfobot) and it will reply with your ID.

### Configure n8n Variable

In n8n → **Settings → Variables**, create:

| Variable | Example Value | Description |
|----------|---------------|-------------|
| `MY_TELEGRAM_CHAT_ID` | `5527077768` | Your personal Telegram chat ID (from step above) |

> [!TIP]
> Workflow A uses `MY_TELEGRAM_CHAT_ID` to send the daily digest to you specifically. Workflow B replies to whatever chat the message came from (`$json.message.chat.id`, extracted automatically from the incoming Telegram Trigger payload), so querying works from the direct bot chat or any group the bot is added to.

---

## 4. Google Sheet Structure

Your sheet named **"Project Log"** must have these exact column headers in row 1:

| Project | Date | What I did | Blocker/Next step | Status |
|---------|------|------------|-------------------|--------|

- **Project:** text, the project name (must match exactly across rows for grouping).
- **Date:** a date (any format Google Sheets recognizes; ISO `YYYY-MM-DD` is safest).
- **What I did:** free text.
- **Blocker/Next step:** free text.
- **Status:** free text (e.g. "In Progress", "Blocked", "Done").

---

## 5. How each workflow behaves

### Workflow A — Daily Digest
- Fires automatically once a day at the Schedule Trigger's set time. No message from you is needed.
- Reads the **single most recent row per project**, summarizes each in 2–3 sentences, sends one Telegram message covering all active projects.
- **Runs every day regardless of whether you logged anything new.** On a day with no new entries, it re-summarizes the same latest entry as before — this is expected, not a bug. If you'd rather it skip silently on no-new-data days, that's a small follow-up change (compare latest log date to today, skip send if nothing new).

### Workflow B — On-Demand Query
- Only runs when you message the bot something like `where did I stop on <project name>`.
- Pulls the **full history** of that one project (not just the latest row) and returns a detailed 100–180 word reconstruction: overall progress, current status, last blocker, single next step.
- **Edge-case handling** (fixed after initial testing surfaced a bug where unmatched queries incorrectly returned Workflow A–style output):
  1. **No project name found in the message** (e.g. "hey", "thanks", "where did I stop on" with nothing after it) → replies with a fixed clarifying message, no Sheets or LLM call is made.
  2. **Project name extracted but zero matching rows** in the sheet (e.g. a typo or a project that doesn't exist) → replies with a fixed "couldn't find that project" message, no LLM call is made.
  3. **Project name extracted and matching rows found** → proceeds through Sheets filter → Gemini → detailed Telegram reply, as above.

---

## 6. Post-Import Checklist

- [ ] Import `workflow-a-digest.json` and `workflow-b-query.json` via **n8n → Workflows → Import from File**.
- [ ] In each workflow's **Read Project Log** node, select your "Project Log" spreadsheet & sheet.
- [ ] In Workflow A's **Schedule Trigger**, verify/change the trigger time (default 08:00).
- [ ] Create the credentials above: **Google Sheets OAuth2**, **Gemini API** (Header Auth), **Telegram Bot**.
- [ ] On both workflows' LLM nodes, confirm the URL points to the Gemini endpoint (not Groq), and the response-path fields downstream use `candidates[0].content.parts[0].text`.
- [ ] Set the n8n variable `MY_TELEGRAM_CHAT_ID` to your Telegram chat ID.
- [ ] Send `/start` to your bot in Telegram so it can message you.
- [ ] Test both workflows manually (Execute Workflow) before publishing.
- [ ] **Publish** both workflows (not just save — publishing is what activates the schedule and the Telegram webhook trigger).
- [ ] Test again with a real Telegram message after publishing, since draft edits only take effect for live triggers once published.

---