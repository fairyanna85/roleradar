# TDD: RoleRadar Self-Improving Job-Scan System

*v1.25 — 2026-10-02*

**Audience:** a fellow LLM builder who needs to reimplement or adapt this system. Assumes you have: a scheduler (cron), a worker runtime with web search + page fetch + shell, a spreadsheet API, and a chat surface for delivery.

**What it is:** a recurring agent that scans the job market for a specific candidate, applies eligibility gates then affinity scoring (see `methodology.md`), and delivers a ranked shortlist — while getting better at its job every run through three feedback loops.

**Provenance:** built and battle-tested over 8 runs (2026-09-23 → 2026-10-01), ~315 roles evaluated, 4 consecutive scheduler failures diagnosed and fixed. The failure modes in §8 are real incidents, not hypotheticals.

---

## 1. Architecture

```
┌──────────┐     ┌─────────────────────────────────────────────────┐
│ Scheduler │────▶│ Worker (single agent, sequential, timeboxed)    │
│ ⟨config:  │     │  W3 inbox scan (20m) → W4 exec/role-model (45m)  │
│ mornings⟩ │     │  → W2 geo track (45m) → W1 board sweep (75m)     │
└──────────┘     │  → merge → gate → score → verify → deliver (45m) │
                 └─────────────────────────────────────────────────┘
        │                                    │
        ▼                                    ▼
┌──────────────────┐              ┌──────────────────────┐
│  State stores    │              │  Delivery surfaces   │
│  - Source registry (4 tabs: Companies/People/Feeds/Boards) │  - Chat summary (leaderboard + 80/20 close) │
│  - Candidate config (Config tab)        │  - Report doc (cards, newest first)         │
│  - Role pipeline (spreadsheet tab)      │  - Dashboard (full-state sync)              │
│  - Seen-postings snapshot (JSON)        │  - Run log (daily memory note)              │
│  - Source-yield tracker (JSON)          │                                             │
└──────────────────┘                      └──────────────────────┘
```

Schedule = `[CONFIG] SCAN_SCHEDULE` (days/times + cadence — daily | every 2 days | twice-weekly | weekly — confirmed at intake, never hardcoded). **W2 is optional:** if the candidate has no secondary market, skip it and fold its 45m into W1.

**Tracks** (each is a coverage domain, not a parallel worker):
- **W3 · Inbox scan** — candidate's email: job-alert digests, recruiter outreach, niche job-board emails. Fastest; do first. If no inbox is connected, skip W3 and log reduced coverage in the run log.
- **W4 · Exec & role-model watch** — monitored execs' social posts, role-model ventures' boards, amplified opportunities.
- **W2 · Geo track** — the candidate's secondary market, if any: native channels first (local job boards, exec-search mandates), employer boards second. Skipped when there is none.
- **W1 · Board sweep** — the source registry URL-by-URL: niche company lists, large-caps, prior shortlist names, direct keyword searches. Includes ~15 min of creative discovery (step 6).

---

## 2. Execution Contract (read this before anything else)

These constraints exist because each was violated once, expensively:

1. **Single worker, sequential, no descendants.** The headless scheduled-run runtime cannot resolve child agents — a contract that told the worker to "dispatch 4 parallel subagents" failed 4 consecutive runs deterministically ("failed while waiting for descendant subagents"). Scheduled work must be single-agent and sequential. Manual/chat recoveries may still fan out.
2. **Hard timeboxes per track.** W3 20m / W4 45m / W2 45m / W1 75m / merge+deliver 45m. A track that exceeds its box stops immediately.
3. **Merge-at-deadline.** A slow track may NEVER zero out the whole report. Partial coverage is reported honestly ("reduced coverage: tracks X, Y incomplete") — never presented as a full scan.
4. **Per-track checkpoints.** After each track, write findings to `scan-YYYY-MM-DD-worker<N>.json` so partial progress survives a later failure.
5. **"Swept, dry" ≠ "not completed."** A track that finished with zero qualifying roles reports the former; they are different states and the run log distinguishes them.

---

## 3. Data Stores & Schemas

### 3.0 Candidate config (spreadsheet tab `Config`) — the canonical home
Everything the intake produces lives here as key/value rows. The worker loads it at pipeline step 1. Nothing lives only in chat context.

| section | key | value | notes |
|---|---|---|---|
| profile | `CANDIDATE_NAME`, `CURRENT_LEVEL`, `BASE_ANCHOR`, `TC_TARGET`, `GEO_SET`, `TRAVEL_CAP`, `LEVEL_FLOOR`, `walk_away_floor` | … | methodology §1 |
| mandates | `MANDATE_BUCKETS`, `EXPERTISE_TAGS`, `TASTE_PROFILE`, `ANTI_PROFILE`, `NORTH_STAR_LIST`, `SECONDARY_MARKET_BANDS` | … | methodology §1 |
| scoring | pillars_tab | see the Pillars tab | pillar names, definitions, weights, 3/6/9 anchors (TDD §3.3) |
| schedule | `SCAN_SCHEDULE` | e.g. Tue + Fri 08:00, twice-weekly | days/times + cadence (daily / every 2 days / twice-weekly / weekly), confirmed at intake |
| assets | `base_resume_path` | e.g. workspace/base-resume.pdf | versioned file; tailor-resume reads it |

### 3.1 Source registry (spreadsheet — the input side)
The registry is the hunt's checklist: **where the scan looks**. Four typed tabs, one per source kind. Every tab has an `active` column (TRUE/FALSE, default TRUE) — the worker skips `active=FALSE` rows but keeps their yield history (soft delete).

