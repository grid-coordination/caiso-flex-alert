# CAISO Flex Alert API Documentation

## Overview

The California Independent System Operator (CAISO) issues **Flex Alerts** — voluntary conservation calls asking electricity consumers to reduce usage during periods when the grid is under stress (typically hot summer afternoons/evenings when demand peaks and supply is tight).

CAISO provides machine-readable endpoints to check the current Flex Alert status. This document describes the known APIs, their response formats, and how to parse them.

## Endpoints

### Active Endpoint: Today's Outlook API (v2)

```
GET https://wwwmobile.caiso.com/TodaysOutlookApi/api/EmergencyNotices/v2/ActiveFlexAlertEvents
```

- **Authentication:** None (public access)
- **Default response format:** JSON (`application/json; charset=utf-8`)
- **XML response:** Send `Accept: application/xml` header
- **Update interval:** ~1 minute
- **Data availability:** Always returns a response, but `Active` and `Future` arrays are empty when no alerts are active or scheduled

This is part of the *Today's Outlook* API suite that powers CAISO's public grid dashboard and mobile interfaces.

Try it:

```bash
curl -s 'https://wwwmobile.caiso.com/TodaysOutlookApi/api/EmergencyNotices/v2/ActiveFlexAlertEvents' | python3 -m json.tool
```

## Response Formats

### JSON (default)

```json
{
  "IsFlexAlertActive": true,
  "UpdatedOn": "06-11-2021 09:05:29",
  "Active": [
    {
      "ID": 6,
      "Title": "Flex alert for 06/11/2021",
      "StartDate": "06-11-2021 09:00",
      "EndDate": "06-11-2021 21:00",
      "Region": "Northern California;VEA Region"
    }
  ],
  "Future": [
    {
      "ID": 1,
      "Title": "Flex alert for 06/14/2021",
      "StartDate": "06-14-2021 16:00",
      "EndDate": "06-14-2021 21:00",
      "Region": "Statewide"
    }
  ]
}
```

When no alert is active:

```json
{
  "IsFlexAlertActive": false,
  "UpdatedOn": "04-21-2026 15:44:06",
  "Active": [],
  "Future": []
}
```

### XML (with `Accept: application/xml`)

```xml
<ActiveFlexAlertEventsDTO
    xmlns:i="http://www.w3.org/2001/XMLSchema-instance"
    xmlns="http://schemas.datacontract.org/2004/07/Caiso.TodaysOutlook.Common.DTO">
  <Active />
  <Future />
  <IsFlexAlertActive>false</IsFlexAlertActive>
  <UpdatedOn>04-21-2026 15:44:23</UpdatedOn>
</ActiveFlexAlertEventsDTO>
```

The XML namespace reveals the internal DTO class: `Caiso.TodaysOutlook.Common.DTO.ActiveFlexAlertEventsDTO` — this is a .NET WCF/Web API data contract.

## Field Reference

### Top-Level Fields

| Field | Type | Description |
|---|---|---|
| `IsFlexAlertActive` | boolean | `true` only when there is at least one **Active** alert. `false` even if Future alerts exist. |
| `UpdatedOn` | string | Timestamp of last update, format: `MM-DD-YYYY HH:MM:SS` |
| `Active` | array | Currently active Flex Alert events |
| `Future` | array | Upcoming/scheduled Flex Alert events (not yet started) |

### Event Fields (objects in `Active` and `Future` arrays)

| Field | Type | Description |
|---|---|---|
| `ID` | integer | Unique event identifier (stable across the lifetime of an alert) |
| `Title` | string | Human-readable title, e.g. `"Flex alert for 06/11/2021"` |
| `StartDate` | string | Event start time, format: `MM-DD-YYYY HH:MM` |
| `EndDate` | string | Event end time, format: `MM-DD-YYYY HH:MM` |
| `Region` | string | Semicolon-delimited list of affected regions |

## Parsing Guide

### Timestamps

The timestamp format has changed over time. The original API (pre-May 2024) used a custom format with implicit US/Pacific timezone. After the May 2024 website relaunch, `UpdatedOn` switched to ISO 8601 with a `Z` suffix (UTC), though `StartDate`/`EndDate` still used the legacy format in available samples.

