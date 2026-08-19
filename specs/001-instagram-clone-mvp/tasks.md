---
description: "Task list for feature implementation"
---

# Tasks: Instagram Clone MVP

**Input**: Design documents from `/specs/001-instagram-clone-mvp/` (`spec.md`, `plan.md`) and the
approved reference docs in `Docs/` (`Architecture.md`, `Requirements.md`, `Tech-Stack.md`,
`Constraints.md`)

**Prerequisites**: `plan.md` ✓, `spec.md` ✓. `research.md`, `data-model.md`, and `contracts/` were
not generated separately — `Docs/Architecture.md` already contains the authoritative DDL and
endpoint table, so tasks below cite it directly instead of a duplicate `specs/` copy.

**Tests**: Included per user story — `Docs/INSTRUCTION.md` definition of done requires passing
tests before any phase is called done, and the constitution's Testing Standards principle makes
them non-negotiable.

**Organization**: Tasks are grouped by user story (matching `spec.md`'s P1–P5) so each story is
independently implementable, testable, and demoable.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1–US5)

## Path Conventions

Monorepo per `plan.md` § Project Structure: `client/src/`, `server/src/`, `shared/`,
`server/netlify/functions/`. Test files live beside their subject as `*.test.ts`.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Repository scaffold — nothing below this phase can start until it's done.

- [ ] T001 Create npm workspaces root `package.json` with `client/`, `server/`, `shared/` workspaces
- [ ] T002 [P] Scaffold `client/` with Vite + React 18 + TypeScript (`client/package.json`, `client/vite.config.ts`, `client/tsconfig.json`)
- [ ] T003 [P] Scaffold `server/` with Express + TypeScript (`server/package.json`, `server/tsconfig.json`, `server/src/app.ts`)
- [ ] T004 [P] Scaffold `shared/` package for zod schemas and shared types (`shared/package.json`, `shared/tsconfig.json`, `shared/src/index.ts`)
- [ ] T005 Add `serverless-http` entry point `server/netlify/functions/api.ts` wrapping the Express app from T003
- [ ] T006 Add `netlify.toml` with the `/api/* → /.netlify/functions/api` rewrite and build config per `Docs/Architecture.md` § Request flow
- [ ] T007 [P] Configure ESLint (shared root config) across `client/`, `server/`, `shared/`
- [ ] T008 [P] Configure `tsc --noEmit` typecheck scripts in each workspace, wired to a root `npm run typecheck`
- [ ] T009 [P] Configure Vitest in `server/` and `client/`, wired to a root `npm run test`
- [ ] T010 Wire root `npm run build` (client Vite build + server tsc build) and `npm run lint`, satisfying the four commands named in `Docs/INSTRUCTION.md:4`

**Checkpoint**: `npm test`, `npm run typecheck`, `npm run lint`, `npm run build` all run (even with zero tests/files yet) — this is Phase 0's exit criterion per `plan.md`.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure every user story depends on. No story work starts before this is done.

**⚠️ CRITICAL**: Blocks all of Phase 3 onward.

