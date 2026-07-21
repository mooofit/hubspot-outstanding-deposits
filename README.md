# Outstanding Deposits Tracker

A real-time dashboard for tracking **outstanding deposit balances** on HubSpot deals. It keeps HubSpot deals and their split-payment transactions mirrored into Supabase, then surfaces the aggregated "who still owes what, and for how long" view in a clean, filterable web dashboard.

> Built for the BCM / Hero Ladies sales pipeline.

---

## What it does

- Listens for changes in HubSpot (deals + a custom "transaction / split payment" object) via a webhook.
- Mirrors those changes into Supabase so the data is always fresh — no manual exports.
- Aggregates each deal's deposits vs. its total contract value to compute **remaining collectables** and **how many days a deposit has been outstanding**.
- Presents everything in a dashboard with search, filtering (by consultant, product, and age bucket), sorting, pagination, and a dark/light theme.
- Locks access behind a corporate email magic-link login.

---

## How it works (architecture)

```
                         (deal & transaction changes)
   ┌─────────────┐  webhook   ┌────────────────────────┐   upsert   ┌──────────────┐
   │   HubSpot   │ ─────────► │  Supabase Edge Function │ ─────────► │   Supabase   │
   │  (CRM data) │            │       (index.ts)        │            │   Postgres   │
   └─────────────┘            └────────────────────────┘            └──────┬───────┘
                                                                           │
                                                          SQL view: dashboard_outstanding_deals_summary
                                                                           │
                                                                    read (anon key + auth)
                                                                           ▼
                                                                   ┌──────────────┐
                                                                   │  index.html  │
                                                                   │  (dashboard) │
                                                                   └──────────────┘
```

**Data flow:**
1. A deal or transaction is created/edited in HubSpot.
2. HubSpot fires a webhook to the Edge Function (`index.ts`).
3. The function normalizes the event and **upserts** it into the relevant Supabase table. For transactions, it also looks up the linked parent deal in HubSpot and keeps the deal's name/stage/amount mirrored.
4. A Supabase SQL view (`dashboard_outstanding_deals_summary`) joins/aggregates deals and their transactions into per-deal summary rows.
5. `index.html` authenticates the user, reads that view, and renders the dashboard.

---

## Repository contents

| File | Role | Runtime |
|------|------|---------|
| `index.ts`   | **Backend** — HubSpot → Supabase sync (webhook receiver) | Supabase Edge Function (Deno) |
| `index.html` | **Frontend** — the dashboard UI | Static page (any host; currently GitHub Pages) |

---

## Backend — `index.ts`

A Deno-based Supabase Edge Function that receives batched HubSpot webhook events and writes them into Supabase.

It handles two categories of events:

**1. Deal events** (`deal.creation`, `deal.propertyChange`)
Maps changed properties into the `hubspot_outstanding_deals` table:

| HubSpot property | Supabase column |
|------------------|-----------------|
| `dealname`   | `deal_name`  |
| `dealstage`  | `deal_stage` |
| `amount`     | `amount`     |

**2. Transaction / split-payment events** (`customObject.creation`, `customObject.propertyChange`)
These are a HubSpot **custom object** (type ID `2-21511658`). Mapped into `hubspot_outstanding_transactions`:

| HubSpot property   | Supabase column | Notes |
|--------------------|-----------------|-------|
| `payment_date`     | `date` + `days` | `days` = whole days between the payment date and today |
| `solution`         | `product`       | |
| `type`             | `type`          | |
| `status`           | `status`        | |
| `hubspot_owner_id` | `consultant`    | stored as the owner ID (resolved to a name in the UI) |
| `payment_amount`   | `deposit`       | |

For every transaction event, `syncParentDealContext()` finds the associated deal, re-fetches its latest `dealname` / `dealstage` / `amount`, mirrors that into `hubspot_outstanding_deals`, and stamps the parent `deal_id` onto the transaction row — so the deals table stays accurate even if a deal-level webhook was missed.

**Environment variables required:**

| Variable | Purpose |
|----------|---------|
| `SUPABASE_URL` | Target Supabase project URL |
| `SUPABASE_SERVICE_ROLE_KEY` | Service-role key (server-side writes) |
| `HUBSPOT_MCP_ACCESS_TOKEN` | HubSpot private-app / access token used to read deals & associations |

---

## Frontend — `index.html`

A single self-contained HTML page (all CSS + JS inline) — no build step. It uses the Supabase JS client from a CDN.

