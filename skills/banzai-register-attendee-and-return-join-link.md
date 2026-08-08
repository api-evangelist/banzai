---
name: Register an attendee for a Demio Event and return their join link
description: >-
  Find the right Demio Event and Session, register a person for it, and hand back the
  unique join link — the marquee flow of the Public Demio API.
api: openapi/banzai-demio-openapi.yml
operations:
  - pingViaHeaders
  - listEvents
  - getEvent
  - registerForEvent
generated: '2026-08-06'
method: generated
---

# Register an attendee for a Demio Event

Base URL: `https://my.demio.com/api/v1`

## Authorization

Send both credentials as headers on every request:

```
Api-Key: <account api key>
Api-Secret: <account api secret>
```

Both values come from Demio **Settings > API** (`https://my.demio.com/manage/api-details`).
A query-string form (`?api_key=…&api_secret=…`) is documented but puts the secret in
URLs and logs — do not use it.

## Steps

1. **Verify credentials — `pingViaHeaders`**
   `GET /ping`. A `200` returns `{"pong": true, "sandbox": <bool>}`. Read `sandbox`:
   if it is `true` you are working against sandbox credentials and registrations will
   not reach a live Event. A `401` means the key/secret pair is wrong; a `403` means
   the credentials are valid but the Demio account is not active.

2. **Find the Event — `listEvents`**
   `GET /events?type=upcoming`. Returns an array of Events, each with `id`, `name`,
   `date_id` (the *next* Session), `status`, `timestamp` (Unix), `zone` and
   `registration_url`. Filter by `name` on your side — there is no search parameter.
   Valid `type` values are `upcoming`, `past`, `automated`.

3. **Pick the Session — `getEvent`**
   Only needed when the attendee must land on a specific occurrence of a series.
   `GET /event/{id}` returns `next_date_id` plus the full `dates[]` array, each entry
   carrying `date_id`, `status` (`scheduled` / `running` / `finished`), `timestamp`,
   `datetime` and `zone`. Add `?active=true` to drop past dates in a series. Note the
   shape difference: the list projection inlines one Session's fields on the Event,
   the detail projection nests them in `dates[]` — do not treat them as the same
   object.

4. **Register the person — `registerForEvent`**
   `PUT /event/register` with `Content-Type: application/json`:

   ```json
   {
     "id": 1,
     "date_id": 35,
     "name": "John",
     "email": "john.doe@example.com",
     "last_name": "Doe"
   }
   ```

   - Identify the Event with either `id` **or** `ref_url` (the registration page URL);
     set the unused one to `null`.
   - Omit `date_id` (or send `null`) to let Demio pick the nearest active Session.
   - `name` and `email` are required. `last_name`, `company`, `website`,
     `phone_number` and `gdpr` are the predefined optional fields.
   - Custom fields are passed as extra top-level keys named by the field's **Unique
     Identifier** from the Event's Registration block, e.g. `{"job_title": "CTO"}`.

5. **Return the join link**
   A `201` returns `{"hash": "…", "join_link": "https://event.demio.com/join/<hash>"}`.
   The hash is unique to that person for that Session — treat it as a credential, and
   never post it anywhere the wrong person can read it.

## Rules an agent must follow

- **No idempotency.** There is no `Idempotency-Key` and Demio does not document what
  a repeated registration for the same email does. On a timeout or 5xx, do **not**
  blind-retry — re-read the participant list for the Session
  (`listSessionParticipants`) and only retry if the person is absent.
- **This operation emails a human.** Registration can trigger Demio confirmation and
  reminder emails to the address you submit. Confirm the address with the user before
  calling it; never register an address you inferred.
- **Errors are strings, not codes.** Failures return `{"messages": ["…"]}` with no
  stable code. `400` is either validation (`Name must be more than 2 symbols`, `Wrong
  Email format`) or malformed JSON (`Bad Request syntax…`). Surface the message text
  verbatim rather than mapping it.
- **Budget your calls.** 180 requests/minute, and a hard daily account quota of 100
  calls on a free trial or 5,000 for paying customers, resetting at 00:00 UTC and
  shared with Zapier. No `429` and no rate-limit headers are documented, so an
  unexplained error late in a batch is most likely quota exhaustion. Cache the Event
  list rather than re-listing per registrant.
- **No pagination.** `GET /events` returns everything; do not look for a cursor.
