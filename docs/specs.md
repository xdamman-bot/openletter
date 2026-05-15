# OpenLetter v2 Product Specification

## 1. Product summary

OpenLetter is a simple service for writing a public email to one or more recipients.

A creator writes a letter in an email-like compose interface, can share a clearly marked draft for private feedback before publishing, then sends it to named recipient email addresses and publishes it at a stable public URL. The public page lets anyone sign the letter with a display name + email + 6-digit confirmation. Recipient replies by email are routed back to the full thread with a link to preview, redact, and publish the response; anyone in the thread can publish after previewing.

The product should stay intentionally small for v1: one web app, one SQLite database, one Docker container, and deployable to Coolify-style environments. Vercel support is out of scope for v1.

## 2. Goals

- Make creating an open letter feel as familiar as composing an email.
- Make the public nature of the letter explicit before publishing.
- Send the initial letter by email to recipients while also publishing it online.
- Avoid leaking recipient email addresses on the public page.
- Let people sign with minimal friction using display name + email + 6-digit confirmation.
- Let signatories optionally subscribe to recipient responses.
- Let people in the recipient email thread preview, redact, and publish recipient replies received by email.
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
- ORM/query layer: Drizzle ORM or Prisma with SQLite; choose the simplest option that works well in a single Docker container.
- Auth/signing: email one-time-code confirmation for v1. Passkeys/WebAuthn are explicitly deferred.
- Email provider: Resend.
- Inbound email: Resend inbound webhook.
- Markdown rendering: server-side sanitized Markdown with a client preview.
- Runtime shape: one app container containing web routes, API routes, database migrations, and background-safe email/webhook handlers.

## 5. Core entities

### 5.1 OpenLetter

Fields:

- `id`: internal unique ID.
- `slug`: public stable URL identifier.
- `status`: `draft`, `published`, `archived`.
- `author_email`: private; used for confirmations, cc on outbound threads, and milestone updates.
- `author_display_name`: required public display name for v1.
- `subject`: public.
- `body_markdown`: public once published.
- `body_html_sanitized`: rendered public HTML cache.
- `recipient_summary`: public safe label based on recipient display names only, e.g. `Jane Doe, ACME Support, and 2 others`; never includes email addresses.
- `created_at`, `updated_at`, `published_at`.
- `last_milestone_sent`: null or one of `100`, `1000`, `10000`, `100000`, `1000000`.

### 5.2 Recipient

Private by default.

Fields:

- `id`.
- `letter_id`.
- `email`: private, never displayed publicly unless explicitly included in a published reply by the recipient.
- `kind`: `to` or `cc`.
- `display_name`: required public-safe name/label used in logs and recipient summaries.
- `created_at`.

### 5.3 Signature

Fields:

- `id`.
- `letter_id`.
- `signer_display_name`: public name.
- `signer_email`: private, nullable after confirmation if the signer did not opt into response notifications.
- `signer_email_hash`: salted hash used for dedupe when raw email is deleted.
- `email_verified_at`: nullable.
- `notify_on_response`: boolean.
- `signed_at`.
- `confirmation_snapshot_sent_at`: nullable.

Rules:

- A person can sign once per letter per verified email hash.
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

### 5.5 EmailMessage

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

### 5.6 RecipientResponse

Fields:

- `id`.
- `letter_id`.
- `email_message_id`.
- `from_email_private`.
- `from_display_name_public`: public.
- `subject`.
- `body_text_original`: private original extracted text.
- `body_text_redacted`: public text after publisher redaction.
- `body_html_sanitized`: public sanitized/redacted HTML.
- `received_at`.
- `preview_token_hash`: token allowing anyone in the email thread to open the preview/redaction page.
- `previewed_at`: nullable.
- `published_at`.

### 5.7 ActivityLog

Public-safe event log for each letter.

Fields:

- `id`.
- `letter_id`.
- `event_type`: `draft_created`, `published`, `initial_email_sent`, `milestone_email_sent`, `response_received`, `response_published`, `signature_created`.
- `public_text`: pre-rendered public-safe log text; never includes email addresses.
- `private_metadata_json`: private structured details for debugging/audit.
- `created_at`.

Rules:

- The public page shows a compact chronological log.
- Log recipient activity by display name, not email address.
- Include delivery-related events such as initial email sent, milestone notification sent, response received, and response published.

### 5.8 EmailSubscription

Tracks response-notification subscriptions independently from signatures.

Fields:

- `id`.
- `email`: private.
- `email_hash`: salted hash for lookup/dedupe.
- `letter_id`.
- `active`: boolean.
- `created_at`, `unsubscribed_at`.

Rules:

- One-click unsubscribe disables the subscription for that letter.
- The unsubscribe page also shows other active subscriptions for the same email, if any.
- If there are no remaining active subscriptions for that email, remove the raw email address entirely from the database and tell the user it was removed; retain only salted hashes needed for dedupe/audit.

## 6. Public routes

### 6.1 `GET /`

