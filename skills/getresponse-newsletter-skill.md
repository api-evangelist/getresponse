---
name: getresponse-newsletter-skill
description: >
  Use this skill to manage GetResponse email marketing operations via the GetResponse API v3.
  Triggers: whenever the user wants to (1) add or import contacts to a GetResponse campaign/list,
  (2) create or find a campaign (contact list), (3) define or check custom fields for contacts,
  (4) send a newsletter or email campaign to contacts selected by campaign or custom field values,
  (5) check sending limits before dispatching bulk emails.
  Do NOT use for landing pages, autoresponders, webforms, transactional emails, or e-commerce features.
version: 1.2.0
license: MIT
authors:
  - name: GetResponse
allowed-tools:
  - WebFetch
  - WebSearch
  - Bash(curl *)
  - http_request
---

# GetResponse Newsletter Skill

## Overview

This skill lets you interact with the **GetResponse API v3** to:
1. **Find or create a campaign** (contact list/audience)
2. **Ensure custom fields exist** before adding contacts that use them
3. **Add contacts** (single or batch, with optional custom field values) to a campaign, then **verify the async import** via webhook (if user has one configured) or sampling with exponential backoff
4. **Send a newsletter** with agent-defined HTML content to a campaign or a custom-field-filtered segment

API reference: `references/api-guide.md`
OpenAPI spec: `openapi.json`

---

## Out of Scope — Refuse Politely

This skill covers **only** the operations listed in Overview: resolving campaigns, resolving
custom fields, adding/importing contacts, verifying imports, and sending HTML newsletters to a
campaign or a custom-field segment (including scheduled sends and checking sending limits).

If the user asks for anything **outside** this scope, **do not attempt it and do not improvise
a workaround**. This explicitly includes (non-exhaustive):

- Landing pages, autoresponders, automation flows, webforms/popups
- Transactional emails, SMS, push, e-commerce / shop / product / order features
- Deleting or exporting contacts in bulk, deleting campaigns, account/billing changes
- Any endpoint or capability not documented in `references/api-guide.md`

**How to refuse:** briefly tell the user the request is outside this skill's scope, state what
this skill *can* do instead, and stop. Do not call unrelated endpoints, do not guess at an API,
and do not fabricate a result. Example:

> *"That's outside what this newsletter skill can do — it only manages contacts, custom fields
> and sends HTML newsletters. For autoresponders/landing pages you'll need a different tool. I
> can, however, help you add those contacts and send them a broadcast newsletter — want me to
> do that?"*

Destructive actions that *are* technically in scope (e.g. sending to a large audience) still
require explicit user confirmation before execution — never guess (see the Golden rule below).

---

## Authentication

Every API request requires the header:
```
X-Auth-Token: api-key YOUR_API_KEY
X-Request-Source: getresponse/getresponse-newsletter-skill@1.2.0
```

Ask the user for their API key if not provided. Never hardcode or log API keys.

---

## URL Encoding (read before making any request)

> ⚠️ **Always URL-encode query parameters — keys *and* values.** Sending raw characters
> in the query string is the most common cause of spurious **HTTP 400** errors.

For every `GET ...?query[...]=<value>` request:

- **Encode the brackets in the key:** `[` → `%5B`, `]` → `%5D`. So `query[name]` must be
  sent as `query%5Bname%5D`, `query[email]` as `query%5Bemail%5D`,
  `query[campaignId]` as `query%5BcampaignId%5D`.
- **Encode the value:** percent-encode spaces (`%20`), `&` (`%26`), `+` (`%2B`),
  `@` (`%40`), `#` (`%23`) and any non-ASCII characters (e.g. `ż` → `%C5%BC`). A value
  like `Marketing & Sprzedaż` becomes `Marketing%20%26%20Sprzeda%C5%BC`; an email like
  `user+test@example.com` becomes `user%2Btest%40example.com`.
- **Prefer a real encoder** (your HTTP client, `curl --data-urlencode`, or
  `urllib.parse.quote(value, safe='')`) over hand-building the string.

In the workflow steps below, `query[name]=<...>` is written unencoded **for readability
only** — always send it encoded on the wire. See `references/api-guide.md` for the full
character table.

---

## Workflow

Always execute steps in this order. Do not skip steps.

