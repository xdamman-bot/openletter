# OpenLetter v2 Product Specification

## 1. Product summary

OpenLetter is a simple service for writing a public email to one or more recipients.

A creator writes a letter in an email-like compose interface, sends it to recipient email addresses, and publishes it at a stable public URL. The public page lets anyone sign the letter with a passkey when available, or with email + 6-digit confirmation when passkeys are unavailable. Recipient replies by email are automatically published below the open letter, and signatories who opted in are notified.

The product should stay intentionally small: one web app, one SQLite database, one Docker container, and deployable to Vercel-compatible or Coolify-style environments.

## 2. Goals

- Make creating an open letter feel as familiar as composing an email.
- Make the public nature of the letter explicit before publishing.
- Send the initial letter by email to recipients while also publishing it online.
- Avoid leaking recipient email addresses on the public page.
- Let people sign with minimal friction:
  - passkey/WebAuthn when supported;
  - email + 6-digit confirmation as fallback.
- Let signatories optionally subscribe to recipient responses.
- Automatically publish recipient replies sent by email.
- Notify the right people at signature milestones and when responses arrive.
- Keep deployment simple: SQLite, one container, minimal moving parts.

## 3. Non-goals for v1

- Social networking features.
- Rich profile pages.
- Full moderation tooling beyond basic abuse controls.
- Multi-tenant organization workspaces.
- Complex petition/campaign analytics.
- Public exposure of recipient email addresses.
- Sending milestone updates to every signatory.

## 4. Suggested technical baseline

- Framework: Next.js + TypeScript.
- Database: SQLite.
- ORM/query layer: Drizzle ORM or Prisma with SQLite; choose the simplest option that works well in Docker and serverless-like environments.
- Auth/signing:
  - WebAuthn/passkeys for signing where supported.
  - Email one-time-code fallback.
- Email provider: Resend.
- Inbound email: Resend inbound webhook.
- Markdown rendering: server-side sanitized Markdown with a client preview.
- Runtime shape: one app container containing web routes, API routes, database migrations, and background-safe email/webhook handlers.

## 5. Core entities

### 5.1 OpenLetter

Fields:

- `id`: internal unique ID.
- `slug`: public stable URL identifier.
- `status`: `draft`, `preview`, `published`, `archived`.
- `author_email`: private; used for confirmations and milestone updates.
- `author_display_name`: optional public display name.
- `subject`: public.
- `body_markdown`: public once published.
- `body_html_sanitized`: rendered public HTML cache.
- `recipient_summary`: public safe label, e.g. `3 recipients` or manually supplied recipient names if later supported.
- `created_at`, `updated_at`, `published_at`.
- `last_milestone_sent`: null or one of `100`, `1000`, `10000`, `100000`, `1000000`.

### 5.2 Recipient

Private by default.

Fields:

- `id`.
- `letter_id`.
- `email`: private, never displayed publicly unless explicitly included in a published reply by the recipient.
- `kind`: `to` or `cc`.
- `display_name`: optional private or semi-private label.
- `created_at`.

### 5.3 Signature

Fields:

- `id`.
- `letter_id`.
- `signer_display_name`: public name.
- `signer_email`: private, nullable when passkey-only signer does not provide email.
- `email_verified_at`: nullable.
- `passkey_credential_id`: nullable.
- `notify_on_response`: boolean.
- `signed_at`.
- `confirmation_snapshot_sent_at`: nullable.

Rules:

- A person can sign once per letter per verified email or passkey credential.
- Public signature list shows display name and signing timestamp, not email address by default.

### 5.4 EmailVerificationCode

Fields:

- `id`.
- `email`.
- `letter_id`.
- `code_hash`.
- `purpose`: `sign`, `author_publish`, or future purpose.
- `expires_at`.
- `consumed_at`.
- `attempt_count`.
- `created_at`.

### 5.5 PasskeyCredential

Fields:

- `id`.
- `credential_id`.
- `public_key`.
- `counter`.
- `signer_email`: nullable.
- `signer_display_name`.
- `created_at`.
- `last_used_at`.

### 5.6 EmailMessage

Stores outbound and inbound email metadata.

Fields:

- `id`.
- `letter_id`.
- `direction`: `outbound` or `inbound`.
- `type`: `initial_letter`, `milestone`, `signature_receipt`, `recipient_response`, `response_notification`, `verification_code`.
- `resend_message_id`: nullable.
- `from_email`.
- `to_emails_private_json`: private.
- `subject`.
- `body_text`.
- `body_html_sanitized`.
- `raw_payload_json`: private; inbound webhook/debug only.
- `created_at`.

### 5.7 RecipientResponse

Fields:

- `id`.
- `letter_id`.
- `email_message_id`.
- `from_email_private`.
- `from_display_name_public`: public.
- `subject`.
- `body_text`.
- `body_html_sanitized`.
- `received_at`.
- `published_at`.

## 6. Public routes

### 6.1 `GET /`

Simple landing page explaining:

