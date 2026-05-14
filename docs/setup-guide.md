# Pragmatico Recording System — Setup Guide

## Architecture

```
Zoom meeting.started
  → Flow 1 (n8n): check whitelist → deploy 3 Recall bots → save bot IDs to static data

Zoom meeting.ended (fires once)
  → Flow 2 (n8n): look up bot IDs → wait 20 min → fetch recordings from Recall API
  → Trigger GitHub Actions

GitHub Actions (free Ubuntu runner):
  → Download MP4 files to disk
  → ffmpeg merge → 1 file
  → Upload as GitHub Release asset (public URL)
  → Upload to Fireflies (1 meeting entry, full transcript)
```

**Why meeting.ended instead of Recall webhooks:**
Recall fires one webhook per bot (3 concurrent executions), which causes race conditions
in n8n static data. Using Zoom's meeting.ended webhook fires exactly once, giving one
clean execution with no coordination needed.

---

## n8n Flows

### Flow 1 — Bot Deployment (`flow1-bot-deployment-v6.json`)
**Trigger:** Zoom webhook → `meeting.started`  
**n8n path:** `/webhook/zoom-webhook`

What it does:
1. Checks if meeting ID is in the Sessions whitelist (Google Sheet)
2. Gets Zoom OAuth token + meeting join URL
3. Creates 3 Recall bots (main room + 2 breakouts)
4. Saves bot IDs to static data (used by Flow 2)
5. Saves bot IDs to Google Sheet (optional — for when Sheets is set up)

### Flow 2 — Fetch & Merge (`flow2-meeting-ended-v1.json`)
**Trigger:** Zoom webhook → `meeting.ended`  
**n8n path:** `/webhook/zoom-meeting-ended`

What it does:
1. Reads bot IDs from static data (saved by Flow 1)
2. Waits 20 minutes for Recall to finish processing recordings
3. Calls Recall API for each bot to get video URLs
4. Triggers GitHub Actions with all 3 URLs

---

## Zoom App Setup

Go to: Zoom Marketplace → your app → Feature → Event Subscriptions

Add **two** event subscriptions (can point to same app, different URLs):

| Event | Endpoint URL |
|---|---|
| `meeting.started` | `https://pragmatico.app.n8n.cloud/webhook/zoom-webhook` |
| `meeting.ended` | `https://pragmatico.app.n8n.cloud/webhook/zoom-meeting-ended` |

Both URLs must be validated — workflows must be **Active** in n8n before validating.

---

## GitHub Actions Secrets

Go to: `https://github.com/dancrrza/pragmatico-automations/settings/secrets/actions`

| Secret Name | Value |
|---|---|
| `FIREFLIES_API_KEY` | Your Fireflies API key for Pragmatico workspace |
| `N8N_CALLBACK_URL` | `https://pragmatico.app.n8n.cloud/webhook/recording-merged` |

---

## n8n Setup

### Credentials to add in n8n:
1. **Google Sheets (OAuth2)** — authorize with the Google account that owns the Sheet
2. **GitHub PAT** — used in both flows to trigger GitHub Actions
   - Go to: `https://github.com/settings/tokens`
   - Create token with `repo` and `workflow` scopes

### Values to replace (search for `CHANGE_THIS`):

| Placeholder | Replace with |
|---|---|
| `CHANGE_THIS_RECALL_API_TOKEN` | Your Recall.ai API token (recall.ai dashboard → API Keys) |
| `CHANGE_THIS_ZOOM_ACCOUNT_ID` | Your Zoom account ID (Zoom marketplace app credentials) |
| `CHANGE_THIS_ZOOM_BASIC_AUTH` | Base64 of `client_id:client_secret` from your Zoom app |
| `CHANGE_THIS_GOOGLE_SHEET_ID` | The ID from your Sheet URL: `https://docs.google.com/spreadsheets/d/THIS_PART/` |
| `CHANGE_THIS_CREDENTIAL_ID` | The n8n credential ID after you set up Google Sheets OAuth |
| `CHANGE_THIS_GITHUB_PAT` | Your GitHub Personal Access Token (repo + workflow scopes) |

### Import order:
1. Import `flow1-bot-deployment-v6.json` → fill in credentials → Activate
2. Import `flow2-meeting-ended-v1.json` → fill in credentials → Activate
3. Disable any older flow versions

---

## Google Sheet Setup (optional — for whitelist + attendance)

Create a Google Sheet with 3 tabs:

### Tab 1: Sessions
| Column | Name | Notes |
|---|---|---|
| A | meeting_id | Zoom meeting ID |
| B | session_title | e.g. "AI Fundamentals Week 1" |
| C | date | YYYY-MM-DD |
| D | bot1_id | Filled automatically by Flow 1 |
| E | bot2_id | Filled automatically by Flow 1 |
| F | bot3_id | Filled automatically by Flow 1 |
| G | expected_attendees | Comma-separated emails |
| H | status | pending / recording / merging / done / error |

**Before each session:** add a row with meeting_id, session_title, date.
This is the whitelist — only these meetings get bots.

### Tab 2: Bot Recordings
| Column | Name | Notes |
|---|---|---|
| A | meeting_id | Links to Sessions tab |
| B | bot_id | Recall.ai bot ID |
| C | video_url | S3 presigned URL |
| D | recorded_at | Timestamp |

### Tab 3: Attendance
| Column | Name | Notes |
|---|---|---|
| A | meeting_id | |
| B | email | Participant email |
| C | name | Participant name |
| D | attended_live | TRUE / FALSE |
| E | join_time | When they joined |
| F | duration_min | Minutes stayed |
| G | watched_recording | TRUE / FALSE |
| H | watched_at | When they clicked the recording link |

---

## Testing

Trigger a test merge manually:
1. Go to: `https://github.com/dancrrza/pragmatico-automations/actions/workflows/merge-recordings.yml`
2. Click "Run workflow"
3. Fill in fresh Recall.ai recording URLs (valid for ~5 hours after recording)
4. Watch logs — should complete in ~3 minutes

---

## Known Limitations

**Transcript repetition:** All 3 bots start in the main room before being assigned to
breakout rooms. The merged video has the intro section repeated (once per bot). Fireflies
transcribes it linearly — the intro appears 3 times in the transcript. Future fix: AI
post-processing to deduplicate repeated sections.

**Zoom meeting security:** Waiting Room must be OFF and "Require authentication" must be
OFF — otherwise bots can't join.
