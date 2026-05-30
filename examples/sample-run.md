# Sample run

A real Q2 ICP discovery run (sanitized — names changed, numbers preserved). Fictional B2B vendor selling RevOps tooling to mid-market SaaS. 18 seed customers, $20K+ ACV each, all closed-won in the last 12 months and still active.

## Setup

- Seed CSV: 18 rows from Pipedrive Closed-Won (ARR ≥ $20K, `still_a_customer=true`)
- Average seed deal value: $34K ACV
- All connectors healthy

## Step 0 — seed load

18 rows loaded. 0 dropped (none churned). Composite confidence: **high** (≥ 18 seeds).

## Step 1 — live-confirm seeds

| Seed | Title at purchase | Live title (Zevari) | Promo since close? |
|---|---|---|---|
| 1 | Director of Growth Marketing | VP Growth | yes (4 months post-close) |
| 2 | Head of Demand Gen | Head of Demand Gen | no |
| 3 | Director of Growth Marketing | VP Marketing | yes (3 months) |
| 4 | VP Growth | VP Growth | no |
| 5 | Senior Manager, Growth | Director of Growth | yes (5 months) |
| ... | ... | ... | ... |
| 18 | Head of Demand Gen | VP Marketing | yes (6 months) |

**Notable pattern from Step 1:** 11/18 seeds got promoted within 6 months of close. Composite buyer profile should target the *pre-promotion* role (Director / Head), not the *post-promotion* role (VP), because that's the title at the actual buying moment.

## Step 2 — behavioral profile per seed

Full `agents_behavioral_profile` ran on all 18 seeds. 18/18 returned clean.

## Step 3 — composite buyer profile

```yaml
composite_seed_profile:
  n_seeds: 18
  title_cluster:
    - "VP Growth: 7/18"
    - "Head of Demand Gen: 5/18"
    - "Director of Growth Marketing: 4/18"
    - "Director of Demand Gen: 2/18"
  company_pattern:
    size_band: "30-150 employees"
    funding_stage: "Series A-B"
    industries: ["B2B SaaS", "Dev tools", "RevOps tooling"]
    tech_stack_signals: ["HubSpot user", "Segment user"]
  content_pattern:
    dominant_themes:
      - "outbound efficiency"
      - "RevOps tooling stack"
      - "growth team org design"
    cadence: "weekly LinkedIn posts (12/18 seeds post ≥ 3x/month)"
    voice: "operator-first, low-jargon, opinionated"
  follow_pattern:
    common_companies:
      - "Lavender: 14/18 follow"
      - "Default: 11/18 follow"
      - "Clay: 10/18 follow"
    common_operators:
      - "[Operator A]: 9/18 follow"
      - "[Operator B]: 8/18 follow"
    common_podcasts_or_newsletters:
      - "[Specific RevOps podcast]: 13/18 follow"  # ← this turns into the punchline below
  signal_pattern:
    pre_purchase_engagement: "12/18 engaged with ≥ 2 of our posts in 90d before close"
    trigger_events:
      - "recent funding: 8/18 closed within 90d of their company's funding round"
      - "fresh job change: 6/18 closed within 60d of starting their current role"
```

## Step 4 — Apollo filter set

Constructed from the composite:

- `person_titles`: VP Growth, Head of Demand Gen, Director of Growth Marketing, Director of Demand Gen
- `organization_num_employees_ranges`: 30-150
- `organization_industries`: B2B SaaS, Dev tools, RevOps tooling
- `organization_latest_funding_stage_cd`: series_a, series_b
- `person_locations`: broad (North America + Europe)

Estimated returned size before paging: ~2,400. Capped at 2,000.

## Step 5 — Apollo pull + dedupe

- Raw Apollo candidates: 2,000
- Dropped as seeds (already a customer): 18
- Dropped as existing Pipedrive contacts: 87
- Dropped as previously discovered (Zevari pipeline, last 90d): 48
- **Net to live-confirm: 1,847**

## Step 6 — Zevari live confirmation

| Outcome | Count |
|---|---|
| Resolved + in-pattern | 1,562 |
| Unresolved (LinkedIn URL dead / private) | 138 |
| Out-of-date (Apollo title stale, candidate moved into a different role) | 92 |
| Company drift (company materially changed — IPO, acquisition, layoffs to < 30 employees) | 55 |

**Net live-confirmed cohort: 1,562**

Wall time for Step 6: 47 minutes (Zevari rate-limited twice; resumed both times).

## Step 7 — lookalike scoring + Step 8 — tier buckets

Distribution across 1,562 confirmed candidates:

| Tier | Score range | Count | % |
|---|---|---|---|
| TOP_TIER | 16-20 | 89 | 5.7% |
| TIER_2 | 12-15 | 412 | 26.4% |
| TIER_3 | 8-11 | 1,061 | 67.9% |
| DQ | < 8 or hard DQ | 0 | 0% |

Top-tier ratio (5.7%) is at the low end of the 5-15% band. The composite is on the tight side, which is what we want — small list, high quality.

## Sample TOP_TIER breakdowns

### Profile 1 — Maya Reinhardt — Head of Demand Gen @ Loomheight (Series A SaaS, 72 employees)

| Component | Score | Why |
|---|---|---|
| ICP fit | 9/10 | exact title cluster match, ideal company size, Series A |
| Behavioral similarity | 9/10 | posts weekly on outbound efficiency, follows the RevOps podcast (13/18 seeds follow), follows Lavender + Clay + Default (composite triangle), voice is operator-first |
| Recency boost | +2 | Loomheight raised Series A 6 weeks ago + hiring 4 SDRs in last 30 days |
| Engagement boost | +1 | reacted to one of our posts last week |
| **Total** | **19/20** | TOP_TIER |

