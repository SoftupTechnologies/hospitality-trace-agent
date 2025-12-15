# Rulebook — Trace Cases (Hotel)

This document is a **standalone rule specification** for generating hotel “traces” (tasks) from booking/reservation data and free-text comments. It is designed to be **easy to extend**: add a new rule as a self-contained block.

## Scope & intent

- Convert booking context + comments into **actionable traces**.
- Be **deterministic and auditable**: rules are explicit, named, and testable.
- Support **incremental growth**: new trace cases can be added without rewriting existing ones.

## Inputs (minimum contract)

An implementation applying this rulebook should have access to:

- **Identifiers**: `bookingId`, `reservationId` (if available)
- **Dates**: `arrival` (ISO 8601 datetime when available)
- **Occupancy**: `adults` (number, when available), `childrenAges` (number array, when available)
- **Comments** (free-text sources; any subset may be present):
  - `bookerComment`
  - `extraBookingComment`
  - `guestComment`
  - `reservationComment`

## Output (task contract)

Each trace/task must follow this structure:

- `title` (string)
- `description` (string, optional but recommended)
- `assigned_to`: `{ "Departments": string[] }`
- `priority` (boolean)
- `due` (ISO 8601 datetime)
- `action`: `"create"` or `"update"`
- `sweeply_trace_id` (string, only when `action="update"`)

## Global processing principles

- **Split multi-intent comments**: one operational intent → one task.
- **Hard-disabled domains win**: if a domain is marked “disabled”, never emit tasks for that intent (no create, no update).
- **Deduplication**:
  - If an identical non-failed task already exists (same `bookingId` + `reservationId` + `title` + same `description` meaning), do not create a duplicate.
  - If your system supports updates and has task IDs, updates are allowed only when you are changing the **same operational intent**.
- **Due dates**:
  - `arrival` means the reservation arrival datetime.
  - `now` means the run timestamp (when the agent is executed).
  - If a rule requires `arrival` but it’s missing, set due to `now` and mention the missing arrival in the description.

## Definitions used in rules

- **Now**: the current run timestamp (execution time).
- **Arrival**: `arrival` datetime from the input payload (ISO 8601).
- **Child over/under 2**: evaluated by numeric ages from `childrenAges` when available; otherwise derived from explicit comment text if it states an age.

---
## Arrival / early check-in / late checkout (simple time-in-title rule)

### R-ARRIVAL-TIME-01 — Arrival time mentioned → include time in title

**Trigger:**
- Any comment clearly mentions an expected **arrival/check-in time** (and is not an early check-in request).

**Action:**
- Create or update a trace:
  - **title**: `Anreise: <time>` if a time is explicitly present in the comment; otherwise `Anreise: [Uhrzeit]`
  - **description**: `Guest reports expected arrival time.`
  - **assigned_to**: `{ "Departments": ["front-office"] }`
  - **due**: `arrival` (or `now` if arrival is missing per global due-date rule)

### R-EARLY-CHECKIN-01 — Early check-in requested → include time in title if present

**Trigger:**
- Comment requests **early check-in** (e.g., "early check-in", "früher Check-in", "früh einchecken").

**Action:**
- Create or update a trace:
  - **title**: `Early check-in requested` (append `: <time>` only if an explicit time is present in the comment)
  - **description**: `Guest requests early check-in.`
  - **assigned_to**: `{ "Departments": ["housekeeping", "front-office"] }`
  - **due**: `arrival` (or `now` if arrival is missing per global due-date rule)

### R-LATE-CHECKOUT-01 — Late checkout requested → include time in title if present

**Trigger:**
- Comment requests **late checkout** (e.g., "late checkout", "später Checkout", "spät auschecken").

**Action:**
- Create or update a trace:
  - **title**: `Late checkout requested` (append `: <time>` only if an explicit time is present in the comment)
  - **description**: `Guest requests late checkout.`
  - **assigned_to**: `{ "Departments": ["housekeeping", "front-office"] }`
  - **due**: `arrival` (or `now` if arrival is missing per global due-date rule)

## Disabled trace domains (do not create / do not update)

### R-DISABLE-CC — Credit card cases are disabled

**Intent examples (non-exhaustive):**
- “KK belasten”, “VKK belasten”, “KK / VKK”
- “Please charge my credit card”
- “Add my credit card”, “CC hinzufügen”
- “Payment received?”, “Zahlung prüfen”

**Rule:**
- If the detected operational intent is **credit card / payment / charging / CC adding / payment verification**, then **output no task** for that intent.
- Do not create or update any traces with titles such as:
  - `KK / VKK belasten`
  - `Zahlung prüfen / erfolgt?`
  - `CC hinzufügen`

**Result:** hard skip (no trace output for CC/payment intents).

### R-DISABLE-INVOICE — Invoice sending is disabled

**Intent examples (non-exhaustive):**
- “Send invoice to this email”
- “Rechnung bitte an …”, “Rechnung versenden”
- “Invoice to company”

**Rule:**
- If the detected operational intent is **sending invoices** (including “send invoice to email” or similar), then **output no task** for that intent.
- Do not create or update traces with titles such as:
  - `Rechnung versenden an: (email)`
  - (and any free-form equivalent about invoice sending)

**Result:** hard skip (no trace output for invoice-sending intents).

---

## Beds & occupancy rules

### R-BED-EXTRA-01 — Extra bed for 3 adults

**Condition:**
- `adults >= 3`

**Action:**
- Create trace:
  - **title**: `Zustellbett vorbereiten`
  - **description**: `Prepare an extra bed due to 3 adults.`
  - **assigned_to**: `{ "Departments": ["housekeeping", "front-office"] }`
  - **due**: `arrival`
  - **priority**: `false`
  - **action**: `create` (or skip if an identical non-failed trace already exists per the Deduplication principle)

