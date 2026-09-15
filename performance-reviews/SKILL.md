---
name: performance-reviews
description: "Build standalone HTML performance-review briefing documents for a manager's direct reports, pulling real Snowflake activity data (pipeline wins/losses, scale/enablement activity, AI tool usage, call presence) plus manager-supplied narrative and Workday feedback, anonymized and free of promotion language. Use when: a manager wants to generate H1/H2 performance review documents for their team, one file per person, in a consistent shareable format. Triggers: performance review briefing, build performance reviews for my team, H1 review document, H2 review document, team performance review HTML."
---

# Performance Review Briefing Builder

Generates one self-contained, shareable HTML performance-review briefing per team member, combining verifiable Snowflake activity data with manager-supplied narrative and anonymized Workday peer feedback.

## Non-negotiable policies (apply to every report, every person)

1. **No names of other people ever appear in an individual's report.** All stakeholder/peer/customer feedback quotes are anonymized — strip names, titles that identify the speaker, and any phrasing that would let the reader infer who said it. If a quote references a *colleague* (not the subject), remove or genericize that reference too (e.g. "a peer who joined the same month" instead of naming them).
2. **No promotion or leveling recommendations.** Do not write anything that reads as "should be promoted," "ready for the next level," etc., even if source feedback says it.
3. **No fabricated attribution.** Only credit a named account/use-case outcome to this person if the underlying data (comments, activity records) actually documents *their* individual contribution — not just formal team-list membership. If the data doesn't support individual attribution, say so plainly in a callout rather than inventing a story. This applies especially to `DIM_USE_CASE`-style team-membership arrays: team membership alone is not evidence of impact.
4. **Ratings are shown as names, never numbers** (e.g. "High Impact", not "3").
5. **IC level is shown as their existing level only** — do not editorialize about being "ready to operate at a higher level."
6. **Every report ends with a "Save as PDF / Print" button** (`window.print()`), hidden via `@media print`.
7. **Charts must be inline SVG, not an external charting library.** These reports are meant to be opened as a plain local file and shared outside any sandboxed viewer — a vendored `/libs/chart.js`-style dependency will silently fail to load and leave a blank box. Build small inline `drawBarChart`/`drawLineChart` JS helpers instead (see Step 5).

## Workflow

### Step 1: Collect scope

