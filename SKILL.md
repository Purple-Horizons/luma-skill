---
name: luma
description: Manage events, guests, calendars, tags, and RSVPs on Lu.ma (Luma). Use when the user asks about events, event planning, meetups, RSVPs, guest lists, attendees, ticket types, coupons, calendars, or anything related to Lu.ma. Also trigger when the user mentions creating events, checking who's attending, sending invites, managing event tags, or importing contacts for events.
---

# Lu.ma Events API

Manage events, guests, calendars, and more via the Lu.ma public API.

## Authentication

All requests use an API key header. The `LUMA_API_KEY` env var must be set.

```bash
curl -s "https://public-api.luma.com/v1/<endpoint>" \
  -H "x-luma-api-key: $LUMA_API_KEY"
```

## API Conventions

- **Base URL**: `https://public-api.luma.com`
- **Reads**: GET requests with query params
- **Writes**: POST requests with JSON body
- **IDs**: Event IDs start with `evt-`
- **Dates**: ISO 8601 format (e.g., `2026-04-01T18:00:00Z`)
- **Pagination**: Cursor-based — use `pagination_cursor`, `pagination_limit` (default 50); response includes `has_more` and `next_cursor`. When counting totals (e.g., guest counts), you MUST loop through all pages until `has_more` is false — a single page only returns up to 50 entries.
- **Sorting**: Use `sort_column` and `sort_direction` (asc/desc). For events, `sort_column=start_at` with `sort_direction=desc` gives newest first.
- **Rate limits**: 500 GET / 100 POST per 5 minutes per calendar

## Events

### List events
```bash
# List upcoming events (after today)
curl -s -G "https://public-api.luma.com/v1/calendar/list-events" \
  -H "x-luma-api-key: $LUMA_API_KEY" \
  --data-urlencode "after=$(date -u +%Y-%m-%dT00:00:00Z)" \
  --data-urlencode "pagination_limit=50"

# Find most recent past event (sort descending, look before now)
curl -s -G "https://public-api.luma.com/v1/calendar/list-events" \
  -H "x-luma-api-key: $LUMA_API_KEY" \
  --data-urlencode "before=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --data-urlencode "sort_direction=desc" \
  --data-urlencode "pagination_limit=1"
```

Params: `after`, `before` (ISO 8601 date filters), `sort_column` (only `start_at`), `sort_direction` (asc/desc), `pagination_limit`, `pagination_cursor`.

### Get event by ID
```bash
curl -s "https://public-api.luma.com/v1/event/get?id=evt-XXXXX" \
  -H "x-luma-api-key: $LUMA_API_KEY"
```

### Create event
```bash
curl -s -X POST "https://public-api.luma.com/v1/event/create" \
  -H "x-luma-api-key: $LUMA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Monthly Meetup",
    "start_at": "2026-04-01T18:00:00Z",
    "end_at": "2026-04-01T20:00:00Z",
    "timezone": "America/New_York",
    "description_md": "Join us for our monthly meetup!",
    "visibility": "public",
    "meeting_url": "https://meet.google.com/abc-defg-hij"
  }'
```

Fields: `name` (required), `start_at` (required), `end_at` (required), `timezone` (required), `description_md`, `cover_url` (must be `https://images.lumacdn.com/...` — upload via `/v1/create-upload-url` first), `visibility` (public/members-only/private), `meeting_url`, `geo_address_json`, `coordinate`, `registration_questions`, `feedback_email`.

### Update event
```bash
curl -s -X POST "https://public-api.luma.com/v1/event/update" \
  -H "x-luma-api-key: $LUMA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "evt-XXXXX",
    "name": "Updated Event Name",
    "description_md": "New description"
  }'
```

### Cancel event
```bash
curl -s -X POST "https://public-api.luma.com/v1/event/cancel" \
  -H "x-luma-api-key: $LUMA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"id": "evt-XXXXX"}'
```

## Guests

### List guests for an event
```bash
# Get all guests (no status filter = all statuses)
curl -s -G "https://public-api.luma.com/v1/event/get-guests" \
  -H "x-luma-api-key: $LUMA_API_KEY" \
  --data-urlencode "event_id=evt-XXXXX"

# Filter by specific status
curl -s -G "https://public-api.luma.com/v1/event/get-guests" \
  -H "x-luma-api-key: $LUMA_API_KEY" \
  --data-urlencode "event_id=evt-XXXXX" \
  --data-urlencode "approval_status=waitlist"
```

