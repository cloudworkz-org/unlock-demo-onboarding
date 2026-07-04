# Cloudworkz Architecture Guide for Claude Code

**Audience:** Claude Code (and any AI coding assistant) building Cloudworkz applications, prototypes, or throwaway utilities.
**Purpose:** Tell you the architectural shape every Cloudworkz app is built towards, the discipline that applies regardless of where you're building, and how to structure code so that what you build today is a natural promotion candidate for the shared monorepo later.
**Companion file:** `BUILDING_BLOCKS_CATALOGUE.md` — every repo maintains one at its root.
**Source of truth:** This guide is a distillation of `Cloudworkz_OS_Architecture_v1.md` for working use. If anything here conflicts with that document, the OS architecture document wins. **If you spot a mismatch between this guide and the OS architecture document, treat the OS document as authoritative for the principle and flag the drift in your next response or PR description so this guide can be updated.** Don't try to reconcile the drift yourself — editing this guide is on the "Claude Code does NOT do" list.

---

## What this guide is, and isn't

This guide is **shape-and-discipline**, not **where-to-import-from**.

Most of the Cloudworkz OS substrate described in the architecture document — the shared shell library, the agent shell, the foundational modules (auth, billing, dashboard), the shared UI library, the agent runtime infrastructure — **does not exist yet**. Don't go looking for it. Don't `import { Shell } from "@cloudworkz/app-shell"` because that package doesn't exist. Don't try to register your module with a non-existent module registry.

Instead, this guide tells you:

1. The architectural **shape** every Cloudworkz application is built towards, even before the shared substrate exists.
2. What **does exist today** that you can reach for.
3. Universal **disciplines** that apply to every line of code, regardless of substrate.
4. How to **shape your code** so it has the form of a building block — even when you're writing everything in-place — so it can be lifted into shared infrastructure later.
5. How to **catalogue what you built** so promotion conversations are concrete instead of archaeological.
6. **Hard nos**: things you must never do.
7. **Decide-and-log defaults**, with the short list of decisions that actually warrant escalation.

**Bias to decide, not to ask.** Most decisions encountered during a build are yours to make and document. Pausing for every architectural question turns a fast prototype into a slow committee. The catalogue is your audit trail — decide, build, log it. The full escalation criteria are at the bottom of this guide; everything not on that short list, you decide.

---

## The architecture you're building against — one screen

Cloudworkz OS is the foundation every Cloudworkz application is built on. The shape is:

- **Applications compose, they don't fork.** A new application is a curated set of *modules* sitting on top of a shared *shell*, with its own *brand config* and its own deployment. Same shell library every time; what changes is the module set and the brand.
- **Everything is multi-tenant from day one.** Org → brand → subscriber. `tenant_id` on every business data row. RLS at the DB layer is the security boundary, not app code.
- **Every action is attributed.** Every mutation captures an `actor_id`. The `audit_log` table is populated automatically by DB triggers. No anonymous writes anywhere.
- **All writes go through bounded functions (RPCs), never direct INSERT/UPDATE.** The function verifies the actor, sets actor context, validates the write, applies it atomically.
- **Agents are first-class actors.** Every agent has a prompt, scope, cost budget, governing accountable, eval suite, and produces a transcript per run.
- **Canonical data uses the two-substrate pattern.** Working substrate (folder of files) for authoring; canonical substrate (DB) for signed-off content; propose-then-apply ingestion between them; supersede-not-edit lifecycle.
- **Seven component types, no more.** Application, Library, Capability Module, Agent Module, Service, MCP Server, External Integration. Everything you build is one of these.

The full document gives the reasoning and the depth. This guide gives you what you need to write code that fits.

---

## What exists today

Be precise about this. Don't import from things that don't exist, and don't assume the existence of services your runtime can't reach.

### Available — what you can actually rely on

