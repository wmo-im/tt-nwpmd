# Client-Side Filtering for NWP Datasets in WIS2

**Cookbook section: Advanced subscription workflows**

---

## 1. Architecture and data flow

```
┌──────────────────────────────────────────────────────────────┐
│                   WCMP2 Record (GDC)                         │
│  links[0].channel  = "origin/.../deterministic/global"       │
│  links[0].filters  = {                                       │
│    parameter_controlled_name: { enum: [...] }                │
│    level_type:                { enum: [...] }                │
│    level_value:               { pattern: ..., enum: [...] }  │
│    statistic:                 { enum: [...] }                │
|    time_statistic:            { enum: [...] }                │
│  }                                                           │
└─────────────────────────────┬────────────────────────────────┘
                              │  publisher reads filter schema
                              ▼
┌──────────────────────────────────────────────────────────────┐
│          Publisher (e.g. ECMWF NWP dissemination)            │
│  Per granule, emits WNM with matching properties:            │
│  {                                                           │
│    "parameter_controlled_name": "air_temperature",           │
│    "parameter_local_name":      "t",                         │
│    "level_type":                "isobaric",                  │
│    "level_value":               "500hPa",                    │
│    "statistic":                 "deterministic",             │
|    "time_statistic":            "instantaneous"              |
│  }                                                           │
└─────────────────────────────┬────────────────────────────────┘
                              │  MQTT publish → Global Broker
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                  WIS2 Global Broker (MQTT)                   │
│  Topic: origin/a/wis2/int-ecmwf/.../deterministic/global    │
└────────────┬─────────────────────────────────────────────────┘
             │  subscribe to topic
             ▼
┌────────────────────────────────────────────────────────────┐
│              Client / Subscriber                           │
│  Filter predicate applied to each incoming WNM:           │
│    properties.parameter_controlled_name == "air_temperature│
│    AND properties.level_type            == "isobaric"      │
│    AND properties.level_value           == "500hPa"        │
│    AND properties.statistic             == "deterministic" │
│                                                            │
│  → Only download matching data files                       │
└────────────────────────────────────────────────────────────┘
```

**Key invariant**: the `id` values of concepts listed in `properties.themes[].concepts` are **identical** to the `enum` values in `links[].filters`. The themes block makes the vocabulary discoverable; the filters block makes the values machine-actionable.

---

## 3. The five filter dimensions

### 3.1 Parameter (`parameter_controlled_name` / `parameter_local_name`)

Two parallel properties identify the meteorological quantity:

| Property | Vocabulary source | Example values |
|---|---|---|
| `parameter_controlled_name` | WMO controlled (TT-NWPMD) at ??? | `air_temperature`, `total_precipitation`, `wind_speed` |
| `parameter_local_name` | Centre-specific vocabulary (e.g. ECMWF GRIB shortName) at `codes.ecmwf.int/grib/param-db` | `t`, `tp`, `10si` |

