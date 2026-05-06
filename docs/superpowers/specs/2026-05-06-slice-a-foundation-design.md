# Slice A — Foundation Design

**Date:** 2026-05-06
**Author:** Claude (under CEO direction)
**Status:** Approved (pending CEO review of this written spec)
**Phase:** Phase 1A, sub-slice A (consolidates "auth & multi-tenancy foundation" + Phase-0 repo skeleton from `BUILD_STRUCTURE.md`)
**Estimated effort:** ~5–7 focused dev-days

---

## 1. Goal

Stand up the bare-minimum foundation that everything else depends on:

> A user can sign up, create a tenant, invite teammates, log in as any of four roles, and the platform correctly isolates each tenant's data — with audit logging and feature flags wired in from day one.

**Testable success state:** Sign up the bootstrap super-admin → create a tenant via `/admin/tenants` → invite a `CLIENT_OWNER` → owner accepts and lands on `/t/[slug]/dashboard` → owner invites `CLIENT_STAFF` → role-correct nav visible per role → cross-tenant Prisma query throws → audit log shows every mutation.

No real product features yet (no GBP data, no leads, no reviews, no Stripe). Those land in Slices B and C.

---

## 2. Out of Scope (Slice A)

These are deliberately deferred; do not build them in this slice:

- Stripe billing, `Subscription` table, paid signup flow → **Slice B**
- GBP OAuth, `GBPSnapshot`, dashboard widgets → **Slice B**
- `Lead`, `createLead()`, Twilio integration → **Slice B**
- `Review`, review queue UI, AI drafts → **Slice B/C**
- Monthly PDF report → **Slice C**
- Postgres Row-Level Security policies → hardening pass at end of Slice C
- OAuth/magic-link login, MFA, password reset email customization → post-Slice C
- Subdomain-per-tenant routing → Phase 4 polish per `BUILD_STRUCTURE.md`
- `packages/ui`, `packages/integrations`, `packages/jobs`, `apps/client-template` → empty stubs only

---

## 3. Architecture

### 3.1 Repo layout (monorepo, pnpm workspaces + Turborepo)

```
/apps
  /platform/                  Next.js 14 (App Router, TypeScript strict)
    /app
      /(public)/login         email + password
      /(public)/signup        bootstrap super-admin OR invite-token-only
      /(public)/accept-invite/[token]
      /(authed)/              session-required layout
        /select-tenant        list of tenants user can access (auto-redirects if exactly 1)
        /t/[tenantSlug]/      tenant-scoped layout: resolves slug → tenant, verifies access, sets ALS
          /dashboard          role-aware placeholder
          /settings/team      invite UI, member list
          /settings/flags     super-admin only — toggle feature flags for this tenant
        /admin/               platformRole = SUPER_ADMIN only
          /tenants            list + create
          /tenants/[id]       per-tenant detail (set tier/status manually in Slice A)
          /tenants/[id]/flags feature flags for that tenant
          /audit-log          system-wide audit log viewer (filterable)
      /api/trpc/[trpc]/route.ts
      /api/auth/callback/route.ts        Supabase auth callback
    /server
      /trpc/                  routers + middleware (see §11, §7, §8)
      /db.ts                  Prisma client w/ tenant-scoping middleware
      /context/als.ts         AsyncLocalStorage for tenant/user context
    /lib                      app-local glue (re-exports from packages where appropriate)
  /client-template/           empty placeholder (.gitkeep + README)

/packages
  /db                         Prisma schema + migrations + seed
    schema.prisma
    /migrations
    seed.ts
    package.json
  /lib
    /auth                     getSession, requireRole, requirePlatformRole
    /feature-flags            isFeatureEnabled, DEFAULT_FLAGS_BY_TIER
    /audit                    audit.log helper, redaction config
    /errors                   typed error classes
    /logger                   pino setup
    package.json
  /ui                         empty stub (.gitkeep + README)
  /integrations               empty stub
  /jobs                       empty stub

/docs
  /superpowers
    /specs/                   design docs (this file)
    /plans/                   implementation plans (writing-plans output)

/.env.example                 every variable across all 3 slices, placeholder values
/.gitignore
/turbo.json
/pnpm-workspace.yaml
/package.json                 root, scripts only
/tsconfig.base.json
/.editorconfig
/.prettierrc
/.eslintrc.cjs
```

