# Portal Template — README (for the builder LLM)

This directory holds the **reference implementation** of the RoleRadar portal
(TDD §6.4) plus the **notification-shape prototype** (TDD §6.5). Do not rebuild
either from the spec — instantiate them.

## Definition of done (before the portal goes live)

- Every demo toast is replaced with a real action wired per TDD §6.4 — no dead buttons.
- **Escape all interpolated data.** Role, company, and candidate strings go through an escaper before touching `innerHTML` — never interpolate raw web-derived text.
- **URL allowlist.** Job and company URLs must be `https://` on the employer's own domain (or a verified redirect the worker resolved). No `javascript:`, no opaque shorteners, no untrusted aggregators.
- Buttons wired: 👍/👎 → UserState read-modify-write by `role_id` (never touch the 37 Role columns); *Tailor resume* → versioned file linked from the card; *+ Add job URL* → fetch → gate → score → add with `source=manual`.
- `status` values use the canonical enum (TDD §3.2) — the pipeline renders `status==="open"`.
- Verified with a read-back (row counts + spot-checked fields) before declaring done. The demo banner is deleted.

## Instantiate in 3 steps

1. **Open `index.html` and replace the entire `PORTAL_DATA` object** (clearly
   marked in the `<script>` section) with the candidate's real data:
   - `candidate`: name, title, location, last_scan
   - `pillars`: 4–7 entries `{key, name, weight}` — weights sum to 100, same
     order as the Roles sheet's `p1`–`p7` columns
   - `roles`: pipeline rows with `id, company, title, score, p[]` (pillar
     scores 1–10 in pillar order), `comp, location, level, ic_mgr, one_liner,
     url, status, user_status, bookmarked, thumb, last_verified, is_gem`
   - `sources`: `companies[]`, `people[]`, `feeds[]`, `boards[]` with the
     fields shown in the demo data (name/niche/url/verified/checked/gems/last_hit)
2. **Wire the action buttons to real actions** (TDD §6.4):
   - 👍 / 👎 → write `thumb`, `bookmarked`, `user_status` to the UserState tab
     (read-modify-write by `role_id`; never touch the 37 role-data columns)
   - *Tailor resume* → rewrite base resume vs the JD; versioned file; link in
     `tailored_resume_link`
   - *Pipeline tab:* wire the "+ Add job URL" button — paste URL → worker fetches
     the JD, runs gates + scoring, adds the row with `source=manual`
     (duplicates merge by `role_id`; gated-out URLs report their cut reason).
   - *Role interest flow* → the modal covers the dossier step (propose outline +
     "what else?" + Drive-storage checkbox → Go). Wire Go to the deep-research
     brief; save the versioned file in the confirmed location; link in
     `dossier_link`. The other two interest questions (track the process?
     proactive-help menu?) are asked in the thread — see TDD §6.4. Persist
     answers in the UserState columns `dossier_wanted, drive_ok,
     track_process, track_surface, proactive`; render them via `interestLine(r)`
     on bookmarked cards (demo data on the first two roles shows the shape).
   - *Status toggle* → cycle `user_status`; feeds the outcome-learning loop
   - *Interview prep* → gap brief through the pillar lens
   - Sources add/remove → write straight to the registry tabs
     (`active=FALSE` soft delete); the next scan reads the sheet
3. **Delete** the demo banner, the DEMO toasts/notes, and this README's
   demo data. Deploy per the candidate's hosting (static file or hosted page).

## `notifications.html` — notification shapes

A visual prototype of the TDD §6.5 message shapes (run TL;DR, interview/offer
detected, deadline nudge, dossier ready, quiet week, rejection logged) as chat
bubbles in the dedicated thread. Fictional demo content. The §6.5 text shapes
are normative; this file shows what they look like.

## Design contract (keep)

- Dark, editorial aesthetic (Bodoni Moda serif display + Hanken Grotesk body —
  degrades gracefully to system serif/sans offline). Mobile-first, scannable;
  score-first cards; pillar heatmap mini-strip on every role card; thumbs
  top-right; quiet weeks and empty states never render as placeholders.
- The two planes stay separate: the scan owns role data, the candidate owns
  user state — the portal never edits scan-owned fields.
