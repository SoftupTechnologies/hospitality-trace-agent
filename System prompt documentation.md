# Hotel Operations Router – Scenario Documentation 


# Selected model

- gpt-5

## 1. What the Agent Reads

From each Apaleo webhook, the agent uses:

- **Booker Comment**  
- **Guest Comment**  
- **Reservation Comment**  
- **Extra Booking Comment**

And additional booking/reservation context:

- **Property ID**, **Property Name**  
- **Booking ID**, **Reservation ID**  
- **Room Type / Unit Group Name**  
- **Arrival Date/Time**, **Departure Date/Time**  
- **Channel** (booking source)  
- **Rate Plan** (`ratePlanCode`, `ratePlanName`, `ratePlanId`)  
- **Number of Adults** (`adults`)  
- **Number of Children** (via `childrenAges` array)  
- **Primary Guest Birthday** (`primaryGuestBirthday`)

### Deduplication  
If **Booker Comment** and **Guest Comment** contain the same text on initial booking creation, they are treated as **one comment** to avoid duplicate tasks.

- **Extra Booking Comment** and a **different Booker Comment** → count as **booking-level**  
- **Guest Comment** and **Reservation Comment** → count as **reservation-level**

---

## 2. Who Receives Tasks

Tasks are assigned using a `Department` array structure:

```json
"assigned_to": {
  "Department": ["Housekeeping", "Reception"]
}
```

### Valid Department Tags

- **Reception** — Guest communications, check-in/out, arrival notes, room preferences, dog amenities, parking, breakfast requests
- **Housekeeping** — Cleaning, room prep, towels, pillows, beds, pet in room
- **Technik** — Broken items, "kaputt", "defekt", technical issues

### Multi-Department Assignment

Some tasks require coordination between departments. When a task involves responsibilities from multiple roles, all relevant department tags are included:

```json
"assigned_to": {
  "Department": ["Housekeeping", "Reception"]
}
```

### Department Override Rules

The following tasks always use specific department combinations:

| Task Type | Departments |
|-----------|-------------|
| Extra bed ("Zustellbett vorbereiten") | `["Housekeeping", "Reception"]` |
| Baby bed ("Babybett vorbereiten") | `["Housekeeping", "Reception"]` |
| Room adjacency ("Zimmer nebeneinander") | `["Reception"]` |
| Early check-in | `["Housekeeping", "Reception"]` |
| Late checkout | `["Housekeeping", "Reception"]` |
| Parking request | `["Reception"]` |
| Booking extension | `["Housekeeping", "Reception"]` |

---

## 3. Scenario Overview  
### *From comment → canonical Sweeply task*

> Canonical task titles are in **German**.  
> Descriptions are generated in **English**.

---

### 3.1 Payment, Credit Card & Invoices — DISABLED

> ⚠️ **These are disabled trace domains.** No tasks are created for these intents.

**Typical comments (NO task created):**

- "Please charge my credit card"  
- "VKK belasten", "KK belasten"  
- "Send invoice to this email"  
- "Add my credit card"  
- "Is payment received?"

**Result:** No task output — these are hard-skipped by the system prompt.

---

### 3.2 Room Location & Allocation (Reception)

**Typical comments:**

- "Quiet room", "ruhiges Zimmer"  
- "High floor", "hohe Etage"  
- "Rooms next to each other"  
- "Nice view"

**Canonical tasks:**

- **„ruhiges Zimmer gewünscht"**  
- **„hohe Etage gewünscht"**  
- **„Zimmer nebeneinander"**  
- **„schöne Aussicht gewünscht"**

**Departments:** `["Reception"]`

---

### 3.3 Beds & Occupancy Logic

Uses: **adults**, **childrenAges**

#### Automatic Extra Bed Logic

Create **„Zustellbett vorbereiten"** when either condition matches:
- `adults >= 3`
- `adults >= 2` AND there exists a child age `> 2` in `childrenAges`

**Departments:** `["Housekeeping", "Reception"]`

#### Automatic Baby Bed Logic

Create **„Babybett vorbereiten"** when:
- `adults >= 1` AND there exists a child age `<= 2` in `childrenAges`

**Departments:** `["Housekeeping", "Reception"]`

**Note:** If both conditions match, create **both** tasks (they represent different operational actions).

#### Comment-Driven Bed Tasks

**Typical comments:**

- "Extra bed", "Aufbettung für Kind"  
- "Baby cot", "Babybett"  
- "Twin beds", "please separate beds"  
- "Push beds together"

**Canonical tasks:**

- **„Zustellbett vorbereiten"** — `["Housekeeping", "Reception"]`
- **„Babybett vorbereiten"** — `["Housekeeping", "Reception"]`
- **„Gast kontaktieren: Twin Bed nicht verfügbar"** — `["Reception"]` (twin beds are not supported)

---

### 3.4 Allergy, Special Bedding & Deep Cleaning (Housekeeping)

**Typical comments:**

- "No feathers"  
- "Allergic bedding needed"  
- "Special cleaning required"

**Canonical tasks:**

- **„Hat ASB" / „Ist ASB"**  
- **„Sonderreinigung / Allergiker"**  
- **„längliches Kopfkissen, nicht ganz so hoch"**

**Departments:** `["Housekeeping"]`

---

### 3.5 Pets & Dog Amenities

**Typical comments:**

- "Dog in room"  
- "Provide dog bowl and treats"

**Canonical tasks:**

- **„Hund im Zimmer"**  
- **„Hundenapf und Leckerli mitgeben"**

**Departments:**  
- `["Housekeeping"]` — Dog in room  
- `["Reception"]` — Dog amenities (bowl, treats)

