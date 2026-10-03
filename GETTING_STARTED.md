# RoleRadar — Getting Started

*v1.25 — 2026-10-02*

You have a zip with 10 files. Here's how to turn it into your own self-improving job-scan system: 20–30 minutes of your time, plus background build time while Muse verifies every source.

## What you need

- **Muse** (the app or web).
- **The wrapper**, `blurb-prompt.md` — ships *alongside* the zip, not inside it. It's the entry point: you copy it first.
- **The 10 files**, inside the zip: `methodology.md`, `TDD.md`, `onboarding-protocol.md`, `job-scan-spreadsheet-template.xlsx`, `GETTING_STARTED.md` (this guide — start here), `MANIFEST.md`, `README.html`, and `portal-template/` (`index.html`, `notifications.html`, `README.md`).
- **Your resume** (file or LinkedIn URL) and 5 minutes for an interview.
- Optional but recommended: **connect your Gmail** so the scan can read your job alerts and recruiter mail (read-only — it never sends anything). If you skip this, the inbox track is skipped and noted as reduced coverage.

## Step 0 — Check your plan supports scheduled tasks (do this first)

The whole system runs on Muse's scheduler. Before the interview, verify your plan supports scheduled tasks (check your Muse plan details — if you're unsure, ask Muse "does my plan support scheduled tasks?"). **If it doesn't, stop here** — the interview is pointless without the scheduler.

## Steps

1. **Copy the wrapper first.** Open `blurb-prompt.md` (next to the zip) — you'll paste it in step 3.
2. **Open Muse** and start a **new side chat** — a separate conversation thread, so the job search lives in its own space away from your daily chat.
3. **Attach the zip** (`roleradar-v1.25.zip`) and **paste the wrapper**. Send. Muse unzips the kit, reads the methodology, TDD, and onboarding protocol, and starts the interview.
5. **Do the interview.** 20 questions, about 25 minutes: unnumbered intro + 20 questions, one per turn. Exactly one question per turn — every question starts with the count ("3 of 20") and ends with what's expected. The builder drafts from your resume; you mostly correct:
   - *Intro (not numbered):* what RoleRadar is and what's coming, each with what it's for. The first question after it is #1.
   - *Question 1 — resume:* share it (upload/paste), or the read-back starts right away if you already attached it.
   - *Question 2 — read-back:* its 5-line take on you. Correct it or say "looks right."
   - *Question 3 — north star:* your ideal role in 2–3 sentences.
   - *Question 4 — dream names:* 2–3 people, companies, or roles you dream about.
   - *Question 5 — dealbreakers:* what makes you say no, across categories.
   - *Question 6 — drafts:* your taste/anti-profile presented for correction.
   - *Question 7 — comp numbers:* base, total comp target, walk-away floor.
   - *Question 8 — logistics:* cities, what remote means, travel, visa.
   - *Question 9 — timeline:* employed, start-by date, urgency.
   - *Question 10 — scoring model:* your 4–7 pillars + weights as a heatmap. Adjust or say "lock it."
   
   Then it validates the whole setup with a "would you take it?" test on hypothetical roles built from real companies before building.
6. **Ten setup questions.** *Where your stuff lives* — backend, front end, notification (one question each). *How you're getting information* — can it scan your inbox for job alerts? For interview and pipeline progress? *Cadence* — how often, which mornings, what time. *Dossier prefs* — anything the dossier should always cover. *Pipeline prefs* — how you want to work the pipeline.
7. **Let it build.** Muse writes your config, seeds your source registry (companies, keyword searches, feeds, people to monitor), verifies every source URL itself — then shows you the registry: "here's where I'll hunt, add or remove anything?" It writes your scoring anchors, validates the setup with a "would you take it?" test, builds your portal, and creates your 🎯 RoleRadar thread. Then it schedules the scans and asks: "first batch goes out {time} — want it now instead?"

Spotted a role yourself? Paste its URL with **+ Add job URL** in the pipeline — it gets gated, scored, and added like any other find.
8. **First scan runs** on the next scheduled morning. In your 🎯 thread you get: a one-page system card (your pillars, weights, anchors, source counts — the "it gets me" recap), a TL;DR of the most promising roles, and the portal link. The report doc holds the full role cards; your spreadsheet pipeline stays up to date.

