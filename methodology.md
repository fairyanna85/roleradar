# RoleRadar — Role-Matching Algorithm: Eligibility Gates → Affinity Scoring

*v1.25 — 2026-10-02*

A reusable methodology for measuring job postings against a candidate. Designed for senior operator/leader searches (Director → C-suite) but adaptable to any level and any function.

**Design principle:** gates before scores. Never rank a role the candidate would never take. Eligibility is binary and cheap; affinity is continuous and expensive. Every role passes through Stage 1 (gates) before it is allowed into Stage 2 (scoring).

Everything candidate-specific lives in the **Candidate Profile Config** (§1). The algorithm itself never changes between candidates — only the config does. All examples below are **fictional and deliberately generic** — never copy them into a real config; derive every value from the intake.

---

## 1. Candidate Profile Config

Fill this block once per candidate and store it in the spreadsheet's **Config tab** (TDD §3.0) — the worker loads it at pipeline step 1. Never keep the config only in chat context. Every `[CONFIG]` value below is referenced by gate and scoring rules.

| Parameter | Description | Example (fictional) |
|---|---|---|
| `CANDIDATE_NAME` | Who the search serves | — |
| `CURRENT_LEVEL` | Candidate's current tier | VP / Director / L6 |
| `BASE_ANCHOR` | Current base salary; posted bands are compared to this, never to total comp | $180k |
| `TC_TARGET` | Total-comp aspiration (context only, not a gate) | ~$260k |
| `GEO_SET` | Acceptable locations (gate, not scored) | Chicago / US-remote |
| `TRAVEL_CAP` | Max acceptable travel | <20%, no red-eyes |
| `LEVEL_FLOOR` | Lowest acceptable tier (below = cut) | Director-equivalent ("Head of" with P&L) |
| `MANDATE_BUCKETS` | The 4–6 role families actually done | e.g. PM flavor: program/transformation leadership, CoS/COO/strategy & ops · e.g. support flavor: support-org leadership, CS operations & tooling, support-led growth · e.g. GTM flavor: revenue operations, partnerships leadership |
| `TASTE_PROFILE` | Cultures/missions the candidate is drawn to (feeds Fulfillment + gem hunting) | e.g. mission-driven fintech; craft-and-rigor cultures |
| `ANTI_PROFILE` | What the candidate is leaving (feeds the culture screen) | e.g. what you're leaving — be concrete: patterns, not vibes |
| `EXPERTISE_TAGS` | What the candidate is distinctively good at (feeds Experience fit + problem-led search) | e.g. PM: transformation leadership, AI systems that scale judgment · e.g. support: scaling support orgs 10x, support economics |
| `NORTH_STAR_LIST` | People and/or ventures the candidate most wants proximity to (feeds the affinity bonus) | e.g. 8–12 named operators/thinkers and/or admired companies |
| `SECONDARY_MARKET_BANDS` | Only if `GEO_SET` has a secondary market with different comp norms: separate bands per employer type | e.g. EU market: local cos ~€120k+ TC; US cos locally €200k+; intl orgs separate |

---

## 2. Stage 1 — Eligibility Gates (binary, evaluated in cost order)

Evaluate cheapest first. A role that fails any gate is **cut with a recorded reason** — the reason feeds the self-improvement loop (§5).

### G1 · Location gate
The role's work location must be inside `GEO_SET`. Location is a **gate, never a scored dimension** — a great role in the wrong city is cut outright, not scored down. Verify the actual office/remote policy, not the posting's marketing ("remote" that means "remote in Texas" fails a NYC-candidate gate).

### G2 · Level / authority screen
Map the title + scope to a tier (see §3.3). Cut when:
- The tier is below `LEVEL_FLOOR` (junior/IC seats for a leader candidate);
- It is a **lieutenant seat**: reports into a strong incumbent doing the same job, ambiguous mandate, or "chief of staff" that is really an EA;
- **No real ownership**: the seat carries accountability without authority — no budget, no headcount, no decision rights over the outcomes it owns;
- **Comp below the walk-away floor**: the posted band's maximum sits below the candidate's floor (cut reason `comp-below-floor`). The floor is a hard cut; the anchor is a scored dimension — don't confuse them.

