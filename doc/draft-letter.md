# Draft: Flex Alert Machine-Readable Data

**To:** [Regulator / Legislator]
**From:** [Name]
**Re:** CAISO Flex Alerts need a machine-to-machine data format

---

California's grid operator (CAISO) issues Flex Alerts to call for voluntary electricity conservation during grid emergencies. These alerts are communicated effectively to humans — via text, email, social media, the [flexalert.org](https://www.flexalert.org) website, and the [ISO Today](https://www.caiso.com/iso-today) smartphone app.

However, there is no reliable way for **machines** to receive and act on these alerts.

This matters because home and building automation systems — smart thermostats, EV chargers, battery storage, water heaters — can respond to grid stress automatically and instantly, without requiring a human to read a text message and manually adjust settings. The technology exists today. What's missing is a dependable data feed.

**The problem:** CAISO provides a [single data feed](https://github.com/grid-coordination/caiso-flex-alert#active-endpoint-todays-outlook-api-v2) for Flex Alert status, but it was designed to power a website dashboard, not to serve as infrastructure for automated demand response. Key issues:

- **Regions are undefined.** The API says "Northern CA Region" but CAISO has [no geographic boundary data](https://github.com/grid-coordination/caiso-flex-alert#regions-are-undefined-and-unmappable) for what that means. There is no way for a device to determine whether an alert applies to its location.
- **The format is fragile.** Timestamps, region names, and field formats have [changed without notice](https://github.com/grid-coordination/caiso-flex-alert#timestamp-format-instability). There is no versioning, no schema, and no deprecation policy.
- **No history.** Alerts [vanish when they expire](https://github.com/grid-coordination/caiso-flex-alert#no-historical-data-or-event-lifecycle). There is no archive for research or grid planning.

**The ask:** CAISO should publish a [versioned, schema-driven API](https://github.com/grid-coordination/caiso-flex-alert#proposal-a-machine-to-machine-flex-alert-api) with:

1. Stable, machine-readable identifiers (not display strings)
2. Downloadable GIS boundaries for alert regions
3. A published schema and version contract
4. A historical query endpoint

A detailed proposal with example payloads and schema designs is available at:
https://github.com/grid-coordination/caiso-flex-alert

This is a low-cost, high-leverage improvement. The data already exists inside CAISO's systems — it just needs to be published in a format that machines can use reliably.