Simple landing page explaining:

- write a public email;
- send it to recipients;
- collect signatures;
- recipient replies can be previewed, redacted, and published publicly by people in the email thread.

Primary CTA: `Write an open letter` → `/new`.

### 6.2 `GET /new`

Open-letter compose page.

Layout:

- Main left column: compose interface.
- Right column: explanation and quick FAQ.

Main compose fields:

- `To`: named recipient rows (`name`, `email`), multiple allowed.
- `Cc`: optional named recipient rows (`name`, `email`), multiple allowed.
- `Subject`.
- `Body`: Markdown editor.
- Markdown preview toggle or side-by-side preview.
- Author email.
- Author display name.

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
  5. Recipient replies can be previewed, redacted, and published by anyone in the email thread.
- FAQ:
  - Are recipient email addresses public? No.
  - What happens when I publish? The email is sent and the public page goes live.
  - Can recipients reply? Yes. Replies are emailed to the thread with a publish button; anyone in the thread can preview, redact, and publish them.
  - How do people sign? With display name, email, and a 6-digit confirmation code.
  - Can signatories get notified? Yes, if they enter an email and opt in.

### 6.3 `POST /api/letters/preview`

Creates or updates a preview letter.

Validation:

- At least one `To` recipient.
- Valid names and emails for `To`, optional valid names and emails for `Cc`.
- Non-empty subject.
- Non-empty Markdown body.
- Valid author email.

Returns:

- draft preview URL, e.g. `/letters/{slug}?draft=1&token={shareToken}`.

No recipient email is sent at this stage. The draft URL can be shared for feedback before publishing, must be clearly marked as a draft, and must not allow signing.

### 6.4 `GET /letters/{slug}?draft=1&token={shareToken}`

Draft preview page for creator review and share-before-publish feedback.

Must show:

- status badge: `Draft — not published`.
- final URL that will be used after publishing.
- rendered public letter.
- signature module hidden or disabled with clear copy that people cannot sign drafts.
- preview of the outgoing email that recipients will receive.
- buttons:
  - `Back to edit`.
  - `Publish and send email`.

Copy near publish CTA:

> Publishing will send this open letter to the recipients you entered and make it publicly available at this URL. Any recipient reply by email can be previewed, redacted, and published here by people in the email thread, then shared with signatories who opted in.

### 6.5 `POST /api/letters/{slug}/publish`

Publishes the letter and sends the initial email to recipients.

Rules:

- Only preview/draft letters can be published.
- Publishing is idempotent: repeated calls do not send duplicate initial emails.
- Initial email is sent to `To` and `Cc` recipients, and the author is always cc'ed so they stay in the email thread.
- Public page becomes available without the draft token.

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
- mini activity log with public-safe events such as `Sent to Jane Doe and ACME Support by email on {datetime}`, `100 signatures notification sent to Jane Doe and ACME Support on {datetime}`, `Response received on {datetime}`, and `Response published on {datetime}`.

Sign form:

- Collect display name + email.
- Send a 6-digit code.
- Confirm code before creating the public signature.
- Allow `Notify me when there is a response`.
- If the signer does not opt into response notifications, delete the raw email address after confirmation and keep only a salted hash for dedupe.

## 7. Email behavior

### 7.1 Initial letter email

Sent when creator publishes.

Recipients:

- All `To` and `Cc` email addresses.
- Author is cc'ed.

Must include:

- Subject from open letter.
- Full letter body.
- Public URL.
- Clear disclosure:

> This is an open letter that is also published at {url}. Any response to this email can be published there after someone in this email thread clicks the publish link, previews the response, and confirms. You will be able to redact private information before publishing.

Reply handling:

- The `Reply-To` or inbound address should encode the letter ID/slug so Resend inbound webhook can route replies to the correct letter.
- Example: `reply+{letterId}@inbound.openletter.earth`.

### 7.2 Signature code and confirmation emails

A 6-digit code is sent before signing. After the code is verified and the signature is created, a confirmation email is sent to the signer.

Recipients:

- The signer only.

Must include:

- Confirmation that they signed.
- Full letter subject and body.
- Current list of signatories.
- Public URL.
- Notification preference status.

Retention rule:

- If `notify_on_response` is false, remove the raw signer email after sending confirmation and keep only a salted hash for duplicate-signature prevention.

### 7.3 Milestone email

Triggered when signature count crosses one of these levels:

- 100
- 1,000
- 10,000
- 100,000
- 1,000,000

Recipients:

- Original letter recipients.
- Author cc'ed.
- Not all signatories.

Must include:

- Current signature count.
- Public URL.
- Full list of signatories.
- Reminder that any recipient response can be previewed, redacted, and published publicly by people in the email thread.

Rules:

- Send each milestone at most once per letter.
- If a letter jumps from 99 to 1,001 signatures, send the highest newly crossed milestone or send sequential milestones according to product decision; default v1 behavior: send only the highest newly crossed milestone to avoid spam.

### 7.4 Recipient response preview, publication, and notification

