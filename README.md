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
| Salesforce Account tab with call list | Yes | Yes |
| Auto-match calls to SF Accounts | Yes | Yes |
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

## Requirements

### Functional Requirements

| # | Requirement | Priority |
|---|------------|----------|
| F1 | Automatically ingest call recordings and transcripts when a Zoom call ends | Must have |
| F2 | Video player with synced scrolling transcript sidebar | Must have |
| F3 | Timestamped comments — pinned to an exact second in the call | Must have |
| F4 | Reps can click a comment timestamp and jump directly to that moment | Must have |
| F5 | Generate a shareable link for any call to send to a prospect | Must have |
| F6 | Shareable links are view-only, no login required for the prospect | Must have |
| F7 | Call analytics: talk ratio, monologue, customer story, interactivity, patience | Must have |
| F8 | Each metric flagged as within / above / below configurable range | Must have |
| F9 | Salesforce Account page shows a "Sales Brain" tab with all calls for that account | Must have |
| F10 | Calls auto-matched to Salesforce Accounts and Opportunities via participant email | Must have |
| F11 | One-click Claude AI: generate follow-up email from transcript | Must have |
| F12 | One-click Claude AI: extract deal intelligence (objections, next steps, sentiment) | Must have |
| F13 | Access control — admins can grant/revoke internal user access | Must have |
| F14 | Rep performance dashboard — aggregate analytics over time per rep | Nice to have |
| F15 | Email notification to rep when a coaching comment is left on their call | Nice to have |
| F16 | Slack notification when a new call is ingested | Nice to have |
| F17 | Custom scorecards per call type (demo, disco, renewal) | Nice to have |

---

### Salesforce Integration Requirements

This is the most visible integration — replicating the Gong tab on the Salesforce Account page.

#### How Gong Does It
Gong installs a custom Salesforce object and a related list on the Account page layout. When a call ends, Gong looks up participant emails in Salesforce, finds the matching Account, and creates a record. The tab auto-populates.

#### Sales Brain Replication

**Custom Salesforce Object: `Sales_Brain_Call__c`**

| Field | Type | Description |
|-------|------|-------------|
| `Title__c` | Text | Call title from Zoom |
| `Recording_URL__c` | URL | Deep link to Sales Brain call review page |
| `Rep__c` | Lookup → User | Salesforce user who hosted the call |
| `Account__c` | Lookup → Account | Matched account |
| `Opportunity__c` | Lookup → Opportunity | Most recent open opportunity for the account |
| `Started__c` | DateTime | Call start time |
| `Duration_Min__c` | Number | Call duration in minutes |
| `Talk_Ratio__c` | Percent | Rep talk ratio |
| `Status__c` | Picklist | Processing / Ready / Shared with Prospect |

**Salesforce Page Layout**

Add `Sales_Brain_Call__c` as a related list on the Account page layout. Name the section **"Sales Brain"**. Columns shown: Title, Primary Opportunity, Rep, Started. Clicking Title opens the Sales Brain call review page.

**Auto-Match Logic (runs on every call ingestion)**

```
1. Zoom webhook fires → Sales Brain gets participant list with email addresses
2. Filter to external (non-EZFacility) email addresses → these are prospects
3. Query Salesforce Contacts: SELECT AccountId FROM Contact WHERE Email IN [prospect emails]
4. If match found → use that AccountId
5. Query open Opportunities: SELECT Id FROM Opportunity WHERE AccountId = [matched] ORDER BY LastModifiedDate DESC LIMIT 1
6. Create Sales_Brain_Call__c record linked to Account + Opportunity
7. Account page tab updates automatically — no manual tagging needed
```

**Fallback if no email match:**
- Call is ingested into Sales Brain without a Salesforce link
- Rep sees a "Link to Salesforce" button on the call page — they can manually select the Account
- This handles cold outbound calls where the prospect is not yet a Contact

**Salesforce API Access**

Uses a Connected App with OAuth 2.0 (JWT flow for server-to-server). Requires:
- `api` scope
- `refresh_token` scope
- A dedicated Salesforce integration user (not a named user seat)

---

### External Service Requirements

| Service | What's Needed | Notes |
|---------|--------------|-------|
| **Zoom** | Business/Enterprise plan with cloud recording + transcription enabled | Already paying |
| **Zoom App** | Server-to-Server OAuth app in Zoom Marketplace | ~10 min setup |
| **Salesforce** | API access + ability to create custom objects + edit page layouts | Already on Sales Console |
| **Salesforce Connected App** | JWT OAuth for server-to-server API calls | Setup in SF Setup |
| **Anthropic** | API key for Claude integration | Pay per use, ~$50–100/mo |
| **Vercel** | Hosting for Next.js app | Free tier sufficient for MVP |
| **Supabase** | PostgreSQL database + auth | Free tier sufficient for MVP |

---

### Access Control Requirements

| Role | Can Do |
|------|--------|
| **Admin** (you) | View all calls, leave comments, manage users, set analytics benchmarks |
| **Rep** | View own calls, see comments left on their calls, generate AI outputs |
| **Viewer** | View assigned calls, leave comments — for managers reviewing across reps |
| **Prospect** | View a specific shared call via link — no login, no other access |

---

### Data Requirements

- Zoom cloud recording must be enabled **before** a call — recordings cannot be retroactively created
- Transcription requires at least one participant to have audio (not just dial-in without mic)
- Prospect emails must exist in Salesforce Contacts for auto-matching to work; fallback is manual linking
- Sales Brain stores call metadata and comments in Supabase; video stays on Zoom's CDN (no re-hosting cost)

---

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
- [ ] Salesforce custom object `Sales_Brain_Call__c` + page layout
- [ ] Auto-match calls to Salesforce Accounts via participant email

### Phase 2 — Coaching Layer
- [ ] Claude AI: email generation from transcript
- [ ] Claude AI: deal intelligence extraction
- [ ] Rep performance dashboard (aggregate analytics over time)
- [ ] Comment notifications (email rep when a comment is left on their call)
- [ ] Custom analytics benchmarks per call type

### Phase 3 — Polish
- [ ] Slack notifications on new call ingested
- [ ] Custom scorecards per call type (demo, disco, renewal)
- [ ] Manual Salesforce account linking fallback UI

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
