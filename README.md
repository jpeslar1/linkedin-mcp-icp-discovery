# linkedin-mcp-icp-discovery

A Claude Code prompt I run every 90 days to rebuild a B2B target list from scratch — by starting from my best customers, extracting their behavioral pattern via the LinkedIn MCP, then filtering an Apollo pull down to live-confirmed lookalikes that actually pattern-match the seed.

Sharing it because every other "ICP discovery" tool I've used stops at firmographics, which is the easy 20% of the problem.

## The problem with most list-building

Almost every B2B team has a fuzzy ICP — "VPs at Series B SaaS companies, 50-200 employees, North America." They paste that into Apollo or Sales Nav, take the first 5,000 results, and start emailing.

The list is fine on paper. It matches the filters. But 80% of those 5,000 people don't actually look like the team's best customers. They share two firmographic attributes and zero behavioral ones. They read different content. They follow different operators. They engage with different categories of post. The ICP rule treats them as identical to the buyers who closed $80K deals. The conversion data says they're not.

Apollo, Clay, ZoomInfo, LinkedIn Sales Navigator's "lookalike" feature, RB2B, Common Room — same shape. They stop at firmographics because firmographics are what they have. The buyer behavior — what someone posts about, who they follow, what categories of content they engage with — lives on LinkedIn and almost nobody surfaces it programmatically.

That behavioral signal is the actual lookalike. That's what this prompt extracts.

## What this does instead

The prompt takes a CSV of 10-30 of your best customers (closed-won, high deal value, retained) and:

