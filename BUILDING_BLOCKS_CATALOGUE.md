# Building Blocks Catalogue

**Repo:** `<this-repo-name>`
**Last updated:** `<YYYY-MM-DD>`
**Maintained by:** Claude Code (under direction from `<lead-developer>`)

This file is a **promotion-candidate index** — building blocks built in this repo that have the shape of one of the seven Cloudworkz component types and could be lifted into shared monorepo infrastructure later.

Companion to `architecture.md`. See that document for what "shape" means and which seven types apply.

---

## Index

Use this table for fast scanning. Entries below give the detail.

| Block | Type | Status | One-line |
|---|---|---|---|
| brand-config-loader | Library | Ready | Reads brand.config.ts and exposes typed brand context. |
| ai-guardrails | Library | Ready | Fail-closed validation wrapper for LLM call sites. |
| actor-attribution-helper | Library | Ready | Captures actor_id on every write; supplies it to bounded write functions. |
| stripe-billing-adapter | External Integration | Needs refactor | Stripe customer/subscription/webhook wrapper. |

**Status values:** `Ready` (stable shape, lifts cleanly) · `Proven in use` (working but ad-hoc; refactor before promotion) · `Needs refactor` (shape is right but implementation has app-specific assumptions) · `Experimental` (in flight, not stable enough for promotion).

---

## Blocks built in this repo

### brand-config-loader

**Type:** Library candidate

**One-line:** Reads `brand.config.ts` and exposes a typed brand context via React hook.

**Lives at:** `src/shared/brand-config/`

**Surface:**
- `loadBrandConfig(path: string): BrandConfig` — load and validate config from path.
- `BrandConfigProvider` — React provider, wrap the app once.
- `useBrand(): BrandConfig` — React hook, consumed anywhere in the tree.

**Depends on:** none (no env vars, no external services).

**Promotion readiness:** Ready. Stable shape, no app-specific assumptions. Would lift cleanly to `@cloudworkz/app-shell` (or `@cloudworkz/brand`).

**Notes:** Built because the shared shell library doesn't exist yet. The shape matches what `Cloudworkz_OS_Architecture_v1` describes for the shell's brand-config responsibility. When two or more apps need brand-aware styling, this is the candidate to lift.

---

### ai-guardrails

**Type:** Library candidate

**One-line:** Fail-closed validation wrapper for LLM call sites — schema check + compliance check + advice-term scan; raises on fail.

**Lives at:** `src/shared/ai-guardrails/`

**Surface:**
- `guardLLMCall(callFn, { schema, complianceRules }): Promise<ValidatedOutput>` — wraps an LLM call, returns validated output or throws.
- `validateOutput(output, { schema, complianceRules }): ValidationResult` — standalone validator.

**Depends on:** Zod (for schema validation). No external services.

**Promotion readiness:** Ready. Pattern is identical to what `unlock-demo-onboarding` PR #2 used; would lift cleanly to a shared `@cloudworkz/ai-guardrails` library.

**Notes:** The principle (fail-closed on every LLM call site, no silent mock fallbacks) is universal — every Cloudworkz app that calls an LLM should use this. Strong promotion candidate.

---

### actor-attribution-helper

**Type:** Library candidate

**One-line:** Captures the current actor's `actor_id` and supplies it to every bounded write function.

**Lives at:** `src/shared/actor/`

**Surface:**
- `getCurrentActor(): { actor_id: string; actor_type: 'human' | 'agent' | 'integration' }` — pulls from session / context.
- `withActor(actor, fn)` — runs `fn` with `actor` as the current actor for the duration.
- `requireActor()` — throws if no actor in context.

**Depends on:** Supabase Auth session (for human actors). For agents and integrations, the caller supplies the actor record.

**Promotion readiness:** Ready, with one caveat — current implementation pulls from Supabase Auth directly. Promotion candidate is the same shape with a pluggable session source (`@cloudworkz/auth`).

**Notes:** Every app that writes to canonical tables needs this. The Supabase RPCs already require an `actor_id` argument; this helper supplies it consistently. Strongest promotion candidate in the catalogue.

---

### stripe-billing-adapter

**Type:** External Integration candidate

