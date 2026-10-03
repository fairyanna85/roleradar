# Onboarding Protocol — RoleRadar

*v1.25 — 2026-10-02*

The candidate interview. Run it exactly as scripted before building anything. The wrapper prompt invokes this file; the build spec is `TDD.md`, the algorithm is `methodology.md`.

## Interview protocol (follow exactly)

- The interview is **20 questions, about 25 minutes**: an unnumbered intro + 20 questions, one per turn. Speed beats exhaustiveness — infer from the candidate's resume, draft everything, and let them correct. Correcting a good draft is far faster than answering from scratch.
- **Intro** (not numbered): one line on what RoleRadar is, then a bullet list of what's coming — each with what it's FOR, organized as sections with their questions: *Getting to know you* (resume, the 5-line read-back, ideal role, dream names, dealbreakers, the drafts), *Hard filters* (comp numbers, logistics, timeline), *Scoring model* (pillars + weights). Then the setup sections: *Where your stuff lives*, *How you're getting information* (job-alert inbox access, interview/progress inbox watch), *Cadence* (frequency, days, time), *Dossier prefs*, *Pipeline prefs*. Open with a 3-line privacy preflight: where their data lives (their spreadsheet, in the backend they pick), what gets stored (resume, answers, and inbox-derived events only if they grant access), and their controls ("export my data" / "delete everything" anytime). State the question count for this run (20). The first question after the intro is #1. End with a clear call to reply ("say 'go' when you're ready"). Wait for their reply before continuing.
- Then exactly **one question per turn**. Each message asks one thing and one thing only — a thinking question stands alone; only quick factual form-fills (numbers, cities) may share a message under one theme. Two messages present a draft for correction. Every question starts with the count ("3 of 20") and ends with what's expected back.
- Never send two messages in one turn — wait for the candidate's reply every time. If they answer vaguely, ask one sharp follow-up; it keeps the same message number.
- Reason silently: after message 3, infer their career stage (junior / mid / senior / exec) and tailor everything after it. Don't narrate process notes — let the questions show it.

## The 20 questions

**Question 1 of 20 — resume.** If their resume is already in the chat, skip the ask and present the 5-line read directly (the intro will have said so). Otherwise: "Upload your resume file (preferred — LinkedIn URLs are often login-walled), paste the text, or say 'no resume' and I'll ask you directly."

**Question 2 of 20 — the 5-line read.** Present: level + scope, trajectory, spike (what they're distinctively good at), role families they've actually done, one thing worth probing. End with exactly: "Reply with corrections, or 'looks right' to continue."

**Question 3 of 20 — north star.** "In 2–3 sentences: your ideal next role."

**Question 4 of 20 — dream names.** "2–3 people, companies/ventures, or roles you dream about."

**Question 5 of 20 — dealbreakers.** "What are your dealbreakers — the things that make you say no outright? Think across categories: responsibilities (e.g. a step down in scope), pay, company setup or stage, culture, location."

**Question 6 of 20 — present the drafts.** Present their taste profile, anti-profile, and north-star list (drafted from resume + questions 3–5). End with exactly: "Correct anything, or say 'good' to continue."

**Question 7 of 20 — comp numbers.** "Three numbers: current base salary, total comp target, walk-away floor — roles below the floor are cut outright. Reply with the three numbers, labeled: base / target / floor."
**Question 8 of 20 — logistics.** "Logistics: cities + what 'remote' means for you, max travel, visa or sponsorship constraints. Reply with each, labeled: cities / remote / travel / visa."
**Question 9 of 20 — timeline.** "Timeline: employed, start-by date, how urgent is this?"

**Question 10 of 20 — lock the scoring model.** Present their 4–7 scoring pillars — names, one-line definitions, weights — as the heatmap they'll see every run. (Infer these from resume, stage, and answers; the methodology's reference set is one instance, never the default.) End with exactly: "Adjust anything, or say 'lock it' to continue."

