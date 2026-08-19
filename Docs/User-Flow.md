# User Flow

## Guest (logged out)

1. Guest lands on the app. Unauthenticated visitors can view a public profile at `/u/:username`
   (posts, bio, follower/following counts) but cannot like, follow, post, or see the feed.
2. Guest can navigate to **Register** or **Log in**.

## Register

1. Guest fills in name, username, email, password.
2. On success: account is created, session cookie is set, user lands on the feed.
3. On failure (duplicate username/email, weak password, rate-limited): inline field errors, form
   stays filled in.

## Log in

1. User enters email/username + password.
2. On success: session cookie is set, user lands on the feed.
3. On failure (bad credentials, rate-limited): a single inline error, no field-level detail (avoid
   leaking which field was wrong).

## Feed (main page, post-login)

1. App opens on the feed: posts from followed users, newest first, paginated by cursor
   (infinite scroll).
2. Top bar shows: search input (find users by username), a **refresh** button, a **+** button to
   create a post, and the user's avatar/username on the upper right.
3. If the user follows no one, the feed shows an empty state pointing at search instead of a blank
   screen.
4. Refresh fetches only posts newer than the currently-loaded newest post and prepends them,
   without resetting scroll position.
5. Clicking the avatar opens a dropdown: avatar + username, **Profile**, **Settings**, **Logout**.

## Search

1. User types in the search box; results list usernames matching the query.
2. Clicking a result opens that user's profile.

## Create a post

1. User clicks **+**, selects an image, optionally adds a caption, and picks SD or HD quality.
2. Image uploads directly to Cloudinary (progress shown); on completion the post is created and
   appears at the top of the user's profile and in followers' feeds.
3. On upload failure (too large, wrong format, network error): inline error, user can retry.

## Profile (own or another user's)

1. Shows avatar, username, bio, follower/following counts, and a grid of the user's posts.
2. On another user's profile: a **Follow**/**Unfollow** button.
3. On the user's own profile: posts can be deleted (with a confirmation prompt); no delete/follow
   controls appear on your own page.

## Post interaction

1. Any post (in feed or on a profile) can be liked/unliked by a logged-in user; the like count
   updates immediately.

## Settings

1. User can change password, change username, edit bio, add/change profile photo.
2. User can delete their account, gated behind a confirmation prompt. Deletion is permanent and
   immediately logs the user out.