**Legacy format** (pre-May 2024, and still seen in `StartDate`/`EndDate`):

| Context | Format | Example | Timezone |
|---|---|---|---|
| `UpdatedOn` | `MM-DD-YYYY HH:MM:SS` | `06-11-2021 09:05:29` | US/Pacific (implicit) |
| `StartDate` / `EndDate` | `MM-DD-YYYY HH:MM` | `06-11-2021 09:00` | US/Pacific (implicit) |

**Post-May 2024 format** (observed in `UpdatedOn`):

| Context | Format | Example | Timezone |
|---|---|---|---|
| `UpdatedOn` | ISO 8601 | `2024-06-17T15:19:40Z` | UTC (explicit) |

Clients should handle both formats defensively.

Python parsing example:

```python
from datetime import datetime
from zoneinfo import ZoneInfo

def parse_timestamp(ts: str) -> datetime:
    """Parse a CAISO timestamp, handling both legacy and ISO 8601 formats."""
    if 'T' in ts:
        # ISO 8601 format (post-May 2024)
        return datetime.fromisoformat(ts.replace('Z', '+00:00'))
    elif ts.count(':') == 2:
        # Legacy UpdatedOn format: MM-DD-YYYY HH:MM:SS
        return datetime.strptime(ts, '%m-%d-%Y %H:%M:%S').replace(
            tzinfo=ZoneInfo('US/Pacific')
        )
    else:
        # Legacy StartDate/EndDate format: MM-DD-YYYY HH:MM
        return datetime.strptime(ts, '%m-%d-%Y %H:%M').replace(
            tzinfo=ZoneInfo('US/Pacific')
        )
```

### Regions

The `Region` field is a delimited string of region names. The delimiter has been semicolon (`;`) in some samples and comma (`,`) in others — clients should handle both.

Region names have changed across API versions. Known values:

| Current Name (post-May 2024) | Legacy Name (pre-May 2024) | Description |
|---|---|---|
| `CAISO Grid` | `Statewide` | System-wide alert. Treat as applying to every region. |
| `Northern CA Region` | `Northern California` | Roughly aligns with PG&E service area |
| `Southern CA Region` | `Southern California` | Roughly aligns with SCE + SDG&E service areas |
| `VEA Region` | `VEA Region` | Valley Electric Association, Pahrump, NV. Part of CAISO balancing area, but a Flex Alert has reportedly never been called specifically for VEA alone. |
| `South OC San Diego Region` | — | Created for specific historical issues; CAISO has discussed retiring this region. |

**CAISO does not formally define the geographic boundaries of these regions.** Per CAISO staff (2023-2024): the regions roughly correspond to investor-owned utility (IOU) service territories, but CAISO has no zip-code-to-region mapping and does not target communications by zip code. Not all customers within these areas necessarily fall under CAISO's balancing area.

As a rough geographic proxy, CAISO staff suggested the California IOU service area maps from the California Energy Commission's GIS data.

To parse into a list:

```python
import re

def parse_regions(region_str: str) -> list[str]:
    """Split Region field on semicolons or commas."""
    return [r.strip() for r in re.split(r'[;,]', region_str) if r.strip()]
```

To check whether an event applies to a given region:

```python
def event_applies_to(event: dict, region: str) -> bool:
    regions = parse_regions(event['Region'])
    statewide = {'Statewide', 'CAISO Grid'}
    return bool(statewide & set(regions)) or region in regions
```

### Determining Alert Status

1. **Quick check:** `IsFlexAlertActive` is `true` only when there is at least one alert in the `Active` array. It is `false` even if `Future` contains scheduled alerts.
2. **Currently active alerts:** Items in the `Active` array (currently in effect as of `UpdatedOn`)
3. **Upcoming alerts:** Items in the `Future` array (scheduled but not yet started)
4. **Region filtering:** An event applies to a region if it is `Statewide`/`CAISO Grid` or the region appears in the parsed `Region` field
5. **Event identity:** The `ID` of a Flex Alert remains constant for its entire lifecycle

## Related Data Sources

