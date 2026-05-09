# Skill — Ontology Explorer (start here)

## Description
Discover and navigate the **OEM Connected Vehicle Lifecycle** ontology — entity types, properties, relationships, and the cross-domain paths that make the graph valuable. Use this skill any time someone asks *"what can this ontology answer?"* or you need to onboard a new business user to the model.

## When to use
- "What entities exist in this ontology?"
- "How are vehicles connected to recalls?"
- "Show me everything in the warranty domain"
- "What is the path from a customer to a battery passport?"
- "I'm new — give me the 5-minute tour"

## Ontology overview

**Scale:** 21 Entity Types · 29 Relationships · 5 Timeseries Streams · ~530 instances · VIN-anchored.

### Domain map

| Domain | Entities | Key Entity Types |
|---|---|---|
| **Manufacturing (seed)** | 6 | Assembly, Component, Supplier, Facility, ProductionOrder, QualityEvent |
| **Customer & Connected** | 4 | ConnectedVehicle, CustomerProfile, VehicleOwner, DriveSession |
| **Software / OTA** | 3 | VehicleVariant, SoftwareRelease, OTAUpdate |
| **Service & Warranty** | 6 | ServiceVisit, RepairTicket, TechServiceBul, WarrantyClaim, WarrantyPolicy, DealerLocation |
| **Compliance & Recall** | 2 | RecallCampaign, BatteryPassport |

### Cross-domain connection paths (the queries that produce ROI)

1. **Recall blast radius** — `RecallCampaign → AFFECTS_VEHICLE → ConnectedVehicle → HAS_CLAIM → WarrantyClaim`
2. **Battery cell genealogy** — `ConnectedVehicle → HAS_BATT_PASS → BatteryPassport → TRACES_COMP → Component → SUPPLIED_BY → Supplier → OPERATES_AT → Facility`
3. **Software patch coverage** — `SoftwareRelease (CVE) → DEPLOYS_VERSION ← OTAUpdate ← RECEIVED_OTA ← ConnectedVehicle`
4. **Customer to warranty** — `CustomerProfile → CUST_OWNS → VehicleOwner → OWNS_VEHICLE → ConnectedVehicle → HAS_CLAIM → WarrantyClaim → COVERED_BY → WarrantyPolicy`
5. **Plant quality to field** — `QualityEvent → AFFECTS_ASM → Assembly → INSTALLED_IN → ConnectedVehicle → HAS_CLAIM → WarrantyClaim`
6. **Mineral compliance** — `ConnectedVehicle → HAS_BATT_PASS → BatteryPassport → TRACES_COMP → Component → SUPPLIED_BY → Supplier → OPERATES_AT → Facility (mining country)`
7. **Dealer recall routing** — `RecallCampaign → AFFECTS_VEHICLE → ConnectedVehicle → HAD_SERVICE → ServiceVisit → AT_DEALER → DealerLocation` with `Dealer_OpenBays` capacity timeseries
8. **Vehicle genealogy** — `ProductionOrder → BUILT_AT → Facility` and `Assembly → INSTALLED_IN → ConnectedVehicle ← BUILT_FROM_PO ← ProductionOrder`

### Relationship directory (29 total)

| Relationship | Source → Target | Domain |
|---|---|---|
| IS_VARIANT | ConnectedVehicle → VehicleVariant | Connected |
| HAS_SW_RELEASE | VehicleVariant → SoftwareRelease | Software |
| BUILT_FROM_PO | ConnectedVehicle → ProductionOrder | Manufacturing |
| HAS_BATT_PASS | ConnectedVehicle → BatteryPassport | Compliance |
| HAD_SESSION | ConnectedVehicle → DriveSession | Connected |
| RECEIVED_OTA | ConnectedVehicle → OTAUpdate | Software |
| HAS_CLAIM | ConnectedVehicle → WarrantyClaim | Warranty |
| HAD_SERVICE | ConnectedVehicle → ServiceVisit | Service |
| BUILT_AT | ProductionOrder → Facility | Manufacturing |
| INSTALLED_IN | Assembly → ConnectedVehicle | Manufacturing |
| USES_COMPONENT | Assembly → Component | Manufacturing |
| AFFECTS_ASM | QualityEvent → Assembly | Quality |
| DEPLOYS_VERSION | OTAUpdate → SoftwareRelease | Software |
| CUST_OWNS | CustomerProfile → VehicleOwner | Customer |
| OWNS_VEHICLE | VehicleOwner → ConnectedVehicle | Customer |
| AT_DEALER | ServiceVisit → DealerLocation | Service |
| LEADS_TO_REPAIR | ServiceVisit → RepairTicket | Service |
| FOLLOWS_TSB | RepairTicket → TechServiceBul | Service |
| COVERED_BY | WarrantyClaim → WarrantyPolicy | Warranty |
| RESOLVED_BY | WarrantyClaim → RepairTicket | Warranty |
| CAUSED_BY_QE | RecallCampaign → QualityEvent | Recall |
| AFFECTS_VARIANT | RecallCampaign → VehicleVariant | Recall |
| AFFECTS_VEHICLE | RecallCampaign → ConnectedVehicle | Recall |
| REQUIRES_TSB | RecallCampaign → TechServiceBul | Recall |
| ISSUED_FOR | BatteryPassport → ConnectedVehicle | Compliance |
| TRACES_COMP | BatteryPassport → Component | Compliance |
| SUPPLIED_BY | Component → Supplier | Supply Chain |
| OPERATES_AT | Supplier → Facility | Supply Chain |
| CERTIFIED_FOR | DealerLocation → VehicleVariant | Service |