Prefer "owns the seat" over "supports the seat." Note: "no real ownership" is **not** "no transformation." A seat that owns outcomes at scale — CSAT, retention, cost-per-ticket, P&L — passes even with zero 0-to-1 narrative. Transformation theater is not the test; authority over outcomes is.

### G3 · Culture elimination screen
Cut employers whose **dominant, enforced** culture matches `ANTI_PROFILE`. Weigh evidence, not presumption:
- **Cut signals:** employer branding dominated by ideological pledges; credible reporting of compelled speech or ideological litmus tests in hiring/promotion; consistent patterns (not single anecdotes) of low accountability, triangulation, or indirect communication across employee reviews.
- **Never a cut alone:** a diverse workforce, a standard corporate values page.
- **Thin evidence:** do not cut — score Values-fit down instead and note the uncertainty on the card.

### G4 · Active-role verification (most expensive — run last)
The role must be **live on the employer's own board/ATS and accepting applications at evaluation time**. Anything closed, reposted-and-dead, or unconfirmable is cut, no exceptions.
- **JS-gated boards** (Workday, Taleo, custom ATS that don't render via text fetch): fallback = employer-posted LinkedIn job page **plus** one fresh aggregator mirror (crawl ≤14 days). Both must be live.
- Record the verification URL(s) — both URLs in the JS-gated fallback case.

### G5 · Logistics cap
Travel, schedule, or legal constraints beyond `TRAVEL_CAP` (or visa/sponsorship mismatches) cut the role.

**Gate order rationale:** G1–G3 are cheap (posting text + light research); G4 is expensive (live verification); run it last so you never verify a role you'd cut anyway.

---

## 3. Stage 2 — Affinity Scoring (only gate-passers)

The candidate defines **4–7 scoring pillars** in the Pillars tab — each with a name, a one-line definition, a weight, and written 3/6/9 anchors. Score 1–10 per pillar. The pillars are **inferred from the intake** (resume → career stage → trade-offs), never a fixed set: what a junior optimizes for (learning velocity, mentorship, brand) looks nothing like what an executive optimizes for, and the engine must not assume otherwise. A candidate may also define a pillar differently than another candidate would — the definition is theirs.

**Composite:** `Σ(wᵢ · sᵢ) / Σ(wᵢ)` — the divisor is always the sum of the weights, whatever they are.

### Reference set — senior operator/leader (one instance, not the default)

| Pillar | Weight | What it measures |
|---|---|---|
| Experience fit | 20% | Overlap between the mandate and `EXPERTISE_TAGS`; has the candidate done this before at this scale? |
| Compensation | 25% | Posted band (base) vs `BASE_ANCHOR`; bonus/equity as labeled upside |
| Status | 5% | Tier of the seat relative to `CURRENT_LEVEL` (§3.3) |
| Fulfillment | 10% | Resonance with `TASTE_PROFILE` — mission, domain, culture |
| Advancement | 10% | Where this seat leads in 2–4 years (scope growth, not just title) |
| Values fit | 10% | Residual cultural alignment after surviving G3 |

A junior candidate's pillars might instead be Compensation, Learning velocity, Mentorship quality, Brand, Work-life balance, Mission. The engine doesn't care what the pillars are — only that they're defined, anchored, and weighted before the first scored run.

### 3.1 Affinity bonus (not a pillar)
`NORTH_STAR_LIST` holds people, dream companies, and dream role archetypes (e.g. "VP Support at a high-growth SaaS"). `+1.0` for a seat at a north-star person's own venture or an exact dream-company match; `+0.5` for their praise-network orbit or a dream-role-archetype match (never invent a connection — every orbit link needs a public evidence URL, stored in the People tab's `evidence_url` column). Add after the composite; cap the total at 10.

