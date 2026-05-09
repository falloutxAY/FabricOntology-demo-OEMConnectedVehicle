# Lakehouse Binding Guide — OEM Connected Vehicle Lifecycle

## Prerequisites

- Fabric capacity F2+ (or P1 Power BI Premium with Fabric enabled).
- All 6 tenant settings enabled (Ontology, Graph, Data Agent, Copilot Azure OpenAI, cross-region processing, cross-region storage).
- Lakehouse with **OneLake security DISABLED** and **schemas DISABLED**.
- All 55 CSV files (22 Dim + 33 Edge) uploaded to the Lakehouse and converted to Delta tables.

## Step 1 — Upload CSVs to Lakehouse

1. Open your Fabric workspace.
2. Create a Lakehouse named `OEMConnectedVehicle_Lakehouse`.
3. Upload all CSVs from `Data/Lakehouse/` to the Lakehouse `Files/Data/Lakehouse/` area.
4. For each CSV, right-click → **Load to Tables** → use defaults (header=true, infer schema).
5. Verify all 49 tables are present under `Tables/`.

## Step 2 — Bind Static Entities (22 entities)

For each entity in `bindings.yaml` `lakehouse.entities`:

1. Open the Ontology item in Fabric → **Add Static Binding**.
2. Select the source Lakehouse and Dim* table.
3. Set the key column (matches the entity's `keyColumn` in `bindings.yaml`).
4. Map each property → CSV column → declared type (string, int, double, boolean, datetime).
5. Confirm the key property is the **first row** of the property mapping table.

> [!IMPORTANT]
> Key column MUST be in the properties array. Skip this and the API returns "Missing mapping for key property".

> [!NOTE]
> Timeseries-bearing entities (ConnectedVehicle, DriveSession, Assembly, Facility, DealerLocation) get their **static** binding here from the Dim* table. Their **timeseries** properties bind separately via the Eventhouse guide — do NOT include them in the static binding.

## Step 3 — Bind 33 Relationships (Edge tables)

For each relationship in `bindings.yaml` `lakehouse.relationships`:

1. Add **Relationship Binding** in the Ontology UI.
2. Select source entity and target entity.
3. Source table = the dedicated `Edge*` table (NEVER a Dim* table — see warning below).
4. `sourceKeyColumn` = exact name of source entity's key property.
5. `targetKeyColumn` = exact name of target entity's key property.

> [!WARNING]
> All 33 relationships use **dedicated Edge tables**. Contextual bindings (using a Dim* as sourceTable) are fundamentally broken because the entity-prefixed FK property name (e.g. `CV_VariantId`) can never equal the target entity's key property name (`VariantId`). Edge tables solve this with two columns named exactly as the source/target entity keys.

### Relationship → Edge table cheat sheet

| Relationship | Edge Table | sourceKey → targetKey |
|---|---|---|
| IS_VARIANT | EdgeCV_Variant | VIN → VariantId |
| HAS_SW_RELEASE | EdgeVariant_SwRelease | VariantId → SwReleaseId |
| BUILT_FROM_PO | EdgeCV_PO | VIN → ProdOrderId |
| HAS_BATT_PASS | EdgeCV_BattPass | VIN → BattPassId |
| HAD_SESSION | EdgeCV_Session | VIN → SessionId |
| RECEIVED_OTA | EdgeCV_OTA | VIN → OTAUpdateId |
| HAS_CLAIM | EdgeCV_Claim | VIN → ClaimId |
| HAD_SERVICE | EdgeCV_Service | VIN → VisitId |
| BUILT_AT | EdgePO_Facility | ProdOrderId → FacilityId |
| INSTALLED_IN | EdgeAsm_CV | AssemblyId → VIN |
| USES_COMPONENT | EdgeAsm_Comp | AssemblyId → ComponentId |
| AFFECTS_ASM | EdgeQE_Asm | EventId → AssemblyId |
| DEPLOYS_VERSION | EdgeOTA_SwRelease | OTAUpdateId → SwReleaseId |
| CUST_OWNS | EdgeCustomer_Owner | CustomerId → OwnerId |
| OWNS_VEHICLE | EdgeOwner_CV | OwnerId → VIN |
| AT_DEALER | EdgeService_Dealer | VisitId → DealerId |
| LEADS_TO_REPAIR | EdgeService_Repair | VisitId → TicketId |
| FOLLOWS_TSB | EdgeRepair_TSB | TicketId → TSBId |
| COVERED_BY | EdgeClaim_Policy | ClaimId → PolicyId |
| RESOLVED_BY | EdgeClaim_Repair | ClaimId → TicketId |
| CAUSED_BY_QE | EdgeRecall_QE | RecallId → EventId |
| AFFECTS_VARIANT | EdgeRecall_Variant | RecallId → VariantId |
| AFFECTS_VEHICLE | EdgeRecall_CV | RecallId → VIN |
| REQUIRES_TSB | EdgeRecall_TSB | RecallId → TSBId |
| ISSUED_FOR | EdgeBPass_CV | BattPassId → VIN |
| TRACES_COMP | EdgeBPass_Comp | BattPassId → ComponentId |
| SUPPLIED_BY | EdgeComp_Supplier | ComponentId → SupplierId |
| OPERATES_AT | EdgeSupplier_Facility | SupplierId → FacilityId |
| CERTIFIED_FOR | EdgeDealer_Variant | DealerId → VariantId |
| GOVERNS_BATT_PASS | EdgeRule_BattPass | RuleId → BattPassId |
| GOVERNS_POLICY | EdgeRule_Policy | RuleId → PolicyId |
| GOVERNS_RECALL | EdgeRule_Recall | RuleId → RecallId |
| GOVERNS_SW | EdgeRule_SwRelease | RuleId → SwReleaseId |

## Step 4 — Refresh Graph

After all bindings are complete, click **Refresh Graph** on the Ontology item. Graph refresh is manual and full each time.

## Limitations to Communicate to the Audience

- One static binding per entity (eventhouse adds timeseries on top — that's allowed).
- No Decimal type — all monetary fields use Double (`WClaim_CostUSD`, `Comp_UnitCost`, `Repair_PartsUSD`).
- Property names ≤ 26 chars and globally unique across all 21 entity types.
- `OPTIONAL MATCH` not supported in Fabric Graph — design queries to require all relationships.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Missing mapping for key property` | Key column not in `properties[]` array | Add key as first property in the entity's properties list |
| `targetKeyRefBindings targetPropertyId 'X' must be present in target EntityType's EntityIdParts` | Edge column name doesn't match target entity key | Rename the column in the Edge CSV to exactly the target key property name |
| `Decimal returns NULL` | TTL or binding declares `decimal` | Use `double` instead — already corrected in this demo |
| Empty graph after refresh | Missing key values / orphan FKs | Check no NULLs in key columns and all Edge FKs reference valid Dim records |

> [!NOTE]
> Timeseries properties (CV_SoCPct, CV_BattTempC, CV_DTCCount, CV_SpeedKmh, CV_RangeKm, CV_OnlineFlag, DS_AvgSpdKmh, DS_HrshBrkCnt, DS_MaxBattTempC, DS_EnergyKWh, Asm_TempC, Asm_TorqueNm, Asm_CycleSec, Fac_EnergyKWh, Fac_HumidPct, Fac_ProdRateHr, Dealer_OpenBays, Dealer_KitsInStk) are bound separately via Eventhouse — see `eventhouse-binding.md`.
