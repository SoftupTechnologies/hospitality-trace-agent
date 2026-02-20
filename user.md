# Sweeply Task Generation (Rule-Driven)

Analyze the record and generate **Sweeply-ready tasks** based on the provided comments and structured fields.

All decisions about **whether a trace is created, updated, or skipped** must follow the **system prompt** and the **Rule Book tools**.

---

## REQUIRED TOOL CALL ORDER

Before generating any tasks, you must call tools in the following order:

1. **Get System Rule Book**  
2. **Get Property Rule Book** (pass `propertyId`)  
3. **Get Trace Logs**

The rule books are the **single source of truth** for:
- which operational intents create traces
- which domains are hard-disabled
- task titles, routing, priority, and due-date logic

---

## INPUT

context: `{{ $json.payload.context }}`  
now: `{{ $json.payload.now }}`  

propertyId: `{{ $json.payload.propertyId }}`  
propertyName: `{{ $json.payload.propertyName }}`  

bookingId: `{{ $json.payload.bookingId }}`  
reservationId: `{{ $json.payload.reservationId }}`  

primaryGuestBirthday: `{{ $json.payload.primaryGuestBirthday }}`  

unitId: `{{ $json.payload.unitId }}`  
unitName: `{{ $json.payload.unitName }}`  
unitGroupId: `{{ $json.payload.unitGroupId }}`  
unitGroupCode: `{{ $json.payload.unitGroupCode }}`  
unitGroupName: `{{ $json.payload.unitGroupName }}`  

arrival: `{{ $json.payload.arrival }}`  
departure: `{{ $json.payload.departure }}`  

channelCode: `{{ $json.payload.channelCode }}`  
ratePlanCode: `{{ $json.payload.ratePlanCode }}`  
ratePlanName: `{{ $json.payload.ratePlanName }}`  
ratePlanId: `{{ $json.payload.ratePlanId }}`  

adults: `{{ $json.payload.adults }}`  
childrenAges: `[{{ $json.payload.childrenAges }}]`

---

## COMMENTS

bookerComment (booking-level): `{{ $json.payload.bookerComment }}`  
extraBookingComment (booking-level): `{{ $json.payload.extraBookingComment }}`  
guestComment (reservation-level): `{{ $json.payload.guestComment }}`  
reservationComment (reservation-level): `{{ $json.payload.reservationComment }}`  

---

## PROCESSING REQUIREMENTS

- Use **only** the provided input fields; ignore empty or missing values.
- Apply **comment deduplication and scope assignment** exactly as defined in the system prompt.
- Split multi-intent comments into **separate operational intents**.
- Do **not** hardcode any task logic, titles, routing, or exclusions.
- Determine all trace creation and suppression strictly via:
  - **Get System Rule Book**
  - **Get Property Rule Book**

---

## RULE APPLICATION

- Evaluate **disabled rules first**  
  - If a disabled rule matches an intent, **do not create or update any trace** for that intent.
- Evaluate **enabled rules** next  
  - A trace may be created only if a matching enabled rule applies.
- Property-specific rules may **override or extend** system-wide rules.

If no rule matches an intent, **no trace is created**.

---

## LIFECYCLE DECISIONS

Use **Get Trace Logs** to determine for each eligible task whether to:

- **create** a new trace  
- **update** an existing trace  
- **skip** (identical non-failed trace already exists)

Follow system prompt rules strictly, including:
- booking-level vs reservation-level scope
- latest non-failed trace selection
- one update per operational intent per run
- failed traces are treated as non-existent

---

## OUTPUT REQUIREMENTS

- Output must be a **strict JSON array** of Sweeply task objects.
- No markdown, no explanations, no text outside the JSON array.
- Every task must include:
  - `"action"` (`"create"` or `"update"`)
  - `"title"`
  - `"description"`
  - `"assigned_to"`
  - `"due"`
  - `"priority"` (Low = false, High = true)
- If `"action": "update"`, include `"sweeply_trace_id"` from the correct existing trace.
- Apply **character safety rules** as defined in the system prompt.

---

### Final Note

The **Rule Books define what creates or suppresses traces**.  
This user prompt exists only to **pass data and enforce correct tool usage**.

No business logic should be inferred outside the rule books.
