# Eventhouse Binding Guide — OEM Connected Vehicle Lifecycle

5 entities have timeseries data that must be bound from an Eventhouse / KQL database.

## Prerequisites

- Eventhouse named `OEMConnectedVehicle_Telemetry` with KQL database `OEMConnectedVehicleDB`.
- Static binding for each entity (from `lakehouse-binding.md`) **must already exist** before adding the timeseries binding — Fabric requires the static binding first to contextualize the timeseries by entity key.

## Step 1 — Ingest CSVs into Eventhouse Tables

For each Eventhouse CSV, run the following KQL command in the database:

### ConnectedVehicleTelemetry

```kql
.create table ConnectedVehicleTelemetry (
    VIN: string,
    Timestamp: datetime,
    CV_SoCPct: real,
    CV_BattTempC: real,
    CV_DTCCount: int,
    CV_SpeedKmh: real,
    CV_RangeKm: real,
    CV_OnlineFlag: int
)

.ingest into table ConnectedVehicleTelemetry ('https://<your-storage>/ConnectedVehicleTelemetry.csv')
    with (format='csv', ignoreFirstRecord=true)
```

### DriveSessionTelemetry

```kql
.create table DriveSessionTelemetry (
    SessionId: string,
    Timestamp: datetime,
    DS_AvgSpdKmh: real,
    DS_HrshBrkCnt: int,
    DS_MaxBattTempC: real,
    DS_EnergyKWh: real
)
```

### AssemblyTelemetry

```kql
.create table AssemblyTelemetry (
    AssemblyId: string,
    Timestamp: datetime,
    Asm_TempC: real,
    Asm_TorqueNm: real,
    Asm_CycleSec: real
)
```

### FacilityTelemetry

```kql
.create table FacilityTelemetry (
    FacilityId: string,
    Timestamp: datetime,
    Fac_EnergyKWh: real,
    Fac_HumidPct: real,
    Fac_ProdRateHr: real
)
```

### DealerCapacityTelemetry

```kql
.create table DealerCapacityTelemetry (
    DealerId: string,
    Timestamp: datetime,
    Dealer_OpenBays: int,
    Dealer_KitsInStk: int
)
```

## Step 2 — Add Timeseries Binding per Entity

### ConnectedVehicle (6 timeseries properties)

| Setting | Value |
|---|---|
| Source | Eventhouse `OEMConnectedVehicle_Telemetry` / database `OEMConnectedVehicleDB` |
| Source table | `ConnectedVehicleTelemetry` |
| Entity key column | `VIN` |
| Timestamp column | `Timestamp` |

| Property | Source Column | Type |
|---|---|---|
| CV_SoCPct | CV_SoCPct | double |
| CV_BattTempC | CV_BattTempC | double |
| CV_DTCCount | CV_DTCCount | int |
| CV_SpeedKmh | CV_SpeedKmh | double |
| CV_RangeKm | CV_RangeKm | double |
| CV_OnlineFlag | CV_OnlineFlag | int |

### DriveSession (4 timeseries properties)

| Setting | Value |
|---|---|
| Source table | `DriveSessionTelemetry` |
| Entity key column | `SessionId` |
| Timestamp column | `Timestamp` |

| Property | Source Column | Type |
|---|---|---|
| DS_AvgSpdKmh | DS_AvgSpdKmh | double |
| DS_HrshBrkCnt | DS_HrshBrkCnt | int |
| DS_MaxBattTempC | DS_MaxBattTempC | double |
| DS_EnergyKWh | DS_EnergyKWh | double |

### Assembly (3 timeseries properties)

| Setting | Value |
|---|---|
| Source table | `AssemblyTelemetry` |
| Entity key column | `AssemblyId` |
| Timestamp column | `Timestamp` |

| Property | Source Column | Type |
|---|---|---|
| Asm_TempC | Asm_TempC | double |
| Asm_TorqueNm | Asm_TorqueNm | double |
| Asm_CycleSec | Asm_CycleSec | double |

### Facility (3 timeseries properties)

| Setting | Value |
|---|---|
| Source table | `FacilityTelemetry` |
| Entity key column | `FacilityId` |
| Timestamp column | `Timestamp` |

| Property | Source Column | Type |
|---|---|---|
| Fac_EnergyKWh | Fac_EnergyKWh | double |
| Fac_HumidPct | Fac_HumidPct | double |
| Fac_ProdRateHr | Fac_ProdRateHr | double |

### DealerLocation (2 timeseries properties)

| Setting | Value |
|---|---|
| Source table | `DealerCapacityTelemetry` |
| Entity key column | `DealerId` |
| Timestamp column | `Timestamp` |

| Property | Source Column | Type |
|---|---|---|
| Dealer_OpenBays | Dealer_OpenBays | int |
| Dealer_KitsInStk | Dealer_KitsInStk | int |

## Step 3 — Refresh

After every binding is added, click **Refresh Graph**. The graph will now contain both static and timeseries data, queryable via GQL. Telemetry properties become accessible directly on the entity (e.g. `cv.CV_SoCPct`).

## Notes

- All timestamps fall within **2026-05-01 to 2026-05-08** — within the 7-day recency window.
- Timeseries data sits alongside, not inside, the static binding — the static binding stays the system of record for entity identity.
- All timeseries properties are annotated `(timeseries)` in the TTL ontology so the parser routes them to Eventhouse, not Lakehouse.
