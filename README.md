# Pragmatico Automations

Automated session recording and attendance tracking for Pragmatico class sessions.

## What This Does

1. Deploys 2 recording bots into Zoom meetings (one per breakout room)
2. Captures timestamps when breakout rooms open and close
3. Merges recordings into one MP4 — intro + breakout room 1 + breakout room 2 + closing
4. Publishes the merged video as a private download link
5. Sends each participant a personalized link to watch the recording
6. Tracks who watched, how much, and when

## Files

```
.github/workflows/
  merge-recordings.yml   ← downloads recordings, merges with ffmpeg, publishes

player.html              ← video player served via GitHub Pages
                           tracks watch progress per participant
```

## Required Secrets

Set these in the repository Settings → Secrets → Actions:

| Secret | Description |
|--------|-------------|
| `FIREFLIES_API_KEY` | API key for transcript upload |
| `N8N_CALLBACK_URL` | Webhook URL to notify when video is ready |

## Video Player

The player is available at:
```
https://dancrrza.github.io/pragmatico-automations/player.html
```

It accepts URL parameters to identify the session and participant:
- `v` — video URL
- `s` — session ID
- `e` — participant email
- `d` — session date
- `t` — session title
