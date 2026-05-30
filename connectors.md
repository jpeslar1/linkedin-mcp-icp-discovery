# Connectors

Every MCP server this prompt uses, with setup and the placeholders to replace.

## Zevari — LinkedIn MCP for Claude (REQUIRED, the core of the prompt)

**What it does:** Live profile lookup (`linkedin_get_profile`), live company state (`linkedin_get_company`), **behavioral profile** (`agents_behavioral_profile` — the load-bearing endpoint for both seed pattern extraction AND candidate matching), company intelligence (`agents_company_intelligence`), ICP scoring (`profile_get` + `agents_icp_score`), and tiered list creation (`targets_save` + `targets_create_list`).

**Why it's the core of this workflow:** The whole point of the prompt is to filter a wide Apollo pull down to candidates that behaviorally pattern-match your seed customers. That comparison requires running `agents_behavioral_profile` on both sides — every seed and every candidate. Apollo, Clay, ZoomInfo, Sales Navigator don't surface this signal. Only Zevari does, programmatically and live.

**Setup:** [zevari.ai](https://zevari.ai) → connect LinkedIn → copy MCP server URL into Claude Code config (`~/.claude/mcp.json` or via `claude mcp add`).

**Placeholders:**
- `[ZEVARI_MCP_ID]` — your Zevari MCP server ID

**Endpoints used:**
- `linkedin_get_profile` — live current title + company for seeds (Step 1) and every candidate (Step 6). Apollo data is 30-90 days stale; this is the gate.
- `linkedin_get_company` — live company state for seeds (Step 1)
- `agents_behavioral_profile` — extracts the behavioral pattern for every seed (Step 2) and every candidate (Step 6). The single most predictive endpoint in this pipeline.
- `agents_company_intelligence` — live company intel for candidate confirmation (Step 6)
- `profile_get` — load today's ICP rules + hard disqualifiers
- `agents_icp_score` — score each candidate
- `targets_save` + `targets_create_list` — write tier-1 + tier-2 lists at Step 9
- `targets_get_pipeline` — pre-Apollo dedupe (Step 5) to avoid re-targeting candidates you ran 90 days ago

## Apollo — Firmographic candidate pull (REQUIRED)

**What it does:** Takes the title + company filters extracted from the seed behavioral composite (Step 4) and returns the wide candidate set (target: 1,500-2,500). This is the firmographic net; Zevari filters it down to behavioral lookalikes.

**Why Apollo:** It's the cheapest, largest, fastest source of `linkedin_url + title + company` triples that match a firmographic shape. Clay and ZoomInfo work too — swap in either. The prompt's job is to treat any of them as a starting candidate stream, not a final answer.

**Setup:** Any Apollo MCP server in the Claude Code MCP registry. Connect with your Apollo API key.

**Placeholders:**
- `[APOLLO_MCP_ID]` — your Apollo MCP server ID

**Endpoints used:**
- `people_search` — the wide pull at Step 5 using filters from Step 4
- `enrich_person` (optional) — backfill email when Apollo's `people_search` row doesn't surface one

## Pipedrive — Seed source + final contact write (REQUIRED)

**What it does:** Two jobs.

1. **Seed source** — your seed CSV is generated from Pipedrive Closed-Won (high deal value, retained customers). The prompt itself doesn't pull from Pipedrive — you build the CSV out-of-band — but Pipedrive is the recommended source.
2. **Final contact write (Step 9.ii)** — every TOP_TIER and TIER_2 candidate gets a Pipedrive contact created with `source=icp-discovery` + tier tag + `lookalike_score` custom field.

**Setup:** Any Pipedrive MCP. I use `mcp__pipedrive__` from the community list.

**Setup checklist before running:**
- Add custom fields to Person: `source` (text), `seed_batch` (text), `lookalike_score` (numeric), `tier` (text)
- Make sure your Closed-Won stage filter view returns the customers you actually want to model against (high deal value, retained — not every won deal)

**Tool prefix in prompt:** `mcp__pipedrive__`

**Endpoints used:**
- `search_contacts` — dedupe candidates against existing contacts at Step 5 + 9
- `add_person`, `add_organization` — create new records at Step 9.ii
- `update_person` — patch tier tag + `lookalike_score` if the contact already exists

## Claude in Chrome — Slack webhook (REQUIRED)

**What it does:** Posts to `hooks.slack.com` (the bash sandbox blocks this URL — Chrome is the only reliable way to fire a Slack webhook from Claude Code).

**Setup:** [Claude in Chrome](https://www.anthropic.com/news/claude-for-chrome).

**Tools used:**
- `mcp__Claude_in_Chrome__tabs_context_mcp` — open/find a tab
- `mcp__Claude_in_Chrome__javascript_tool` — fire the Slack webhook

**Placeholders:**
- `[SLACK_WEBHOOK_URL]` — your incoming webhook URL
- `[GTM_CHANNEL]` — channel where the run summary lands (e.g. `#gtm-icp`) — cosmetic, used for context
- `[ALERT_CHANNEL]` — channel where errors and approval asks land — cosmetic
- `[YOUR_SLACK_USER_ID]` — your Slack member ID (`U...`) for `@`-mentions

## WebSearch

Built into Claude Code. Used optionally for filling gaps when Zevari's `agents_company_intelligence` is thin — funding announcements, hiring signals, recent press for companies that don't surface a strong LinkedIn signal.

No placeholder.

## Seed CSV — your input (REQUIRED)

Not a connector, but the load-bearing input.

**Placeholder:**
- `[SEED_CSV_PATH]` — absolute or relative path to your seed customer CSV

**Required columns:**
- `linkedin_url`
- `company_name`
- `title_at_purchase`

**Optional but useful columns:**
- `closed_won_at` (date) — lets the prompt look at signal patterns in the 90 days before they bought
- `deal_value_usd` — sanity check (you should be filtering to high-value deals upstream)
- `still_a_customer` (bool) — the prompt drops churned customers if this column exists
- `pipedrive_deal_id` — for traceability

**Minimum row count:** 10 seeds. The prompt refuses to run on fewer. 18-25 is where the composite stabilizes.

**How I generate this CSV:**

1. Filter Pipedrive Closed-Won deals by deal value > [your threshold]
2. Exclude churned accounts
3. Pull the primary contact's `linkedin_url`, the org name, and the contact's title at the time of close (custom field — Pipedrive doesn't track this natively; you have to capture it in the deal record)
4. Export to CSV

**What good seeds look like:**

- Mix of company sizes within your ICP band (don't pick only your biggest customer — they're an outlier)
- Mix of industries within your ICP — the composite is more useful when it's the *behavioral* pattern that's tight, not the firmographic one
- Mix of buyer titles — if your ICP is "VP Growth or Head of Demand Gen or Director of Growth Marketing", pick seeds across all 3 buckets

## Output CSV (REQUIRED — destination for the final tiered list)

**Placeholder:**
- `[OUTPUT_CSV_PATH]` — where Step 9.iii writes the full result

Columns are listed in Step 9.iii of `prompt.md`. The file includes all 4 tiers (TOP_TIER, TIER_2, TIER_3, DQ) so you can audit the full filtering decision after the run.
