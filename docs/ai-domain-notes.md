# Domain notes: explaining the approach to business and domain experts

This document helps you explain the Casper's Kitchens demo to a **domain expert** or **Business Analyst (BA)** in business terms, show how it resembles reality, and draw **parallels to another domain** (e.g. healthcare) without going deep into technical implementation.

---

## 1. What the demo represents in the real world

### 1.1 The business setting

**Casper's** models a **ghost kitchen / delivery** operation:

- **Multiple brands** (e.g. Mexican, Asian, fast food) operate from shared **kitchen locations**.
- **Customers** place **orders** (items, quantity, price, delivery address).
- Each order goes through a **lifecycle**: placed → kitchen starts → kitchen finishes → ready for pickup → driver arrives → driver picks up → driver en route (optional pings) → **delivered**.
- After delivery, the business may **recommend a refund** (e.g. for late delivery) and may receive **customer complaints** that need a **response** (e.g. credit, escalation).

So in reality we have: **orders**, **locations**, **brands**, **menu items**, **delivery events**, **refund decisions**, and **complaint handling**.

### 1.2 What “resembles reality”

| Aspect | In the demo | In reality |
|--------|--------------|------------|
| **Order lifecycle** | Each order has a sequence of events (created → kitchen → driver → delivered) with timestamps. | Same: real systems record order placed, prep started/finished, driver assigned, picked up, delivered. |
| **Time and speed** | Events are replayed so that “simulation time” runs faster than wall-clock (e.g. 60×). | We don’t simulate in real time; we compress history so a demo can show hours of “business” in minutes. |
| **Refunds** | A “refund recommender” looks at delivered orders and suggests refund amount/class and reason (e.g. late by X minutes). | Mirrors real logic: compare delivery time to expectations (e.g. P75), suggest full/partial/none with reason. |
| **Complaints** | A share of delivered orders generate a “complaint” (category + text); an “agent” suggests category, decision (e.g. credit vs escalate), amount, and reply. | Mirrors real flow: some deliveries lead to complaints; ops or an agent triages and suggests response. |
| **Reference data** | Brands, locations, menus, categories, items, and which brands operate at which locations. | Same as real master data: brands, sites, menus, products, and site–brand mapping. |

### 1.3 What is simplified

- **No real customers or payments.** Orders and events are generated from a fixed dataset replayed in time; there are no live orders or payment systems.
- **Complaints are synthetic.** Complaint text is generated (e.g. by an LLM) from a sample of delivered orders, not from real customer feedback.
- **Refund “recommendations” are suggestions.** They are not automatically applied; the demo shows what a system could recommend (e.g. for a Refund Manager UI).
- **Single “company.”** One catalog of brands/locations/items; no multi-tenant or multi-org complexity.

These simplifications are useful to state upfront when discussing with a BA: *“We’re not simulating live payments or real customers; we’re simulating the event stream and the kinds of decisions your ops would see.”*

---

## 2. Domain entities and how they relate (business view)

Use this when a domain expert asks *“What things does the system care about?”*

- **Brand** – A virtual brand (e.g. a cuisine or concept) that has menu items and can operate at one or more locations.
- **Location** – A physical kitchen site where one or more brands operate.
- **Menu / category / item** – What can be sold: menu, category (e.g. drinks, mains), and individual items (name, price, brand).
- **Order** – One customer order: ID, location, list of line items (item, quantity, price), delivery address, and a **lifecycle** made of **events**.
- **Event** – Something that happened at a point in time to an order (e.g. order created, kitchen started, driver picked up, delivered). Events are ordered in time; “delivered” is the completion event.
- **Refund recommendation** – For a delivered order: suggested refund amount (or none), “class” (e.g. full/partial/none), and a short reason (e.g. “late by X minutes”).
- **Complaint** – Linked to an order: category (e.g. delivery delay, missing items, quality), complaint text, and optionally a **complaint response** (decision, credit amount, suggested reply).