- **Tech stack defaults** (per `Cloudworkz_OS_Architecture_v1`, Section 9). TypeScript 5.x strict, Python 3.11+ with uv, Node 20+, Next.js 15 App Router + React 19, Supabase (Postgres + Auth + at-rest encryption + RLS), Drizzle ORM, TanStack Router + Query, Tailwind + shadcn/ui, React Hook Form + Zod, pnpm workspaces + Turborepo, ESLint + Prettier + husky + lint-staged, GitHub Actions, Sentry, Doppler/GCP Secret Manager, Inngest, Postmark, Anthropic Claude. Use these unless you have a specific reason not to.
- **Cloudworkz monorepo** (`cloudworkz/`) as a reference implementation. Even if your repo is separate, you can copy from it:
  - Root-level TypeScript config, ESLint config, Prettier config, husky/lint-staged hooks.
  - GitHub Actions CI workflows (lint, typecheck, test, build).
  - Drizzle migration patterns for the actor/audit/RLS schema (see below).
- **The actor + audit_log + RLS + write_RPC pattern as worked SQL.** The cloudworkz monorepo's migrations implement this pattern end-to-end on its Supabase project; the SQL is the canonical reference. Copy and adapt for your app's own Supabase project — every Cloudworkz app provisions its own DB, applies the same pattern, gets actor attribution + audit + tenant isolation from day one.
- **This document and the source it derives from** (`Cloudworkz_OS_Architecture_v1`). These are the architectural source of truth; they're reachable to anyone with Drive access.

### Planned, not yet built — do not import from these

- `@cloudworkz/app-shell` (the shell library)
- `@cloudworkz/agent-shell` (the agent runtime library)
- `@cloudworkz/ui` (shared component library)
- `@cloudworkz/adapters-core` (shared adapter framework)
- `module-auth`, `module-billing`, `module-dashboard` (foundational modules)
- Application-specific modules (`module-asset-register`, `module-tax-iht`, etc.)
- HTTP transport for MCP servers (current servers are stdio-only)
- Module registry, brand-config plumbing, default-vs-subscribable activation infrastructure
- Cost ledger, transcript provider, dispatch contract, four-level budget enforcement
- Encryption infrastructure beyond Supabase-default at-rest
- Most MCP servers other than `cloudworkz-os` (no Slack MCP, no HMRC MCP, no Companies House MCP)

When you need something from this list, **don't fake an import** and **don't hallucinate a stub library**. Build the equivalent **inline** in this repo, structured so it can be lifted later (see "Shape-aware patterns" below), and add an entry to `BUILDING_BLOCKS_CATALOGUE.md`. This is the normal case — you don't need approval to build a brand-config loader, an actor helper, an LLM wrapper, etc. Just build it in the right shape and catalogue it.

---

## Universal disciplines — every line of code, no exceptions

These apply whether you're inside the cloudworkz monorepo, in Tom's demo repo, in Roy's standalone business utility, or anywhere else. Substrate or no substrate.

### Identity, audit, and writes

- **Every business data row has an `actor_id` column** capturing who created it. Pass the actor through whatever write path you have, even if today that's an inline function. Anonymous writes are forbidden.
- **Every business data table has a `tenant_id` column**, even if the app is single-tenant today. Two extra columns now save a migration of millions of rows later.
- **RLS is enabled on every Supabase table.** Even single-user apps. Defence in depth, and means the table is multi-tenant-ready when the app graduates.
- **Writes go through bounded functions, not scattered SQL.** Even if you don't have a real RPC layer, structure your writes so there's exactly one function per table that performs writes. Direct `INSERT`/`UPDATE` in route handlers is forbidden.
- **Reads can be direct** (via Supabase client or ORM) provided RLS is on. Reads don't need to go through bounded functions unless you have a reason.

### Logging and observability

- **Structured logs (JSON), PII-redacted at write time.** Real names, emails, NI numbers, addresses, financial figures — never in logs. Tokenise or omit at the point of writing the log line, not in a downstream scrub.
- **Errors caught and re-raised with context.** A swallowed error is worse than a crash because the next maintainer has no idea what went wrong.
- **No `console.log` left in committed code.** Use the project's structured logger, even if it's just a thin wrapper today.

### AI and LLM calls

