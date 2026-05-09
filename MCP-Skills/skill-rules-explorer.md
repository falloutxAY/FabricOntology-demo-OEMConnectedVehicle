# Skill — Compliance Rules Explorer

## Description
Discover, read, and reason over the **`ComplianceRule`** entity — the natural-language regulatory and internal policies that the OEM operates under. Use this skill any time someone asks *"which rules apply to X?"*, *"what does the rule actually say?"*, *"who set the rule?"*, or *"would we pass an audit on Y today?"*.

This is the skill that makes the demo land in **regulated industries**. It is the answer to *"can your AI agent be trusted to follow the regulation, or did your developers hard-code their interpretation of it?"*

## When to use
- "What rules govern our EU battery passports?"
- "Show me the actual text of the IRA Section 30D rule"
- "Which rule limits Critical CVE patch time, and what is the SLA?"
- "What is our internal goodwill policy?"
- "Are there any Mandatory rules effective in the last 12 months that we are not currently checking?"
- "If we changed the FTA country list tomorrow, which VINs would flip from non-compliant to compliant?"

## What's in the rule library (8 seeded rules)

| RuleId | Domain | Authority | Severity | Title |
|---|---|---|---|---|
| `CR-IRA-30D` | IRA | US-IRS | Mandatory | IRA Section 30D Critical Mineral Origin |
| `CR-EU-BATT` | EU-Battery | EU-Commission | Mandatory | EU Battery Regulation 2023/1542 Passport |
| `CR-NHTSA-T1` | NHTSA | NHTSA | Mandatory | 5-Day Notification + 60-Day Owner Outreach |
| `CR-NHTSA-T2` | NHTSA | NHTSA | Mandatory | 18-Month 85% Completion Threshold |
| `CR-CSRD-S3` | CSRD | EU-Commission | Mandatory | EU CSRD Scope 3 Category 11 Disclosure |
| `CR-IW-EVPK` | Internal-Warranty | NovaDrive-Internal | Mandatory | EV Pack 8yr / 160k km Coverage |
| `CR-IW-GOOD` | Internal-Warranty | NovaDrive-Internal | Internal-Goodwill | Goodwill Boundary Policy |
| `CR-OTA-CVE` | OTA-Security | NovaDrive-Internal | Mandatory | Critical CVE 30-Day Patch SLA |

Each rule's `CR_NLText` is the **full plain-English statement** — country lists, time windows, percentage thresholds, severity logic, all in language Compliance owns and edits.

## Hero question

> **"Show me every Mandatory rule that governs our EU-registered EVs, what each one actually says, and which of our vehicles would currently fail any of them."**

### What the agent does

1. `describe_entity('ComplianceRule')` → confirms `RuleId`, `CR_Domain`, `CR_NLText`, `CR_Authority`, `CR_Severity`, `CR_EffectiveDt`.
2. `find_paths('ComplianceRule', 'ConnectedVehicle')` → discovers `GOVERNS_BATT_PASS → ← HAS_BATT_PASS` (2 hops to ConnectedVehicle via the BatteryPassport governance edge).
3. `run_gql` (one query for all rules governing EU-registered EV battery passports).
4. **Reads each rule's `CR_NLText` and applies it** — the FTA country list lives in the IRA rule text, the passport-completeness criteria live in the EU-Battery rule text, the kg CO2e disclosure logic lives in the CSRD rule text.
5. Returns a per-VIN compliance status with the rule cited.

### GQL

```gql
MATCH (rule:ComplianceRule)-[:GOVERNS_BATT_PASS]->(bp:BatteryPassport)<-[:HAS_BATT_PASS]-(cv:ConnectedVehicle)
FILTER cv.CV_RegMarket IN ['Germany','France','UK','Italy','Spain','Netherlands','Sweden']
   AND rule.CR_Severity = 'Mandatory'
LET vin = cv.VIN
LET market = cv.CV_RegMarket
LET ruleId = rule.RuleId
LET ruleTitle = rule.CR_Title
LET ruleAuthority = rule.CR_Authority
LET ruleEffective = rule.CR_EffectiveDt
LET mineral = bp.BPass_Mineral
LET originCountry = bp.BPass_OriginCty
LET soH = bp.BPass_SoH
LET kgCO2e = bp.BPass_kgCO2e
RETURN vin, market, ruleId, ruleTitle, ruleAuthority, ruleEffective, mineral, originCountry, soH, kgCO2e
ORDER BY vin, ruleId
```