- write a public email;
- send it to recipients;
- collect signatures;
- recipient replies are published publicly.

Primary CTA: `Write an open letter` → `/new`.

### 6.2 `GET /new`

Open-letter compose page.

Layout:

- Main left column: compose interface.
- Right column: explanation and quick FAQ.

Main compose fields:

- `To`: recipient email addresses, multiple allowed.
- `Cc`: optional recipient email addresses, multiple allowed.
- `Subject`.
- `Body`: Markdown editor.
- Markdown preview toggle or side-by-side preview.
- Author email.
- Optional author display name.

Required explicit disclosure near CTA:

> This open letter will be emailed to the recipients above and published publicly online. Recipient email addresses will not be shown publicly, but the subject, letter body, signatures, and any recipient replies will be visible on the public page.

Primary CTA:

- `Preview the open letter`.

Right column content:

- How it works:
  1. Write your letter like an email.
  2. Preview the public page and outgoing email.
  3. Publish to send it to recipients and create a public URL.
  4. Anyone can sign the public letter.
  5. Recipient replies are automatically published below the letter.
- FAQ:
  - Are recipient email addresses public? No.
  - What happens when I publish? The email is sent and the public page goes live.
  - Can recipients reply? Yes, replies are published on the public page.
  - How do people sign? With passkey if available, otherwise email confirmation.
  - Can signatories get notified? Yes, if they enter an email and opt in.

### 6.3 `POST /api/letters/preview`

Creates or updates a preview letter.

Validation:

- At least one `To` recipient.
- Valid emails for `To`, optional valid emails for `Cc`.
- Non-empty subject.
- Non-empty Markdown body.
- Valid author email.

Returns:

- preview URL, e.g. `/letters/{slug}?preview=1`.

No recipient email is sent at this stage.

### 6.4 `GET /letters/{slug}?preview=1`

Preview page.

Must show:

- status badge: `Preview`.
- final URL that will be used after publishing.
- rendered public letter.
- signature module disabled or clearly marked as unavailable until published.
- preview of the outgoing email that recipients will receive.
- buttons:
  - `Back to edit`.
  - `Publish and send email`.

Copy near publish CTA:

> Publishing will send this open letter to the recipients you entered and make it publicly available at this URL. Any recipient reply by email will automatically be published here and shared with signatories who opted in.

### 6.5 `POST /api/letters/{slug}/publish`

Publishes the letter and sends the initial email to recipients.

Rules:

- Only preview/draft letters can be published.
- Publishing is idempotent: repeated calls do not send duplicate initial emails.
- Initial email is sent to `To` and `Cc` recipients, plus author copy if desired.
- Public page becomes available without `preview=1`.

### 6.6 `GET /letters/{slug}`

Public open-letter page.

Shows:

- subject.
- rendered letter body.
- public recipient summary without email addresses.
- publication date.
- signature count.
- signature list.
- sign form.
- recipient responses below the letter.

Sign form:

- If passkey/WebAuthn is available:
  - sign with passkey as primary flow;
  - optionally collect email for response notifications.
- If passkey is unavailable or the user chooses email fallback:
  - collect display name + email;
  - send 6-digit code;
  - confirm code;
  - create signature.
- In both flows:
  - allow `Notify me when there is a response` if an email is provided.

## 7. Email behavior

### 7.1 Initial letter email

Sent when creator publishes.

Recipients:

- All `To` and `Cc` email addresses.
- Author may receive a copy.

Must include:

- Subject from open letter.
- Full letter body.
- Public URL.
- Clear disclosure:

> This is an open letter that is also published at {url}. Any response to this email will automatically be published there and shared with signatories who asked to be notified.

Reply handling:

- The `Reply-To` or inbound address should encode the letter ID/slug so Resend inbound webhook can route replies to the correct letter.
- Example: `reply+{letterId}@inbound.openletter.earth`.

### 7.2 Signature confirmation email

Sent to a signer when they sign and provide an email.

Recipients:

- The signer only.

Must include:

- Confirmation that they signed.
- Full letter subject and body.
- Current list of signatories.
- Public URL.
- Notification preference status.

### 7.3 Milestone email

Triggered when signature count crosses one of these levels:

- 100
- 1,000
- 10,000
- 100,000
- 1,000,000

Recipients:

- Original letter recipients.
- Author.
- Not all signatories.

Must include:

- Current signature count.
- Public URL.
- Full list of signatories.
- Reminder that any recipient response will be published publicly.

Rules:

- Send each milestone at most once per letter.
- If a letter jumps from 99 to 1,001 signatures, send the highest newly crossed milestone or send sequential milestones according to product decision; default v1 behavior: send only the highest newly crossed milestone to avoid spam.

### 7.4 Recipient response notification

Triggered when an inbound recipient reply is published.

Recipients:

- Signatories who provided an email and opted into response notifications.
- Author may receive a copy.

Must include:

- Public URL.
- Responder public display name if available.
- Response body excerpt.
- Link to read full response.

## 8. Resend inbound webhook

### 8.1 Route

