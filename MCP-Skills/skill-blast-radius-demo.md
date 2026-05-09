# Skill — Blast Radius Demo (the 30-second CXO showcase)

## Description
The single most important MCP skill for an executive demo. One question. The agent walks the entire ontology, calls 5+ MCP tools, and returns a complete cross-functional briefing in **under 30 seconds** — the kind of briefing that today takes a war room of three to five teams two days to assemble.

## When to use
- A C-suite demo, board-level prep, or analyst showcase.
- The "OK, but what does the AGENT actually do?" moment.
- Any time you need to land the *one* memorable thing about Fabric Ontology MCP in 30 seconds.

## The hero question (just this one — practiced, polished)

> **"Recall `REC-2026-014` just opened against the NMC battery cell lot. Tell me everything that matters in 30 seconds: who's exposed, what's it costing, where it traces back to, and what we do next."**

## What the agent does (live, on-screen, with telemetry of every tool call)

The agent should be configured to **show its work** — every MCP tool call appears in the side panel as it executes. This is the demo. The answer is the proof.

### Tool calls (the choreography)

1. `list_entities()` → confirms 21 entities are loaded.
2. `describe_entity('RecallCampaign')` → grabs `RecallId`, `Recall_Title`, `Recall_OpenDt`, `Recall_State`.
3. `describe_entity('ConnectedVehicle')` → confirms VIN spine.
4. `find_paths('RecallCampaign', 'WarrantyClaim')` → discovers `AFFECTS_VEHICLE → HAS_CLAIM` (2 hops).
5. `find_paths('RecallCampaign', 'Supplier')` → discovers `CAUSED_BY_QE → AFFECTS_ASM → USES_COMPONENT → SUPPLIED_BY` (4 hops).
6. `find_paths('RecallCampaign', 'DealerLocation')` → discovers `AFFECTS_VEHICLE → IS_VARIANT ← CERTIFIED_FOR` (3 hops).
7. `run_gql(...)` × 3 — one per scope dimension (financial, supply chain, dealer routing).
8. (Optional) `query_timeseries('DealerLocation', 'Dealer_OpenBays', last 1h)` — pulls real-time capacity.
9. The agent assembles the result into a 4-bullet executive briefing.

### The 4-bullet briefing the agent returns

> ⚡ **REC-2026-014 — NMC Battery Pack Thermal Runaway — opened 2026-05-04**
>
> **Scope** — 8 VINs in field across USA (5), Germany (2), China (1), France (1). All NMC811 packs. Variants: NovaDrive EQ5 AMG Line + EQ5 Luxury.
>
> **Financial exposure** — $17,918 already filed in warranty claims. Predicted full-pack-replacement exposure: ~$67,200. Plus IRA Section 30D mineral-compliance failure on 5 US EVs = $37,500 lost customer-incentive value.
>
> **Root cause trace** — All 8 packs from VoltCell Energy Systems (Wuhan plant), lot LOT-2026-NMC-CN-0447. Caused by quality event QE-001 (cell thermal runaway in qualification test, raised 2026-05-03). TSB-2026-BMS-014 (NMC pack thermal containment retrofit) is the documented procedure.
>
> **Action** — Route 8 VINs to: Detroit Flagship + Boston East (USA, 5 VINs), Stuttgart Hauptstrasse + Munich (Germany, 2), Shanghai Pudong (1), Paris 16e (1). Capacity check: all dealers have ≥1 open bay and ≥1 recall kit in stock right now (live `Dealer_OpenBays` / `Dealer_KitsInStk` telemetry). NHTSA notification deadline: 5 business days. Customer outreach: synthetic CRM identifies 8 owners; recommend personalized email + courtesy loaner.

## What to say while the agent works

> *"What you're watching is the agent reading the ontology like a book. It's calling tools — `list_entities`, `describe_entity`, `find_paths`, `run_gql` — to discover the relationships it needs, then composing a single graph traversal. The graph already encodes the meaning of 'recall', 'warranty exposure', 'supplier', 'plant', 'dealer capacity'. The agent doesn't need to know SQL. It doesn't need to know our 49 Lakehouse tables. It just needs to ask the question in our business language. **Today this briefing takes a war room of 3-5 teams two days. The agent did it in under 30 seconds.**"*

## What to say after

> *"And the same MCP endpoint, the same ontology, answers the OTA software question for the security team, the IRA compliance question for the CFO, and the supply-chain risk question for the CPO. **Define the meaning once, every agent speaks it.** That's the bet."*

## What NOT to claim

- This is **preview** functionality. Do not promise GA dates.
- The 30-second timing requires a warm graph and a fast model — practice the demo against a real environment first.
- The IRA $7,500 number is real (US IRA Section 30D enacted law). The $17,918 warranty exposure is from this demo's synthetic data — do not represent it as a real customer figure.

## Demo-day setup

1. Have `Ontology/ontology-diagram-slide.html` open in a second tab.
2. Have the MCP endpoint URL pre-pasted into the agent client.
3. Have the model selected (Claude Sonnet 4.5, GPT-5, or Gemini 2.5 Pro all work; Claude tends to give the cleanest tool-call narration).
4. Practice the question one time before the room. Just one.
5. After the bullet briefing lands, **stop talking** for 3 seconds. Let it land.
