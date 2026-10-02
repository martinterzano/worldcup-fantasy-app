# Listita Mundial 2026

> Fantasy football app built for the FIFA World Cup 2026. Three participants draft national team players, score points from live match events, and track a live ranking throughout the tournament. Live at [listita-mundial.martinterzano.com](https://listita-mundial.martinterzano.com).

**Stack:** Node.js · Express · Prisma · PostgreSQL · Next.js 14 · Tailwind CSS · Railway · Vercel · API-Football · **Status:** in production (tournament ran Jun–Jul 2026)

---

## Contents

1. [What it is](#1-what-it-is)
2. [The pipeline: from API event to live ranking](#2-the-pipeline-from-api-event-to-live-ranking)
3. [The coverage map: the hardest design problem](#3-the-coverage-map-the-hardest-design-problem)
4. [Idempotency and data integrity](#4-idempotency-and-data-integrity)
5. [Architecture](#5-architecture)
6. [AI-native development model](#6-ai-native-development-model)
7. [What this project demonstrates](#7-what-this-project-demonstrates)

---

## 1. What it is

The app is a private fantasy football game for three people and a World Cup (48 national teams, 104 matches). Before the tournament, each participant drafts players from the 48 squads. As the tournament runs, every real match event (goal, assist, yellow card, red card, clean sheet, others) adds or removes points for the participant who drafted that player. A public dashboard shows the live ranking, a feed of recent events, and upcoming matches.

The previous version for Qatar 2022 was an Excel. This replaced it.

**Scoring table:**

| Event | Points |
|---|---|
| Goal | +50 |
| Assist | +20 |
| Yellow card | -5 |
| Red card | -100 |
| Own goal | -100 |
| Defender: clean sheet | +10 |
| Goalkeeper: clean sheet | +20 |
| Goalkeeper: goal conceded | -10 |

All values are configurable at runtime from the admin panel via a `PointsConfig` table (no code change needed to adjust the scoring rules mid-tournament).

## 2. The pipeline: from API event to live ranking

**Ingestion.** A cron job runs every 10 minutes on days with matches. It queries the API-Football endpoint for finished fixtures, identifies unprocessed ones, and calls the processing endpoint for each.

**Processing.** For each finished match, the backend fetches match events (goals, assists, cards) and lineups from API-Football, builds a per-player coverage map for every minute of the match (see §3), resolves which participant owns responsibility for each event, and writes one `PointTransaction` per event with the participant ID, player name, event type, match reference, and minute.

**Serving.** The public dashboard reads from the `PointTransaction` table directly. Standings are a sum by participant. The activity feed is the 20 most recent transactions ordered by creation time. Both queries are simple aggregations on the same table, so the dashboard never needs to reprocess events.

The shape of this pipeline (external API source, event-by-event transformation with business logic applied to each event, ledger-style output, derived views) is structurally identical to an operational ETL pipeline. The domain is football; the pattern is data engineering.

## 3. The coverage map: the hardest design problem

The problem: a player can be substituted mid-match. If player A (drafted by participant 1) is substituted by player B (drafted by participant 2) in the 60th minute, goals scored by A before minute 60 go to participant 1, and goals scored by B after minute 60 go to participant 2.

This sounds simple until you add two complications: (a) a substituted player can itself be replaced, creating chains; and (b) a substitute may have no owner (nobody drafted them), in which case the substitute inherits the owner of the player they replaced.

The solution is a **coverage map**: a list of `{apiPlayerId, ownerParticipantId, activeFrom, activeTo}` records built from the match lineups and substitution events before any event is processed. Every event is then resolved against this map by minute. The map handles chains naturally (a substitute that inherits ownership and then gets substituted passes that ownership forward), and covers the no-owner case by tracing ownership through the chain to the closest ancestor who has one.

Building the map correctly before processing events means the event-scoring logic itself is simple: look up the coverage map for the event minute, get the owner, write the transaction. The complexity is front-loaded into one function with a clear contract.

## 4. Idempotency and data integrity

**Idempotency.** The processing endpoint checks whether `MatchEvent` records already exist for the match before doing anything. If they do, it returns immediately without writing new transactions. This means the cron job can safely call the endpoint multiple times (network retries, cron overlaps, manual re-runs) without double-counting points.

**Explicit player name in each transaction.** A substitute who has no owner still generates a `PointTransaction` (for traceability), but that player is not in the draft. Storing the player name directly in the transaction means the activity feed is always human-readable without requiring a join to a player table that may not have the record.

**Configurable scoring as a singleton table.** `PointsConfig` has exactly one row, addressed by a fixed ID. All endpoints that compute points read from this table at query time. Changing the scoring table mid-tournament takes effect immediately on all future event processing without touching any code.

## 5. Architecture

```
API-Football (external)
      ↓  (cron job, every 10 min on match days)
Node.js + Express (Railway)
      ↓  (Prisma ORM)
PostgreSQL (Railway)
      ↑
Next.js 14 frontend (Vercel)
      ↑ public, no login required
Participants and admin (browser)
```

### Data model (core entities)

```
Participant          → the three players
NationalTeam         → 48 teams (LARGE/SMALL classification for draft rules)
Player               → squad members, with apiId for event matching
DraftPick            → which participant owns which player
GoalkeeperPick       → separate mini-draft for goalkeepers
Match                → 104 fixtures, with status (SCHEDULED/LIVE/FINISHED)
MatchEvent           → goal/assist/card/clean-sheet records from API-Football
PointTransaction     → one row per event, per participant affected (the ledger)
PointsConfig         → singleton row for scoring values
```

### Admin functions

The admin panel handles setup (import 48 squads from API-Football), the draft interface (who picks, which slot, search and assign players), match management (sync calendar, trigger processing per match), and scoring config. None of this requires participant accounts. The three-person scope made authentication unnecessary.

## 6. AI-native development model

The technical implementation (Express API, Prisma schema, Next.js frontend, cron job) was built with **Claude Code as a technical co-worker**.

**What I designed:**

- Complete game rules, including all edge cases (substitution chains, clean-sheet eligibility by position, LARGE/SMALL team classification for draft slots)
- The data model: every entity, relationship, enum, and constraint
- The integration design for API-Football: which endpoints to use, at what frequency, how to handle match states
- The processing pipeline design: coverage map logic, idempotency mechanism, event-to-transaction mapping
- The public dashboard architecture: no login, layout decisions, which metrics to surface and how
- All validation against expected game behaviour

**What Claude Code executed:**

- Express API (all endpoints, services, middleware, error handling)
- Prisma schema, migrations, and all processing logic
- Cron job for fixture polling
- Full Next.js frontend (dashboard, admin pages, participant views)

The split is deliberate. The design is where the domain knowledge lives. The execution is where Claude Code adds speed. The result is a production system that a solo practitioner without a dedicated front-end or back-end background could not have shipped in the same timeline, built to the same level of fidelity, with full ownership of every decision that matters.

This working model is **not prompt engineering**. It is architectural design followed by precise delegation to a capable executor, with systematic validation at every step.

## 7. What this project demonstrates

- **Real-time data pipeline design.** The flow from API-Football to live ranking (ingest, transform with business logic, write to ledger, serve derived views) is a reduced but structurally faithful version of an operational ETL pipeline. Idempotency and event traceability are first-class concerns, not afterthoughts.
- **Non-trivial algorithmic design.** The coverage map for substitution handling is a genuine design problem: ordered events, inherited ownership, chain resolution. Designed completely before implementation started.
- **External production API integration.** Rate limit management, conditional polling, header-based auth, match state handling. Directly transferable to data integrations in professional contexts.
- **Configurable business rules.** Scoring values live in the database, not in code. Changing them at runtime takes effect immediately. The pattern (rules as data) is one of the most durable patterns in production data systems.
- **AI-native delivery.** Solo design and product ownership on a full-stack system spanning backend, frontend, database, and third-party API. This is the working model for what a senior DS/AI practitioner can produce in 2026 with access to the right tools.

---

Martín Terzano · [helliumlab.com](https://helliumlab.com) · [LinkedIn](https://www.linkedin.com/in/martinterzano) · martin@helliumlab.com