- [ ] T011 Configure Drizzle ORM + drizzle-kit against Neon in `server/src/db/client.ts` (pooled connection, per research decision on `@neondatabase/serverless` vs `pg.Pool` — resolve and record the choice here since no `research.md` exists)
- [ ] T012 Define the full schema in `server/src/db/schema.ts` — `users`, `posts`, `follows`, `likes` tables, columns, and all indexes exactly as specified in `Docs/Architecture.md` § Data model
- [ ] T013 Generate and apply the initial migration via `drizzle-kit` from T012
- [ ] T014 [P] Define shared zod schemas in `shared/src/schemas.ts`: register, login, update-profile, create-post — validation rules from `Docs/Constraints.md` (password complexity, username charset, 5MB/format limits)
- [ ] T015 [P] Define shared TS types in `shared/src/types.ts` mirroring the schema (User, Post, FollowEdge, Like)
- [ ] T016 Implement JWT issue/verify helpers in `server/src/lib/jwt.ts` (payload `{ sub, tv }` per `Docs/Architecture.md` § Auth flow)
- [ ] T017 Implement auth middleware in `server/src/middleware/auth.ts`: verifies JWT, checks `payload.tv === user.token_version`, rejects with 401 on mismatch
- [ ] T018 Implement a rate-limit store (decide: Postgres table vs Netlify built-in, per `plan.md`'s open decision) and wrap it as `server/src/middleware/rateLimit.ts`
- [ ] T019 Wire global error-handling middleware in `server/src/middleware/errorHandler.ts`

**Checkpoint**: Database migrated, auth middleware testable in isolation, shared schemas importable from both `client/` and `server/`.

---

## Phase 3: User Story 1 - Account and identity (Priority: P1) 🎯 MVP

**Goal**: Register, log in, log out; sessions survive reload; password change revokes every session.

**Independent Test**: Register a new account, reload while logged in, log out, confirm the stale cookie is rejected on a protected route.

**Covers**: R-01, R-02, R-03

### Tests for User Story 1

- [ ] T020 [P] [US1] Integration test: register happy path, duplicate username/email, weak password — `server/src/routes/auth.test.ts`
- [ ] T021 [P] [US1] Integration test: login happy path, bad credentials, rate-limit trip — `server/src/routes/auth.test.ts`
- [ ] T022 [P] [US1] Integration test: password change invalidates a previously issued cookie via `token_version` — `server/src/routes/auth.test.ts`

### Implementation for User Story 1

- [ ] T023 [US1] Implement `POST /api/auth/register` in `server/src/routes/auth.ts` (bcrypt hash, insert user, issue cookie) — depends on T012, T014, T016
- [ ] T024 [US1] Implement `POST /api/auth/login` in `server/src/routes/auth.ts` — depends on T023
- [ ] T025 [US1] Implement `POST /api/auth/logout` in `server/src/routes/auth.ts` (clears cookie) — depends on T017
- [ ] T026 [US1] Apply T018 rate limiting to register and login routes (5 logins/15min per IP+email, 3 registrations/hour per IP, per `Docs/Constraints.md`)
- [ ] T027 [P] [US1] Register/Login pages in `client/src/pages/Register.tsx`, `client/src/pages/Login.tsx` using T014 schemas + react-hook-form
- [ ] T028 [US1] Auth session hook/context in `client/src/lib/auth.ts` (current-user query via TanStack Query, logout action)

**Checkpoint**: User Story 1 fully functional and independently testable.

---

## Phase 4: User Story 2 - Publish a post (Priority: P2)

**Goal**: Upload an image directly to Cloudinary, publish a post with optional caption and SD/HD quality, delete own posts.

**Independent Test**: Publish a post, confirm it renders on the author's profile, delete it, confirm it's gone.

**Covers**: R-07, R-08, R-09

### Tests for User Story 2

- [ ] T029 [P] [US2] Integration test: sign endpoint returns a valid signature — `server/src/routes/uploads.test.ts`
- [ ] T030 [P] [US2] Integration test: create post happy path, and rejection when `imagePublicId` isn't a recent signed upload — `server/src/routes/posts.test.ts`
- [ ] T031 [P] [US2] Integration test: delete rejected for a non-owner, succeeds for the owner — `server/src/routes/posts.test.ts`

### Implementation for User Story 2

- [ ] T032 [US2] Implement `POST /api/uploads/sign` in `server/src/routes/uploads.ts` (Cloudinary signature, API secret server-side only) — depends on T017
- [ ] T033 [US2] Implement `POST /api/posts` in `server/src/routes/posts.ts` (validates ownership context, inserts row with `quality` enum) — depends on T012, T014, T032
- [ ] T034 [US2] Implement `DELETE /api/posts/:id` in `server/src/routes/posts.ts`, owner-only — depends on T033
- [ ] T035 [P] [US2] Cloudinary direct-upload helper in `client/src/lib/upload.ts` (request signature, multipart POST to Cloudinary, progress)
- [ ] T036 [P] [US2] Create-post UI in `client/src/pages/CreatePost.tsx` (image picker, caption, SD/HD toggle) — depends on T035
- [ ] T037 [US2] SD/HD URL-transform helper in `shared/src/cloudinary.ts` (`q_auto:eco,w_1080` / `q_auto:best,w_1920` per `Docs/Architecture.md` § Upload flow)

**Checkpoint**: User Stories 1 AND 2 both work independently.

---

## Phase 5: User Story 3 - Profiles and settings (Priority: P3)

**Goal**: View any profile (including logged out); owner can edit password/username/bio/avatar or delete the account.

**Independent Test**: Load a profile while logged out; log in as the owner and change the bio; delete the account and confirm cascade.

**Covers**: R-05, R-06

### Tests for User Story 3

- [ ] T038 [P] [US3] Integration test: `GET /api/users/:username` succeeds with **no auth cookie sent** — `server/src/routes/users.test.ts` (the guest-access case called out explicitly in `plan.md`)
- [ ] T039 [P] [US3] Integration test: `PATCH /api/users/me` partial update, and rejection when unauthenticated — `server/src/routes/users.test.ts`
- [ ] T040 [P] [US3] Integration test: `DELETE /api/users/me` cascades to posts, likes, both follow directions, and Cloudinary assets are targeted for deletion — `server/src/routes/users.test.ts`

### Implementation for User Story 3

- [ ] T041 [US3] Implement `GET /api/users/:username` in `server/src/routes/users.ts`, mounted **outside** the global auth middleware — depends on T012
- [ ] T042 [US3] Implement `PATCH /api/users/me` in `server/src/routes/users.ts` (reuses T032 sign endpoint for avatar) — depends on T017, T041
- [ ] T043 [US3] Implement `DELETE /api/users/me` in `server/src/routes/users.ts` (`ON DELETE CASCADE` does the fan-out; route deletes Cloudinary assets by `public_id` first) — depends on T042
- [ ] T044 [P] [US3] Profile page `client/src/pages/Profile.tsx` — avatar, bio, counts, post grid; renders for guests too
- [ ] T045 [P] [US3] Settings page `client/src/pages/Settings.tsx` — password/username/bio/avatar forms, delete-account confirmation flow per `Docs/User-Flow.md`

**Checkpoint**: User Stories 1–3 all work independently; guest profile access verified.

---

## Phase 6: User Story 4 - Find and follow people (Priority: P4)

**Goal**: Search users by username, follow/unfollow, view follower/following lists.

**Independent Test**: Search a known username, follow them, confirm the edge appears in both users' lists.

**Covers**: R-04, R-10, R-11

### Tests for User Story 4

- [ ] T046 [P] [US4] Integration test: search matches by username prefix, empty/short query handled — `server/src/routes/users.test.ts`
- [ ] T047 [P] [US4] Integration test: follow/unfollow, self-follow rejected, double-follow is idempotent — `server/src/routes/follows.test.ts`

### Implementation for User Story 4

- [ ] T048 [US4] Implement `GET /api/users?q=` in `server/src/routes/users.ts` using the `users_username_lower_idx` index — depends on T012
- [ ] T049 [US4] Implement `POST /api/users/:username/follow` and `DELETE .../follow` in `server/src/routes/follows.ts` — depends on T017
- [ ] T050 [US4] Implement `GET /api/users/:username/followers` and `.../following` (cursor-paginated) in `server/src/routes/follows.ts`
- [ ] T051 [P] [US4] Search UI in `client/src/components/SearchBox.tsx` (top-bar search, result list)
- [ ] T052 [P] [US4] Follow/unfollow button + followers/following list UI in `client/src/pages/Profile.tsx` (extends T044)

**Checkpoint**: User Stories 1–4 all work independently.

---

## Phase 7: User Story 5 - Feed and likes (Priority: P5)

**Goal**: Cursor-paginated feed from followed users, manual refresh prepends newer posts, like/unlike.

**Independent Test**: Follow a user with posts, load the feed, scroll to page 2, have them publish, press refresh.

**Covers**: R-12, R-13, R-14

### Tests for User Story 5

- [ ] T053 [P] [US5] Integration test: feed pagination via `?cursor=`, empty-follow-list case — `server/src/routes/feed.test.ts`
- [ ] T054 [P] [US5] Integration test: `?after=` returns only strictly newer posts — `server/src/routes/feed.test.ts`
- [ ] T055 [P] [US5] Integration test: like/unlike, double-like is idempotent — `server/src/routes/posts.test.ts`

### Implementation for User Story 5

- [ ] T056 [US5] Implement `GET /api/feed` in `server/src/routes/feed.ts` using the exact query in `Docs/Architecture.md` § Feed query (`posts_created_at_idx`, tuple cursor) — depends on T012, T049
- [ ] T057 [US5] Implement `POST /api/posts/:id/like` and `DELETE .../like` in `server/src/routes/posts.ts` — depends on T033
- [ ] T058 [P] [US5] Feed page `client/src/pages/Feed.tsx` — infinite scroll via TanStack Query, empty state pointing at search (T051)
- [ ] T059 [US5] Refresh button behavior in `client/src/pages/Feed.tsx`: `?after=` fetch, prepend without resetting scroll — depends on T058

**Checkpoint**: All five user stories independently functional — MVP complete.

---

## Phase 8: Polish & Cross-Cutting Concerns

- [ ] T060 [P] Top bar + avatar dropdown (`client/src/components/TopBar.tsx`) per `Docs/User-Flow.md` — Profile/Settings/Logout links
- [ ] T061 [P] Loading/error/empty state audit across every async surface per constitution Principle III
- [ ] T062 [P] Accessibility pass: keyboard reachability, focus indicators, alt text fallback (`caption ?? "Photo by ${username}"`), WCAG AA contrast
- [ ] T063 Performance check: confirm every list query hits an index (`EXPLAIN`), bundle size under 250KB gzipped
- [ ] T064 End-to-end smoke test covering all five user stories in one run

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies
- **Foundational (Phase 2)**: Depends on Phase 1 — blocks all user stories
- **User Stories (Phases 3–7)**: All depend on Phase 2. Sequential by priority (P1→P5) is the
  default per `Docs/INSTRUCTION.md`'s phase-gate/one-PR-per-phase workflow; parallel is possible
  only with multiple developers
- **Polish (Phase 8)**: Depends on all five user stories

### User Story Dependencies

- US1 (Account/identity): no dependency on other stories — first, since every route needs auth
- US2 (Publish a post): needs US1's auth middleware
- US3 (Profiles/settings): needs US2's posts to render on a profile
- US4 (Find/follow): independent of US2/US3 content-wise, but ordered after them per requirement grouping
- US5 (Feed/likes): needs US2 (posts to show) and US4 (follow graph to query)

### Parallel Opportunities

- T002–T004 (client/server/shared scaffolds) in parallel
- T007–T009 (lint/typecheck/test config) in parallel
- T014–T015 (shared schemas/types) in parallel
- All `[P]`-marked tests within a story in parallel
- All `[P]`-marked UI tasks within a story in parallel once their route dependency lands

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Phase 1: Setup
2. Phase 2: Foundational
3. Phase 3: User Story 1
4. **STOP and VALIDATE** independently, per `Docs/INSTRUCTION.md`'s phase-gate rule
5. Commit, get approval, continue

### Incremental Delivery

Phases 3→7 in order, each ending in a checkpoint and a phase commit, one PR at the end covering
the whole feature — per `Docs/INSTRUCTION.md:29`.

---

## Notes

- `[P]` tasks touch different files with no dependency on each other
- `[Story]` maps each task to its user story for traceability back to `spec.md`
- Tests are written first within each story and must fail before the corresponding implementation task lands
- Every requirement `R-01`–`R-14` appears in exactly one story's "Covers" line — cross-checked against `Docs/Requirements.md`
- No `research.md`/`data-model.md`/`contracts/` exist; tasks cite `Docs/Architecture.md` directly instead — flagged in the plan as a known gap, not silently patched
