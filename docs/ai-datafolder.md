# The `data/` folder: what each part does and how they work together

This document explains everything under **`data/`** in the Casper's Kitchens repo: what each file and folder is for, how they relate to each other, and which path the **current demo** uses.

---

## 1. High-level map

The repo has **three** main data-generation approaches under `data/`:

| Area | Purpose | Used by current demo? |
|------|---------|------------------------|
| **canonical/** | Pre-generated 90-day dataset + **replay** (scheduled job writes events to a volume). | **Yes** – this is the main path. |
| **generator/** | **Live** Python simulator: generates events in real time (or accelerated) from JSON configs, writes to volume. | No – legacy/alternative. |
| **universe/** | **Rust** simulation engine (sites, kitchens, population, events). Excluded from bundle sync. | No – alternative/experimental. |

There is also a **dimensional** reference: the **raw_data** stage expects `data/dimensional/*.parquet` (brands, menus, categories, items). That folder is **not** present in the repo; the **canonical_data** stage uses **canonical/canonical_dataset/** instead. So the **current** flow uses **canonical** only.

---

## 2. `data/canonical/` – main path (canonical dataset + replay)

This is the **primary** data path for the demo: a fixed 90-day dataset plus a **replay** process that writes events into the events volume on a schedule.

### 2.1 Directory layout

```
data/canonical/
├── README.md                      # Full docs: dataset, replay, schema, generation
├── generate_canonical_dataset.py  # Offline script: builds events.parquet from dimensionals
├── canonical_generator_simple.ipynb   # Replay notebook (used by scheduled job)
├── canonical_generator.ipynb      # Alternative replay using custom streaming source
├── caspers_data_source.py         # PySpark custom streaming source (CaspersDataSource)
├── caspers_streaming_notebook.py  # Databricks notebook version of the streaming source
└── canonical_dataset/             # Directory of parquet files (see below)
```

### 2.2 `canonical_dataset/`

Holds the **dimensional** tables and the **events** table that the replay consumes.

| File | Role |
|------|------|
| **events.parquet** | ~1M+ events, ~75K orders over 90 days (Jan 1 – Mar 30, 2024). One row per event (order_created, gk_started, …, driver_ping, delivered). Produced by **generate_canonical_dataset.py**. |
| **locations.parquet** | 4 ghost kitchen locations (id, name, lat/lon, base_orders_day, growth_rate_daily, location_code). |
| **brands.parquet** | 24 brands. |
| **brand_locations.parquet** | Which brands operate at which location, with start_day/end_day and growth rates. |
| **categories.parquet** | Menu categories. |
| **menus.parquet** | Menus. |
| **items.parquet** | 181 menu items (id, name, price, category_id, brand_id, etc.). |

**generate_canonical_dataset.py** **reads** the dimensionals from `canonical_dataset/` and **writes** only **events.parquet**. So the dimensionals must already exist (shipped in repo or created elsewhere). Running the script regenerates the 90 days of events (~10 min: OSM networks, routes, Poisson/Gaussian sampling).

### 2.3 What each canonical file does

- **generate_canonical_dataset.py**  
  Offline Python script. Loads locations, brands, brand_locations, categories, items from `canonical_dataset/`. For 90 days and 4 locations: computes order counts (Poisson, minute weights, day-of-week, growth), generates orders with baskets, service times (Gaussian), driver arrival (Beta), OSM routing, driver pings; writes one **events.parquet** with columns such as ts_seconds, order_id, event_type_id, location_id, sequence, items_json, customer_lat/lon, route_json, ping_*, etc.

- **canonical_generator_simple.ipynb**  
  **Replay** notebook used by the **Canonical Data Replay** scheduled job (e.g. every 3 minutes). Reads **events.parquet**, maintains **watermark** and **sim_start** in state files, computes the next time window from elapsed real time × speed multiplier, optionally **loops** the 90-day cycle and suffixes order_id with `-L1`, `-L2`, … Transforms to output schema (event_type string, ts, body JSON, event_id) and **appends** JSON to the events volume. Updates watermark after write. See **docs/ai-datagen.md** for design details.

- **canonical_generator.ipynb**  
  Alternative replay that uses a **custom PySpark streaming source** (same idea as caspers_data_source) with checkpoint-based offsets. Can run as a Spark structured stream; the “simple” notebook is the one actually used by the init job.

- **caspers_data_source.py**  
  Implements a **PySpark Data Source** named `"caspers"`. Options: datasetPath, simulationStartDay, speedMultiplier. Loads `events.parquet` with pandas, implements `initialOffset()` and `getBatch()` so Spark Structured Streaming can read “micro-batches” of events by simulation time. Used for local or notebook-based streaming tests, not by the default scheduled replay.

- **caspers_streaming_notebook.py**  
  Databricks notebook version of the same custom source: configures a stream with `spark.readStream.format("caspers")` and the same options. Alternative to the simple notebook if you want stream processing with Spark checkpointing.

### 2.4 How canonical fits into the demo

1. **Canonical_Data** stage: creates catalog/schema/volumes; loads dimensionals from **canonical_dataset/** (locations, brands, menus, categories, items, brand_locations) into the simulator schema; creates the **Canonical Data Replay** job that runs **canonical_generator_simple** on a schedule.
2. The replay job writes new event JSON files into the events volume.
3. The **Lakeflow** pipeline reads from that volume (Cloud Files) and fills **lakeflow.all_events** and downstream silver/gold tables.

So: **canonical_dataset/** (dimensionals + events.parquet) → **canonical_generator_simple** (replay) → **events volume** → **Lakeflow**.

---

## 3. `data/generator/` – live Python simulator (legacy/alternative)

This is the **older** approach: a single notebook that generates events **on the fly** (with optional time acceleration) and writes them to the same kind of events volume. No pre-generated parquet; no replay.

### 3.1 Directory layout

```
data/generator/
├── generator.ipynb       # Ghost Kitchen Event Simulator 2.1
└── configs/
    ├── README.md         # How to configure locations and what each parameter does
    ├── sanfrancisco.json # Example config for one location
    └── chicago.json      # Another location config
```

### 3.2 What it does

- **generator.ipynb**  
  Takes a **SIM_CFG_JSON** (one location config from `configs/`) plus CATALOG, SCHEMA, VOLUME. Loads OSM road network for the kitchen address, builds order timelines (Gaussian service times, Beta driver arrival, routing, pings).  
  - **Back-fill:** Generates all historical events from `start_days_ago` up to “now” and writes them in one go.  
  - **Live:** Schedules future orders (Poisson per minute, day-of-week and intraday weights) and “plays” them with `asyncio.sleep((ts - now) / speed_up)`, writing events as they become due.  
  Writes JSON events to `/Volumes/{CATALOG}/{SCHEMA}/{VOLUME}/`. Schema uses `gk_id`, `location` (name), and similar event types to the canonical output.

- **configs/*.json**  
  One JSON file per location. Parameters include:
  - **Time:** start_days_ago, end_days_ahead, speed_up
  - **Volume:** orders_day_1, orders_last, noise_pct
  - **Service:** svc (cs, sf, fr, rp as [mean, std] minutes)
  - **Driver:** driver_arrival (alpha, beta, after_ready_pct), driver_mph
  - **Brands:** brand_momentum, momentum_rates
  - **Geography:** gk_location, location_name, radius_mi
  - **Technical:** batch_rows, batch_seconds, ping_sec, random_seed, dq (data quality)

  The **LOCATIONS** job parameter (e.g. `sanfrancisco.json` or `sanfrancisco.json,chicago.json`) selects which config(s) run. See **configs/README.md** for full parameter docs.

### 3.3 How it relates to canonical

- **Same domain:** Same event types (order_created → … → delivered), same idea of locations, orders, routes, pings.
- **Different execution:** Generator = one long-running (or scheduled) process that creates events in real/simulated time and writes them. Canonical = static parquet + a small replay job that only **slices** the dataset and writes the next window. The demo uses canonical for reliability and portability.
- **Dimensional data:** The generator does **not** populate the simulator dimensionals; those come from **raw_data** (which expects `data/dimensional/*.parquet`) or from **canonical_data** (which uses `data/canonical/canonical_dataset/*.parquet`). So if you use the generator, you’d typically pair it with a bootstrap that provides dimensionals (e.g. raw_data with a dimensional folder).

---

## 4. `data/universe/` – Rust simulation engine (alternative, not in bundle)

A **Rust**-based simulation with its own model: sites, kitchens, stations, population, and event generation. It is **excluded** from the Databricks bundle sync (`databricks.yml` has `exclude: ./data/universe/**`), so it is **not** deployed with the default demo.

### 4.1 Directory layout (conceptual)

```
data/universe/
├── crates/
│   ├── universe/           # Core library
│   │   ├── Cargo.toml, README.md
│   │   ├── src/
│   │   │   ├── lib.rs           # SimulationSetup, entry points
│   │   │   ├── simulation/      # Simulation engine, events, next step
│   │   │   ├── state/           # State (orders, population, movement, objects)
│   │   │   ├── agents/          # SiteRunner, PopulationRunner, kitchen logic
│   │   │   ├── builders/        # Event builders, results
│   │   │   ├── context/        # SimulationContext, storage, schemas
│   │   │   ├── models/         # Protobuf-generated (caspers.*.v1)
│   │   │   ├── templates/       # Load templates from object store
│   │   │   └── ...
│   │   └── templates/base/      # Default JSON configs
│   │       ├── sites/           # london.json, amsterdam.json
│   │       ├── brands/          # mexican.json, fast_food.json, asian.json
│   │       └── ingredients.json
│   └── cli/                 # CLI for running the universe
```

### 4.2 What it does

- **SimulationSetup** loads **sites** (e.g. `sites/london.json`, `sites/amsterdam.json`) and **brands** (e.g. `brands/mexican.json`) from an object-store path. Each site has kitchens; each kitchen has stations (workstation, oven, stove).
- **Simulation** holds **State** (orders, population, movement, objects), **SiteRunner** and **PopulationRunner** agents, and drives the simulation (events, next step).
- **Templates** define the initial world (locations, brands, ingredients). The Rust code can write results (events, metrics) to storage via the context.

So this is a **different** implementation of a ghost-kitchen-style simulation (more entity types, Rust, object-store backed), not wired into the current Python/Databricks demo pipeline. Use it if you want to experiment with a Rust-based event source or a different data model.

---

## 5. `data/dimensional/` (missing in repo)

The **raw_data** stage notebook references:

- `../data/dimensional/brands.parquet`
- `../data/dimensional/menus.parquet`
- `../data/dimensional/categories.parquet`
- `../data/dimensional/items.parquet`

There is **no** `data/dimensional/` directory in the repo. So:

- Either that folder is created/generated elsewhere and not committed, or
- **raw_data** is a legacy path and the **canonical_data** stage is the intended one: it uses **canonical_dataset/** for dimensionals (and for events via replay).

The **current** init flow uses **Canonical_Data**, which reads dimensionals from **canonical_dataset/** and uses **canonical_generator_simple** for events. So you can ignore **dimensional** unless you are explicitly using the raw_data + generator path.

---

## 6. How they work together (summary)

- **Current demo:**  
  **canonical_dataset/** (dimensionals + events.parquet) → **canonical_generator_simple** (replay job) → events volume → Lakeflow. Dimensionals are also loaded from **canonical_dataset/** into the simulator schema by the Canonical_Data stage.

- **Alternative (live gen):**  
  **generator/configs/*.json** → **generator.ipynb** → events volume. Dimensionals would need to come from somewhere (e.g. raw_data with a populated **dimensional/** or a copy of canonical_dataset dimensionals).

- **Alternative (Rust):**  
  **universe** templates and Rust engine produce their own events/state; not connected to the Databricks pipeline and excluded from sync.

- **dimensional:**  
  Referenced only by raw_data; folder absent; canonical_data + canonical_dataset replace this in the main flow.

Putting it in one diagram:

```
Current demo:
  data/canonical/canonical_dataset/*.parquet
       │
       ├── dimensionals ──► Canonical_Data stage (load into simulator schema)
       │
       └── events.parquet ──► canonical_generator_simple (replay) ──► volume ──► Lakeflow

Legacy/alternative:
  data/generator/configs/*.json ──► generator.ipynb ──► volume  (dimensionals from elsewhere)
  data/universe (Rust) ──► standalone simulation (not in bundle)
```

This should give you a clear map of what lives in **data/** and how it fits together.
