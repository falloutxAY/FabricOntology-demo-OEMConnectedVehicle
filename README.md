# OEM Connected Vehicle Lifecycle — NovaDrive Motors

> *From factory to customer to recall — one semantic graph, infinite questions.*

A full end-to-end Fabric Ontology demo for the **fictional OEM NovaDrive Motors**, modeling the complete connected-vehicle lifecycle. Built from the [`AutoManufacturing-SupplyChain`](../AutoManufacturing-SupplyChain/README.md) seed pattern and extended across **14 OEM value-chain modules** with the **VIN as the universal identity spine**.

## Why this demo exists

The 2021 chip shortage cost the auto industry **$210B**. Annual warranty claims run **$43.1B**. NHTSA averages **997 recalls / 29M vehicles per year**. None of these are data problems — they are **semantic connectivity problems**. Every crisis requires traversing four to seven systems (PLM, MES, ERP, connected-vehicle platform, warranty, DMS, recall registry) that share no common identity spine and no agreed vocabulary.

This demo shows what happens when you give an AI agent a **graph that already speaks the business**.

## Demo facts

| | |
|---|---|
| **Entities** | 22 (8 manufacturing seed + 13 lifecycle extensions + 1 governance) |
| **Relationships** | 33 — all bound via dedicated Edge tables (29 lifecycle + 4 governance) |
| **Compliance Rules** | 8 NL rules (IRA §30D, EU Battery Reg, NHTSA T1+T2, CSRD, internal warranty, goodwill, OTA-CVE SLA) |
| **Timeseries entities** | 5 (ConnectedVehicle, DriveSession, Assembly, Facility, DealerLocation) |
| **Lakehouse rows** | ~480 across 55 tables (22 Dim + 33 Edge) |
| **Eventhouse rows** | ~116 across 5 telemetry tables |
| **Identity spine** | `ConnectedVehicle.VIN` — every business unit joins through it |
| **Business units served** | Manufacturing · Quality · Supply Chain · Customer · Connected/SDV · Service & Warranty · Sustainability · Finance · Recall · **Compliance / Legal** |
| **Modules covered** | M02 (Engineering) + M04/M05 (Supply Chain + Manufacturing — seed) + M08 (Customer) + M09 (Connected) + M10 (After-Sales) + M11 (Warranty/Recall) + M14 (Sustainability) + **Governance overlay** |

## Folder structure

```
demo-OEMConnectedVehicle/
├── README.md                          (this file)
├── .demo-metadata.yaml
├── ontology-structure.md              (entity/relationship reference)
├── demo-questions.md                  (5 GQL business questions)
├── Ontology/
│   ├── oem-connected-vehicle-lifecycle.ttl
│   └── ontology-diagram-slide.html
├── Bindings/
│   ├── bindings.yaml                  (parser source of truth)
│   ├── lakehouse-binding.md
│   └── eventhouse-binding.md
├── Data/
│   ├── Lakehouse/                     (49 CSVs — Dim* + Edge*)
│   └── Eventhouse/                    (5 telemetry CSVs)
└── MCP-Skills/                        (the NEW Ontology MCP feature)
    ├── README.md
    ├── skill-ontology-explorer.md
    ├── skill-rules-explorer.md           (NEW — ComplianceRule reasoning)
    ├── skill-quality-recall.md
    ├── skill-connected-vehicle-software.md
    ├── skill-warranty-service.md
    ├── skill-sustainability-compliance.md
    ├── skill-supply-chain.md
    └── skill-blast-radius-demo.md
```

## Prerequisites

- Microsoft Fabric capacity F2+ (or P1 Power BI Premium with Fabric enabled)
- All 6 Fabric tenant settings enabled (Ontology preview, Graph preview, Data Agent, Copilot Azure OpenAI, cross-region processing, cross-region storage)
- A Lakehouse with **OneLake security DISABLED** and **schemas DISABLED**
- An Eventhouse and KQL database for the 5 telemetry tables
- (For the new MCP demo) a client that speaks MCP — VS Code agent mode, Azure AI Foundry, or any custom MCP host

## Quick start

1. **Upload data**
   - All 55 Lakehouse CSVs → load to Delta tables (header=true, infer schema). **Important:** `DimComplianceRule.csv` has multi-line natural-language values per row; ensure your loader handles RFC 4180 quoted strings correctly.
   - All 5 Eventhouse CSVs → ingest via the KQL commands in `Bindings/eventhouse-binding.md`.
