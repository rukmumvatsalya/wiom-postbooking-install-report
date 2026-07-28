# Context & methodology

How the **booking-to-installation** report was built, what each number means, and where to be careful.
Read this before quoting any figure.

---

## Purpose

Understand what customers do and ask **after they book but before they are installed** — the acquisition
funnel (app install → booked → installed) and the support-chat experience along the way — comparing two
windows: **8–30 June 2026** vs **1–22 July 2026** (IST).

---

## Data sources

All pulled live from Metabase (Snowflake), database `Snowflake` (id 113).

| Table | What it provides |
|---|---|
| `PROD_DB.DYNAMODB_READ.INTENT_CLASSIFICATIONS` | The support chat. Every customer message, its classified intent, sentiment, and the customer's `USER_STATE` at the time. Post-booking rows are those where `APP_PAGE_NAME IS NULL` and `USER_STATE` is populated. |
| `PROD_DB.PUBLIC.BOOKING_LOGS` | The booking event log (`EVENT_NAME`, `MOBILE`, `ADDED_TIME`). Used for the earlier fee→chat contact-rate section. |
| **`Booking To Install.xlsx` → `B2I_Agg` sheet** | **The authoritative booking-to-install dataset** (one row per booking window, 12,392 rows, May–Jul 2026) with every stage timestamp, `is_installed`, `is_refunded`, `final_category` (non-conversion reason), `group_name` (cost-breakdown variant), and `b2i_tat_days`. **This is now the source for the funnel, non-conversion reasons, install-by-variant, and TAT.** |
| App-install counts (48,118 June / 49,800 July) | **Supplied externally** (app analytics). Used as the funnel's top denominator. |

`PROD_DB.DBT_CSP.FCT_BOOKING_TO_INSTALL_JOURNEY` (the warehouse equivalent) is **stale** — only 98 rows from
May 2026 — so the funnel uses the `B2I_Agg` extract instead.

**Funnel figures updated 28 Jul 2026** to the `B2I_Agg` authoritative data, computed for the report's exact
windows (8–30 Jun / 1–22 Jul, by `booking_confirm_date`). Headline: bookings 3,162 (Jun) / 4,790 (Jul);
install % ~25% / 26%; the biggest leak is **technician assigned (50%) → arrived (~28%)**, and the top reason a
booking never installs is **non-serviceable area (55–57% of non-installs)**. Variants **G (9.4%)** and
**H (12.4%)** install far below I/I1/C/D/E (31–35%). The earlier `BOOKING_LOGS` funnel numbers (3,084 → 827 etc.)
are superseded by these.

---

## Windows, timezones, de-duplication

- **Windows:** 8–30 June (23 days) vs 1–22 July (22 days). Per-customer rates compare directly; raw totals do not (unequal length).
- **Timezone:** `INTENT_CLASSIFICATIONS.CREATED_AT` is UTC → always `DATEADD(minute, 330, …)` for IST.
  `BOOKING_LOGS.ADDED_TIME` is **also UTC** (stored as `TIMESTAMP_NTZ`, no timezone) → same `DATEADD(minute, 330, …)` shift.
  *(Corrected 28 Jul 2026: an earlier version of this note claimed `ADDED_TIME` was already IST. It is not — the
  hour-of-day activity trough lands at 3:30–6:30 AM IST only under the UTC reading, and the published funnel
  numbers themselves reproduce only with the +330 shift, so the queries were right and this note was wrong.)*
- **De-duplication:** messages de-duped on `(MOBILE, TRIM(USER_MESSAGE), TIMESTAMP)`.
- **`_FIVETRAN_DELETED` is NOT filtered** on `INTENT_CLASSIFICATIONS` — DynamoDB TTL is 7 days, so filtering
  would silently keep only the last week.
- **Identity:** post-booking `MOBILE` is a real phone number; used as the join key across chat and booking logs
  (normalised to the last 10 digits, valid `^[6-9][0-9]{9}$`).

---

## The acquisition funnel

Cohorted on customers who **booked in the window**; later stages counted at any date since (as of 27 Jul 2026).

| Stage | Event (`BOOKING_LOGS`) | June | July |
|---|---|---|---|
| App installs | *(external)* | 48,118 | 49,800 |
| Booked | `booking_confirmed` | 3,084 | 4,662 |
| — paid booking fee | `booking_fee_captured` | 2,603 | 3,290 |
| Moved to install | `slot_confirmed_by_customer` | 1,967 | 2,661 |
| Installed | `installed` | 827 | 1,266 |

