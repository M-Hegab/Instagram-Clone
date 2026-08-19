# Project Constraints

## Scope

- Implement only the features in `Requirements.md`. MVP items only, unless a Bonus item is
  explicitly pulled in.

## Users & Scale

- Users must be registered to post, follow, or like.
- Guests can view public profiles and posts (read-only, logged out).
- Only registered users can create posts, follow users, and like posts.
- This is an MVP: correctness and clarity over scale.

## Performance

- Normal API requests respond within 500ms under expected load, excluding image upload time and
  third-party (Cloudinary) delays.
- Users refresh the feed manually to load new posts; no background polling.
- Supports up to 100 concurrent users and up to 100 posts per day.
- Upload speed depends on the user's own connection — no server-side upload speed guarantee.
- Normal page interactions should feel responsive.

## Authentication & Security

- Password: 8–50 characters, must contain at least one lowercase letter, one uppercase letter,
  one number, and one special character.
- Passwords are never stored in plain text — hashed with bcrypt before storage.
- Session token: JWT in an httpOnly, Secure, SameSite=Lax cookie, 7-day expiry.
- Password change or "log out everywhere" invalidates all existing sessions immediately
  (`users.token_version`).
- Login: rate-limited to 5 attempts per 15 minutes per IP+email.
- Registration: rate-limited to 3 attempts per hour per IP.
- No one can edit or post to a profile other than their own.

## Database

- Username is unique (case-insensitive).
- Email is unique.
- Deleting an account is a hard delete: cascades to that user's posts, likes, and follow edges in
  both directions.

## Media

- Uploaded images must be under 5MB.
- Accepted formats: JPEG, PNG, WebP.
- Images upload directly from the browser to Cloudinary (signed upload); image bytes never pass
  through the API server.

## Compatibility

- Web-based only, no native mobile app.
- Must work in modern browsers on both desktop and mobile (responsive layout).

## Deployment

- Must run on free-tier services: Netlify (frontend + API functions), Neon (database), Cloudinary
  (media).

## Out of Scope

- Anything not listed in `Requirements.md` MVP section. Bonus items are explicitly deferred.
