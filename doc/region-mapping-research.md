# Flex Alert Region Mapping Research

The goal: given a street address, lat/lon, or zip code, determine which CAISO Flex Alert region(s) apply.

This is harder than it should be. CAISO does not publish geographic definitions for their Flex Alert regions. This document collects what we've found that might help piece together a mapping.

## What CAISO has told us

From direct correspondence with CAISO staff (Kyle Clayton, 2023-2024):

- CAISO has **no zip-code-to-region mapping** and does not target communications by zip code.
- Zip codes collected from Flex Alert subscribers are only used to count sign-ups per area.
- When a Flex Alert is issued (most often system-wide), **all subscribers are notified regardless of region**.
- The regions **roughly align with IOU service areas**: North = PG&E, South = SCE + SDG&E.
- **VEA Region** = Valley Electric Association, Pahrump, NV. Part of CAISO balancing area. A Flex Alert has never been called specifically for VEA.
- **South OC San Diego Region** has not been used in a long time — was created for specific historical issues. CAISO has discussed retiring it.
- Kyle suggested the CEC's IOU map as an approximate proxy.

## CEC Electric IOU Map (PDF only)

**Source:** [Electric Investor Owned Utility Areas](https://gis.data.ca.gov/documents/CAEnergy::electric-investor-owned-utility-areas/about) — California Energy Commission, 2020

**Download:** https://www.arcgis.com/sharing/rest/content/items/be5721ddbfac47e382dc0dea9c41ac20/data (1.4 MB PDF)

The map shows six IOUs:

| IOU | Coverage | Flex Alert Region (approximate) |
|---|---|---|
| PG&E (Pacific Gas & Electric) | Most of northern/central California | Northern CA Region |
| SCE (Southern California Edison) | Inland southern California | Southern CA Region |
| SDG&E (San Diego Gas & Electric) | San Diego area | Southern CA Region |
| PacifiCorp | Small area in far northeast CA | ? (within CAISO BAA?) |
| Liberty Utilities | Small area in eastern Sierra | ? (within CAISO BAA?) |
| Bear Valley Electric Service | Small mountain area in San Bernardino Co. | ? (within CAISO BAA?) |

**Limitations:**
- PDF only — no GIS feature layer, GeoJSON, or Shapefile available from CEC for download
- The underlying ArcGIS project exists (`T:\Projects\Open Data Hub\ArcPro_Projects\Electric_IOU_Areas\Electric_IOU_Areas.aprx`) but is not published as a queryable service
- White areas on the map (municipal utilities, co-ops) are geographically within the state but outside IOU territories — many are also outside CAISO's balancing area (e.g., LADWP, SMUD, SVP, TID)
- Even if digitized, IOU boundaries don't cleanly map to "does this Flex Alert apply to me?" because:
  - Municipal utility customers within the geographic footprint may not be in CAISO's balancing area
  - CAISO's own region definitions may not align perfectly with IOU boundaries
  - The smaller IOUs (PacifiCorp, Liberty, Bear Valley) don't obviously map to any Flex Alert region

## The fundamental mismatch

There are at least three different geographic concepts at play, and none of them align perfectly:

1. **CAISO Balancing Authority Area (BAA)** — the grid territory CAISO operates. Includes PG&E, SCE, SDG&E, and VEA, plus some smaller entities. Does NOT include LADWP, SMUD, SVP, TID, IID, and many other municipal/co-op utilities.

2. **IOU service territories** — where each investor-owned utility provides retail electric service. These are well-defined with regulatory boundaries, but they are not identical to the CAISO BAA.

3. **Flex Alert regions** — informal, undefined labels applied to conservation alerts. Roughly correspond to IOU territories but with no formal mapping.

A consumer at a given address needs to traverse all three layers to determine applicability:
- Am I in the CAISO BAA? (If not, Flex Alert is irrelevant regardless of region)
- If so, which Flex Alert region am I in?

## Grid Emergencies History Report

The PDF at `doc/pdf/grid-emergencies-history-report-1998-to-present.pdf` is the closest thing to a historical archive. Revised 04/14/2026. Key findings:

### Transmission Emergencies reveal finer-grained geography

While Flex Alert regions are coarse ("CAISO Grid", "Northern CA"), the **Transmission Emergency** records in the Reason field contain much more specific location information. Examples from 2024:

| Date | Region Field | Actual Location (from Reason) |
|---|---|---|
| 3/1/2024 | Northern CA | "Local Humboldt Region Only" |
| 7/2/2024 | Northern CA | "Wild Fire and potential loss of local area transmission and generation" |
| 7/11/2024 | Northern CA | "Rio Oso/Palermo Area" |
| 7/12/2024 | Northern CA | "Rio Oso/Palermo Area" |
| 7/15/2024 | CAISO Grid | "Lost Hills, Rancho and White Fires" |
| 7/15/2024 | Northern CA | "Midway area due to fires" |
| 7/18/2024 | Northern CA | "Humboldt area due to the Hill fire" |
| 7/19/2024 | Northern CA | "Palermo/Rio Oso due to high temperatures and high loading" |
| 4/30/2025 | Northern CA Region | "High Line Loading in the Rio Oso Area" |

This shows that **CAISO can and does identify specific sub-regional areas** when the grid issue is localized — they just bury this information in free-text Reason strings rather than structured data. The formal Region field stays at "Northern CA" even when the issue is specific to Humboldt or Rio Oso.

### Alert terminology changed in May 2022

CAISO transitioned to NERC's Energy Emergency Alert (EEA) system. The old terms and their new equivalents:

| Pre-May 2022 | Post-May 2022 |
|---|---|
| Alert | EEA Watch |
| Warning | (no direct equivalent; folded into EEA Watch) |
| Stage 1 Emergency | EEA 1 |
| Stage 2 Emergency | EEA 2 |
| Stage 3 Emergency | EEA 3 |
| 1-Hour Probable Load Interruptions | (retired) |
| Voluntary Load Reduction Program | (retired) |

### EEA 3 has two sub-stages

From the Emergency Notifications Fact Sheet:
- **EEA 3 — Preparing for rotating power outages:** Utilities alerted to prepare, but outages not yet ordered
- **EEA 3 — Ordering rotating power outages:** Utilities ordered to begin rotating outages

### Flex Alert frequency data

| Year | Flex Alert Days | Notes |
|---|---|---|
| 2000 | 20 | |
| 2001 | 26 | California energy crisis |
| 2006 | 18 | Record peak 50,270 MW |
| 2017 | 4 | |
| 2020 | 10 | |
| 2021 | 8 | |
| 2022 | 11 | All-time peak record: 52,061 MW on Sep 6 |
| 2023 | 0 | |
| 2024 | 0 | |
| 2025 | 0 | |
| 2026 | 0 | (through Apr 2026) |

No Flex Alerts since 2022. The grid has been stable despite continued high peak loads (48,323 MW in 2024).

### Notice ID structure

Event IDs follow a pattern like `202436223643` with a revision suffix (e.g., `-98`, `-37`, `-11`). The suffix appears to increment with each update to the notice. ENDED notices get their own suffix number. This is another example of lifecycle information that exists internally but isn't exposed in the API.

### Extreme Weather Event Communications Timeline

From `doc/pdf/extreme-weather-event-process-and-communications.pdf` (CS/08/2025):

- **4-7 days out:** CAISO monitors demand forecast, assesses resource adequacy, coordinates with Governor's Office, neighboring BAs, utilities
- **1-4 days out:** Issues RMO notifications, coordinates with DOE, neighboring BAs, DR Event Board
- **1 day out:** Issues Flex Alert and/or EEA Watch. Communication channels: Today's Outlook, ISO Today mobile app, MNS, emergency notifications email, news releases, Daily Briefing, FlexAlert.org, subscription lists, social media
- **Operating day:** Issues EEA notices as needed. De-escalation/all-clear notices issued via same channels.

Flex Alerts are generally issued **day-ahead** (the day before). EEA Watch may be issued day-of depending on conditions.

## CAISO Developer Portal

The developer portal at `developer.caiso.com` provides access to OASIS and market data APIs. As of April 2026, it contains **no Flex Alert data, no region definitions, and no GIS data**. The Flex Alert endpoint is part of the separate Today's Outlook API suite and requires no developer account.

## Potential GIS data sources to investigate

### CAISO BAA boundary
- [ ] Does CAISO publish a GIS boundary for their balancing authority area?
- [ ] FERC/NERC BAA maps — are these available as downloadable GIS data?
- [ ] EIA has utility service territory data: https://www.eia.gov/maps/layer_info-m.php — includes balancing authority boundaries

### IOU service territory boundaries (GIS format)
- [ ] CPUC may have regulatory service territory boundaries in GIS format
- [ ] EIA Form 861 data includes utility service territories
- [ ] Individual IOUs may publish their own service territory GIS data
- [ ] HIFLD (Homeland Infrastructure Foundation-Level Data) has electric utility boundaries

### California-specific
- [ ] CEC may have unpublished GIS layers behind the PDF map — worth asking
- [ ] CalEnviroScreen and other CA state tools may embed utility territory data
- [ ] CA Public Utilities Commission GIS data

## Ideas for a practical mapping

Even without perfect data, a "good enough" approach might be:

1. **Start with EIA utility service territory polygons** for PG&E, SCE, SDG&E, VEA
2. **Union PG&E → Northern CA Region**, **Union SCE + SDG&E → Southern CA Region**, **VEA → VEA Region**
3. **Intersect with CAISO BAA boundary** to exclude areas within IOU territory that aren't in the CAISO balancing area (if any exist)
4. **Accept the gap:** addresses in municipal utility territories within the CAISO BAA (if any) won't map to a Flex Alert region, but Flex Alerts are arguably not aimed at those customers anyway

This would be imperfect but far better than nothing — and better than what CAISO provides today.