**`Companies`** — every employer board monitored.

| company | niche | domain | career page URL | URL status | notes | active |
|---|---|---|---|---|---|---|
| Name | Niche bucket (frontier AI lab, EdTech, …) | Company domain | Canonical check target (official page or ATS board) | verified / blank | Anything the worker needs | TRUE |

**`People`** — every monitored person.

| name | title | company | X handle | X profile URL | date added | background match | evidence_url | active |
|---|---|---|---|---|---|---|---|---|
| Name | Title | Company | @handle (never invented) | x.com/… | ISO date | Why they're watched: lookalike profile, role model, orbit exec, … | Public evidence URL for orbit links (methodology §3.1) | TRUE |

**`Feeds`** — machine-readable job feeds (ATS board APIs — faster and more reliable than scraping pages).

| feed name | feed URL | type | purpose | notes | active |
|---|---|---|---|---|---|
| e.g. Greenhouse board API | `https://boards-api.greenhouse.io/v1/boards/{board}/jobs` | ATS API | New postings as JSON per company board token | `{board}` = per-company token | TRUE |
| e.g. Lever postings API | `https://api.lever.co/v0/postings/{site}?mode=json` | ATS API | New postings as JSON per company site | `{site}` = per-company slug | TRUE |
| e.g. Ashby posting API | `https://api.ashbyhq.com/posting-api/job-board/{board}` | ATS API | New postings as JSON per board | `{board}` = per-company slug | TRUE |

**Token resolution:** the worker discovers each company's ATS + board token on first check and records the fully-resolved feed URL in that company's `notes` cell (Companies tab) — the `{board}`/`{site}` placeholders are never resolved from memory.

**`Boards`** — query boards and search surfaces.

| board | URL | how to use | notes | active |
|---|---|---|---|---|
| e.g. a16z jobs board | https://a16z.com/jobs | Director-tier across buckets; AI/EdTech emphasis | Discovery only — verify on employer board | TRUE |
| e.g. LinkedIn Jobs US | https://www.linkedin.com/jobs/search/?keywords=… | Director+ seniority filters, candidate's home market | Discovery only | TRUE |

**Registry rules:** never an aggregator URL as a check target (only official company/ATS pages); never invent an X handle; if a board permanently moves, the worker updates the URL so the next run follows; blank URL = discover per run and log what was used. New gem sources are appended with today's date (the feedback loop).

### 3.2 Role pipeline (spreadsheet tab `Roles`) — 37 columns
`role_id, company, title, score, dream_bonus, dream_person, is_gem, gem_practice, p1, p2, p3, p4, p5, p6, p7, level, ic_mgr, comp_band, comp_is_estimate, comp_diff, comp_source, equity, equity_note, signon_buyout, signon_note, location, travel, hiring_manager, process, mandate_summary, why_fit, one_liner, apply_link, status, last_verified, weakest_pillar, source`

`role_id` is the merge key (slug: `{company}-{title}-{location}`). Normalize before slugging: lowercase, strip legal suffixes (Inc, LLC), map city aliases (NYC → New York), match on company domain where known — the same role must always produce the same id. `p1`–`p7` are the candidate's pillar scores (1–10), mapped by the Pillars tab — unused pillars stay blank. `weakest_pillar` names the lowest-scoring pillar (feeds the card + outcome learning). `is_gem` (TRUE/FALSE) + `gem_practice` (one of the 7 practice names) record creative-discovery provenance — the yield tracker and Stats tab count gems from these flags. `source` records pipeline provenance: `manual` for user-added URLs, otherwise the practice or registry tab that found it. `one_liner` is the card's one-line assessment. `level` ∈ {"C-level","SVP","VP","Director","Head of","L7","L8",…} — the tier label as written, cross-checked to `CURRENT_LEVEL`. The sheet is the system of record.

### 3.3 Pillars tab — the candidate's scoring rubric
One row per pillar (4–7 rows). The intake defines these; the reference set in methodology §3 is one instance.

| pkey | name | definition | weight | anchor_3 | anchor_6 | anchor_9 | version_date |
|---|---|---|---|---|---|---|---|
| p1 | e.g. Learning velocity | e.g. how fast this seat builds rare skills | 0–1 | what a 3 looks like | what a 6 looks like | what a 9 looks like | ISO date |

Weights are free; the composite divisor is always their sum. Any weight or anchor change is versioned with a new date — never edited in place silently.

### 3.4 Seen-postings snapshot (`job-scan-seen.json`)
Every role **ever evaluated**, including cuts. Shape:
```json
{ "roles": { "<role_id>": {
    "company": str, "title": str, "seen": "YYYY-MM-DD",
    "status": "open|watchlist|below-bar|cut|closed|dead|parked",
    "location": str, "score": float|null, "link": str|null,
    "cut_reason": str|null, "note": str|null } } }
```
This is the dedup boundary: nothing is re-evaluated without a recorded reason.

### 3.5 Source-yield tracker (`source-yield.json`)
```json
{ "sources": { "<tab>::<name>": {
    "tab": "Companies|People|Feeds|Boards", "name": str,
    "checked": int, "roles_found": int, "gems_found": int,
    "last_yield": "YYYY-MM-DD"|null, "cadence": "every run|every other run" } },
  "reviews": [ { "date": str, "demoted": [], "promoted": [], "query_changes": [] } ] }
```
Keys join to registry rows on (tab, name); the portal's per-card yield stats read from here. `gems_found` increments from the Roles `is_gem` flags.