Bookmarked a role? Tap **I'm interested** — three quick questions: build the dossier (and may I store it on Drive?), follow you through the process (in the portal or the sheet? — interview and offer emails trigger things automatically if you granted inbox access at setup), and which proactive help you want — I suggest the concrete menu (interview prep, follow-up drafts, deadline nudges, negotiation prep) and you toggle. The dossier outline comes back for your steer before anything gets researched.

## How we work together

- **Getting info from email** — if you granted access, job alerts become a feed, and interview invites, recruiter follow-ups, offers, and rejections update your pipeline on their own and trigger the right help (interview prep, follow-up drafts, negotiation prep). If you didn't, just report outcomes in the thread — "I applied", "rejected after round 2" — and everything still works.
- **The spreadsheet** — every scan reads it, adds new roles, updates statuses, and merges by role ID. Your side (thumbs, bookmarks, statuses, notes) is never overwritten. It's the system's memory; the portal is the window.
- **Dossiers** — built when you tap "I'm interested": outline first for your steer, then the research. Stored as versioned files and refreshed as the process moves.
- **Delivery** — each run lands in your 🎯 RoleRadar thread: one-liners with context, top picks up top, dream-vector leads headlined, portal link quiet in the corner. Rejections stay flat — no pep talk, no fanfare. The report doc holds the full cards.
- **Your steering** — thumbs up/down on any card, "I'm interested" for the full flow, corrections anytime ("that 7.5 is wrong — the comp is below my floor" → it corrects the role and tightens the config). "Add Acme Corp to my companies" → it validates the careers page and adds it.
- **It learns** — sources that produce get expanded, ones that don't get checked less often; your outcomes tune the scoring over time. After a month, the Runs tab shows you exactly what your scan is doing.

## Logistics — where everything lives

- **Sources** → the registry tabs in your spreadsheet (Companies, Boards, Feeds, People) — in whichever backend you picked. Add or remove in chat ("add Acme Corp") or edit the sheet directly.
- **Proposals** → the scan's proposed roles land in the pipeline each run; dossier outlines are proposed in the thread for your steer before any research starts.
- **Pipeline** → the spreadsheet is the system of record (Roles + your UserState, never overwritten); the portal is where you manage it — thumbs, statuses, "I'm interested", + Add job URL.

## Your weekly 10 minutes (the conversational feedback loop)

The system learns from you, not just from its own loops. Anytime — not just after a scan — reply in the side chat:
- "Add Acme Corp to my companies" → it validates the careers page and adds it.
- "That 7.5 is wrong — the comp is below my floor" → it corrects the role and tightens the config.
- "I'm pausing for a month" → see below.

Role outcomes ("I applied", "rejected after round 2") feed the scoring loops — the difference between a tracking system and a learning one.

## Pausing or stopping

- **Pause:** tell Muse in the side chat ("pause the scan until March"). It disables the schedule; your config, registry, and pipeline stay intact.
- **Resume:** "resume the scan" — it re-verifies the registry URLs first (boards move), then restarts the schedule.
- **Stop for good:** "delete the schedule and archive everything." Your spreadsheet and report doc remain yours.

## What adapts to you vs what's fixed

- **Yours:** the candidate profile (level, comp, locations, taste, anti-profile, north stars), the pillar weights, the source registry, the schedule.
- **Fixed:** the algorithm (gates → scoring), the execution contract, the self-improvement loops, the delivery formats. These are the product — they don't change between candidates.

## Notes

- The scan is **read-only**: it never contacts employers, recruiters, or anyone on your behalf. Every role is verified live on the employer's own board before it reaches you.
- **Privacy:** your resume and comp details live in your spreadsheet's Config tab and your Muse workspace. They're never sent anywhere except the sources the scan checks (company career pages, job boards). Nothing is shared, sold, or published.
- The system gets better over time: sources that never produce get checked less often, sources that hit get expanded, and every run's findings are logged. After a month, review the Runs tab — it shows you exactly what your scan is doing.