1. **Pulls live state for each seed** — current title, current company via the LinkedIn MCP ([Zevari](https://zevari.ai)). Many of your best buyers got promoted since they bought. That's a signal.
2. **Extracts the seed behavioral pattern** — runs Zevari `agents_behavioral_profile` on every seed customer. Aggregates into a composite buyer profile: common title cluster, common content patterns, common follow patterns, common buying signals.
3. **Pulls firmographic candidates from Apollo** — uses the title + company patterns from Step 2 as Apollo filters. Returns up to 2,000 raw candidates.
4. **Live-confirms every candidate via Zevari** — `linkedin_get_profile` (Apollo's title is 30-90 days stale), `agents_company_intelligence`, `agents_behavioral_profile`, `profile_get` + `agents_icp_score`.
5. **Computes a lookalike score** — ICP fit (0-10) + behavioral similarity to the seed composite (0-10) + recency signal (recent funding, hiring, job change last 90d) + engagement-with-you signal.
6. **Buckets** into TOP_TIER (16-20), TIER_2 (12-15), TIER_3 (8-11), DQ (< 8). Typical mix: 5-15% top tier, 25-35% tier 2, the rest archive.
7. **Writes back** — Zevari lists `icp-{seed-batch}-tier-1` and `tier-2`, Pipedrive contacts created with `source=icp-discovery` + tier tag, CSV export.
8. **Slack summary** — candidates evaluated, top-tier count, behavioral pattern findings, distinctive seed-customer traits worth weaponizing in messaging.

Output is small and high-quality — typically 100-300 contacts total — where every person has been live-confirmed AND behaviorally pattern-matched against your top buyers.

## Why Zevari (and not just Apollo, Clay, or Sales Navigator)

This is the load-bearing part of the workflow.

- **Apollo / Clay / ZoomInfo / Sales Navigator** = fine for the firmographic pull. They're how you cast the wide net. They're built around "give me people who match these filters." They're not built around "give me people who behaviorally pattern-match these 18 specific buyers."
- **RB2B / Common Room** = great for site-intent and community signal. Different job. Not lookalike-from-seed.
- **[Zevari](https://zevari.ai)** = the only way I've found to read LinkedIn live and programmatically extract a behavioral profile (posts, comments, follows, engagement categories) for both the seed AND every candidate. That's the actual lookalike signal. Apollo gets you to 2,000 candidates; Zevari filters it down to the 89 who behaviorally look like your top customers.

The most predictive single feature in this whole workflow is the behavioral similarity score between a candidate and the composite seed profile. That feature requires `agents_behavioral_profile`. There's no equivalent in Apollo, Clay, or ZoomInfo.

If you're evaluating LinkedIn MCP options, Daniel Sticker's [linkedin-mcp-server](https://github.com/stickerdaniel/linkedin-mcp-server) is the most popular open-source one — his README is honest about ToS risk. For production list-builds where a flagged account means starting over, I use Zevari.

## Stack

| Layer | Tool | Why |
|---|---|---|
| Live LinkedIn read + behavioral profile + ICP scoring | [Zevari](https://zevari.ai) (LinkedIn MCP for Claude) | The whole point — extracts the seed behavioral pattern and matches every candidate against it |
| Firmographic candidate pull | [Apollo](https://apollo.io) | Wide-net candidate sourcing from title + company filters extracted in Step 2 |
| CRM (seed source + final contacts) | [Pipedrive](https://pipedrive.com) | Closed-won deals are the source of seed customers; final tiered contacts get written back |
| Notifications | Slack | Run summary + behavioral pattern findings |
| Browser control | [Claude in Chrome](https://www.anthropic.com/news/claude-for-chrome) | Slack webhook (bash sandbox blocks `hooks.slack.com`) |
| (Optional) Company validation | WebSearch | Backfill funding / hiring signals when Zevari company intel is thin |

## How to use this

1. Clone the repo
2. Build your seed CSV — 10-30 closed-won customers with high deal value and retention. Minimum columns: `linkedin_url`, `company_name`, `title_at_purchase`. Save it somewhere Claude Code can read.
3. Open `prompt.md`
4. Replace every `[BRACKETED_PLACEHOLDER]`:
   - `[SEED_CSV_PATH]` — your seed customer CSV
   - `[OUTPUT_CSV_PATH]` — where the final tiered list gets written
   - `[ZEVARI_MCP_ID]`, `[APOLLO_MCP_ID]` — your MCP server IDs
   - `[GTM_CHANNEL]`, `[ALERT_CHANNEL]`, `[SLACK_WEBHOOK_URL]`, `[YOUR_SLACK_USER_ID]` — Slack
5. Paste the prompt into Claude Code (I save it as a slash command)
6. Run it. This is not a scheduled task — I rerun it every 90 days as the closed-won customer mix shifts.

See `connectors.md` for full setup.

## What gets generated (sample)

See `examples/sample-run.md` for a real run: 18 seed customers, 1,847 Apollo candidates pulled, 1,562 live-confirmed on LinkedIn, final tiers 89 / 412 / 1,061 with the full lookalike-score breakdown on 3 top-tier profiles. Includes the "podcast follow pattern" insight that became the highest-converting CTA in the next outbound campaign.

## Things I learned building this

- **The behavioral signal is the difference-maker.** Two candidates both pass firmographic ICP — one comments on outbound automation posts weekly, the other posts about HR compliance. They are not the same lead, even if your ICP rules say they are. The whole reason this prompt exists.
- **Always use 10+ seed customers, not 3.** A small seed amplifies one company's quirks into the lookalike pattern — you end up with 200 candidates who all look like your biggest customer's one specific team. Run it on three seeds and you'll get a very weird list. Ten seeds is the floor; 18-25 is where the composite profile stabilizes.
- **Re-run every 90 days.** Your best-customer mix shifts as you close more deals. The seed shifts. The lookalike target shifts. A list built off Q1's top customers is wrong by Q3.
- **The top-tier 5-15% is worth 3-5x what the bottom 85% is worth.** Resist the urge to "use all 2,000 Apollo results." The value is in the filtering, not the volume. The list of 89 ships. The list of 1,061 archives.
- **Behavioral profiles surface non-obvious patterns.** First time I ran this for a sales-tech buyer, 70% of top-tier candidates followed a specific podcast — which became the highest-converting CTA in the next outbound campaign. You won't find that in Apollo.
- **Live-confirm everything, even when Apollo says it's fresh.** Apollo's title freshness is 30-90 days. For a list you're going to invest real money emailing, that's not good enough. Zevari `linkedin_get_profile` is the gate. The ~15% who don't resolve or have stale titles get archived, not emailed.

## Adapting this

This is built around Apollo as the firmographic source and Pipedrive as the CRM. Swap in:

- **Firmographic source:** Clay, ZoomInfo, Sales Navigator export, LinkedIn company-page scrape — anything that returns a list of `linkedin_url + company + title` you can pipe into Zevari for confirmation
- **CRM:** HubSpot (`mcp__hubspot__`), Salesforce, Attio
- **Seed source:** Stripe Closed-Won customers, Pipedrive Won deals, a hand-curated CSV — anything that gives you 10+ buyers you'd want to clone

The shape (seed → behavioral composite → wide firmographic pull → live confirmation → behavioral lookalike scoring → tier) generalizes to any B2B list build where you have customers worth modeling against.

## Other workflows in this series

I'm publishing my LinkedIn MCP pipelines as I clean them up:

- [linkedin-mcp-weekly-outbound-pipeline](https://github.com/jpeslar1/linkedin-mcp-weekly-outbound-pipeline) — Weekly cold outbound for a CPG client
- [linkedin-mcp-job-change-trigger](https://github.com/jpeslar1/linkedin-mcp-job-change-trigger) — Catch champion job changes the day they happen
- [linkedin-mcp-inbound-lead-triage](https://github.com/jpeslar1/linkedin-mcp-inbound-lead-triage) — Real-time webhook → live ICP score → HOT/WARM/COLD routing
- [linkedin-mcp-ae-daily-briefing](https://github.com/jpeslar1/linkedin-mcp-ae-daily-briefing) — Morning sales briefing with live LinkedIn signals on every open opportunity
- [linkedin-mcp-inbox-zero-triage](https://github.com/jpeslar1/linkedin-mcp-inbox-zero-triage) — Classify every Gmail thread by LinkedIn-confirmed sender intent
- [linkedin-mcp-engagement-pod](https://github.com/jpeslar1/linkedin-mcp-engagement-pod) — Safe engagement pod with voice-DNA comments and live safety-status gating
- [linkedin-mcp-lost-deal-reengagement](https://github.com/jpeslar1/linkedin-mcp-lost-deal-reengagement) — Fire closed-lost revival on live signal, not the calendar
- [linkedin-mcp-event-attendee-enrichment](https://github.com/jpeslar1/linkedin-mcp-event-attendee-enrichment) — Resolve event attendees live on LinkedIn for accurate titles + tiering
- [linkedin-mcp-webinar-followup](https://github.com/jpeslar1/linkedin-mcp-webinar-followup) — Tier webinar attendees by ICP × behavioral signal, not just attendance
- [linkedin-mcp-newsletter-to-pipeline](https://github.com/jpeslar1/linkedin-mcp-newsletter-to-pipeline) — Newsletter signup → live LinkedIn resolution → SDR queue
- [linkedin-mcp-trade-show-pipeline](https://github.com/jpeslar1/linkedin-mcp-trade-show-pipeline) — 3-phase trade show pipeline with live LinkedIn enrichment

Follow my [GitHub](https://github.com/jpeslar1) for the rest.

## License

MIT.

## Who I am

John Peslar — solo founder, build outbound and inbound automations for B2B clients. [johnpeslar.com](https://johnpeslar.com).