- **Fail-closed on every LLM call site.** If the LLM call fails, returns malformed output, or violates output constraints, raise — never silently substitute a mock value. The user must see "unavailable" before they see fake content.
- **Compliance-bounded outputs.** If your app generates content that might be customer-facing, run it through a code-level validation step before serving (not a human-review gate). The validation pattern: validate the output's *shape* against a schema (Zod) AND validate its *content* against compliance rules (no advice-terms in regulated contexts, required disclaimers present where applicable). If validation fails, fail closed. Write the validator yourself if one doesn't exist — it's library-shaped code and a strong catalogue candidate.
- **No fabricated third-party attribution.** Never invent quotes, ratings, statistics, news items, or analyst opinions under real brand/masthead/bank/institution names. If you don't have a real source, don't generate one. (This is the lesson from unlock-demo-onboarding PR #2.)
- **Honest interim framing.** When your inputs aren't canonical (placeholder data, work-in-progress models, demo values), label the output as illustrative. Don't present model-based output as if it's authoritative.
- **Map by meaning, not by code.** When two systems share a code namespace (S001, S002, etc.) but the codes mean different things in each system, never code-match blindly. Map by meaning. (The library S002 ≠ repo S002 lesson.)

### Secrets and configuration

- **Secrets never in committed `.env` files.** Use Doppler, GCP Secret Manager, or the platform's equivalent — Roy and Tom can get free tiers of both for prototype-scale use.
- **Connection strings, API keys, tokens — not in code, not in commits, not in CI environment variables visible in logs.** Always retrieved from the secret manager at runtime.
- **Brand config in `brand.config.ts`** at the app root, even if no shell consumes it yet. The shape the OS architecture defines today is the shape your app uses now.

### Code quality

- **TypeScript strict mode on.** No `any` without a justified comment. No silent type assertions.
- **ESLint + Prettier + husky + lint-staged.** Configure on day one. The cloudworkz repo has the canonical configs — copy them.
- **No host-specific code in application logic.** The app shouldn't know whether it's on Vercel, Cloud Run, or your laptop. Keeps a future re-host as a containerisation exercise, not a rewrite.

---

## Shape-aware patterns — building TO the architecture

When you build something, recognise which of the seven types it is shaped like and structure it accordingly. Even when the equivalent shared infrastructure doesn't exist yet, you build *as if it did* — that way, lifting your code into the shared infrastructure later is a refactor, not a rewrite.

### Application shape

A deployable end-user product. Has its own `package.json`, its own deployment, its own `brand.config.ts`, its own curated module set (even if "curated" today means three folders).

Structure:

```
my-app/
├── brand.config.ts            # Brand tokens (colours, logo, font, voice)
├── module.config.ts           # The set of modules this app ships with
├── app.config.ts              # Tenancy posture: customer_org_model, multi_brand_per_customer, brand_switching_ui
├── src/
│   ├── shell/                 # Shell-shaped code (routing, layout, brand provider)
│   ├── modules/               # One folder per module
│   ├── shared/                # In-app shared types and helpers
│   └── adapters/              # External-system adapters
├── BUILDING_BLOCKS_CATALOGUE.md
└── CLAUDE.md                  # Repo-level Claude context
```

The `brand.config.ts` and `module.config.ts` files have no consumer today other than the app itself. That's fine — the *shape* is what matters. When the shared shell lands, it reads these same files; your app graduates by changing imports, not by restructuring.

### Module shape

A self-contained feature with a manifest. Folder structure, even if no module registry exists:

```
src/modules/billing/
├── module.config.ts           # Module manifest: id, name, routes, navigation, settings, dependencies, billing config, permissions, hooks
├── pages/                     # The module's UI pages
├── components/                # The module's components (use shared/ui when available, locally-shared otherwise)
├── data/                      # The module's data access layer (bounded write functions)
├── settings/                  # The module's settings panel
├── api/                       # The module's server routes (Next.js route handlers)
└── README.md                  # Short: what this module does, its public surface
```

The `module.config.ts` declares everything: routes, navigation entries, settings, billing config (`free` / `subscription` / `metered` / `subscription_plus_metered`), activation mode (`default` / `subscribable` / `early-access`) with an optional `early_access_cohort` field naming the cohort gated for preview before general release, permissions required, lifecycle hooks. Even with no module loader, **declare these fields** — the app's own routing reads them, or the manifest is informational for now and a real loader will consume it later.

**No reaching into other modules' internals.** If module A needs something from module B, either (a) module B exposes a public surface that module A imports, or (b) communication happens via events (a simple typed event bus, even if today it's just a Node EventEmitter).

