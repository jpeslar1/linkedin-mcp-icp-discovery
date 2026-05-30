# ICP Discovery — Seed → Behavioral Lookalike → Tiered List

A Claude Code prompt that rebuilds a B2B target list from scratch. Takes a CSV of your best closed-won customers, extracts their composite behavioral pattern via the LinkedIn MCP (Zevari), pulls a wide firmographic candidate set from Apollo, then live-confirms and behaviorally-scores every candidate against the seed pattern. Output is a small, tiered list (typically 100-300 contacts) where every person has been pattern-matched against your top buyers.

The reason this prompt exists: every other ICP / lookalike tool stops at firmographics. "VPs at Series B SaaS, 50-200 employees" returns 50,000 candidates and you have no idea which 500 actually look like your best customers. The buyer behavior — what they post about, who they follow, what they engage with — is the actual lookalike signal. That signal lives on LinkedIn and Zevari is how you read it programmatically.

---

## Overview

1. Load seed customer CSV from `[SEED_CSV_PATH]`.
2. Live-confirm each seed via Zevari `linkedin_get_profile` + `linkedin_get_company`.
3. Extract behavioral profile per seed via Zevari `agents_behavioral_profile`.
4. Aggregate into a composite buyer profile (title cluster, content patterns, follow patterns, signal patterns).
5. Build Apollo filter set from the composite. Pull up to 2,000 raw candidates.
6. Live-confirm every candidate via Zevari.
7. Score: ICP fit + behavioral similarity + recency + engagement-with-you.
8. Bucket: TOP_TIER / TIER_2 / TIER_3 / DQ.
9. Write back to Zevari lists, Pipedrive contacts, CSV export.
10. Slack summary with behavioral pattern findings.
11. Reflection.

Run cadence: every 90 days. NOT a scheduled task. The seed customer mix shifts as you close more deals, so the lookalike target shifts. A list built off Q1's customers is wrong by Q3.

Batch shape: this is a one-shot run, not a queue drain. Expect ~30-60 minutes end-to-end depending on Apollo candidate count and Zevari rate limits.

---

## ERROR & APPROVAL NOTIFICATIONS

If anything stops the task or needs approval, send to `[ALERT_CHANNEL]` via Chrome (bash sandbox blocks `hooks.slack.com`):

1. Call `mcp__Claude_in_Chrome__tabs_context_mcp` with `createIfEmpty: true`
2. If the tab is on `chrome://`, navigate to `https://google.com` first
3. Run via `mcp__Claude_in_Chrome__javascript_tool`:

```js
fetch('[SLACK_WEBHOOK_URL]', {
  method: 'POST',
  mode: 'no-cors',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({
    text: "<@[YOUR_SLACK_USER_ID]> ⚠️ *ICP Discovery — Action Needed*\n\n<error or approval ask>"
  })
}).then(r => ({status: r.status, type: r.type})).catch(e => ({error: e.message}))
```

`{status: 0, type: "opaque"}` = success.

---

## CONNECTORS

- **Zevari (LinkedIn MCP):** `mcp__[ZEVARI_MCP_ID]` — live profile, company state, behavioral profile, ICP scoring, list creation. Load-bearing in every step.
- **Apollo:** `mcp__[APOLLO_MCP_ID]` — firmographic candidate pull from Step 5 filters
- **Pipedrive:** `mcp__pipedrive__` — seed customer source (closed-won), final tiered contacts get written here
- **Chrome:** `mcp__Claude_in_Chrome__` — Slack webhook (summary + alerts)
- **WebSearch:** optional company validation (funding announcements, hiring signals) when Zevari company intel is thin

See `connectors.md`.

---

## STEP 0 — LOAD SEED CUSTOMERS

Read `[SEED_CSV_PATH]`. Minimum required columns:

- `linkedin_url`
- `company_name`
- `title_at_purchase`

Optional but useful columns:
- `closed_won_at` (date)
- `deal_value_usd`
- `still_a_customer` (bool)
- `pipedrive_deal_id`

Hard requirement: **at least 10 seed customers.** If fewer than 10 rows, stop and ping `[ALERT_CHANNEL]` with `"seed too small: N rows, need 10+. Sample composite will be noise."` Do not proceed.

Soft requirement: 18-25 seeds is where the composite stabilizes. If between 10-17, note it in the Slack summary so the user knows the composite confidence is lower.

If `still_a_customer` is present, drop any row where it's false (churned customers are not the lookalike target you want).

---

