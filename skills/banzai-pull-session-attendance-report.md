---
name: Pull the attendance report for a Demio Session
description: >-
  Resolve an Event to the Session you care about and pull its participant list with
  attendance status and custom registration-field values.
api: openapi/banzai-demio-openapi.yml
operations:
  - pingViaHeaders
  - listEvents
  - getEvent
  - getEventSession
  - listSessionParticipants
generated: '2026-08-06'
method: generated
---

# Pull the attendance report for a Demio Session

Base URL: `https://my.demio.com/api/v1`. Authorize with the `Api-Key` and `Api-Secret`
headers from Demio **Settings > API**.

## Steps

1. **Verify credentials — `pingViaHeaders`**
   `GET /ping` → `{"pong": true, "sandbox": <bool>}`. Sandbox credentials will not
   return live attendance data.

2. **Resolve the Event — `listEvents`**
   `GET /events?type=past` for a webinar that has already run. Match on `name`
   client-side.

3. **Resolve the Session ID — `getEvent`**
   `GET /event/{id}` returns `dates[]`. The reporting surface is keyed on `date_id`
   (Demio's UI calls it the **Session ID**), *not* on the Event ID. Pick the entry
   whose `status` is `finished` and whose `timestamp` matches the run you want.
   `getEventSession` (`GET /event/{id}/date/{date_id}`) confirms one Session's status
   and human-readable `datetime` if you need to double-check before reporting.

4. **Pull participants — `listSessionParticipants`**
   `GET /report/{date_id}/participants` returns
   `{"participants": [{email, name, custom_fields[], attended, status}]}`.
   Filter server-side with `?status=`, one of `attended`, `did not attend`,
   `completed`, `left early`, `banned`. Omit it to get everyone.

5. **Read the fields correctly**
   - `attended` is the boolean; `status` is the finer grain (`completed` vs `left
     early` vs `did not attend` vs `banned`).
   - `custom_fields[]` entries are `{id, name, value}` where `id` is the field's
     Unique Identifier — the same key used when registering. Predefined fields such as
     `last_name` and `website` appear here too, not as top-level properties.
   - There is no participant id. **Email is the only key**, so a person who registered
     twice under two addresses appears twice.

## Rules an agent must follow

- **This returns personal data.** Names, email addresses and whatever the Event's
  registration form collected. Do not dump the raw response into a shared surface,
  and aggregate rather than enumerate unless the user asked for the list.
- **404 discipline.** `Event not found` and `Event Date not found` are the two 404
  messages; both mean you used an ID from the wrong entity. Re-resolve with
  `listEvents` / `getEvent` rather than guessing an adjacent integer.
- **No pagination, no totals.** The whole participant list comes back in one body and
  there is no count field — compute totals yourself.
- **Budget your calls.** 180 requests/minute and a daily account quota (100 on trial,
  5,000 for customers, reset 00:00 UTC, shared with Zapier). Reporting across many
  Sessions burns quota fast: resolve `date_id`s in one pass over `getEvent`, then make
  one report call per Session.