### 3.2 Tech stack baseline (Slice A)

| Concern | Choice | Notes |
|---|---|---|
| Framework | Next.js 14 App Router | TypeScript strict |
| Package manager | pnpm | required |
| Monorepo | Turborepo | `turbo.json` with `dev`, `build`, `lint`, `test`, `typecheck` |
| ORM | Prisma | tenant-scoping middleware |
| DB | Postgres via Supabase | Supabase CLI for local Docker |
| Auth | Supabase Auth | `@supabase/ssr` for cookie sessions |
| API | tRPC v11 | type-safe end-to-end |
| UI | Tailwind + shadcn/ui (zinc) | installed in `apps/platform` only for Slice A |
| Forms | React Hook Form + Zod | |
| Logger | pino | with redaction list (see §13) |
| Tests | Vitest | unit + tRPC integration |
| E2E | Playwright | scaffold + 1 smoke test, full suite Slice C |
| Lint/format | ESLint + Prettier | strict ruleset |
| Node | 20 LTS | enforced via `.nvmrc` and `engines` field |

---

## 4. Data Model

Prisma models — Slice A only. Every tenant-scoped model has `tenantId String` and is registered in the tenant-scoping middleware (see §7).

### 4.1 Models

```prisma
model User {
  id              String        @id @default(cuid())
  supabaseUserId  String        @unique  // FK to Supabase auth.users.id
  email           String        @unique
  name            String?
  platformRole    PlatformRole  @default(NONE)
  tenantUsers     TenantUser[]
  createdAt       DateTime      @default(now())
  updatedAt       DateTime      @updatedAt

  @@index([platformRole])
}

model Tenant {
  id              String        @id @default(cuid())
  slug            String        @unique
  name            String
  tier            Tier
  status          TenantStatus  @default(ACTIVE)
  tenantUsers     TenantUser[]
  invites         Invite[]
  featureFlags    FeatureFlag[]
  createdByUserId String
  createdAt       DateTime      @default(now())
  updatedAt       DateTime      @updatedAt

  @@index([status])
}

model TenantUser {
  id              String      @id @default(cuid())
  tenantId        String
  userId          String
  role            TenantRole
  invitedByUserId String?
  invitedAt       DateTime    @default(now())
  acceptedAt      DateTime?
  tenant          Tenant      @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  user            User        @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([tenantId, userId])
  @@index([userId])
}

model Invite {
  id              String      @id @default(cuid())
  tenantId        String
  email           String
  role            TenantRole
  token           String      @unique
  invitedByUserId String
  expiresAt       DateTime
  acceptedAt      DateTime?
  acceptedByUserId String?
  tenant          Tenant      @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  createdAt       DateTime    @default(now())

  @@index([tenantId, email])
  @@index([expiresAt])
}

model FeatureFlag {
  id              String      @id @default(cuid())
  tenantId        String
  key             String
  enabled         Boolean
  metadata        Json?
  updatedByUserId String?
  updatedAt       DateTime    @updatedAt
  tenant          Tenant      @relation(fields: [tenantId], references: [id], onDelete: Cascade)

  @@unique([tenantId, key])
}

model AuditLog {
  id            String    @id @default(cuid())
  tenantId      String?
  actorUserId   String?
  action        String
  targetType    String?
  targetId      String?
  metadata      Json?
  ip            String?
  userAgent     String?
  createdAt     DateTime  @default(now())

  @@index([tenantId, createdAt])
  @@index([actorUserId, createdAt])
  @@index([action, createdAt])
}
```

### 4.2 Enums

