# Skill — Quality & Recall Intelligence

## Description
The Quality team's eye into the ontology. Trace defects from production telemetry to field warranty claims, scope recalls precisely, and find Critical events that cluster by station, supplier, or lot. **Use when** a quality engineer, recall coordinator, or plant director asks *"how big is this problem and who is on the hook?"*

## When to use
- "Which Critical quality events happened this week?"
- "Which assemblies with defects had abnormal process telemetry?"
- "Recall REC-2026-014 just opened — what's the scope?"
- "Show me defects that share the same supplier lot"
- "Which plant has the worst defect rate this week?"
- "If we recall every vehicle built between dates X and Y, how many is that?"

## Hero question (the demo moment)

> **"Recall `REC-2026-014` was opened on the NMC battery cell lot. Show me the full scope: VINs affected, market split, warranty cost already paid, and trace the lot back to supplier and plant."**

### What the agent does (call sequence)

1. `describe_entity('RecallCampaign')` — confirms `RecallId` is the key.
2. `find_paths('RecallCampaign', 'ConnectedVehicle')` — picks `AFFECTS_VEHICLE` (1 hop) and `CAUSED_BY_QE → AFFECTS_ASM → INSTALLED_IN` (3 hops) as alternatives.
3. `find_paths('RecallCampaign', 'WarrantyClaim')` — finds `AFFECTS_VEHICLE → HAS_CLAIM` (2 hops).
4. `find_paths('RecallCampaign', 'Supplier')` — finds the cell-lot trace via `CAUSED_BY_QE → AFFECTS_ASM → USES_COMPONENT → SUPPLIED_BY` (4 hops).
5. `run_gql(...)` — executes the query and aggregates by `Supplier`, `Facility`, `Market`.

### GQL the agent generates

```gql
MATCH (r:RecallCampaign {RecallId: 'REC-2026-014'})-[:AFFECTS_VEHICLE]->(cv:ConnectedVehicle)
      -[:HAS_BATT_PASS]->(bp:BatteryPassport)-[:TRACES_COMP]->(c:Component)
      -[:SUPPLIED_BY]->(s:Supplier)-[:OPERATES_AT]->(f:Facility)
MATCH (cv)-[:HAS_CLAIM]->(wc:WarrantyClaim)
LET vin = cv.VIN
LET market = cv.CV_RegMarket
LET supplier = s.Sup_Name
LET plant = f.Fac_Name
LET lot = c.Comp_LotId
RETURN vin, market, supplier, plant, lot, sum(wc.WClaim_CostUSD) AS exposureUSD
GROUP BY vin, market, supplier, plant, lot
ORDER BY exposureUSD DESC
```

### Expected business answer

> *Recall REC-2026-014 covers 8 VINs across USA (5), Germany (2), China (1), France (1). All 8 packs trace to **VoltCell Energy Systems** at the **Wuhan Battery Plant**, lot **LOT-2026-NMC-CN-0447**. Current warranty exposure already filed: **$17,918**. The full predicted exposure if all 8 vehicles need pack replacement at $8,400 average = **$67,200**. Recommend issuing TSB-2026-BMS-014 (already linked) to the 3 US dealers within 50 km of these owners (Detroit Flagship, Boston East, Dallas Fleet — all EV-certified, capacity from `Dealer_OpenBays` telemetry: 2/3/4 bays open).*

## Second question — quality–telemetry correlation

> **"For all Critical and Major quality events this week, what were the build-time temperature, torque, and cycle-time readings, and do they cluster by facility?"**

### GQL

```gql
MATCH (qe:QualityEvent)-[:AFFECTS_ASM]->(a:Assembly)-[:INSTALLED_IN]->(cv:ConnectedVehicle)
      -[:BUILT_FROM_PO]->(po:ProductionOrder)-[:BUILT_AT]->(f:Facility)
FILTER qe.QE_Severity IN ['Critical','Major']
LET event = qe.EventId
LET severity = qe.QE_Severity
LET facility = f.Fac_Name
LET avgTempC = a.Asm_TempC
LET avgTorqueNm = a.Asm_TorqueNm
LET cycleSec = a.Asm_CycleSec
RETURN event, severity, facility, avgTempC, avgTorqueNm, cycleSec
ORDER BY severity DESC, avgTempC DESC
```

