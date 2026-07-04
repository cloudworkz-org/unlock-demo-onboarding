# Unlock Demo Onboarding

Investor-facing DEMO of the Unlock platform: features, personas, and belief logic are pre-scripted so a salesperson can walk an investor through a clean, deterministic onboarding journey. **This is a DEMO snapshot — the primary product code lives in the separate `unlock-platform` repo, not here.** Treat divergence from `unlock-platform` as intentional unless told otherwise.

## Architecture

Read [`docs/architecture/architecture.md`](docs/architecture/architecture.md) at session start before building anything — it's the Claude-Code-readable distillation of the Cloudworkz OS architecture every app repo builds towards. Log promotion-candidate building blocks (and consciously-skipped gaps) in [`BUILDING_BLOCKS_CATALOGUE.md`](BUILDING_BLOCKS_CATALOGUE.md) as you build. See [[Intelligence/decisions/2026-06-15-app-repo-architecture-bootstrap-V1]] for the convention.

## Directory map

| Dir | What's in it |
|---|---|
| `client/` | React + Vite SPA (wouter, TanStack Query, Radix/shadcn, Tailwind) — the demo UI |
| `server/` | Express API + Vite middleware; entry `server/index.ts`, routes `server/routes.ts` |
| `shared/` | Drizzle schema (`schema.ts`), shared models, storage buckets — used by client + server |
| `frontend/` | Separate sub-app with its own `package.json` (check its README before touching) |
| `scripts/` | One-off data importers (property/HPI/postcode) — `tsx` scripts |
| `tests/` | Vitest suite (persona, scenario-stress, safety-lights, onboarding-v2) + `golden/` |
| `docs/` | Brief, components, acceptance criteria, deployment, onboarding manuals |
| `_export/` | Engine snapshots (persona/simulation/safety-lights) for parity reference |
| `attached_assets/` | Scenario library JSON + demo source assets |

## Do-once commands

- Install: `npm install`
- Dev (client + API, one server): `npm run dev` — serves on **port 5000** (override via `PORT` env)
- Test: `npm test` (vitest, config `vitest.config.server.ts`)
- Typecheck: `npm run check` · DB push: `npm run db:push`

## Personas & belief logic

- `PERSONA_AND_BELIEF_LOGIC.md` (repo root, ~47KB) — persona creation/identification (§1), Q→A matching (§2), belief-based outcomes (§3), portfolio/scenario impact (§4), allocation engine (§5), Monte Carlo (§6), action plans (§7).
- `Demo_Platform_User_Manual.md` (repo root, ~97KB) — full demo platform walkthrough/feature manual.
- Read these instead of re-deriving demo behaviour from source.

## Active branch

`feat/v2-scenario-stress-engine` (not the default branch — confirm before assuming `main`).
