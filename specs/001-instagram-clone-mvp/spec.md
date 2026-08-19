# Feature Specification: Instagram Clone MVP

**Feature Branch**: `001-instagram-clone-mvp`

**Created**: 2026-08-19

**Status**: Approved

**Input**: The approved documentation set in `Docs/`.

> **Source of truth**: `Docs/` is authoritative. This spec indexes it for the Spec Kit tooling and
> does not restate it. Requirements live in `Docs/Requirements.md` as `R-01`–`R-14`; the data model,
> endpoint table, auth flow, and upload flow live in `Docs/Architecture.md`; non-functional limits
> live in `Docs/Constraints.md`; screen-level flows live in `Docs/User-Flow.md`.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Account and identity (Priority: P1)

A visitor registers with name, username, email, and password, logs in, and logs out. Their session
survives a page reload and ends everywhere when they change their password.

**Why this priority**: Every other requirement is gated on an authenticated identity. Nothing else
can be built or demonstrated first.

**Independent Test**: Register a new account, reload the page while still logged in, log out, and
confirm protected routes reject the stale cookie.

**Acceptance Scenarios**:

1. **Given** a valid unused username and email, **When** the visitor registers, **Then** an account is created and a session cookie is set.
2. **Given** an existing account, **When** the user changes their password, **Then** every previously issued session is rejected.
3. **Given** 5 failed logins within 15 minutes, **When** a 6th is attempted, **Then** it is rate-limited.

Covers **R-01**, **R-02**, **R-03**.

---

### User Story 2 - Publish a post (Priority: P2)

A logged-in user selects an image, optionally writes a caption, chooses SD or HD, and publishes.
The image uploads directly to Cloudinary; the post appears on their profile. They can delete it.

**Why this priority**: Posts are the content the entire feed and profile depend on. Without them
the app has nothing to display.

**Independent Test**: Publish a post and confirm it renders on the author's own profile, then
delete it and confirm it disappears.

**Acceptance Scenarios**:

1. **Given** a JPEG under 5MB, **When** the user publishes it, **Then** the post is created and the image bytes never traverse the API server.
2. **Given** a post owned by another user, **When** deletion is attempted, **Then** it is rejected.

Covers **R-07**, **R-08**, **R-09**.

---

### User Story 3 - Profiles and settings (Priority: P3)

Any visitor — logged in or not — can view a user's profile: avatar, username, bio, follower and
following counts, and their posts. The owner can change password, username, bio, and avatar, or
delete the account entirely.

**Why this priority**: Makes published content reachable and gives users control over their own
identity. Depends on posts existing.

**Independent Test**: Load a profile while logged out, then log in as the owner and change the bio.

**Acceptance Scenarios**:

1. **Given** a logged-out visitor, **When** they open `/u/:username`, **Then** the profile renders without requiring authentication.
2. **Given** an account with posts, likes, and follows, **When** the owner deletes it, **Then** all of that data is removed.

Covers **R-05**, **R-06**.

---

### User Story 4 - Find and follow people (Priority: P4)

A user searches for another user by username, opens their profile, and follows them. Both users
can see their follower and following lists.

**Why this priority**: Without discovery a new account's feed is empty permanently — this is what
makes the feed non-trivial.

**Independent Test**: Search a known username, follow that user, and confirm the relationship
appears in both users' lists.

**Acceptance Scenarios**:

1. **Given** a username prefix, **When** the user searches, **Then** matching users are returned.
2. **Given** an existing follow, **When** the user unfollows, **Then** the edge is removed.

Covers **R-04**, **R-10**, **R-11**.

---

### User Story 5 - Feed and likes (Priority: P5)

The user opens the feed and sees posts from people they follow, newest first, paging as they
scroll. A refresh button pulls in newer posts without losing scroll position. Posts can be liked
and unliked.

**Why this priority**: The payoff that ties every earlier story together. Requires auth, posts, and
the follow graph to already work.

**Independent Test**: Follow a user who has posts, load the feed, scroll to page 2, then have that
user publish and press refresh.

**Acceptance Scenarios**:

1. **Given** a user following nobody, **When** the feed loads, **Then** an empty state pointing at search is shown rather than a blank screen.
2. **Given** a loaded feed, **When** a new post arrives and refresh is pressed, **Then** only newer posts are prepended and scroll position is preserved.

Covers **R-12**, **R-13**, **R-14**.

### Edge Cases

See `Docs/User-Flow.md` for the per-screen error and empty states. Notable boundaries:

- A user follows nobody — feed empty state (`Docs/User-Flow.md` § Feed, item 3).
- An image at exactly the 5MB limit, and an unsupported format (`Docs/Constraints.md` § Media).
- A signed Cloudinary upload that completes but whose `POST /api/posts` never fires (orphan asset).
- Two posts sharing an identical `created_at` — cursor tie-break by `id`.
- A guest hitting an authenticated-only route.

## Requirements *(mandatory)*

### Functional Requirements

The 14 functional requirements are defined in `Docs/Requirements.md` as **R-01** through **R-14**
and are not duplicated here. `Docs/Architecture.md` § Endpoints maps every one of them to a
concrete endpoint, with no orphans in either direction.

### Key Entities

Defined with full DDL in `Docs/Architecture.md` § Data model:

- **users** — identity, credentials, profile fields, and `token_version` for session revocation
- **posts** — one image reference plus optional caption, owned by a user
- **follows** — directed edge between two users, composite primary key
- **likes** — user-to-post edge, composite primary key

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Normal API requests respond within 500ms under expected load, excluding image upload and Cloudinary time (`Docs/Constraints.md` § Performance).
- **SC-002**: The system supports 100 concurrent users and 100 posts per day without degradation.
- **SC-003**: A new user can register, find another user, follow them, and see their post in the feed without external help.
- **SC-004**: All 14 requirements are demonstrable end-to-end, each with passing tests.

## Assumptions

- Deployment targets free tiers only: Netlify, Neon, Cloudinary (`Docs/Constraints.md` § Deployment).
- All profiles are public; there is no private-account or follow-approval flow.
- Account deletion is a hard delete and is not recoverable.
- Google sign-in, email verification, comments, filters, text overlay, global feed, WebSocket
  auto-refresh, direct messages, and stories are explicitly deferred to the Bonus block.
- Image editing in the MVP is limited to an SD/HD quality toggle applied as a Cloudinary URL
  transform — no client-side canvas editing.