## STEP 1 — LIVE-CONFIRM EACH SEED

For each seed, run in parallel (max 3 concurrent Zevari calls):

1. `linkedin_get_profile` on `linkedin_url` — extract current `title`, current `company`, current `company_url`, `headline`, `location`, `started_at_current_role`.
2. `linkedin_get_company` on `current_company_url` — extract size, industry, funding stage.

Important: many of your best buyers got promoted since they bought. If the seed's `title_at_purchase` differs from their current title, flag it. This tells you whether your buyer typically gets promoted after using your product (lookalike signal — find people in the *pre-promotion* role you'd expect to grow into the *post-promotion* role).

If Zevari rate-limits during seed enrichment → pause 5 min, resume. Do not proceed without all seeds confirmed. The seed step is the foundation; you cannot skip leads here.

---

## STEP 2 — EXTRACT BEHAVIORAL PROFILE PER SEED

For each seed, call Zevari `agents_behavioral_profile` on `linkedin_url`. Extract:

- **Title trajectory** — current title vs `title_at_purchase`, promotion timing
- **Content authored** — posts in last 90 days, dominant themes, post cadence
- **Content engaged with** — what categories of post they react to / comment on
- **Follow graph** — companies followed, operators followed, podcasts/newsletters followed (when surfaced)
- **Signal patterns** — what they engaged with from YOUR team in the 90 days before they closed-won (if `closed_won_at` is provided)

If `agents_behavioral_profile` returns `server_side_ai_unavailable` for any seed → retry once. If still failing → stop and alert. You cannot build a composite from partial data; skipping seeds biases the composite.

---

## STEP 3 — AGGREGATE INTO A COMPOSITE BUYER PROFILE

In-prompt aggregation across all seed behavioral profiles. Output a structured composite:

```yaml
composite_seed_profile:
  n_seeds: [N]
  title_cluster:
    - "[Title pattern 1 with count e.g. 'VP Growth: 7/18']"
    - "[Title pattern 2 e.g. 'Head of Demand Gen: 5/18']"
    - "[Title pattern 3 e.g. 'Director of Growth Marketing: 4/18']"
  company_pattern:
    size_band: "[e.g. 30-150 employees]"
    funding_stage: "[e.g. Series A-B]"
    industries: ["[e.g. B2B SaaS]", "[e.g. dev tools]"]
    tech_stack_signals: ["[e.g. uses HubSpot]", "[e.g. uses Segment]"]
  content_pattern:
    dominant_themes: ["[e.g. outbound efficiency]", "[e.g. RevOps tooling]"]
    cadence: "[e.g. weekly LinkedIn posts]"
    voice: "[e.g. operator-first, low-jargon]"
  follow_pattern:
    common_companies: ["[N seeds follow CompanyX]"]
    common_operators: ["[N seeds follow @operator]"]
    common_podcasts_or_newsletters: ["[if surfaced]"]
  signal_pattern:
    pre_purchase_engagement: "[e.g. 12/18 engaged with at least 2 of our posts in the 90d before close]"
    trigger_events: ["[e.g. recent funding]", "[e.g. job change to current role]"]
```

This composite is the lookalike target for the rest of the run. Save it in-prompt; you'll reference it in Steps 5 and 6.

---

## STEP 4 — BUILD APOLLO FILTER SET

From the composite, construct an Apollo person-search filter:

