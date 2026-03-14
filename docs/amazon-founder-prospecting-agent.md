# Amazon Founder Prospecting Agent Blueprint

This guide explains how to build an AI-driven prospecting agent that:

- Connects to **Apollo** and **Instantly**
- Uses a third data source (**Crunchbase**) for growth validation
- Finds **US-based Amazon sellers** with specific criteria
- Prioritizes **founders/CEO/decision-makers** (not employees)
- Exports clean lead lists into **Google Sheets**

---

## 1) What the agent should do

### Business objective
Build a repeatable lead pipeline for outreach to growing US Amazon brands.

### Hard filters (must pass)
1. Company sells products on Amazon (primary sales channel should be Amazon-focused).
2. Estimated company monthly revenue is **>= $30,000**.
3. Main decision-maker contact is present (Founder, Co-Founder, CEO, Owner, President, Managing Director).
4. Product price point is **> $30**.
5. Company shows growth in last 6 months (not stagnant in last 12–24 months).
6. Geography is USA.

### Output
A validated lead table in Google Sheets with one row per decision-maker/company.

---

## 2) Recommended stack

## Core systems
- **Apollo**: Contact and company enrichment (titles, LinkedIn, emails, firmographics).
- **Instantly**: Outreach sequencing once leads are validated.
- **Crunchbase** (third source): Growth signals and company activity (funding, growth metadata, hiring/traction hints).
- **Google Sheets API**: Final source-of-truth lead sheet.
- **Orchestrator** (choose one):
  - Low-code: n8n
  - Code-first: Python (FastAPI + Celery/cron)

## Optional but useful
- **Keepa API** (or similar Amazon product intelligence provider): historical pricing/rank cues to estimate momentum.
- **OpenAI or equivalent LLM**: normalize titles, classify Amazon focus, score confidence.

---

## 3) Data model (Google Sheet columns)

Create this header row in your target sheet:

1. `run_id`
2. `timestamp_utc`
3. `company_name`
4. `website`
5. `hq_country`
6. `amazon_store_url`
7. `amazon_confidence_score` (0-100)
8. `est_monthly_revenue_usd`
9. `price_point_gt_30_confirmed` (true/false)
10. `growth_6m_score` (0-100)
11. `stagnation_flag` (true/false)
12. `decision_maker_name`
13. `decision_maker_title`
14. `decision_maker_seniority`
15. `work_email`
16. `linkedin_url`
17. `apollo_person_id`
18. `apollo_company_id`
19. `crunchbase_org_id`
20. `source_notes`
21. `validation_status` (`accepted`, `review`, `rejected`)
22. `rejection_reason`

---

## 4) End-to-end workflow

## Step A — Seed company discovery
Use Apollo company search with filters:

- Location: United States
- Employee range: e.g., 2–200 (tune for founder-led brands)
- Industry keywords: `consumer goods`, `ecommerce`, `retail`, `DTC`, `marketplace`
- Keywords in description: `amazon`, `fba`, `seller central`, `private label`

Store all seed companies in a temporary staging table.

## Step B — Verify Amazon-first selling behavior
For each seed company:

1. Crawl website and detect Amazon links (`amazon.com`, Amazon Storefront URLs).
2. Extract text clues (`sold on Amazon`, `available on Amazon`, `FBA`).
3. Optional: use Keepa/product APIs to detect active Amazon catalog behavior.
4. Score confidence:
   - +50 if Amazon storefront link found
   - +20 if multiple products found on Amazon
   - +15 if Amazon mentioned in hero/about copy
   - +15 if marketplace tooling found (FBA/SellerCentral references)

Require `amazon_confidence_score >= 70`.

## Step C — Product price filter (> $30)
For verified Amazon brands:

1. Scrape/capture top products from Amazon listing pages.
2. Compute median listed price.
3. Mark `price_point_gt_30_confirmed = true` only if median > 30.

If not enough product evidence, mark `review` instead of auto-reject.

## Step D — Revenue filter (>= $30k/month)
Use a blended estimate (single-source estimates are noisy):

- Signals: estimated web traffic, product count, BSR/ranking trends, review velocity, employee growth.
- Build a weighted revenue heuristic and confidence band.

Example acceptance rule:
- Accept if `est_monthly_revenue_usd >= 30000` with confidence >= 0.65.
- Otherwise `review`.

## Step E — Growth/stagnation filter
Use Crunchbase + web signals:

- Positive growth indicators in last 6 months:
  - hiring trend up
  - funding/news updates
  - catalog expansion
  - review velocity increase
- Stagnation indicators in 12+ months:
  - no product additions
  - flat/decreasing visibility
  - no hiring/activity signals

Generate:
- `growth_6m_score` (0-100)
- `stagnation_flag` true if stale

Accept only if:
- `growth_6m_score >= 60`
- `stagnation_flag = false`

## Step F — Decision-maker extraction (not employees)
Use Apollo people search by company with title/seniority filters:

Allowed titles (priority order):
1. Founder / Co-Founder
2. CEO
3. Owner / President
4. Managing Director / Principal

Reject roles containing:
- `assistant`, `intern`, `coordinator`, `specialist`, `associate`, `executive assistant`

