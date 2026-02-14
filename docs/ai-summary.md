# Casper's Kitchens – Architecture Summary

This document describes how the **caspers-kitchens** project generates data dynamically, stores it in Databricks tables, and how pipelines, jobs, and external systems relate. Use it as a blueprint to design a similar demo in another domain (e.g. healthcare).

**Default values** (from `databricks.yml` and bundle default target): catalog **caspersdev**, schema **simulator**, volume **events**, **START_DAY** 70, **SPEED_MULTIPLIER** 60.0, **SCHEDULE_MINUTES** 3, **PIPELINE_SCHEDULE_MINUTES** 0 (continuous). Lakebase instance names: **caspersdevrefundmanager**, **caspersdevcomplaintmanager**. Refund agent **caspers_refund_agent**, complaint agent **caspers_complaint_agent**, **COMPLAINT_RATE** 0.15.

---

## 1. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        DYNAMIC DATA GENERATION (scheduled)                       │
│  Canonical Data Replay Job (every 3 minutes)                                       │
│  → Reads events.parquet, replays by virtual time, writes JSON to UC Volume      │
└─────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  UC Volume: /Volumes/caspersdev/simulator/events                                   │
│  Format: One JSON file per event (Cloud Files / Auto Loader source)              │
└─────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Spark Declarative Pipeline (Lakeflow) – Order Items                             │
│  Mode: Continuous (default) or triggered every N min                              │
│  Bronze → Silver → Gold medallion in caspersdev.lakeflow                          │
└─────────────────────────────────────────────────────────────────────────────────┘
                                          │
                    ┌─────────────────────┼─────────────────────┐
                    ▼                     ▼                     ▼
┌───────────────────────┐  ┌───────────────────────┐  ┌───────────────────────┐
│ Refund recommender    │  │ Complaint generator    │  │ Complaint agent        │
│ (scheduled job 10min) │  │ (scheduled job 10min)  │  │ (scheduled job 10min)  │
│ → refund_recommendations │ → raw_complaints       │  │ → complaint_responses  │
└───────────────────────┘  └───────────────────────┘  └───────────────────────┘
                    │                     │                     │
                    ▼                     ▼                     ▼
┌───────────────────────┐  ┌───────────────────────┐
│ Lakebase (Reverse ETL) │  │ Complaint Lakebase     │
│ refund_recommendations │  │ complaint_responses    │
│ → Postgres (refund mgr)│  │ → Postgres (complaints)│
└───────────────────────┘  └───────────────────────┘
                    │
                    ▼
