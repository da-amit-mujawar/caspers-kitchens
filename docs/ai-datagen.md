# Designing real-looking data generation: lessons from the canonical generator

This document teaches the **design and implementation** of the canonical data generator (simple replay) and the ideas behind the **canonical dataset** it consumes. The goal is to help your team design systems that produce **real-looking data** with realistic **temporal** and **random** behavior, and to do it in a **reusable, testable** way. Platform-specific details (e.g. Databricks paths) are downplayed; the focus is on logic, state, and programming patterns.

---

## 1. The two-phase design: why “canonical dataset” + “replay”

Many demos need a **continuous stream of events** that looks like production (orders, deliveries, sensor ticks, etc.). Two main approaches:

- **Live generator:** Code runs on a schedule and generates events in real time using randomness. Hard to reproduce, sensitive to clock and environment, and awkward to “rewind” or run faster than real time.
- **Replay from a fixed dataset:** A **canonical dataset** (e.g. 90 days of events in one parquet file) is prepared once. A **replay job** runs on a schedule and, each time, **advances a cursor** over that dataset and **writes only the “new” slice** into the stream. Time is **virtual**: “how far we are” in the dataset is determined by **elapsed wall-clock time × speed**, not by “now” inside the data.

The canonical generator (simple) uses the second approach. Benefits:

- **Reproducibility:** Same parquet → same sequence of events everywhere.
- **Portability:** One file (e.g. 34 MB for 90 days) can be shipped; no heavy runtime deps for generation.
- **Control:** Start at any “day,” run at 1×, 60×, or 3600× speed; “time travel” is just moving the cursor.
- **Reliability:** If the job fails, the next run reprocesses from the last known cursor (idempotent writes).

So the **design split** is:

1. **Offline (once):** Build a **canonical dataset** with realistic structure, temporal patterns, and randomness (e.g. `generate_canonical_dataset.py` → `events.parquet`).
2. **Online (scheduled):** A **replay** process that keeps **state** (how far we’ve written), computes **how far we should be** from wall-clock and speed, and **writes the next slice** of events, then **updates state**.

The rest of this note focuses on the **replay** (canonical_generator_simple) and, in parallel, the **ideas** used in the canonical dataset so you can replicate them elsewhere.

---

## 2. Core concepts for “real-looking” data

Before implementation details, three pillars:

### 2.1 Temporal realism

- **Ordering:** Events for one entity (e.g. order) must be **ordered in time** and follow a **lifecycle** (e.g. created → started → finished → delivered). The canonical dataset is generated that way; the replay preserves order by advancing a **single cursor** and writing contiguous time windows.
- **Rates and peaks:** In the dataset generator, “when” events occur is modeled (e.g. lunch/dinner peaks, day-of-week, per-location curves). The replay doesn’t invent new times; it **reveals** already-generated times at a chosen **speed** (e.g. 60× so 1 real minute = 1 sim hour).
- **Alignment with “today” (optional):** The replay can **shift** timestamps so that “day 70” in the dataset lines up with “today minus 70 days” on the calendar. That keeps demos “current” without changing the data.

### 2.2 Controlled randomness

- **Deterministic where it matters:** The **canonical dataset** is built with fixed seeds so the same run produces the same parquet. The **replay** is deterministic given state (watermark, sim_start) and parameters (START_DAY, SPEED_MULTIPLIER).
- **Random where it looks real:** In the dataset, order counts per minute (e.g. Poisson), service times (e.g. Gaussian), driver arrival (e.g. Beta), basket composition (e.g. 70% single brand, 30% multi-brand), and customer locations (sample from road network) are randomized. So the **content** is varied and realistic; the **replay** only slices it in time.

### 2.3 State and idempotency

- **Watermark:** A single number meaning “we have written all events with virtual time ≤ this.” Stored in a small file (or key) and updated only **after** a successful write. If the job fails before the update, the next run writes the **same** window again; if the sink is append-only, that’s safe (idempotent by design if downstream can dedupe, or you write exactly once).
- **Sim start:** Wall-clock time of the **first** run. Used so that “elapsed sim time” = (now − sim_start) × speed. Only set once so that restarts don’t reset the clock.

