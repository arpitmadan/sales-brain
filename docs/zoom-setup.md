# Zoom API Setup

Sales Brain uses Zoom's Server-to-Server OAuth to pull recordings and transcripts automatically when a call ends.

## Step 1: Enable Cloud Recording + Transcription in Zoom

1. Log into Zoom Admin as an account owner
2. Go to **Account Management > Account Settings > Recording**
3. Enable:
   - Cloud recording
   - Audio transcript
   - (Optional) Record active speaker with shared screen

## Step 2: Create a Server-to-Server OAuth App

1. Go to [Zoom App Marketplace](https://marketplace.zoom.us) → Develop → Build App
2. Choose **Server-to-Server OAuth**
3. Name it something like `Sales Brain`
4. Copy the **Account ID**, **Client ID**, **Client Secret** — these go into your `.env`

## Step 3: Set App Scopes

Under **Scopes**, add:
- `recording:read:admin` — list and download cloud recordings
- `user:read:admin` — identify internal users (reps) vs. external (prospects)
- `meeting:read:admin` — fetch meeting metadata

## Step 4: Configure the Webhook

1. In the app, go to **Feature > Event Subscriptions**
2. Add a new subscription
3. Set the **Event notification endpoint URL** to:
   ```
   https://your-domain.com/api/webhooks/zoom
   ```
4. Subscribe to event type: `recording.completed`
5. Copy the **Secret Token** → goes into `ZOOM_WEBHOOK_SECRET_TOKEN` in your `.env`

## Step 5: Activate the App

Click **Activate** on the app page. The app is now live for your Zoom account.

---

## How the Webhook Works

When a Zoom call ends and recording processing finishes (~5–10 min after call), Zoom sends a POST to your webhook endpoint:

```json
{
  "event": "recording.completed",
  "payload": {
    "object": {
      "uuid": "abc123",
      "host_id": "zoom_user_id",
      "topic": "Demo with Acme Corp",
      "start_time": "2026-06-24T14:00:00Z",
      "duration": 45,
      "recording_files": [
        {
          "recording_type": "shared_screen_with_speaker_view",
          "download_url": "https://zoom.us/rec/download/...",
          "file_type": "MP4"
        },
        {
          "recording_type": "audio_transcript",
          "download_url": "https://zoom.us/rec/download/...",
          "file_type": "VTT"
        }
      ]
    }
  }
}
```

Sales Brain's webhook handler:
1. Validates the request signature using `ZOOM_WEBHOOK_SECRET_TOKEN`
2. Pulls the MP4 and VTT download URLs
3. Fetches the VTT transcript from Zoom (using a temporary access token)
4. Stores everything in Supabase
5. Runs the analytics computation pipeline
6. Notifies the rep that their call is ready for review