┌───────────────────────┐
│ Databricks App         │
│ (Refund Manager UI)    │
└───────────────────────┘
```

---

## 2. Dynamic Data Generation (How Data Is Produced at Intervals)

### 2.1 Mechanism

- **No live simulator in the default path.** Data comes from a **canonical dataset** (pre-generated `events.parquet`) and is **replayed** at a configurable speed.
- A **scheduled job** runs a notebook that:
  1. Reads **state** (watermark, sim start time) from files in a UC volume.
  2. Computes the **next time window** to replay (based on real elapsed time × speed multiplier).
  3. Filters `events.parquet` to that window, optionally **looping** a 90-day dataset.
  4. Transforms events to the **output JSON schema** (event_type, ts, body, etc.).
  5. **Writes** new events as JSON into the events volume and **updates** the watermark.

### 2.2 Key Components

| Component | Role |
|----------|------|
| **Canonical_Data** (init stage) | Creates catalog/schema/volumes; loads dimensional parquet into `caspersdev.simulator.*`; **creates the scheduled job** for replay. |
| **Canonical Data Replay (Simple)** notebook | `data/canonical/canonical_generator_simple.ipynb` – run by that job every **3 minutes**. |
| **Parameters** | `CATALOG` (caspersdev), `VOLUME` (events), `SCHEMA` (simulator), `START_DAY` (70), `SPEED_MULTIPLIER` (60.0), `SCHEDULE_MINUTES` (3). |
| **State files** | Stored under `caspersdev.simulator.misc` volume: `_watermark` (last processed virtual sim timestamp), `_sim_start` (first run wall-clock time). |

### 2.3 Replay Logic (Simplified)

- **First run:** Replay from day 0 up to `START_DAY` + current time-of-day.
- **Later runs:** `new_end_seconds = start_position + (real_elapsed_seconds × SPEED_MULTIPLIER)`; only events in `(watermark, new_end_seconds]` are written.
- **Looping:** Dataset is 90 days; after that, the same 90 days can be replayed again with `order_id` suffixed (e.g. `-L1`, `-L2`) to keep keys unique.
- **Exactly-once / idempotency:** If the job fails before updating the watermark, the next run reprocesses the same window (writes are append-only to the volume).

### 2.4 Where Generated Data Lands

- **Path:** `/Volumes/caspersdev/simulator/events/`
- **Format:** One JSON file per event (e.g. Spark `write.json()` append).
- This path is the **source** for the Lakeflow pipeline (Cloud Files / Auto Loader).

---

## 3. Input and Output Data Formats

### 3.1 Canonical Dataset (Input to Replay)

- **Location:** `data/canonical/canonical_dataset/` (e.g. `events.parquet`).
- **Typical columns (events):** `event_type_id`, `ts_seconds`, `location_id`, `order_id`, `sequence`, plus event-specific (e.g. `customer_lat`, `customer_lon`, `customer_addr`, `items_json`, `route_json`, `ping_*`, etc.).
- **Event type IDs:** 1=order_created, 2=gk_started, 3=gk_finished, 4=gk_ready, 5=driver_arrived, 6=driver_picked_up, 7=driver_ping, 8=delivered.

### 3.2 Volume JSON (Output of Replay → Input to Lakeflow)

Each event file is a single JSON object with:

- **event_id** (UUID)
- **event_type** (string): e.g. `order_created`, `gk_started`, …, `delivered`
- **ts** (string): `yyyy-MM-dd HH:mm:ss.SSS`
- **location_id** (int)
- **order_id** (string)
- **sequence** (int)
- **body** (JSON string), structure depends on event_type:
  - **order_created:** `customer_lat`, `customer_lon`, `customer_addr`, `items` (array of `{id, category_id, menu_id, brand_id, name, price, qty}`)
  - **driver_picked_up:** optional `route_points`
  - **driver_ping:** optional `progress_pct`, `loc_lat`, `loc_lon`
  - **delivered:** optional `delivered_lat`, `delivered_lon`
  - Others: often `{}`

### 3.3 Dimensional Data (Loaded by Canonical_Data Stage)

Loaded from parquet into `caspersdev.simulator`:

- **brands**, **locations**, **menus**, **categories**, **items**, **brand_locations**
- Source: `data/canonical/canonical_dataset/*.parquet` (canonical path; raw_data uses `data/dimensional/*.parquet` in an alternate path).

---

## 4. Databricks Objects and Their Relationships

### 4.1 Catalogs and Schemas

| Object | Purpose |
|--------|--------|
| **caspersdev** | Main UC catalog. |
| **caspersdev.simulator** | Dimensional tables + volumes (events, misc). |
| **caspersdev.lakeflow** | Medallion tables created by the Order Items declarative pipeline. |
| **caspersdev.recommender** | Refund recommendations table + checkpoints volume. |
| **caspersdev.complaints** | Raw complaints, complaint responses, checkpoints. |
| **caspersdev.ai** | UC functions used by the refund agent (get_order_details, get_order_delivery_time). |
| **caspersdevrefundmanager** (Lakebase) | UC catalog name for the refund manager Postgres instance (DB catalog, not a normal schema). |
| **caspersdevcomplaintmanager** (Lakebase) | UC catalog name for the complaint manager Postgres instance. |
| **caspersdev._internal_state** | UC-state: `resources` table for tracking jobs, pipelines, endpoints, etc., for cleanup. |

### 4.2 Volumes

| Volume | Use |
|--------|-----|
| **caspersdev.simulator.events** | Ingested event JSONs (output of canonical replay). |
| **caspersdev.simulator.misc** | Replay state: `_watermark`, `_sim_start`. |
| **caspersdev.recommender.checkpoints** | Refund recommender stream checkpoint. |
| **caspersdev.complaints.checkpoints** | Complaint generator and complaint agent stream checkpoints. |

### 4.3 Tables (by Schema)

**simulator (dimensional):**  
brands, locations, menus, categories, items, brand_locations.

**lakeflow (medallion):**

- **all_events** (bronze): raw event stream from volume (Cloud Files JSON).
- **silver_order_items**: one row per order line; columns include order_id, location_id, order_ts, order_day, item_id, menu_id, category_id, brand_id, item_name, price, qty, extended_price.
- **gold_order_header**: one row per order; order_id, location_id, order_day, order_revenue, total_qty, total_items, brands_in_order.
- **gold_item_sales_day**: item-level units_sold, gross_revenue by (item_id, menu_id, category_id, brand_id, day).
- **gold_brand_sales_day**: brand-level orders (approx), items_sold, brand_revenue by (brand_id, day).
- **gold_location_sales_hourly**: orders (approx), revenue by (location_id, hour_ts).

**recommender:**

- **refund_recommendations**: order_id, ts, order_ts, agent_response (JSON string: refund_usd, refund_class, reason). In caspersdev.recommender. Written by Refund Recommender Stream job.

**complaints:**

- **raw_complaints**: complaint_id, order_id, ts, complaint_category, complaint_text, generated_by. In caspersdev.complaints. Written by Complaint Generator job.
- **complaint_responses**: complaint_id, order_id, ts, agent_response (JSON). In caspersdev.complaints. Written by Complaint Agent Stream job. CDF enabled for Lakebase sync.

**ai (functions):**

- **get_order_details(oid)**: returns events for an order from `caspersdev.lakeflow.all_events` + location name.
- **get_order_delivery_time(oid)**: returns creation/delivery timestamps and duration from `caspersdev.lakeflow.all_events`.

### 4.4 Pipelines

| Pipeline | Type | Source | Output | Schedule |
|----------|------|--------|--------|----------|
| **Order Items Medallion Declarative Pipeline** | Lakeflow (Spark Declarative) | Volume JSON → all_events | silver_* and gold_* in caspersdev.lakeflow | Continuous (default) or triggered every N min |
| **Pipeline Update Scheduler** (optional job) | Job | – | Runs the above pipeline | Every N minutes when pipeline is in triggered mode (e.g. free target: 3) |
| **pg_recommendations** (synced table) | Lakebase reverse ETL | caspersdev.recommender.refund_recommendations | Postgres in caspersdevrefundmanager instance | Continuous |
| **pg_complaint_responses** (synced table) | Lakebase reverse ETL | caspersdev.complaints.complaint_responses | Postgres in caspersdevcomplaintmanager instance | Continuous |

### 4.5 Jobs (Scheduled)

| Job | Schedule | What it does |
|-----|----------|----------------|
| **Canonical Data Replay (Simple)** | Every 3 minutes | Replays canonical events into the events volume; updates watermark. |
| **Refund Recommender Stream** | Every 10 min | Reads delivered events from caspersdev.lakeflow.all_events; calls refund agent (or fake); appends to caspersdev.recommender.refund_recommendations. |
| **Complaint Generator Stream** | Every 10 min | Reads delivered from caspersdev.lakeflow.all_events; samples by COMPLAINT_RATE (0.15); generates complaint text (ai_gen); writes raw_complaints. |
| **Complaint Agent Stream** | Every 10 min | Reads raw_complaints; calls complaint agent; writes complaint_responses. |
| **Pipeline Update Scheduler** | Every N min when not continuous (e.g. 3 for free target) | Triggers the Order Items Lakeflow pipeline. |

### 4.6 Model Serving / Agents

- **Refund agent** (`caspers_refund_agent`): Chat-style endpoint; input = order context (e.g. order_id); output = JSON with refund_usd, refund_class, reason. Used by Refund Recommender Stream.
- **Complaint agent** (`caspers_complaint_agent`): Responses API; input = complaint text + order_id; output = JSON with order_id, complaint_category, decision, credit_amount, rationale, customer_response. Used by Complaint Agent Stream.

### 4.7 Lakebase (Reverse ETL)

- **Refund manager:** Database instance `caspersdevrefundmanager`; Postgres DB `caspers`. Synced table: `caspersdev.recommender.pg_recommendations` ← source `caspersdev.recommender.refund_recommendations` (PK: order_id), continuous.
- **Complaint manager:** Database instance `caspersdevcomplaintmanager`; Postgres DB `caspers_complaints`. Synced table: `caspersdev.complaints.pg_complaint_responses` ← source `caspersdev.complaints.complaint_responses` (PK: complaint_id), continuous.

### 4.8 Apps

- **Databricks App (Refund Manager):** Consumes data in the refund manager Lakebase DB (e.g. for ops UI). Created in apps stage after Lakebase is set up.

### 4.9 Initializer Job and State

- **Casper's Initializer** (single job with many tasks): Runs stages in order (Canonical_Data → Lakeflow → Refund/Complaint agents and streams → Lakebase → App). Different targets in `databricks.yml` (default, complaints, free, all) enable different task subsets.
- **UC-state** (`utils/uc_state`): Records created resources (jobs, pipelines, endpoints, database instances, database catalogs, etc.) in `caspersdev._internal_state.resources` so **destroy** can delete them in the right order.

---

## 5. Pipeline Dependency Graph (Stages)

```
Canonical_Data
    → creates catalog/schemas/volumes, dimensional tables, and the Canonical Data Replay job
    → (job runs on schedule) writes event JSON into events volume
    → (optional) wait for first data in volume

Spark_Declarative_Pipeline (Lakeflow)
    → depends on: Canonical_Data
    → creates Order Items pipeline; reads from volume → all_events → silver → gold

Refund_Recommender_Agent
    → depends on: Spark_Declarative_Pipeline
    → creates refund agent endpoint + UC functions

Refund_Recommender_Stream
    → depends on: Spark_Declarative_Pipeline
    → creates scheduled job that reads caspersdev.lakeflow.all_events (delivered) → caspersdev.recommender.refund_recommendations

Complaint_Agent
    → depends on: Spark_Declarative_Pipeline
    → creates complaint agent endpoint

Complaint_Generator_Stream
    → depends on: Spark_Declarative_Pipeline
    → creates job: caspersdev.lakeflow.all_events (delivered) → sample → ai_gen → caspersdev.complaints.raw_complaints

Complaint_Agent_Stream
    → depends on: Complaint_Agent, Complaint_Generator_Stream
    → creates job: raw_complaints → agent → caspersdev.complaints.complaint_responses

Complaint_Lakebase
    → depends on: Complaint_Agent_Stream
    → Lakebase instance + synced table for complaint_responses

Lakebase_Reverse_ETL
    → depends on: Refund_Recommender_Stream
    → Lakebase instance + synced table for refund_recommendations

Databricks_App_Refund_Manager
    → depends on: Lakebase_Reverse_ETL
    → Creates app and warehouse for refund manager UI
```

---

## 6. Designing a Similar Demo (e.g. Healthcare)

Use this as a checklist; substitute domain entities (e.g. patient, encounter, order, claim) for order/delivery/complaint/refund.

1. **Domain event model**
   - Define event types (e.g. patient_admitted, lab_ordered, result_ready, claim_submitted).
   - Define a canonical event schema (event_type, ts, ids, body per type).
   - Produce a canonical dataset (parquet or similar) with a fixed time window (e.g. 90 days).

2. **Dynamic data generation**
   - UC catalog + schema + volume for “events”.
   - A “replay” notebook: read canonical events, state (watermark/sim_start), compute next window, map to your JSON schema, write to volume, update state.
   - A scheduled job (e.g. every 3–5 min) that runs this notebook.

3. **Medallion pipeline**
   - Lakeflow pipeline: Cloud Files on the events volume → bronze table (e.g. all_events) → silver (normalized/exploded) → gold (aggregations by day/facility/product, etc.).
   - Same pattern: configuration for catalog/schema/volume; continuous or triggered schedule.

4. **Downstream apps (optional)**
   - **Refund-like:** e.g. “recommendation” or “anomaly” stream: read a subset of events (e.g. “completed encounters”), call an agent or model, write to a table (e.g. recommendations or alerts). Schedule (e.g. every 10 min).
   - **Complaint-like:** e.g. “synthetic feedback” or “audit events”: sample events, generate text (LLM or rules), write to raw table; then another job that processes them with an agent and writes responses.

5. **Reverse ETL and app**
   - Lakebase: one (or two) DB instances; synced tables from your “recommendations” and “responses” Delta tables (with CDF if needed) to Postgres for apps.
   - Databricks App (or external app) reading from Lakebase for operational UI.

6. **State and cleanup**
   - Use a UC-state style table to register jobs, pipelines, endpoints, Lakebase instances/catalogs so a single “destroy” flow can remove them in the correct order (e.g. jobs → pipelines → endpoints → apps → Lakebase → catalog).

7. **Parameters**
   - Catalog, schema, volume names, schedule intervals, speed multiplier, and any domain-specific (e.g. complaint_rate → “audit_sample_rate”) as job/widget parameters and in databricks.yml targets.

8. **Data formats**
   - Document the canonical parquet schema and the volume JSON schema (event_type, ts, body per type) so the Lakeflow pipeline and downstream jobs stay aligned.

This summary and the dependency graph above should be enough to replicate the Casper’s pattern in another domain (e.g. healthcare products, claims, or encounters) with a clear mapping from “order/delivery/refund/complaint” to your entities and tables.
