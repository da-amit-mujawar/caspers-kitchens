# The `all` target: full reference

The **all** target defines the Casper's Initializer job with **both** the refund and complaint tracks: data, pipeline, two agents, two streams, two Lakebase instances, and the Refund Manager app. This document describes the job and every task in full.

---

## Job

- **Name:** Casper's Initializer  
- **Target:** `all` (use `databricks bundle deploy -t all` and run the job from the workspace, or run the job with base parameters set for the all target).  
- **Queue:** enabled  
- **Performance target:** PERFORMANCE_OPTIMIZED  

---

## Parameters (defaults)

| Parameter | Default | Purpose |
|-----------|---------|---------|
| CATALOG | caspersdev | Unity Catalog catalog for all objects |
| EVENTS_VOLUME | events | Volume name for event JSONs (under simulator schema) |
| LLM_MODEL | databricks-meta-llama-3-3-70b-instruct | Foundation model for agents |
| REFUND_AGENT_ENDPOINT_NAME | caspers_refund_agent | Serving endpoint name for the refund agent |
| COMPLAINT_AGENT_ENDPOINT_NAME | caspers_complaint_agent | Serving endpoint name for the complaint agent |
| COMPLAINT_RATE | 0.15 | Fraction of delivered orders that get a synthetic complaint (e.g. 15%) |
| SIMULATOR_SCHEMA | simulator | Schema for dimensionals and events volume |
| START_DAY | 70 | Replay start day (0–89) in the canonical dataset |
| SPEED_MULTIPLIER | 60.0 | Replay speed (e.g. 60× real time) |
| SCHEDULE_MINUTES | 3 | How often the Canonical Data Replay job runs (minutes) |
| PIPELINE_SCHEDULE_MINUTES | 0 | 0 = Lakeflow pipeline runs continuously; >0 = triggered every N minutes |

---

## Tasks (execution order and dependencies)

Tasks run in an order consistent with the dependency graph below. Parallelism: Refund_Recommender_Agent, Refund_Recommender_Stream, Complaint_Agent, and Complaint_Generator_Stream all depend only on Spark_Declarative_Pipeline, so they can run in parallel after the pipeline is up.

### 1. Canonical_Data

- **Depends on:** none  
- **Notebook:** `stages/canonical_data`  
- **What it does:** Creates the catalog (if needed), simulator schema, and volumes (events, misc). Loads dimensional tables (brands, locations, menus, categories, items, brand_locations) from `data/canonical/canonical_dataset/` into the simulator schema. Creates the **Canonical Data Replay (Simple)** scheduled job so event JSONs are written to the events volume on a schedule (e.g. every 3 minutes). Waits until at least some data appears in the volume so the pipeline can infer schema.  
- **Outcome:** Catalog/schema/volumes, dimensionals in simulator, replay job running, events landing in the volume.

### 2. Spark_Declarative_Pipeline

- **Depends on:** Canonical_Data  
- **Notebook:** `stages/lakeflow`  
- **What it does:** Creates the **Order Items Medallion Declarative Pipeline** (Lakeflow): reads JSON from the events volume (Cloud Files), writes to bronze `all_events`, then silver (e.g. silver_order_items) and gold tables (gold_order_header, gold_item_sales_day, gold_brand_sales_day, gold_location_sales_hourly) in the lakeflow schema. Pipeline runs continuously (PIPELINE_SCHEDULE_MINUTES=0) or is triggered on a schedule. Registers the pipeline in UC-state for cleanup.  
- **Outcome:** Lakeflow pipeline running; tables in `catalog.lakeflow` populated from the event stream.

### 3. Refund_Recommender_Agent

- **Depends on:** Spark_Declarative_Pipeline  
- **Notebook:** `stages/refunder_agent`  
- **What it does:** Creates the **refund agent** model serving endpoint (e.g. caspers_refund_agent). Registers UC functions (e.g. get_order_details, get_order_delivery_time, get_location_timings) and views (e.g. order_delivery_times_per_location_view) in the ai schema so the agent can query order and delivery data. The agent takes an order_id (or context) and returns a structured recommendation (refund_usd, refund_class, reason).  
- **Outcome:** Refund agent endpoint live; UC functions and views available for the agent.

### 4. Refund_Recommender_Stream

- **Depends on:** Spark_Declarative_Pipeline  
- **Notebook:** `stages/refunder_stream`  
- **What it does:** Creates a **scheduled job** (e.g. every 10 minutes) that runs the refund recommender stream notebook: reads delivered events from `catalog.lakeflow.all_events`, calls the refund agent (or uses fallback responses) per order, and appends rows to `catalog.recommender.refund_recommendations`. Uses a checkpoint volume so progress is resumable. Registers the job in UC-state.  
- **Outcome:** Scheduled job running; refund_recommendations table growing with each run.

### 5. Complaint_Agent

- **Depends on:** Spark_Declarative_Pipeline  
- **Notebook:** `stages/complaint_agent`  
- **What it does:** Creates the **complaint agent** model serving endpoint (e.g. caspers_complaint_agent). Registers UC functions (e.g. get_order_overview, get_order_timing, get_location_timings) and the order_delivery_times_per_location_view (if not already present) so the agent can use order and timing data. The agent takes complaint text and order context and returns a structured response (category, decision, credit_amount, rationale, customer_response).  
- **Outcome:** Complaint agent endpoint live; UC functions available for the agent.