Filter by `approval_status`: approved, session, pending_approval, invited, declined, waitlist. Omit `approval_status` to get ALL guests regardless of status — this is the right approach for total RSVP counts.

Sort by: name, email, created_at, registered_at, checked_in_at.

**Counting guests**: The API returns max 50 per page. To get a total count, you must paginate: check `has_more` in the response, and if true, pass `next_cursor` as `pagination_cursor` in the next request. Keep looping until `has_more` is false, summing `len(entries)` from each page.

### Get a single guest
```bash
curl -s "https://public-api.luma.com/v1/event/get-guest?event_id=evt-XXXXX&guest_id=GUEST_ID" \
  -H "x-luma-api-key: $LUMA_API_KEY"
```

### Add guests
```bash
curl -s -X POST "https://public-api.luma.com/v1/event/add-guests" \
  -H "x-luma-api-key: $LUMA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "event_id": "evt-XXXXX",
    "guests": [
      {"email": "jane@example.com", "name": "Jane Doe"}
    ]
  }'
```

Guests are added with status "Going" and one default ticket type. Use `ticket`/`tickets` param for custom ticket type assignments.

### Update guest status
```bash
curl -s -X POST "https://public-api.luma.com/v1/event/update-guest-status" \
  -H "x-luma-api-key: $LUMA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "event_id": "evt-XXXXX",
    "guest_id": "GUEST_ID",
    "approval_status": "approved"
  }'
```

Statuses: approved, session, pending_approval, invited, declined, waitlist.

### Send invites
```bash
curl -s -X POST "https://public-api.luma.com/v1/event/send-invites" \
  -H "x-luma-api-key: $LUMA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "event_id": "evt-XXXXX",
    "guest_ids": ["GUEST_ID_1", "GUEST_ID_2"]
  }'
```

Sends email and SMS (if phone linked to Luma account).

## Ticket Types

```bash
# List ticket types
curl -s "https://public-api.luma.com/v1/event/ticket-types/list?event_id=evt-XXXXX" \
  -H "x-luma-api-key: $LUMA_API_KEY"

# Create ticket type
curl -s -X POST "https://public-api.luma.com/v1/event/ticket-types/create" \
  -H "x-luma-api-key: $LUMA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "event_id": "evt-XXXXX",
    "name": "VIP",
    "price": 5000,
    "quantity": 50
  }'

# Update / Delete ticket type
curl -s -X POST "https://public-api.luma.com/v1/event/ticket-types/update" ...
curl -s -X POST "https://public-api.luma.com/v1/event/ticket-types/delete" ...
```

## Hosts

```bash
# Add host to event
curl -s -X POST "https://public-api.luma.com/v1/event/create-host" \
  -H "x-luma-api-key: $LUMA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"event_id": "evt-XXXXX", "user_id": "USER_ID"}'

# Update / Remove host
curl -s -X POST "https://public-api.luma.com/v1/event/update-host" ...
curl -s -X POST "https://public-api.luma.com/v1/event/remove-host" ...
```

## Coupons

```bash
# List event coupons
curl -s "https://public-api.luma.com/v1/event/list-coupons?event_id=evt-XXXXX" \
  -H "x-luma-api-key: $LUMA_API_KEY"

# Create event coupon
curl -s -X POST "https://public-api.luma.com/v1/event/create-coupon" \
  -H "x-luma-api-key: $LUMA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "event_id": "evt-XXXXX",
    "code": "EARLYBIRD",
    "discount_type": "percentage",
    "discount_amount": 20,
    "max_uses": 100
  }'

# Calendar-level coupons
curl -s "https://public-api.luma.com/v1/calendar/list-coupons" ...
curl -s -X POST "https://public-api.luma.com/v1/calendar/create-coupon" ...
curl -s -X POST "https://public-api.luma.com/v1/calendar/update-coupon" ...
```

## Calendar & People

### List people in calendar
```bash
curl -s "https://public-api.luma.com/v1/calendar/list-people" \
  -H "x-luma-api-key: $LUMA_API_KEY"
```

### Import people
```bash
curl -s -X POST "https://public-api.luma.com/v1/calendar/import-people" \
  -H "x-luma-api-key: $LUMA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "people": [
      {"email": "contact@example.com", "name": "New Contact"}
    ]
  }'
```

