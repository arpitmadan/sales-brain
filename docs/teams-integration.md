# Microsoft Teams Integration

When a Zoom call is processed by Sales Brain, a notification is automatically posted to the **"Sales Call Recordings"** Teams channel. The rep is @mentioned so they receive a direct alert.

## What the Notification Looks Like

```
┌─────────────────────────────────────────────────────────┐
│ 🧠 Sales Brain — New Call Ready                          │
├─────────────────────────────────────────────────────────┤
│ EZFacility Demonstration · Finezza Futsal Academy        │
│                                                         │
│ Rep        @Marcus Pruitt                               │
│ Account    Finezza Futsal Academy                       │
│ Duration   45 min                                       │
│ Date       Jun 23, 2026 · 10:59 AM                      │
│                                                         │
│ Talk Ratio     75% ⚠ Above range                        │
│ Interactivity  9.6  ✓ Within range                      │
│ Patience       0.41s ⚠ Below range                      │
│                                                         │
│            [ View Call → ]                              │
└─────────────────────────────────────────────────────────┘
```

The **View Call** button links directly to the Sales Brain call review page — video player, transcript, analytics, and comment thread.

---

## Setup: Incoming Webhook (5 minutes)

This approach requires no Azure app registration or admin permissions — just a Teams channel webhook URL.

### Step 1 — Create the Teams Channel

In Microsoft Teams:
1. Create a new channel in your Sales team called **"Sales Call Recordings"**
2. Add all reps and managers who should see call notifications

### Step 2 — Add an Incoming Webhook

1. In the channel, click **"..." → Manage channel → Edit**
2. Go to **Connectors → Incoming Webhook → Configure**
3. Name it `Sales Brain`, upload a logo if desired
4. Click **Create** → copy the webhook URL
5. Paste the URL into your `.env` as `TEAMS_WEBHOOK_URL`

### Step 3 — Rep Teams ID Mapping

To @mention a rep in Teams, Sales Brain needs their Teams user ID mapped to their Zoom host ID. Store this in the `users` table:

```
users
  ├── id
  ├── email
  ├── name
  ├── zoom_user_id       ← matched from Zoom host_id on the call
  ├── teams_user_id      ← Microsoft Teams AAD object ID
  └── role
```

To find a user's Teams ID: In Teams, click their profile → the URL contains their AAD object ID. Alternatively, a one-time admin query via Microsoft Graph: `GET /users?$filter=mail eq 'rep@ezfacility.com'`.

---

## How It Works

When a call finishes processing in Sales Brain:

```
Call processing complete
        │
        ▼
Look up rep → get teams_user_id
        │
        ▼
Build Adaptive Card payload
        │
        ├── Call title, account, date, duration
        ├── Top 3 analytics with flags
        ├── @mention rep: <at>Rep Name</at>
        └── "View Call" button → Sales Brain URL
        │
        ▼
POST to TEAMS_WEBHOOK_URL
        │
        ▼
Message appears in "Sales Call Recordings" channel
Rep receives a notification badge
```

---

## Payload Format (Teams Adaptive Card)

Teams uses **Adaptive Cards** for rich message formatting. The payload sent to the webhook:

```json
{
  "type": "message",
  "attachments": [
    {
      "contentType": "application/vnd.microsoft.card.adaptive",
      "content": {
        "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
        "type": "AdaptiveCard",
        "version": "1.4",
        "body": [
          {
            "type": "TextBlock",
            "text": "🧠 Sales Brain — New Call Ready",
            "weight": "Bolder",
            "size": "Medium"
          },
          {
            "type": "TextBlock",
            "text": "{{call.title}} · {{account.name}}",
            "wrap": true
          },
          {
            "type": "FactSet",
            "facts": [
              { "title": "Rep", "value": "<at>{{rep.name}}</at>" },
              { "title": "Account", "value": "{{account.name}}" },
              { "title": "Duration", "value": "{{call.duration_min}} min" },
              { "title": "Date", "value": "{{call.started_at}}" },
              { "title": "Talk Ratio", "value": "{{analytics.talk_ratio}}% {{analytics.talk_ratio_flag}}" },
              { "title": "Interactivity", "value": "{{analytics.interactivity}} {{analytics.interactivity_flag}}" },
              { "title": "Patience", "value": "{{analytics.patience_sec}}s {{analytics.patience_flag}}" }
            ]
          }
        ],
        "actions": [
          {
            "type": "Action.OpenUrl",
            "title": "View Call →",
            "url": "https://salesbrain.ezfacility.com/calls/{{call.id}}"
          }
        ],
        "msteams": {
          "entities": [
            {
              "type": "mention",
              "text": "<at>{{rep.name}}</at>",
              "mentioned": {
                "id": "{{rep.teams_user_id}}",
                "name": "{{rep.name}}"
              }
            }
          ]
        }
      }
    }
  ]
}
```

Flag values in the payload:
- `✓ Within range`
- `⚠ Above range`
- `⚠ Below range`

---

## Future: Coaching Comment Notifications

When a manager leaves a timestamped comment on a rep's call, a second Teams notification can be sent — either to the same channel or as a direct message to the rep:

```
┌─────────────────────────────────────────────────────┐
│ 💬 New coaching comment on your call                │
├─────────────────────────────────────────────────────┤
│ EZFacility Demo · Finezza Futsal Academy             │
│                                                     │
│ From    Arpit Madan                                 │
│ At      14:32 in the call                           │
│                                                     │
│ "Try asking an open-ended question here instead     │
│  of jumping to the feature explanation."            │
│                                                     │
│         [ Jump to Comment → ]                       │
└─────────────────────────────────────────────────────┘
```

The **Jump to Comment** link opens the Sales Brain player at the exact timestamp of the comment.

This is Phase 2 — direct DM requires Microsoft Graph API access rather than just the webhook.
