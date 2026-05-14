# Pragmatico Recording System — Setup Guide

## Architecture

```
Zoom meeting.started
  → Flow 1: check whitelist → deploy 3 Recall bots → save bot IDs to Google Sheet

Each bot finishes recording → Recall webhook fires (×3)
  → Flow 2: get video URL → store in Google Sheet → when all 3 collected → trigger GitHub Actions

GitHub Actions runner (free Ubuntu):
  → Download 3 MP4 files to disk (no memory limits)
  → ffmpeg merge → 1 file
  → Upload to Fireflies (1 meeting entry, full transcript)
  → Notify n8n callback

n8n callback:
  → Pull Zoom attendance report
  → Email absentees with tracking link
  → Log attendance in Google Sheet
```

---

## Google Sheet Setup

Create a Google Sheet with these 3 tabs:

### Tab 1: Sessions
| Column | Name | Notes |
|---|---|---|
| A | meeting_id | Zoom meeting ID (from URL or Zoom dashboard) |
| B | session_title | e.g. "AI Fundamentals Week 1" |
| C | date | YYYY-MM-DD |
| D | bot1_id | Filled automatically by Flow 1 |
| E | bot2_id | Filled automatically by Flow 1 |
| F | bot3_id | Filled automatically by Flow 1 |
| G | expected_attendees | Comma-separated emails |
| H | status | pending / recording / merging / done / error |

**Before each session:** add a row with meeting_id, session_title, date, expected_attendees.
This is how you control which meetings get bots (whitelist).

### Tab 2: Bot Recordings
| Column | Name | Notes |
|---|---|---|
| A | meeting_id | Links to Sessions tab |
| B | bot_id | Recall.ai bot ID |
| C | video_url | S3 presigned URL (valid 5 hours) |
| D | recorded_at | Timestamp from Recall webhook |

Filled automatically by Flow 2.

### Tab 3: Attendance
| Column | Name | Notes |
|---|---|---|
| A | meeting_id | |
| B | email | Participant email |
| C | name | Participant name |
| D | attended_live | TRUE / FALSE |
| E | join_time | When they joined |
| F | duration_min | How long they stayed |
| G | watched_recording | TRUE / FALSE |
| H | watched_at | When they clicked the recording link |

---

## GitHub Actions Secrets

Go to: `https://github.com/dancrrza/pragmatico-automations/settings/secrets/actions`

Add these 2 secrets:

| Secret Name | Value |
|---|---|
| `FIREFLIES_API_KEY` | Your Fireflies API key for Pragmatico workspace |
| `N8N_CALLBACK_URL` | `https://pragmatico.app.n8n.cloud/webhook/recording-merged` |

---

## n8n Setup

### Credentials to add in n8n:

1. **Google Sheets (OAuth2)** — authorize with the Google account that owns the Sheet
2. **GitHub PAT** — used in Flow 2 to trigger GitHub Actions
   - Go to: `https://github.com/settings/tokens`
   - Create token with `repo` and `workflow` scopes
   - Paste into the "Trigger GitHub Actions" node header

### Values to change in both flows (search for `CHANGE_THIS`):

| Placeholder | Replace with |
|---|---|
| `CHANGE_THIS_RECALL_API_TOKEN` | Your Recall.ai API token (from recall.ai dashboard → API Keys) |
| `CHANGE_THIS_ZOOM_ACCOUNT_ID` | Your Zoom account ID (from Zoom marketplace app credentials) |
| `CHANGE_THIS_ZOOM_BASIC_AUTH` | Base64 of `client_id:client_secret` from your Zoom app |
| `CHANGE_THIS_GOOGLE_SHEET_ID` | The ID from your Sheet URL: `https://docs.google.com/spreadsheets/d/THIS_PART/` |
| `CHANGE_THIS_CREDENTIAL_ID` | The n8n credential ID after you set up Google Sheets OAuth |
| `CHANGE_THIS_GITHUB_PAT` | Your GitHub Personal Access Token (repo + workflow scopes) |

---

## Note on File Storage (0x0.st)

The GitHub Actions workflow uses **0x0.st** as temporary storage for the merged MP4 before passing it to Fireflies. No account required. Files are kept for 30–90 days depending on size.

For production with sensitive recordings, replace the upload step with **Cloudinary**:
1. Create a free Cloudinary account
2. Add 3 secrets to GitHub: `CLOUDINARY_CLOUD`, `CLOUDINARY_KEY`, `CLOUDINARY_SECRET`
3. Replace the 0x0.st step in the workflow with the Cloudinary upload

---

## Testing the Merge Manually

You can trigger a test merge from GitHub directly:

1. Go to: `https://github.com/dancrrza/pragmatico-automations/actions/workflows/merge-recordings.yml`
2. Click "Run workflow"
3. Fill in 2-3 Recall.ai recording URLs from a past session
4. Set a session title and date
5. Run

Watch the logs to verify each step completes.
