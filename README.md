# Claude-Routine-Guideline-Update

Automated clinical guideline monitoring and publication via Claude Remote Routine.

The skill researches newly published Thai and international clinical guidelines, analyses them through a Thai-context lens, writes a structured brief, and publishes it to this repository — all without Bash, shell, or git CLI.

This repository also hosts the **Calendar-Event-Check** skill, which checks Google Calendar for tomorrow's events and delivers a LINE notification — likewise without any shell access.

---

## Repository Structure

```
.claude/
  skills/
    Claude-Routine-Guideline-Update/
      skill.md              # Guideline-update skill instructions
    Calendar-Event-Check/
      skill.md              # Calendar event check skill instructions
reference/
  sources.md               # Registry of verified guideline sources (auto-updated)
  perspectives.md          # Thai-context analysis framework (edit to customise)
articles/
  DD-MM-YYYY-brief.md      # Generated briefs (one per run)
README.md
env.example
.gitignore
```

---

## Skill 1 — Claude-Routine-Guideline-Update

Researches, analyses, and publishes Thai & international clinical guideline updates.

See the [skill instructions](.claude/skills/Claude-Routine-Guideline-Update/skill.md) for the full execution flow.

### Running in Claude Web Routine

#### Prerequisites

| Requirement | Details |
|-------------|--------|
| Claude account | claude.ai with Routine (scheduled tasks) access |
| GitHub connector | Connected in Claude settings — grants MCP read/write to this repo |
| LINE Messaging API | Optional — only needed for LINE notifications |

#### 1. Connect the GitHub Connector

1. Open **claude.ai → Settings → Connectors** (or the equivalent Integrations panel).
2. Click **Add connector → GitHub**.
3. Authorise Claude to access this repository (`{your-org}/claude-routine-guideline-update`).
4. Confirm the connector shows **Connected** status.

If the connector is unavailable at run time, the skill aborts and logs an error — it will **not** fall back to shell commands.

#### 2. Set Environment Variables

Copy `env.example` to `.env` (local development only — never commit `.env`).

For Claude Web Routine, set variables in **Settings → Routine → Environment**:

| Variable | Required | Value |
|----------|----------|-------|
| `GITHUB_REPO_OWNER` | Yes | Your GitHub username or organisation name |
| `GITHUB_REPO_NAME` | Yes | `claude-routine-guideline-update` (or your fork name) |
| `LINE_CHANNEL_ACCESS_TOKEN` | No | From LINE Developers Console → Messaging API channel |
| `LINE_TO` | No | LINE User ID (`U...`) or Group ID (`C...`) |

#### 3. Create the Routine

1. In Claude Web, open **Routines → New Routine**.
2. Set **Skill**: `Claude-Routine-Guideline-Update`
3. Set **Schedule** (choose one):
   - Weekly brief: `0 8 * * 1` (Monday 08:00 Bangkok time)
   - Bi-weekly: `0 8 1,15 * *` (1st and 15th of each month)
   - Daily scan: `0 7 * * *` (07:00 daily — high volume, use with care)
4. Set **Timezone**: `Asia/Bangkok`
5. Optionally pass a `TOPIC` argument to narrow the research scope.
6. Save and enable the Routine.

#### 4. Verify a Test Run

1. Trigger a manual run from the Routine dashboard.
2. Check this repository — a new file should appear under `articles/`.
3. Check `reference/sources.md` for the appended source table.
4. If LINE is configured, verify the push message was received.

---

## Skill 2 — Calendar-Event-Check

Checks Google Calendar for tomorrow's events and sends a LINE notification if any are found. If no events exist, the skill exits silently.

See the [skill instructions](.claude/skills/Calendar-Event-Check/skill.md) for the full execution flow.

### Running in Claude Web Routine

#### Prerequisites

| Requirement | Details |
|-------------|--------|
| Claude account | claude.ai with Routine (scheduled tasks) access |
| GitHub connector | Connected in Claude settings — used for connectivity verification |
| Google Cloud project | OAuth 2.0 credentials with Calendar API enabled |
| Google Calendar | At least one calendar accessible by the OAuth account |
| LINE Messaging API | Optional — skip LINE step if absent |

#### 1. Connect the GitHub Connector

1. Open **claude.ai → Settings → Connectors**.
2. Click **Add connector → GitHub** and authorise access to this repository.
3. Confirm the connector shows **Connected** status.

The skill uses this connector as a liveness check at startup. If unavailable, the skill aborts rather than running in a degraded state.

#### 2. Obtain a Google OAuth Refresh Token

This is a one-time setup. The Routine uses the refresh token to generate short-lived access tokens on every run.

