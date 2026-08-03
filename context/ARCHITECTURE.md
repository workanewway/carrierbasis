# context/ARCHITECTURE.md — Stack, Conventions, Auth/RLS, Sensitive Data

> Loaded by the Foundry after CONTEXT.md. Design WITHIN this stack; do not
> substitute frameworks or clients.

## Stack

- **Backend:** serverless TypeScript on Vercel, one file per route under `api/`
  (e.g. `api/compliance-files/[id]/lock.ts`). NOT Next.js/app-router/RSC.
- **Data:** Supabase (Postgres), RLS ENFORCED (see below). Cloudflare R2 for
  documents + locked records (bucket `vetting-platform-docs`, staging bucket
  `vetting-platform-docs-staging`). FMCSA QCMobile is the authoritative carrier
  source; the tenant TMS (Sunnybrook first) is the second live source.
- **Frontend:** static HTML + vanilla JS calling the API with the tenant key
  (no React/Next/hooks/framework). The browser NEVER talks to Supabase directly
  (no `createClient`) — only `/api/*`. App serves at the ROOT of BOTH
  `app.carrierbasis.com` (live 2026-08-02) and `broker.workanewway.com` (kept
  live indefinitely; no forced sunset).

## Endpoint conventions (the canonical pattern — follow it exactly)

Model any new endpoint on `api/vettings/[id]/apply.ts` (the canonical v1
pattern; still the reference) and the v2 endpoints under `api/carrier-profiles`,
`api/loads`, `api/compliance-files`, `api/eligibility`:

- Handler `(req, _res, ctx)`; wrap with `withHandler(handler, { methods:[...] })`.
- **EVERY relative import ends `.js`** (compiled ESM resolution).
- Pick the Supabase client per AUTH below — `ctx.supabase` for tenant data,
  `getSupabase()` only pre/cross-tenant. Never a bare client.
- `throw errors.xxx()` (never call sendError directly). Return plain objects
  (never `res.json()`) — the wrapper emits `{ success: true, ... }`.
- Long jobs set `export const maxDuration` (60 for FMCSA/TMS/Claude paths).
- **Any rule-governed check calls `effectiveRuleConfig()` from `lib/policy.ts`
  — never a hardcoded severity/blocking literal.** As of 2026-07-22 this is
  actually enforced (see context/ENTITIES.md "Broker Policy"); a new check
  that skips it will silently ignore whatever a broker configures.
- Do NOT import `@anthropic-ai/sdk` — call the Anthropic REST API via native
  `fetch` (see `api/compliance-files/[id]/assess.ts`).
- Persistence writes that matter MUST check they took (`.select(...)` + throw on
  0 rows) — a silent 0-row update returning success was a real v1 bug.

## Auth & tenant isolation (RLS is real — do not bypass it)

RLS is enforced via a JWT claim. TWO clients:

- **`ctx.supabase`** = per-request TENANT-SCOPED client. `lib/auth.ts` mints an
  ES256 JWT (`role:'authenticated'` + `tenant_id`); `current_tenant_id()` reads
  `request.jwt.claims->>'tenant_id'`, so RLS policies actually filter.
- **`getSupabase()`** = service client, BYPASSRLS (sees all tenants).

**THE RULE (I6):** a tenant-OWNED-data endpoint MUST use `ctx.supabase`, NEVER
`getSupabase()`. Keep `.eq('tenant_id', ctx.tenantId)` as belt-and-suspenders.
404 (not 403) on cross-tenant miss. `getSupabase()` ONLY for pre/cross-tenant
endpoints (auth/admin/onboarding/crons) or GLOBAL tables (`carriers` cache).
"authenticated" ≠ "tenant-scoped" — scope by whether it reads tenant-OWNED data.

Tenant-owned tables: `carrier_profiles`, `loads`, `compliance_files`,
`documents`, `narration_entries`, own `tenants` row. Global: `carriers` cache.

