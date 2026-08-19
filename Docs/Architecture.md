# Architecture

## Repository layout

```
instagram-clone/
  client/                  React + Vite SPA
    src/
  server/                  Express API
    src/
      routes/
      db/                  Drizzle schema + client
    netlify/functions/
      api.ts               serverless-http(app) entry point
  shared/                  zod schemas + TS types, imported by both client and server
  netlify.toml             build config + /api/* rewrite
```

`shared/` exists so a validation rule (e.g. password strength, username charset) is defined once
and used by both the React form and the Express route handler — never duplicated.

## Request flow

```
Browser
  │
  ├─ static asset request ──► Netlify CDN (client/ build)
  │
  └─ /api/* request ────────► Netlify rewrite ──► netlify/functions/api.ts
                                                       │
                                                serverless-http(app)
                                                       │
                                                Express router
                                                       │
                                                Drizzle ORM
                                                       │
                                                Neon (pooled connection)
```

Frontend and API are served from the same Netlify site/domain, so the session cookie is same-origin
— no CORS configuration needed.

## Data model

```sql
CREATE TABLE users (
  id             uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  username       text NOT NULL UNIQUE,   -- stored lowercase, 3-30 chars, [a-z0-9._]
  email          text NOT NULL UNIQUE,
  name           text NOT NULL,
  password_hash  text NOT NULL,
  bio            text,                    -- max 150 chars, enforced in shared/ schema
  avatar_public_id text,                  -- Cloudinary public_id, null until uploaded
  token_version  int  NOT NULL DEFAULT 0, -- bump to invalidate all existing JWTs
  created_at     timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE posts (
  id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id          uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  image_public_id  text NOT NULL,          -- Cloudinary public_id
  quality          text NOT NULL,          -- 'sd' | 'hd'
  caption          text,                   -- max 2200 chars, enforced in shared/ schema
  created_at       timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE follows (
  follower_id   uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  following_id  uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  created_at    timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (follower_id, following_id),
  CHECK (follower_id <> following_id)
);

CREATE TABLE likes (
  user_id     uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  post_id     uuid NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  created_at  timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (user_id, post_id)
);

-- Feed query: newest posts from users I follow, cursor-paginated
CREATE INDEX posts_created_at_idx  ON posts (created_at DESC, id);
CREATE INDEX posts_user_id_idx     ON posts (user_id);

-- "who do I follow" / "who follows me" lookups
CREATE INDEX follows_follower_idx  ON follows (follower_id);
CREATE INDEX follows_following_idx ON follows (following_id);

-- username search (R-04) and case-insensitive uniqueness
CREATE INDEX users_username_lower_idx ON users (lower(username));
```

`ON DELETE CASCADE` on every foreign key is what makes account deletion (R-06) a single
`DELETE FROM users WHERE id = $1` — posts, likes, and both directions of follows are removed by
Postgres, not by application code. The API layer separately deletes the corresponding Cloudinary
assets (avatar + each post image) by `public_id` before or after the row delete.

## Feed query

```sql
SELECT p.*, u.username, u.avatar_public_id
FROM posts p
JOIN users u ON u.id = p.user_id
WHERE p.user_id IN (
  SELECT following_id FROM follows WHERE follower_id = $me
)
AND (p.created_at, p.id) < ($cursor_created_at, $cursor_id)  -- omitted on first page
ORDER BY p.created_at DESC, p.id DESC
LIMIT 20;
```

The composite `(created_at, id)` cursor is used instead of `OFFSET` because offset pagination
silently duplicates or skips rows when new posts arrive between page loads — a `WHERE (created_at,
id) < (...)` tuple comparison doesn't have that failure mode, and it uses the
`posts_created_at_idx` index directly. Refresh (R-13) runs the same query with `>` instead of `<`,
ordered ascending, to fetch only what's newer than the currently-loaded top post, then prepends the
results client-side.

## Endpoints

