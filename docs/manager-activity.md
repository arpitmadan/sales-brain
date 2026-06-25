# Manager Activity Dashboard

A dedicated section of the Sales Brain dashboard that tracks and displays Arpit's coaching activity — how many calls were listened to, how many received active feedback, and the breakdown per rep. Filterable by any custom date range.

This makes coaching accountability visible and measurable, mirroring the activity tracking Gong provides for managers.

---

## Dashboard Layout

```
┌─────────────────────────────────────────────────────────────────────┐
│  My Coaching Activity                    [ Last 30 days ▼ ] [Custom]│
├──────────────┬──────────────┬──────────────┬────────────────────────┤
│  Calls       │  Calls with  │  Comments    │  Team Coverage         │
│  Listened    │  Feedback    │  Left        │                        │
│              │              │              │                        │
│     18       │     11       │     34       │     62%                │
│              │              │              │  of calls reviewed     │
└──────────────┴──────────────┴──────────────┴────────────────────────┘

  Per Rep Breakdown
  ┌────────────────┬──────────────┬──────────────┬──────────┬─────────────┐
  │ Rep            │ Calls Rec.   │ Reviewed     │ Comments │ Last Review │
  ├────────────────┼──────────────┼──────────────┼──────────┼─────────────┤
  │ Marcus Pruitt  │ 12           │ 8 (67%)      │ 19       │ Today       │
  │ Sarah Klein    │ 9            │ 6 (67%)      │ 10       │ Yesterday   │
  │ James Torres   │ 8            │ 4 (50%)      │ 5        │ 3 days ago  │
  └────────────────┴──────────────┴──────────────┴──────────┴─────────────┘
```

---

## Metric Definitions

### Calls Listened
Distinct calls where Arpit watched **≥ 50% of the recording** within the selected date range.

- Watching 10 seconds and closing does not count
- The 50% threshold is configurable in admin settings
- Each call counts once regardless of how many times it was rewatched

### Calls with Feedback
Distinct calls within the date range where Arpit left **at least one timestamped comment**.

- A subset of "Calls Listened" — you can only leave feedback on a call you've watched
- If Arpit leaves 5 comments on one call, it still counts as 1 call with feedback

### Comments Left
Total count of individual timestamped comments Arpit posted within the date range.

- Each comment = one comment regardless of length
- Gives a sense of feedback depth vs. breadth

### Team Coverage
```
team_coverage = calls_listened / total_calls_recorded_by_team * 100
```
Shows what percentage of all team calls Arpit reviewed in the period.

---

## Date Range Filter

Matches Gong's timeline UX:

| Option | What it shows |
|--------|--------------|
| Last 7 days | Rolling 7-day window from today |
| Last 30 days | Rolling 30-day window (default) |
| Last Quarter | Current calendar quarter to date |
| This Month | Calendar month to date |
| Custom | From/to date picker — any range |

The filter applies to **when the call was recorded**, not when Arpit watched it.

---

## Tracking "Listened" — How It Works

To accurately know whether Arpit watched a call, the video player sends periodic progress updates to the Sales Brain API while the video is playing.

**`call_views` table:**

```sql
call_views (
  id              uuid primary key,
  call_id         uuid references calls(id),
  user_id         uuid references users(id),
  started_at      timestamptz,        -- when they first opened the call
  last_position_ms bigint,            -- furthest point reached in the video
  duration_ms     bigint,             -- total call duration (denormalized)
  percent_watched numeric generated   -- last_position_ms / duration_ms * 100
)
```

**Player behavior:**
- Every 10 seconds while playing, the player POSTs `{ call_id, position_ms }` to `/api/views/ping`
- On close or pause, a final POST is sent with the current position
- `last_position_ms` is updated to the **furthest** position reached — scrubbing forward does not inflate the number
- If the same user opens the same call again, the existing `call_views` row is updated (upsert on `call_id + user_id`)

**"Listened" query:**

```sql
SELECT COUNT(DISTINCT call_id) AS calls_listened
FROM call_views
WHERE user_id = :arpit_user_id
  AND percent_watched >= 50
  AND started_at BETWEEN :start_date AND :end_date
```

---

## Per Rep Breakdown — Queries

**Calls recorded per rep in period:**
```sql
SELECT rep_id, COUNT(*) AS calls_recorded
FROM calls
WHERE date BETWEEN :start AND :end
GROUP BY rep_id
```

**Calls reviewed per rep (listened by Arpit):**
```sql
SELECT c.rep_id, COUNT(DISTINCT cv.call_id) AS calls_reviewed
FROM call_views cv
JOIN calls c ON c.id = cv.call_id
WHERE cv.user_id = :arpit_user_id
  AND cv.percent_watched >= 50
  AND c.date BETWEEN :start AND :end
GROUP BY c.rep_id
```

**Comments left per rep:**
```sql
SELECT c.rep_id, COUNT(co.id) AS comments_left
FROM comments co
JOIN calls c ON c.id = co.call_id
WHERE co.user_id = :arpit_user_id
  AND co.created_at BETWEEN :start AND :end
GROUP BY c.rep_id
```

**Last review date per rep:**
```sql
SELECT c.rep_id, MAX(cv.started_at) AS last_reviewed
FROM call_views cv
JOIN calls c ON c.id = cv.call_id
WHERE cv.user_id = :arpit_user_id
GROUP BY c.rep_id
```

All four queries are joined in the application layer to build the per-rep table.

---

## Where It Lives in the Dashboard

The Manager Activity section appears on the **Admin home page** — the landing page Arpit sees when he logs into Sales Brain. Reps do not see this section; it is scoped to users with the `admin` role.

Reps see their own version: a simpler view showing "X of your calls have been reviewed, Y comments left on your calls" — no cross-rep visibility.

---

## Future: Coaching Goals

Phase 3 addition — Arpit can set a coaching target:

> "Review at least 2 calls per rep per week"

The dashboard would show a progress bar per rep against the target, and send a Teams reminder on Friday if the target hasn't been met for any rep.