**Auth surface:** DUAL-MODE (per-user auth shipped 2026-07-28). Either credential
works on the Authorization header and both converge on the same tenant-scoped
ES256 JWT + RLS:
- **API key** (original): hashed access code → `nwd_<slug>_<rand>` key. `/api/access`.
- **User session**: Supabase Auth login → session JWT. `/api/login` (password +
  refresh grants, proxied server-side so the browser only talks to `/api/*`);
  `/api/set-password` completes invites (password only); `lib/auth.ts` verifies
  the session token against the project JWKS, then resolves `memberships` to a
  tenant. `ctx.user` (id/email/role/firstName/lastName) and `ctx.authMethod`
  ('api_key'|'session') carry identity; names come from user_metadata in the
  signed token. First/last name is admin-managed identity (set at invite,
  editable in Settings → Users, action `set_name`) and becomes the authenticated
  reviewer name on seals.
`memberships (user_id, tenant_id, role admin/member)` is the users↔tenants join;
service-client-managed (no authenticated write policy), tenant members can SELECT
their own roster. `lib/requireTenantAdmin(ctx, action)` gates settings WRITES
(policy/sources/users) on a session admin; reads + sealing stay open to members.
The shared client layer is `public/js/platform-session.js` (session-first,
API-key fallback, single-flight refresh). CUTOVER is per-tenant: revoke a tenant's
`api_keys` once all its people have logins; both modes work until then.

Signing keys are per-environment: staging and production use DIFFERENT ES256
keypairs (`SUPABASE_JWT_SIGNING_KEY`, env-scoped — no code change).

## Sensitive-data handling

- **TMS credentials are ENCRYPTED at rest.** `tenant_integrations.credentials`
  (jsonb) holds the TMS login as an AES-256-GCM envelope
  `{"enc":"enc:v1:<iv>:<tag>:<ct>"}`. `lib/crypto.ts` en/decrypts; key
  `TMS_ENCRYPTION_KEY` (base64 32 bytes) is set in BOTH environments with the
  SAME value (staging must decrypt the same envelopes). `lib/tms.ts` decrypts on
  read. Never write plaintext; deploy decrypt code to an env BEFORE encrypting
  its rows.
- **READ-ONLY TMS credentials.** A decrypted credential may carry
  `read_only: true`. EVERY outbound TMS call MUST go through
  `lib/tms.ts tmsFetch()`, which throws `TmsReadOnlyError` on any non-GET/HEAD/
  OPTIONS request when that flag is set. This exists so a non-production
  environment can hold a REAL client's TMS credential for testing with no
  possibility of mutating the client's live data. When TMS assignment
  write-back (D7) is built it MUST route through `tmsFetch` — never call `fetch`
  against a TMS directly. Managed by the local `tms-cred.mjs` utility
  (`set --read-only`; `rotate` preserves the flag).
- **API keys are hashed-only.** `/api/access` mints a fresh key per connect,
  stores only its SHA-256 hash; plaintext returned once.
- **Document analysis degrades gracefully.** A failed AI analysis still stores
  the doc in R2 and returns partial success — an upload is never lost.
- **Narration is append-only + non-blocking.** `narration_entries` rows are
  never updated/deleted (RLS `USING(false)`); a failed write degrades to an
  inline note, never blocks the action. POLYMORPHIC ownership (corrected
  2026-07-20 from an earlier CF-only design): a row belongs to EITHER a
  Compliance File OR a living Carrier, never both. Four actions as of
  2026-07-22 (`fmcsa`/`tms`/`assessment`/`evidence`) — see context/DATA.md for
  the database constraint gotcha that cost a full debugging day when the 4th
  was added without updating it.
- **Any endpoint that fires a fire-and-forget narration/analysis write after
  its main response needs `export const maxDuration = 60`** — the endpoint's
  own success response can return well before that background work finishes,
  and without the extension the serverless instance is very likely torn down
  mid-write, silently. This bit `api/carrier-profiles/[id]/apply.ts`
  specifically (2026-07-22): it started narrating without getting the same
  extension its siblings (`refresh.ts`, `assess.ts`, `documents.ts`) already
  had. Check for this whenever ANY endpoint gains a new `waitUntil`-registered
  call, not just the obviously-slow ones.

## Environments (two of everything — the #1 historical time sink)

Production (`main` → prod Vercel + prod Supabase + `vetting-platform-docs`) and
staging (`staging` branch → preview deploys + staging Supabase +
`vetting-platform-docs-staging`). Vercel env scoping: production values are
Production-only; staging values are plain Preview; shared config (Anthropic,
FMCSA, `TMS_ENCRYPTION_KEY`) is Production+Preview. When verifying, always name
the environment — confusing the two (SQL editor on staging while PowerShell hits
prod) has cost hours.
