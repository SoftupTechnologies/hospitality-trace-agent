# User Prompt - Sweeply Task Generation

Analyze the record and generate Sweeply-ready tasks based on the provided comments and structured fields
(comments, unit details, guest counts, and primaryGuestBirthday).

Before generating tasks, **call the “Get trace logs” tool** to retrieve existing past traces for this booking and reservation.  
Use these logs to determine whether each task should be **created**, **updated**, or **skipped**, following the system prompt rules.

## INPUT
context: `{{ $json.context }}`
now: `{{ $json.now }}`

propertyId: `{{$json.propertyId}}`  
propertyName: `{{$json.propertyName}}`  
reservationId: `{{$json.reservationId}}`  
bookingId: `{{$json.bookingId}}`  
bookerFirstName: `{{$json.bookerFirstName}}`  
bookerLastName: `{{$json.bookerLastName}}`  
primaryGuestBirthday: `{{$json.primaryGuestBirthday}}`  
primaryGuestFirstName: `{{$json.primaryGuestFirstName}}`
primaryGuestLastName: `{{$json.primaryGuestLastName}}`
unitId: `{{$json.unitId}}`  
unitName: `{{$json.unitName}}`  
unitGroupId: `{{$json.unitGroupId}}`  
unitGroupCode: `{{$json.unitGroupCode}}`  
unitGroupName: `{{$json.unitGroupName}}`  
arrival: `{{$json.arrival}}`  
departure: `{{$json.departure}}`  
channelCode: `{{$json.channelCode}}`  
ratePlanCode: `{{$json.ratePlanCode}}`  
ratePlanId: `{{$json.ratePlanId}}`  
adults: `{{$json.adults}}`  
childrenAges: [{{ $json.childrenAges }}]

## COMMENTS
bookerComment (booking-level): `{{$json.bookerComment}}`  
extraBookingComment (booking-level): `{{$json.extraBookingComment}}`  
guestComment (reservation-level): `{{$json.guestComment}}`  
reservationComment (reservation-level): `{{$json.reservationComment}}`  

## REQUIREMENTS
- Use **only** provided fields; ignore empty or missing ones.  
- Derive tasks from:
  - booking- and reservation-level comments, and  
  - structured fields (unit type, guest counts, primaryGuestBirthday, Dayuse keyword, etc.),  
  exactly as defined in the **system prompt**.
- Apply the system prompt logic fully, including:  
  - routing rules  
  - canonical title mapping  
  - semantic interpretation  
  - question-tone handling  
  - disabled trace domains (hard skip)  
  - follow-up safety-net (any comment requiring follow-up should produce a trace unless disabled / already covered)  
  - room-location requests  
  - twin-bed unsupported handling  
  - parking request handling (two-stage)  
  - Dayuse logic  
  - extra-bed occupancy logic  
  - birthday logic  
  - dog Reception vs Housekeeping split
- Split multi-intent comments into multiple tasks when required.  
- Use optional fields only when they exist in the input.  
- Apply due-date rules defined in the system prompt.  

### Hard-disabled domains (no create, no update)
Per the system prompt, do not output any task (no create, no update) for:
- Credit card / payment intents (charging, adding CC, payment verification, etc.)
- Invoice sending / payer intents (send invoice, invoice to email/company, payer corrections, etc.)

### Deterministic system tasks (create-or-skip only)
Follow the system prompt rules for **deterministic** tasks:

- Baby crib → “Babybett vorbereiten”  
- Extra bed → “Zustellbett vorbereiten”  
- Guest birthday → “Guest birthday”  
- Twin bed unsupported → “Gast kontaktieren: Twin Bed nicht verfügbar”  
- Parking request (two-stage) →  
  - “Parking request received”  
  - “Send parking details to guest”
- Dayuse →  
  - “Dayuse room cleaning after 14:00”  
  - “Dayuse booking: 09:00–14:00”

For these deterministic tasks:

- You may only **create** them if no identical non-failed task exists yet.  
- If an identical non-failed task already exists for the same bookingId + reservationId + title + description, **skip**.  
- You must **never output an `action: "update"`** for these tasks.

### Comment-driven tasks (may update)
For all other operational intents derived from comments:

- Use the trace logs to decide **create / update / skip** in line with the system prompt, including:
  - same operational intent  
  - scope (booking-level vs reservation-level)  
  - timeline rules  
  - “one update per operational intent per run”

Every task must include an `"action"` field (`"create"` or `"update"`).  
If updating, include `"sweeply_trace_id"` taken from the correct existing trace.

### OUTPUT
- Output must be a **strict JSON array** of task objects matching the Sweeply schema.  
- No markdown, no text outside the JSON array.