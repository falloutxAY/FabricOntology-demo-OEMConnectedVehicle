# Skill — Warranty & Service Intelligence

## Description
The Warranty / Aftersales / Dealer Operations team's eye into the ontology. Concentration analysis on warranty cost (by component, lot, supplier, plant, policy), service capacity routing, repair-ticket throughput, and goodwill identification. **Use when** a Warranty VP, Aftersales Director, or Dealer Network manager asks *"where is the cost going and how do we reduce it?"*

## When to use
- "Where is warranty cost concentrated this week?"
- "Which DTC codes are driving the most claims?"
- "Which dealers are absorbing the most recall volume?"
- "Which open repair tickets are blocked on parts?"
- "Show me the high-dollar claims that haven't been approved yet"
- "Which TSBs are most often referenced — can we automate that procedure?"

## Hero question

> **"Where is warranty cost concentrated by **policy type** and **DTC code** this week, and which suppliers' components are driving it?"**

### What the agent does

1. `describe_entity('WarrantyClaim')` — confirms `WClaim_CostUSD`, `WClaim_DTC`, `WClaim_FilingDt`.
2. `find_paths('WarrantyClaim', 'Supplier')` — finds `← HAS_CLAIM ← ConnectedVehicle ← INSTALLED_IN ← Assembly → USES_COMPONENT → Component → SUPPLIED_BY → Supplier` (5 hops).
3. `run_gql` — aggregates total cost grouped by policy + DTC + supplier.

### GQL

```gql
MATCH (cv:ConnectedVehicle)-[:HAS_CLAIM]->(wc:WarrantyClaim)-[:COVERED_BY]->(p:WarrantyPolicy)
MATCH (cv)<-[:INSTALLED_IN]-(a:Assembly)-[:USES_COMPONENT]->(c:Component)-[:SUPPLIED_BY]->(s:Supplier)
FILTER wc.WClaim_FilingDt >= ZONED_DATETIME('2026-05-01T00:00:00Z')
LET coverage = p.WPol_Coverage
LET dtc = wc.WClaim_DTC
LET supplier = s.Sup_Name
LET claimCount = count(*)
LET totalCostUSD = sum(wc.WClaim_CostUSD)
RETURN coverage, dtc, supplier, claimCount, totalCostUSD
GROUP BY coverage, dtc, supplier
ORDER BY totalCostUSD DESC
```

### Business answer

> *Top 3 cost concentrations this week:*
> - *EVPack policy + DTC P0A8B (battery pack failure) + VoltCell Energy Systems → **$8,400** in 1 claim (single high-cost battery replacement)*
> - *EVPack policy + DTC P0A0F (BMS fault) + VoltCell Energy Systems → **$7,800** in 1 claim*
> - *Powertrain policy + DTC P0A1A (drive unit) + InnerveDrive Motors → **$3,850** in 1 claim*
> *VoltCell-supplied components are driving 2 of 3 top concentrations and recall REC-2026-014 hasn't even started reimbursing dealers yet. Recommend escalating with VoltCell on lot LOT-2026-NMC-CN-0447 and accelerating supplier-recovery.*

## Second question — service-capacity routing

> **"Recall REC-2026-014 covers 8 VINs. Route each to its nearest EV-certified dealer with the most open bays and the most recall kits in stock right now."**

### GQL

```gql
MATCH (r:RecallCampaign {RecallId: 'REC-2026-014'})-[:AFFECTS_VEHICLE]->(cv:ConnectedVehicle)
MATCH (cv)-[:IS_VARIANT]->(v:VehicleVariant)<-[:CERTIFIED_FOR]-(d:DealerLocation)
FILTER d.Dealer_Country = cv.CV_RegMarket AND d.Dealer_EVCert = true
LET vin = cv.VIN
LET market = cv.CV_RegMarket
LET dealer = d.Dealer_Name
LET dealerCity = d.Dealer_City
LET openBays = d.Dealer_OpenBays
LET kitsInStk = d.Dealer_KitsInStk
RETURN vin, market, dealer, dealerCity, openBays, kitsInStk
ORDER BY vin, kitsInStk DESC, openBays DESC
```