### 3.2 Compensation rules
- **Anchor rule:** posted bands are base salary. Compare to `BASE_ANCHOR`, never to total comp. Bonus + equity sit on top and are usually unadvertised.
- **Walk-away floor:** hard cut at gate time (G2) when the band's max < floor — never a scored dimension.
- **Estimation rule:** comp is never "unknown." When undisclosed, research market data (Levels.fyi, Glassdoor, Pave for tech; Salary.com, Robert Half guides, peer postings for non-tech functions) and publish a labeled estimate (`est.`, `≈`, one-line source note). An honest labeled estimate beats a blank; never invent precision.
- **Record two filterable facts per role:** (a) does the company grant equity at this level (RSU/options/BSPCE)? (b) does it practice sign-on bonuses / unvested-equity buyouts? Each with a one-line evidence note (`equity_note`, `signon_note` columns).

### 3.3 Level tiering
Tier from required years of experience cross-checked with title scope, relative to `CURRENT_LEVEL`:

| Tier | Score | Typical title |
|---|---|---|
| C-level | 10 | CEO / COO / co-founder |
| +2 above candidate | 9 | SVP / EVP |
| +1 above candidate | 8 | VP / Senior Director |
| **Candidate parity** | **7** | **"Head of" / Director with mandate** |
| −1 | 6 | Senior Manager / junior Director |
| −2 | 5 | Manager |
| −3 | 4 | Senior IC |
| Below | 3 | IC / associate |

±1 for exceptional/weak company brand or selectivity, noted on the card. Every card labels **IC vs MGR** prominently (MGR preferred for leader candidates).

---

## 4. The Role Card (delivery unit)

