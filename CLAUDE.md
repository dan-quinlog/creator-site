# creator-site

A social-feed style site for a local business: business info on the left, a contact call-to-action on the right, and an endless-scroll project feed in the center. The owner logs in to manage posts and read contact requests. Shared rules in `../CLAUDE.md` and `~/.claude/CLAUDE.md` also apply.

## Project facts
- AWS profile: `neechan` (pass `--profile neechan` on every `ampx`/AWS command)
- AWS region: `us-east-1`
- IaC: Amplify Gen 2 (`amplify/`)
- Narrow tests: `npm run test:changed`. Full suite (promotions only): `npm run test:full`
- Hosting model: **one deployment per business.** Each new client gets a personalized copy with its own backend. No multi-tenancy in the code.

## Personalizing for a business
- Business info, branding images, and copy live in `src/config/business.ts`.
- Colors are CSS variables in `src/styles/theme.css`.
- Placeholder images: top banner, left-column image, right-column CTA image.

## Architecture
- Auth: Cognito with self-signup disabled. The owner account belongs to the `Owner` group. Visitors use guest (identity pool) access.
- Authorization is group-based, not `allow.owner()`: posts are public and contact requests are written by anonymous visitors.
- Images go to Amplify Storage under `posts/*` (guest/auth read, `Owner` write).
- Images are resized in the browser before upload (longest edge ≤ 1600px, re-encoded as WebP or JPEG), and the gallery uses fixed aspect-ratio frames with `object-fit: cover`. This way a huge upload can't break the layout.
- Email alerts: a Lambda on the ContactRequest DynamoDB stream reads `SiteSettings` and, if alerts are on, sends via SES. The sender identity must be verified in SES (`us-east-1`).

## Data model and access patterns
```
Post { id, title, body, images: string[] (max 4 S3 keys), status: draft|published,
       feed: 'MAIN', publishedAt }
  GSI feed + publishedAt DESC
  1. Public: list published posts newest first, paged (limit + nextToken)
  2. Owner: list all posts incl. drafts newest first
  3. Owner: get/update/delete by id
  auth: guest + authenticated read · group Owner full CRUD

ContactRequest { id, name, email, phone?, message, status: new|read|archived, createdAt }
  4. Public: create only (honeypot field filtered client-side + in resolver)
  5. Owner: list newest first, filter by status
  6. Owner: update status / delete
  auth: guest create · group Owner read/update/delete

SiteSettings { id: 'SITE', emailAlertsEnabled: boolean, alertEmail }
  7. Owner: get/update the single settings record
  8. Alert Lambda: read settings by id (IAM)
  auth: group Owner read/update · Lambda via resource access
```

## Build phases (one feature branch + PR into dev each)
0. Setup: Vite scaffold, strict TS, Vitest, scripts, this file.
1. Layout shell: nature theme, 3-column responsive grid, branding placeholders, feed with fake posts.
2. Backend: Amplify auth/data/storage; sandbox deploy after approval.
3. Live feed: infinite scroll (IntersectionObserver sentinel), loading/empty/end states, gallery of up to 4 images.
4. Contact modal: `<dialog>`, validation, honeypot, guest create, success state.
5. Owner tools: login, create/edit/delete posts, draft/publish, image upload with client-side resize (max 4).
6. Owner inbox: list contact requests, mark read/archived, "new" badge.
7. Email alerts: SiteSettings toggle + alert email in admin; stream-triggered Lambda sends via SES.

## Gotchas
- TypeScript is 7.x (from the Vite template).
- React StrictMode double-runs effects; guard effects that create records.
