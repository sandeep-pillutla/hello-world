# Road Trip Agent — Project Plan

_Draft v0.1 · 2026-10-28_

## 1. Summary

The road trip agent runs a cross-country Tesla trip on **monday.com**. Four linked boards (Itinerary, Reservations, Tour Guide, Travellers & Preferences) hold the plan. Built-in Timeline, Map and dashboard views show route, budget, battery plan and pending approvals. A separate **agent service** built on Claude reads and writes the boards through monday's API. It reacts to webhooks and scheduled jobs, and checks every change with a **feasibility checker** (drive time, battery reserve, opening hours, time zones). With the owner's permission, **Tesla's Fleet API** supplies live battery level and location and lets the agent send each day's route to the car. With FSD (Supervised), the car follows that route while the driver stays responsible. A **car dashboard page** (Tesla browser, viewed while parked) and a **companion phone app** with a hands-free **tour guide** read from the same boards. Travellers choose a **trip pace** (Chill ≈ 2 weeks, Standard ≈ 1 week, Ludicrous ≈ 3–4 days with two drivers) for the whole trip or per stretch. Bookings only happen after a traveller marks them "Approved".

## 2. Goals for version 1

- Plan a workable coast-to-coast Tesla trip at any of the three paces, shown on monday.com.
- Re-plan the next 1–2 days when something changes, with a person approving the result.
- Send each day's route to the car and track battery use against the plan.
- A hands-free tour guide that works offline.

**Not in version 1:** booking directly through hotel APIs (link out instead), vehicles other than Teslas, routes outside the US, controlling the car beyond sending a navigation route, and multiple customers. Version 1 is for one household's own trips.

## 3. Assumptions (to confirm)

- One developer, part-time. Estimates below are rough developer-weeks.
- The first real use is a personal trip, not a public product.
- The same language throughout: **TypeScript** for the agent service, dashboard and phone app.
- A paid monday.com plan that includes API access and enough automation actions per month.

## 4. Proposed technology

| Part | Choice | Notes |
|---|---|---|
| Agent | Claude via the Agent SDK or API, with tool use | A stronger model for planning and a cheaper, faster one for routine checks |
| Service | Node/TypeScript, a small HTTP server | Receives webhooks, runs scheduled jobs and phone-app requests |
| Storage | Postgres (SQLite to start) | Live GPS and battery readings, caches, tour content. monday holds the plan only |
| monday | GraphQL API (`api.monday.com/v2`) or monday's MCP server | Batch writes to stay within rate limits |
| Routing | Google Routes, Mapbox or HERE (**decision needed**) | Check each provider's caching and display terms |
| Weather | National Weather Service API, plus historical averages for far-off dates | Free; includes official alerts |
| Charging | Tesla Fleet API (nearby Superchargers) plus NREL's charger database | |
| Places and parks | NPS API, Recreation.gov, Wikipedia/Wikidata, HMdb.org | Also the tour guide's sources |
| Car dashboard | A small web app (for example Next.js) | Large text, Today view, a few buttons |
| Phone app | React Native / Expo | Background location, text-to-speech, offline pack |

## 5. Milestones

### M0 — Foundations (~1 week)
- [ ] monday.com account on a plan with API access; API token stored as a secret
- [ ] Anthropic API key
- [ ] **Tesla developer account and Fleet API app registration.** This needs a domain and a hosted public key, and approval can take time, so **start now**
- [ ] Choose a routing provider and get an API key
- [ ] Repo layout, linting, CI, secrets handling

### M1 — monday board template (~3–5 days)
- [ ] Create the four boards, columns, groups, Connect Boards links and dependencies (see §6)
- [ ] Timeline, Map and Calendar views and the trip dashboard
- [ ] Automations: cancel-by reminders, Blocked alerts, Approved → webhook, move the current day to the "Today" group
- [ ] Onboarding form (WorkForm) that fills Travellers & Preferences
- [ ] A script that rebuilds the template through the API, so it can be repeated
- **Done when:** a short test trip can be planned by hand on the board

### M2 — Planner and feasibility checker (~3–4 weeks)
- [ ] Trip data model: Trip, Day, Leg, Stop, ChargeStop, Reservation, Pace profile
- [ ] Routing tool and leg builder
- [ ] **Range model:** baseline energy per mile, adjusted for speed, temperature, elevation and wind. Calibrate it later with real trip data
- [ ] Charging planner (Superchargers, hotels with charging, gap detection)
- [ ] Pace profiles (Chill, Standard, Ludicrous) and side-by-side comparison
- [ ] Feasibility checker that sets the Feasibility column
- [ ] Historical weather averages; a hand-maintained list of seasonal road closures; park entry reservations
- [ ] Agent loop: read the form, build the plan, write to monday in batches, post explanations as item updates
- [ ] **Test set:** 20–30 scenario trips, scored automatically
- **Done when:** LA→NYC produces a workable plan at all three paces, and all test scenarios pass the checker

