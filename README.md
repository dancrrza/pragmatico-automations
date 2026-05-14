# Pragmatico Automations

Automated recording and attendance tracking for Pragmatico AI class sessions.

## What This Does

1. Deploys 3 Recall.ai bots into Zoom meetings (main room + 2 breakout rooms)
2. Collects all 3 recordings when the session ends
3. Merges them into 1 MP4 file via GitHub Actions (ffmpeg)
4. Uploads the merged file to Fireflies as a single meeting entry
5. Tracks who attended live vs. who needs to watch the recording

## Files

```
.github/workflows/
  merge-recordings.yml     ← GitHub Actions: download + merge + upload to Fireflies

n8n/
  flow1-bot-deployment-v6.json   ← Zoom webhook → deploy bots → save to Google Sheet
  flow2-collect-merge-v5.json    ← Recall webhook → collect URLs → trigger merge

docs/
  setup-guide.md           ← Full setup instructions (Google Sheet, secrets, credentials)
```

## Quick Start

See [docs/setup-guide.md](docs/setup-guide.md) for full instructions.

**Required secrets in GitHub Actions Settings:**
- `FIREFLIES_API_KEY`
- `N8N_CALLBACK_URL`

**Required changes in n8n flows:**
- Google Sheet ID
- Google Sheets OAuth2 credential
- GitHub Personal Access Token
