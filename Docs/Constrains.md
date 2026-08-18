# Project Constraints

## Scope

- implement only the feature/requirments asked in the Requirments.md

## Users & Scale

- users must be registerd to post and make actions in the app.
- Guest can see public profiles + posts.
- Only registered users can create posts, follow users, and like.
- it's an MVP project

## Performance

- Users refresh the feed manually to load new posts.
- The application should support up to 100 concurrent users.
- can handle posts up to 100 daily
- upload speed is based the internet of the user.
- user can refresh to get the new feed
- Normal page interactions should feel responsive.

## Authentication & Security

- password have to be more more than 8 char (from 8 to 50).
- password have to contain at least 1 small character, 1 capital characer, 1 number and 1 special character.
- Passwords must never be stored in plain text.
- Passwords must be securely hashed before storage.
- no one can edit or post on the user profile excpet him self.

## Database

- username must be unique.
- Emails must be unique.

## Media

- to upload images the image must be less than 5mb.

## Compatibility

- the project will be web based only not a mobile app.
- must be compatible for all web browser (windows and mobiles).
- Normal API requests should respond within 500 ms under expected load, excluding image upload time and third-party service delays.

## Deployment

- The project should use free servecis.

## Out of Scope

- implement only the feature/requirments asked in the Requirments.md not more or less than that.