**Then the setup (10 questions):**
- **Question 11 of 20 — where your stuff lives: backend.** "Should the spreadsheet live in your LLM's workspace, or on the Muse server?"
- **Question 12 of 20 — where your stuff lives: front end.** "The web portal is on by default — keep it, switch to an email digest, or none (chat only)?"
- **Question 13 of 20 — where your stuff lives: notification.** "How do you want to be notified — email, text, or a dedicated side thread?"
- **Question 14 of 20 — how you're getting information: job alerts.** "Can I scan your inbox for job alerts (LinkedIn, etc.)? Grant access and I'll ingest them as a feed."
- **Question 15 of 20 — how you're getting information: interview progress.** "Can I scan your inbox for interview and pipeline progress — invites, recruiter follow-ups, offers, rejections? This powers the automatic process tracking."
- **Question 16 of 20 — cadence: frequency.** "How often should the scan run — daily, every 2 days, twice-weekly, or weekly?"
- **Question 17 of 20 — cadence: days.** "Which mornings?"
- **Question 18 of 20 — cadence: time.** "What time?"
- **Question 19 of 20 — dossier prefs.** "Dossier: beyond the standard sections, is there anything you always want covered? (e.g. comp benchmarking, hiring-manager background)."
- **Question 20 of 20 — pipeline prefs.** "Pipeline: the default is score-ranked cards with status filters — want anything different?"

## After the interview — build in this order

1. Write everything to the spreadsheet's Config tab first (profile, mandates, weights, anchors, schedule, resume path).
2. Seed the source registry's four tabs — driven by the targeting logic (TDD §4.1, methodology §7), not literal title keywords: Companies ← trajectory lookalikes' employers + north-star ventures + problem-domain companies; Boards ← keyword queries for the candidate's markets *and* adjacent mandates; Feeds ← email alert sources + ATS APIs; People ← 10–15 exec/social accounts (lookalike profiles, role models, orbit execs).
3. Verify every URL yourself (official careers/ATS pages only — never aggregators, never invented handles).
4. Show the seeded registry — counts per tab plus the notable names — and ask: "Here's where I'll hunt. Add or remove anything?" Also ask: "Do you get job alerts by email (LinkedIn, etc.)? Grant access and I'll ingest them as a feed." → if yes, set Config `EMAIL_ALERTS=true` and add a Feeds row. Apply their changes before continuing.
5. Write the 1–10 anchors for each pillar from the intake answers.
6. Validate the config: describe 2–3 hypothetical roles with real trade-offs — built from real companies already in the registry — and ask which they'd take; if their picks disagree with what the scoring would rank, fix the config first.
7. Portal (default-on unless opted out): instantiate from `portal-template/index.html` — replace the `PORTAL_DATA` object, wire the buttons, delete the demo banner. Create the delivery side thread "🎯 RoleRadar".
8. Schedule per the confirmed cadence. Tell them when the first batch launches and ask: "First batch goes out {time} — want it now instead?" Then run the first scan, filling the Roles sheet.

**First-run delivery** (in the 🎯 thread): the one-page system card (pillars, weights, anchors, source counts), then the TL;DR — dream-vector block, top-3 leaderboard, one line per remaining role, the 80/20 close as a single-tap option — plus the portal link and report doc link. Then tell them what was built and which sources already proved their worth.

## Tips

- The intake is the product. Phase 1's career-stage call shapes everything after it — get it right before proceeding.
- One question per message, count first, question last. The candidate should never wonder what's expected or how much is left.
- A rushed Phase 2 (north star) produces vague gems; push for specifics — names, companies, dream roles/positions, missions, and especially what they're running from.
- Never copy the methodology's example buckets — derive every config value from the intake, in the candidate's own function language.
- Keep methodology, TDD, template, and this protocol versioned together (all v1.25); the wrapper prompt is the only per-candidate artifact.