---

## 3. How the canonical generator (simple) works

We walk through the replay logic step by step. Assume a **90-day** canonical dataset: events in `events.parquet` with columns such as `ts_seconds`, `order_id`, `event_type_id`, `location_id`, `sequence`, and payload columns used to build `body`.

### 3.1 Configuration and constants

- **Parameters:** `START_DAY` (e.g. 70), `SPEED_MULTIPLIER` (e.g. 60.0). Optional: output path, state path (we abstract them).
- **Constants:** A fixed **epoch** (e.g. 2024-01-01 00:00:00) and **cycle length** = 90 days in seconds. So “virtual time” is measured in seconds since epoch; after 90 days we can **loop** and reuse the same 90 days again (with order IDs disambiguated, see below).

### 3.2 State: two values

1. **Watermark** = last virtual timestamp (in seconds) that we have **already written**. Read from a small file (e.g. single line). If missing → first run.
2. **Sim start** = wall-clock time of the first run. Read from another small file. If missing → use “now” as sim start and persist it after the first successful run.

So: **first run** = no watermark; **later runs** = watermark and sim start both set.

### 3.3 Position: where should we be “now”?

- **First run:** We want to “fast-forward” to **START_DAY** and current time-of-day, then start from there. So:
  - `new_end_seconds = epoch + (START_DAY × 86400) + (current hour × 3600 + minute × 60 + second)`  
  We write all events in the dataset with `ts_seconds` in `(epoch, new_end_seconds]`. No speed multiplier yet; we’re just establishing “we’re at day START_DAY, current time.”
- **Subsequent runs:** Elapsed **real** time since sim start = `now - sim_start`. Elapsed **sim** time = `elapsed_real × SPEED_MULTIPLIER`. The “start position” for the sim clock is:
  - `start_position = epoch + (START_DAY × 86400) + (sim_start’s time-of-day in seconds)`  
  Then:
  - `new_end_seconds = start_position + elapsed_sim`  
  So we only **write** events with virtual time in the **open interval (watermark, new_end_seconds]**.

If `new_end_seconds ≤ watermark`, there is nothing new to write; exit without writing.

### 3.4 Looping: reusing the 90-day window

The dataset has only 90 days. To run “forever,” we **loop**: virtual time keeps increasing, but we **map** it back to the same 90-day window over and over.

- **Loop index:** `loop_idx = (virtual_seconds - epoch) // cycle_seconds`. So loop 0 = first 90 days, loop 1 = next 90 days (reuse same data), etc.
- **Segment per loop:** For each `loop_idx` in `[start_loop, end_loop]` we define:
  - The **virtual** segment for this loop: `[max(watermark, loop_start_virtual), min(new_end_seconds, loop_end_virtual)]`.
  - The **dataset** segment: same length, but in the **base** 90-day window:  
    `segment_start_dataset = epoch + (segment_start_virtual - loop_start_virtual)`  
    and similarly for end. So we’re **slicing the same parquet** at different virtual positions.
- **Filter:** Take rows where `ts_seconds` is in `(segment_start_dataset, segment_end_dataset]`.
- **Virtual timestamp:** Assign `virtual_ts_seconds = ts_seconds + loop_idx × cycle_seconds` so that timestamps **increase monotonically** across loops.
- **Unique keys across loops:** To avoid duplicate order IDs when we reuse the same 90 days, **suffix** order_id with `-L{loop_idx}` for `loop_idx > 0` (e.g. `ABC123` → `ABC123-L1`). No schema change; just a string concat.

Collect all segments, concatenate, then transform and write.

### 3.5 Transform: from internal schema to output schema

The parquet has compact columns (e.g. `event_type_id`, `ts_seconds`, `items_json`, `customer_lat`, …). The replay produces a **stream schema**: e.g. `event_id`, `event_type` (string), `ts` (formatted string), `location_id`, `order_id`, `sequence`, `body` (JSON string).