### R-BED-EXTRA-02 — Extra bed for 2 adults + child over 2

**Condition:**
- `adults >= 2`
- AND there exists a child age `> 2` in `childrenAges`

**Action:**
- Create trace `Zustellbett vorbereiten` (same fields as R-BED-EXTRA-01)

### R-BED-BABY-01 — Baby bed for 2 adults + child under or equal 2

**Condition:**
- `adults >= 2`
- AND there exists a child age `<= 2` in `childrenAges`

**Action:**
- Create trace:
  - **title**: `Babybett vorbereiten`
  - **description**: `Place a baby bed (child age 2 or under).`
  - **assigned_to**: `{ "Departments": ["housekeeping", "front-office"] }`
  - **due**: `arrival`
  - **priority**: `false`
  - **action**: `create` (or skip if an identical non-failed trace already exists per the Deduplication principle)

**Notes:**
- If both “extra bed” and “baby bed” conditions match, create both traces (they represent different operational actions).

---

## Bed type support constraints

### R-TWIN-UNSUPPORTED-01 — Twin bed mentioned → contact guest (not supported)

**Trigger:**
- Any comment text mentions **twin bed** or equivalent phrasing, e.g.:
  - “twin bed”, “twin beds”, “2 single beds”
  - German variants like “Twin”, “zwei Einzelbetten”

**Rule:**
- The hotel does **not support twin beds**. Create a trace for Front Office to contact the guest and clarify alternatives.

**Action:**
- Create trace:
  - **title**: `Gast kontaktieren: Twin Bed nicht verfügbar`
  - **description**: `Guest requested twin bed(s), but twin beds are not supported. Contact guest to offer alternatives.`
  - **assigned_to**: `{ "Departments": ["front-office"] }`
  - **due**: `arrival`
  - **priority**: `false` (set `true` only if arrival is imminent according to your system’s urgency policy)
  - **action**: `create` (or skip if identical non-failed trace exists)

---

## Parking rules (two-stage traces)

Parking requests must produce **two distinct traces**:
1) one for **“request received”** due **now**
2) one for **“send parking details”** due **at arrival**

### R-PARKING-REQUEST-01 — Parking requested → create same-day trace

**Trigger:**
- Any comment indicates parking is needed / requested, e.g. “parking”, “Parkplatz”, “park”, “car”, “garage”.

**Action:**
- Create trace:
  - **title**: `Parking request received`
  - **description**: `Guest requests parking. Please check availability and reserve if applicable.`
  - **assigned_to**: `{ "Departments": ["front-office"] }`
  - **due**: `now`
  - **priority**: `false`
  - **action**: `create` (or skip if identical non-failed trace exists)

### R-PARKING-DETAILS-01 — Parking requested → create arrival-day trace

**Trigger:**
- Same trigger as R-PARKING-REQUEST-01 (parking requested).

**Action:**
- Create trace:
  - **title**: `Send parking details to guest`
  - **description**: `Send parking instructions/details to the guest for arrival.`
  - **assigned_to**: `{ "Departments": ["front-office"] }`
  - **due**: `arrival`
  - **priority**: `false`
  - **action**: `create` (or skip if identical non-failed trace exists)

---

## Breakfast rules

### R-BREAKFAST-REQUEST-01 — Breakfast requested → create/update trace

**Trigger:**
- Any comment explicitly requests breakfast / adding breakfast (English or German), e.g.:
  - "add breakfast", "with breakfast", "breakfast included please"
  - "Frühstück dazu", "mit Frühstück", "Frühstück hinzufügen"

**Do NOT trigger:**
- Pure informational questions like "Is breakfast included?" (treat as a general inquiry instead in your system).

**Action:**
- Create or update a trace:
  - **title**: `Breakfast requested`
  - **description**: `Guest requests breakfast to be included/added. Capture any specifics mentioned in the comment.`
  - **assigned_to**: `{ "Departments": ["front-office"] }`
  - **due**: `arrival` (or `now` if arrival is missing per global due-date rule)

---

## Follow-up safety net

### R-FOLLOWUP-REQUIRED-01 — Any comment requiring follow-up → create/update trace

**Purpose:**
Catch actionable guest comments that require staff follow-up but do not match a more specific rule.

**Trigger (non-exhaustive):**
- Explicit request for confirmation/clarification/contact/response, e.g.:
  - "Please confirm ...", "Bitte bestätigen ..."
  - "Let me/us know ...", "Bitte geben Sie uns Bescheid ..."
  - "Please advise", "Bitte um Rückmeldung"
  - "Contact me/us", "Bitte kontaktieren Sie mich/uns"
  - "Call me", "Rufen Sie mich an"

**Guardrails:**
- Do NOT emit this trace if the comment already produced a more specific trace for the same intent.
- Do NOT emit this trace if the intent is in a hard-disabled domain (credit card/payment or invoice/payer).
- Do NOT emit this trace if your system already emits a dedicated "inquiry reply" trace for this comment (avoid duplicates).

**Action:**
- Create or update a trace:
  - **title**: `Follow up required`
  - **description**: `Guest comment requires follow-up. Review and respond/clarify as needed.`
  - **assigned_to**: `{ "Departments": ["front-office"] }`
  - **due**: `arrival` (or `now` if arrival is missing per global due-date rule)

## Change log

- **2025-12-15**: Initial rulebook created with CC/invoice disablement, updated bed rules, twin-bed unsupported handling, and two-stage parking traces.