1. Go to [Google Cloud Console](https://console.cloud.google.com/) → **APIs & Services → Credentials**.
2. Create an **OAuth 2.0 Client ID** (type: Desktop app or Web app).
3. Enable the **Google Calendar API** in your project.
4. Run the OAuth consent flow once locally (e.g. with `oauth2l` or a small Python script using `google-auth-oauthlib`) requesting the scope `https://www.googleapis.com/auth/calendar.readonly`.
5. Save the returned `refresh_token` — this is long-lived and only needs to be generated once.

#### 3. Set Environment Variables

For Claude Web Routine, set variables in **Settings → Routine → Environment**:

| Variable | Required | Value |
|----------|----------|-------|
| `GOOGLE_CALENDAR_ID` | Yes | `primary` or a full calendar address |
| `GOOGLE_CLIENT_ID` | Yes | OAuth client ID from Google Cloud Console |
| `GOOGLE_CLIENT_SECRET` | Yes | OAuth client secret |
| `GOOGLE_REFRESH_TOKEN` | Yes | Long-lived refresh token from the consent flow |
| `GITHUB_REPO_OWNER` | Yes | Your GitHub username or organisation name |
| `GITHUB_REPO_NAME` | Yes | `claude-routine-guideline-update` (or your fork name) |
| `LINE_CHANNEL_ACCESS_TOKEN` | No | LINE Messaging API long-lived channel access token |
| `LINE_TO` | No | LINE User ID (`U...`) or Group ID (`C...`) |

For local development, copy `env.example` to `.env` and fill in the values. **Never commit `.env`.**

#### 4. Create the Routine

1. In Claude Web, open **Routines → New Routine**.
2. Set **Skill**: `Calendar-Event-Check`
3. Set **Schedule** — recommended: `0 21 * * *` (21:00 nightly — checks the next day's calendar at 9 PM Bangkok time)
4. Set **Timezone**: `Asia/Bangkok`
5. Save and enable the Routine.

Suggested schedules:

| Use case | Cron | Bangkok time |
|----------|------|--------------|
| Evening reminder (recommended) | `0 21 * * *` | Every day 21:00 |
| Morning check | `0 6 * * *` | Every day 06:00 |
| Weekdays only | `0 21 * * 1-5` | Mon–Fri 21:00 |

#### 5. Verify a Test Run

1. Trigger a manual run from the Routine dashboard.
2. If events exist for tomorrow, you should receive a LINE message within seconds.
3. If no events exist, the skill completes silently — check the Routine run log for `[INFO] No events tomorrow`.
4. Check the run log for any `[ERROR]` lines if the notification was not received.

#### 6. Skill Guardrails

| Guardrail | Behaviour |
|-----------|-----------|
| No Bash / shell / git CLI | GitHub MCP connector + WebFetch only |
| GitHub connector unavailable | Abort run; log timestamped error |
| `LINE_*` env vars absent | Skip LINE step; skill completes normally |
| LINE API HTTP non-200 | Log error (status + body); no silent retry |
| No events tomorrow | Exit quietly; no LINE message sent |
| Google token refresh failure | Abort; log HTTP status and response body |
| Google Calendar API non-200 | Abort; log HTTP status and response body |

---

## Skill 1 — Guardrails

| Guardrail | Behaviour |
|-----------|----------|
| No Bash/shell/git CLI | Skill uses only GitHub MCP connector + WebFetch |
| Unverifiable guideline | Excluded from article; logged in `sources.md` under Unverified Attempts |
| GitHub connector unavailable | Abort run, log error, do not publish |
| `LINE_*` env vars absent | Skip LINE step; commit proceeds normally |
| LINE API HTTP non-200 | Log error with status + body; no silent retry |
| Fabricated content | Strictly forbidden — every fact must cite a primary source |

---

## Skill 1 — Output Format

Each run produces `articles/DD-MM-YYYY-brief.md` with:

- **TL;DR** — 3 bullets (also sent via LINE)
- **Background** — context and scope
- **What Changed** — delta from previous guideline
- **Thai Context Analysis** — applicability, drug availability, implementation barriers
- **Action Items** — role-specific table (Clinician / Pharmacist / Policy)
- **References** — numbered, primary sources only

---

## LINE Notification Formats

**Skill 1 — Guideline Update:**
```
📋 Guideline Update — DD-MM-YYYY

• [TL;DR bullet 1]
• [TL;DR bullet 2]
• [TL;DR bullet 3]

🔗 https://github.com/{owner}/{repo}/blob/{SHA}/articles/DD-MM-YYYY-brief.md
```

**Skill 2 — Calendar Event Check:**
```
📅 นัดหมายพรุ่งนี้ (DD Mmm BBBB)

• {Event title} — HH:MM น. – HH:MM น.
• {All-day event} — ทั้งวัน

รวม N รายการ
```

---

## Local Development

Both skills are designed for Claude Remote Routine. For local testing:

1. Copy `env.example` → `.env` and fill in values.
2. Open a Claude Code session in this directory.
3. Run `/Calendar-Event-Check` or `/Claude-Routine-Guideline-Update [TOPIC]` in the Claude Code terminal.
4. Skills execute using your local GitHub MCP server and WebFetch for external APIs.

> Note: Local runs still call LINE and Google APIs via WebFetch — ensure all env vars are set if you want to test the full flow.

---

## Contributing

- To improve the analysis framework, edit `reference/perspectives.md` and submit a PR.
- To change the article template, edit `.claude/skills/Claude-Routine-Guideline-Update/skill.md` (Step 4 section).
- To adjust the calendar notification format, edit `.claude/skills/Calendar-Event-Check/skill.md` (Step 5 section).
- Issues and PRs welcome.
