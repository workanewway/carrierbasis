# CONTEXT.md — Canonical Platform Context (INDEX)

> SINGLE SOURCE OF TRUTH for what this platform IS. Foundry research/design agents
> fetch it live (research→`main`, design→`staging`); chat + the vetting-platform
> skill defer to it. A change that alters shape (surfaces, endpoints, data model,
> auth, conventions) updates this file — and the relevant module below — in the
> SAME change.
>
> **This file is an INDEX.** It carries the non-negotiable invariants inline (so
> an agent is safe even if a module fetch fails) plus a manifest of context
> modules. The fetcher (`getProjectContext`) reads this file, then fetches each
> listed module and concatenates them. Keep THIS file well under 11,000 chars;
> put detail in the modules, each also under ~11,000. The product name is
> "CarrierBasis" (exact casing — see the naming invariant below); never render
> a tenant name as customer-facing framing.

## What this platform is (one paragraph)

A standalone multi-tenant SaaS for freight brokers that produces court-ready,
tamper-evident records of "reasonable care" in carrier selection — a compliance
tool after *Montgomery v. Caribe Transport II* (May 2026) let safety-based
negligent-selection claims through the FAAAA preemption safety exception. Sold
broadly to SMB brokers; **not built for any single broker.** The platform
separates a continuously-maintained view of each carrier from the immutable
evidence created when a broker commits a carrier to a load: three entities
(**Carrier**, **Load**, **Compliance File**) plus one non-persisted capability
(**Eligibility Check**). Full model in `context/ENTITIES.md`.

## NON-NEGOTIABLE INVARIANTS (inline — always in force, even if a module fetch fails)

- **THE PRODUCT NAME IS "CarrierBasis"** (named 2026-08-02; a product of NewWay
  Digital). Exact casing, one word, capital C and B — never "Carrier Basis",
  never "carrierbasis" in prose; "CB" is the icon monogram only. Brand mark
  assets come from the designer's kit (reused verbatim, never recreated):
  short lockup in the nav rail, stacked lockup on login/connect, the kit icon
  for favicons, the short-navy lockup in the compliance PDF header
  (`lib/brandAssets.ts`). Kit palette: navy `#16263F`, ember `#EF8B44`, sky
  `#9CC0E8`, deep-sky `#5B7EA6`; type Archivo (wordmark) + IBM Plex Mono
  (tagline "Decision. Documented. Sealed."). Endorsed-brand structure:
  CarrierBasis leads; NewWay Digital appears only in the endorsement position
  ("a product of NewWay Digital" — footers, invoices) — never the NewWay gear
  in the app chrome. The product UI keeps its own operational dark-tile
  aesthetic; the marketing site's look is NOT imported into the app. NEVER
  surface a tenant/client's name in product UI (that half of the old rule
  stands unchanged). Home: **app.carrierbasis.com** (live 2026-08-02) plus a
  `carrierbasis.com/login` redirect. `broker.workanewway.com` still serves the
  same app indefinitely — both hosts work; there is no forced sunset.
- **Evidence is a DEEP SNAPSHOT, never a reference (I1).** The Carrier keeps NO
  history — it is overwritten as sources refresh. So the Compliance File is the
  SOLE keeper of point-in-time truth: at lock it copies EVERYTHING relied upon
  (carrier facts, TMS result, load fields, pinned document hashes, AI analysis,
  human review, decision, timeline). No evidentiary field may be a live lookup.
- **CF immutability boundary = Locked (I4).** Any mutation of a Compliance File
  at or after `locked` is refused (409). `assigned`/`filed` are status-only and
  never touch evidence. The R2 JSON + PDF exist ONLY from the lock transition on.
- **RLS is real — do not bypass it.** Tenant-OWNED data (carrier_profiles, loads,
  compliance_files, documents, narration) MUST use the per-request tenant-scoped
  client (`ctx.supabase`), NEVER the service client (`getSupabase()`, which
  BYPASSRLS). Global tables (`carriers` cache) may use the service client. 404
  (not 403) on a cross-tenant miss. Full model in `context/ARCHITECTURE.md`.
- **Operational state never leaks into evidence (I3).** A Carrier's operational
  `state` (unreviewed/review_in_progress/ready/needs_attention) is display
  vocabulary; it never renders as a determination on a Compliance File.
- **No Load, no Compliance File (I5).** A CF is always carrier×load. Manual load
  entry exists so this never hard-depends on a live TMS.
