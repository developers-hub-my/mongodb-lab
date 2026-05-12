# Training Checklist — MongoDB Database and Security

A working checklist for delivering the one-day MongoDB course end-to-end.
Tick items as you complete them. The list is split into trainer prep,
day-of delivery, and post-training follow-up.

## Pre-Training (Week Before)

### Content readiness

- [ ] Walk through the full slide deck once, end-to-end
- [ ] Dry-run all three exercises against a fresh `docker compose up`
- [ ] Confirm `sample-data/` files import cleanly into the `library` database
- [ ] Verify mongo-express loads at `http://localhost:8081`
- [ ] Confirm `mongodump` / `mongorestore` work for Exercise 3
- [ ] Re-read the [security checklist](docs/03-reference/02-security-checklist.md)
      so live answers are accurate

### Participant logistics

- [ ] Confirm participant count (max 40) and produce a name list
- [ ] Send participants the prerequisites email at least 5 days before
- [ ] Ask them to confirm Docker Desktop is installed and `docker --version`
      runs successfully
- [ ] Share the lab kit link or zip ahead of time

### Materials

- [ ] Print the [Security Checklist](docs/03-reference/02-security-checklist.md)
      as a handout (one per participant)
- [ ] Print the [mongosh Cheatsheet](docs/03-reference/01-mongosh-cheatsheet.md)
      as a handout (one per participant)
- [ ] Prepare red sticky notes (one pad per table) as the "need help" signal
- [ ] Prepare a visible timer (phone, wall clock, or projector overlay)
- [ ] Confirm post-training assessment form is ready

### Venue and tooling

- [ ] Confirm Wi-Fi capacity for 40 simultaneous Docker pulls (or pre-stage
      images on USB)
- [ ] Test the projector with your laptop the day before
- [ ] Confirm power outlets / extension cords are sufficient
- [ ] Identify restroom and prayer room locations for the welcome briefing

## Day 0 (Evening Before)

- [ ] Run `docker compose down -v && docker compose up -d` on the demo
      laptop to verify a clean-room start
- [ ] Top up battery / pack charger
- [ ] Stage the lab kit zip on a USB drive as a Wi-Fi-failure backup
- [ ] Review the speaker notes for Slides 13, 23, 26, 42 — the four
      hands-on moments

## Day Of — Morning Block

- [ ] 09:00 — Welcome, housekeeping, ground rules
- [ ] 09:15 — Module 1: NoSQL fundamentals (75 min)
- [ ] 09:55 — **Lab Setup Check** ([guide](docs/01-getting-started/02-lab-setup.md))
      — every participant has a working `mongosh` prompt
- [ ] 10:30 — Morning break (30 min)
- [ ] 11:00 — Module 2: CRUD and querying (120 min)
- [ ] 11:30 — **Exercise 1: CRUD** ([guide](docs/02-exercises/01-crud.md)) — 20 min
- [ ] 12:00 — Indexing teach
- [ ] 12:35 — **Exercise 2: Querying & Indexing** ([guide](docs/02-exercises/02-querying-indexing.md))
      — 25 min
- [ ] 13:00 — Module 2 recap, then lunch

## Day Of — Afternoon Block

- [ ] 14:15 — Module 3: Database security (135 min)
- [ ] 15:30 — **Exercise 3: Security Hardening** ([guide](docs/02-exercises/03-security-hardening.md))
      — 30 min
- [ ] 16:00 — Module 3 recap and security checklist walk-through
- [ ] 16:30 — Reflection prompts (5 min silent + share)
- [ ] 16:45 — Where to go from here (resources)
- [ ] 17:00 — Close, Q&A, hand out assessment

## Per-Exercise Sanity Checks

Before announcing each exercise, confirm:

- [ ] Timer is visible to participants
- [ ] You have walked the room at least once during the previous module
- [ ] You know which participants are ahead and which are behind
- [ ] The relevant lab guide is already open on the projector

## Post-Training (Within 48 Hours)

- [ ] Collect and review feedback forms
- [ ] Send participants a thank-you email with:
  - [ ] Link to the lab kit so they can keep practising
  - [ ] Resources from the slide deck (M001 free course, community forum)
  - [ ] Any questions you committed to follow up on
- [ ] File the assessment results
- [ ] Note any slide / exercise issues in a `lessons-learned.md` (or commit
      directly to the kit's `docs/`)
- [ ] Update [`docs/`](docs/README.md) with anything that surprised you or
      that participants got stuck on

## Lab Kit Health (Quarterly)

- [ ] Run `docker compose pull` to refresh images
- [ ] Re-run all three exercises end-to-end
- [ ] Check that `mongo:7` is still the right pin (or bump to current LTS)
- [ ] Re-validate the security checklist against current MongoDB guidance