```prisma
enum PlatformRole {
  NONE         // regular client user, no platform-wide privileges
  TEAM         // internal staff / VA
  MANAGER      // internal manager — can approve negative review responses (later slice)
  SUPER_ADMIN  // agency owners — cross-tenant access
}

enum TenantRole {
  CLIENT_OWNER  // signed up the account
  CLIENT_STAFF  // invited by client_owner
  TEAM          // internal staff assigned to this tenant
  MANAGER       // internal manager assigned to this tenant
}

enum Tier {
  STANDARD
  FLAGSHIP
}

enum TenantStatus {
  ACTIVE
  PAST_DUE
  CANCELED
  SUSPENDED
}
```

### 4.3 Tenant-scoped models registry

```ts
// packages/lib/src/tenant-scoping/scoped-models.ts
export const TENANT_SCOPED_MODELS = [
  'TenantUser',
  'Invite',
  'FeatureFlag',
  'AuditLog',  // when tenantId is set; nullable cases are super-admin/system
] as const;
```

Slice B will append `Lead`, `CallRecord`, `Review`, `GBPSnapshot`, etc. to this list as those models are introduced.

---

## 5. Auth & Sessions

### 5.1 Provider

Supabase Auth, email + password only. `@supabase/ssr` for server-side cookie sessions (the official Next.js 14 pattern). No magic link, no OAuth, no MFA in Slice A.

### 5.2 Bootstrap super-admin

The very first user to sign up becomes `platformRole = SUPER_ADMIN`. After `User.count() > 0`, the signup endpoint refuses standalone signups and only accepts the `accept-invite` flow.

Implementation: signup tRPC procedure checks `User.count()` inside a transaction before creating the user.

Race condition: theoretical (two simultaneous first-time signups) — handled by a unique constraint, second one falls back to invite-required path with a clear error.

### 5.3 Signup flow (post-bootstrap)

1. User clicks invite link: `/accept-invite/[token]`
2. Server validates token (exists, not expired, not accepted).
3. If user is logged in and `User.email !== Invite.email` → 403 with explicit message.
4. If not logged in → render Supabase signup form pre-filled with invite email (read-only).
5. On Supabase signup success → callback creates `User` row + `TenantUser` row, marks `Invite.acceptedAt`.
6. Redirect to `/t/[slug]/dashboard`.

### 5.4 Login flow

`/(public)/login` → Supabase email/password → callback exchanges code → redirect to `/select-tenant` (or `/t/[slug]/dashboard` if exactly one tenant, or `/admin/tenants` if super-admin with no tenant memberships).

### 5.5 Session shape

The tRPC context per request:

```ts
type Session = {
  userId: string;
  email: string;
  platformRole: PlatformRole;
  // populated only inside /t/[tenantSlug]/* routes:
  currentTenantId?: string;
  currentTenantUserRole?: TenantRole;
  // super-admin viewing a tenant they're not a TenantUser of:
  isCrossTenantView?: boolean;
};
```

---

## 6. Tenant Routing & Resolution

### 6.1 URL pattern

`/t/[tenantSlug]/...`. Slug is the human-readable handle (e.g. `acme-hvac`).

### 6.2 Resolution & access check (in `(authed)/t/[tenantSlug]/layout.tsx`)

```
1. Read session from Supabase cookies.
2. Look up Tenant by slug.
3. If not found → 404.
4. If session.platformRole === SUPER_ADMIN:
     - access granted, isCrossTenantView=true if no TenantUser row, audit log entry written.
5. Else:
     - Look up TenantUser(tenantId, userId).
     - If not found → 404 (NOT 403, prevents tenant enumeration).
     - access granted, currentTenantUserRole = TenantUser.role.
6. Wrap child render in AsyncLocalStorage with { userId, currentTenantId, currentTenantUserRole, isCrossTenantView }.
```

### 6.3 `/select-tenant`

Lists every tenant the current user can access (TenantUsers + super-admin = all). Auto-redirects if the list has exactly one element.

