# Catalog Object Index (caspersdev)

Index of Unity Catalog objects created by the Casper's Kitchens demo (default catalog **caspersdev**). Section 1 is a one-sentence tree; Section 2 has short detailed descriptions (max 200 words each).

---

## 1. Object Tree (names and relationships)

One-sentence description per node. Indent = child of parent.

```
caspersdev
  Main UC catalog for the demo; contains all Delta tables, volumes, functions, and synced-table definitions.

  simulator
    Schema holding dimensional reference data and event/stage volumes.

    brands
      Table. Brand reference (brand_id, name, etc.); loaded from canonical parquet.
    locations
      Table. Kitchen/location reference; joined to events by location_id.
    menus
      Table. Menu reference; used for item hierarchy.
    categories
      Table. Product category reference.
    items
      Table. Menu item reference (id, name, price, category_id, brand_id, etc.).
    brand_locations
      Table. Many-to-many link between brands and locations.
    events (volume)
      Volume. Ingested event JSON files; written by Canonical Data Replay job, read by Lakeflow.
    misc (volume)
      Volume. Replay state files: _watermark, _sim_start.

  lakeflow
    Schema for the Order Items medallion pipeline (bronze/silver/gold).

    all_events
      Table (bronze). Raw event stream from caspersdev.simulator.events via Cloud Files.
    silver_order_items
      Table. One row per order line; from order_created events with extended_price, order_day.
    gold_order_header
      Table. One row per order; order_revenue, total_qty, total_items, brands_in_order.
    gold_item_sales_day
      Table. Item-level units_sold, gross_revenue by (item_id, menu_id, category_id, brand_id, day).
    gold_brand_sales_day
      Table. Brand-level orders (approx), items_sold, brand_revenue by (brand_id, day).
    gold_location_sales_hourly
      Table. Hourly orders (approx), revenue by (location_id, hour_ts).

  recommender
    Schema for refund recommendations and stream checkpoints.

    refund_recommendations
      Table. order_id, ts, order_ts, agent_response (JSON); written by Refund Recommender Stream job.
    pg_recommendations
      Synced table. Reverse ETL from refund_recommendations to Postgres (caspersdevrefundmanager).
    checkpoints (volume)
      Volume. Refund recommender stream checkpoint.

  complaints
    Schema for complaint generation and agent responses.

    raw_complaints
      Table. complaint_id, order_id, ts, complaint_category, complaint_text, generated_by; from Complaint Generator.
    complaint_responses
      Table. complaint_id, order_id, ts, agent_response (JSON); from Complaint Agent Stream; CDF enabled.
    pg_complaint_responses
      Synced table. Reverse ETL from complaint_responses to Postgres (caspersdevcomplaintmanager).
    checkpoints (volume)
      Volume. Complaint generator and complaint agent stream checkpoints.

  ai
    Schema for UC functions and views used by refund and complaint agents.

    get_order_details(oid)
      Function. Returns events for an order from lakeflow.all_events plus location name.
    get_order_delivery_time(oid)
      Function. Returns creation/delivery timestamps and duration for an order.
    get_order_overview(oid)
      Function. Order summary for complaint agent.
    get_order_timing(oid)
      Function. Order timing for complaint agent.
    get_location_timings(loc)
      Function. P50/P75/P99 delivery times by location name.
    order_delivery_times_per_location_view
      View. Last 24h order delivery times by location; used by get_location_timings.

  _internal_state
    Schema for UC-state (resource tracking for destroy).

    resources
      Table. Stores job_id, pipeline_id, endpoint names, etc., for ordered cleanup.

caspersdevrefundmanager
  Lakebase catalog; Postgres-backed (DB name: caspers). Target of pg_recommendations sync.

caspersdevcomplaintmanager
  Lakebase catalog; Postgres-backed (DB name: caspers_complaints). Target of pg_complaint_responses sync.
```

---

## 2. Detailed descriptions

### 2.1 Catalog: caspersdev

**caspersdev** is the primary Unity Catalog catalog for the demo. All Delta tables, volumes, functions, and views live here. It is created by the Canonical_Data stage (or Raw_Data in alternate flows). The bundle variable `catalog` defaults to `caspersdev`; destroy drops this catalog (and its contents) last, after jobs, pipelines, endpoints, Lakebase instances, and database catalogs are removed using state stored in `caspersdev._internal_state.resources`.