**Three cautions:**
1. **App-install → booked is a period ratio, not a tracked cohort.** The app-analytics id does not join to the
   booking mobile, so this is *bookings-in-window ÷ installs-in-window*, not "these exact 48,118 people."
2. **Installs are right-censored.** A customer who booked on 22 July has had only days to be installed, so July's
   install count is understated. June (a month of runway, ~27% of bookings installed) is the more complete read.
3. **The "paid booking fee" row is all fee payers in the window, not a subset of the booked row.** ~109 (June) /
   ~158 (July) fee payers have no `booking_confirmed` event inside the same window; restricting to the booked
   cohort gives 2,494 / 3,132. The all-payers definition is the one the contact-rate section also uses, so the
   report is internally consistent — just don't read the funnel as strictly nested at that row.

---

## The support-chat tiers

The 18 booking-to-installation `USER_STATE`s split into four groups by **what customers are actually there for**,
measured by their intent mix. This split is the core of the analysis — see the v1 correction below.

| Tier | Customers (both windows) | Frustrated | Meaning |
|---|---|---|---|
| Getting to the technician | 6,530 | 10.6% | Genuine install queue — patient, asking status / reschedule / call-tech |
| Slot breached | 1,293 | 32.8% | Promised slot missed — anger + churn |
| Just installed, net not working | 1,237 | 25.7% | Post-install support (24.8% "internet not working") |
| Address pending (mostly existing) | 6,519 | 24.5% | ~54% existing customers chatting about their *current* broken service |

**Metric definitions**
- **Frustrated** = share of messages the production model tagged `FRUSTRATED` or `ANGRY`.
- **Dead-end** = free-text messages the classifier mapped to `FallbackIntent`.
- **Messages per customer** = de-duped messages ÷ distinct customers, within the window.
- **Contact rate** = of customers who paid a booking fee in the window, the share who also appear in support chat
  (~87% June, ~89% July — a floor, since payers who chat after the window aren't counted).
- **"What they ask" counts** = unique **customer × journey-state** pairs, not unique customers: a customer asking
  the same thing from two states counts twice. This mostly affects cross-state intents — checkStatus's 5,035 (June)
  is ~3,600 distinct customers across ~5,000 state-contexts; concentrated intents (internetIssue, callRohit) are
  barely inflated.
- **State-table "Customers"** = June + July per-window counts summed (a customer chatting in both windows counts
  twice). The four-tier table instead counts distinct customers across both windows combined — the two tables'
  customer columns are deliberately on different bases and won't reconcile by addition.

---

## What changed from the first draft (v1 → v2)

This report was corrected twice after critical review:

1. **v1 mis-scoped the biggest state.** It treated all 18 states as one clean install funnel and reported ~18%
   frustration as install anger. Interrogation showed `ADDRESS_SUBMISSION_PENDING` (the largest state) is **54%
   existing customers** — its frustration is **85.9%** among "my internet isn't working" messages vs **3.7%** among
   genuine install questions. The four-tier split fixes this misattribution.
2. **The masthead mixed measurement bases.** It paired a per-window total (7,328 → 8,113 customers who chatted from
   install-labelled states) with a both-windows-combined "genuine install-wait" figure, implying most were genuine.
   The true per-window genuine install-wait population is **2,842 → 4,194**. Labels corrected.

---

## Limitations

- **No per-state population denominator.** Frustration/cancel rates are of customers who *chatted*, not everyone in
  each state. The booking-fee funnel gives an overall contact rate, not a per-stage one. A per-state denominator
  would need the `MESSAGE_HISTORY` outbound-workflow join, still unresolved (post-booking id is `MOBILE`/`ACCOUNT_ID`,
  not `USER_ID`).
- **Why existing customers sit in install states is unconfirmed** — most likely lapsed customers re-booking, whose
  account shows a new booking while their old connection is live and broken. Worth confirming with the
  state-machine owners.
- **Sentiment is a model label,** and the bot's reply is only partly recorded — this measures what customers said,
  not whether they were helped.
- **Thin states** (under ~120 customers) are marked in the report; don't draw conclusions from them.

---

## Privacy

Source data contains real customer phone numbers, names and addresses. Every verbatim in the report was limited to
phrases used by **2+ distinct customers** and scrubbed of numbers, names and addresses. The published HTML was
swept for PII before each push (0 phone numbers, 0 names, 0 addresses). Raw data files, SQL, and the Metabase key
are git-ignored and never committed.