### 6. Complaint_Generator_Stream

- **Depends on:** Spark_Declarative_Pipeline  
- **Notebook:** `stages/complaint_generator_stream`  
- **What it does:** Creates a **scheduled job** (e.g. every 10 minutes) that runs the complaint generator notebook: reads delivered events from `catalog.lakeflow.all_events`, samples a fraction (COMPLAINT_RATE, e.g. 15%) of orders, assigns a complaint category (e.g. delivery_delay, missing_items, food_quality), generates complaint text (e.g. via ai_gen), and writes to `catalog.complaints.raw_complaints`. Uses a checkpoint so runs are resumable. Registers the job in UC-state.  
- **Outcome:** Scheduled job running; raw_complaints table filling with synthetic complaints.

### 7. Complaint_Agent_Stream

- **Depends on:** Complaint_Agent, Complaint_Generator_Stream  
- **Notebook:** `stages/complaint_agent_stream`  
- **What it does:** Creates a **scheduled job** (e.g. every 10 minutes) that runs the complaint agent stream notebook: reads from `catalog.complaints.raw_complaints`, calls the complaint agent for each row, and writes to `catalog.complaints.complaint_responses` (complaint_id, order_id, ts, agent_response JSON). Enables Change Data Feed on complaint_responses for Lakebase. Uses a checkpoint. Registers the job in UC-state.  
- **Outcome:** Scheduled job running; complaint_responses table filling with triage and suggested replies.

### 8. Complaint_Lakebase

- **Depends on:** Complaint_Agent_Stream  
- **Notebook:** `stages/complaint_lakebase`  
- **What it does:** Provisions a **Lakebase (managed Postgres) instance** (e.g. catalog name `caspersdevcomplaintmanager`), creates a database (e.g. caspers_complaints), and registers it as a Unity Catalog database catalog. Creates a **synced table** that continuously replicates `catalog.complaints.complaint_responses` to Postgres (primary key: complaint_id). Registers the instance and synced table in UC-state.  
- **Outcome:** Second Lakebase instance and Postgres DB; complaint responses synced for use by a Complaints Manager app or any client.

### 9. Lakebase_Reverse_ETL

- **Depends on:** Refund_Recommender_Stream  
- **Notebook:** `stages/lakebase`  
- **What it does:** Provisions a **Lakebase instance** (e.g. catalog name `caspersdevrefundmanager`), creates a database (e.g. caspers), and registers it as a UC database catalog. Creates a **synced table** that continuously replicates `catalog.recommender.refund_recommendations` to Postgres (primary key: order_id). Registers the instance and synced table in UC-state.  
- **Outcome:** Refund Lakebase instance and Postgres DB; refund recommendations synced for the Refund Manager app.

### 10. Databricks_App_Refund_Manager

- **Depends on:** Lakebase_Reverse_ETL  
- **Notebook:** `stages/apps`  
- **What it does:** Creates a **Databricks App** (Refund Manager) and a SQL warehouse (if needed), and configures the app to use the refund manager Lakebase instance so users can view and act on refund recommendations. Registers the app in UC-state.  
- **Outcome:** Refund Manager app available in the workspace; users can open it and work with synced refund data.

---

## Dependency graph (all target)

```
Canonical_Data
    └── Spark_Declarative_Pipeline
            ├── Refund_Recommender_Agent
            ├── Refund_Recommender_Stream ──────► Lakebase_Reverse_ETL ──► Databricks_App_Refund_Manager
            ├── Complaint_Agent
            ├── Complaint_Generator_Stream
            │           └── Complaint_Agent_Stream ──► Complaint_Lakebase
            └── (Complaint_Agent_Stream depends on both Complaint_Agent and Complaint_Generator_Stream)
```

---

## What you get when the job completes

- **Data:** Canonical replay job and Lakeflow pipeline feeding events and medallion tables.  
- **Refund path:** Refund agent endpoint, refund recommender stream job, Lakebase Postgres with refund_recommendations synced, Refund Manager app.  
- **Complaint path:** Complaint agent endpoint, complaint generator job, complaint agent stream job, Lakebase Postgres with complaint_responses synced (ready for a Complaints Manager app).  
- **State:** Jobs, pipelines, endpoints, Lakebase instances, and app registered in UC-state (catalog._internal_state.resources) so the destroy flow can tear them down in the right order.

---

## How this differs from the default target

The **default** target runs only six tasks: Canonical_Data, Spark_Declarative_Pipeline, Refund_Recommender_Agent, Refund_Recommender_Stream, Lakebase_Reverse_ETL, Databricks_App_Refund_Manager. It does **not** include Complaint_Agent, Complaint_Generator_Stream, Complaint_Agent_Stream, or Complaint_Lakebase, and the job has no COMPLAINT_AGENT_ENDPOINT_NAME or COMPLAINT_RATE parameters. So default = refund-only; **all** = refund + full complaint track (agent, synthetic complaints, complaint agent stream, second Lakebase DB).
