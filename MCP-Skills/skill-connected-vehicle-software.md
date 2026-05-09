# Skill — Connected Vehicle & Software (SDV / Security)

## Description
The SDV / Security / OTA team's eye into the ontology. Find vehicles running vulnerable software, track patch coverage, identify VINs that cannot be reached by OTA, and triage the geographic distribution of unpatched units. **Use when** the security or SDV team asks *"who is exposed and where are they?"*

## When to use
- "CVE-2026-AUTO-0042 was just disclosed — who's exposed?"
- "What's our patch coverage on ADAS 3.4.2?"
- "Which OTA pushes are stuck (Pending or Failed)?"
- "How many connected vehicles are offline more than 72 hours?"
- "Which fault patterns in telemetry haven't had the matching TSB performed?"
- "Where are the unpatched vehicles geographically — and which dealers can reach them?"

## Hero question

> **"Critical CVE-2026-AUTO-0042 affects ADAS stack versions 3.3.5 and 3.4.1. Tell me: how many VINs are exposed, how many are patched to 3.4.2, how many OTA pushes are stuck, and where the unreachable ones are."**

### What the agent does

1. `describe_entity('SoftwareRelease')` — confirms the `SwRel_CVE` and `SwRel_Stack` properties.
2. `describe_entity('OTAUpdate')` — confirms `OTA_Status` (Success / Failed / Pending / Skipped) and `OTA_RetryCnt`.
3. `find_paths('SoftwareRelease', 'ConnectedVehicle')` — finds `← HAS_SW_RELEASE ← VehicleVariant ← IS_VARIANT ← ConnectedVehicle` (3 hops) and `← DEPLOYS_VERSION ← OTAUpdate ← RECEIVED_OTA ← ConnectedVehicle` (3 hops via OTA).
4. `run_gql` — joins both: every CV whose variant carries the vulnerable version, segmented by whether their most recent OTA pushed the patch.

### GQL

```gql
MATCH (cv:ConnectedVehicle)-[:IS_VARIANT]->(v:VehicleVariant)-[:HAS_SW_RELEASE]->(sw:SoftwareRelease)
FILTER sw.SwRel_Stack = 'ADAS' AND sw.SwRel_CVE = 'CVE-2026-AUTO-0042'
MATCH (cv)-[:RECEIVED_OTA]->(ota:OTAUpdate)-[:DEPLOYS_VERSION]->(patch:SoftwareRelease)
FILTER patch.SwRel_Stack = 'ADAS' AND patch.SwRel_Version = '3.4.2'
LET vin = cv.VIN
LET market = cv.CV_RegMarket
LET vulnerableVersion = sw.SwRel_Version
LET otaStatus = ota.OTA_Status
LET retries = ota.OTA_RetryCnt
LET online = cv.CV_OnlineFlag
RETURN vin, market, vulnerableVersion, otaStatus, retries, online
ORDER BY otaStatus, retries DESC
```

### Business answer the agent constructs

> *7 VINs were originally exposed to CVE-2026-AUTO-0042. **5 are patched** (OTA Success), **2 are stuck**:*
> - *VIN `1NDEQ52026A100003` (USA) — OTA-003 Failed after 2 retries. Online. Recommend dealer push at Boston East (DEAL-BOS-01, 5 open bays).*
> - *VIN `WDB22220262100006` (Germany) — OTA-017 Failed after 3 retries. Online. Recommend dealer push at Munich (DEAL-MUN-01, 4 open bays).*
> *No vehicles are offline. **Liability window is 7 days** (since CVE-2026-AUTO-0042 publication on 2025-09-12). Patching the 2 stuck VINs through dealer service closes the exposure.*

## Second question — fault pattern not yet remediated

> **"Show me connected vehicles that have emitted DTC `B1234` in their warranty claims but have NOT had TSB-2026-AB-009 performed via a repair ticket."**

### GQL

```gql
MATCH (cv:ConnectedVehicle)-[:HAS_CLAIM]->(wc:WarrantyClaim)
FILTER wc.WClaim_DTC = 'B1234'
MATCH (cv)-[:HAD_SERVICE]->(sv:ServiceVisit)-[:LEADS_TO_REPAIR]->(rt:RepairTicket)-[:FOLLOWS_TSB]->(t:TechServiceBul)
FILTER t.TSBId <> 'TSB-2026-AB-009'
LET vin = cv.VIN
LET market = cv.CV_RegMarket
LET dtc = wc.WClaim_DTC
LET claimCost = wc.WClaim_CostUSD
RETURN DISTINCT vin, market, dtc, claimCost
ORDER BY claimCost DESC
```