### Timeseries entities (Eventhouse)

| Entity | Metrics | Cadence |
|---|---|---|
| ConnectedVehicle | CV_SoCPct, CV_BattTempC, CV_DTCCount, CV_SpeedKmh, CV_RangeKm, CV_OnlineFlag | per-trip rollup |
| DriveSession | DS_AvgSpdKmh, DS_HrshBrkCnt, DS_MaxBattTempC, DS_EnergyKWh | per session |
| Assembly | Asm_TempC, Asm_TorqueNm, Asm_CycleSec | per build cycle |
| Facility | Fac_EnergyKWh, Fac_HumidPct, Fac_ProdRateHr | hourly |
| DealerLocation | Dealer_OpenBays, Dealer_KitsInStk | hourly |

## Demo questions for this skill

### Q1 — Path discovery (the wow moment)

> **"What is the shortest path from a `Supplier` to a `WarrantyClaim` in this ontology, and what business question does that path answer?"**

**Why this wows.** The agent reasons across the entire ontology to find: `Supplier ← SUPPLIED_BY ← Component ← USES_COMPONENT ← Assembly → INSTALLED_IN → ConnectedVehicle → HAS_CLAIM → WarrantyClaim` (5 hops). Then it explains the *business meaning*: **"Which warranty claims trace back to a specific supplier's components?"** That is the supplier-quality-scoring conversation a Quality VP has been trying to have with their data team for years.

### Q2 — Domain expansion impact analysis

> **"If we wanted to add a new `LeasingContract` entity for fleet customers, which existing entities and relationships would it need to connect to?"**

**Why this wows.** The agent reasons that LeasingContract would need to connect to: `CustomerProfile` (the lessee), `VehicleOwner` (point-in-time ownership becomes lease-period), `ConnectedVehicle` (the leased VIN), `WarrantyPolicy` (lease-specific coverage), `DealerLocation` (the issuing dealer). Demonstrates the ontology as a *living architecture blueprint* — not just a query target.

### Q3 — Hidden semantic gap

> **"This ontology models warranty claims and recalls. What's missing if we wanted to model **goodwill repairs** — repairs that aren't covered by warranty but that the OEM pays for to retain customer goodwill?"**

**Why this wows.** The agent reasons that today, RepairTicket exists, but there is no entity distinguishing warranty-paid vs. goodwill-paid vs. customer-paid. The fix: add a `RepairFundingType` property on `RepairTicket`, or introduce a `GoodwillPolicy` entity parallel to `WarrantyPolicy`. The ontology becomes a *gap-finder for unmodeled business reality*.

## Instructions for the agent

When a user asks about ontology structure:
1. Use the MCP `list_entities` tool to enumerate all 21 entity types if you don't already know them.
2. Use `describe_entity` to introspect properties, key column, and relationships for any specific entity.
3. Use `find_paths` (or graph traversal) when asked about how two entities connect.
4. Always emphasize that the ontology is a **live, queryable semantic layer** — not just documentation. Every relationship in the directory above is an *executable* edge, not just a diagram label.
5. When proposing a new entity, list both the connections it needs (relationships) **and** the global property uniqueness implications (every property must be ≤26 chars and unique across all entities).
