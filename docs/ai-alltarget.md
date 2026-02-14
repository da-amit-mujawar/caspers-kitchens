# What the `all` target adds (that `default` doesn’t have)

If you ran the bundle with the **default** target, you got the **refund** track only. The **all** target adds the full **complaints** track on top of that.

---

## Default target (what you ran)

The job runs these tasks in order:

1. **Canonical_Data** – bootstrap catalog, simulator schema/volumes, dimensionals, canonical replay job  
2. **Spark_Declarative_Pipeline** – Lakeflow medallion (events → silver/gold)  
3. **Refund_Recommender_Agent** – deploy refund agent endpoint + UC functions  
4. **Refund_Recommender_Stream** – scheduled job: delivered orders → agent → refund_recommendations  
5. **Lakebase_Reverse_ETL** – Lakebase instance + sync refund_recommendations → Postgres  
6. **Databricks_App_Refund_Manager** – Refund Manager app + warehouse  

So you get: **data → pipeline → refund agent → refund stream → Lakebase (refund) → Refund Manager app.**

---

## What `all` adds (the “fun stuff” you missed)

The **all** target includes **four extra tasks** between the pipeline and the refund Lakebase/app. They are the full **complaint** flow:

| Task | What it does |
|------|----------------|
| **Complaint_Agent** | Deploys the **complaint agent** model serving endpoint (e.g. `caspers_complaint_agent`). Same idea as the refund agent: takes complaint text + order context, returns structured response (category, decision, credit_amount, rationale, customer reply). Uses UC functions (order overview, timing, location timings). |
| **Complaint_Generator_Stream** | Creates a **scheduled job** (e.g. every 10 min) that samples delivered orders by COMPLAINT_RATE (e.g. 15%), generates synthetic complaint text (e.g. via `ai_gen`), and writes to **raw_complaints** (complaint_id, order_id, category, complaint_text). So you get a stream of “customer complaints” without real customers. |
| **Complaint_Agent_Stream** | Creates another **scheduled job** (e.g. every 10 min) that reads **raw_complaints**, calls the complaint agent for each row, and writes **complaint_responses** (complaint_id, order_id, agent_response JSON). So every synthetic complaint gets a suggested triage and reply. |
| **Complaint_Lakebase** | Provisions a **second Lakebase instance** (e.g. `caspersdevcomplaintmanager`) and a **synced table** that continuously replicates **complaint_responses** to Postgres. So you can build a Complaints Manager app (or any UI) on top of that DB. |

**all** also adds the **complaint-related job parameters**: `COMPLAINT_AGENT_ENDPOINT_NAME`, `COMPLAINT_RATE`.

---

## In one sentence

**Default** = refund-only (recommendations + Lakebase + Refund Manager app).  
**All** = refund **and** complaints: a second agent, synthetic complaint generation, complaint agent stream, and a second Lakebase DB for complaint responses, so you have both “refund suggestions” and “complaint triage + suggested replies” with data flowing to Postgres for apps.

---

## How to get the “all” experience now

You can either:

- **Redeploy with the `all` target** and run the Casper’s Initializer job again (it will create the extra tasks; run the new ones: Complaint_Agent, Complaint_Generator_Stream, Complaint_Agent_Stream, Complaint_Lakebase), or  
- **Run those four stages manually** (same notebooks as in the job) in the same order, with the same catalog and parameters, so you get the complaint agent, the two complaint jobs, and the complaint Lakebase instance without re-running the whole job.

The refund path you already have is unchanged; **all** just adds the complaint path alongside it.