Ask the manager for:
- **Team roster**: names (and emails, if known — needed for CoCo/AI-usage lookups)
- **Review period**: fiscal half start/end dates (confirm the org's fiscal year start — do not assume Feb 1)
- **Output location**: a durable local directory (not a temp/scratch path)
- Whether this is an **H1-only** report or should also include an **"Areas of Focus for H2"** section

**⚠️ STOP**: Confirm roster and dates before querying anything.

### Step 2: Collect manual inputs per person

Some of the most valuable content — anonymized peer feedback and the manager's own narrative — does not live in a queryable table. **Explicitly ask for these before drafting**, rather than leaving a "pending manager input" placeholder:

1. **Peer/stakeholder feedback export.** Ask the manager to attach their org's feedback-collection export for each person (e.g. a Workday "View Feedback Received" spreadsheet, 360-review export, or similar). If provided, read it (see `references/reading_feedback_exports.md` if the format is unfamiliar) and extract 3-8 anonymized, substantive quotes per person — paraphrase or lightly edit to remove names/identifying detail while preserving the substance.
2. **Manager narrative.** Ask for anything not in the data: customer outcomes worth naming, specific coaching feedback already delivered, context on leave/ramp/territory changes, and any H2 focus themes the manager already has in mind for specific people.
3. **Team-wide policy language for H2 Focus** (if doing H2 focus sections). Ask whether there are standing asks the manager wants repeated across the team (e.g. a tool-adoption push, a documentation/specialist-comments ask, an IC-level-specific mentoring expectation, a scale-assets conversion ask). Do not invent this — get it from the manager or leave it out.

**⚠️ STOP**: Do not proceed to data pulls until at least the feedback export (if one exists) has been provided or the manager has confirmed there isn't one.

### Step 3: Identify data sources

The specific tables below are examples from one AMS Expansion DE team — **verify equivalents exist in the peer's environment before reusing these names**, since org/team structures differ:

| Signal | Example source | Notes |
|---|---|---|
| Attainment / win counts / go-lives / accounts engaged | `SALES.SE_REPORTING.SE_SPECIALIST_METRICS_DAILY` | Filter `EMPLOYEE_NAME`, `TIME_PERIOD_TYPE = 'Day'`, date range |
| Pipeline win/loss detail, named accounts | `MDM.MDM_INTERFACES.DIM_USE_CASE` | `USE_CASE_TEAM_NAME_LIST` (array) for team membership; `SPECIALIST_COMMENTS`, `SE_COMMENTS`, `IMPLEMENTATION_COMMENTS`, `NEXT_STEPS` for narrative — **run `DESCRIBE TABLE` first**, do not assume column names (e.g. it is `SPECIALIST_COMMENTS`, not `UC_COMMENTS`) |
| Scale/enablement activity (docs, sessions, tickets) | Team's Jira project | Filter by project ID and assignee |
| AI/CoCo tool usage | `SNOWSCIENCE.ENGINEERING_SYSTEMS.AI_USAGE_DAILY_TOKEN_COST` | Filter by `EMAIL`, date range |
| Customer call presence | A Gong-derived view/table | If the canonical rolled-up view isn't grantable, a raw call-summary table with name-mention matching is a workable fallback — see Step 4 note |

Use `snowflake_object_search` to find the peer's actual equivalents if unknown. Confirm column names via `DESCRIBE TABLE` before writing queries — do not guess.

### Step 4: Query per person

For each team member, for the confirmed date window:
1. KPI/attainment rollup
2. Named pipeline outcomes — **only include named-account bullets if the comment/narrative fields (not just team-list membership) actually document this person's individual actions.** Otherwise fall back to an aggregate, honestly-worded callout (see Policy #3).
3. Scale/enablement activity
4. AI tool usage, with a short "what's working / opportunity" read
5. Call presence, if a Gong-equivalent source exists

**Gong name-matching fallback (if no clean canonical view is grantable):** match on `"<Full Name> from Snowflake"` in a call-summary/brief field, or match on **surname alone** for common first names to avoid false positives (a bare first name like "Ryan" can match thousands of irrelevant customer-side mentions). Always sanity-check a sample of matches before trusting the count.

### Step 5: Generate one HTML file per person

Follow the `html-authoring` skill's conventions (sandboxed CSP, `snowflake-source` meta tag, `snowflake-report-metadata` JSON block with dataSources/sections). Standard section order:

1. Header + badges (rating-as-name, IC level, any leave/ramp context)
2. H1 headline KPIs
3. Pipeline win/loss detail (chart + table)
4. Customer & technical outcomes
5. Scale & enablement
6. Stakeholder & customer feedback (anonymized quotes from Step 2)
7. Call analysis (if available)
8. AI tool adoption
9. Areas of Focus & Personal Development for H2 (if requested) — team-wide policy bullets from Step 2, plus person-specific themes from the manager narrative and the data
10. Sources note (list every table queried + "manager H1 review narrative")

**Charts:** use inline SVG helpers, not an external library:
```javascript
function drawBarChart(id, categories, series){ /* build <svg> string, fill="currentColor" for dark/light mode, set el.innerHTML */ }
function drawLineChart(id, labels, values, color){ /* handle null values as gaps, build path with M/L commands */ }
```

**Export button** in every file:
```html
<div class="export-bar"><button class="export-btn" id="exportPdfBtn" type="button">Save as PDF / Print</button></div>
<script>document.getElementById('exportPdfBtn').addEventListener('click', function () { window.print(); });</script>
```
```css
@media print { .export-bar { display: none !important; } body { max-width: 100%; } }
```

Save each file as `<FirstName>_<LastName>_<Period>_Performance_Review.html` in the confirmed output directory.

### Step 6: Build an index page

One landing HTML page with a card per person linking to their file, so the manager has a single link to open the whole set.

### Step 7: Iterate

Performance review content gets edited a lot — expect (and welcome) many rounds of surgical, per-person edits ("remove this bullet for X," "add this quote for Y"). When editing:
- Grep the target file for the exact text before editing, don't assume line numbers are stable across edits.
- After removing a bullet, check whether anything else in the file *references* it (a later bullet that says "as already acknowledged above," a nav/TOC entry, a metadata `sections` array) and clean those up too.
- After any batch edit across multiple files, spot-check that shared CSS classes (like a `.quote` style) exist in every file — a missing style class is a common silent bug that looks fine in the raw HTML but renders wrong.

## Stopping Points

- ✋ Step 1: Roster and date window confirmed
- ✋ Step 2: Manual inputs (feedback export, manager narrative, policy language) collected or explicitly declined
- ✋ Before Step 5 generation: confirm the manager is happy with the data found per person, especially any named-account outcomes, before drafting prose

## Output

One HTML file per team member plus an index page, in the manager's chosen output directory.
