# Post-booking chat — booking to installation

What booked customers ask while they wait to be installed.

Analysis of Wiom post-booking support chat across the booking-to-installation journey,
comparing 8–30 June 2026 against 1–22 July 2026, with the app-install → booked → installed funnel.

**[Read the report →](https://rukmumvatsalya.github.io/wiom-postbooking-install-report/)**

## Scope

- 18 booking-to-installation journey states, tiered by what customers are really there for
- 64,410 support messages; funnel from BOOKING_LOGS (app install → booked → slot → installed)
- Verbatims scrubbed of PII (no phone numbers, names, or addresses)

## Build

Single self-contained HTML file — no scripts, no external requests. Open `index.html` or serve the directory.

## Methodology

See [CONTEXT.md](CONTEXT.md) — data sources, metric definitions, the acquisition-funnel logic, caveats (contamination, censoring, identity gap), and the v1→v2 corrections.