**Features:**
- **Magic-link auth** — user enters a corporate email and receives a sign-in link (Supabase OTP). Only whitelisted domains are allowed.
- **Summary stat cards** — Total Deals, Avg. Outstanding Days, Longest Pending, Total Collectables.
- **Data table** — Linked Deal Name, Consultant, Product, Total Deposit Sum, Remaining Collectables, First Deposit Date, Outstanding Days.
- **Search** by deal/client name.
- **Filters** by consultant, product, and age bucket (0–14 / 15–30 / 31+ days).
- **Sortable** columns and **pagination** (15 rows/page).
- **Colour-coded aging** — green (≤14d), orange (15–31d), red (31d+).
- **Dark/light theme** toggle (persisted in `localStorage`).

**Reads from:** the Supabase view `dashboard_outstanding_deals_summary`, selecting:
`Deal ID`, `Deal Name`, `Consultant ID`, `Product Name`, `Total Deposit Amount`, `Remaining Collectables`, `First Deposit Date`, `No. of Outstanding Days`.

**Key config (top of the `<script>` block):**

| Constant | What it controls |
|----------|------------------|
| `SUPABASE_URL` / `SUPABASE_ANON_KEY` | Which Supabase project the dashboard reads (anon key — safe for the browser; access is enforced by auth + RLS) |
| `APPROVED_DOMAINS` | Email domains allowed to request a login link |
| `CONSULTANT_MAP`   | Maps HubSpot owner IDs → human-readable consultant names |
| `PAGE_SIZE`        | Rows per page (default 15) |

---

## Data model (Supabase)

| Object | Type | Populated by |
|--------|------|--------------|
| `hubspot_outstanding_deals` | table | `index.ts` (deal events + parent-deal sync) |
| `hubspot_outstanding_transactions` | table | `index.ts` (transaction events) |
| `dashboard_outstanding_deals_summary` | view | SQL aggregation over the two tables above |

The summary view is what the dashboard queries. It is expected to expose the columns listed in the "Reads from" section — if you rename columns there, update the `.select(...)` string in `index.html` to match.

---

## Setup & deployment

**Backend (Edge Function):**
1. Set the three environment variables (above) as Edge Function secrets in your Supabase project.
2. Deploy `index.ts` as an Edge Function.
3. In HubSpot, create a webhook subscription pointing at the function URL, subscribed to: `deal.creation`, `deal.propertyChange`, `customObject.creation`, and `customObject.propertyChange` for the transaction custom object (`2-21511658`).

**Frontend (dashboard):**
1. Host `index.html` on any static host (currently served via **GitHub Pages**).
2. Make sure the redirect URL used for the magic link matches where the page is actually hosted (see `emailRedirectTo` in `handleLogin()`).
3. Add allowed users' email domains to `APPROVED_DOMAINS`, and set that same redirect URL as an allowed **Redirect URL** in Supabase Auth settings.

No build tooling is needed — both files are edited directly.

---

## Notes for collaborators / known gotchas

- **`APPROVED_DOMAINS` vs. the login placeholder don't match.** The input placeholder suggests `briancha.com / heroladies.com / mooofit.com`, but `APPROVED_DOMAINS` currently contains placeholder values (`yourcompany2.com`, `yourcompany3.com`). Update this list to the real domains before rollout.
- **`emailRedirectTo` is hard-coded** to `https://mooofit.github.io/hubspot-outstanding-deposits/`. If you move the dashboard, change it here **and** in Supabase Auth's allowed redirect URLs, or login links will break.
- **Consultant names are a static map.** New HubSpot owners show as `Deactivated/Removed` until added to `CONSULTANT_MAP`. Unassigned transactions show as `Unassigned`.
- **The anon key in `index.html` is public by design** — it only permits what your Supabase Row-Level Security policies allow. Do **not** put the service-role key in the frontend; it belongs only in the Edge Function's environment.
- **Small typo to clean up:** in the `.auth-input` CSS rule there's a stray `xssd` after `outline: none;`. Harmless, but worth removing.
- **The `days` figure is computed at write time** (when the webhook fires), while the dashboard's day buckets rely on the view. If you need live day counts, compute them in the view rather than trusting the stored `days` column.

---

## Want to collaborate?

Contributions welcome — a few good starting points:
- Wiring `CONSULTANT_MAP` to a live HubSpot owners lookup instead of a hard-coded map.
- Making `APPROVED_DOMAINS` / redirect URLs environment-driven instead of hard-coded.
- Adding CSV export or per-consultant summary views.
- Hardening the webhook (signature verification on incoming HubSpot requests).

If you'd like access to the Supabase project or HubSpot app to test end-to-end, reach out to the maintainer.
