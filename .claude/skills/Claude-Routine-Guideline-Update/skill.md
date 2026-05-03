# Claude-Routine-Guideline-Update

> **Runtime target:** Claude Remote Routine (web sandbox)  
> **Constraint:** No Bash / shell / git CLI — all I/O via GitHub MCP connector and WebFetch only.

## Invocation

```
/Claude-Routine-Guideline-Update [TOPIC]
```

- `TOPIC` (optional) — e.g. "hypertension", "diabetes T2", "sepsis". When omitted, scan all major Thai + international guideline publishers for the past 30 days.

---

## Execution Flow

### Step 1 — Research new guidelines

1. Use **WebSearch** to find guideline updates published within the last 30 days:
   - Thai sources: Royal College of Physicians of Thailand, Thai Hypertension Society, CCRT, NHSO clinical standards, กรมการแพทย์, สมาคมวิทยาศาสตร์สุขภาพที่เกี่ยวข้อง
   - International sources: AHA/ACC, ESC, WHO, NICE, Lancet, NEJM, JAMA, BMJ, IDSA, ISTH, and any domain-specific body relevant to TOPIC.
2. For each candidate result **fetch the primary source URL** via **WebFetch** to confirm:
   - Publication date is within the research window.
   - The document is an actual guideline / position statement / consensus (not news or opinion).
   - Full text or official summary is accessible.
3. **Abort any guideline that cannot be verified** at the primary source. Log the attempted URL under `## Unverified Attempts` in `reference/sources.md`.
4. Collect 3–10 verified guidelines. Aim for a balanced mix of Thai and international when TOPIC allows.

### Step 2 — Populate reference/sources.md

Read the current `reference/sources.md` via the GitHub MCP connector (get file contents), then append or update the `## Sources — DD-MM-YYYY` section using the template:

```markdown
## Sources — DD-MM-YYYY

| # | Title | Organisation | Date | URL | Key Change |
|---|-------|-------------|------|-----|------------|
| 1 | ... | ... | ... | ... | ... |
```

Rules:
- All URLs must be primary-source (official society / publisher page).
- `Key Change` = one sentence max summarising the delta from previous version.
- Do **not** fabricate entries.

Commit the updated file via GitHub MCP:
- path: `reference/sources.md`
- branch: `main`
- message: `refs: update sources DD-MM-YYYY`

### Step 3 — Analyse against Thai context

Load `reference/perspectives.md` via GitHub MCP. Apply each perspective dimension listed there to every verified guideline:

1. **Delta analysis** — What changed vs the previous version of this guideline (or vs the current Thai standard of care)?
2. **Thai applicability** — Drug availability, reimbursement under UC/CSMBS/SSO, infrastructure requirements, cultural factors.
3. **Implementation gap** — Identify the single most important barrier to adoption in Thai hospitals (tertiary vs community).
4. **Risk/benefit for Thai patients** — Note any ethnicity-specific data if cited in the guideline.

Record findings in working memory (do not commit this intermediate analysis).

### Step 4 — Write article

Generate a well-structured Markdown article. Save as `articles/DD-MM-YYYY-brief.md`.

Required sections in order:

```markdown
# [TOPIC] Guideline Update — DD-MM-YYYY

> **TL;DR**
> - [bullet 1 — most important change]
> - [bullet 2]
> - [bullet 3]

## Background
...

## What Changed
...

## Thai Context Analysis
...

## Action Items

| Role | Action | Priority |
|------|--------|----------|
| Clinician | ... | High/Med/Low |
| Hospital pharmacist | ... | ... |
| Policy / NHSO | ... | ... |

## References
<!-- numbered, matching sources.md -->
```

Writing rules:
- Language: Thai for body text, English for proper nouns, drug names, acronyms.
- Reading level: suitable for Thai internist / GP.
- No hallucinated statistics. Every numeric claim must cite a reference number.
- Action Items table must be specific and actionable, not generic.

### Step 5 — Commit article via GitHub MCP

1. Check GitHub connector availability — if connector returns an error on any read/write call, **abort** and log:
   ```
   [ERROR] GitHub connector unavailable at <ISO timestamp>. Article not published.
   ```
2. Commit the article file:
   - **owner / repo**: from env `GITHUB_REPO_OWNER` / `GITHUB_REPO_NAME`
   - **branch**: `main`
   - **path**: `articles/DD-MM-YYYY-brief.md`
   - **message**: `brief: (TOPIC) DD-MM-YYYY`
3. After commit succeeds, retrieve the commit SHA from the API response.
4. Construct permalink:
   ```
   https://github.com/{GITHUB_REPO_OWNER}/{GITHUB_REPO_NAME}/blob/{SHA}/articles/DD-MM-YYYY-brief.md
   ```

### Step 6 — Send LINE notification (optional)

Skip this step entirely if `LINE_CHANNEL_ACCESS_TOKEN` or `LINE_TO` is absent from the environment.

If env vars are present, send via **WebFetch** (POST):

```
POST https://api.line.me/v2/bot/message/push
Content-Type: application/json
Authorization: Bearer {LINE_CHANNEL_ACCESS_TOKEN}

{
  "to": "{LINE_TO}",
  "messages": [
    {
      "type": "text",
      "text": "📋 Guideline Update — DD-MM-YYYY\n\n• {TL;DR bullet 1}\n• {TL;DR bullet 2}\n• {TL;DR bullet 3}\n\n🔗 {permalink}"
    }
  ]
}
```

Handling:
- HTTP 200 → log success.
- HTTP non-200 → log `[ERROR] LINE API returned {status}: {body}`. **Do not retry silently.**
- Network / WebFetch error → log and continue. Do not block the run.

---

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `GITHUB_REPO_OWNER` | Yes | GitHub username / org that owns the target repo |
| `GITHUB_REPO_NAME` | Yes | Target repository name |
| `LINE_CHANNEL_ACCESS_TOKEN` | No | LINE Messaging API channel access token |
| `LINE_TO` | No | LINE user ID or group ID to push message to |

---

## Timezone

All date strings (DD-MM-YYYY) and timestamps use **Asia/Bangkok (UTC+7)**.

When computing the current date:
1. Fetch current UTC time (available in Claude runtime context).
2. Add 7 hours.
3. Format as `DD-MM-YYYY`.

---

## Guardrails Summary

| Rule | Behaviour |
|------|-----------|
| No Bash/shell/git CLI | Use only GitHub MCP + WebFetch |
| GitHub connector down | Abort entire run, log error |
| LINE env vars absent | Skip LINE step, still commit |
| LINE API non-200 | Log error, no silent retry |
| Unverifiable guideline | Exclude from article, log in sources.md |
| Fabricated content | Strictly forbidden — cite or exclude |
