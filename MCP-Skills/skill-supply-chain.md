# Skill — Supply Chain & Procurement

## Description
The Procurement / Supply Chain Risk / Operations team's eye into the ontology. Single-source supplier exposure, geographic concentration risk, plant-revenue impact analysis, and supplier-quality scoring. **Use when** the CPO, COO, or Supply Chain Director asks *"if X goes dark, what breaks?"* — the question the 2021 chip shortage taught the industry to fear.

## When to use
- "If supplier X goes dark for 6 weeks, which platforms and revenue are at risk?"
- "Which components are single-sourced from high-risk geopolitical zones?"
- "Show me supplier quality scores correlated with field defect rates"
- "Which plants depend on which Tier-2 suppliers?"
- "What's our exposure to a Wuhan plant disruption?"

## Hero question (the chip-shortage moment, but for any supplier)

> **"If `SUP-VOLT` (VoltCell Energy Systems, Wuhan) experiences a 6-week production halt starting today, which vehicle variants and how many VINs in the build pipeline are at risk?"**

### What the agent does

1. `describe_entity('Supplier')` — confirms `Sup_Tier`, `Sup_Country`, `Sup_Rating`, `Sup_Certified`.
2. `find_paths('Supplier', 'VehicleVariant')` — finds `← SUPPLIED_BY ← Component ← USES_COMPONENT ← Assembly → INSTALLED_IN → ConnectedVehicle → IS_VARIANT → VehicleVariant` (6 hops).
3. `find_paths('Supplier', 'ProductionOrder')` — same chain plus the ConnectedVehicle → BUILT_FROM_PO step.
4. `run_gql` — aggregates by variant, by PO, by plant.

### GQL

```gql
MATCH (s:Supplier {SupplierId: 'SUP-VOLT'})<-[:SUPPLIED_BY]-(c:Component)<-[:USES_COMPONENT]-(a:Assembly)-[:INSTALLED_IN]->(cv:ConnectedVehicle)
      -[:IS_VARIANT]->(v:VehicleVariant)
MATCH (cv)-[:BUILT_FROM_PO]->(po:ProductionOrder)-[:BUILT_AT]->(f:Facility)
LET supplier = s.Sup_Name
LET componentName = c.Comp_Name
LET variant = v.VV_Name
LET plant = f.Fac_Name
LET poNumber = po.PO_Number
LET poQty = po.PO_Quantity
LET vinCount = count(DISTINCT cv)
RETURN supplier, componentName, variant, plant, poNumber, poQty, vinCount
GROUP BY supplier, componentName, variant, plant, poNumber, poQty
ORDER BY vinCount DESC
```

### Business answer

> *VoltCell Energy Systems disruption impact:*
> - *3 active production orders affected (PO-2026-DET-001 / 002 at Detroit, PO-2026-STG-001 at Stuttgart, PO-2026-WUH-001 at Wuhan).*
> - *Total in-pipeline VINs: 8 already-built + ~3,300 across the open POs.*
> - *Variants impacted: NovaDrive EQ5 AMG Line, EQ5 Luxury, Light SUV (NMC pack variants only — LFP variants OK).*
> - *Plants at risk: all three NovaDrive assembly plants (Detroit, Stuttgart, Wuhan).*
> - ***Mitigation:** KSpark Cells (LFP, South Korea, FTA-eligible) is the only qualified alt; lead time 8-10 weeks for revalidation. Recommend immediate air-freight expediting from KSpark while qualifying second NMC source. **Daily revenue at risk: ~$10–25 M depending on plant.***

## Second question — supplier-quality scorecard

> **"Rank suppliers by total field warranty cost on components they supplied this week, alongside their `Sup_Rating` and `Sup_Certified` status."**

```gql
MATCH (cv:ConnectedVehicle)<-[:INSTALLED_IN]-(a:Assembly)-[:USES_COMPONENT]->(c:Component)-[:SUPPLIED_BY]->(s:Supplier)
MATCH (cv)-[:HAS_CLAIM]->(wc:WarrantyClaim)
FILTER wc.WClaim_FilingDt >= ZONED_DATETIME('2026-05-01T00:00:00Z')
LET supplier = s.Sup_Name
LET supplierCountry = s.Sup_Country
LET supRating = s.Sup_Rating
LET certified = s.Sup_Certified
LET totalCostUSD = sum(wc.WClaim_CostUSD)
LET claimCount = count(wc)
RETURN supplier, supplierCountry, supRating, certified, claimCount, totalCostUSD
GROUP BY supplier, supplierCountry, supRating, certified
ORDER BY totalCostUSD DESC
```

### Pattern the agent finds

> *VoltCell has 4.3 rating but is now driving the most warranty cost ($16,800+ this week). InnerveDrive (4.8) and SafeAir (4.9) are clean. The Sup_Rating is a lagging indicator — the field-warranty-cost view is the leading indicator. **Recommendation:** add this query as a weekly procurement scorecard.*

## Third question — geographic concentration risk

> **"Show me, for each component, the supplier country and the country of the manufacturing plant where it's installed. Where do we have all-eggs-in-one-region risk?"**

```gql
MATCH (a:Assembly)-[:USES_COMPONENT]->(c:Component)-[:SUPPLIED_BY]->(s:Supplier)
MATCH (a)-[:INSTALLED_IN]->(cv:ConnectedVehicle)-[:BUILT_FROM_PO]->(po:ProductionOrder)-[:BUILT_AT]->(f:Facility)
LET componentName = c.Comp_Name
LET supplierCountry = s.Sup_Country
LET plantCountry = f.Fac_Country
LET vinCount = count(DISTINCT cv)
RETURN componentName, supplierCountry, plantCountry, vinCount
GROUP BY componentName, supplierCountry, plantCountry
ORDER BY vinCount DESC
```

### Insight

> *NMC battery cells: 100% sourced from China (VoltCell), installed at all 3 plants. **Single-region cell exposure.** LFP is the diversification — but LFP is also single-source (KSpark, South Korea). Recommend **dual-source qualification** is procurement priority #1.*

## Why MCP changes this skill

The CPO does not stitch SQL across PLM, supplier master, and the connected-car platform. With the MCP endpoint and this skill, they get an *executive-grade supply-chain risk dashboard* on demand — including the financial-revenue impact and the alternate-sourcing reasoning, in a single conversation.
