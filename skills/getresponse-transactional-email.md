---
name: getresponse-transactional-email
description: >
  Use this skill to send and track GetResponse transactional email — receipts, password resets,
  shipping notices — and to manage the templates they render from.
  Triggers: whenever the user wants to (1) create or update a transactional email template,
  (2) send a one-to-one transactional message, or (3) read delivery/engagement statistics for
  transactional sends.
  Do NOT use for marketing newsletters or broadcasts (use getresponse-newsletter-skill), for
  autoresponder sequences, or for SMS.
api: openapi/getresponse-transactional-emails-openapi.yml
operations:
  - getTransactionalEmailsTemplatesList
  - createTransactionalEmailTemplate
  - getTransactionalEmailsTemplatesById
  - updateTransactionalEmailsTemplate
  - createTransactionalEmail
  - getTransactionalEmailsById
  - getTransactionalEmailsList
  - getTransactionalEmailsStatistics
  - getFromFieldList
---

# GetResponse transactional email

Grounded in the provider-published OpenAPI at
`https://apireference.getresponse.com/open-api.json`. Every operationId below is verified
present in that spec.

## Before you start

- **Auth.** `X-Auth-Token: api-key <KEY>` (the `api-key ` prefix is literal). MAX accounts add
  `X-Domain: <account-domain>`.
- **This is the highest-consequence surface in the API.** A transactional send goes to a real
  inbox immediately and cannot be recalled. There is no idempotency key
  (`conventions/getresponse-conventions.yml`), so a retried `createTransactionalEmail` sends a
  second copy. Never blind-retry a send. On a timeout, reconcile with
  `getTransactionalEmailsList` before deciding whether to send again.

## Step 1 — Resolve the sender

`getFromFieldList` (`GET /from-fields`) and pick a verified from-field. A send from an
unverified address fails with error code 1001. Do not create a from-field on the user's behalf
without asking — it requires an email confirmation they have to action.

## Step 2 — Resolve or create the template

1. `getTransactionalEmailsTemplatesList` (`GET /transactional-emails/templates`) — search
   before creating. Filter with `query%5Bname%5D=<name>` (brackets percent-encoded).
2. `createTransactionalEmailTemplate` (`POST /transactional-emails/templates`) with the
   subject, HTML and plain-text bodies and the from-field.
3. `updateTransactionalEmailsTemplate`
   (`POST /transactional-emails/templates/{transactionalEmailTemplateId}`) to revise it.
   Verify the merge tags render before you send — a parse failure surfaces as error code 1012
   ("message parsing error or missing merge words") at send time, not at template-save time.

Read a single template back with `getTransactionalEmailsTemplatesById`.

## Step 3 — Send

`createTransactionalEmail` (`POST /transactional-emails`) with the recipient, the template id
and the substitution values.

Confirm with the user before this call, every time, unless they have explicitly authorised an
unattended loop and told you the recipient set. State the recipient and the template name back
to them first.

## Step 4 — Track

- `getTransactionalEmailsById` (`GET /transactional-emails/{transactionalEmailId}`) — the status
  of one send.
- `getTransactionalEmailsList` (`GET /transactional-emails`) — reconcile a batch. This is the
  operation to use after a timeout, before considering a resend.
- `getTransactionalEmailsStatistics` (`GET /transactional-emails/statistics`) — aggregate
  delivery and engagement.

Engagement also arrives as events on the webhook surface ("Message opened", "Link clicked",
"Contact bounce removed") — see `asyncapi/getresponse-webhooks.yml`. Note that webhook
subscriptions cannot be created through the API; the user must add them in the GetResponse UI.

## Rate limits

30,000 calls per 10-minute frame, 80/second, 10 concurrent. Read `X-RateLimit-Remaining` from
every response; on a 429 (code 1015) wait `context.timeToReset` seconds. There is no
`Retry-After` header.

## Error handling

Branch on the numeric `code`, not the HTTP status — see `errors/getresponse-error-codes.yml`.
Notably: **1014 is a 403, not a 401**, and 1011/1012 are message-level failures that will not
be fixed by retrying. Log the `uuid` from every error body.
