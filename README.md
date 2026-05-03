# Claude-Routine-Guideline-Update

Automated clinical guideline monitoring and publication via Claude Remote Routine.

The skill researches newly published Thai and international clinical guidelines, analyses them through a Thai-context lens, writes a structured brief, and publishes it to this repository — all without Bash, shell, or git CLI.

---

## Repository Structure

```
.claude/
  skills/
    Claude-Routine-Guideline-Update/
      skill.md              # Full skill instructions (read by Claude)
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

## Running in Claude Web Routine

### Prerequisites

| Requirement | Details |
|-------------|---------|
| Claude account | claude.ai with Routine (scheduled tasks) access |
| GitHub connector | Connected in Claude settings — grants MCP read/write to this repo |
| LINE Messaging API | Optional — only needed for LINE notifications |

### 1. Connect the GitHub Connector

1. Open **claude.ai → Settings → Connectors** (or the equivalent Integrations panel).
2. Click **Add connector → GitHub**.
3. Authorise Claude to access this repository (`{your-org}/claude-routine-guideline-update`).
4. Confirm the connector shows **Connected** status.

The skill uses the GitHub connector for:
- Reading `reference/sources.md` and `reference/perspectives.md`
- Committing generated articles to `articles/`
- Updating `reference/sources.md` with new source entries

If the connector is unavailable at run time, the skill aborts and logs an error — it will **not** fall back to shell commands.

### 2. Set Environment Variables

Copy `env.example` to `.env` (local development only — never commit `.env`).

For Claude Web Routine, set variables in **Settings → Routine → Environment**:

| Variable | Required | Value |
|----------|----------|-------|
| `GITHUB_REPO_OWNER` | Yes | Your GitHub username or organisation name |
| `GITHUB_REPO_NAME` | Yes | `claude-routine-guideline-update` (or your fork name) |
| `LINE_CHANNEL_ACCESS_TOKEN` | No | From LINE Developers Console → Messaging API channel |
| `LINE_TO` | No | LINE User ID (`U...`) or Group ID (`C...`) |

### 3. Create the Routine

1. In Claude Web, open **Routines → New Routine**.
2. Set **Skill**: `Claude-Routine-Guideline-Update`
3. Set **Schedule** (choose one):
   - Weekly brief: `0 8 * * 1` (Monday 08:00 Bangkok time)
   - Bi-weekly: `0 8 1,15 * *` (1st and 15th of each month)
   - Daily scan: `0 7 * * *` (07:00 daily — high volume, use with care)
4. Set **Timezone**: `Asia/Bangkok`
5. Optionally pass a `TOPIC` argument to narrow the research scope.
6. Save and enable the Routine.

### 4. Verify a Test Run

1. Trigger a manual run from the Routine dashboard.
2. Check this repository — a new file should appear under `articles/`.
3. Check `reference/sources.md` for the appended source table.
4. If LINE is configured, verify the push message was received.

### 5. Customise the Analysis Framework

Edit `reference/perspectives.md` to adjust how guidelines are analysed — e.g. add a dimension for cost-effectiveness or modify the Thai drug availability sources. The skill reads this file fresh on each run.

---

## Skill Guardrails

| Guardrail | Behaviour |
|-----------|-----------|
| No Bash/shell/git CLI | Skill uses only GitHub MCP connector + WebFetch |
| Unverifiable guideline | Excluded from article; logged in `sources.md` under Unverified Attempts |
| GitHub connector unavailable | Abort run, log error, do not publish |
| `LINE_*` env vars absent | Skip LINE step; commit proceeds normally |
| LINE API HTTP non-200 | Log error with status + body; no silent retry |
| Fabricated content | Strictly forbidden — every fact must cite a primary source |

---

## Output Format

Each run produces `articles/DD-MM-YYYY-brief.md` with:

- **TL;DR** — 3 bullets (also sent via LINE)
- **Background** — context and scope
- **What Changed** — delta from previous guideline
- **Thai Context Analysis** — applicability, drug availability, implementation barriers
- **Action Items** — role-specific table (Clinician / Pharmacist / Policy)
- **References** — numbered, primary sources only

---

## LINE Notification Format

```
📋 Guideline Update — DD-MM-YYYY

• [TL;DR bullet 1]
• [TL;DR bullet 2]
• [TL;DR bullet 3]

🔗 https://github.com/{owner}/{repo}/blob/{SHA}/articles/DD-MM-YYYY-brief.md
```

The permalink points to the exact commit SHA so the link remains stable even if the file is later updated.

---

## Local Development

This skill is designed for Claude Remote Routine. For local testing:

1. Copy `env.example` → `.env` and fill in values.
2. Open a Claude Code session in this directory.
3. Run `/Claude-Routine-Guideline-Update [TOPIC]` in the Claude Code terminal.
4. The skill will execute but will use your local GitHub MCP server instead of the cloud connector.

> Note: Local runs still use WebFetch for LINE — ensure `LINE_*` vars are set if you want to test notifications.

---

## Contributing

- To improve the analysis framework, edit `reference/perspectives.md` and submit a PR.
- To change the article template structure, edit `.claude/skills/Claude-Routine-Guideline-Update/skill.md` (Step 4 section).
- Issues and PRs welcome.