---

### 3.6 Arrival Times & Dayuse

**Arrival comments:**

- "Arriving at 23:00"  
- "Early arrival"

**Canonical task:**

- **„Anreise: [Uhrzeit]"**  
  **Departments:** `["Reception"]`

---

**Dayuse comments:**  
(e.g., "Dayuse", "Dayuse room") or `ratePlanCode`/`ratePlanName` contains `DAYUSEBUSDBL2` or `DAYUSECOMDBL2`

> Note: Dayuse logic is only valid for `context = 'reservation'`

**Task created:**

- **Title:** "Dayuse booking: 09:00–14:00"  
- **Description:** "Handle dayuse booking from 09:00 to 14:00."  
- **Departments:** `["Housekeeping", "Reception"]`  
- **Due:** arrival date

Dayuse logic **never replaces** extra-bed/occupancy logic. If guest count triggers occupancy rules, both tasks are created.

---

### 3.7 Guest Birthday During Stay

If **primaryGuestBirthday** falls between **arrival** and **departure** (inclusive):

**Task:**

- **Title:** `Guest birthday`  
- **Description:** "The guest has a birthday during their stay."  
- **Departments:** `["Reception"]`  
- **Due:** `primaryGuestBirthday`

---

### 3.8 Guest Questions & Requests for Clarification

If comment contains "?", "can you…?", "is it possible…?":

**Task:**  
- **„Reply to guest inquiry by email"**

**Departments:** `["Reception"]`

---

### 3.9 Payer / Rate / Billing Structure — DISABLED

> ⚠️ **This is a disabled trace domain.** No tasks are created for these intents.

**Typical comments (NO task created):**

- "Company pays"  
- "Invoice to agency"  
- Rate abbreviations: BB, HB, FB, Corp, inkl., exkl.

**Result:** No task output — these are hard-skipped by the system prompt (falls under invoice/payer disabled domain).

---

### 3.10 Parking Request (Two-Stage)

If any comment indicates parking is needed (e.g., "parking", "Parkplatz", "garage"):

**Two tasks created:**

1. **"Parking request received"**  
   - Description: "Guest requests parking. Please check availability and reserve if applicable."  
   - Departments: `["Reception"]`  
   - Due: `now`

2. **"Send parking details to guest"**  
   - Description: "Send parking instructions/details to the guest for arrival."  
   - Departments: `["Reception"]`  
   - Due: `arrival`

---

### 3.11 Breakfast / Meal Plan Requested

If any comment explicitly requests breakfast or a meal plan:

**Task:**  
- **Title:** "Breakfast requested" or "Mealplan requested"  
- **Description:** "Guest requests breakfast to be included/added."  
- **Departments:** `["Reception"]`

> Note: Pure informational questions like "Is breakfast included?" are handled via the inquiry-email rule (Section 3.8) instead.

---

### 3.12 Follow-up Required (Safety Net)

If a comment clearly requires staff follow-up (confirmation, clarification, contacting the guest) and did not produce a more specific task:

**Task:**  
- **Title:** "Follow up required"  
- **Description:** "Guest comment requires follow-up. Review and respond/clarify as needed."  
- **Departments:** `["Reception"]`

> Guardrails: Not created if the comment already triggered "Reply to guest inquiry by email" or falls under disabled domains.

---

## 4. How Tasks Are Created, Updated or Skipped

The agent compares new instructions with **existing Sweeply trace logs** using:

- Booking ID  
- Reservation ID  
- Canonical Task Title  
- Description  
- Sweeply Status  
- Latest timestamp (`created_at`)

### Skip  
If an identical, non-failed task already exists → **no new task**.

### Create  
If no valid task with the same meaning exists → **create a new Sweeply task**.

### Update  
If the meaning changed (time, quantity, direction) → **update the latest relevant task**.

**Important:** Only ONE update per operational intent per run. The agent always selects the latest non-failed trace and updates only that one.

---

## 5. Booking-Level vs Reservation-Level Logic

### Reservation-level  
Tasks created only for that **specific Reservation ID**.

### Booking-level  
Tasks created **once per booking**, not per reservation.  
If a booking-level task already exists, it won't be created again for new reservations under the same booking.

---

## 6. Failed Tasks

If an existing Sweeply task has status **"failed"**, the agent treats it as if:

- The task does **not** exist  
- A new task **may be created**

---

## 7. Canonical Meaning Integrity

Canonical titles (e.g., "Hund im Zimmer", "ruhiges Zimmer gewünscht") are **one-directional**.

If a guest **cancels** or **reverses** such a request:

- Do **not** reuse the canonical title  
- Instead:  
  - Update the existing task **or**  
  - Create a non-canonical cancellation task

Examples:  
"Dog not coming anymore" → **not** "Hund im Zimmer"  
"Please remove extra bed" → **not** "Zustellbett vorbereiten"

---

## 8. Output Format (High-Level)

Each task contains:

- **Title** (German if canonical, English if non-canonical)  
- **Description** (English)  
- **assigned_to** — Object with `Department` array  
- **Priority** — Boolean (`true` = high, `false` = normal)  
- **Due date** — **Required** when arrival date exists (ISO 8601 datetime)  
- **Action:** `create` or `update`  
- **Sweeply Trace ID** (required for updates)

**Example task:**

```json
{
  "title": "Zustellbett vorbereiten",
  "description": "Prepare an extra bed due to occupancy.",
  "assigned_to": {
    "Department": ["Housekeeping", "Reception"]
  },
  "priority": false,
  "due": "2024-12-15T14:00:00Z",
  "action": "create"
}
```

All tasks are returned as a **JSON array** for Sweeply.