- **CAISO Developer Portal** (`developer.caiso.com`): Requires registration with a company email. Provides access to OASIS and other CAISO data APIs, but the Flex Alert data comes from the separate Today's Outlook API endpoint documented above (no developer account required).
- **CEC MIDAS**: The California Energy Commission rebroadcasts CAISO Flex Alerts on their MIDAS server. The CEC does not issue its own alerts. Historical Flex Alert data has not been successfully retrieved from MIDAS.
- **Grid Emergencies History** (PDF): CAISO publishes a PDF listing historical grid emergencies from 1998 to present, but it contains no machine-readable event data.

## CAISO Alert Severity Levels

Flex Alerts are one level in a broader escalation hierarchy. From the [CAISO Emergency Notifications Fact Sheet](https://www.caiso.com/Documents/Emergency-Notifications-Fact-Sheet.pdf):

| Alert | Description |
|---|---|
| **Flex Alert** | Voluntary conservation call when ISO anticipates using nearly all available resources |
| **EEA Watch** | Early warning that energy deficiencies may occur |
| **EEA 1** | All resources in use or committed; energy deficiencies expected |
| **EEA 2** | ISO requests emergency energy from all resources; emergency demand response activated |
| **EEA 3** | ISO unable to meet minimum contingency reserve requirements; controlled power curtailments imminent or in progress |

The v2 endpoint documented above only covers **Flex Alerts**. It is unknown whether the EEA alerts are available through a similar v2 endpoint.

## Limitations and Critique

The current API was designed to power a human-facing web dashboard. Several of its fields — `Title`, `Region`, timestamp strings — are formatted for human readers, not for machine-to-machine communication. This is a critical distinction: as home automation, building energy management systems, and demand-response platforms increasingly need to receive and act on grid alerts programmatically, the API must serve two audiences. Communicating alerts to humans remains important, but we also need to communicate them to machines — and the current format is not adequate for that purpose.

Below are the key challenges, particularly for anyone trying to determine whether a Flex Alert applies to a specific location or individual.

### Regions are undefined and unmappable

The most significant limitation is that **CAISO does not define the geographic boundaries of its regions**. The region names ("Northern CA Region", "Southern CA Region", etc.) are informal labels that roughly correspond to investor-owned utility (IOU) service territories, but CAISO has confirmed they maintain no zip-code-to-region mapping, no GIS boundary data, and no formal definition of what "Northern" vs. "Southern" California means in this context.

This makes it impossible to reliably answer "does this Flex Alert apply to my address?" without making assumptions. A consumer in a municipal utility district (e.g., LADWP, SMUD, SVP) that overlaps geographically with a CAISO region but is not part of CAISO's balancing area has no way to know from this data whether the alert is relevant to them.

In practice, CAISO's own notification system sidesteps this problem entirely: when a Flex Alert is issued, **all subscribers are notified regardless of region**, even for regional alerts. The region field is effectively advisory.

### The `Region` field is a fragile encoding

Multiple region values are packed into a single string with an ambiguous delimiter (semicolons in some versions, commas in others). Region names have changed across API versions without a version indicator. There is no enum or schema — consumers must match against known strings that can change without notice.

A more robust design would provide:
- A `Regions` array (JSON list) instead of a delimited string
- Stable, machine-readable region identifiers (e.g., `CAISO_GRID`, `NORTHERN_CA`) distinct from display names
- A companion endpoint or schema defining the valid region values

### No geographic granularity below "half the state"

Even if regions were precisely defined, the granularity is extremely coarse. California is 164,000 square miles with dramatically different climate zones, grid topology, and demand patterns. "Northern California" and "Southern California" are each larger than many US states. The now-dormant "South OC San Diego Region" shows that finer-grained targeting was contemplated at one point, but it was never broadly used and CAISO has discussed retiring it.

For a system whose purpose is to trigger conservation behavior in specific areas under stress, the inability to target alerts below "half the state" is a significant design gap. A zip-code or county-level mapping, or alignment with utility service territory boundaries (which do have published GIS data), would make the data far more actionable.

### Timestamp format instability

The `UpdatedOn` field changed from a custom format (`MM-DD-YYYY HH:MM:SS` with implicit Pacific time) to ISO 8601 (`2024-06-17T15:19:40Z` in UTC) after the May 2024 website relaunch. It is unclear whether `StartDate`/`EndDate` have also changed. There was no versioning, changelog, or advance notice — consumers had to discover the change by observation.

Clients must now handle both formats defensively, and cannot know whether further format changes will occur.

### No historical data or event lifecycle

The API is a point-in-time snapshot: it shows what is active now and what is scheduled. Once an alert ends, it disappears from the response with no trace. CAISO has confirmed they maintain **no archive of historical JSON payloads**.

There is no way to:
- Query past Flex Alerts programmatically
- Determine whether an alert was modified or cancelled after initial publication
- Build a dataset for analysis (frequency, duration, regional patterns over time)

An event log with lifecycle states (created, modified, cancelled, expired) and a query-by-date-range endpoint would make the data useful for research, grid analytics, and retrospective analysis.

### No versioning or stability guarantees

The API has no version indicator in the response payload, no published schema, no changelog, and no deprecation policy. The endpoint URL, field names, timestamp formats, and region names have all changed historically without notice. The previous JSON endpoint (`FlexAlerts.json`) was simply turned off when the website was relaunched in May 2024.

Consumers building automated systems on this API should expect it to break without warning and plan for defensive parsing and monitoring.

### Summary of suggested improvements

| Issue | Suggested Improvement |
|---|---|
| Undefined regions | Publish GIS boundaries or map regions to utility service territories |
| Delimited string regions | Provide a `Regions` JSON array with stable identifiers |
| Coarse geography | Support zip-code or county-level targeting |
| Inconsistent timestamps | Use ISO 8601 exclusively across all fields |
| No history | Provide a historical query endpoint and event lifecycle log |
| No versioning | Add a `Version` field; publish a schema and changelog |
| No stability contract | Document an API deprecation policy with advance notice |

## Proposal: A Machine-to-Machine Flex Alert API

The current endpoint was built to feed a website. What follows is a proposal for a proper machine-to-machine API that could be consumed reliably by home automation controllers, building energy management systems, demand-response aggregators, and research tools — without guessing at string formats or geographic boundaries.

### Design Principles

1. **Separate human-readable text from machine-actionable data.** Display strings like `Title` can coexist alongside structured fields, but every field a machine needs to act on must have a stable, typed, unambiguous representation.
2. **Self-describing payloads.** Every response must declare its schema version so that clients can detect and adapt to changes without silent breakage.
3. **Discoverable, downloadable schemas.** The API must publish formal schemas at a well-known URL. Clients can validate responses, generate code, and detect incompatible changes automatically.
4. **Stable identifiers, not display strings.** Regions, alert types, and lifecycle states must use machine-readable identifiers that are independent of how they are displayed to humans.
5. **Geographic precision.** Alerts that are meant to trigger action at a specific location must provide enough geographic data to determine applicability — not just a label like "Northern California."

### Versioning

Every response includes a top-level `schemaVersion` field using semantic versioning:

```json
{
  "schemaVersion": "1.0.0",
  ...
}
```

**Version contract:**
- **Patch** (1.0.x): Documentation clarifications, no payload changes
- **Minor** (1.x.0): New optional fields added; existing fields unchanged. Clients that ignore unknown fields are unaffected.
- **Major** (x.0.0): Breaking changes — field removals, type changes, semantic changes. Served at a new base path (e.g., `/v3/`). Prior version continues to be served for a documented deprecation period (minimum 12 months).

The current schema version and the deprecation schedule for prior versions should be published at a well-known metadata endpoint (see below).

### Schema Discovery

The API publishes formal schemas at well-known URLs relative to the API base:

```
GET /api/EmergencyNotices/v3/schema.json          → JSON Schema (draft 2020-12)
GET /api/EmergencyNotices/v3/openapi.yaml          → OpenAPI 3.1 specification
GET /api/EmergencyNotices/v3/regions.json           → Region registry (see below)
GET /api/EmergencyNotices/v3/alert-types.json       → Alert type registry
GET /api/EmergencyNotices/v3/meta.json              → API metadata (current version, deprecation dates)
```

Schemas are versioned alongside the API. Clients can fetch `schema.json` at startup or build time to validate responses, generate types, or detect incompatible changes. The `meta.json` endpoint provides:

```json
{
  "currentVersion": "1.0.0",
  "supportedVersions": [
    { "version": "1.0.0", "status": "current", "basePath": "/v3/" }
  ],
  "deprecatedVersions": [
    { "version": "0.1.0", "basePath": "/v2/", "sunsetDate": "2025-12-31" }
  ],
  "schemaUrls": {
    "jsonSchema": "/v3/schema.json",
    "openApi": "/v3/openapi.yaml",
    "regions": "/v3/regions.json",
    "alertTypes": "/v3/alert-types.json"
  }
}
```

### Region Registry

Instead of embedding opaque display strings in alert payloads, regions are defined in a registry that maps stable identifiers to human names and geographic boundaries:

```json
{
  "schemaVersion": "1.0.0",
  "regions": [
    {
      "id": "CAISO_GRID",
      "name": "CAISO Grid (Statewide)",
      "description": "Entire CAISO balancing authority area",
      "scope": "system-wide",
      "geoJson": "https://wwwmobile.caiso.com/api/EmergencyNotices/v3/regions/CAISO_GRID.geojson"
    },
    {
      "id": "NORTHERN_CA",
      "name": "Northern California Region",
      "description": "Roughly corresponds to PG&E service territory within CAISO BAA",
      "scope": "sub-region",
      "parentRegion": "CAISO_GRID",
      "iouAlignment": ["PG&E"],
      "geoJson": "https://wwwmobile.caiso.com/api/EmergencyNotices/v3/regions/NORTHERN_CA.geojson"
    },
    {
      "id": "SOUTHERN_CA",
      "name": "Southern California Region",
      "description": "Roughly corresponds to SCE and SDG&E service territories within CAISO BAA",
      "scope": "sub-region",
      "parentRegion": "CAISO_GRID",
      "iouAlignment": ["SCE", "SDG&E"],
      "geoJson": "https://wwwmobile.caiso.com/api/EmergencyNotices/v3/regions/SOUTHERN_CA.geojson"
    },
    {
      "id": "VEA",
      "name": "Valley Electric Association Region",
      "description": "VEA service territory in Pahrump, NV; part of CAISO BAA",
      "scope": "sub-region",
      "parentRegion": "CAISO_GRID",
      "iouAlignment": ["VEA"],
      "geoJson": "https://wwwmobile.caiso.com/api/EmergencyNotices/v3/regions/VEA.geojson"
    }
  ]
}
```

Each region links to a downloadable GeoJSON boundary file. This enables:
- **Point-in-polygon lookups:** Given a latitude/longitude, determine which CAISO region(s) contain it
- **Map visualization:** Render alert regions on a map without hardcoded geometry
- **Automated updates:** If CAISO redefines a boundary, clients that fetch the registry pick up the change

The `parentRegion` field expresses containment: a `CAISO_GRID` alert implicitly covers all sub-regions. The `iouAlignment` field documents the approximate correspondence to utility service territories, helping consumers who already know their utility but not their CAISO region.

### Alert Type Registry

Similarly, alert severity levels get stable identifiers:

```json
{
  "schemaVersion": "1.0.0",
  "alertTypes": [
    {
      "id": "FLEX_ALERT",
      "name": "Flex Alert",
      "severity": 1,
      "description": "Voluntary conservation request",
      "voluntary": true
    },
    {
      "id": "EEA_WATCH",
      "name": "EEA Watch",
      "severity": 2,
      "description": "Early warning of potential energy deficiency",
      "voluntary": true
    },
    {
      "id": "EEA_1",
      "name": "Energy Emergency Alert Stage 1",
      "severity": 3,
      "description": "All resources in use or committed; energy deficiencies expected",
      "voluntary": false
    },
    {
      "id": "EEA_2",
      "name": "Energy Emergency Alert Stage 2",
      "severity": 4,
      "description": "Emergency energy requested; emergency demand response activated",
      "voluntary": false
    },
    {
      "id": "EEA_3",
      "name": "Energy Emergency Alert Stage 3",
      "severity": 5,
      "description": "Controlled power curtailments imminent or in progress",
      "voluntary": false
    }
  ]
}
```

The `severity` field provides a sortable numeric ranking. The `voluntary` field tells automation systems whether this is a request or a directive — a meaningful distinction for automated response policies.

### Proposed Alert Payload

```json
{
  "schemaVersion": "1.0.0",
  "updatedAt": "2024-09-05T23:05:29Z",
  "active": [
    {
      "id": "fa-2024-0905-001",
      "alertType": "FLEX_ALERT",
      "title": "Flex Alert for September 5, 2024",
      "status": "active",
      "regions": ["CAISO_GRID"],
      "startTime": "2024-09-05T23:00:00Z",
      "endTime": "2024-09-06T05:00:00Z",
      "issuedAt": "2024-09-05T15:00:00Z",
      "modifiedAt": "2024-09-05T23:05:29Z",
      "revision": 2,
      "lifecycle": [
        {
          "event": "created",
          "timestamp": "2024-09-05T15:00:00Z",
          "note": "Initial issuance"
        },
        {
          "event": "modified",
          "timestamp": "2024-09-05T23:05:29Z",
          "note": "End time extended from 04:00Z to 05:00Z"
        }
      ]
    }
  ],
  "future": [
    {
      "id": "fa-2024-0906-001",
      "alertType": "FLEX_ALERT",
      "title": "Flex Alert for September 6, 2024",
      "status": "scheduled",
      "regions": ["NORTHERN_CA", "SOUTHERN_CA"],
      "startTime": "2024-09-06T23:00:00Z",
      "endTime": "2024-09-07T04:00:00Z",
      "issuedAt": "2024-09-05T20:00:00Z",
      "modifiedAt": "2024-09-05T20:00:00Z",
      "revision": 1,
      "lifecycle": [
        {
          "event": "created",
          "timestamp": "2024-09-05T20:00:00Z",
          "note": "Initial issuance"
        }
      ]
    }
  ]
}
```

Key differences from the current API:

| Aspect | Current | Proposed |
|---|---|---|
| **Schema version** | Absent | `schemaVersion` in every response |
| **Timestamps** | Mixed formats, implicit timezone | ISO 8601 with explicit UTC throughout |
| **Regions** | Delimited string of display names | JSON array of stable identifiers from the registry |
| **Alert types** | Implicit (endpoint only returns Flex Alerts) | Explicit `alertType` from the registry; single endpoint for all alert levels |
| **Title** | Only human-readable identifier | Retained for display, but `alertType` + `regions` carry the machine semantics |
| **Lifecycle** | Absent; events vanish when they end | `lifecycle` array records creation, modification, and cancellation |
| **Revision tracking** | Absent | `revision` counter increments on any change; `modifiedAt` timestamp |
| **Event identity** | Numeric `ID`, reuse policy unknown | String `id` with structured format (type-date-sequence) |
| **Status** | Inferred from which array an event appears in | Explicit `status` field (`scheduled`, `active`, `cancelled`, `expired`) |

### Historical Query Endpoint

In addition to the real-time snapshot, a companion endpoint would serve historical data:

```
GET /api/EmergencyNotices/v3/history?from=2024-01-01&to=2024-12-31&alertType=FLEX_ALERT
```

This would return all alerts (including cancelled and expired) within the date range, with their full lifecycle. This enables research, post-event analysis, and building local archives without scraping the real-time endpoint.

### What This Enables

With these changes, a home automation system could:

1. **At setup time:** Fetch `regions.json`, download the GeoJSON boundaries, and determine which region(s) the home's coordinates fall within.
2. **At runtime:** Poll the alert endpoint, match `regions` array values against the pre-computed region membership, and trigger conservation actions (thermostat setback, EV charge deferral, battery discharge) only when the alert genuinely applies to that location.
3. **On schema change:** Detect a `schemaVersion` bump, fetch the updated schema, and either adapt automatically (minor version) or alert the operator (major version).
4. **For analytics:** Query the history endpoint to analyze alert frequency, duration trends, and regional patterns over time.

None of this is possible with the current API, where a machine cannot reliably determine whether "Northern CA Region" includes a given street address, and where the payload format can change without warning or detection.