### Agent shape

Any code wrapping an LLM call is agent-shaped. Even one inline LLM call deserves the discipline:

```
src/agents/meeting-synth/
├── agent.config.ts            # Identity, scope, cost budget, governing accountable, eval suite path
├── prompts/                   # Prompt templates, versioned
├── tools/                     # Tool definitions if the agent uses tools
├── eval/                      # Gold-standard input/output pairs for regression testing
├── transcript/                # Where this agent's run transcripts get written
└── run.ts                     # The entry point: receives input, produces output, records transcript
```

**Every agent run produces a transcript.** Input received, prompts used, tool calls made, responses, final output, cost. Even if there's no central transcript store yet, write the transcript to a local file or table — the agent's audit trail.

**Cost-tracking is structured even without a central ledger.** Capture per-call: agent_id, model, input_tokens, output_tokens, estimated_cost, timestamp. When the cost ledger lands, your data lifts in cleanly.

**No agent calls the LLM API directly.** Wrap the API call in a helper that takes the prompt + scope + cost-check + transcript-write as a unit. That helper is your local agent-shell.

### Adapter shape

Any code talking to an external system (REST API, SDK, scraped session, webhook receiver) is adapter-shaped. The lifecycle contract from the architecture document applies:

```typescript
// src/adapters/stripe-billing/
export interface Adapter {
  init(): Promise<void>;            // Set up connection state
  auth(): Promise<void>;            // Authenticate / refresh tokens
  observe(): Promise<HealthState>;  // Health check, ready-state
  teardown(): Promise<void>;        // Clean shutdown
}

// Plus adapter-specific methods (the actual work):
export interface StripeAdapter extends Adapter {
  createCustomer(...): Promise<Customer>;
  subscribeCustomer(...): Promise<Subscription>;
  // etc.
}

// Plus a capability descriptor:
export const stripeCapabilities = {
  canCreateCustomers: true,
  canSubscribeCustomers: true,
  canHandleWebhooks: true,
  // ...
} as const;
```

Don't force the adapter into a uniform `read/write/subscribe` shape. Stripe is REST + webhooks. Anthropic is streamed responses with cost tracking. A scraped platform is a stateful browser session. The lifecycle contract is uniform; the work methods are adapter-specific.

### Library shape

Code that's reused across multiple modules or apps. Lives in a clearly-named folder, exports a typed public surface, no app-specific assumptions:

```
src/shared/brand-config/
├── index.ts                   # Public exports — what's allowed to be imported
├── load.ts                    # Internal implementation
├── types.ts                   # The BrandConfig type
└── README.md                  # What this is, how to use it
```

The test: could you copy this folder to another repo and have it work with no app-specific imports? If yes, it's library-shaped. Add it to `BUILDING_BLOCKS_CATALOGUE.md` as a Library candidate.

### MCP server shape

If your app needs language-model-driven access to a domain, build an MCP server (its own folder, its own deployment if needed). One MCP server per coherent domain. Read-only by default. Use stdio transport today; HTTP transport when the shared HTTP MCP infrastructure ships.

```
mcp-servers/my-domain/
├── server.py                  # The server entry point
├── tools/                     # One file per tool
├── schemas/                   # Pydantic / Zod schemas for tool I/O
├── README.md                  # Tool catalogue, transport, install instructions
└── pyproject.toml             # uv project file (Python) or package.json (TS)
```

Don't reach for MCP unless an LLM is choosing between capabilities. For deterministic operations, use a regular adapter or service.

