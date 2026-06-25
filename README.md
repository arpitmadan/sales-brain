# Sales Brain

An internal call review and coaching platform built on Zoom recordings and Claude AI — designed to replace Gong for EZFacility's sales team.

## The Problem

EZFacility pays ~$2,200/month for Gong. Zoom (which we already pay for) records and transcribes every call. The gap between Zoom and Gong is the software layer built around those recordings: timestamped coaching comments, a shared review dashboard, shareable prospect links, and AI-generated deal intelligence.

Sales Brain closes that gap.

## What It Does

### Core Features

**Call Ingestion**
- Zoom webhook triggers automatically when a call recording is ready
- Recording and transcript are pulled from Zoom API and stored
- Participants are identified as rep (host) vs. prospect (external)

**Review Dashboard**
- Watch any call recording with a synced transcript sidebar
- Leave timestamped comments pinned to an exact moment in the call
- Reps see comments with the timestamp so they can jump directly to the flagged moment
- Any internal user can be granted access to review calls

**Prospect Sharing**
- Generate a shareable link for any recording to send to prospects post-demo or post-disco
- Optional password protection and expiry
- Prospects get view-only access — no account required

**Call Analytics**
All metrics computed from Zoom's speaker-diarized transcript:

| Metric | Description | Source |
|--------|-------------|--------|
| Talk Ratio | % of call time the rep spoke | Rep duration ÷ total duration |
| Longest Monologue | Longest continuous rep speech block | Max consecutive rep utterance |
| Longest Customer Story | Longest continuous prospect speech block | Max consecutive prospect utterance |
| Interactivity | Speaker switches per hour | Switch count ÷ duration |
| Patience | Avg gap between prospect finishing and rep responding | Mean gap at speaker transitions |

Each metric is flagged as within/above/below range based on configurable team benchmarks.

**Claude AI Integration**
- One-click follow-up email generation from transcript
- Deal intelligence extraction: objections raised, next steps, key topics, sentiment
- Custom prompts can be added per call type (demo, disco, renewal, etc.)

## Why Build It

| | Gong | Sales Brain |
|--|------|-------------|
| Monthly cost | $2,200 | ~$150–200 (infra + API) |
| Timestamped coaching comments | Yes | Yes |
| Prospect sharing | Yes | Yes |
| Call analytics | Yes | Yes |
| AI email / deal intel | Yes (generic) | Yes (custom prompts) |
| Salesforce sync | Yes | Phase 2 |
| Team performance dashboards | Yes | Phase 2 |
| Ownership of data | No | Yes |

Break-even: under 3 months of Gong savings.

## Technical Architecture

### Stack

| Layer | Technology |
|-------|-----------|
| Frontend + API | Next.js (App Router) |
| Hosting | Vercel |
| Database | Supabase (PostgreSQL) |
| Auth | Supabase Auth or Clerk |
| Video storage | Zoom Cloud (existing) or S3 |
| AI | Anthropic Claude API |
| Call ingestion | Zoom Webhooks + Server-to-Server OAuth |

### Data Flow

```
Zoom Call Ends
      │
      ▼
Zoom Webhook → Sales Brain API
      │
      ├── Pull recording URL
      ├── Pull VTT transcript (speaker-labeled, timestamped)
      ├── Parse speakers → identify rep vs. prospect
      ├── Compute analytics (talk ratio, monologue, patience, etc.)
      └── Store in Supabase
              │
              ▼
        Dashboard (Next.js)
              │
              ├── Video player + transcript sidebar
              ├── Timestamped comment thread
              ├── Analytics panel (with range benchmarks)
              ├── Claude AI panel (email, deal intel)
              └── Share link generator
```

### Database Schema (core tables)

```
calls          — id, zoom_meeting_id, rep_id, title, date, duration, recording_url
participants   — id, call_id, zoom_user_id, name, role (rep | prospect)
utterances     — id, call_id, participant_id, start_ms, end_ms, text
call_analytics — id, call_id, talk_ratio, longest_monologue_ms, longest_customer_story_ms, interactivity_score, patience_ms
comments       — id, call_id, user_id, timestamp_ms, body, created_at
shares         — id, call_id, token, expires_at, password_hash
users          — id, email, name, role (admin | rep | viewer)
```

## Build Plan

### Phase 1 — MVP (target: 2–3 weeks)
- [ ] Zoom Server-to-Server OAuth app setup
- [ ] Webhook endpoint to ingest recordings on call end
- [ ] Transcript parser: utterance extraction + speaker identification
- [ ] Analytics computation pipeline
- [ ] Video player UI with transcript sidebar
- [ ] Timestamped comment system
- [ ] Basic auth (internal users only)
- [ ] Shareable prospect links

### Phase 2 — Coaching Layer
- [ ] Claude AI: email generation from transcript
- [ ] Claude AI: deal intelligence extraction
- [ ] Rep performance dashboard (aggregate analytics over time)
- [ ] Comment notifications (email rep when a comment is left on their call)
- [ ] Custom analytics benchmarks per call type

### Phase 3 — Integrations
- [ ] Salesforce activity sync (log call as activity on opportunity)
- [ ] Slack notifications on new call ingested
- [ ] Custom scorecards per call type (demo, disco, renewal)

## Zoom Setup Required

1. Go to [Zoom App Marketplace](https://marketplace.zoom.us)
2. Create a **Server-to-Server OAuth** app
3. Grant scopes: `recording:read:admin`, `user:read:admin`
4. Enable **cloud recording** and **audio transcript** in Zoom account settings
5. Set webhook endpoint to `https://your-domain.com/api/webhooks/zoom`
6. Subscribe to event: `recording.completed`

## Environment Variables

```env
ZOOM_ACCOUNT_ID=
ZOOM_CLIENT_ID=
ZOOM_CLIENT_SECRET=
ZOOM_WEBHOOK_SECRET_TOKEN=

NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=

ANTHROPIC_API_KEY=

NEXTAUTH_SECRET=
NEXTAUTH_URL=
```

## Team

Built for EZFacility sales operations. Internal use only.