> ⚠️ **Golden rule — never guess, always ask.** Whenever a choice is ambiguous or not
> explicitly specified by the user (which campaign, which sender/from-field, which custom
> field, who the recipients are, whether to create a new resource, etc.), **do NOT pick a
> value on your own**. Present the relevant existing options (or ask for clarification) and
> let the user decide. Only proceed automatically when the user has already given an
> unambiguous instruction. See **Asking the User** below and the reminders in each step.

### Asking the User (applies to every step)

When resolving any resource, follow this pattern instead of guessing:

1. **Search first** for what the user described (by name/query).
2. **Exactly one confident match** → you may use it, but briefly state which one you picked
   (e.g. *"Using campaign 'Spring-2026' (id: abc123)."*) so the user can correct you.
3. **Multiple matches, no match, or an ambiguous description** → **stop and ask**. Show a
   short list of the most relevant existing options (default: up to **10**, most recent
   first) numbered for easy selection, and ask the user to pick one or provide a different
   name/value. Example:
   > *"I found several matching campaigns. Which one should I use?*
   > *1) Spring-2026 (id: abc123)*
   > *2) Spring-Newsletter (id: def456)*
   > *…*
   > *Or tell me a different campaign name."*
4. **Creating a new resource** (campaign, custom field, etc.) → only create it after the
   user explicitly confirms, or when the user clearly asked to create something new. Never
   silently create a resource just because a lookup returned nothing.

### Step 1 — Resolve Campaign

**Goal:** obtain a valid `campaignId`. **Never guess the campaign — confirm it with the user.**

1. Call `GET /campaigns?query[name]=<campaign_name>` (use `listCampaigns`).
   **URL-encode the value** (e.g. `Q2 Leads` → `query%5Bname%5D=Q2%20Leads`) — a raw space,
   `&`, `+` or a national character here will trigger HTTP 400 (see **URL Encoding** above).
2. **Exactly one confident match** → use its `campaignId`, and state which campaign you
   selected so the user can correct you.
   > **What counts as a confident match:** `query[name]` performs a *partial/prefix* match on
   > the GetResponse side, so a single returned result is **not** automatically the right one.
   > Treat it as confident only when the returned campaign name **equals the requested name
   > exactly (case-insensitive)**. If the only result merely *contains* or *starts with* the
   > requested text (e.g. you asked for `Q2` and got `Q2-Leads-Backup`), that is **not** a
   > confident match → go to step 3 and ask.
3. **Multiple matches, or a single non-exact/ambiguous match** → **stop and ask**. Present the
   matching campaigns (numbered, most recent first) and ask the user to choose one or give a
   more precise name. Do not pick one arbitrarily.
4. **No match** → do **not** silently create a campaign. Instead:
   - List the user's most recent campaigns (call `listCampaigns`, take up to **10**, most
     recent first) and ask: *"I couldn't find a campaign matching '<name>'. Here are your
     10 most recent campaigns — should I use one of these, or create a new campaign named
     '<name>'?"*
   - Only after the user confirms creation, create it with `POST /campaigns`
     (use `createCampaign`):
     - Required field: `name`
     - Recommended: set `optinTypes.api` to `"single"` so API-added contacts skip double opt-in
     - Save the returned `campaignId`.

### Step 2 — Resolve Custom Fields (skip if contacts have no custom fields)

**Never guess a custom field mapping and never silently create one — confirm with the user.**

For **each** custom field name referenced in the contact data:

1. Call `GET /custom-fields?query[name]=<field_name>` (use `listCustomFields`).
   Remember to URL-encode the key brackets (`query%5Bname%5D`) and the value.
2. **Confident match (exact name, case-insensitive)** → record its `customFieldId` (briefly
   state which field you matched).
   > **Check the field `type` before reusing it.** Existing custom fields have a fixed `type`
   > (`text`, `number`, `date`, `single_select`, `multi_select`, …). If the existing field's
   > `type` is **incompatible** with the data you're about to write (e.g. the field is
   > `single_select` / `date` / `number` but your values are free-form text, or the value
   > isn't one of the field's allowed `values`), do **not** silently reuse it and do **not**
   > create a duplicate. **Stop and ask** the user how to proceed (use the existing field
   > anyway, pick a different field, or create a new one). Only reuse silently when the
   > existing field's type clearly fits the data.