### 3.6 Run log — two artifacts, don't confuse them
- **Runs tab** (spreadsheet): one throughput row per scan run — `run date, sources checked, roles detected, cut, qualified, added, gems (creative), new sources, coverage gaps, notes`. Pipeline step 7 appends the row.
- **Daily run note** (memory/log): narrative per-run notes — counts, per-source yield vs dry, coverage checklist (which boards swept, which JS-gated + fallback used), gaps (visible, never silent).

---

## 4. Pipeline (one run, in order)

1. **Load state** — candidate config (Config tab), pipeline sheet, seen snapshot, source registry, yield tracker.
2. **W3 → W4 → W2 → W1** — scan per §1, checkpoints after each.
3. **Merge & dedupe** against the seen snapshot.
4. **Gate** (§2 of methodology): G1 location → G2 level/authority → G3 culture → G4 active verification → G5 logistics. Record cut reasons in the snapshot.
5. **Score** gate-passers (§3 of methodology): the candidate's pillars, composite, affinity bonus, comp rules, IC/MGR label.
6. **Creative discovery** — reserve ~15 min of W1's 75-min box for the 7 practices. A *perfect gem* = perfect fit for the candidate's taste **and** expertise (both spelled out in the candidate config — never a generically strong role). The practices, each with its execution:
   - **Lookalike profiles:** people with the candidate's trajectory arc at adjacent companies — sweep their employers' boards.
   - **Problem-led search:** search the candidate's *problem domain*, not just titles — translate per function (e.g. support: public support-quality crises, orgs scaling support 10x, support-led growth motions).
   - **Funding triggers:** newly funded companies in the candidate's domains — announcement → careers page within days.
   - **Alumni tracing:** the candidate's past employers' alumni now hiring managers elsewhere — their orgs' boards.
   - **Investor-taste cascade:** pick 3–5 investors whose taste matches `TASTE_PROFILE`; sweep their recent portfolio additions' careers pages.
   - **Spinout radar:** new ventures spun out of registry companies or people — catch them before they're on any list.
   - **Dream-person curation:** north-star people/ventures' boards + socials, always — the highest-signal practice.
