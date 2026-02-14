Copy one of the blocks below and paste into email or Slack as-is.

-------------------------------------------------------------------------------
TECHNICAL PITCH
-------------------------------------------------------------------------------

The demo is set in ghost kitchen / delivery operations: multiple brands and locations, customer orders that move through a lifecycle (placed → kitchen → driver → delivered), and two downstream workflows—refund recommendations per delivered order and customer complaints with triage and suggested responses. Against that backdrop: this repo is a single, runnable blueprint for a modern data-and-AI platform: event streams, medallion pipelines, LLM agents that query your data via tools and functions, reverse ETL into Postgres, and apps that consume it—all fed by realistic simulated events (replay, growth curves, temporal patterns) so it feels like production, not toy data. Run one job and you get a live refund recommender and optional complaint-triage agent, with recommendations and responses synced to Postgres and a Refund Manager–style app. You can see the full stack in one afternoon.

-------------------------------------------------------------------------------
BUSINESS / DOMAIN PITCH
-------------------------------------------------------------------------------

The demo is set in ghost kitchen / delivery: multiple locations and brands, customer orders, and deliveries. The two workflows we care about are (1) after each delivery, should we offer a refund and why, and (2) when a customer complains, how do we categorize it, decide (e.g. credit vs escalate), and reply. With that in mind: this repo shows a working example of operations augmented by AI: the system looks at every completed order and suggests whether to offer a refund and why (e.g. late delivery), and optionally generates and triages customer complaints—suggesting category, decision (e.g. credit vs escalate), and a reply—so you have a clear starting point instead of a blank screen. The demo runs on realistic-looking orders and deliveries (peaks, growth, multiple locations), so it feels credible, not like fake data. One run gives you both the refund recommendations and complaint handling workflows end to end—you can see the value in an afternoon.

-------------------------------------------------------------------------------
REALISM & AI PITCH (data, geography, AI)
-------------------------------------------------------------------------------

The demo is set in ghost kitchen / delivery: multiple brands and locations, orders, and two workflows—refund recommendations and complaint handling. What makes it feel real: the data is generated with realistic time patterns (lunch and dinner peaks, day-of-week, growth or decline by location) and replayed so you see a continuous stream of orders and deliveries. Geography is real: customer and kitchen locations sit on real road networks (OpenStreetMap), routes and drive times come from shortest-path routing, and driver pings follow the route so you get plausible GPS-style updates. The AI side is real too: an LLM-based refund agent looks at each delivered order (and can query order details and delivery times) and suggests refund amount and reason; complaint text is AI-generated from categories (e.g. late delivery, missing items), and a complaint agent triages each one and suggests category, decision (credit vs escalate), and a reply. So you get realistic data, realistic geo, and real AI in the loop—not scripted answers.
