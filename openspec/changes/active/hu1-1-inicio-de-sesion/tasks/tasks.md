# Tasks: Inicio de sesion

## Review Workload Forecast

| Field | Value |
|-------|-------|
| Estimated changed lines | 600-900 |
| 400-line budget risk | High |
| Chained PRs recommended | Yes |
| Suggested split | PR 1: Bootstrap+Infra → PR 2: Auth+UI → PR 3: Routes+Tests |
| Delivery strategy | ask-on-risk |
| Chain strategy | pending |

Decision needed before apply: Yes
Chained PRs recommended: Yes
Chain strategy: pending
400-line budget risk: High

### Suggested Work Units

| Unit | Goal | Likely PR | Notes |
|------|------|-----------|-------|
| 1 | Bootstrap, Docker, Prisma, env | PR 1 | Base: feature/hu1-inicio-de-sesion |
| 2 | Auth lib, login UI, server actions | PR 2 | Depends on PR 1 |
| 3 | Middleware, route protection, tests | PR 3 | Depends on PR 2 |

## Phase A: Bootstrap

- [ ] A.1 Initialize Next.js 15 with App Router, TypeScript, Tailwind CSS
- [ ] A.2 Configure `tsconfig.json` strict mode and `@/*` path aliases
- [ ] A.3 Initialize Shadcn UI with Radix primitives and `components.json`
- [ ] A.4 Install runtime deps: `prisma`, `@prisma/client`, `iron-session`, `bcryptjs`
- [ ] A.5 Install dev deps: `jest`, `@testing-library/react`, `@testing-library/jest-dom`, `ts-node`

## Phase B: Infrastructure

- [ ] B.1 Create `docker-compose.yml` with Postgres on port 5432
- [ ] B.2 Define `prisma/schema.prisma` with `User` model (id, email, passwordHash, role, timestamps)
- [ ] B.3 Run `prisma migrate dev --name init` and generate client
- [ ] B.4 Create `prisma/seed.ts` to insert superadmin `admin@admin.com` with hashed password
- [ ] B.5 Add `.env.local` template with `DATABASE_URL`, `IRON_SESSION_PASSWORD`, `SESSION_COOKIE_NAME`

## Phase C: Auth Core

- [ ] C.1 Create `lib/password.ts` with `hashPassword()` and `verifyPassword()` using bcryptjs
- [ ] C.2 Create `lib/session.ts` with iron-session config and `SessionData` interface
- [ ] C.3 Create `lib/auth.ts` with `loginUser(email, password)` and `logoutUser()` helpers
- [ ] C.4 Create `lib/prisma.ts` singleton for PrismaClient instance

## Phase D: Login UI

- [ ] D.1 Create `app/login/page.tsx` with email/password form using Shadcn Input and Button
- [ ] D.2 Create `app/login/actions.ts` Server Action to validate credentials and create iron-session
- [ ] D.3 Add form error state and display "Credenciales incorrectas" on auth failure
- [ ] D.4 Add redirect to `/dashboard` on successful login

## Phase E: Route Protection

- [ ] E.1 Create `middleware.ts` to check session cookie and redirect unauthenticated to `/login`
- [ ] E.2 Create `app/dashboard/layout.tsx` with session validation and logout button
- [ ] E.3 Create `app/dashboard/page.tsx` as protected landing page
- [ ] E.4 Create `app/logout/actions.ts` Server Action to destroy session and redirect to `/login`

## Phase F: Tests (TDD)

- [ ] F.1 Configure Jest with `jest.config.ts`, `jest.setup.ts`, and Next.js transform
- [ ] F.2 Write unit test for `lib/password.ts` (hash/verify roundtrip)
- [ ] F.3 Write unit test for `lib/auth.ts` (login success/failure scenarios)
- [ ] F.4 Write integration test for login Server Action (valid/invalid credentials)
- [ ] F.5 Write integration test for middleware (authenticated vs unauthenticated access)