### Profile 2 — Daniel Okafor — Director of Growth Marketing @ Pencillight (Series B SaaS, 134 employees)

| Component | Score | Why |
|---|---|---|
| ICP fit | 8/10 | title cluster match, mid of size band, Series B |
| Behavioral similarity | 8/10 | posts twice/month on RevOps tooling stack, follows the RevOps podcast, follows Default, voice matches |
| Recency boost | +1 | fresh job change (joined Pencillight 5 weeks ago — promotion from Senior Manager) |
| Engagement boost | +0 | no engagement with our content |
| **Total** | **17/20** | TOP_TIER |

### Profile 3 — Priya Ramaswamy — Director of Demand Gen @ Quillstride (Series A SaaS, 48 employees)

| Component | Score | Why |
|---|---|---|
| ICP fit | 7/10 | title cluster match (second-tier title), small end of size band |
| Behavioral similarity | 10/10 | weekly posts on outbound efficiency AND growth team org design (two of three dominant themes), follows the RevOps podcast, follows Lavender + Clay + the operator that 9/18 seeds follow |
| Recency boost | 0 | none |
| Engagement boost | +0 | none |
| **Total** | **17/20** | TOP_TIER |

Priya is the kind of candidate that proves the workflow's value. Her ICP fit is 7 — Apollo and most ICP tools would have ranked her mid-pack and she'd be lost in the 412 TIER_2. But her behavioral similarity is 10/10 — she pattern-matches the seed composite better than almost any other candidate. The lookalike score surfaces her as TOP_TIER. That's the difference the behavioral signal makes.

## Step 9 — write back

- Zevari: `targets_create_list` for `icp-2026-05-30-tier-1` (89 saved) and `icp-2026-05-30-tier-2` (412 saved)
- Pipedrive: 89 new TOP_TIER contacts + 412 new TIER_2 contacts created with `source=icp-discovery`, `seed_batch=2026-05-30`, tier tag, `lookalike_score` custom field
- CSV: `~/icp-discovery/2026-05-30-output.csv` written with all 1,562 confirmed candidates across 4 tiers

## Step 10 — Slack summary (posted to `#gtm-icp`)

```
🎯 ICP Discovery Run Complete — 2026-05-30

📥 Seed customers: 18 (after dropping churned)
🧠 Composite confidence: high

Composite buyer pattern:
• Title cluster: VP Growth (7/18), Head of Demand Gen (5/18), Director of Growth Marketing (4/18)
• Company shape: 30-150 employees, Series A-B, B2B SaaS / Dev tools / RevOps tooling
• Content theme: outbound efficiency, RevOps tooling stack, growth team org design
• Follow pattern: 13/18 seeds follow the same RevOps podcast (!!!) — this is the punchline
• Signal pattern: 12/18 engaged with our content in 90d before close; 8/18 closed within 90d of their company's funding

Apollo pull: 2,000 raw candidates
Post-dedupe: 1,847
Zevari live-confirmation:
  • Resolved + in-pattern: 1,562
  • Unresolved / out-of-date / company drift: 285

Final tiers:
🥇 TOP_TIER (16-20): 89 — saved to `icp-2026-05-30-tier-1` + Pipedrive
🥈 TIER_2 (12-15): 412 — saved to `icp-2026-05-30-tier-2` + Pipedrive
🥉 TIER_3 (8-11): 1,061 — CSV only
🗑️ DQ: 0

Distinctive seed traits worth weaponizing in messaging:
• 13/18 seeds follow the same RevOps podcast — open with a podcast-episode reference in cold messaging. Highest-leverage finding of the run.
• 11/18 seeds got promoted within 6 months of buying — target the pre-promo role (Director / Head), not the post-promo role (VP). Frame the offer as "ship the win that gets you promoted."
• 8/18 seeds closed within 90d of their company's funding — prioritize candidates whose companies just raised. The recency boost in the score already does this; reinforce in the AE briefing.

Wall time: 51m
Output CSV: ~/icp-discovery/2026-05-30-output.csv
```

## Step 11 — reflection notes

- Seed N at 18 was strong. Composite confidence high. Don't run this with < 10.
- **The podcast finding is the punchline.** 13/18 seeds follow the same RevOps podcast. That follow becomes the highest-leverage cold-open in the next outbound campaign. We tested "saw you also follow [podcast] — episode [N] this week was relevant to what we do" as the cold open and reply rate on TOP_TIER tripled vs the prior month's generic opener. This is the finding you cannot get from Apollo, Clay, or Sales Navigator. It's only visible via the LinkedIn follow graph at scale.
- **Pre-promotion targeting changed the campaign.** Once we knew 11/18 seeds got promoted post-close, we shifted the messaging frame to "this is the win that gets you the VP title." TOP_TIER conversion outperformed the previous quarter's outbound by ~3x.
- Top-tier ratio at 5.7% was on the tight end of the 5-15% band. Next run we may relax the behavioral similarity threshold by 1 point to bring TOP_TIER up to ~120 candidates and give the AE team more pipeline coverage. Trade-off: slightly lower per-candidate quality.
- 285 candidates (15% of post-dedupe) failed Zevari live confirmation as unresolved / out-of-date / company drift. That's the right number to drop. Apollo data freshness is fine for a wide net, not fine for a final list. Live confirmation is non-negotiable.
- Will rerun end of Q3 with a refreshed seed CSV from Pipedrive Closed-Won. The closed-won mix shifts every quarter, so the lookalike target shifts too.