- **JSON snapshot is the cryptographic source of truth; the PDF is presentation.**
- **AUTH is DUAL-MODE (per-user auth shipped 2026-07-28).** Two credentials, one
  enforcement layer. (a) TENANT access code → API key (the original path, still
  fully live); (b) per-USER login via Supabase Auth → session token. `lib/auth.ts`
  accepts either on the Authorization header and converges BOTH on the same
  per-request tenant-scoped ES256 JWT + RLS — endpoints and policies are agnostic
  to which door was used. `memberships (user_id, tenant_id, role admin/member)`
  joins users to tenants; `ctx.user` + `ctx.authMethod` expose the logged-in
  identity when present. Authorization is unchanged (still tenant-scoped RLS);
  identity is what's new. CUTOVER is per-tenant: once everyone at a tenant has a
  login, that tenant's `api_keys` rows are revoked and its access code stops
  mattering — until then both work. CONSEQUENCE UPDATE (2026-07-28): the sealed
  `reviewer` is now set — a session seal records an AUTHENTICATED reviewer
  (`reviewer_identity_source='authenticated'`, `reviewer_user_id`, the reviewer's
  account name from user_metadata read off the signed token — falling back to
  email, never free-typed — plus optional role, no "(self-attested)" annotation);
  an API-key seal still records a self-attested free-text reviewer. Records sealed
  before the flip stay "(self-attested)"
  forever — no retroactive relabel (immutable evidence). Settings WRITES (policy,
  data sources, users) are admin-only via `requireTenantAdmin`; reads and sealing
  are open to any member.
- **HIDE, NEVER DELETE (2026-07-21).** A carrier profile may be referenced by a
  sealed compliance file, and evidence must never lose the record it points at.
  `carrier_profiles.hidden_at` is presentational only. This instinct applies to
  anything an evidentiary record can point at.
- **NEVER INFER MEANING FROM DISPLAY TEXT (2026-07-21).** `state_reasons` holds
  rule objects `{rule_id, label, severity, source, blocking?}`. `rule_id` is a
  PERMANENT identity because sealed files carry it forever; `label` is free to
  reword. Read `reason.blocking` / `reason.rule_id` — never `label.includes(...)`.
  Two live bugs came from exactly that, including a hard-block decision. All
  readers go through `normalizeStateReasons()` in `lib/carrierState.ts`.
- **ONE RULE VOCABULARY, declared in `lib/rules.ts` (2026-07-21).** Every rule
  the platform can evaluate is registered there exactly once, and broker policy
  is keyed by those ids. Two naming rules are binding: ids are POSITIVE (they
  name what must be TRUE, so `authority.active`, never `authority.not_active` —
  a policy is a list of requirements), and they are namespaced BY SUBJECT, never
  by source (`agreement.on_file`, not `tms.no_agreement`; `source` is already
  its own field). Never rename or reuse an id — add. The `floor.*` namespace and
  the negative-polarity state-reason ids were the FIRST generation, unified
  while no production file had sealed; they map forward via `RENAMED_RULE_IDS`.
  **The floor is DERIVED from the registry via `resolvePolicy()`** — there is no
  separate hand-maintained default object; `PLATFORM_DEFAULT_POLICY` (the old
  placeholder) was deleted 2026-07-21 once `resolvePolicy(null)` became the
  single definition of "no tenant policy saved."
- **THE VOCABULARY EXISTING IS NOT THE SAME AS IT BEING ENFORCED (2026-07-22).**
  `broker_policies` + `effectiveRuleConfig()` shipped 2026-07-21, but for a full
  day nothing outside CF-creation's narrow uses (resolution reconfirmation, the
  sealed snapshot) ever consulted it — carrier-level Concerns, carrier `state`,
  and load eligibility ALL ran on `lib/rules.ts`'s hardcoded defaults regardless
  of what a broker configured. Confirmed live: a broker-configured hard stop
  produced no visible effect anywhere. Now fixed — `lib/carrierState.ts` and
  `api/eligibility.ts` both call `effectiveRuleConfig()`. **When adding a new
  rule-governed check, verify it's actually WIRED to the policy engine, not just
  that the rule id is registered** — those are two different facts, and only
  one of them was true for a full day.
- **A closed vocabulary enforced by a DATABASE CHECK CONSTRAINT needs the
  constraint updated in the SAME change as the code (2026-07-22).** Adding
  `'evidence'` to the `NarrationAction` TypeScript union without updating
  `narration_entries.action`'s own `check (...)` constraint meant every insert —
  real or degraded-stub fallback — was silently rejected by Postgres for hours,
  with zero error surfacing anywhere (a failed degraded write fails silently by
  design). TypeScript has no visibility into a database's own enforcement.
  Check schema files, not just the type, before adding to ANY closed vocabulary.
- **Anything entering IMMUTABLE evidence has an ordering deadline.** A change to
  what gets sealed must land BEFORE the first real seal, because snapshots are
  hash-verified and cannot be retrofitted. Ask this question early, not late.

## CONTEXT MODULES — the fetcher loads each of these after this index

```context-manifest
context/ENTITIES.md      — Carrier / Load / Compliance File / Eligibility: states, lifecycles, invariants I1–I7, decisions D1–D7
context/ARCHITECTURE.md  — stack, endpoint conventions, auth/RLS model, sensitive-data handling
context/SURFACES.md      — page/endpoint surface map, UI vocabulary, platform.css design rules
context/DATA.md          — tables, columns, R2 layout, seal mechanics
context/ROADMAP.md       — deferred items, each marked explicitly NOT BUILT
```

If a module fails to fetch, the fetcher surfaces that visibly and proceeds on
this index plus the modules that loaded — degraded, not dangerous. The invariants
above are the safety floor that always applies.

## Tenants

Internal test tenants: **"bivium"** and **"acme"** (access codes the same). They
exercise multi-tenancy — not real customers; their names never appear in
customer-facing copy, titles, or UI.