One card per qualifying role, written **to** the candidate (second person). Ends with the apply link. Fields: title — company; seniority; level · IC/MGR; the job; why it's exciting; why it's a fit (concrete links to the candidate's profile, not generic); values; affinity bonus; compensation; equity; sign-on/buyout; location; people (hiring manager, marked unconfirmed if unverified — never invent a URL); process (rounds, homework, timeline — flag tedious); **Score: x.xx** — one-line assessment (from the `one_liner` column); Apply link.

The job / why-it's-exciting / values sections are generated prose at delivery time — not stored columns. The score, one-liner, and all facts come from the pipeline row.

---

## 5. Cut Taxonomy (feeds the learning loop)

Every cut records one primary reason. The standard set:

`location` · `level-below-floor` · `ic-seat` · `lieutenant-seat` · `comp-below-anchor` · `comp-below-floor` · `culture-screen` · `stale-or-unconfirmable` · `closed` · `weak-mandate-fit` · `travel-cap` · `junior-title`

If one reason exceeds ~50% of cuts over three consecutive runs, the discovery queries are tightened to filter it earlier (documented in the run log). The taxonomy is what makes the system self-improving rather than merely repetitive.

---

## 6. Calibration Notes

- Write pillar anchors **once** per candidate before the first scored run (stored in the Pillars tab); revisit quarterly.
- The reference set's weights are one instance — there are no default weights. Every candidate's weights come from the intake trade-offs. Whatever the weights, the composite divisor is their sum; version any change with a date in the Pillars tab.
- The gates are deliberately stricter than the scores: it is better to cut a debatable role at G2/G3 than to score it a 5.5 and waste the candidate's attention. Attention is the scarcest resource in the system.

---

## 7. Creative Targeting — why smart beats literal

**The failure of literal targeting.** Title keywords + level + location finds what everyone finds, and misses three ways: (1) *title mismatch* — the mandate you want exists under titles you'd never search; (2) *unposted need* — the companies that most need your spike aren't posting the obvious title, they have the *problem*; (3) *adjacent mandates* — your history implies seats you could own but would never query. The candidate's edge is their specific shape; targeting must be shaped like them, not like the job board's taxonomy.

**The logic — invert the search.** Instead of candidate → keywords → postings, derive *target hypotheses* from the candidate's signals, then execute:

| Signal (where from) | Targeting hypothesis | Rationale |
|---|---|---|
| Trajectory arc (resume, Phase 1) | **Lookalike profiles:** people who walked your path — where are they now? Their employers hire your shape. | Career paths rhyme; employers who hired your doppelgänger will recognize you. |
| Spike (resume → `EXPERTISE_TAGS`) | **Problem-led:** who has the problem you're distinctively good at solving, *right now*? | Your spike is a solution; search for the problem, not the title. A company in post-merger chaos needs transformation leadership before it posts for it. |
| Role families (resume → `MANDATE_BUCKETS`) | **Mandate expansion:** adjacent seats your families imply (partnerships → ecosystem GM, channel chief). | Titles are employer slang; mandates are the real unit. Adjacent mandates are where the mandate-fit is high and the competition is thin. |
| North-star list (intake) | **Dream-venture sweeps + orbit:** the companies of people you admire, and of people who publicly praise them. | Highest-signal practice: taste is predictive, and admired people cluster at interesting companies. |
| Taste profile (intake → `TASTE_PROFILE`) | **Mission/culture matching:** postings whose language matches your taste; investors whose taste matches it (portfolio sweep). | Culture fit is detectable in language before it's detectable in interviews. |
| Dealbreakers (intake → `ANTI_PROFILE`, floors) | **Negative space:** entire categories excluded from targeting — not scored down, never swept. | Don't spend discovery budget on what the gates will kill. |
| Pillar weights (intake) | **Effort allocation:** high-weight pillars get more creative-discovery time (e.g. builder scope 25% → overweight "first hire" / "build the function" language). | Discovery time is finite; spend it where the candidate's scoring cares most. |
| Career stage (inferred) | **Source selection:** exec → partner networks and curated lists; junior → boards and communities. | The right pond matters more than the right bait. |

**Rules.** (1) A *perfect gem* = taste fit × expertise fit — both from the candidate's config, never a generically strong role. (2) Every creative target is traceable to a signal — the hypothesis is the audit trail; if you can't name the signal, don't chase the target. (3) Creative discovery is budgeted (~15 min of W1 per run), not unbounded — the table above is how the budget is split.

### 7.1 Execution procedures (what the builder/worker actually does)

**P1 — Lookalike extraction** (build; refresh on resume change). Input: trajectory arc from Phase 1 (role families × industries × seniority inflections). Steps: (1) find 8–12 people with a rhyming arc — same inflection points, adjacent industries — via search/X/LinkedIn; (2) list their current employers → Companies tab (note the `why`); (3) the people themselves → People tab as lookalike profiles.

**P2 — Problem translation** (build + per run). Input: `EXPERTISE_TAGS`. Steps: (1) translate each tag into 2–3 problem statements ("orgs integrating post-merger", "support orgs scaling 10x on flat headcount"); (2) each run, scan news/funding/layoff signals for companies exhibiting the problem; (3) sweep their careers pages. Output: targets enter the pipeline with `gem_practice=problem-led`.

**P3 — Mandate expansion** (build). Input: `MANDATE_BUCKETS`. Steps: (1) for each bucket, list adjacent mandates — same skills, different titles; (2) write adjacent-title keyword queries → Boards tab. Output: the board sweep covers mandates, not just titles.

**P4 — Dream-venture mapping** (build + per run). Input: `NORTH_STAR_LIST`. Steps: (1) map each name → ventures founded/led/funded + orbit (public praise/support, evidence URL each — never invented); (2) ventures → Companies tab (affinity bonus armed); (3) people → People tab. Per run: sweep their boards + socials.

**P5 — Taste→language** (per run). Input: `TASTE_PROFILE`. Steps: (1) derive language markers (e.g. "optimistic lens" → *build, frontier, abundance* vs *risk, guardrails*); (2) overweight postings matching the markers when sweeping; (3) pick 3–5 taste-aligned investors → sweep their recent portfolio additions' careers pages.

**P6 — Negative space** (build). Input: `ANTI_PROFILE`, floors, `GEO_SET`. Steps: (1) compile excluded categories (titles, company types, geos); (2) apply at discovery (don't sweep) *and* at gates (don't score). Record skipped categories in the run log.

**P7 — Effort allocation** (per run). Input: Pillars tab weights. Steps: (1) split the ~15-min creative budget across practices proportionally to what the top pillars imply; (2) record the split in the run log.