---

### 2.2 Schema: simulator

**caspersdev.simulator** holds dimensional tables (brands, locations, menus, categories, items, brand_locations) loaded from parquet in the Canonical_Data stage, and two volumes: **events** (where the Canonical Data Replay job appends one JSON file per event) and **misc** (replay state: `_watermark`, `_sim_start`). The Lakeflow pipeline reads from `simulator.events` via Cloud Files to populate `lakeflow.all_events`. Dimensional tables are referenced by the pipeline and by the refund agent UC functions (e.g. location name join).

---

### 2.3 Schema: lakeflow

**caspersdev.lakeflow** contains the Order Items medallion pipeline output. **all_events** is the bronze streaming table (raw JSON from the volume). **silver_order_items** explodes `order_created` events into one row per line item with extended_price and order_day. Gold tables aggregate from silver: **gold_order_header** (per-order revenue/counts), **gold_item_sales_day**, **gold_brand_sales_day**, and **gold_location_sales_hourly** (with approximate distinct counts and watermarks). Downstream jobs (Refund Recommender Stream, Complaint Generator Stream) read `all_events` filtered by `event_type = 'delivered'`.

---

### 2.4 Schema: recommender

**caspersdev.recommender** holds **refund_recommendations** (order_id, ts, order_ts, agent_response JSON), written by the Refund Recommender Stream job, and **pg_recommendations**, a Lakebase synced table that continuously replicates refund_recommendations to the Postgres database in the caspersdevrefundmanager instance. The **checkpoints** volume stores the refund recommender stream checkpoint so runs are resumable.

---

### 2.5 Schema: complaints

**caspersdev.complaints** holds **raw_complaints** (complaint_id, order_id, ts, complaint_category, complaint_text, generated_by), produced by the Complaint Generator job from a sample of delivered orders with LLM-generated text, and **complaint_responses** (complaint_id, order_id, ts, agent_response), produced by the Complaint Agent Stream job. complaint_responses has Change Data Feed enabled for Lakebase. **pg_complaint_responses** syncs complaint_responses to Postgres in caspersdevcomplaintmanager. The **checkpoints** volume stores checkpoints for both the complaint generator and complaint agent streams.

---

### 2.6 Schema: ai

**caspersdev.ai** contains UC functions and one view used by the refund and complaint agents. **get_order_details(oid)** and **get_order_delivery_time(oid)** are used by the refund agent; **get_order_overview(oid)**, **get_order_timing(oid)**, and **get_location_timings(loc)** by the complaint agent. **order_delivery_times_per_location_view** computes recent delivery-time percentiles by location and is queried by get_location_timings. Agents are deployed as model serving endpoints; their registered model names live in the same catalog (e.g. caspersdev.ai.refunder, caspersdev.ai.complaint_agent).

---

### 2.7 Schema: _internal_state

**caspersdev._internal_state** is used by the UC-state utility. The **resources** table stores serialized metadata (job_id, pipeline_id, endpoint names, database instance names, etc.) for every resource created by the init stages. The destroy flow reads this table and deletes resources in a fixed order (jobs, pipelines, endpoints, apps, database catalogs, database instances, then catalog) so teardown is safe and repeatable.

---

### 2.8 Synced tables (pg_recommendations, pg_complaint_responses)

**caspersdev.recommender.pg_recommendations** and **caspersdev.complaints.pg_complaint_responses** are Lakebase synced tables: they define a continuous sync from a Delta source table (refund_recommendations, complaint_responses) to a table in a Lakebase-managed Postgres instance. Primary keys are order_id and complaint_id respectively. The target Postgres databases are exposed as UC catalogs **caspersdevrefundmanager** (DB: caspers) and **caspersdevcomplaintmanager** (DB: caspers_complaints). The Refund Manager app reads from the refund manager Postgres.

---

### 2.9 Lakebase catalogs: caspersdevrefundmanager, caspersdevcomplaintmanager

**caspersdevrefundmanager** and **caspersdevcomplaintmanager** are Unity Catalog catalogs backed by Lakebase (managed Postgres). They are created as database instances and then registered as database catalogs with logical DB names `caspers` and `caspers_complaints`. They do not contain Delta tables; data arrives via the synced tables in caspersdev. The Refund Manager Databricks App connects to the refund manager instance for its UI.