`POST /api/webhooks/resend/inbound`

### 8.2 Responsibilities

- Verify webhook signature.
- Parse inbound email.
- Identify target letter from inbound address or message headers.
- Confirm sender is one of the original recipients when possible.
- Sanitize email body.
- Create a `RecipientResponse`.
- Publish the response below the letter.
- Notify signatories who opted in.

### 8.3 Privacy and safety

- Do not publish raw email headers.
- Do not publish private recipient email addresses.
- Strip tracking pixels and unsafe HTML.
- Prefer plain text body when available; otherwise sanitize HTML aggressively.
- Store raw payload privately for debugging with retention controls.

## 9. Privacy rules

- Recipient email addresses are never shown publicly by default.
- Signer email addresses are never shown publicly by default.
- Public signature list shows display name and date.
- Inbound recipient replies are public, but private metadata is not.
- The creation and preview screens must clearly state that the letter and replies are public.

## 10. Abuse controls for v1

Minimum protections:

- Rate-limit preview creation, publish, verification-code sending, and signing.
- Limit recipients per letter initially, e.g. 20 total `To` + `Cc`.
- Limit body size.
- Require author email verification before publish, or at minimum before sending emails.
- Verify Resend webhook signatures.
- Ensure publish endpoint is idempotent.
- Sanitize Markdown and inbound email HTML.
- Add unsubscribe link for notification emails.

## 11. Deployment constraints

The application should run as one deployable unit.

Docker:

- Single Dockerfile.
- App server and API routes in one container.
- SQLite file stored in a configurable volume path, e.g. `/data/openletter.sqlite`.
- Migrations run safely on startup or via a simple command.

Environment variables:

- `DATABASE_URL` or `SQLITE_PATH`.
- `APP_URL`, e.g. `https://openletter.earth`.
- `RESEND_API_KEY`.
- `RESEND_WEBHOOK_SECRET`.
- `INBOUND_EMAIL_DOMAIN`.
- `EMAIL_FROM`.
- `WEBAUTHN_RP_ID`.
- `WEBAUTHN_ORIGIN`.

Vercel note:

- SQLite on Vercel needs a persistent SQLite-compatible provider or mounted storage alternative. The app should keep the persistence layer abstract enough to use local SQLite in Docker/Coolify and a Vercel-compatible SQLite option if deployed there.

Coolify note:

- Preferred simple deployment target: one container with persistent volume mounted at `/data`.

## 12. UX acceptance criteria

### Compose page

- User can enter multiple `To` addresses.
- User can enter optional `Cc` addresses.
- User can enter subject and Markdown body.
- User can preview Markdown before creating preview.
- Right column explains public publishing and reply behavior.
- CTA says exactly: `Preview the open letter`.

### Preview page

- Shows `Preview` status.
- Shows final URL.
- Shows rendered letter.
- Shows outgoing recipient email preview.
- Provides `Back to edit`.
- Provides `Publish and send email`.
- Clearly states publishing sends email and makes the page public.

### Public page

- Shows public letter without recipient emails.
- Lets users sign with passkey where supported.
- Provides email-code fallback.
- Lets signers provide email for response notifications.
- Shows signatures and responses.

### Email flows

- Initial publish sends the letter to recipients.
- Signature with email sends confirmation receipt with full letter and current signatories.
- Milestone thresholds notify recipients + author only.
- Recipient replies are published and notify opted-in signatories.

## 13. Initial implementation phases

### Phase 1: Repo and specification

- Create repository.
- Add this product specification.
- Decide final stack details for Next.js, SQLite, email, Markdown, and WebAuthn.

### Phase 2: Skeleton app

- Initialize Next.js + TypeScript app.
- Add SQLite schema and migrations.
- Add Dockerfile and local development instructions.

### Phase 3: Letter creation and preview

- Implement `/new` compose UI.
- Implement Markdown preview.
- Implement preview persistence.
- Implement preview page with outgoing email preview.

### Phase 4: Publish and email sending

- Integrate Resend outbound email.
- Implement idempotent publish.
- Send initial email with public URL and reply-public disclosure.

### Phase 5: Public page and signing

- Implement public letter page.
- Implement passkey signing.
- Implement email-code fallback signing.
- Implement signer confirmation email.

### Phase 6: Replies and notifications

- Implement Resend inbound webhook.
- Publish recipient responses.
- Notify opted-in signatories.
- Implement milestone notification emails.

## 14. Open product decisions

- Should author email verification be required before preview or only before publish? Recommended: before publish.
- Should recipient display names be supported in v1 or only raw private emails? Recommended: emails only in v1.
- Should signer display names be required? Recommended: yes, with email/passkey verification.
- How should duplicate signatures be handled across passkey and email fallback? Recommended: dedupe by verified email when present, otherwise by passkey credential.
- Should milestone jumps send only the highest milestone or every crossed milestone? Recommended: highest newly crossed milestone only.
- What should be the exact inbound email address format? Recommended: `reply+{letterId}@{inboundDomain}`.