If no founder/CEO found, route to manual review.

## Step G — Deduplication & final scoring
Deduplicate by:
- company domain
- normalized company name
- LinkedIn profile

Create final score:

`final_score = 0.30*amazon_conf + 0.25*growth_score + 0.20*revenue_conf + 0.15*decision_maker_quality + 0.10*data_completeness`

Only export `accepted` + `review` to Google Sheets, with clear status and reasons.

## Step H — Push to Google Sheets and Instantly
- Upsert records into Google Sheet tab `Qualified_Leads`.
- For `accepted` leads only, create/update in Instantly list `US_Amazon_Founder_Outreach`.
- Keep immutable audit log tab `Run_Audit`.

---

## 5) Implementation option 1: n8n (fastest to launch)

Build these n8n nodes/workflows:

1. **Cron Trigger** (daily/weekly)
2. **HTTP Request (Apollo Companies)**
3. **Code Node** (normalize + stage)
4. **HTTP Request / Scraper** (website Amazon detection)
5. **HTTP Request (Crunchbase enrichment)**
6. **Code Node** (scoring + filters)
7. **HTTP Request (Apollo People)**
8. **Code Node** (title filtering + dedupe)
9. **Google Sheets Node** (append/upsert)
10. **HTTP Request (Instantly API)**
11. **Slack/Email alert** for run summary

Use retries + dead-letter branch for failed enrichments.

---

## 6) Implementation option 2: Python service (more control)

## Suggested project layout

```text
agent/
  app.py
  pipelines/
    discover_companies.py
    validate_amazon_presence.py
    estimate_revenue.py
    score_growth.py
    find_decision_makers.py
    export_google_sheets.py
    push_instantly.py
  clients/
    apollo_client.py
    crunchbase_client.py
    instantly_client.py
    sheets_client.py
  models/
    lead.py
  tests/
```

## Run sequence
1. `discover_companies`
2. `validate_amazon_presence`
3. `estimate_revenue`
4. `score_growth`
5. `find_decision_makers`
6. `final_rank_and_export`

Schedule with cron or a queue worker.

---

## 7) Prompting layer for LLM-based classification (optional)

Use LLM only for ambiguous classification tasks, not as sole data source.

Example prompt objective:
- Determine if company appears Amazon-first seller.
- Identify if contact title is true decision-maker.
- Return strict JSON only.

Guardrails:
- Reject non-JSON output.
- Re-run with constrained prompt on parse error.
- Keep deterministic temperature (0–0.2).

---

## 8) Quality checks (must-have)

Before a lead is marked `accepted`, require:

1. Amazon evidence present (URL/text) and score >= 70.
2. Product pricing evidence supports > $30.
3. Revenue estimate >= $30k with confidence >= 0.65.
4. Growth score >= 60 and not stagnant.
5. Decision-maker title in approved list.
6. US geography confirmed.

If one critical check fails -> `rejected` with reason.
If evidence is incomplete but promising -> `review`.

---

## 9) Testing and validation checklist

## Unit tests
- Title normalization/classifier tests
- Revenue heuristic math tests
- Growth scoring tests
- Dedupe logic tests

## Integration tests
- Apollo API contract test
- Crunchbase API contract test
- Google Sheets write/read test
- Instantly list upsert test

## Data QA tests (every run)
- Duplicate rate < 5%
- Missing decision-maker email < 20%
- Rejected for "not Amazon" tracked by count
- Acceptance rate monitored over time

## Manual QA (weekly sample of 20)
- Verify Amazon listing reality
- Verify decision-maker legitimacy
- Verify growth interpretation

---

## 10) Security, compliance, and deliverability

- Store API keys in environment variables or secrets manager.
- Log only needed PII; avoid over-collection.
- Follow applicable laws (CAN-SPAM, GDPR where relevant).
- Warm Instantly domains and monitor bounce/complaint rates.
- Suppress contacts who opt out.

---

## 11) First 7-day rollout plan

## Day 1–2
- Connect Apollo, Crunchbase, Google Sheets, Instantly.
- Build seed + enrichment pipeline.

## Day 3–4
- Add scoring and hard filters.
- Add dedupe and status reasons.

## Day 5
- Run on small batch (100 companies).
- Manually audit 20 accepted + 20 rejected.

## Day 6
- Tune thresholds (amazon score, growth, revenue confidence).

## Day 7
- Move to scheduled runs.
- Activate Instantly push for accepted leads.

---

## 12) Practical default thresholds (starting point)

- `amazon_confidence_score >= 70`
- `est_monthly_revenue_usd >= 30000`
- `revenue_confidence >= 0.65`
- `median_product_price > 30`
- `growth_6m_score >= 60`
- `stagnation_flag = false`

Tune these after first 2–3 runs using QA feedback.

---

## 13) What you need from your side

To deploy quickly, prepare:

1. Apollo API key + workspace info
2. Instantly API key + target list ID
3. Crunchbase API key
4. Google Cloud service account JSON with Sheets access
5. Google Sheet URL and tab names
6. Preferred run schedule (daily/weekly)
7. Outreach rules (max daily sends, target personas, excluded industries)

With these, the agent can be production-ready in days, not weeks.