---

## 7. Tenant Isolation (Prisma Middleware)

### 7.1 The middleware

```ts
// apps/platform/server/db.ts (excerpt)
import { PrismaClient } from '@prisma/client';
import { TENANT_SCOPED_MODELS } from '@gbp/lib/tenant-scoping/scoped-models';
import { als } from './context/als';
import { TenantContextError } from '@gbp/lib/errors';

export const prisma = new PrismaClient();

prisma.$use(async (params, next) => {
  if (!TENANT_SCOPED_MODELS.includes(params.model as any)) return next(params);
  const ctx = als.getStore();
  if (!ctx) throw new TenantContextError(`Query against ${params.model} with no tenant context`);
  if (ctx.crossTenant) return next(params);  // super-admin explicit cross-tenant
  if (!ctx.tenantId) throw new TenantContextError(`No tenantId in context for ${params.model}`);

  // Inject tenantId into where clauses on read; force tenantId on writes.
  switch (params.action) {
    case 'findUnique':
    case 'findUniqueOrThrow':
      // findUnique can't use tenantId in compound key unless it exists; convert to findFirst.
      params.action = 'findFirst' as any;
      params.args.where = { ...params.args.where, tenantId: ctx.tenantId };
      break;
    case 'findFirst':
    case 'findFirstOrThrow':
    case 'findMany':
    case 'count':
    case 'aggregate':
    case 'groupBy':
      params.args ??= {};
      params.args.where = { ...params.args.where, tenantId: ctx.tenantId };
      break;
    case 'create':
      params.args.data = { ...params.args.data, tenantId: ctx.tenantId };
      break;
    case 'createMany':
      params.args.data = (params.args.data as any[]).map(d => ({ ...d, tenantId: ctx.tenantId }));
      break;
    case 'update':
    case 'updateMany':
    case 'delete':
    case 'deleteMany':
    case 'upsert':
      params.args.where = { ...params.args.where, tenantId: ctx.tenantId };
      if (params.action === 'upsert' || params.action === 'update') {
        // never let tenantId be changed
        delete (params.args.data as any)?.tenantId;
        delete (params.args.update as any)?.tenantId;
      }
      break;
  }
  return next(params);
});
```

### 7.2 Cross-tenant escape hatch (super-admin only)

```ts
import { withCrossTenant } from './context/als';

await withCrossTenant({ userId, reason: 'super_admin_audit_view' }, async () => {
  // queries here run unscoped, every write goes through audit log
});
```

Calls `audit.log({ action: 'super_admin.cross_tenant_query', metadata: { reason } })` before entering the block.

### 7.3 Invariant: RSCs never call Prisma directly

All data access (read or write) flows through tRPC — either via the React Query client in client components or via `createServerCaller()` in server components. This ensures every query passes through the tRPC middleware chain (auth → tenant-context → audit) before hitting Prisma.

This is enforced via lint rule (`no-restricted-imports` for `@/server/db` outside `server/trpc/**`) added in Slice A.

---

## 8. Roles & Authorization

Two layers, both checked:

1. **`User.platformRole`** — global. `SUPER_ADMIN` always wins; `TEAM`/`MANAGER` are internal staff identifiers.
2. **`TenantUser.role`** — per-tenant. `CLIENT_OWNER`/`CLIENT_STAFF`/`TEAM`/`MANAGER`.

### 8.1 Capability matrix (Slice A)

| Capability | SUPER_ADMIN (platform) | MANAGER (TenantUser) | TEAM (TenantUser) | CLIENT_OWNER | CLIENT_STAFF |
|---|---|---|---|---|---|
| View tenant dashboard | ✓ (any tenant) | ✓ | ✓ | ✓ | ✓ |
| Invite CLIENT_STAFF | ✓ | ✓ | — | ✓ | — |
| Invite TEAM/MANAGER | ✓ | — | — | — | — |
| Toggle feature flags for tenant | ✓ | — | — | — | — |
| Create new tenant | ✓ | — | — | — | — |
| View `/admin/*` | ✓ | — | — | — | — |
| View audit log (own tenant) | ✓ | ✓ (Slice C UI) | — | ✓ (Slice C UI) | — |