### Service shape

A long-running backend with its own deployment, reachable from multiple apps or outside the request lifecycle. Until a Capability Module is consumed by more than one Application (Juanes' promotion criterion), it stays a module. Don't pre-emptively build services.

If you find yourself building one, structure it like an Application but with no UI: own deployment, own `actor_id` (services are actors), audit-attributed, RPC-mediated writes back to canonical data.

---

## The building blocks catalogue

Every repo maintains a file at its root: **`BUILDING_BLOCKS_CATALOGUE.md`**.

The catalogue captures **promotion candidates** — blocks you built in this repo that have the shape of one of the seven types and could be lifted into shared infrastructure later. It's a forward-looking index, not internal documentation.

### Your responsibility as Claude Code

When you build something that has the shape of one of the seven types (Library, Capability Module, Agent Module, Adapter, MCP Server, Service — Application is the container, not a candidate):

1. **Add a catalogue entry as you build the block** (don't defer it; deferred catalogues don't get written).
2. **Update the entry when the block's surface or dependencies change.**
3. **Also record gaps** — when you needed something that should be a shared building block but built it inline because the shared version doesn't exist. Mark these in the "Gaps deliberately not built" section.

### Catalogue entry shape

The catalogue has an **index table at the top** for fast scanning across repos, then one section per block with structured fields. See the template `BUILDING_BLOCKS_CATALOGUE.md` for the worked pattern. Keep entries short — the catalogue points at the candidate, the candidate's own README does the deep dive.

### Why this matters

Werner periodically scans `BUILDING_BLOCKS_CATALOGUE.md` across all Cloudworkz repos to decide what to promote into the shared monorepo. Without the catalogue, candidates live in heads and folder structures and get rediscovered through archaeology. With it, the conversation is concrete: here are five repos' worth of brand-config-loader implementations, which one do we lift.

---

## Hard nos

Always. No exceptions. No "just for this throwaway."

- **Don't import from libraries that don't exist.** If you're about to write `import { x } from "@cloudworkz/something"` and that package isn't in your `package.json`, build the equivalent inline instead (see "Shape-aware patterns"). Catalogue it. Don't hallucinate the import.
- **Don't bypass `actor_id` and `audit_log`.** Even on a throwaway. Two columns and one trigger are cheap; retrofitting attribution to historical data is expensive.
- **Don't write to tables without a bounded write function.** "Direct INSERT in a route handler" is the pattern that creates the worst legacy code.
- **Don't store secrets in `.env` files committed to git.** Use a secret manager from day one.
- **Don't log PII.** Real names, emails, NI numbers, addresses, financial figures — never. Tokenise or omit at write time.
- **Don't fabricate third-party content.** No invented news items, no made-up analyst ratings under real bank names, no fake "industry peer" data, no fictional quotes attributed to real institutions.
- **Don't silently substitute mock values when an LLM call fails.** The user must see "unavailable" before they see fake-looking real output.
- **Don't disable RLS to "make it easier to debug."** If RLS is in your way, fix the policy or use a privileged role for one query — never disable the table.
- **Don't reach across module boundaries.** Module A doesn't import from `src/modules/billing/internal/`. It imports from `src/modules/billing/` (the module's public surface) or it doesn't import at all.

---

## Decide, build, log — escalate sparingly

Claude Code's default is to **decide and document, not to ask**. Most decisions encountered during a build are yours to make. Pausing for every architectural question turns a fast prototype into a slow committee.

The catalogue is your audit trail. Decide, build, log it in `BUILDING_BLOCKS_CATALOGUE.md`. If a human disagrees on review, the conversation is concrete — "why did you pick X?" → catalogue entry → discussion.

### The short list of things that DO require escalation

These are the few decisions costly enough to undo, or affecting more than this one app, that warrant a pause.

1. **Disabling, weakening, or working around a universal discipline.** If `actor_id`, `audit_log`, RLS, fail-closed AI, no-fabricated-content, or any other rule in the "Universal disciplines" section is in your way and you're tempted to bypass it — stop and ask. The discipline is the architecture; weakening it without a Werner-level call degrades every app's foundation.

2. **Replacing a major tech stack default.** Adding a small npm helper is fine and doesn't need approval. Replacing Next.js with Remix, swapping Supabase for Firestore, introducing a new database, or picking a graph DB / vector DB / search engine the platform hasn't already chosen — escalate. The stack defaults are deliberate; deviating affects more than this repo.

3. **Touching shared canonical data.** If your work would write to or migrate tables in a shared canonical schema (e.g., the Content Brain's atoms / renderings / compliance / pillars / campaigns / voice / recipes / content_rules tables when accessible), escalate. These are governed by their domain owner. Reading from them through controlled paths is fine; writing or migrating is a curator-level decision.

4. **Operating on real production / customer data.** Throwaways and prototypes run on synthetic data, anonymised fixtures, or developer-test rows. If your work would touch live customer records, escalate.

5. **Promoting code into shared infrastructure.** Don't lift a library out of this repo into `@cloudworkz/*` yourself. That's a Werner-level decision made when reviewing the catalogue across multiple apps. Build your code in its Library or Module shape, log it in the catalogue with `Ready` status, and let Werner decide whether and when to promote.

### Things you decide autonomously and document

Default behaviour. No need to ask first.

- **New actor types** (`"scheduled-task"`, `"system"`, `"webhook"`, etc.) if your app needs them. The actors table is generic; new types compose. Log them in the catalogue.
- **New MCP servers for new domains** you're wrapping, provided no existing MCP server already covers that domain. Check the catalogue first; if uncertain, do a quick search in the cloudworkz monorepo. If clear, build.
- **Field additions to your app's own tables.** Migrate, regenerate types, ship.
- **npm package choices within the existing stack family** — a Tailwind plugin, a Zod helper, a Drizzle utility, a Recharts add-on. Pick whichever fits, document in the dependency list.
- **Composition patterns within your app** — what gets its own module folder, what stays inline, when to extract a helper into `shared/`, how routes are organised. These are local design calls.
- **Capability Module shape decisions** — what your module's manifest declares, what its public surface is, what it depends on. The seven-type taxonomy is the vocabulary; *which* module-shape your code takes is your call.
- **Adapter shapes for external systems** — Stripe, Anthropic, Postmark, Companies House, whatever. Follow the lifecycle contract; the specifics are yours.
- **Whether something is shape-shaped or stays a one-off** — most utility functions don't need to be promotion candidates. Only add catalogue entries for things that genuinely look like one of the seven types.

### What Claude Code does NOT do

These are not "escalate" items — they're "not your job" items.

- **Promoting Capability Modules to Services.** A module becomes a Service when consumed by more than one Application; that's evaluated by Werner across the catalogue. Claude Code builds the Capability Module shape; the promotion is curatorial.
- **Lifting code into `@cloudworkz/*` shared libraries.** Build it in-place. Catalogue it. Let Werner promote.
- **Editing the Cloudworkz OS architecture document itself** or this architecture guide. These are governed; suggested changes go to Werner.
- **Approving its own PRs in repos with PR review.** Demo / prototype repos with self-merge on green CI are fine; the monorepo isn't.

### The escalation procedure

When escalation is needed: stop the current line of work, write the question in the repo's `ESCALATIONS.md` (or a similar agreed location), and ask Werner via the active channel (Slack DM, the current chat, whatever's in use). State: what you were doing, what decision is in front of you, what the options are, what your default would be if forced to pick. Wait for the answer. Don't speculate-and-do.

---

## Companion file — `BUILDING_BLOCKS_CATALOGUE.md`

Every repo has one at the root. Initialise it from the template (in `cloudworkz/docs/architecture/os/BUILDING_BLOCKS_CATALOGUE_TEMPLATE.md`) on day one. Maintain it as you build.

---

*Cloudworkz Architecture Guide for Claude Code · v1 · derived from Cloudworkz_OS_Architecture_v1 · Werner Snyman · June 2026*