> Note: anti-join semantics ("does NOT have TSB-2026-AB-009") would normally need OPTIONAL MATCH. Because Fabric Graph does not support OPTIONAL MATCH, the agent splits the question into "has DTC B1234" + "list of TSB-2026-AB-009 repair targets" and computes the difference client-side. This is a documented MCP-skill pattern, not a graph-engine limitation people will trip over.

## Third question — fleet-wide patch readiness

> **"For each variant, what % of vehicles are running the latest patched ADAS version?"**

```gql
MATCH (cv:ConnectedVehicle)-[:IS_VARIANT]->(v:VehicleVariant)
MATCH (cv)-[:RECEIVED_OTA]->(ota:OTAUpdate)-[:DEPLOYS_VERSION]->(sw:SoftwareRelease)
FILTER sw.SwRel_Stack = 'ADAS'
LET variant = v.VV_Name
LET latestVer = sw.SwRel_Version
LET success = ota.OTA_Status
RETURN variant, latestVer, count(*) AS vinCount
GROUP BY variant, latestVer
ORDER BY variant, vinCount DESC
```

## Fourth question — rule-aware Critical-CVE 30-day SLA

> **"Apply the OTA security policy `CR-OTA-CVE` to our fleet. Which Critical CVEs are past the 30-day SLA, and which VINs are still unpatched?"**

### What the agent does

1. `run_gql` to fetch `CR-OTA-CVE` and **read its `CR_NLText`** — the rule states *"Critical CVE must be patched within 30 calendar days of SwRel_PublishDt; OTA Failed/Pending after 14 days = flag for dealer-assisted update."*
2. Traverse `ComplianceRule → GOVERNS_SW → SoftwareRelease` to get the affected releases.
3. Compute days-since-publish from `SwRel_PublishDt` to today.
4. Identify VINs whose latest OTA for that stack is not Success (or whose publish date is past the 30-day SLA).
5. Cite the rule in the answer.

### GQL

```gql
MATCH (rule:ComplianceRule {RuleId: 'CR-OTA-CVE'})-[:GOVERNS_SW]->(sw:SoftwareRelease)
FILTER sw.SwRel_Severity = 'Critical' AND sw.SwRel_CVE <> ''
MATCH (cv:ConnectedVehicle)-[:IS_VARIANT]->(v:VehicleVariant)-[:HAS_SW_RELEASE]->(sw)
MATCH (cv)-[:RECEIVED_OTA]->(ota:OTAUpdate)-[:DEPLOYS_VERSION]->(patch:SoftwareRelease)
FILTER patch.SwRel_Stack = sw.SwRel_Stack
LET vin = cv.VIN
LET market = cv.CV_RegMarket
LET cve = sw.SwRel_CVE
LET vulnVersion = sw.SwRel_Version
LET publishDt = sw.SwRel_PublishDt
LET otaStatus = ota.OTA_Status
LET retries = ota.OTA_RetryCnt
LET ruleCited = rule.RuleId
RETURN vin, market, cve, vulnVersion, publishDt, otaStatus, retries, ruleCited
ORDER BY publishDt, otaStatus
```

### Business answer

> *Per `CR-OTA-CVE` (NovaDrive-Internal, effective 2025-01-01, 30-day Critical CVE patch SLA):*
> - *CVE-2026-AUTO-0042 (ADAS) was published 2026-01-22. Today is 2026-05-08 — **the SLA window closed long ago**. 5 VINs are patched (Success). 2 VINs are non-compliant per the rule: `1NDEQ52026A100003` (USA, OTA Failed, 2 retries) and `WDB22220262100006` (Germany, OTA Failed, 3 retries). Per the rule's escalation clause, both should be flagged for dealer-assisted manual update immediately. **Rule cited:** `CR-OTA-CVE`, NovaDrive-Internal, effective 2025-01-01.*

## Why MCP changes this skill

The CISO does not write GQL. Today they file a request and wait days for the security data team to stitch together CVE → release → variant → VIN → OTA status. With the MCP endpoint and this skill, the CISO asks the question conversationally and gets back a *patch readiness dashboard* and a *dealer-routing remediation plan* in one turn.