| Method & Path | Auth | Requirement | Body / Query | Response |
|---|---|---|---|---|
| `POST /api/auth/register` | – | R-01 | `{ name, username, email, password }` | sets cookie, `{ user }` |
| `POST /api/auth/login` | – | R-02 | `{ identifier, password }` | sets cookie, `{ user }` |
| `POST /api/auth/logout` | ✓ | R-03 | – | clears cookie |
| `GET /api/users?q=` | – | R-04 | `?q=<prefix>` | `{ users: [{ username, avatar_public_id }] }` |
| `GET /api/users/:username` | – | R-05 | – | `{ user, followerCount, followingCount, posts }` |
| `PATCH /api/users/me` | ✓ | R-06 | `{ username?, bio?, password?, avatar? }` | `{ user }` |
| `DELETE /api/users/me` | ✓ | R-06 | – | 204, clears cookie |
| `POST /api/uploads/sign` | ✓ | R-07 | `{ folder }` | `{ signature, timestamp, apiKey, cloudName }` |
| `POST /api/posts` | ✓ | R-07, R-08 | `{ imagePublicId, quality, caption? }` | `{ post }` |
| `DELETE /api/posts/:id` | ✓ (owner only) | R-09 | – | 204 |
| `POST /api/users/:username/follow` | ✓ | R-10 | – | `{ following: true }` |
| `DELETE /api/users/:username/follow` | ✓ | R-10 | – | `{ following: false }` |
| `GET /api/users/:username/followers` | – | R-11 | cursor | `{ users, nextCursor }` |
| `GET /api/users/:username/following` | – | R-11 | cursor | `{ users, nextCursor }` |
| `GET /api/feed` | ✓ | R-12, R-13 | `?cursor=` or `?after=` | `{ posts, nextCursor }` |
| `POST /api/posts/:id/like` | ✓ | R-14 | – | `{ liked: true, likeCount }` |
| `DELETE /api/posts/:id/like` | ✓ | R-14 | – | `{ liked: false, likeCount }` |

Every requirement R-01 through R-14 maps to at least one row above; every row maps to a requirement
— no orphans in either direction.

## Auth flow

1. **Register/Login**: password checked with bcrypt (or hashed on register); on success the API
   issues a JWT — payload `{ sub: userId, tv: token_version }` — set as an httpOnly, Secure,
   SameSite=Lax cookie, 7-day expiry.
2. **Every authenticated request**: middleware verifies the JWT signature and expiry, then checks
   `payload.tv === user.token_version` in the database. A mismatch (post password-change, or after
   "log out everywhere") rejects the request with 401 even though the JWT itself is still validly
   signed.
3. **Password change**: `users.token_version` is incremented, which invalidates every previously
   issued cookie immediately — the revocation problem inherent to stateless JWTs is closed by this
   single column instead of a session table.
4. **Rate limiting**: login and register routes are wrapped with a limiter keyed on IP (+ email for
   login) per `Constraints.md`.

## Upload flow (image storage)

Image bytes never pass through the Express API or the Netlify Function — this both avoids the
~6MB Netlify Functions body-size limit and keeps upload latency off the 500ms API budget.

```
1. Client requests a signature:
     POST /api/uploads/sign  →  { signature, timestamp, apiKey, cloudName }
     (server signs with the Cloudinary API secret, never exposed to the client)

2. Client uploads the file directly to Cloudinary:
     POST https://api.cloudinary.com/v1_1/<cloudName>/image/upload
     (multipart form: file, signature, timestamp, apiKey)
     ← { public_id, ... }

3. Client creates the post with the returned public_id:
     POST /api/posts  { imagePublicId, quality, caption }
     → server validates ownership context, inserts the row
```

SD/HD (R-08) is not two stored files — it's one Cloudinary asset served through different URL
transforms at render time:

```
SD:  https://res.cloudinary.com/<cloud>/image/upload/q_auto:eco,w_1080/<public_id>.jpg
HD:  https://res.cloudinary.com/<cloud>/image/upload/q_auto:best,w_1920/<public_id>.jpg
```

The `quality` column on `posts` records which transform to use when rendering that post; both are
always derivable from the same stored `image_public_id`.
