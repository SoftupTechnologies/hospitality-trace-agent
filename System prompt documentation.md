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
- **Number of Children**  
- **Booker Birthday**

### Deduplication  
If **Booker Comment** and **Guest Comment** contain the same text on initial booking creation, they are treated as **one comment** to avoid duplicate tasks.

- **Extra Booking Comment** and a **different Booker Comment** → count as **booking-level**  
- **Guest Comment** and **Reservation Comment** → count as **reservation-level**

---

## 2. Who Receives Tasks

Every task is routed to **exactly one** hotel role:

- **Property Admin** → `["admin","system"]`  
- **Reservation Manager** → `["reservations","allocations"]`  
- **Rezeptionist/in** → `["frontdesk","guest"]`  
- **Senior Rezeptionist/in** → `["frontdesk","senior"]`  
- **Housekeeping** → `["housekeeping","rooms"]`

---

## 3. Scenario Overview  
### *From comment → canonical Sweeply task*

> Canonical task titles are in **German**.  
> Descriptions are generated in **English**.

---

### 3.1 Payment, Credit Card & Invoices (Front Office)

**Typical comments:**

- “Please charge my credit card”  
- “VKK belasten”, “KK belasten”  
- “Send invoice to this email”  
- “Add my credit card”  
- “Is payment received?”

**Canonical tasks:**

- **„KK / VKK belasten“**  
- **„Zahlung prüfen / erfolgt?“**  
- **„Rechnung versenden an: (email)“**  
- **„CC hinzufügen“**

**Role:** Rezeptionist/in

---

### 3.2 Room Location & Allocation (Front Office / Reservation Manager)

**Typical comments:**

- “Quiet room”, “ruhiges Zimmer”  
- “High floor”, “hohe Etage”  
- “Rooms next to each other”  
- “Nice view”

**Canonical tasks:**

- **„ruhiges Zimmer gewünscht“**  
- **„hohe Etage gewünscht“**  
- **„Zimmer nebeneinander“**  
- **„schöne Aussicht gewünscht“**

---

### 3.3 Beds & Occupancy Logic

Uses: **Adults**, **Children**, **Room Type**

**Automatic extra bed logic:**  
If total guests > capacity → create:

- **„Zustellbett vorbereiten“**  
  **Role:** Housekeeping

**Typical comments:**

- “Extra bed”, “Aufbettung für Kind”  
- “Baby cot”, “Babybett”  
- “Twin beds”, “please separate beds”  
- “Push beds together”

**Canonical tasks:**

- **„Zustellbett vorbereiten“**  
- **„Aufbettung für Kind“**  
- **„Babybett vorbereiten“**  
- **„Bettentyp vorbereiten (Twin / King)“**  
- **„Betten zusammenstellen / trennen“**

---

### 3.4 Allergy, Special Bedding & Deep Cleaning (Housekeeping)

**Typical comments:**

- “No feathers”  
- “Allergic bedding needed”  
- “Special cleaning required”

**Canonical tasks:**

- **„Hat ASB“ / „Ist ASB“**  
- **„Sonderreinigung / Allergiker“**  
- **„längliches Kopfkissen, nicht ganz so hoch“**

---

### 3.5 Pets & Dog Amenities

**Typical comments:**

- “Dog in room”  
- “Provide dog bowl and treats”

**Canonical tasks:**

- **„Hund im Zimmer“**  
- **„Hundenapf und Leckerli mitgeben“**

**Roles:**  
- Housekeeping (dog in room)  
- Rezeptionist/in (dog amenities)

---

### 3.6 Arrival Times & Dayuse

**Arrival comments:**

- “Arriving at 23:00”  
- “Early arrival”

**Canonical task:**

- **„Anreise: [Uhrzeit]“**  
  **Role:** Rezeptionist/in

---

**Dayuse comments:**  
(e.g., “Dayuse”, “Dayuse room”, “Dayuse RO/SZ…”)

**Tasks created:**

1. **Housekeeping**  
   - *Dayuse cleaning after stay*

2. **Front Office**  
   - *Dayuse booking: 10:00–17:00*

Dayuse logic **never replaces** extra-bed logic.

---

### 3.7 Guest Birthday During Stay

If **Booker Birthday** falls between **Arrival** and **Departure**:

**Task:**

- **Title:** `Guest birthday`  
- **Description:** “The guest has a birthday during their stay.”  
- **Role:** Rezeptionist/in

---

### 3.8 Guest Questions & Requests for Clarification

If comment contains “?”, “can you…?”, “is it possible…?”:

**Task:**  
- **„Reply to guest inquiry by email“**

**Role:** Rezeptionist/in

---

### 3.9 Payer / Rate / Billing Structure

**Typical comments:**

- “Company pays”  
- “Invoice to agency”  
- Rate abbreviations: BB, HB, FB, Corp, inkl., exkl.

**Task:**  
- **“Send invoice to correct payer”**

**Role:** Rezeptionist/in or Reservation Manager

---

## 4. How Tasks Are Created, Updated or Skipped

The agent compares new instructions with **existing Sweeply trace logs** using:

- Booking ID  
- Reservation ID  
- Canonical Task Title  
- Description  
- Sweeply Status  
- Latest timestamp

### Skip  
If an identical, non-failed task already exists → **no new task**.

### Create  
If no valid task with the same meaning exists → **create a new Sweeply task**.

### Update  
If the meaning changed (time, quantity, direction) → **update the latest relevant task**.

---

## 5. Booking-Level vs Reservation-Level Logic

### Reservation-level  
Tasks created only for that **specific Reservation ID**.

### Booking-level  
Tasks created **once per booking**, not per reservation.  
If a booking-level task already exists, it won’t be created again for new reservations under the same booking.

---

## 6. Failed Tasks

If an existing Sweeply task has status **“failed”**, the agent treats it as if:

- The task does **not** exist  
- A new task **may be created**

---

## 7. Canonical Meaning Integrity

Canonical titles (e.g., “Hund im Zimmer”, “ruhiges Zimmer gewünscht”) are **one-directional**.

If a guest **cancels** or **reverses** such a request:

- Do **not** reuse the canonical title  
- Instead:  
  - Update the existing task **or**  
  - Create a non-canonical cancellation task

Examples:  
“Dog not coming anymore” → **not** “Hund im Zimmer”  
“Please remove extra bed” → **not** “Zustellbett vorbereiten”

---

## 8. Output Format (High-Level)

Each task contains:

- **Title** (German if canonical, English if non-canonical)  
- **Description** (English)  
- **Assigned role** and tags  
- **Priority**  
- **Due date** (if applicable)  
- **Action:** `create` or `update`  
- **Sweeply Trace ID** (for updates)

All tasks are returned as a **JSON array** for Sweeply.