3. **Ambiguous match** (e.g. several similarly named fields, or only a partial/prefix match
   rather than an exact name) → **stop and ask** which existing field to use, showing the
   candidates and their types.
4. **Not found** → do **not** silently create it. Ask the user first, e.g.: *"There's no
   custom field named '<field_name>'. Should I create it (type: text), or did you mean one
   of your existing fields: <list>?"* Only after confirmation, create it with
   `POST /custom-fields` (use `createCustomField`):
   - Required: `name` (lowercase, underscores, no spaces), `type` (use `"text"` for strings)
   - Always include `"hidden": "false"` and `"values": []` — the API requires both fields even for `text` type
   - Save the returned `customFieldId`.

Run these lookups in parallel when there are multiple custom fields to resolve, but batch
any "which field / create?" questions into a single message to the user.

### Step 3 — Add Contacts

Choose the appropriate method based on contact count:

#### Single contact (1 contact):
Use `POST /contacts` (use `createContact`):
```json
{
  "email": "user@example.com",
  "name": "First Last",
  "campaign": { "campaignId": "<campaignId>" },
  "customFieldValues": [
    { "customFieldId": "<id>", "value": ["<value>"] }
  ]
}
```
Response 202 = accepted (async). Response 409 = already exists (not an error, continue).

#### Multiple contacts (2–1000):
Use `POST /contacts/batch` (use `batchCreateContacts`):
```json
{
  "campaignId": "<campaignId>",
  "contacts": [
    { "email": "a@example.com", "name": "A", "customFieldValues": [...] },
    { "email": "b@example.com", "name": "B", "customFieldValues": [...] }
  ]
}
```

#### Large batches (>1000 contacts):
1. Check limits: `GET /accounts/sending-limits` (use `getSendingLimits`).
2. Split contacts into chunks of 1000. Keep track of each chunk (index + the emails it contains)
   so you can report precisely on partial failures.
3. Send chunks in parallel — up to 10 concurrent requests (respect rate limits: 80 req/s, 30 000 req/10 min).
4. **Handle each chunk's result individually — a multi-chunk import can partially succeed:**
   - A chunk that returns **202** was *accepted* for async processing (it is **not** a guarantee
     every record inside it is valid — see Step 3b; bad emails etc. can still be dropped
     asynchronously).
   - A chunk that returns a **4xx/5xx** (or fails to send) is a failed chunk. **Retry that chunk
     once.** If it still fails, mark it as failed and **keep going with the remaining chunks** —
     do not abort the whole import because one chunk failed.
   - On HTTP **429**, follow the rate-limit rule (read `Retry-After`, pause, retry) — that is a
     throttle, not a permanent chunk failure.
5. **Report per-chunk results to the user** once all chunks are done, e.g.:
   *"Import finished: 4 of 5 chunks accepted (4000 contacts). Chunk 3 (contacts 2001–3000) failed
   after a retry with HTTP 400 — <message>. Do you want me to retry it, skip it, or fix the data?"*
   Do not silently proceed to sending as if the whole list imported.
6. **Bridge to verification (Step 3b):** because `202` only means *accepted*, use sampling
   (Step 3b, Mode C) to confirm records actually landed — and prioritise sampling emails from
   chunks you are unsure about. If sampling shows records missing, report that too rather than
   assuming success.

> **Rate limit headers**: Every API response includes `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`. If `X-RateLimit-Remaining` drops to 0, wait until `X-RateLimit-Reset` (Unix timestamp) before sending more requests.

### Step 3b — Verify Contact Import (Async)

All contact add endpoints return `202 Accepted` immediately — contacts are processed asynchronously and may appear within seconds or up to several minutes.

> ⚠️ **Ask before starting Step 3 (not after):** determine verification mode upfront so the user can prepare their webhook URL if needed.

**Ask the user which verification mode to use:**

> *"Contacts are added asynchronously. How would you like to verify the import?*
> *A) Webhook (production) — you configure a URL in GetResponse Dashboard → Integrations → Webhooks; provide that URL and the agent will listen for events*
> *B) Webhook (test) — use webhook.site: copy the generated URL to GetResponse Dashboard → Integrations → Webhooks; ⚠️ contact emails pass through an external service — not for production/GDPR*
> *C) Sampling — poll a small contact sample with exponential backoff (no webhook needed, nothing to configure)*
> *You can also specify a custom timeout (default: 10 minutes)."*