- `person_titles` — the title cluster from Step 3 (top 3-5 patterns)
- `organization_num_employees_ranges` — from `size_band`
- `organization_industries` — from `industries`
- `organization_latest_funding_stage_cd` — from `funding_stage`
- `person_locations` — keep broad initially (you'll filter on behavioral score, not geo)

If the composite is too narrow (Apollo returns < 200 candidates), broaden by dropping `tech_stack_signals` and going one band wider on size. Note the relaxation in the Slack summary.

If the composite is too broad (Apollo returns > 5,000 candidates), tighten by holding to the top 2 title patterns only and narrowing the size band. Note the tightening in the summary.

Target: pull 1,500-2,500 candidates. This is the wide net you'll filter down to ~100-300 tier-1+2 in Step 7.

---

## STEP 5 — PULL APOLLO CANDIDATES

Call Apollo `people_search` with the Step 4 filter set. Page through results up to the cap (2,000).

For each Apollo candidate, capture: `linkedin_url`, `name`, `current_title`, `current_company`, `email` (if Apollo surfaces it), `organization_id`.

Important: **do not trust Apollo's title or company.** Apollo data is 30-90 days stale. You are pulling Apollo for the firmographic shape only. The title gets re-confirmed in Step 6 by Zevari.

De-dupe pass:
- Drop any candidate whose `linkedin_url` matches a seed (you don't want to re-target your own customers)
- Drop any candidate whose `linkedin_url` is already in `mcp__pipedrive__` as a contact (`search_contacts` by linkedin_url or name+company)
- Drop any candidate already in a Zevari list from the last 90 days via `targets_get_pipeline`

Record the post-dedupe candidate count. This is the live-confirmation cohort.

---

## STEP 6 — LIVE-CONFIRM EVERY CANDIDATE VIA ZEVARI

For each surviving candidate, run in parallel batches (max 3 concurrent Zevari calls). Be patient — this is the slow step.

For each candidate:

1. **Profile** — Zevari `linkedin_get_profile` on `linkedin_url`. Extract live title, live company, live company URL, headline, location.
   - If the LinkedIn URL doesn't resolve (deleted, private, redirected) → mark `unresolved`, skip rest of confirmation for this candidate, log in summary.
   - If live title no longer matches one of the title-cluster patterns from Step 3 (Apollo data was stale and they moved into a different role) → mark `out_of_date`, skip, log.
2. **Company intel** — Zevari `agents_company_intelligence` on live company URL. Confirm size band, industry, funding stage match the composite. If the company has materially changed (e.g. Apollo said Series B, Zevari says recently IPO'd) → re-evaluate against the composite. If it no longer fits → mark `company_drift`, skip.
3. **Behavioral profile** — Zevari `agents_behavioral_profile` on the candidate's LinkedIn URL. Extract the same dimensions you pulled for seeds in Step 2: title trajectory, content authored, content engaged with, follow graph, signal patterns.
4. **ICP score** — Zevari `profile_get` (today's ICP rules) + `agents_icp_score` against the combined firmographic + behavioral data.

Rate-limit handling: no more than 3 concurrent Zevari calls. If Zevari rate-limits mid-batch → pause 5 min, resume. Track time-to-complete and warn in the summary if total wall-time exceeded 60 min.

---

## STEP 7 — LOOKALIKE SCORING

For each live-confirmed candidate, compute the lookalike score (0-20):

- **ICP fit (0-10)** — from `agents_icp_score`. Direct passthrough.
- **Behavioral similarity to seed composite (0-10)** — in-prompt comparison of the candidate's behavioral profile against the Step 3 composite. Score:
  - +2 if dominant content themes overlap with composite `content_pattern.dominant_themes`
  - +2 if posting cadence matches composite `content_pattern.cadence` band
  - +2 if voice/posture matches composite `content_pattern.voice`
  - +2 if candidate follows ≥ 2 of the composite `follow_pattern.common_companies` or `common_operators`
  - +2 if candidate's pre-purchase signal pattern (engagement with our content, recent funding/job change/hiring) matches composite `signal_pattern`
- **Recency boost (0-3)** — add to the score:
  - +1 if recent funding (< 90 days)
  - +1 if hiring spree (≥ 3 relevant openings posted in last 30 days)
  - +1 if fresh job change (< 60 days into current role and the move was a promotion)
- **Engagement-with-you boost (0-3)** — add:
  - +1 per engagement with your content in last 90 days (cap at +3)

Total = ICP fit + behavioral similarity + recency + engagement-with-you. Max effective range is 0-20 (technically up to 26 with boosts but most candidates land 0-20).

Apply hard disqualifiers from `profile_get` (if any candidate fails a hard rule → DQ regardless of score).

---

## STEP 8 — BUCKET

| Tier | Score range | Typical % | Action |
|---|---|---|---|
| TOP_TIER | 16-20 | 5-15% | Save to Zevari `icp-{seed-batch-slug}-tier-1`, create Pipedrive contact tagged `top_tier` |
| TIER_2 | 12-15 | 25-35% | Save to Zevari `icp-{seed-batch-slug}-tier-2`, create Pipedrive contact tagged `tier_2` |
| TIER_3 | 8-11 | rest | Save to CSV only, no Zevari list, no Pipedrive write |
| DQ | < 8 OR hard DQ | varies | Archive in CSV with reason, no further write |

`{seed-batch-slug}` = `YYYY-MM-DD` (the date you ran this), e.g. `icp-2026-05-30-tier-1`.

Important: do not auto-enroll any tier into outbound campaigns from this prompt. That's a separate decision. This prompt produces a tiered list; sequencing happens downstream (your weekly outbound prompt, your AE briefing, manual review).

---

## STEP 9 — WRITE BACK

### 9.i — Zevari lists

- `targets_create_list` for `icp-{seed-batch-slug}-tier-1` and `tier-2`.
- `targets_save` each TOP_TIER candidate into tier-1 list with metadata: `source=icp-discovery`, `lookalike_score`, `icp_fit`, `behavioral_similarity`, `recency_boost`, `engagement_boost`.
- `targets_save` each TIER_2 candidate into tier-2 list with same metadata.

### 9.ii — Pipedrive

For each TOP_TIER and TIER_2 candidate:

- `search_contacts` by `linkedin_url` — if exists, update with the new tier tag and `lookalike_score` custom field. If not, `add_person` with:
  - `name`, `email` (if surfaced), `linkedin_url`, `company`, `title` (the live one from Zevari, not Apollo)
  - Custom fields: `source=icp-discovery`, `seed_batch={slug}`, `lookalike_score`, `tier=top_tier|tier_2`
  - `add_organization` if the org doesn't exist yet
- Do NOT create a deal. This is a contact-only write. Deals get created downstream when outreach starts.

### 9.iii — CSV export

Write the full result to `[OUTPUT_CSV_PATH]`. Columns:

`tier, name, linkedin_url, email, current_title, current_company, company_size, industry, lookalike_score, icp_fit, behavioral_similarity, recency_boost, engagement_boost, top_pattern_matches, dq_reason`

Include all 4 tiers (TOP_TIER, TIER_2, TIER_3, DQ) so the user can audit the full filtering decision.

---

## STEP 10 — SLACK SUMMARY

Send to `[GTM_CHANNEL]` via Chrome `javascript_tool`:

```
<@[YOUR_SLACK_USER_ID]> 🎯 *ICP Discovery Run Complete — {seed-batch-slug}*

📥 Seed customers: [N] (after dropping churned)
🧠 Composite confidence: [high if 18+ seeds, medium if 10-17]

Composite buyer pattern:
• Title cluster: [top 3 from composite, with counts]
• Company shape: [size band, funding stage, industry]
• Content theme: [dominant themes]
• Follow pattern: [top 1-2 surprising follows — these are the muscle in messaging]
• Signal pattern: [pre-purchase engagement summary]

Apollo pull: [N raw candidates]
Post-dedupe (vs seeds, Pipedrive, Zevari pipeline): [N]
Zevari live-confirmation:
  • Resolved + in-pattern: [N]
  • Unresolved / out-of-date / company drift: [N]

Final tiers:
🥇 TOP_TIER (16-20): [N] — saved to `icp-{slug}-tier-1` + Pipedrive
🥈 TIER_2 (12-15): [N] — saved to `icp-{slug}-tier-2` + Pipedrive
🥉 TIER_3 (8-11): [N] — CSV only
🗑️ DQ: [N] — CSV only

Distinctive seed traits worth weaponizing in messaging:
• [Pattern 1 — e.g. "11/18 seeds follow @some-operator — name-check them in cold open"]
• [Pattern 2 — e.g. "8/18 seeds posted about outbound efficiency in last 60d — lead with that angle"]
• [Pattern 3 — e.g. "average seed got promoted within 4 months of close — target the pre-promo role"]

Wall time: [Xm]
Output CSV: [OUTPUT_CSV_PATH]
```

---

## STEP 11 — REFLECTION

- Was the seed N ≥ 18? If not, flag that the composite is lower-confidence and the user should curate more closed-won customers before the next run.
- Any surprising title patterns? Sometimes 3 of your top 18 customers are in a title you didn't even know was in your ICP. Flag for RevOps.
- Any surprising follow patterns? Common podcast / newsletter follows are the highest-leverage messaging input — flag the top one explicitly.
- Did Apollo return < 200 candidates after filter? The composite was too narrow — note the filters you relaxed.
- Did Zevari live-confirmation drop > 25% of Apollo candidates as `unresolved` / `out_of_date` / `company drift`? Apollo data is unusually stale for this vertical — note for the next run.
- Top-tier ratio came in much higher or much lower than the 5-15% band? Either the composite is overfit (very few candidates pass) or the wide net is too narrow (too many candidates pass). Flag for next-run calibration.

---

## Schedule

Not scheduled. Run manually every 90 days as the closed-won customer mix shifts. Suggested cadence: end of each quarter, refresh the seed CSV from Pipedrive Closed-Won (high deal value + still active), re-run.