### M3 — Webhooks, approvals and re-planning (~2 weeks)
- [ ] Receive monday webhooks: Approval changed, Pace changed, item added or edited
- [ ] Proposal flow: the agent writes *Proposed* items; travellers approve them
- [ ] Re-planning of the next 1–2 days; the travellers' own edits always win
- [ ] Morning briefing posted as a monday update, which notifies travellers' phones
- **Done when:** changing a day's pace on the board produces a correct proposal for that day and the ones after it

### 🚗 Pilot 1 — a weekend trip (300–500 mi)
Use M1–M3 on a real short trip. Record actual battery use and every problem found.

### M4 — Tesla connection (~2 weeks)
- [ ] Fleet API owner sign-in (OAuth) and permissions
- [ ] Read battery level, location and charging state
- [ ] Send each day's route and charging stops to the car's navigation
- [ ] Monitor battery use against the plan and suggest re-plans when it runs high
- [ ] Track Fleet API usage costs

### M5 — Car dashboard page (~1 week)
- [ ] Today view in the Tesla browser, with large text
- [ ] Buttons: Approve, Add stop, Running late, Change pace
- [ ] Simple sign-in suited to a car screen

### M6 — Phone app and tour guide (~4–6 weeks)
- [ ] Content pipeline: gather sources along the route, write scripts from sources only, check them against the sources
- [ ] Nightly job that prepares the next day's offline pack (scripts, audio, map tiles, plan)
- [ ] Ranking, a limit on how often it speaks, and pace-based settings
- [ ] Location triggers timed by speed and distance; holds during complex driving
- [ ] Voice commands: tell me more, skip, quiet, add that stop
- [ ] Trip journal on the Tour Guide board

### 🚗 Pilot 2 — a long trip
A trip of a week or more, with everything enabled.

### Later
- Direct booking through partner APIs (a business agreement, not just engineering)
- Integrations with state 511 road-condition systems for each state on the route
- Running themes, kids mode, passenger mode
- Multiple customers and a monday marketplace app

## 6. monday board design (reference)

**Itinerary** (one group per day; items are Drive, Charge, Sight, Meal, Lodging or Event)
Type (Status) · When (Timeline) · Local time zone · Location · Drive time · Miles · Battery arrive % · Battery depart % · Charge minutes · Pace (Status) · Feasibility (Status) · Approval (Status: Proposed, Approved, Booked, Cancelled) · Cost · Cancel by (Date) · Source / checked at · Depends on (Dependency) · Reservation (Connect Boards)

**Reservations**: confirmation number, cancellation policy, cost, files, linked itinerary item.
**Tour Guide**: location, trigger distance, tier, theme, script, sources, played or skipped.
**Travellers & Preferences**: drivers, interests, budget, vehicle, FSD (yes/no), default pace.

## 7. Safety rules (apply to every milestone)

- Nothing involving money happens without *Approved*. Prefer refundable rates.
- Plan at legal speeds. Enforce daily driving-hour limits. Ludicrous requires 2+ drivers. FSD never relaxes these limits.
- Every leg keeps a safe battery reserve (default 15–20%).
- No tour announcements during complex driving; everything is hands-free.
- Every fact has a source and a time. Tour scripts are written only from sources.

## 8. Risks

| Risk | Impact | What to do about it |
|---|---|---|
| Tesla Fleet API approval, costs or limits | Delays M4 | Register in M0; everything works without the car connection |
| monday API rate limits or plan limits | Slow updates, failed automations | Batch writes; keep live data off the board; confirm the plan early |
| Tour guide states wrong facts | Lost trust | Sources-only scripts, checked against their sources, cited on the board |
| Range estimates are wrong | Running low on charge | Conservative reserve; calibrate from pilot data; re-plan from live battery data |
| Road-condition data differs by state | Missed closures | Start with National Weather Service alerts and seasonal closures; add states as needed |
| Booking API access | No direct booking | Link out to book in version 1 |
| Driver distraction or liability | Safety | Hands-free only, no announcements during complex driving, clear FSD wording |

## 9. Open decisions

1. Who is building this, and by when? Is there a target trip date?
2. Personal tool or product? This affects the Tesla app registration, multiple customers and the monday marketplace.
3. Routing provider: Google, Mapbox or HERE.
4. Hosting for the agent service and dashboard.
5. Text-to-speech provider for the tour guide.
6. monday.com plan tier.

## 10. Next steps (this week)

1. Answer the open decisions above, especially the target date and personal vs. product.
2. Start the Tesla Fleet API registration (it takes the longest).
3. Set up the monday account and API token; build the M1 template by hand.
4. Set up the repo layout and start the trip data model and feasibility checker (M2).
