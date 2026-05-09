# Skill — Sustainability & Compliance

## Description
The ESG / Sustainability / Compliance / Regulatory team's eye into the ontology. Battery passport status, IRA Section 30D compliance, EU Battery Regulation 2023/1542 readiness, lifecycle CO2 hotspots, and supply-chain ESG risk. **Use when** the CFO, CSO, or Regulatory Affairs lead asks *"are we still eligible for [credit / market / certification]?"*

## When to use
- "Which US EVs would fail an IRA Section 30D mineral audit?"
- "Show me battery passport SoH distribution by mineral chemistry"
- "What is our average lifecycle CO2 by variant and by mineral source?"
- "Which suppliers are not ISO-certified, and which of our components do they touch?"
- "Where does our cobalt come from, by VIN?"
- "Which EU-registered EVs would pass a CSRD Scope 3 Category 11 audit?"

## Hero question (the IRA $7,500 moment) — rule-aware

> **"Apply rule `CR-IRA-30D` (IRA Section 30D Critical Mineral Origin) to our US-registered EVs and tell me which would FAIL today, citing the rule, and which dealers can serve them."**

### What the agent does

1. `describe_entity('ComplianceRule')` — confirms the `CR_NLText` property holds the rule.
2. `run_gql` to retrieve `CR-IRA-30D` and **read its `CR_NLText`** verbatim. The agent extracts the FTA country list and failure conditions from the rule text — no hard-coded list anywhere.
3. `find_paths('ComplianceRule', 'ConnectedVehicle')` — discovers `GOVERNS_BATT_PASS → ← HAS_BATT_PASS` (2 hops).
4. `find_paths('ConnectedVehicle', 'DealerLocation')` — finds `IS_VARIANT → VehicleVariant ← CERTIFIED_FOR`.
5. `run_gql` joins the rule, the governed BatteryPassport, the VIN, and the eligible US dealers.
6. **Applies the rule in its reasoning step** — filters the result to VINs whose `BPass_OriginCty` is NOT in the FTA list it just read from `CR_NLText`.

### GQL

```gql
MATCH (rule:ComplianceRule {RuleId: 'CR-IRA-30D'})-[:GOVERNS_BATT_PASS]->(bp:BatteryPassport)<-[:HAS_BATT_PASS]-(cv:ConnectedVehicle)
FILTER cv.CV_RegMarket = 'USA'
MATCH (cv)-[:IS_VARIANT]->(v:VehicleVariant)<-[:CERTIFIED_FOR]-(d:DealerLocation)
FILTER d.Dealer_Country = 'USA' AND d.Dealer_EVCert = true
LET vin = cv.VIN
LET variant = v.VV_Name
LET mineral = bp.BPass_Mineral
LET mineralCountry = bp.BPass_OriginCty
LET ruleId = rule.RuleId
LET ruleAuthority = rule.CR_Authority
LET dealer = d.Dealer_Name
LET dealerCity = d.Dealer_City
RETURN DISTINCT vin, variant, mineral, mineralCountry, ruleId, ruleAuthority, dealer, dealerCity
ORDER BY vin, dealerCity
```

### Business answer

> *Applying `CR-IRA-30D` (US-IRS, effective 2024-01-01) to all US-registered EVs in the fleet:*
> *5 VINs (`1NDEQ52026A100001`, `…002`, `…003`, `…015`, plus the AMG variants from Detroit) **fail** the IRA §30D mineral-origin test — all carry NMC811 packs sourced from **DR Congo cobalt** (KasaiCobalt → VoltCell). DR Congo is NOT in the FTA list as written in `CR_NLText`. At $7,500 IRA credit per vehicle = **$37,500 in lost customer-incentive value already booked**. The next 200 NMC EVs in the production pipeline would lose **$1.5 M** if not re-sourced. Switching to KSpark LFP packs (Australian lithium — explicitly on the FTA list per `CR_NLText`) flips them all to compliant. Affected vehicles can be serviced at Detroit Flagship and Boston East (both EV-certified). **Rule cited:** `CR-IRA-30D`, US-IRS, effective 2024-01-01.*

## Second question — battery passport CO2 hotspots

> **"Calculate average lifecycle CO2 per VIN by mineral chemistry and country of origin. Where are our hotspots?"**

```gql
MATCH (cv:ConnectedVehicle)-[:HAS_BATT_PASS]->(bp:BatteryPassport)
LET mineral = bp.BPass_Mineral
LET originCountry = bp.BPass_OriginCty
LET avgCO2e = avg(bp.BPass_kgCO2e)
LET avgSoH = avg(bp.BPass_SoH)
LET vinCount = count(*)
RETURN mineral, originCountry, vinCount, avgCO2e, avgSoH
GROUP BY mineral, originCountry
ORDER BY avgCO2e DESC
```

### Insight

> *NMC811 from DR Congo: avg **3,840 kg CO2e** per pack across 8 VINs. LFP from Australia: avg **2,082 kg CO2e** per pack across 6 VINs. **45 % carbon reduction** by switching chemistry — and that's before mineral-mining-route improvements. For CSRD Scope 3 Cat 11 (use-phase) and Cat 12 (end-of-life), this is the single biggest material lever.*

## Third question — supplier ESG risk

> **"Which non-certified suppliers (Sup_Certified = false) touch components installed on US-registered vehicles, and what's the warranty cost on those vehicles?"**

```gql
MATCH (cv:ConnectedVehicle)<-[:INSTALLED_IN]-(a:Assembly)-[:USES_COMPONENT]->(c:Component)-[:SUPPLIED_BY]->(s:Supplier)
FILTER cv.CV_RegMarket = 'USA' AND s.Sup_Certified = false
MATCH (cv)-[:HAS_CLAIM]->(wc:WarrantyClaim)
LET supplier = s.Sup_Name
LET supplierCountry = s.Sup_Country
LET componentName = c.Comp_Name
LET claimCount = count(wc)
LET totalClaimCostUSD = sum(wc.WClaim_CostUSD)
RETURN supplier, supplierCountry, componentName, claimCount, totalClaimCostUSD
GROUP BY supplier, supplierCountry, componentName
ORDER BY totalClaimCostUSD DESC
```

> *KasaiCobalt Mining (DR Congo) is non-certified and supplies raw cobalt that becomes part of every NMC811 pack on every US EQ5. Combined warranty cost on those VINs: **see the full claim sum**. CSRD red flag — supplier ESG risk needs documented mitigation plan before next reporting cycle.*

## Why MCP changes this skill

The Sustainability team historically lives in spreadsheets pulled from PLM, supplier master, and battery-passport CSVs from contract manufacturers. With the MCP endpoint and this skill, they ask the regulatory question directly and get a unified answer that already carries the financial impact (IRA credit value, CSRD audit findings, warranty exposure) — making sustainability a first-class P&L conversation. **And because the rule lives in the `ComplianceRule` entity (not in GQL), Compliance owns the rule — not Engineering.** Edit `CR_NLText` to update the FTA list, the agent uses the new list on the next query, audit log shows which version applied when.
