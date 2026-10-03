# RoleRadar — never miss the role you were meant for

A self-improving AI job scan, built with Muse. It hunts roles across company boards,
keyword searches, your inbox, and the people who post the jobs worth having.
It scores every match against what *you* care about. Then it gets better at it every week.

You make one call per role: in or out.

## Start here (25 minutes)

1. Download this repo as a zip: **Code → Download ZIP**.
2. Open a new Muse side chat and attach the zip.
3. Copy the prompt below, paste it in, send.
4. Answer 20 questions, one per turn. Tomorrow morning, your first briefing arrives.

### The prompt — copy from here

```
Attached is the RoleRadar kit (this repo, downloaded as a zip). Unzip it, then:

1. Read methodology.md (the algorithm), TDD.md (the build spec), and onboarding-protocol.md (the interview script).
2. Interview me following the onboarding protocol exactly — 20 questions, one per turn.
3. Build the system per the TDD: seed the spreadsheet, instantiate the portal template, schedule the scans.

Start with the protocol's intro message.
```

## What's in the repo

- `methodology.md` — the algorithm: gates, scoring, discovery
- `TDD.md` — the engineering spec for the builder
- `onboarding-protocol.md` — the 20-question interview script
- `job-scan-spreadsheet-template.xlsx` — 10-tab workbook: config, registry, pipeline, state
- `portal-template/` — the dashboard template: instantiate, don't rebuild
- `GETTING_STARTED.md` — setup walkthrough
- `README.html` — the full selling page (screenshots included)

## License

MIT — do what you want with it.