### Pattern the agent finds

> *4 Critical battery defects this week. Two of them (ASM-006 at Stuttgart, ASM-010 at Wuhan) ran at **70 °C+** versus a plant average of **45 °C**, and torque was **180 Nm+** vs. target **130 Nm**. Both stations were also flagged QCFailed. Recommendation: pull the build sheets for those stations and check the torque-controller calibration history.*

## Third question — recall scope minimization

> **"Recall RC-2026-015 covers airbag lot LOT-AB-JP-2026-031. What's the VIN list (only vehicles that were actually built with that lot), and which dealers should we route them to?"**

### GQL (combined recall scope + dealer routing)

```gql
MATCH (cv:ConnectedVehicle)<-[:INSTALLED_IN]-(a:Assembly)-[:USES_COMPONENT]->(c:Component)
FILTER c.Comp_LotId = 'LOT-AB-JP-2026-031'
MATCH (cv)-[:IS_VARIANT]->(v:VehicleVariant)<-[:CERTIFIED_FOR]-(d:DealerLocation)
FILTER d.Dealer_Country = cv.CV_RegMarket
LET vin = cv.VIN
LET market = cv.CV_RegMarket
LET dealer = d.Dealer_Name
LET dealerCity = d.Dealer_City
LET evCert = d.Dealer_EVCert
LET openBays = d.Dealer_OpenBays
LET kitsInStk = d.Dealer_KitsInStk
RETURN vin, market, dealer, dealerCity, evCert, openBays, kitsInStk
ORDER BY vin, openBays DESC
```

## Fourth question — rule-aware NHTSA timing

> **"Apply NHTSA timing rule `CR-NHTSA-T1` to our open Critical recalls. Are we within the 5-day notification window and on track for the 60-day owner-outreach deadline?"**

### What the agent does

1. `run_gql` to fetch `CR-NHTSA-T1` and **read its `CR_NLText`** — the rule states *"5 business days to NHTSA notification, 60 days to owner notification, civil penalties up to $135M per violation."*
2. Traverse `ComplianceRule → GOVERNS_RECALL → RecallCampaign` to get the affected open recalls.
3. Compute `Recall_OpenDt` vs. today vs. the rule's deadlines.
4. Cite the rule in the answer.

### GQL

```gql
MATCH (rule:ComplianceRule {RuleId: 'CR-NHTSA-T1'})-[:GOVERNS_RECALL]->(rec:RecallCampaign)-[:CAUSED_BY_QE]->(qe:QualityEvent)
FILTER rec.Recall_State = 'Open' AND qe.QE_Severity = 'Critical'
LET recallId = rec.RecallId
LET recallTitle = rec.Recall_Title
LET nhtsaId = rec.Recall_NHTSAId
LET openDt = rec.Recall_OpenDt
LET ruleCited = rule.RuleId
LET authority = rule.CR_Authority
RETURN recallId, recallTitle, nhtsaId, openDt, ruleCited, authority
ORDER BY openDt
```

### Business answer

> *Per `CR-NHTSA-T1` (NHTSA, effective 2024-01-01, 5-day notify + 60-day owner-outreach):*
> *3 Critical recalls open. REC-2026-014 opened 2026-05-04 (4 days ago) — within the 5-day NHTSA notification window. Owner-notification deadline: 2026-07-03 (56 days remaining). REC-2026-015 opened 2026-05-05, REC-2026-016 opened 2026-05-07 — both still within both windows. **No violations imminent. Rule cited:** `CR-NHTSA-T1`, NHTSA, effective 2024-01-01.*

## Why MCP changes this skill

A traditional Quality Engineer needs to file a help-desk ticket with the data team for **every** new question. With the MCP endpoint and this skill, they ask in plain English and get the cross-system answer back in seconds — including the warranty financials and the dealer routing, which used to require pulling in three other teams.