### Routing answer

> *Eight VINs allocated:*
> - *USA (5 VINs) → Detroit Flagship has 2 open bays + 3 kits; Boston East has 5 + 5; **route 3 VINs Detroit, 2 VINs Boston**.*
> - *Germany (2 VINs) → Stuttgart Hauptstrasse 1 bay + 6 kits; Munich 3 bays + 4 kits; **route 1 each**.*
> - *China (1 VIN) → Shanghai Pudong 5 bays + 7 kits; **route there**.*
> - *France (1 VIN) → Paris 16e 3 bays + 4 kits; **route there**.*
> *No geographic black holes. All 8 routed within country.*

## Third question — proactive service before warranty escalates

> **"Which VINs in the field have emitted any DTC pattern that has a corresponding TSB, but have NOT yet had the TSB performed?"**

```gql
MATCH (cv:ConnectedVehicle)-[:HAS_CLAIM]->(wc:WarrantyClaim)
MATCH (cv)-[:HAD_SERVICE]->(sv:ServiceVisit)-[:LEADS_TO_REPAIR]->(rt:RepairTicket)-[:FOLLOWS_TSB]->(t:TechServiceBul)
LET vin = cv.VIN
LET dtc = wc.WClaim_DTC
LET claimState = wc.WClaim_State
LET tsbDone = t.TSBId
LET claimCost = wc.WClaim_CostUSD
RETURN vin, dtc, claimState, tsbDone, claimCost
ORDER BY claimCost DESC
```

## Fourth question — rule-aware goodwill recommendation

> **"Apply the internal goodwill policy (`CR-IW-GOOD`) to my open repair tickets. For any vehicle that's just outside warranty AND has a failure that matches an active TSB, recommend goodwill treatment with the rule cited."**

### What the agent does

1. `run_gql` to fetch `CR-IW-GOOD` and **read its `CR_NLText`** — the rule states *"within 5% of warranty mileage boundary OR within 60 days of warranty expiration, cover 50% labor + 100% TSB-mandated parts."*
2. Traverse `ComplianceRule → GOVERNS_POLICY → WarrantyPolicy` to find which warranty policies the rule applies to.
3. Cross-reference with VIN ownership age + odometer to find vehicles within the goodwill boundary.
4. Recommend goodwill state for matching repair tickets, citing `CR-IW-GOOD`.

### GQL

```gql
MATCH (rule:ComplianceRule {RuleId: 'CR-IW-GOOD'})-[:GOVERNS_POLICY]->(p:WarrantyPolicy)<-[:COVERED_BY]-(wc:WarrantyClaim)<-[:HAS_CLAIM]-(cv:ConnectedVehicle)
MATCH (cv)-[:HAD_SERVICE]->(sv:ServiceVisit)-[:LEADS_TO_REPAIR]->(rt:RepairTicket)-[:FOLLOWS_TSB]->(t:TechServiceBul)
FILTER rt.Repair_State = 'Open'
LET vin = cv.VIN
LET odoKm = cv.CV_OdoKm
LET kmLimit = p.WPol_KmLimit
LET coverage = p.WPol_Coverage
LET ticketId = rt.TicketId
LET tsbId = t.TSBId
LET claimCost = wc.WClaim_CostUSD
LET ruleCited = rule.RuleId
RETURN vin, odoKm, kmLimit, coverage, ticketId, tsbId, claimCost, ruleCited
ORDER BY odoKm DESC
```

### Business answer

> *Two open repair tickets on vehicles approaching their warranty boundary. Per `CR-IW-GOOD`, both qualify for goodwill: cover 50% labor + 100% TSB parts. The agent recommends flipping `WClaim_State` from 'Submitted' to 'Goodwill' on the linked claims, capping OEM exposure at 50% of labor cost while preserving customer NPS. **Rule cited:** `CR-IW-GOOD`, NovaDrive-Internal, effective 2024-01-01.*

## Why MCP changes this skill

A traditional Warranty Director gets a weekly Power BI report. With the MCP endpoint and this skill, they get an agent that *answers follow-up questions in real time*, including the dealer-capacity routing that requires telemetry data, all in one conversation.