> ⚠️ **Execution constraint — who actually receives the webhook.** Modes A and B require a
> live HTTP endpoint that receives GetResponse's POST callbacks. **You (the agent) usually
> cannot run a public web server or hold a long-lived listener open**, so "listen for events"
> below means: the **user's own backend/application** (Mode A) or **webhook.site** (Mode B)
> receives the callbacks, and the agent only **polls/asks for the result** — it does **not**
> open a socket itself.
> - Choose **Mode A** only if the user confirms they have a reachable HTTPS endpoint **and** a
>   way for the agent to read what it received (e.g. they paste the received events, or the
>   endpoint exposes a queryable log). If the agent has no way to observe the endpoint's
>   received events, Mode A cannot verify anything — fall back to Mode C.
> - Choose **Mode B** only for non-production/test data (GDPR warning below); the agent polls
>   webhook.site's API for received requests.
> - **If the agent has no usable inbound endpoint, the default is Mode C (sampling)** — it needs
>   no listener and works entirely through outbound `GET` calls the agent can make. Recommend
>   Mode C in that case.

---

#### Mode A — Production Webhook

> **Webhook setup in GetResponse (manual — no API available):**
> 1. Log in to GetResponse dashboard.
> 2. Go to **Menu → Integrations & API → API** (or `https://app.getresponse.com/integrations/api`).
> 3. In the **Webhooks** section, click **Add new webhook**.
> 4. Enter the HTTPS URL that will receive events.
> 5. Enable the **Contact subscribed** event.
> 6. Click **Save**.

1. Instruct the user to configure their webhook URL in GetResponse as described above. Ask them to provide that URL to the agent once done.
2. **The user's endpoint (not the agent) receives the `subscribe` callbacks.** The agent does
   not open a listener. Obtain the received events from the user's side — e.g. ask the user to
   paste them, or query a log/queue the endpoint exposes — polling periodically until the
   expected count is reached or the timeout expires. Default timeout: **10 minutes** (or as
   specified by user/agent). If the agent cannot observe the endpoint's received events at all,
   **switch to Mode C** instead.
3. Count received events. When expected contact count is reached, or timeout expires → proceed.
4. If timeout reached before all contacts confirmed: report how many were confirmed, continue with Step 4.

---

#### Mode B — Test Webhook (webhook.site)

⚠️ **GDPR warning**: contact email addresses will pass through webhook.site servers (external US service). Only use for development/testing with non-production data. Inform the user before proceeding.

1. Fetch a unique URL: `GET https://webhook.site/token` → save `uuid` and `url`.
2. Provide the generated URL to the user: *"Please paste this URL into GetResponse Dashboard → Integrations → Webhooks: `<webhook.site url>`"*. Wait for the user to confirm they've configured it before continuing.
3. Poll for incoming events: `GET https://webhook.site/token/<uuid>/requests` every **15 seconds**.
   - Count events where `type` = `subscribe`.
   - Stop when expected count reached, or timeout (default: **10 minutes**) expires.
4. Report confirmation count to user, continue with Step 4.

---

#### Mode C — Sampling with Exponential Backoff

Verify a small sample of contacts via `GET /contacts?query[email]=<email>` — do **not** query every contact individually. **URL-encode the email** (e.g. `user+test@example.com` → `query%5Bemail%5D=user%2Btest%40example.com`); a raw `+` is decoded server-side as a space and the lookup fails (or returns 400).

**Sample size:**
| Contacts added | Sample size | Max GET requests |
|---|---|---|
| 1–10 | all | ≤ 10 |
| 11–1000 | 5 (first, ~25%, ~50%, ~75%, last) | 5 |
| >1000 (multi-batch) | 1 per batch, max 10 total | ≤ 10 |

**Backoff schedule:**
| Attempt | Wait before check | Elapsed |
|---|---|---|
| 1 | 5 s | 5 s |
| 2 | 15 s | 20 s |
| 3 | 30 s | 50 s |
| 4 | 60 s | ~2 min |
| 5 | 120 s | ~4 min |
| 6 | 300 s | ~9 min |
| 7 (max) | — | timeout |

- If sample is 100% visible → assume full import succeeded, proceed.
- If sample is partially visible → wait and retry.
- After 7 attempts (≈10 min default): report partial confirmation, continue with Step 4.
- Custom timeout: adjust number of attempts proportionally.