- **event_type:** Map `event_type_id` (1..8) to strings (`order_created`, `gk_started`, …, `delivered`).
- **ts:** Format `virtual_ts_seconds` (plus optional **time shift**, see below) as `yyyy-MM-dd HH:mm:ss.SSS`.
- **body:** Build a JSON string per event type (e.g. for `order_created`: customer_lat, customer_lon, customer_addr, parsed `items_json`; for `driver_ping`: progress_pct, loc_lat, loc_lon; etc.). Other types get `{}` or minimal payload.
- **event_id:** Generate a new UUID per row (so each written event has a unique id in the stream).
- **Time shift (optional):** So that “day 70” in the dataset aligns with “today − 70 days” on the calendar:  
  `dataset_day_0 = today_midnight - timedelta(days=START_DAY)`  
  `TIME_SHIFT = (dataset_day_0 - epoch).total_seconds()`  
  Then use `virtual_ts_seconds + TIME_SHIFT` when formatting `ts`. This keeps demo dates “current” without changing the relative ordering or the 90-day cycle.

### 3.6 Write and state update

- **Write** the transformed dataframe to the sink (e.g. append JSON files to a directory). No overwrite of previously written data.
- **Update watermark:** Set watermark = `new_end_seconds` (overwrite the watermark file). This is the **only** place the cursor advances. If the job crashes after write but before this, the next run will reprocess the same window; if the sink is append-only, you may get duplicates unless the consumer dedupes (e.g. by event_id or order_id + sequence).
- **Update sim_start:** Only on the **first** run, after a successful write, persist `sim_start` so future runs can compute elapsed time correctly.

That’s the full loop: **read state → compute new position → slice dataset (with loop mapping) → transform → write → update state.**

---

## 4. How the canonical dataset gets its “real look” (design ideas)

The **replay** doesn’t create new events; it only **slices and maps** the canonical dataset. The “real-looking” character comes from how that dataset was generated. Below are the main **design and implementation** ideas (from `generate_canonical_dataset.py` and README); you can reuse them in other domains.

### 4.1 Temporal structure

- **Lifecycle:** Each order has a **fixed sequence** of event types (created → started → finished → ready → driver_arrived → picked_up → [pings] → delivered). Timestamps are **computed in order**: each step uses the previous step’s time plus a **random duration** (e.g. Gaussian). So ordering and causality are always valid.
- **Peaks and seasonality:** Minute-of-day weights (e.g. lunch 11:00–13:30, dinner 17:00–20:00) and day-of-week multipliers (e.g. Fri/Sat higher). One location can have a different pattern (e.g. late-night spike). Orders per minute are then drawn from **Poisson(λ)** with λ proportional to those weights, so counts vary realistically.
- **Growth/decline:** Per-location “base orders per day” and “daily growth rate.” Orders for day d = base × (1 + growth)^d × day_of_week_mult × noise. So some locations grow, some stay flat, some decline over the 90 days.

### 4.2 Randomness with the right distribution

- **Service times:** Gaussian(mean, std) per stage (e.g. 2±1 min, 10±3 min), with a minimum (e.g. 0.1 min) to avoid negatives.
- **Driver arrival:** Beta(α, β) to place the arrival between “order created” and “ready,” or “ready” and “pickup,” so it’s not uniform and not deterministic.
- **Basket:** 70% single brand / 30% 2–3 brands; brand weights from growth trajectory; 1–3 items per brand, qty 1–2. So baskets are varied but structured.
- **Customer location:** Sampled from **real** addressable nodes on an OSM road network; route and drive time from shortest path. So geography and drive times are plausible.

### 4.3 Identifiers and uniqueness

- **Order IDs:** Random 6-character alphanumeric; reject until unique across the whole dataset. In replay, when looping, suffix `-L1`, `-L2` so the same logical “order” in loop 1 and loop 2 don’t collide.

