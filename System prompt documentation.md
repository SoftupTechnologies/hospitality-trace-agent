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
- **Rate Plan**  
- **Number of Adults**  
- **Number of Children** (via `childrenAges` array)  
- **Booker Birthday**

### Deduplication  
If **Booker Comment** and **Guest Comment** contain the same text on initial booking creation, they are treated as **one comment** to avoid duplicate tasks.

- **Extra Booking Comment** and a **different Booker Comment** → count as **booking-level**  
- **Guest Comment** and **Reservation Comment** → count as **reservation-level**

---

## 2. Who Receives Tasks

Tasks are assigned using a `Departments` array structure:

```json
"assigned_to": {
  "Departments": ["housekeeping", "front-office"]
}
```

### Valid Department Tags

- **property-admin** — Maintenance, equipment, infrastructure, broken items ("kaputt", "defekt")
- **reservation-manager** — Bookings, rate corrections, payer responsibilities, billing structures
- **front-office** — Guest communications, check-in/out, arrival notes, room preferences, CC changes, dog amenities
- **senior-front-office** — VIP guests, complaints, escalations
- **housekeeping** — Cleaning, room prep, towels, pillows, beds, pet in room

### Multi-Department Assignment

Some tasks require coordination between departments. When a task involves responsibilities from multiple roles, all relevant department tags are included:

```json
"assigned_to": {
  "Departments": ["housekeeping", "front-office"]
}
```

### Department Override Rules

The following tasks always use specific department combinations:

| Task Type | Departments |
|-----------|-------------|
| Extra bed ("Zustellbett vorbereiten") | `["housekeeping", "front-office"]` |
| Baby bed ("Babybett vorbereiten") | `["housekeeping", "front-office"]` |
| Room adjacency ("Zimmer nebeneinander") | `["front-office"]` |
| Early check-in | `["housekeeping", "front-office"]` |
| Late checkout | `["housekeeping", "front-office"]` |
| Parking request | `["front-office"]` |
| Booking extension | `["housekeeping", "front-office"]` |

---

## 3. Scenario Overview  
### *From comment → canonical Sweeply task*

> Canonical task titles are in **German**.  
> Descriptions are generated in **English**.

---

### 3.1 Payment, Credit Card & Invoices (Front Office)

**Typical comments:**

- "Please charge my credit card"  
- "VKK belasten", "KK belasten"  
- "Send invoice to this email"  
- "Add my credit card"  
- "Is payment received?"

**Canonical tasks:**

- **„KK / VKK belasten"**  
- **„Zahlung prüfen / erfolgt?"**  
- **„Rechnung versenden an: (email)"**  
- **„CC hinzufügen"**

**Departments:** `["front-office"]`

---

### 3.2 Room Location & Allocation (Front Office)

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

**Departments:** `["front-office"]`

---

### 3.3 Beds & Occupancy Logic

Uses: **Adults**, **childrenAges**, **Room Type (unitGroupName)**

**Automatic extra bed logic:**  
If effective_guest_count > capacity → create:

- **„Zustellbett vorbereiten"**  
  **Departments:** `["housekeeping", "front-office"]`

**Note:** Babies (age ≤ 2) do not count toward room capacity. Only adults and children over age 2 are counted for extra-bed calculations (`effective_guest_count`).

**Typical comments:**

- "Extra bed", "Aufbettung für Kind"  
- "Baby cot", "Babybett"  
- "Twin beds", "please separate beds"  
- "Push beds together"

**Canonical tasks:**

- **„Zustellbett vorbereiten"** — `["housekeeping", "front-office"]`
- **„Aufbettung für Kind"** — `["housekeeping", "front-office"]`
- **„Babybett vorbereiten"** — `["housekeeping", "front-office"]`
- **„Bettentyp vorbereiten (Twin / King)"** — `["housekeeping"]`
- **„Betten zusammenstellen / trennen"** — `["housekeeping"]`

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

**Departments:** `["housekeeping"]`

---

### 3.5 Pets & Dog Amenities

**Typical comments:**

- "Dog in room"  
- "Provide dog bowl and treats"

**Canonical tasks:**

- **„Hund im Zimmer"**  
- **„Hundenapf und Leckerli mitgeben"**

**Departments:**  
- `["housekeeping"]` — Dog in room  
- `["front-office"]` — Dog amenities (bowl, treats)

---

### 3.6 Arrival Times & Dayuse

**Arrival comments:**

- "Arriving at 23:00"  
- "Early arrival"

**Canonical task:**

- **„Anreise: [Uhrzeit]"**  
  **Departments:** `["front-office"]`

---

**Dayuse comments:**  
(e.g., "Dayuse", "Dayuse room", "Dayuse RO/SZ…")

**Tasks created:**

1. **Housekeeping** (`["housekeeping"]`)  
   - *"Dayuse room cleaning after 14:00"*  
   - Due: arrival_date @ 14:00

2. **Front Office** (`["front-office"]`)  
   - *"Dayuse booking: 09:00–14:00"*  
   - Due: arrival date

Dayuse logic **never replaces** extra-bed logic. These are always created as two separate tasks.

---

### 3.7 Guest Birthday During Stay

If **Booker Birthday** falls between **Arrival** and **Departure**:

**Task:**

- **Title:** `Guest birthday`  
- **Description:** "The guest has a birthday during their stay."  
- **Departments:** `["front-office"]`

---

### 3.8 Guest Questions & Requests for Clarification

If comment contains "?", "can you…?", "is it possible…?":

**Task:**  
- **„Reply to guest inquiry by email"**

**Departments:** `["front-office"]`

---

### 3.9 Payer / Rate / Billing Structure

**Typical comments:**

- "Company pays"  
- "Invoice to agency"  
- Rate abbreviations: BB, HB, FB, Corp, inkl., exkl.

**Task:**  
- **"Send invoice to correct payer"**

**Departments:** `["front-office"]` or `["reservation-manager"]`

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
- **assigned_to** — Object with `Departments` array  
- **Priority** — Boolean (`true` = high, `false` = normal)  
- **Due date** — **Required** when arrival date exists (ISO 8601 datetime)  
- **Action:** `create` or `update`  
- **Sweeply Trace ID** (required for updates)

**Example task:**

```json
{
  "title": "Zustellbett vorbereiten",
  "description": "Prepare extra bed for guest.",
  "assigned_to": {
    "Departments": ["housekeeping", "front-office"]
  },
  "priority": false,
  "due": "2024-12-15T14:00:00Z",
  "action": "create"
}
```

All tasks are returned as a **JSON array** for Sweeply.
