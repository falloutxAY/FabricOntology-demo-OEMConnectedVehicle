# MCP Skills — OEM Connected Vehicle Lifecycle

> **What this folder is.** Pre-packaged demo prompts and skills that show the **Fabric Ontology MCP endpoint** in action. Each skill is one business team's-eye view into the same shared semantic graph. Same data, same ontology, six different ways to make a business unit nod and say *"yes, that's exactly the question I keep failing to get answered."*

## What is the Ontology MCP endpoint?

A Fabric Ontology item exposes a **Model Context Protocol (MCP)** endpoint over HTTP/SSE:

```
https://api.fabric.microsoft.com/v1/mcp/dataPlane/workspaces/<ws-id>/items/<item-id>/ontologyEndpoint
```

Any MCP-aware client — VS Code Copilot agent mode, Claude Desktop, Azure AI Foundry agents, or a custom MCP host — can connect, discover tools, and ask questions in **natural language**. The MCP server translates them into GQL graph traversals against the ontology, executes them, and returns business-shaped answers — not row sets.

```
┌─────────────────────────────┐
│  MCP CLIENT                 │   "Which VINs are exposed to recall REC-2026-014
│  - VS Code agent mode       │    and what is the warranty exposure?"
│  - Claude Desktop           │           │
│  - Azure AI Foundry         │           ▼
│  - Custom MCP host          │   ┌──────────────────────────────┐
└─────────────┬───────────────┘   │  ONTOLOGY MCP TOOLS          │
              │ HTTP / SSE          │  - list_entities             │
              ▼                     │  - describe_entity           │
   ┌──────────────────────────┐    │  - find_paths                │
   │  FABRIC ONTOLOGY ITEM    │    │  - run_gql                   │
   │  - 21 entities           │    │  - nl2ontology (NL → GQL)    │
   │  - 29 relationships      │    └──────────────────────────────┘
   │  - 5 timeseries streams  │            │
   └─────────┬────────────────┘            ▼
             │                    Single GQL graph traversal
   ┌─────────┴────────────┐       MATCH (r:RecallCampaign {RecallId: 'REC-2026-014'})
   │  Lakehouse           │              -[:AFFECTS_VEHICLE]->(cv:ConnectedVehicle)
   │  (static, 49 tables) │              -[:HAS_CLAIM]->(wc:WarrantyClaim) ...
   │  Eventhouse          │
   │  (5 timeseries)      │
   └──────────────────────┘
```

## Why MCP changes the demo

Before MCP, the demo story was *"the ontology lets a developer write 1 GQL query instead of 8 JOINs."* True, but a developer story.

With the MCP endpoint, the story becomes *"the ontology lets a **business user** ask in plain English, and any agent — Claude, GPT-5, Copilot — answers correctly because the graph already encodes the meaning."* That is a **business user** story. Big difference in the room.

| Before MCP (NL2SQL / NL2DAX) | With Ontology MCP |
|---|---|
| Agent must learn 49 table names + FK columns | Agent sees 22 business concepts (`ConnectedVehicle`, `WarrantyClaim`, `ComplianceRule`, ...) |
| Multi-hop = orchestrating N JOINs in prompt | Multi-hop = single typed graph walk |
| Schema drift breaks the agent | Schema-of-meaning is centrally governed |
| One agent per source system | One graph for every agent |
| Timeseries = separate temporal-JOIN logic | Timeseries lives on the entity, queried like static |
| "Vehicle that has NOT had service" requires anti-join | Graph absence semantics are first-class |
| Regulations hard-coded in SQL/GQL | **`ComplianceRule` entity holds rules in plain English; agent reads + applies. Compliance owns it, not Engineering.** |

## Eight skills in this folder

| File | Audience | Hero Question |
|---|---|---|
| [`skill-ontology-explorer.md`](skill-ontology-explorer.md) | Anyone — start here | "What can this ontology answer?" |
| [`skill-rules-explorer.md`](skill-rules-explorer.md) | **Compliance / Legal / Auditors** | "What rules currently govern X, what do they actually say, and would we pass an audit today?" |
| [`skill-quality-recall.md`](skill-quality-recall.md) | Quality / Manufacturing Ops | "Which Critical defects this week trace to which suppliers and plants? Are we within NHTSA timing?" |
| [`skill-connected-vehicle-software.md`](skill-connected-vehicle-software.md) | Connected / SDV / Security | "Which VINs are running vulnerable software, and which are past the Critical-CVE 30-day SLA?" |
| [`skill-warranty-service.md`](skill-warranty-service.md) | Warranty / Aftersales | "Where is warranty cost concentrated? Who qualifies for goodwill per `CR-IW-GOOD`?" |
| [`skill-sustainability-compliance.md`](skill-sustainability-compliance.md) | ESG / Compliance / Finance | "Which EVs would fail an IRA §30D mineral-origin audit per the rule as currently written?" |
| [`skill-supply-chain.md`](skill-supply-chain.md) | Procurement / Supply Chain | "If a single supplier or plant goes dark, which platforms and revenue are at risk?" |
| [`skill-blast-radius-demo.md`](skill-blast-radius-demo.md) | **CXO showcase** — the headline | "30 seconds: tell me everything that matters about recall REC-2026-014." |

## How to use a skill in a live demo

1. Connect VS Code to the Ontology MCP endpoint (Settings → MCP Servers → Add).
2. Open the skill markdown file.
3. Copy the hero question into the agent.
4. The agent introspects the ontology via the MCP tool catalog (`list_entities`, `describe_entity`, `find_paths`), translates the question to GQL, runs it, and returns a business-shaped answer.
5. Walk the audience through *why each MCP tool was called* — that is the demo. The query result is just the proof.

## Demo-day setup checklist

- [ ] Ontology item published, all bindings green.
- [ ] Eventhouse ingestion confirmed (run a `take 1` against each telemetry table).
- [ ] MCP endpoint URL copied into client config.
- [ ] Auth token / managed identity working — try `list_entities` as a smoke test.
- [ ] Demo browser tab on `Ontology/ontology-diagram-slide.html` ready for the architecture moment.
- [ ] One question rehearsed live; the rest can be discovered live with the audience.

## Closing pitch (the line to land in the room)

> *"Your Lakehouse and Semantic Model are not going anywhere. The Ontology MCP endpoint just makes them speak the language your business already speaks. Define the meaning once, and every agent — Copilot, Claude, GPT — speaks it."*
