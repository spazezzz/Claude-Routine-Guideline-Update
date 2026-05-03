# Calendar-Event-Check

> **Runtime target:** Claude Remote Routine (web sandbox)  
> **Constraint:** No Bash / shell / git CLI — all I/O via GitHub MCP connector and WebFetch only.

## Invocation

```
/Calendar-Event-Check
```

No arguments. Runs once per invocation: checks tomorrow's Google Calendar events and optionally sends a LINE notification.

---

## Execution Flow

### Step 0 — Verify GitHub connector

Attempt to read `README.md` from the repository via the GitHub MCP connector.

- Call succeeds → proceed to Step 1.
- Connector returns an error or is unavailable → **abort immediately** and log:
  ```
  [ERROR] GitHub connector unavailable at <ISO-8601 timestamp UTC+7>. Skill aborted.
  ```
  Do not execute any further step.

### Step 1 — Compute tomorrow's date (Asia/Bangkok)

1. Obtain current UTC time from the Claude runtime context.
2. Add **7 hours** to convert to Asia/Bangkok (UTC+7).
3. Add **1 day** to obtain tomorrow.
4. Derive the following values (all in UTC+7):
   - `TOMORROW_DATE` — `YYYY-MM-DD` (e.g. `2026-05-04`)
   - `TIME_MIN` — `YYYY-MM-DDT00:00:00+07:00`
   - `TIME_MAX` — `YYYY-MM-DDT00:00:00+07:00` for the day after tomorrow
   - `DISPLAY_DATE` — Thai short format `DD Mmm BBBB` where BBBB = Gregorian year + 543
     - Thai month abbreviations: ม.ค. ก.พ. มี.ค. เม.ย. พ.ค. มิ.ย. ก.ค. ส.ค. ก.ย. ต.ค. พ.ย. ธ.ค.
     - Example: `04 พ.ค. 2569`

### Step 2 — Refresh Google OAuth access token

Use **WebFetch** (POST) to obtain a fresh access token:

```
POST https://oauth2.googleapis.com/token
Content-Type: application/x-www-form-urlencoded

client_id={GOOGLE_CLIENT_ID}
&client_secret={GOOGLE_CLIENT_SECRET}
&refresh_token={GOOGLE_REFRESH_TOKEN}
&grant_type=refresh_token
```

- HTTP 200 → extract `access_token` from the JSON response body. Store as `ACCESS_TOKEN`.
- HTTP non-200 → **abort** and log:
  ```
  [ERROR] Google token refresh failed at <timestamp>: HTTP {status} — {body}
  ```

### Step 3 — Fetch tomorrow's events from Google Calendar

Use **WebFetch** (GET) to call the Calendar Events list API:

```
GET https://www.googleapis.com/calendar/v3/calendars/{GOOGLE_CALENDAR_ID}/events
    ?timeMin={TIME_MIN}
    &timeMax={TIME_MAX}
    &singleEvents=true
    &orderBy=startTime
    &maxResults=50
Authorization: Bearer {ACCESS_TOKEN}
```

URL-encode the `+` in the timezone offset (`+07:00` → `%2B07%3A00`).

Response handling:
- HTTP 200 → parse JSON. Extract the `items` array.
  - `items` empty or absent → proceed to **Step 4a** (no events).
  - `items` non-empty → proceed to **Step 4b** (events found).
- HTTP non-200 → **abort** and log:
  ```
  [ERROR] Google Calendar API returned {status} at <timestamp>: {body}
  ```

### Step 4a — No events tomorrow

If `items` is empty:

- Log: `[INFO] No events tomorrow ({TOMORROW_DATE}). Skill completed with no action.`
- **Stop. Do not send any LINE notification.**

### Step 4b — Events found — format event list

For each object in `items`, extract:

| Field | Source field | Fallback |
|-------|-------------|----------|
| Title | `summary` | `"(ไม่มีชื่อ)"` |
| Start | `start.dateTime` | `start.date` (all-day) |
| End | `end.dateTime` | `end.date` (all-day) |

Format each event as one line:

- Timed event: `• {title} — {HH:MM} น. – {HH:MM} น.`
- All-day event: `• {title} — ทั้งวัน`

All times must be converted to **Asia/Bangkok (UTC+7)** before display.

If `start.dateTime` ends in `Z` (UTC), add 7 hours. If it carries a numeric offset, convert accordingly.

### Step 5 — Send LINE notification

**Skip this entire step** if `LINE_CHANNEL_ACCESS_TOKEN` or `LINE_TO` is absent from the environment. The skill still completes normally.

Compose the message text:

```
📅 นัดหมายพรุ่งนี้ ({DISPLAY_DATE})

{event line 1}
{event line 2}
…

รวม {N} รายการ
```

Example output:

```
📅 นัดหมายพรุ่งนี้ (04 พ.ค. 2569)

• ประชุมทีม — 09:00 น. – 10:00 น.
• Review PR — 14:00 น. – 15:30 น.
• วันหยุดพักผ่อน — ทั้งวัน

รวม 3 รายการ
```

Send via **WebFetch** (POST):

```
POST https://api.line.me/v2/bot/message/push
Content-Type: application/json
Authorization: Bearer {LINE_CHANNEL_ACCESS_TOKEN}

{
  "to": "{LINE_TO}",
  "messages": [
    {
      "type": "text",
      "text": "{message}"
    }
  ]
}
```

Handling:
- HTTP 200 → log `[INFO] LINE notification sent successfully at <timestamp>.`
- HTTP non-200 → log `[ERROR] LINE API returned {status}: {body}`. **Do not retry.**
- WebFetch / network error → log `[ERROR] LINE WebFetch failed: {error}`. Do not block skill completion.

---

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `GOOGLE_CALENDAR_ID` | Yes | Calendar ID — use `primary` for the account's main calendar, or a full address like `name@group.calendar.google.com` |
| `GOOGLE_CLIENT_ID` | Yes | OAuth 2.0 client ID from Google Cloud Console |
| `GOOGLE_CLIENT_SECRET` | Yes | OAuth 2.0 client secret |
| `GOOGLE_REFRESH_TOKEN` | Yes | Long-lived offline refresh token (obtained once via OAuth consent flow) |
| `LINE_CHANNEL_ACCESS_TOKEN` | No | LINE Messaging API long-lived channel access token |
| `LINE_TO` | No | Recipient — LINE User ID (`U…`) or Group ID (`C…`) |
| `GITHUB_REPO_OWNER` | Yes | GitHub username / organisation owning this repository |
| `GITHUB_REPO_NAME` | Yes | Repository name (e.g. `claude-routine-guideline-update`) |

---

## Timezone

All date/time values use **Asia/Bangkok (UTC+7)**.

Conversion rule (no system clock / no shell):
1. Obtain the current UTC epoch from the Claude runtime context.
2. Add **25 200 seconds** (7 × 3 600) to get Bangkok epoch.
3. Derive wall-clock date/time from that adjusted epoch.
4. Always append `+07:00` when constructing RFC 3339 strings for API calls.

---

## Guardrails Summary

| Rule | Behaviour |
|------|-----------|
| No Bash / shell / git CLI | GitHub MCP connector + WebFetch only |
| GitHub connector unavailable | Abort entire run; log timestamped error |
| `LINE_*` env vars absent | Skip LINE step; skill completes normally |
| LINE API HTTP non-200 | Log error (status + body); no silent retry |
| No events tomorrow | Exit quietly; no LINE message sent |
| Google token refresh failure | Abort; log HTTP status and response body |
| Google Calendar API non-200 | Abort; log HTTP status and response body |
