# Tech Stack

## Repository

- **Layout**: monorepo — `client/` (React app), `server/` (Express API), `shared/` (zod schemas
  and TypeScript types imported by both)

## Frontend (`client/`)

- **React 18** + **TypeScript** — UI
- **Vite** — build tool and dev server
- **React Router** — client-side routing
- **TanStack Query** — server state: caching, refetch, loading/error states, feed refresh
- **Tailwind CSS** — styling
- **react-hook-form** + **zod** (via `shared/`) — forms and validation

## Backend (`server/`)

- **Node.js** + **TypeScript** — runtime
- **Express** — HTTP API framework
- **serverless-http** — wraps the Express app for Netlify Functions
- **Drizzle ORM** + **drizzle-kit** — schema, queries, migrations
- **jsonwebtoken** — session cookies
- **bcrypt** — password hashing
- **zod** (via `shared/`) — request validation, shared with the frontend

## Data & Media

- **Database**: Neon (PostgreSQL), pooled connection string (required for serverless functions)
- **Media storage**: Cloudinary — images uploaded directly from the browser via a signed upload;
  API stores only `image_public_id`

## Linting & Type Checking

- **ESLint** — linting, client and server, shared config at the repo root
- **tsc** (`--noEmit`) — typechecking, client and server

## Testing

- **Vitest** — unit and integration tests, client and server
- **supertest** — API integration tests against the Express app
- Integration tests run against a dedicated Neon branch, migrated and seeded per run

## Deployment

- **Netlify** — hosts both the static `client/` build and the Express API as a Netlify Function
  (`netlify/functions/api.ts`), rewritten from `/api/*`. Single domain, no CORS.