### 8.2 Helpers in `@gbp/lib/auth`

```ts
requireSession(): Session                                  // throws UnauthorizedError if no session
requirePlatformRole(role: PlatformRole): Session           // throws ForbiddenError
requireTenantRole(role: TenantRole | TenantRole[]): Session // throws ForbiddenError, requires tenant context
```

These helpers are used by both tRPC procedure builders and server components.

### 8.3 tRPC procedure builders

```ts
publicProcedure                  // no auth
authedProcedure                  // requireSession
tenantProcedure                  // authedProcedure + currentTenantId is set
clientOwnerProcedure             // tenantProcedure + (tenantRole === CLIENT_OWNER || platformRole === SUPER_ADMIN)
managerProcedure                 // tenantProcedure + (platformRole === SUPER_ADMIN || platformRole === MANAGER || tenantRole === MANAGER)
superAdminProcedure              // authedProcedure + platformRole === SUPER_ADMIN
crossTenantProcedure             // superAdminProcedure + auto-wraps body in withCrossTenant({ reason })
```

Rule: a super-admin always satisfies role checks for tenant-scoped procedures, even when accessing a tenant they have no `TenantUser` row in (the cross-tenant view path from §6.2). The `isCrossTenantView` flag in the session is recorded in audit metadata; it is **not** the same as the Prisma `crossTenant` flag in §7 (which disables tenant scoping entirely). Cross-tenant view = "viewing one tenant without TenantUser membership"; Prisma crossTenant = "querying across tenants with no scoping at all".

---

## 9. Invites

### 9.1 Token

`crypto.randomUUID()` v4 (cryptographically random). Stored as `Invite.token`. Sent in URL. **Logged in audit metadata as the token's last 4 chars only**, never full.

### 9.2 Lifecycle

- Created by `tenantUser.invite` mutation (CLIENT_OWNER+ for CLIENT_STAFF; SUPER_ADMIN for TEAM/MANAGER/CLIENT_OWNER).
- Default expiry: 7 days.
- Email delivery: **stubbed in Slice A** — full token is **never** written to AuditLog (only last-4); instead, the team-management UI fetches the full invite URL via a `tenantUser.getPendingInviteLink` query gated to the inviter, the tenant's CLIENT_OWNER, or super-admin. Real email send arrives in Slice B with SendGrid.
- Resend creates a new token; old one invalidated.
- Revoke: `Invite.expiresAt = now()`.
- Accept: token validated → Supabase signup (if new) → User row → TenantUser row → `Invite.acceptedAt`/`acceptedByUserId` set.

---

## 10. Feature Flags

### 10.1 Storage

```ts
// per FeatureFlag row: tenantId, key, enabled, metadata?
```

### 10.2 Defaults by tier

```ts
// packages/lib/src/feature-flags/defaults.ts
export const DEFAULT_FLAGS_BY_TIER: Record<Tier, Record<string, boolean>> = {
  STANDARD: {
    flag_admin_audit_viewer: false,  // single placeholder flag for Slice A
  },
  FLAGSHIP: {
    flag_admin_audit_viewer: false,
  },
};
```

Real flags are added per-feature in later slices. Slice A has exactly **one** placeholder to prove the pipeline.

### 10.3 Resolver

```ts
// returns row.enabled if a row exists for (tenantId, key); else default; else false.
async function isFeatureEnabled(tenantId: string, key: string): Promise<boolean>
```

Implementation: per-request `Map<tenantId, Map<key, boolean>>` cache populated on first call (one DB query per tenant per request).

### 10.4 Tenant creation seeding

When a Tenant is created, all default flags for its tier are inserted into `FeatureFlag`. Done in the same transaction as Tenant creation.