### Person tags
```bash
# List / Create / Update / Delete person tags
curl -s "https://public-api.luma.com/v1/calendar/list-person-tags" -H "x-luma-api-key: $LUMA_API_KEY"
curl -s -X POST "https://public-api.luma.com/v1/calendar/create-person-tag" ...
curl -s -X POST "https://public-api.luma.com/v1/calendar/update-person-tag" ...
curl -s -X POST "https://public-api.luma.com/v1/calendar/delete-person-tag" ...

# Apply / Remove tags from people
curl -s -X POST "https://public-api.luma.com/v1/calendar/apply-person-tags" ...
curl -s -X POST "https://public-api.luma.com/v1/calendar/remove-person-tags" ...
```

### Event tags
```bash
# List / Create / Update / Delete event tags
curl -s "https://public-api.luma.com/v1/calendar/list-event-tags" -H "x-luma-api-key: $LUMA_API_KEY"
curl -s -X POST "https://public-api.luma.com/v1/calendar/create-event-tag" ...
curl -s -X POST "https://public-api.luma.com/v1/calendar/update-event-tag" ...
curl -s -X POST "https://public-api.luma.com/v1/calendar/delete-event-tag" ...

# Apply / Remove tags from events
curl -s -X POST "https://public-api.luma.com/v1/calendar/apply-event-tags" ...
curl -s -X POST "https://public-api.luma.com/v1/calendar/remove-event-tags" ...
```

## Memberships

```bash
# List membership tiers
curl -s "https://public-api.luma.com/v1/membership/list-tiers" \
  -H "x-luma-api-key: $LUMA_API_KEY"

# Add member to tier
curl -s -X POST "https://public-api.luma.com/v1/membership/add-member" \
  -H "x-luma-api-key: $LUMA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"tier_id": "TIER_ID", "email": "member@example.com"}'

# Update member status
curl -s -X POST "https://public-api.luma.com/v1/membership/update-member-status" ...
```

## Webhooks

```bash
# List webhooks
curl -s "https://public-api.luma.com/v1/webhook/list" \
  -H "x-luma-api-key: $LUMA_API_KEY"

# Create webhook
curl -s -X POST "https://public-api.luma.com/v1/webhook/create" \
  -H "x-luma-api-key: $LUMA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-server.com/webhook",
    "event_types": ["guest.registered", "event.created"]
  }'

# Update / Delete webhook
curl -s -X POST "https://public-api.luma.com/v1/webhook/update" ...
curl -s -X POST "https://public-api.luma.com/v1/webhook/delete" ...
```

Event types: `event.created`, `event.updated`, `event.canceled`, `guest.registered`, `guest.updated`, `ticket.registered`, `calendar.event.added`, `calendar.person.subscribed`.

## Utilities

### Check auth
```bash
curl -s "https://public-api.luma.com/v1/user/get-self" \
  -H "x-luma-api-key: $LUMA_API_KEY"
```

### Upload image (for event covers)
```bash
# 1. Get presigned upload URL
curl -s -X POST "https://public-api.luma.com/v1/create-upload-url" \
  -H "x-luma-api-key: $LUMA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"file_name": "cover.jpg", "content_type": "image/jpeg"}'

# 2. Upload file to the returned URL (PUT)
# 3. Use the resulting https://images.lumacdn.com/... URL as cover_url
```

### Lookup entity
```bash
curl -s "https://public-api.luma.com/v1/lookup-entity?entity_type=event&entity_id=evt-XXXXX" \
  -H "x-luma-api-key: $LUMA_API_KEY"
```

## Common Workflows

### Get upcoming events with RSVP counts
1. List events with `after=<today>` to get upcoming events
2. For each event, call `get-guests` WITHOUT `approval_status` filter (gets all guests)
3. Paginate through all pages (check `has_more`, use `next_cursor`) to get accurate total
4. Present as a table: event name, date, total RSVPs

### Find most recent event and check its waitlist
1. List events with `before=<now>` and `sort_direction=desc` and `pagination_limit=1`
2. Take the first result — that's the most recent past event
3. Call `get-guests` with `approval_status=waitlist` for that event

### Get all people with their tags
1. Call `list-person-tags` to get all available tags
2. Call `list-people` with pagination to get all people
3. Each person entry includes their assigned tag IDs — match against the tags list

## Notes
- Cover images must be uploaded via `/v1/create-upload-url` first — only `https://images.lumacdn.com/...` URLs are accepted
- `calendar/list-events` only returns events **managed by** the calendar, not events merely listed on it
- Pagination: always check `has_more` and use `next_cursor` for subsequent pages — many endpoints return max 50 items per page
- All write operations use POST (even updates and deletes)
- When the user asks about "the latest event" or "most recent event", use `before` + `sort_direction=desc` to find it — don't rely on the default sort order