7. **Update state** — snapshot (all evaluated incl. cuts), pipeline sheet (upsert by `role_id`; set `one_liner`, `is_gem`, `gem_practice`; dead roles → `status=parked` with reason+date), source registry (new gem sources appended with the tab's URL column + date), yield tracker (per-source counts), Runs tab (append the row).
8. **Self-improvement review** (every 4th run, ~fortnightly): demote/promote sources per §5.2; mine cut reasons per §5.3.
9. **Deliver** — chat summary, report doc, dashboard sync (§6).

### 4.1 Targeting logic (signal → hypothesis → practice)
The reasoning layer above the 7 practices (methodology §7). Each row: the signal, the config field it reads, the practices it feeds, and when it runs. Seed the registry from the build-cadence rows at build; re-derive the per-run rows every run.

| Signal | Config field | Procedure (methodology §7.1) | Cadence |
|---|---|---|---|
| Trajectory arc | Resume Phase 1 read | P1 lookalike extraction | Build; refresh if the resume changes |
| Spike | `EXPERTISE_TAGS` | P2 problem translation | Per run |
| Role families | `MANDATE_BUCKETS` | P3 mandate expansion (+P2 per run) | Build; per-run for problem-led |
| North-star list | `NORTH_STAR_LIST` | P4 dream-venture mapping | Per run |
| Taste profile | `TASTE_PROFILE` | P5 taste→language | Build; per run |
| Dealbreakers | `ANTI_PROFILE`, floors, `GEO_SET` | P6 negative space | Build (gates enforce per run) |
| Pillar weights | Pillars tab | P7 effort allocation | Per run |
| Career stage | Inferred (Phase 1) | Source selection | Build |
| Past employers | Resume | Alumni tracing | Build; per run for new moves |
| Funding events | — (external) | Funding triggers, spinout radar | Per run |

Rule: every creative target added to the pipeline records its practice in `gem_practice`; the run log records the hypothesis (which signal drove it). A target with no traceable signal is not chased.

### 4.2 Source taxonomy — universal vs specialized
**Universal sources** (every instance, no exceptions):
- Every Companies-tab career page (direct, official — never aggregators).
- LinkedIn Jobs keyword queries — one per `MANDATE_BUCKETS` entry, with seniority + `GEO_SET` filters.
- The candidate's email job alerts (if `EMAIL_ALERTS` granted) → ingested as a Feeds row.
- People-tab socials (X/LinkedIn) for hiring signals.

**Specialized sources** (the builder picks 2–4 per candidate domain at build):
- Niche job boards for the domain (e.g. AI safety, academia, climate).
- Investor portfolio job pages matching `TASTE_PROFILE`.
- Community boards (Slack / Discord / Substack) in the candidate's domains.
- Exec-search partner pages (exec-stage candidates).

**How people and companies get identified:** people via P1 (lookalike extraction) + P4 (dream-venture mapping) → People tab with a `why` note + evidence URL; companies via P1 employers + P4 ventures + P2 problem-exhibitors + funding triggers → Companies tab. Never invented — every registry row carries a verifiable URL.

---

## 5. Self-Improvement Subsystem

Three loops, different timescales:

### 5.1 Source feedback loop (per run)
Any gem from a source **not** on the registry gets appended to the right registry tab (with the tab's URL column + date). The registry grows a little almost every run. This is how network-sourced windfalls compound instead of evaporating.

### 5.2 Yield reallocation (every 4th run, ~fortnightly)
Read `source-yield.json`:
- **Demote:** zero gems over the last 6 runs → `cadence: every other run`.
- **Promote:** 2+ gems in the last 6 runs → expand: find 3 lookalike companies, append to registry.
- **Audit (engagement):** flag sources whose roles got zero engagement (no thumbs up/down) across the last 6 runs → propose retiring them to the candidate. Never auto-delete — the candidate confirms.
- Record the review in `reviews[]`.

### 5.3 Cut-reason mining (every 4th run)
Tally `cut_reason` over the last 3 runs. If one reason exceeds 50% of cuts, tighten the discovery queries to filter it earlier (e.g. seniority filters if `ic-seat` dominates) and note the change in the run log.

### 5.4 URL maintenance (per run, opportunistic)
If a board permanently moved, update it per tab — Companies: `career page URL`/`URL status`; People: `X profile URL`; Feeds: `feed URL`; Boards: `URL`. Blank URL = discover per run + log what was used (no guessing).

---

## 6. Delivery Contracts

### 6.1 Chat summary (mobile-first, scannable)
Deliveries go to a dedicated side thread named "🎯 RoleRadar", created at build time. Every run: sync the portal/dashboard first, then post the TL;DR in the thread with the portal link.
0. **TL;DR header** — one line on what this run found ("3 new qualifying roles, 1 at a north-star"), then the portal link.
1. **Dream-vector block** (above the leaderboard): roles at north-star ventures, each naming the person/venture connection. Then an **orbit block** one tier below (connection named, never outranking a dream lead). Never invent a connection. This headlining rule is the system's emotional core — "it hunts what I dream about" — and must never be buried under flags.
2. **Top-3 leaderboard** — `🥇 **<score>** — <company> · <title>` + one-liner. Score first, never dotted leaders.
3. **One line per remaining role** — score + company/title + one-liner.
4. **80/20 close** — the ONE highest-leverage move as a concrete offer presented as an **interactive option the candidate accepts in one tap** (a widget/option, never a passive bullet point).
5. **Report doc link.**
6. Quiet week = exactly one line: "no new qualifying roles this scan." Never inflate.
7. **Language & currency:** deliver in the candidate's intake language and currency (e.g. a French candidate's € bands) — never default to English/$.
8. **First run only — the system card:** a one-page config recap (pillars + weights, anchors summary, source counts, what the config says about them). The screenshot-able "it gets me" artifact.
9. **Between runs:** a gem scoring ≥8.0 at a north-star venture may be pushed immediately rather than waiting for the next run.

### 6.2 Report doc
Running document, newest scan on top. One **role card** per qualifying role (see methodology §4). Update procedure: export current doc as text first — if the candidate edited it since the last upload, **do not overwrite**; flag the conflict instead.

### 6.3 Dashboard sync — FULL-STATE REPLACE semantics
The dashboard action **replaces** the entire jobs state with exactly the rows pushed. Therefore:
- **Every push carries the complete row set.** A partial push wipes everything else. This is the #1 footgun.
- **Never hand-emit large payloads.** For payloads beyond a few KB, hand-transcription introduces defects on nearly every attempt (proven repeatedly). The reliable path is a builder-side file-restore action that reads the staged JSON file byte-exact and runs it through the action handler logic.
- **User state is sacred — read-modify-write.** The worker reads the live `UserState` first and carries it forward verbatim in every push; the replace scope is the role-data rows only. `UserState` is never overwritten, even on full-state sync.
- **Verify every push** with a mechanical read-back diff (replicate the handler's transforms; exit 0 = clean) before declaring done. Counts alone don't verify — diff field-by-field.

### 6.4 Portal (dashboard) spec

A web dashboard the candidate opens daily. Four tabs. The builder LLM creates it per this spec.

**Data model — two planes, strictly separated:**
- *Role data (scan-owned):* the 37-column pipeline rows. The scan writes; the portal never edits these fields.
- *User state (candidate-owned):* a separate `UserState` tab keyed by `role_id`: `role_id, bookmarked (bool), thumb (up/down/none), user_status (new/applied/interviewing/offer/rejected/parked/dismissed), tailored_resume_link, dossier_link, notes, updated_at` — plus the interest-flow columns: `dossier_wanted (bool), drive_ok (bool), track_process (bool), track_surface (portal/sheet), proactive` (comma-separated toggles from the menu below; empty = none). The scan NEVER writes here. This separation is what makes "user state is sacred" enforceable even under full-state sync.

**Instantiation definition of done** (portal-template/README.md): every demo toast replaced with a real action; all interpolated role/company/candidate data escaped before `innerHTML`; job URLs allowlisted to `https://` on the employer's own domain; `status` per the §3.2 enum; verified with read-back before declaring done.

**Tab 1 — Stats.** Glanceable counts, zero interaction required: candidates in pipeline; bookmarked; active by status (applied / interviewing / offer); new this week; average score of active pipeline; gems found (creative-discovery count, from the `is_gem` flags). Each count deep-links to the filtered Pipeline view.

**Tab 2 — Pipeline.** One card per active role: score with pillar-heatmap mini-strip, company, title, comp band, location, level · IC/MGR, one-line assessment (from `one_liner`), link to the live job description. Right side of each card: thumb-up / thumb-down.
- Thumb-up → `thumb=up`, `bookmarked=true`; the card appears in the Bookmarked tab.
- Thumb-down → `thumb=down`, `bookmarked=false`, `user_status=dismissed`; the card falls off the pipeline immediately (recoverable from history — never hard-deleted).
- Re-tapping the active thumb clears it (`thumb=none`) and restores the prior state.
- Sorted by score descending; filterable by pipeline status (`user_status`), scan status (`status`), location, pillar.
- **Add job URL:** a button above the list — paste any posting URL → the worker fetches the JD, runs gates + scoring, and adds it to the pipeline with `source=manual`. Duplicates merge by `role_id`; gated-out URLs are reported with their cut reason, not silently dropped.

**Tab 3 — Bookmarked.** The candidate's shortlist. Each card carries action buttons:
- *Tailor resume* → rewrites the candidate's base resume (read from `base_resume_path` in the Config tab) against this JD's minimum + preferred requirements. Hard rule: reframe and emphasize only — never invent experience, titles, dates, or numbers. Output stored as a versioned file linked from the card (downloadable).
- *Role interest flow* — when the candidate bookmarks a role or taps "I'm interested", ask three precise questions (one message, quick picks):
  1. **Dossier:** "Want me to build the dossier on this one? Can I store it on Drive?" → if yes: propose the outline (standard sections below, through the candidate's pillar lens) + "what else do you want covered?" → build on "go". Storage permission is captured here, at point of need.
  2. **Process tracking:** "Want me to follow along through the process — interviews, negotiations? Where do we interact: portal or sheet?" (Email watch was granted or declined at setup, Q15 — the triggers fire on their own when `EMAIL_WATCH` is on, otherwise on the candidate's report.) The role's card becomes the interaction surface; stage changes — detected or reported — drive the timeline.
  3. **Proactive help:** never an open "want help?" — the builder *suggests* the concrete menu (interview-prep brief before each round, follow-up note drafts, deadline nudges, negotiation prep against your floor/target, open-questions tracker) and the candidate toggles items on/off.
- *The dossier brief itself:* the role, the company (business, culture signals, financial health), the hiring team/manager (background, public writing, tenure) — one evidence-backed section per pillar. Stored as a versioned file `dossier-<date>.md` in the confirmed location; URL in `dossier_link`.
- *Proactive menu (trigger → output):* each toggle fires on its trigger — auto-detected from email when `EMAIL_WATCH` is on, otherwise on the candidate's report — and delivers to the `track_surface`, unless the candidate says otherwise —
  - `interview_prep`: `user_status` → interviewing → gap brief through the pillar lens (strengths to lean on, gaps to prepare, questions to ask), in the thread + on the card.
  - `followup_drafts`: candidate reports a round done → thank-you / follow-up note draft.
  - `deadline_nudges`: a decision deadline is known, or a stage ages past 7 days → nudge in the thread.
  - `negotiation_prep`: `user_status` → offer → comp analysis vs the candidate's floor/target, with walk-away math.
  - `open_questions`: ongoing → maintained list of open questions on the role's card.
- *Drive permission rule:* ask per role at the point of need ("Can I store it on Drive?"). If the candidate answers "always", set Config `DRIVE_ALWAYS=true` and stop asking.
- *Email watch (event triggers):* see §6.6.
- *Status toggle* → new → applied → interviewing → offer → rejected (or parked). Drives the Stats tab.
- *Interview prep* → when `user_status` moves to interviewing: a role-vs-candidate gap brief through the pillar lens (strengths to lean on, gaps to prepare for, questions to ask).
- The dead-JD rule applies here too: a posting confirmed offline is marked `parked` and falls off (see constraints).

**Tab 4 — Sources.** The input side, read live from the registry's four tabs and shown as four sub-tabs: **Companies** (cards with niche, domain, career-page URL + verification status), **People** (cards with title, company, X profile link, background-match note), **Feeds** (the ATS/API feeds), **Boards** (query boards with usage notes). The candidate can **add** a company or person (validated: official pages only, never aggregators, never invented handles) or **delete** one (soft — `active=FALSE`, yield history survives). Feeds/Boards additions are worker-suggested: the candidate can propose one, the worker validates the feed/board before adding. Writes go straight to the sheet — the sheet is the system of record and the next scan reads it. Each card shows its yield stats (checked ×N, gems found, last hit) from the yield tracker — the self-improvement loop made visible.

**Constraints (hard rules):**
1. *Dead JDs fall off automatically.* The scan's pipeline sanity check re-verifies every active role's posting each run; a JD confirmed dead/closed flips to `parked` (with reason+date) and the portal removes it from Pipeline and Bookmarked views (counts update). "Verification inconclusive" never removes — only confirmed-dead.
2. *Thumbed-down cards fall off the pipeline* immediately into `dismissed` (recoverable, never deleted).
3. *User state is sacred.* Scan pushes are read-modify-write: merge by `role_id`, touch only the 37 role-data columns; `UserState` is never overwritten, even on full-state sync.
4. *Staleness visibility.* Every card shows `last_verified` age; roles unverified for >7 days carry a subtle "re-check pending" marker until the next scan refreshes them.
5. *Outcome learning.* When `user_status` moves to applied/interviewing/offer/rejected, the worker logs the outcome with the role's pillar profile. Rejected-role patterns feed the cut-reason mining; interviewed-role patterns inform weight suggestions at the quarterly anchor review. This is what makes it a learning system rather than a tracking system.

### 6.5 Notification shapes (event triggers)
Every notification follows the same card anatomy: eyebrow (`RoleRadar · {time}`, plus `· detected from email` when applicable) → headline (emoji + event + title @ company) → score chip where scored (the visual anchor; color by tier: 8.5+ green, 7–8.4 blue, below 7 grey) → one context line, never a paragraph → exactly one verb-first tap (rendered as the chat's single-tap option) → quiet portal link bottom-right. The channel is the Q13 pick (dedicated side thread "🎯 RoleRadar" by default). A visual prototype lives at `portal-template/notifications.html`.
- **Run briefing** — `☀️ Morning briefing — {n} new roles` / dream-vector hero (gold card: `◆ DREAM VECTOR`, score chip, title · company, one-line why) / ranked rows (score chip + company · title + one-line why each) / tap: *Build the dossier on {top pick} →*.
- **Interview** — `📅 Interview — {title} @ {company}` / `{round} · {interviewer names} · {when}` / tap: *Get my prep brief →*.
- **Offer** — `💰 Offer — {title} @ {company}` / `{one-line walk-away math vs floor/target}` / tap: *Open negotiation brief →*.
- **Deadline nudge** — `⏰ Nudge — {company} wants an answer by {date}` / `{what it's for + why it matters}` / tap: *Show me the thread →*.
- **Dossier ready** — `📄 Dossier ready — {title}, {company}` / tap: *Open dossier →*.
- **Quiet week** — `No new qualifying roles this scan.` One muted line. Never inflate.
- **Rejection logged** — `Noted — {company} passed. Logged for outcome learning.` One flat line. No pep talk, no fanfare.

### 6.6 Email watch (event triggers)
*Source:* the candidate's email via the mail connector/skill — only when Config `EMAIL_WATCH=true` (asked at setup, Q15).
*Scope:* matching is scoped to tracked companies and role titles only. Unrelated email is never read; email content is never stored beyond the extracted event (company, stage, date, names).
*Event patterns:*
- `interview_invite` — invitation or scheduling email for a tracked role (interviewer names, calendar invite, "we'd like to invite you").
- `offer` — offer letter, verbal-offer recap, or comp/benefits thread naming numbers.
- `negotiation` — counter-offer discussion, equity/benefits negotiation.
- `rejection` — rejection email → `user_status=rejected` + outcome-learning log.
- `deadline` — explicit response deadline ("please let us know by Friday").
*Dedup:* seen message IDs are recorded per run; the same email never fires twice.
*Cadence:* checked on every scan run.
*Conflict handling:* a detection may only advance `user_status` forward along new → applied → interviewing → offer → rejected, never backward; a stale thread (email older than the current stage) is ignored. Every auto-advance is labeled "detected from email" so the candidate can correct it.
*Revocation:* if access is revoked or errors persist, fall back to manual reporting — one notice, then quiet.

---

## 7. Failure Modes & Hardening (incident log)

| Incident | Lesson |
|---|---|
| 4 consecutive scheduler failures (2026-09-26 → 10-01) | Scheduled workers must never spawn subagents; single-agent sequential only (§2.1) |
| 7 defective hand-emitted payload pushes in one session | Never hand-transcribe payloads >a few KB; builder file-restore is the only reliable path (§6.3) |
| Partial push wiped live dashboard rows | Full-state sync: every push carries the complete set (§6.3) |
| Morning scan results never delivered (worker died pre-merge) | Per-track checkpoints + merge-at-deadline (§2.3, §2.4) |
| Stale conflict: doc edited by candidate mid-run | Export-compare before overwrite (§6.2) |
| Drive file update silently did nothing | Request body goes in `--json`, media bytes via `--upload`; verify with read-back |

---

## 8. Instantiation Checklist (new candidate)

1. Run the intake (blurb prompt) and write everything to the **Config tab** — this is the entire per-candidate surface. Builder warning: **never copy the methodology's example buckets** — derive `MANDATE_BUCKETS`/`EXPERTISE_TAGS` from Phase 1, in the candidate's own function language.
2. Seed the **source registry's four tabs**: Companies ← niche company lists for the candidate's domains; Boards ← 5–10 keyword queries for their markets + native channels; Feeds ← the candidate's email alert sources + ATS APIs; People ← 10–15 exec/social accounts (execs, role models, orbit).
3. Seed the **north-star list** (dream people and/or ventures + orbit with an evidence URL each) or set affinity bonus to +0 and skip.
4. Define the candidate's **pillars** in the Pillars tab: 4–7 inferred pillars with names, one-line definitions, weights, and 3/6/9 anchors (TDD §3.3). The reference set is an example — never the default.
5. Confirm the composite divisor = sum of the weights; version the rubric with a date.
6. Create the **pipeline sheet** (37 columns), **seen snapshot** (`{}`), **yield tracker** (seeded from registry). Save the base resume as a versioned file; record the path in Config.
7. Ask the 10 setup questions — question 11: backend (spreadsheet in the LLM workspace vs on the Muse server); question 12: front end (portal by default; opt out to email digest or none); question 13: notification (email | text | dedicated side thread); questions 14–15: inbox access — job alerts (→ `EMAIL_ALERTS`) and interview-progress watch (→ `EMAIL_WATCH`); questions 16–18: **SCAN_SCHEDULE** — frequency (daily | every 2 days | twice-weekly | weekly), days, time; question 19: dossier standing extras (→ Config `DOSSIER_ALWAYS`); question 20: pipeline prefs (→ Config `PIPELINE_PREFS`) — then schedule; confirm the first run delivers all surfaces. No secondary market → skip W2, fold its 45m into W1. Create the "🎯 RoleRadar" delivery side thread.
8. After run 4, check that the first yield review actually demoted/promoted something — the loop is the product.

## 9. Candidate Onboarding Protocol (intake → instance)

This is the procedure the blurb prompt drives. Follow it exactly — the quality of the instance is bounded by the quality of the intake.

### 9.1 The intake (10 questions, about 6 minutes — then 10 setup questions)

**Protocol:** an unnumbered intro + 10 interview questions + 10 setup questions (the intro states the actual count for this run). Speed beats exhaustiveness — infer from the resume, draft everything, let the candidate correct. The intro is not numbered: one line on what RoleRadar is, then a bullet list of what's coming, each with what it's for; end with a clear call to reply and wait. Then exactly one question per turn — one thinking question stands alone; only quick factual form-fills may share a message under one theme. Two messages present a draft for correction. Every question starts with the count ("3 of 20") and ends with what's expected back. Any question grouping multiple sub-asks must end with explicit response guidance — labeled slots or a reply template ("Reply with base / target / floor") — so the candidate knows the shape and the builder can parse it. Never send two messages in one turn. A sharp follow-up on a vague answer keeps the same number. **Stated-assumption defaults:** when an answer is partial, prefer stating a reasonable default inferred from the resume ("I'll assume X — correct me if wrong") over burning follow-ups — the 5-minute budget is sacred and correction is cheap. Reason silently — the stage inference after message 3 shapes everything after it but is never narrated.

**Question 1 — resume.** If the resume is already in the chat, skip the ask and present the 5-line read directly. Otherwise ask for the file (preferred — LinkedIn URLs are often login-walled), pasted text, or "no resume" (→ manual Q&A).

**Question 2 — the 5-line read:** level + scope, trajectory, spike (→ `EXPERTISE_TAGS`), role families actually done (→ `MANDATE_BUCKETS`), one thing worth probing. End: "Reply with corrections, or 'looks right' to continue." Then **infer the career stage** (junior / mid / senior / exec) silently and use it to shape every later inference. The trajectory also seeds the lookalike-profile discovery practice.

**Question 3 — north star:** ideal role in 2–3 sentences.

**Question 4 — dream names:** 2–3 dream people/companies/ventures/roles.

**Question 5 — dealbreakers:** ask across categories (responsibilities, pay, setup/stage, culture, location) with no leading example — list the categories, never plant an answer. Culture/politics answers → `ANTI_PROFILE` (the G3 screen needs patterns, not vibes); scope answers inform `LEVEL_FLOOR` and mandate fit; pay/location answers get pinned to numbers in question 7.

**Question 6 — present the drafts:** `TASTE_PROFILE` (specific, not "good culture"), `ANTI_PROFILE`, `NORTH_STAR_LIST` (named people, companies, and/or role archetypes; orbit additions need a public evidence URL each — never invent a connection). End: "Correct anything, or say 'good' to continue."

**Question 7 — comp numbers:** "Reply with the three numbers, labeled: base / target / floor." → `BASE_ANCHOR` (posted bands compare to base, never TC), TC target (context), walk-away floor (**hard cut**: posted band max < floor → cut at G2, reason `comp-below-floor`). Derive a first `LEVEL_FLOOR` from the floor — confirm it, don't anchor on it (question 10 sets it properly). If base or floor numbers are still missing after question 7, mark comp-dependent scoring provisional and ask once in the build report — never stall the build on it.
**Question 8 — logistics:** "Reply with each, labeled: cities / remote / travel / visa." → cities + what "remote" means → `GEO_SET`, max travel → `TRAVEL_CAP`, visa constraints (G1/G5 cannot run without these — never skip). Secondary market with different comp norms → `SECONDARY_MARKET_BANDS`, never converted US comp.
**Question 9 — timeline:** employed? start-by? urgency? (tunes the 80/20 close and watchlist surfacing).

**Question 10
**Question 10 — lock the scoring model:** from the resume, stage, and questions 3–5, propose 4–7 pillars — names, one-line definitions, weights (divisor = sum). Never default to a fixed set. Write 3/6/9 anchors per pillar, grounded in intake quotes. Present as the heatmap the candidate will see every run. End: "Adjust anything, or say 'lock it' to continue."

**Question 11 — where your stuff lives: backend:** "Should the spreadsheet live in your LLM's workspace, or on the Muse server?" → determines the scan's read/write path.
**Question 12 — where your stuff lives: front end:** "The web portal is on by default — keep it, switch to an email digest, or none (chat only)?" → portal built by default unless opted out.
**Question 13 — where your stuff lives: notification:** "How do you want to be notified — email, text, or a dedicated side thread?" → notification channel; the side thread ("🎯 RoleRadar") is created at build and pinged on every refresh. Dossier/resume storage (Drive `RoleRadar/<Company>/` vs workspace) is confirmed with the candidate at the first dossier build — not here.
**Question 14 — how you're getting information: job alerts:** "Can I scan your inbox for job alerts (LinkedIn, etc.)? Grant access and I'll ingest them as a feed." → if yes, set Config `EMAIL_ALERTS=true` and add a Feeds row.
**Question 15 — how you're getting information: interview progress:** "Can I scan your inbox for interview and pipeline progress — invites, recruiter follow-ups, offers, rejections? This powers the automatic process tracking." → if yes, set Config `EMAIL_WATCH=true` (triggers §6.6). **Backend:** spreadsheet in the candidate's LLM workspace (direct read/write) or hosted on the Muse server → determines the scan's read/write path. **Front end:** portal by default (opt out to email digest or none). **Notification:** email, text, or a dedicated side thread (created at build; pinged on every refresh — the "🎯 RoleRadar" thread is this option). Dossier/resume storage (Drive `RoleRadar/<Company>/` vs workspace) is confirmed with the candidate at the first dossier build — not here.

**Question 16 — cadence: frequency:** "How often should the scan run — daily, every 2 days, twice-weekly, or weekly?" → `SCAN_SCHEDULE.frequency`.
**Question 17 — cadence: days:** "Which mornings?" → `SCAN_SCHEDULE.days`.
**Question 18 — cadence: time:** "What time?" → `SCAN_SCHEDULE.time`.

**Question 19 — dossier prefs:** "Dossier: beyond the standard sections, is there anything you always want covered? (e.g. comp benchmarking, hiring-manager background)." → Config `DOSSIER_ALWAYS`. Standing default; the per-role "what else?" still applies.
**Question 20 — pipeline prefs:** "Pipeline: the default is score-ranked cards with status filters — want anything different?" → Config `PIPELINE_PREFS`. This is the last question before the build.

### 9.2 Anchor writing (per pillar, per candidate)
For each pillar, write what 3 / 6 / 9 means **for this candidate**, grounded in intake quotes. Example for a Fulfillment-style pillar: "9 = the mission paragraph from Phase 2 almost verbatim; 6 = adjacent domain, culture unknown; 3 = matches anti-profile." Anchors are what make two runs score consistently — without them, scoring drifts. Store in the Pillars tab.

### 9.3 Validation before first run (the "would you take it?" test)
Before scheduling, describe 2–3 hypothetical roles with genuine trade-offs — **built from real companies already in the registry** (a real name feels psychic; a generic hypothetical feels like a quiz). Ask which they'd take. If their picks disagree with the scoring's ranking, **fix the config, not the test** — usually a pillar weight or a gate threshold is wrong. This validation is the primary weight-calibration mechanism (it replaced the old trade-off questionnaire): concrete picks over abstract questions. Repeat until the ranking matches their gut on all cases.

### 9.4 Build order
**Interview** (10 questions: 1–6 getting-to-know-you, 7–9 hard filters, 10 scoring model) → **Setup** (10 questions: 11 = backend; 12 = front end; 13 = notification; 14 = job-alert inbox access → `EMAIL_ALERTS`; 15 = interview-progress inbox watch → `EMAIL_WATCH`; 16–18 = cadence frequency / days / time; 19 = dossier prefs → `DOSSIER_ALWAYS`; 20 = pipeline prefs → `PIPELINE_PREFS`) → Config write → registry seeding → URL verification → **registry review** (show the candidate counts + notable names per tab: "Here's where I'll hunt — add or remove anything?") → anchors → weights lock → validation test → portal (default-on unless opted out) + create the "🎯 RoleRadar" delivery side thread → schedule per confirmed cadence → **first-batch ask** ("First batch goes out {time} — want it now instead?") → first run (fills Roles sheet) → first-run delivery in the 🎯 thread: system card + TL;DR (dream-vector block, top-3 leaderboard, one line per remaining role, 80/20 close as a single-tap option) + portal link + report doc link. The first run's report must also include: what was built, what was found, which sources already proved their worth (early yield signal). The **system card** is a one-page config recap (pillars + weights, anchors summary, source counts, what the config says about them) — the screenshot-able "it gets me" artifact.

### 9.5 What the intake predicts about the instance
- Vague Phase 2 → vague gems. If the candidate can't name what they're running from, the culture gate will be weak — say so.
- If the validation picks disagree with the draft weights, the weights were wrong — fix them and re-validate.
- The north-star list is the highest-leverage 10 minutes: every name becomes a venture sweep + social monitor + potential affinity bonus. Push for 10+ names; companies, ventures, and dream role archetypes count.
- If Phase 1 misjudges the career stage, every later phase misfires — the stage call is the highest-leverage inference in the intake. When unsure, ask.

## 10. Extension Points
- Multi-candidate: the methodology is already parameterized; the open question is registry/pipeline namespacing per candidate (separate tabs vs separate spreadsheets).
- Interview-prep pipeline: specified in §6.4 Tab 3 (gap brief at interviewing status).

## Appendix A — Example artifacts (fictional)

A fictional candidate grounds the shapes below: **Jordan Lee**, VP Operations, Chicago, $180k base anchor, support-org leadership background. Every value is invented — copy the shapes, never the values.

**System card (delivered with the first run):**
> **Your RoleRadar is live.** Here's what it knows about you:
> **Pillars:** Experience fit 20% · Compensation 25% · Status 5% · Fulfillment 15% · Advancement 10% · Values fit 5% (divisor 0.80)
> **Anchors (abridged):** Fulfillment 9 = "support org as the company's growth engine, not a cost center"; 3 = "ticket-factory culture."
> **Watching:** 64 companies · 12 people · 5 feeds · 8 boards. North stars: 2 admired support leaders + 3 companies.
> **How it learns:** dead sources get checked less often; hot ones get expanded; your thumbs and outcomes retune the hunt.

**Dream-vector delivery block (§6.1, item 1):**
> 🌟 **Dream-vector — Northwind (Amara Chen's venture).** VP Customer Operations, Chicago hybrid, $195–230k base. Amara is north-star #2; her thesis on support-led growth is your Phase 2 paragraph nearly verbatim. **Score: 8.40** (+1.0 affinity).
> ▫️ *Orbit:* Corelink — COO praised Amara's operating system publicly ([evidence link]); Director Support Ops, remote-US. **7.10.**

**80/20 close as a single-tap option (§6.1, item 4):** never a bullet point — an interactive option the candidate accepts in one tap, e.g. a "Yes — dossier the Northwind role" choice.