---

#### When verification can be skipped entirely

Skip Step 3b if **all** of the following are true:
- Newsletter will be sent to the **whole campaign** (`selectedCampaigns`) — not to individual `selectedContacts`
- User/agent explicitly says verification is not needed

In this case, inform the user: *"Contacts submitted for import (async). Newsletter will be sent to all active contacts in the campaign at send time — no verification needed."*

### Step 4 — Send Newsletter

#### 4a — Get Sender Address
**Never guess the sender — confirm the from-field with the user.**

1. Call `GET /from-fields` (use `listFromFields`) to retrieve the available sender addresses.
2. **The user already specified a sender** → use the matching `fromFieldId`. Match on an
   **exact** email/name (case-insensitive); if what the user said only partially matches one
   of several addresses, treat it as ambiguous → go to step 4 and ask. State which one you used.
3. **Exactly one from-field exists** → use it, but tell the user which address will be used.
4. **Multiple from-fields and none specified** → **stop and ask**. Present the available
   sender addresses (numbered; mark the default if `isDefault` is true) and ask which one to
   send from. Do not silently fall back to the default or the "first" address. Example:
   > *"Which sender address should this newsletter come from?*
   > *1) Marketing <marketing@example.com> (default)*
   > *2) Support <support@example.com>"*

#### 4b — Determine Recipients
**Never guess who the recipients are — confirm the audience with the user** before sending
(e.g. whether it's the whole campaign or a filtered segment, and which custom-field value to
filter on). Choose **one** strategy:

| Strategy | When to use | `sendSettings` field |
|---|---|---|
| By campaign | Send to all contacts in the campaign | `selectedCampaigns: ["<campaignId>"]` |
| By custom field | Send to contacts matching a custom field value | Use `searchContactsByConditions` first (see below), then `selectedContacts` |

**Searching by custom field** — call `POST /search-contacts/contacts` (use `searchContactsByConditions`):
```json
{
  "subscribersType": ["subscribed"],
  "sectionLogicOperator": "and",
  "section": [
    {
      "campaignIdsList": ["<campaignId>"],
      "subscriberCycle": ["receiving_autoresponder", "not_receiving_autoresponder"],
      "subscriptionDate": "all_time",
      "logicOperator": "and",
      "conditions": [
        {
          "conditionType": "custom",
          "scope": "<customFieldId>",
          "operator": "is",
          "operatorType": "string_operator_list",
          "value": "<custom_field_value>"
        }
      ]
    }
  ]
}
```
- `campaignIdsList`: array of plain campaignId strings
- `subscriberCycle`: use both values to include all contacts regardless of autoresponder state
- `subscriptionDate`: use `"all_time"` unless filtering by date is needed
- `scope`: the `customFieldId` of the custom field to filter on
- `value`: a plain string (not an array)

Collect the returned `contactId` values, then use `selectedContacts` in the newsletter's `sendSettings`.

> ⚠️ **Zero recipients → do NOT send.** If the search returns **0 contacts** (empty result),
> **stop and do not create/send the newsletter** — sending to an empty segment is almost never
> intended and can error or send to no one. Instead, tell the user the filter matched nobody and
> ask how to proceed, e.g.:
> *"No contacts match department = 'Engineering' in this campaign, so there's no one to send to.
> Did you mean a different value/campaign, should I widen the filter, or send to the whole
> campaign instead?"*
> Only proceed once the user gives an audience that resolves to **at least one** recipient (or
> explicitly switches to `selectedCampaigns`). The same applies if the whole campaign has no
> subscribed contacts.

#### 4c — Create and Send Newsletter
Call `POST /newsletters` (use `createNewsletter`):
```json
{
  "subject": "<subject line>",
  "name": "<internal name>",
  "type": "broadcast",
  "content": {
    "html": "<FULL HTML CONTENT PROVIDED BY AGENT>",
    "plain": "<plain text version>"
  },
  "fromField": { "fromFieldId": "<id>" },
  "campaign": { "campaignId": "<id>" },
  "sendSettings": {
    "selectedCampaigns": ["<campaignId>"]
  },
  "flags": ["openrate", "clicktrack"]
}
```