2. **Create ontology** — upload `Ontology/oem-connected-vehicle-lifecycle.ttl` to a new Fabric Ontology item.
3. **Bind static entities** — follow `Bindings/lakehouse-binding.md` (22 entities; the key column must be the first property in each).
4. **Bind 33 relationships** — every relationship uses a dedicated Edge* table; never use a Dim* table as `sourceTable`. The 4 new `GOVERNS_*` relationships connect `ComplianceRule` to BatteryPassport, WarrantyPolicy, RecallCampaign, and SoftwareRelease.
5. **Bind 5 timeseries entities** — follow `Bindings/eventhouse-binding.md`.
6. **Refresh Graph** — Fabric Graph refresh is manual + full each time.
7. **Enable the MCP endpoint** — the Ontology item exposes an HTTP/SSE MCP endpoint at `https://api.fabric.microsoft.com/v1/mcp/dataPlane/workspaces/<ws-id>/items/<item-id>/ontologyEndpoint`.
8. **Run the 5 questions** in `demo-questions.md` and the 6 MCP skills in `MCP-Skills/`.

## Three scripted demo walkthroughs

### Walkthrough 1 — *The 30-second blast radius* (CXO audience)
1. Open VS Code with the MCP endpoint connected.
2. Type the question from `MCP-Skills/skill-blast-radius-demo.md`: *"Recall REC-2026-014 just opened. Tell me everything that matters in 30 seconds."*
3. Watch the agent: identify affected VINs → cross-reference warranty exposure → trace cell lot to supplier and plant → check dealer capacity to absorb the wave → propose a regional routing plan.
4. Pull-quote: **"Eight VINs, $17,918 in current warranty exposure, single supplier (VoltCell), single plant (Wuhan), three dealers with capacity for the next wave."**

### Walkthrough 2 — *Manufacturing quality before the customer ever sees it* (COO/Plant audience)
1. Run the quality-telemetry correlation question (`demo-questions.md` Q3 / `MCP-Skills/skill-quality-recall.md`).
2. Show ASM-006 and ASM-010 sitting +25 °C above plant average and +50 Nm above torque target.
3. Show that those exact stations produced the QCFailed assemblies and the Critical QualityEvents.
4. Pull-quote: **"The graph caught the process drift before the vehicles even left the plant."**

### Walkthrough 3 — *IRA + EU compliance at audit time* (CFO/Compliance audience)
1. Run the IRA mineral question (`demo-questions.md` Q4 / `MCP-Skills/skill-sustainability-compliance.md`).
2. Show 5 US EVs failing the FTA-country test (NMC811 from DR Congo).
3. Show the agent **traversing to `ComplianceRule.CR-IRA-30D` first**, reading `CR_NLText`, then applying the FTA list from the rule — *no hard-coded list anywhere in GQL*.
4. Open `MCP-Skills/skill-rules-explorer.md` and show the rule text directly. Edit one country in `DimComplianceRule.csv`; refresh; re-run; new answer. **Compliance owns the rule, not Engineering.**
5. Pull-quote: **"$37,500 of IRA credit is already lost on these 5 vehicles. The graph would have flagged it before the order was placed — and the rule is auditable text, not a SQL filter."**

## Known limitations (preview)

- Fabric Graph does not support `OPTIONAL MATCH` — design queries to require all relationships, or split into two queries and let the agent stitch.
- Graph refresh is manual and full each time (no auto-refresh on source change).
- Property names ≤ 26 characters and globally unique across all entities.
- No `Decimal` type; all monetary fields use `double`.
- `OneLake security` must be disabled on the bound Lakehouse.
- Cross-ontology GQL traversal is not currently confirmed in Fabric documentation — keep all 21 entities in a single Ontology item.

## Relationship to existing repo demos

| Existing | Role here |
|---|---|
| `AutoManufacturing-SupplyChain/` | Seed pattern for M04/M05 (Manufacturing + Supply Chain). The 6 reused entities (Component, QualityEvent, Supplier, Assembly, Facility, ProductionOrder) are kept as a first-class **manufacturing value lens** — they prove that the new lifecycle entities don't displace the old ones, they extend them. |
| `demo-MedDeviceSupplyChain/` | Pattern library for regulatory recall genealogy. |
| `demo-SoccerFanOps/MCP-Skills/` | Template for the MCP-Skills folder structure used here. |

## Credits and references

- Source research: `mcpdemo/automotive-fabric-ontology-mcp-demo-research.md`
- Generation guide: `.agentic/agent-instructions.md` (spec v3.7)
- Catena-X SLDT semantic models (CC-BY-4.0) for battery passport / quality / claim shape inspiration.
- COVESA VSS v7 (MPL-2.0) for the connected-vehicle telemetry signal taxonomy.
- All OEM names in this demo are fictional. NovaDrive Motors does not exist; any resemblance to a real OEM is intentional only at the *narrative* level (the four anchor stories — Mercedes-Benz / VW / Stellantis / Toyota — are documented in the source research file as positioning options).