**Design rationale**: A subscriber may know only the WMO controlled name (portable across all NWP centres) or only the local name (when working with a specific centre's data). Providing both allows either workflow. The `parameter_controlled_name` is mandatory for all WIPPS-DC publishers; `parameter_local_name` is centre-specific and optional, but strongly recommended.

**Mandatory WMO-485 App 2.2.1 / 2.2.5 parameters** that all global NWP publishers must cover:

| WMO controlled name | ECMWF local name | Notes |
|---|---|---|
| `geopotential_height` | `z` | 850/500/250/200 hPa |
| `air_temperature` | `t` | 850/500/250/200 hPa + 2m |
| `eastward_wind` | `u` | 925/850/700/500/250/200 hPa |
| `northward_wind` | `v` | as above |
| `wind_speed` | `10si` | 10m surface |
| `wind_speed_of_gust` | `10fg` | 10m, max in period |
| `relative_humidity` | `r` | 850/700/500/200 hPa |
| `mean_sea_level_pressure` | `msl` | surface |
| `dew_point_temperature` | `2d` | 2m surface |
| `total_precipitation` | `tp` | surface accumulation |
| `total_solid_precipitation` | `sf` | surface accumulation |
| `convective_available_potential_energy` | `cape` | surface |
| `total_column_water_vapour` | `tcwv` | surface integral |
| `total_cloud_cover` | `tcc` | surface |

### 3.2 Level type (`level_type`)

Specifies the vertical coordinate of the field. Vocabulary published at ???.

| Value | Meaning | Typical fields |
|---|---|---|
| `isobaric` | Constant pressure level | T, Z, U, V, Q, R at pressure levels |
| `surface` | Earth surface or single level | MSLP, TP, TCC, CAPE, SP |
| `height_above_ground` | Fixed height above terrain | 2m temperature, 10m wind, 10m gust |
| `atmosphere_layer` | Layer between two pressure levels | Wind shear magnitude |

Surface-level fields (`level_type = surface`) do **not** carry a `level_value` property in the WNM.

### 3.3 Level value (`level_value`)

The numeric level value with its unit suffix. Rules:

- **Isobaric**: integer hPa value followed by `hPa` — e.g. `500hPa`, `850hPa`
- **Height above ground**: integer metre value followed by `m` — e.g. `2m`, `10m`
- **Atmosphere layer**: bounding pressure levels separated by `-`, top first — e.g. `250-850hPa`, `700-925hPa`
- **Surface**: omit `level_value` entirely

The enforced pattern is `^[0-9]+(-[0-9]+)?(hPa|m)$`. This prevents the free-text divergence that would otherwise occur between publishers (e.g. `500 hPa` vs `500hpa` vs `p500`).

**Recommended level values per WMO-485**:

| Level | Applicable parameters |
|---|---|
| `1000hPa`, `925hPa`, `850hPa`, `700hPa`, `500hPa`, `250hPa`, `200hPa` | T, Z, U, V, Q, R |
| `2m` | 2m temperature, 2m dewpoint |
| `10m` | 10m wind speed, 10m gust, 10m U/V |
| `250-850hPa` | Wind shear magnitude (EPS) |
| `700-925hPa` | Wind shear magnitude (EPS) |

### 3.4 Statistic (`statistic`)

Describes the **ensemble or probabilistic treatment** applied to produce the grid value. Vocabulary published at `codes.wmo.int/nwp/statistic`.

| Value | Applicable to | Companion filters required |
|---|---|---|
| `deterministic` | Single-run NWP (e.g. IFS-OPER) | none |
| `ensemble_mean` | EPS | none |
| `ensemble_spread` | EPS (standard deviation) | none |
| `probability` | EPS | `threshold_value` + `threshold_direction` |
| `percentile` | EPS | `percentile_value` |

### 3.5 Time statistic (`time_statistic`)

Describes how the field value was **computed over its validity period** (temporal processing). This dimension is entirely independent of the ensemble/probabilistic `statistic` dimension. Vocabulary published at ???.

| Value | Meaning | Example fields |
|---|---|---|
| `instantaneous` | Point-in-time value — no temporal aggregation | T, Z, U, V, RH, Q at pressure levels; MSLP, CAPE, TCWV, TCC |
| `time_minimum` | Minimum over the forecast period (e.g. 3h or 6h window) | Minimum 2m temperature (`2mn`) |
| `time_maximum` | Maximum over the forecast period | Maximum 2m temperature (`2mx`), 10m wind gust (`10fg`) |
| `time_mean` | Arithmetic mean over the forecast period | Some radiation or flux products |
| `time_accumulation` | Accumulated value over the period | Total precipitation (`tp`), snowfall (`sf`), convective precipitation (`cp`) |
| `time_standard_deviation` | Standard deviation over the period | Variability diagnostics |

**Rationale**: without this dimension a subscriber cannot distinguish the instantaneous 2m temperature from the 3h minimum or maximum 2m temperature using the same `air_temperature` controlled name. For example:

```json
{ "parameter_controlled_name": "air_temperature", "level_type": "height_above_ground",
  "level_value": "2m", "statistic": "deterministic", "time_statistic": "instantaneous" }

{ "parameter_controlled_name": "air_temperature", "level_type": "height_above_ground",
  "level_value": "2m", "statistic": "deterministic", "time_statistic": "time_minimum" }

{ "parameter_controlled_name": "air_temperature", "level_type": "height_above_ground",
  "level_value": "2m", "statistic": "deterministic", "time_statistic": "time_maximum" }
```

### 3.5.1 Probability companion filters

When `statistic = "probability"`, two additional properties must be present in the WNM:

| Property | Type | Values | Example |
|---|---|---|---|
| `threshold_value` | string | `^[0-9]+(\.[0-9]+)?(mm\|m/s)$` | `"10mm"`, `"15m/s"` |
| `threshold_direction` | string | `"above"` or `"below"` | `"above"` |

These encode the condition: "probability that the field is **above** (or below) the given threshold". WMO-485 App 2.2.5 mandates the following probability thresholds for precipitation and wind:

- Total precipitation (6h / 24h accumulation): 1, 5, 10, 25, 50, 100 mm
- 10m wind speed: 10, 15, 20, 25 m/s
- 10m wind gust: 15, 25, 35 m/s

### 3.5.2 Percentile companion filter

When `statistic = "percentile"`, one additional property must be present:

| Property | Type | Range | Conventions |
|---|---|---|---|
| `percentile_value` | integer | 0–100 | 0 = ensemble minimum; 100 = ensemble maximum |

WMO-485 App 2.2.5 mandates publication of the 25th, 50th, 75th percentiles and the maximum (100) for most EPS parameters. Publication of the minimum (0) is required for temperature.

---

## 4. Publisher implementation guide

### 4.1 What the publisher must add to each WNM

The `properties` object of every WNM must include the filter keys declared in the WCMP2 record's `filters` block. For a deterministic NWP dataset:

```json
{
  "id": "...",
  "type": "Feature",
  "conformsTo": ["http://wis.wmo.int/spec/wnm/1/conf/core"],
  "properties": {
    "pubtime": "2026-05-14T00:15:00Z",
    "data_id": "20260514/00z/0p25/oper/20260514000000-240h-oper-fc.grib2",
    "metadata_id": "urn:wmo:md:int-ecmwf:ifs:oper:forecast",
    "datetime": "2026-05-24T00:00:00Z",
    "parameter_controlled_name": "air_temperature",
    "parameter_local_name": "t",
    "level_type": "isobaric",
    "level_value": "500hPa",
    "statistic": "deterministic",
    "time_statistic": "instantaneous"
  },
  "geometry": { "type": "Polygon", "coordinates": [[[-180,-90],[-180,90],[180,90],[180,-90],[-180,-90]]] },
  "links": [{
    "href": "https://data.ecmwf.int/forecasts/20260514/00z/0p25/oper/20260514000000-240h-oper-fc.grib2",
    "rel": "canonical",
    "type": "application/x-grib2"
  }]
}
```

For an EPS probability product:

```json
{
  "properties": {
    "pubtime": "2026-05-14T00:20:00Z",
    "data_id": "20260514/00z/0p25/enfo/20260514000000-240h-enfo-prob-tp-10mm.grib2",
    "metadata_id": "urn:wmo:md:int-ecmwf:ifs:enfo:forecast",
    "datetime": "2026-05-24T00:00:00Z",
    "parameter_controlled_name": "total_precipitation",
    "parameter_local_name": "tp",
    "level_type": "surface",
    "statistic": "probability",
    "time_statistic": "time_accumulation",
    "threshold_value": "10mm",
    "threshold_direction": "above"
  }
}
```

For an EPS percentile product:

```json
{
  "properties": {
    "pubtime": "2026-05-14T00:20:00Z",
    "data_id": "20260514/00z/0p25/enfo/20260514000000-240h-enfo-pct50-t-850hPa.grib2",
    "metadata_id": "urn:wmo:md:int-ecmwf:ifs:enfo:forecast",
    "datetime": "2026-05-24T00:00:00Z",
    "parameter_controlled_name": "air_temperature",
    "parameter_local_name": "t",
    "level_type": "isobaric",
    "level_value": "850hPa",
    "statistic": "percentile",
    "time_statistic": "instantaneous",
    "percentile_value": 50
  }
}
```

### 4.2 Multi-parameter granules

If the publisher produces one GRIB file containing **multiple parameters**, the WNM cannot carry a single `parameter_controlled_name` string. Use a JSON array instead and update the WCMP2 filter schema `type` from `"string"` to `"array"` with `"items": {"type": "string", "enum": [...]}`.

```json
"parameter_controlled_name": ["air_temperature", "geopotential_height", "relative_humidity"]
```

This is the appropriate approach for publishers following the traditional NWP practice of bundling multiple fields in a single GRIB file per forecast step. Client-side filtering then applies **array containment** semantics: a subscriber filtering on `air_temperature` receives the WNM if that name appears anywhere in the array.

---

## 5. Use cases

### Use case 1 — Deterministic NWP: upper-air temperature only

**Scenario**: An aviation weather centre needs only temperature at standard pressure levels from the ECMWF IFS operational forecast. They want to ignore all surface and wind fields.

**WCMP2 discovery**: The subscriber reads the `filters` block of `urn:wmo:md:int-ecmwf:ifs:oper:forecast` and notes the filter keys.

**Subscription predicate**:
```
parameter_controlled_name = "air_temperature"
AND level_type = "isobaric"
AND statistic = "deterministic"
```

**Example matching WNM properties**:
```json
{
  "parameter_controlled_name": "air_temperature",
  "parameter_local_name": "t",
  "level_type": "isobaric",
  "level_value": "850hPa",
  "statistic": "deterministic"
}
```

**Effect**: The subscriber receives WNMs for T at 1000, 925, 850, 700, 500, 400, 300, 250, 200, 150, 100 hPa and ignores all others. Data download volume is reduced by approximately 90% for this use case.

---

### Use case 2 — Deterministic NWP: surface parameters for impact forecasting

**Scenario**: A national hydrological service needs precipitation, 2m temperature, 10m wind, and MSLP. They do not need upper-air fields.

**Subscription predicate**:
```
level_type IN ["surface", "height_above_ground"]
AND statistic = "deterministic"
```

**Example matching WNM properties** (precipitation):
```json
{
  "parameter_controlled_name": "total_precipitation",
  "parameter_local_name": "tp",
  "level_type": "surface",
  "statistic": "deterministic"
}
```

**Example matching WNM properties** (2m temperature):
```json
{
  "parameter_controlled_name": "air_temperature",
  "parameter_local_name": "2t",
  "level_type": "height_above_ground",
  "level_value": "2m",
  "statistic": "deterministic"
}
```

**Note**: `level_type` is a better predicate than `parameter_controlled_name` here because new surface parameters added by the publisher will automatically be received without updating the subscription.

---

### Use case 3 — Deterministic NWP: single parameter and level

**Scenario**: A medium-range forecasting centre wants only 500 hPa geopotential height for synoptic chart production.

**Subscription predicate**:
```
parameter_controlled_name = "geopotential_height"
AND level_type = "isobaric"
AND level_value = "500hPa"
AND statistic = "deterministic"
```

**Example matching WNM properties**:
```json
{
  "parameter_controlled_name": "geopotential_height",
  "parameter_local_name": "z",
  "level_type": "isobaric",
  "level_value": "500hPa",
  "statistic": "deterministic"
}
```

This is the most selective predicate possible — one field from hundreds.

---

### Use case 4 — EPS: heavy precipitation probabilities

**Scenario**: A disaster risk reduction centre wants only the exceedance probability products for precipitation above WMO-485-mandated thresholds, for all forecast steps up to day 10.

**Subscription predicate**:
```
parameter_controlled_name = "total_precipitation"
AND statistic = "probability"
AND threshold_direction = "above"
AND threshold_value IN ["1mm", "10mm", "25mm", "50mm", "100mm"]
```

**Example matching WNM properties** (probability of ≥ 10mm/24h):
```json
{
  "parameter_controlled_name": "total_precipitation",
  "parameter_local_name": "tp",
  "level_type": "surface",
  "statistic": "probability",
  "threshold_value": "10mm",
  "threshold_direction": "above"
}
```

**Note**: The `level_value` is absent because precipitation accumulations are surface fields with no meaningful pressure or height coordinate.

---

### Use case 5 — EPS: probabilistic wind portfolio (mean, spread, probabilities, percentiles)

**Scenario**: An energy sector user needs a complete probabilistic picture of 10m wind speed: ensemble mean, spread, exceedance probabilities, and the 10th and 90th percentiles. They want no other parameters.

**Subscription predicate**:
```
parameter_controlled_name = "wind_speed"
AND level_type = "height_above_ground"
AND level_value = "10m"
```

By selecting only on `parameter_controlled_name` and level, this single subscription captures **all** statistical treatments of 10m wind speed:

**Matching WNMs** (four distinct types per forecast step):

*Ensemble mean*:
```json
{
  "parameter_controlled_name": "wind_speed",
  "parameter_local_name": "10si",
  "level_type": "height_above_ground",
  "level_value": "10m",
  "statistic": "ensemble_mean"
}
```

*Ensemble spread*:
```json
{
  "parameter_controlled_name": "wind_speed",
  "parameter_local_name": "10si",
  "level_type": "height_above_ground",
  "level_value": "10m",
  "statistic": "ensemble_spread"
}
```

*Exceedance probability at 15 m/s*:
```json
{
  "parameter_controlled_name": "wind_speed",
  "parameter_local_name": "10si",
  "level_type": "height_above_ground",
  "level_value": "10m",
  "statistic": "probability",
  "threshold_value": "15m/s",
  "threshold_direction": "above"
}
```

*10th percentile*:
```json
{
  "parameter_controlled_name": "wind_speed",
  "parameter_local_name": "10si",
  "level_type": "height_above_ground",
  "level_value": "10m",
  "statistic": "percentile",
  "percentile_value": 10
}
```

If the user wants to **exclude** the raw probability products (because they only want mean, spread, and percentiles), they add:
```
AND statistic != "probability"
```

---

### Use case 6 — EPS: wind shear for aviation turbulence alerts

**Scenario**: An RSMC providing aviation hazard forecasts needs wind shear magnitude between 250 hPa and 850 hPa.

**Subscription predicate**:
```
parameter_controlled_name = "wind_shear_magnitude"
AND level_type = "atmosphere_layer"
AND level_value = "250-850hPa"
```

**Example matching WNM properties** (50th percentile):
```json
{
  "parameter_controlled_name": "wind_shear_magnitude",
  "level_type": "atmosphere_layer",
  "level_value": "250-850hPa",
  "statistic": "percentile",
  "percentile_value": 50
}
```

**Note**: `parameter_local_name` may be absent for derived quantities not present as a native GRIB parameter in all systems. The absence of `parameter_local_name` does not invalidate the WNM; it simply means the local-name filter cannot be applied.

---

## 6. Vocabulary governance

Several controlled vocabulary endpoints need to be operational for the filter design to be fully normative.

Until the WMO vocabularies are published, publishers should use the placeholder scheme URLs as defined in these example records and document them in their national metadata practices. The concept `id` values themselves (the snake_case tokens) are stable and should not change when the URLs are finalised.