**Relationships (in plain language):**

- An order belongs to one location and contains items from one or more brands.
- Each order has many events; the last meaningful one for “completion” is “delivered.”
- A refund recommendation is “one per delivered order” (the system suggests it).
- A complaint is “one per order” (we can have zero or one in the demo); a complaint response is the suggested handling for that complaint.

You can draw this as a simple diagram: **Brand ↔ Location** (many-to-many), **Order → Location**, **Order → Events**, **Order → Refund recommendation**, **Order → Complaint → Complaint response**.

---

## 3. What the system “does” (business outcomes)

Describe to a BA what the system *produces* without mentioning pipelines or tables:

1. **Continuous stream of “what happened.”**  
   Orders and their lifecycle events appear over time (as if new orders were coming in and being fulfilled). *If they ask: we replay a pre-built set of events on a schedule so it looks continuous.*

2. **Refund suggestions.**  
   For each delivered order, the system suggests whether to refund and how much, with a reason (e.g. based on delivery time vs expectations). These suggestions are the kind of thing an ops team or Refund Manager app would use to approve or reject.

3. **Complaint handling suggestions.**  
   For each (synthetic) complaint, the system suggests category, decision (e.g. offer credit vs escalate), amount, and a short customer-facing reply. Again, this is input to a person or tool that handles complaints.

4. **Aggregated views.**  
   The system maintains summaries such as: revenue and order counts by order; by item and day; by brand and day; by location and hour. So the business can answer “how did we do by brand/location/time?” without talking about “gold tables.”

5. **Operational use.**  
   Refund suggestions and complaint responses can be pushed into an operational database (e.g. for a Refund Manager UI or a complaints queue) so that what the “brain” recommends is what ops see and act on.

---

## 4. Discussing with a domain expert or BA

### 4.1 Good questions to ask them

- “Does this **order lifecycle** (placed → kitchen → driver → delivered) match how you track orders in production?”
- “Do you **recommend refunds** based on delivery time (e.g. vs a target or percentile)? Is that how you’d want to see it in a demo?”
- “How do you **categorize complaints** today (e.g. delivery delay, missing items, quality)? Do you triage (e.g. credit vs escalate)?”
- “What **reference data** do you have (brands, locations, menus, items, which brands at which locations)? Is that the same level of detail we need?”
- “Who **consumes** refund recommendations and complaint responses—ops team, an app, a CRM?”

### 4.2 What to show (without technical detail)

- **List of “things”:** orders, events, locations, brands, items, refund recommendations, complaints, complaint responses.
- **One order’s timeline:** event types and timestamps from “order created” to “delivered.”
- **Example refund recommendation:** order ID, suggested amount/class, reason (e.g. “late by 2 minutes”).
- **Example complaint and response:** category, complaint text (short), suggested decision and reply.
- **Simple diagram:** order → events → “delivered” → refund recommendation; order → complaint → complaint response.

### 4.3 How to phrase the “simulation”

- “We use a **fixed set of historical-style events** and **replay them in time** so that, in a short demo, you see many orders and deliveries as if they were happening live. No real payments or real customers are involved.”
- “**Complaints** are generated from a sample of delivered orders so we can show how the system would **triage and respond**; in production you’d replace this with real feedback.”

---

## 5. Drawing a parallel to another domain (e.g. healthcare)

Use this section to **map Casper’s to another domain** and have the same kind of conversation with a healthcare (or other) BA.

### 5.1 Entity mapping: Casper’s → Healthcare example

