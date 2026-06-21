# WC26 · Betting Desk

A single-user, self-hosted betting dashboard for the 2026 FIFA World Cup.
Tracks straight bets and parlays, bankroll, P&L/ROI/CLV, and shops lines across
books with no-vig fair odds and EV%.

**Status:** Phases 1–4 complete (scaffold, schema, bet tracking, dashboard,
**live The Odds API integration with caching**, and the value layer — no-vig
fair odds, EV%, hold, CLV, fractional Kelly). The odds board uses a mock fixture
until `ODDS_API_KEY` is set, then switches to live data automatically.

## Stack

Next.js 16 (App Router) · TypeScript · Tailwind 4 · shadcn/ui · Supabase
(Postgres + Auth + RLS) · Recharts · Vitest.

## Local setup

> Node is installed at `~/.local/node`. If `node` isn't on your PATH, run:
> `export PATH="$HOME/.local/node/bin:$PATH"`

1. **Create a Supabase project** (free tier is fine) at supabase.com.

2. **Run the migration.** In the Supabase dashboard → SQL Editor, paste and run
   [`supabase/migrations/0001_init.sql`](supabase/migrations/0001_init.sql).
   This creates the tables, indexes, RLS policies, and the settings trigger.

3. **Configure env.** Copy `.env.example` → `.env.local` and fill in from
   Supabase → Project Settings → API:
   ```
   NEXT_PUBLIC_SUPABASE_URL=...
   NEXT_PUBLIC_SUPABASE_ANON_KEY=...
   SUPABASE_SERVICE_ROLE_KEY=...        # reserved for snapshot persistence
   ```
   **Odds (optional):** set `ODDS_API_KEY` (from the-odds-api.com) to go live.
   Also tunable: `ODDS_API_SPORT_KEY` (default `soccer_fifa_world_cup`),
   `ODDS_API_REGIONS` (default `us`), `ODDS_CACHE_TTL_SECONDS` (default `90`).
   Leave the key empty to keep using the mock fixture.

4. **Run it.**
   ```
   npm run dev
   ```
   Open http://localhost:3000 → you'll be redirected to `/login`. Click
   **Create account (first run)** once, then sign in. (If Supabase email
   confirmation is on, confirm via the emailed link first, or disable it under
   Auth → Providers → Email for a frictionless single-user setup.)

5. **Add your opening deposit** under Settings → Bankroll. Bankroll starts at $0.

## Scripts

| Command | What |
|---|---|
| `npm run dev` | Dev server |
| `npm run build` | Production build |
| `npm test` | Unit tests (odds math, settlement, metrics) |
| `npm run lint` | ESLint |

## Defaults (per your setup)

- Bankroll starts **empty** — add deposits in Settings.
- Default book: **FanDuel** (editable in Settings).
- Odds display: **American** (decimal stored internally; toggle in Settings).
- Stakes tracked in **dollars + units** (unit size set in Settings, default $100).

## Architecture notes

- **All odds math is pure and unit-tested** in `src/lib/` (`odds.ts`,
  `settlement.ts`, `metrics.ts`, `board-value.ts`). Decimal odds internally;
  American is display-only.
- **Settlement rule:** a parlay wins only if all non-void/push legs win;
  void/push legs drop out and combined odds recompute over the rest.
- **Odds provider is abstracted** behind `OddsProvider`
  (`src/lib/odds-provider/`). `getOddsProvider()` returns `TheOddsApiProvider`
  when `ODDS_API_KEY` is set, else the mock. The live adapter caches responses
  in-memory for `ODDS_CACHE_TTL_SECONDS`, tracks the `x-requests-remaining`
  quota header, and falls back to stale cache / mock on error. `listSports()`
  helps confirm the active sport key.
- **Security:** RLS on every user table scoped to `auth.uid()`; the odds API key
  lives in server env only — all fetches happen in server components/actions and
  it is never sent to the client.

## What's next (Phase 5 polish)

- Persist `odds_snapshots` to Supabase (service role) for historical line charts.
- Live grading of open bets from the board's current prices.
- More chart ranges, saved filters, deeper mobile pass, loading skeletons.
- Verify the active World Cup sport key via the sports list before each session.