Triggered when an inbound recipient reply is received from a valid thread participant.

Thread recipients before publication:

- Reply to everyone in the original thread and always cc the author, even if the inbound email omitted them.

Must include before publication:

- A button/link to preview and publish the response.
- Clear copy that publication will first show a preview.
- Clear copy that the publisher can redact private information or PII before publishing.

Recipients after publication:

- Signatories who provided an email and opted into response notifications.

Must include after publication:

- Public URL.
- Responder public display name if available.
- Response body excerpt.
- Link to read full response.
- One-click unsubscribe link.

## 8. Resend inbound webhook

### 8.1 Route

`POST /api/webhooks/resend/inbound`

### 8.2 Responsibilities

- Verify webhook signature.
- Parse inbound email.
- Identify target letter from inbound address or message headers.
- Ignore automatic responses such as auto-replies, bounces, out-of-office messages, delivery status notifications, and mail with standard auto-submitted/list/bulk headers.
- Confirm sender is one of the original recipients or the author/thread participants; if not, ignore the email.
- Sanitize email body.
- Create a `RecipientResponse` in an unpublished preview state.
- Reply to the email thread with a publish-preview link, always cc'ing the author.
- Publish the response only after a thread participant previews, optionally redacts, and confirms publication.
- Notify signatories who opted in after publication.

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
- Add one-click unsubscribe link for notification emails.

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
Coolify note:

- Preferred simple deployment target: one container with persistent volume mounted at `/data`.

## 12. UX acceptance criteria

### Compose page

- User can enter multiple named `To` recipients with name + email.
- User can enter optional named `Cc` recipients with name + email.
- User can enter subject and Markdown body.
- User can preview Markdown before creating preview.
- Right column explains public publishing and reply behavior.
- CTA says exactly: `Preview the open letter`.

### Preview page

- Shows `Draft — not published` status.
- Shows final URL.
- Shows rendered letter.
- Shows outgoing recipient email preview.
- Provides `Back to edit`.
- Provides `Publish and send email`.
- Clearly states publishing sends email and makes the page public.

### Public page

- Shows public letter without recipient emails.
- Lets users sign with display name, email, and 6-digit code.
- Lets signers provide email for response notifications.
- Shows signatures and responses.
- Shows a public-safe mini activity log.

### Email flows

- Initial publish sends the letter to recipients.
- Signature with email sends confirmation receipt with full letter and current signatories.
- Milestone thresholds notify recipients + author only.
- Recipient replies are previewed/redacted/published by thread participants, then notify opted-in signatories.

## 13. Initial implementation phases

### Phase 1: Repo and specification

- Create repository.
- Add this product specification.
- Decide final stack details for Next.js, SQLite, email, and Markdown.

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
- Implement email-code signing.
- Implement signer confirmation email.

### Phase 6: Replies and notifications

- Implement Resend inbound webhook.
- Implement response preview/redaction/publish flow.
- Notify opted-in signatories.
- Implement milestone notification emails.

## 14. Nostr integration option

Nostr can strengthen censorship resistance, but it should complement the v1 database rather than replace it.

Potential mapping:

- The open letter can be published as a Nostr long-form content event (`kind 30023`) with title/summary/tags and canonical URL.
- Each signature can optionally be mirrored as a signed reaction or custom event referencing the letter event.
- Multiple relays can store and redistribute the letter/signature events, making takedown harder and enabling independent mirrors.

Why this does not remove the need for a database in v1:

- Email workflows still need private state: author email, recipient emails, inbound routing tokens, preview/publish tokens, unsubscribe state, webhook dedupe, redaction drafts, and abuse controls. This data must not be public on relays.
- Nostr event deletion is best-effort, while this product needs reliable privacy behavior such as removing raw email addresses when they are no longer needed.
- Signature dedupe, 6-digit code verification, notification subscriptions, and milestone emails require transactional state.
- Relay availability and indexing are eventually consistent; the web app needs a reliable source of truth for previews, publishing, email sends, and activity logs.

Recommended v1 stance:

- Keep SQLite as the source of truth.
- Design entities with optional `nostr_event_id`, `nostr_pubkey`, and `nostr_published_at` fields later if/when mirroring is added.
- Add Nostr as a v1.1/v2 publishing/mirroring layer after core email + signing flows work.
- Explore Nostr-native signing only after deciding the UX for non-Nostr users and how to reconcile Nostr signatures with email-code signatures.

## 15. Open product decisions

- Should author email verification be required before preview or only before publish? Recommended: before publish.
- Should recipient display names be required in v1? Decision: yes, ask for both name and email.
- Should signer display names be required? Recommended: yes, with email-code verification.
- How should duplicate signatures be handled? Recommended: dedupe by salted verified-email hash.
- Should milestone jumps send only the highest milestone or every crossed milestone? Recommended: highest newly crossed milestone only.
- What should be the exact inbound email address format? Recommended: `reply+{letterId}@{inboundDomain}`.