### 4.4 Payload richness by event type

- **order_created:** Customer lat/lon, address, full item list (id, name, price, qty, etc.).
- **driver_picked_up:** Full route (list of lat/lon) from routing.
- **driver_ping:** Interpolated position and progress % along the route, every 60 s.
- **delivered:** Customer lat/lon again.

So downstream can “see” a full story without inventing data at replay time.

---

## 5. Programming and design lessons

### 5.1 Single source of truth for “what happened”

The canonical parquet is the **only** definition of the event sequence. The replay doesn’t generate new events; it only **selects** a time window and **maps** schema and time. That keeps semantics clear and avoids drift between “generator” and “replay.”

### 5.2 Watermark as the only moving cursor

One scalar (watermark) is enough for a **strictly increasing** virtual time. No need to store “which files” or “which rows”; “up to this timestamp” is sufficient. Stored in a cheap, overwrite-only way (e.g. single file or key).

### 5.3 First run vs subsequent runs

- **First run:** Establish “where the sim starts” (START_DAY + time-of-day) and write everything up to that point; set sim_start.
- **Later runs:** Advance by (real elapsed) × (speed). Clean separation avoids special cases in the main loop.

### 5.4 Loop = modulo in time, disambiguate in keys

Virtual time grows unbounded; the dataset is finite. So: **virtual_time → (loop_index, time_in_cycle)**. Use **loop_index** to slice the same 90-day window and to **suffix** entity IDs so that the same logical entity in different loops don’t collide. No need to duplicate the dataset.

### 5.5 Idempotency and failure

If you **update the watermark only after a successful write**, then a crash after write but before update causes the **same** window to be written again next time. With append-only output, that’s **at-least-once**. If the consumer can dedupe (e.g. by event_id or (order_id, sequence)), you get effectively once. Designing for “replay same window safely” keeps the system simple and robust.

### 5.6 Schema mapping in one place

Internal schema (compact, parquet-friendly) → output schema (stream-friendly, e.g. event_type string, ts string, body JSON) in a **single transform** step. That keeps the replay code agnostic of how the dataset was generated; only the column names and types need to match.

### 5.7 Parameters over hardcoding

START_DAY, SPEED_MULTIPLIER, and (in the dataset) things like service-time means, growth rates, and seeds are **parameters**. So the same code can produce “start at day 0,” “start at day 70,” “run at 1× or 60×,” and different “narratives” (e.g. one location growing, one declining) without code changes.

---

## 6. Teaching checklist for your team

When teaching “how to design real-looking data and replay it,” you can walk through:

1. **Two-phase design:** Canonical dataset (offline, reproducible) + replay (online, stateful, scheduled). Why this beats “generate on the fly” for demos and portability.
2. **State:** Watermark (how far we’ve written) and sim_start (when we started). How they’re read/written and why order of operations (write → then update watermark) matters.
3. **Position:** First run = fast-forward to START_DAY + time-of-day; later runs = start_position + (elapsed_real × speed). How that yields a single `new_end_seconds` and a window `(watermark, new_end_seconds]`.
4. **Looping:** Virtual time modulo cycle → loop index; same 90-day slice reused; virtual_ts = dataset_ts + loop × cycle; order_id suffix for uniqueness.
5. **Transform:** Internal columns → output event_type, ts, body, event_id. Optional time shift for calendar alignment.
6. **Dataset design (for realism):** Lifecycle ordering, time distributions (Gaussian, Beta, Poisson), peaks and growth, unique IDs, rich payloads per event type. How these ideas carry to other domains (e.g. healthcare encounters, insurance claims, sensor streams).

You can implement a **minimal replay** in any language (Python, Scala, etc.) with: one parquet (or CSV) of events with a numeric timestamp column, a watermark file, a sim_start file, and the same formulas for position and looping. The creative part is in the **canonical dataset** design (temporal and random structure); the **replay** is a small, deterministic state machine that “plays back” that design at a chosen speed.