- `content.html` must be the complete HTML document provided by the calling agent — do not truncate or modify it.
- Omit `sendOn` to send immediately; include it (ISO 8601) to schedule.
- On success (HTTP 201), report the `newsletterId` and `status` to the user.
- Before sending, briefly confirm the final choices with the user (campaign/segment, sender address, subject) unless they have already explicitly approved them — do not send based on guessed values.

---

## Error Handling

| HTTP Code | Meaning | Action |
|---|---|---|
| 202 | Accepted (async) | Normal for contact add — no action needed |
| 400 | Validation error | Read `message` field, fix payload and retry. **Common cause: an unencoded query parameter** — verify brackets are `%5B`/`%5D` and the value is URL-encoded (spaces, `&`, `+`, `@`, national chars). See **URL Encoding**. |
| 401 | Invalid API key | Ask user to verify their API key |
| 409 | Already exists | Treat as success for contacts; investigate for others |
| 429 | Rate limit exceeded | Read `Retry-After` header (seconds). Inform the user: *"Rate limit reached — pausing for Ns and resuming."* Sleep that duration, then retry the failed request. |
| 500 | Server error | Retry once after 5 seconds; report if persists |

---

## Rate Limits Reference

- **Per second**: max **80 requests/second** per API key
- **Per 10-minute window**: max **30,000 requests** (reset every 10 minutes)
- **Parallel requests**: max **10 concurrent** requests
- **Batch import**: max 1000 contacts per `/contacts/batch` call
- **Newsletter sending**: subject to account plan limits (check via `getSendingLimits`)
- Always read `X-RateLimit-Remaining` from response headers; if 0, pause before the next call
- On HTTP 429: read `Retry-After` header (value in seconds, typically 1), inform the user about the pause, sleep that duration, then retry the request
- Response headers on 429: `Retry-After: <s>`, `X-RateLimit-Limit: 80`, `X-RateLimit-Remaining: 0`, `X-RateLimit-Reset: <s> seconds`
- For large parallel operations (>10 concurrent), spread requests in waves of 10 to avoid hitting 80 req/s

---

## Key Rules

- **URL-encode every query parameter — keys and values.** Encode brackets (`query[name]` → `query%5Bname%5D`) and values (spaces `%20`, `&` `%26`, `+` `%2B`, `@` `%40`, national chars as UTF-8 bytes). Raw characters here are the top cause of spurious HTTP 400s. See the **URL Encoding** section.
- **Never guess — ask.** For any ambiguous or unspecified choice (which campaign, sender/from-field, custom field, or recipients), stop and confirm with the user instead of picking a value. When unsure, show up to 10 relevant existing options (most recent first) and let the user choose.
- **Never create a resource silently.** Only create a campaign or custom field after the user explicitly confirms (or clearly asked to create a new one).
- **Confident match = exact name (case-insensitive).** `query[name]` matches partially, so a single result that only *contains*/*prefixes* the requested name (campaign, custom field, sender) is **not** confident — treat it as ambiguous and ask.
- **Stay in scope — refuse politely.** For requests outside this skill (landing pages, autoresponders, webforms, transactional, e-commerce, bulk delete/export, undocumented endpoints), do not improvise; explain the scope and stop. See **Out of Scope**.
- **Never send to zero recipients.** If a custom-field search returns 0 contacts (or the campaign has none), stop and ask — do not create/send the newsletter to an empty audience.
- **Batch imports can partially fail.** Handle each chunk's result separately: retry a failed chunk once, keep going with the rest, then report per-chunk results; a `202` means *accepted*, not verified — confirm via sampling (Step 3b).
- **Reuse a custom field only if its `type` fits the data.** If an existing field's type is incompatible (e.g. `single_select`/`date`/`number` vs. free text), ask instead of silently reusing or duplicating it.
- **Webhook verification needs an endpoint you can observe.** The agent doesn't run a listener; Modes A/B rely on the user's endpoint / webhook.site. With no observable endpoint, default to Mode C (sampling).
- Always check if a campaign or custom field exists before creating it — avoid duplicates.
- Custom field `name` must be lowercase with underscores (e.g. `company_name`, not `Company Name`).
- Custom field `value` in contact payloads is always an **array**: `["value"]`, never a plain string.
- The HTML newsletter content is defined by the calling agent — never generate placeholder HTML.
- Never expose or log the API key in outputs.
- If any required piece of information is missing (API key, campaign name, HTML content), ask the user before proceeding.
