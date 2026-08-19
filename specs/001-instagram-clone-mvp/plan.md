# Implementation Plan: Instagram Clone MVP

**Branch**: `001-instagram-clone-mvp` | **Date**: 2026-08-19 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-instagram-clone-mvp/spec.md`, which indexes the
approved documentation set in `Docs/`.

## Summary

Build the Instagram clone MVP defined by `R-01`–`R-14` in `Docs/Requirements.md`: registration and
session auth, image posts published through a direct-to-Cloudinary signed upload, public profiles
with settings, username search and a follow graph, and a cursor-paginated feed with likes.

The technical approach is already settled in `Docs/Architecture.md`: a monorepo whose Express API
runs as a single Netlify Function via `serverless-http`, Drizzle over Neon Postgres, and a React +
Vite SPA served from the same Netlify domain so the JWT session cookie stays same-origin.

## Technical Context

**Language/Version**: TypeScript, Node.js (LTS), React 18

**Primary Dependencies**: Express, serverless-http, Drizzle ORM + drizzle-kit, jsonwebtoken,
bcrypt, zod, React Router, TanStack Query, Tailwind CSS, react-hook-form

**Storage**: Neon (PostgreSQL) via a pooled connection; media in Cloudinary, with only
`image_public_id` persisted in Postgres

**Testing**: Vitest (unit + integration), supertest against the Express app

**Target Platform**: Web — modern desktop and mobile browsers, responsive; no native app

**Project Type**: Web application (SPA + serverless API) in a single monorepo

**Performance Goals**: Normal API requests under 500ms excluding image upload and Cloudinary time

**Constraints**: Free-tier services only (Netlify, Neon, Cloudinary); images under 5MB, JPEG/PNG/
WebP only; image bytes must never traverse the API server

**Scale/Scope**: 100 concurrent users, 100 posts/day; 14 requirements across 6 delivery phases

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

**Status: NOT APPLICABLE — no constitution ratified.**

`.specify/memory/constitution.md` is still the unfilled Spec Kit template; no principles have been
adopted. There are therefore no gates to evaluate and none can fail. Project standards are instead
sourced from the approved docs and are binding for this feature:

- `Docs/Constraints.md` — security floor (bcrypt, cookie flags, rate limits, authorization),
  performance budget, media limits, deployment restrictions
- `Docs/Tech-Stack.md` — the fixed stack and tooling
- `Docs/INSTRUCTION.md` — process: guard skills, phase gating, definition of done

If a constitution is ratified later, this section must be re-evaluated before further phases.

## Project Structure

### Documentation (this feature)

```text
specs/001-instagram-clone-mvp/
├── plan.md              # This file
├── spec.md              # Indexes Docs/ — the source of truth
├── checklists/          # Requirements-quality review artifacts
└── tasks.md             # Created by /speckit-tasks, not by /speckit-plan
```

`research.md`, `data-model.md`, and `contracts/` are intentionally not duplicated here — the data
model and endpoint contract already exist in `Docs/Architecture.md`, and copying them into `specs/`
would reintroduce the drift that the documentation pass removed. Open technical decisions are
tracked under **Unresolved decisions** below.

### Source Code (repository root)

```text
client/                    React + Vite SPA  →  Netlify CDN
  src/
    components/
    pages/
    lib/                   query client, api fetch wrapper

server/                    Express API
  src/
    routes/                auth, users, posts, feed, uploads
    middleware/            auth (JWT + token_version), rate limiting
    db/                    Drizzle schema, migrations, client
  netlify/functions/
    api.ts                 serverless-http(app) entry point

shared/                    zod schemas + TS types, imported by client and server

netlify.toml               build config + /api/* → /.netlify/functions/api rewrite
```

**Structure Decision**: Monorepo with npm workspaces, exactly as specified in
`Docs/Architecture.md` § Repository layout. `shared/` exists so each validation rule (password
strength, username charset, caption length) is defined once and consumed by both the React form and
the Express handler, rather than duplicated and allowed to drift.

## Unresolved decisions

These must be settled during Phase 0 implementation; none of them change the contract in
`Docs/Architecture.md`.

1. **Test database strategy** — `Docs/Tech-Stack.md` promises a Neon branch "migrated and seeded
   per run", which requires the Neon API per test run. Cheaper alternative: one long-lived `test`
   branch with truncation between tests.
2. **Neon driver** — `@neondatabase/serverless` (HTTP, designed for functions) vs `pg.Pool`.
3. **Rate-limit storage** — in-memory counters do not survive across function instances, so the
   limits in `Docs/Constraints.md` need a shared store (a Postgres table, or Netlify's built-in
   rate limiting).
4. **Cloudinary signed-upload details** — which fields are signed, the folder convention, and how
   an orphaned upload (signed and uploaded, but `POST /api/posts` never called) is reclaimed.

## Delivery phases

Per `Docs/INSTRUCTION.md`: stop at the end of each phase, summarize, wait for approval; one commit
per phase, one PR at the end. Each phase depends only on earlier ones.

| Phase | Requirements | Scope |
|---|---|---|
| **0. Scaffold** | – | Monorepo + workspaces, Vite, Express + serverless-http, Drizzle, Vitest, ESLint, tsc, `netlify.toml`; all four tables in one migration |
| **1. Auth** | R-01, R-02, R-03 | bcrypt, JWT cookie, `token_version` middleware, rate limiting |
| **2. Uploads & Posts** | R-07, R-08, R-09 | Sign endpoint, direct upload, create with SD/HD, owner-only delete |
| **3. Profiles & Settings** | R-05, R-06 | Profile page incl. **guest access**, avatar upload, settings, account deletion |
| **4. Social graph** | R-04, R-10, R-11 | Username search, follow/unfollow, follower/following lists |
| **5. Feed & Likes** | R-12, R-13, R-14 | Cursor pagination, refresh-prepends-newer, like/unlike |

**Phase 0 exit criteria**: `npm test`, `npm run typecheck`, `npm run lint`, and `npm run build`
must all actually run and produce pasteable output. `Docs/INSTRUCTION.md` names these four commands
and makes typecheck + tests the definition of done for every phase, so Phase 1 cannot satisfy its
own DoD until Phase 0 establishes them.

**Phase 3 must include a logged-out test**: `GET /u/:username` is the only unauthenticated route
(`Docs/Constraints.md`, `Docs/User-Flow.md`). A global auth middleware silently breaks it.

## Complexity Tracking

No constitution gates exist, so there are no violations to justify.