### 10.5 Toggle UI

`/admin/tenants/[id]/flags` — list of flags with toggle switch. Each toggle calls `featureFlag.set` mutation, which writes the row and emits `audit.log({ action: 'flag.set', ... })`.

---

## 11. Audit Log

### 11.1 Two write paths

**Auto (tRPC middleware):** every successful mutation writes an AuditLog with `action = "{router}.{procedure}"`, `actorUserId = session.userId`, `tenantId = ctx.currentTenantId`, `metadata = sanitizedInput`.

**Manual (`audit.log()`):** for non-tRPC paths (auth callbacks, system jobs in later slices) and fine-grained events (login success/failure, tenant switch, super-admin cross-tenant entry, invite token issued/revoked).

```ts
// packages/lib/src/audit/log.ts
export async function log(params: {
  action: string;
  tenantId?: string | null;
  actorUserId?: string | null;
  targetType?: string;
  targetId?: string;
  metadata?: Record<string, unknown>;
  ip?: string;
  userAgent?: string;
}): Promise<void>
```

### 11.2 Sanitization

Before write, run input through pino redaction list: drop keys matching `password`, `*_token`, `authorization`, `cookie`, `secret`, `*_key`, plus a custom list including `ssn`, `dob`. Email is allowed (it's already in the User table). Invite tokens get last-4 only.

### 11.3 Reads

`/admin/audit-log` — paginated table for super-admin: filter by tenant, actor, action, date range. UI is basic in Slice A (table + filters; nothing fancy). Per-tenant audit log UI for client roles is **Slice C**.

### 11.4 Retention

No deletion in Slice A. Retention policy is a Phase 4 decision.

---

## 12. UI Shell

### 12.1 Library

shadcn/ui (zinc) installed in `apps/platform`. Components added on demand only — at minimum: `Button`, `Input`, `Form`, `Card`, `Table`, `Sheet`, `Dialog`, `Avatar`, `DropdownMenu`, `Separator`, `Switch`, `Toast`, `Sidebar` (or hand-rolled).

### 12.2 Layouts

- **`(public)/layout.tsx`** — centered card layout for login/signup/accept-invite.
- **`(authed)/layout.tsx`** — verifies session; redirects to `/login` if absent.
- **`(authed)/t/[tenantSlug]/layout.tsx`** — tenant resolution + ALS wrap; sidebar visible.
- **`(authed)/admin/layout.tsx`** — `requirePlatformRole(SUPER_ADMIN)`; admin nav visible.

### 12.3 Sidebar nav (filtered by role)

```
Tenant nav (visible inside /t/[slug]/*):
  - Dashboard                       all tenant roles
  - Settings → Team                 CLIENT_OWNER, MANAGER, SUPER_ADMIN
  - Settings → Feature Flags        SUPER_ADMIN only
  - (later slices fill in: Leads, Reviews, Reports, etc.)

Admin nav (visible inside /admin/*):
  - Tenants
  - Audit Log
  - Users                          (deferred, Slice B/C)
```

### 12.4 Page contents (Slice A)

- **`/t/[slug]/dashboard`** — `Welcome, {name}.` Card showing tenant name + tier + status + your role. Empty placeholder cards labeled "GBP Dashboard (coming Slice B)", "Lead Inbox (Slice B)", "Review Queue (Slice B/C)".
- **`/t/[slug]/settings/team`** — invite form (email + role select), member list with pending invites, copy-link button on pending invites.
- **`/t/[slug]/settings/flags`** — list of flags with switches (super-admin only; redirected otherwise).
- **`/admin/tenants`** — table of all tenants; "Create Tenant" dialog (name, slug, tier, owner email).
- **`/admin/tenants/[id]`** — detail view with editable tier/status (manual in Slice A).
- **`/admin/tenants/[id]/flags`** — same UI as `/t/[slug]/settings/flags` but accessible cross-tenant.
- **`/admin/audit-log`** — filterable table.

---

## 13. Errors & Logging

### 13.1 Typed errors

```ts
// packages/lib/src/errors/index.ts
export class UnauthorizedError extends Error  { code = 'UNAUTHORIZED'; }
export class ForbiddenError extends Error     { code = 'FORBIDDEN'; }
export class NotFoundError extends Error      { code = 'NOT_FOUND'; }
export class ValidationError extends Error    { code = 'VALIDATION'; }
export class TenantContextError extends Error { code = 'TENANT_CONTEXT'; }
export class ConflictError extends Error      { code = 'CONFLICT'; }
```

### 13.2 tRPC error formatter

Maps known errors to HTTP codes; logs full stack server-side; returns `{ code, message }` only to client. Unknown errors → generic 500 + correlation ID.

### 13.3 Logger (pino)

```ts
// packages/lib/src/logger/index.ts
import pino from 'pino';
export const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  redact: {
    paths: [
      'password', '*.password',
      'token', '*.token', 'authorization', '*.authorization',
      'cookie', '*.cookie', 'secret', '*.secret',
      '*_key', '*.api_key',
      'ssn', 'dob',
    ],
    censor: '[REDACTED]',
  },
});
```

`console.log` is banned via ESLint `no-console` rule.

### 13.4 Correlation IDs

Every request gets a `requestId` via Next.js middleware (`crypto.randomUUID()`); included in logger child + audit log metadata. Surfaced in client error messages so support can correlate.

---

## 14. Testing Strategy

### 14.1 Vitest unit tests

- `feature-flags`: tier-default fallback, row override, cache invalidation.
- `auth/role-checks`: every helper, every role combination.
- `audit/sanitize`: redaction patterns.
- `tenant-scoping/middleware` (mocked Prisma): each Prisma action class produces correct args.
- `invites`: token generation, expiry check, accept idempotency.

### 14.2 tRPC integration tests

Real Postgres via Supabase CLI (separate test schema). Reset between tests.

- Tenant isolation: two tenants seeded; query inside tenant A's context cannot read tenant B's data (across all tenant-scoped models). Both reads and writes.
- Cross-tenant escape hatch: only super-admin can enter; emits audit log.
- Invite flow end-to-end: invite → accept → membership active.
- Bootstrap super-admin: first signup gets SUPER_ADMIN; second standalone signup is rejected.
- Audit middleware: every mutation produces an AuditLog row with sanitized input.

### 14.3 Playwright

Scaffold `apps/platform/e2e/` with one smoke test (`/login` renders). Full E2E suite arrives in Slice C.

### 14.4 CI

Not in Slice A. GitHub Actions deferred until first remote push (which requires CEO approval anyway). Developer runs `pnpm verify` (an alias for `typecheck && lint && test`) manually before each commit, per `CLAUDE.md`. A husky pre-commit hook enforcing this is a Slice C polish item.

---

## 15. Local Dev Workflow

### 15.1 Required tools

- Node 20 LTS (enforced via `.nvmrc`, `engines`)
- pnpm ≥ 9
- Docker Desktop (for Supabase CLI)
- Supabase CLI ≥ 1.150

### 15.2 First-run

```bash
pnpm install
pnpm db:start           # supabase start
pnpm db:migrate         # prisma migrate dev
pnpm db:seed            # creates one super-admin (email/password from .env.local) + demo tenant
pnpm dev                # starts Next.js on :3000
```

### 15.3 `.env.example`

Includes every variable across all 3 slices, grouped:

```
# --- Slice A ---
DATABASE_URL=postgresql://postgres:postgres@127.0.0.1:54322/postgres
DIRECT_URL=postgresql://postgres:postgres@127.0.0.1:54322/postgres
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
NEXT_PUBLIC_APP_URL=http://localhost:3000
LOG_LEVEL=info
SEED_SUPER_ADMIN_EMAIL=
SEED_SUPER_ADMIN_PASSWORD=

# --- Slice B (placeholders) ---
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
STRIPE_PUBLISHABLE_KEY=
STRIPE_PRICE_STANDARD=
STRIPE_PRICE_FLAGSHIP=

GBP_OAUTH_CLIENT_ID=
GBP_OAUTH_CLIENT_SECRET=
GBP_OAUTH_REDIRECT_URL=

TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=

INNGEST_EVENT_KEY=
INNGEST_SIGNING_KEY=

ANTHROPIC_API_KEY=
SENDGRID_API_KEY=

# --- Slice C placeholders ---
LOCAL_FALCON_API_KEY=
BRIGHTLOCAL_API_KEY=
ASSEMBLYAI_API_KEY=
UPTIMEROBOT_API_KEY=
SENTRY_DSN=
```

### 15.4 Seed script

`packages/db/seed.ts`:
- Creates one Supabase auth user using `SEED_SUPER_ADMIN_EMAIL`/`PASSWORD`.
- Creates corresponding `User` row with `platformRole = SUPER_ADMIN`.
- Creates one demo tenant `acme-hvac` (tier STANDARD, status ACTIVE).
- Default feature flags inserted.
- Logs the login URL.

---

## 16. Definition of Done

Slice A is done when ALL of these are true:

- [ ] `pnpm install && pnpm db:start && pnpm db:migrate && pnpm db:seed && pnpm dev` succeeds on a fresh clone with Docker running.
- [ ] Sign up the bootstrap super-admin via the seeded credentials → land on `/admin/tenants`.
- [ ] Create a new tenant via the admin UI → tenant appears in list with default flags seeded.
- [ ] Invite a CLIENT_OWNER by email → copy invite link → open in incognito → signup → land on `/t/[slug]/dashboard`.
- [ ] As CLIENT_OWNER, invite a CLIENT_STAFF → accept → both visible in member list.
- [ ] Switch between tenants via `/select-tenant` (super-admin and a multi-tenant team user both work).
- [ ] Cross-tenant Prisma query inside tenant context throws `TenantContextError` (verified by integration test).
- [ ] Super-admin uses cross-tenant escape hatch → audit log entry confirms entry.
- [ ] Toggling a feature flag writes an audit log entry; reading the flag reflects the toggle.
- [ ] `/admin/audit-log` displays mutations performed during the session, filterable by tenant and actor.
- [ ] `pnpm typecheck && pnpm lint && pnpm test` all pass.
- [ ] Zero `console.log` in committed code; zero `any` without justification comment.

---

## 17. Open Questions / Deferred

These are intentionally **not** decided in Slice A — flagged here so they're not forgotten:

1. **Email delivery for invites** — stubbed in Slice A (audit log + copy-link). Real send arrives with SendGrid in Slice B.
2. **Password reset / email verification customization** — Supabase defaults for now; branded templates Slice C.
3. **MFA** — post-Slice C.
4. **RLS policies** — defense-in-depth pass at end of Slice C.
5. **CI / GitHub Actions** — added at first remote push (requires CEO approval).
6. **Strategist assignment workflow** — Phase 2.
7. **Subdomain routing** — Phase 4.
8. **Audit log retention** — Phase 4.

---

## 18. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Prisma middleware bypassed by direct `prisma.$queryRaw` | Medium | Critical | Lint rule banning raw SQL outside migrations; integration test that asserts cross-tenant blocked |
| AsyncLocalStorage context lost across async boundaries in RSC | Low | Critical | RSC-never-calls-Prisma-directly invariant + lint rule |
| Bootstrap super-admin race | Very low | Low | Unique email constraint + clear error |
| Supabase auth + Prisma User row drift | Medium | Medium | Auth callback creates User row in same transaction; reconciliation job in Slice C |
| Slug collision | Medium | Low | Unique constraint + retry-with-suffix on tenant create |
| Seed script run in production | Low | Critical | Seed refuses to run unless `NODE_ENV !== 'production'` |

---

*End of Slice A design.*