**One-line:** Stripe wrapper for customer creation, subscription management, and webhook handling.

**Lives at:** `src/adapters/stripe-billing/`

**Surface:**
- `init()` / `auth()` / `observe()` / `teardown()` — lifecycle contract.
- `createCustomer(...)`, `subscribeCustomer(...)`, `cancelSubscription(...)`, `getInvoices(...)` — work methods.
- `handleWebhook(event)` — webhook receiver.
- Capability descriptor: `{ canCreateCustomers, canSubscribeCustomers, canHandleWebhooks, supportsMetered }`.

**Depends on:** Stripe SDK, `STRIPE_SECRET_KEY` env var (from secret manager), webhook signing secret.

**Promotion readiness:** Needs refactor. Currently mixes Stripe-specific concerns with this app's billing UI flow. Promotion candidate is the adapter portion (lifecycle + work methods + capability descriptor) lifted into `@cloudworkz/adapters-core/stripe`; the app-specific UI flow stays in `module-billing`.

**Notes:** The `module-billing` foundational module (when built) will consume an adapter of this shape. This implementation is informative for the shape but not lift-ready as-is.

---

## Gaps deliberately not built

When you needed something that should be a shared building block but built it inline (or didn't build it at all) because the shared version doesn't exist yet, record it here. Werner uses this list to see what *should* be in the shared monorepo even when no app has formalised it yet.

### module-loader

**What it would be:** A function that walks a registry of modules and wires their routes, navigation entries, settings panels, and lifecycle hooks into the application's shell.

**Why this app didn't build it:** Single-app with only a handful of routes — manual route registration in `src/shell/router.ts` is cheaper than building a registry consumer.

**What this app does instead:** Each module declares its `module.config.ts` for documentation, but `src/shell/router.ts` registers routes manually by importing each module's `pages/` folder.

**Promotion path:** When `@cloudworkz/app-shell` ships, the shell library reads `module.config.ts` files from a configured module path and does the wiring automatically. This app's manual registration deletes; the `module.config.ts` files stay (they're already correct).

---

### transcript-store

**What it would be:** A central store (DB table + writer + reader) for agent run transcripts — input, prompts, tool calls, outputs, cost. Auditable, queryable by agent, time-range, and outcome.

**Why this app didn't build it:** Only one agent in this app, and its transcripts are written to local JSONL files in `src/agents/<agent>/transcript/`. Building the central store for one consumer is premature.

**What this app does instead:** JSONL files per agent run. Same shape data, just file-local instead of central.

**Promotion path:** When the agent runtime infrastructure (transcript provider + cost ledger + dispatch contract) ships, the local JSONL writer is replaced with a call to the central provider. The transcript shape doesn't change — it lifts directly.

---

### http-mcp-transport

**What it would be:** Shared HTTP transport infrastructure for MCP servers, with authentication, per-request actor mapping, and rate limiting. Required for any non-Cowork MCP consumer (modules with LLM features, deployed agents, future external clients).

**Why this app didn't use it:** It doesn't exist. Current MCP servers are stdio-only.

**What this app does instead:** If it needs canonical content from the Content Brain, it queries the Supabase tables directly via the application's data layer (with `tenant_id` scoping and RLS enforcing isolation), bypassing the MCP server.

**Promotion path:** When HTTP transport ships, modules with LLM features replace direct DB calls with MCP tool calls. The direct-DB code is removable at that point.

---

## How to add a new entry

When you build a new block of one of the catalogued types:

1. Add a row to the **Index** table at the top (block name, type, status, one-line).
2. Add a detailed section below in the "Blocks built in this repo" area, following the entry shape (Type / One-line / Lives at / Surface / Depends on / Promotion readiness / Notes).
3. Keep entries short. The catalogue points at the candidate; the candidate's own README does the deep dive.
4. Update the entry when the block's surface or dependencies change.

When you decide *not* to build something that should be a shared block:

1. Add a section to "Gaps deliberately not built" with: what it would be / why this app didn't build it / what this app does instead / promotion path.

---

*Template version 1 · pairs with `architecture.md` · derived from Cloudworkz_OS_Architecture_v1 · June 2026*