| Casper’s | Healthcare analogy (example) | Notes |
|----------|-----------------------------|--------|
| Brand | Payer, or care program, or product line | “Who” is associated with the encounter/claim. |
| Location | Facility, site, or clinic | Where the encounter happens. |
| Menu / category / item | Service catalog: department, procedure type, procedure (code, name, expected cost) | What can be “ordered” or billed. |
| Order | Encounter, or claim, or episode of care | One logical unit that has a lifecycle and can be completed. |
| Order created | Encounter start, or claim submitted, or admission | “It started.” |
| Kitchen / driver events | Intermediate clinical or admin steps (e.g. tests ordered, results back, referral) | Steps between start and completion. |
| Delivered | Discharge, or claim adjudicated, or episode end | “It’s done.” |
| Refund recommendation | Reimbursement adjustment, or denial/appeal suggestion, or quality-based adjustment | “We suggest this monetary or status outcome for this completed unit.” |
| Complaint | Patient feedback, grievance, or appeal request | “Customer” is unhappy; needs a response. |
| Complaint response | Grievance resolution, appeal decision, or patient reply | Suggested decision and response. |

You can substitute **retail (returns, feedback)**, **insurance (claims, disputes)**, or **field service (jobs, completion, customer feedback)** using the same pattern: **core transaction → lifecycle events → completion → recommendations and feedback handling**.

### 5.2 Event lifecycle mapping

- **Casper’s:** order_created → gk_started → gk_finished → gk_ready → driver_arrived → driver_picked_up → (driver_ping) → delivered.  
- **Healthcare (example):** encounter_opened → orders_placed → results_received → (optional steps) → encounter_closed / claim_submitted → claim_adjudicated.

The important idea: **a single “transaction” has a ordered sequence of events; one of them is “completed.”** For the demo, “completed” events (e.g. delivered, claim adjudicated) are the ones that trigger **recommendations** (refund / adjustment) and optionally **feedback** (complaint / grievance).

### 5.3 Outcomes and “what the system does” in the other domain

- **Refund recommendation** → **Adjustment / appeal recommendation:** “For this completed encounter/claim we suggest this adjustment or appeal decision, for this reason.”
- **Complaint + response** → **Grievance + resolution:** “For this feedback/grievance we suggest this category, decision (e.g. uphold, overturn, offer resolution), and reply.”
- **Aggregated views** → **Reporting:** “By facility, by product line, by day/week; volumes, amounts, and key metrics.”

You can say to a healthcare BA: *“Imagine the same flow: something starts, goes through steps, completes. When it’s complete we suggest an adjustment or appeal outcome; when there’s feedback we suggest how to triage and respond. The data model is the same shape—we just swap order/refund/complaint for encounter/adjustment/grievance.”*

### 5.4 Questions to ask in the other domain

- “What is the **one main thing** that has a lifecycle and a clear ‘completed’ state (e.g. encounter, claim, job)?”
- “What **events** do you record between start and completion? Can you list them in order?”
- “After completion, do you **recommend or decide** something (e.g. adjustment, denial, appeal)? What drives that (e.g. rules, timing, quality)?”
- “Do you get **feedback or disputes** (e.g. grievances, appeals)? How do you categorize and respond (e.g. uphold, overturn, offer resolution)?”
- “What **reference data** do you use (facilities, products, codes, payers)? Same level of detail as we have for brands/locations/items?”

---

## 6. One-page “reality vs demo” cheat sheet

Handy for a short alignment meeting:

| Topic | Reality | Demo |
|-------|---------|------|
| **Orders / transactions** | Real customers, real payments. | Simulated: fixed dataset replayed in time; no payments. |
| **Events** | Real system events (POS, delivery app, etc.). | Same event types and order; timestamps from replay. |
| **Refunds** | Business rules + ops approval. | System suggests amount and reason; no auto-apply. |
| **Complaints** | Real customer feedback. | Synthetic complaints from a sample of “delivered” orders. |
| **Responses** | Ops or agent writes reply. | System suggests category, decision, and reply text. |
| **Reference data** | Real brands, locations, menus. | Same structure; data from parquet/canonical set. |
| **Use** | Production decisions and reporting. | Shows how data flows and what recommendations look like; good for design and validation. |

---

This document stays at **domain and business level** so you can align with experts and BAs, then reuse the same structure (lifecycle, completion, recommendations, feedback) when you map to another domain.