### Business answer the agent constructs

> *Three Mandatory rules govern EU-registered EV battery passports:*
> - **CR-IRA-30D** — does NOT apply to EU registrations (IRS jurisdiction is US-only). The agent flags this and skips.
> - **CR-EU-BATT** — applies to all 4 EU-registered VINs (`WDB22220262100004`, `…005`, `…006`, `…014`). All four have a complete BatteryPassport with SoH, kgCO2e, mineral, origin country → **PASS**.
> - **CR-CSRD-S3** — applies to all 4. All four have BPass_kgCO2e populated (range 2,180-3,940 kg). The agent reads the CSRD rule text and confirms BatteryPassport CO2e data is sufficient for Scope 3 Cat 11 calculation → **PASS**, but flags that 2 of the 4 (NMC packs at ~3,900 kg) are 80% above the LFP packs (~2,200 kg), creating a hotspot worth disclosing in the narrative section of the CSRD report.
>
> *Net: 4 EU EVs, 3 applicable rules each, all currently compliant. Rule citations: `CR-EU-BATT` (EU-Commission, effective 2027-02-18), `CR-CSRD-S3` (EU-Commission, effective 2024-01-01).*

## Second question — rule transparency

> **"Show me the actual text of rule `CR-IRA-30D`. I want to read what we are reasoning over."**

### GQL

```gql
MATCH (r:ComplianceRule {RuleId: 'CR-IRA-30D'})
LET ruleId = r.RuleId
LET title = r.CR_Title
LET authority = r.CR_Authority
LET effective = r.CR_EffectiveDt
LET severity = r.CR_Severity
LET fullText = r.CR_NLText
RETURN ruleId, title, authority, effective, severity, fullText
```

### What the agent returns

The full `CR_NLText` verbatim — including the FTA country list, the failure conditions, and the explicit instruction to the agent on how to apply it. **No interpretation, no paraphrase, no developer-translated thresholds.** The Compliance officer can read exactly what the agent is reasoning from.

> Talking point: *"You can audit the rule. You can edit the rule. You don't have to file a ticket with the data team."*

## Third question — what-if simulation

> **"If we added Indonesia and DR Congo to the FTA list tomorrow, how many of our currently-non-compliant US EVs would flip to compliant?"**

The agent doesn't actually edit the rule — but it can **simulate** by re-reading `CR_NLText`, mentally extending the FTA list, and re-running the IRA query against the new hypothetical list. The result is a *what-if* number for the policy team, with zero code change.

```gql
MATCH (r:ComplianceRule {RuleId: 'CR-IRA-30D'})-[:GOVERNS_BATT_PASS]->(bp:BatteryPassport)<-[:HAS_BATT_PASS]-(cv:ConnectedVehicle)
FILTER cv.CV_RegMarket = 'USA'
LET vin = cv.VIN
LET originCountry = bp.BPass_OriginCty
LET mineral = bp.BPass_Mineral
RETURN vin, originCountry, mineral
ORDER BY originCountry
```

> *5 US EVs sourced from DR Congo. If DR Congo were added to the FTA list tomorrow, all 5 would flip from non-compliant to compliant. Estimated recovered IRA credit value: $37,500. (Indonesia: 0 vehicles in current dataset.)*

## Why MCP changes this skill

Today, *"what rule are we using and what does it say?"* is a Slack thread between Compliance, Legal, and Engineering — and the answer is buried in a SQL view nobody can read without a developer. With the MCP endpoint and this skill, the rule **is** the answer, in plain English, with a graph edge linking it directly to every entity it governs. **The agent doesn't interpret the regulation; it reads it.** That's the difference between a clever demo and a regulated-industry-grade product.
