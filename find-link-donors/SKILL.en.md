---
name: find-link-donors
description: Find link-building donor sites through the Collaborator.pro Public API based on user-provided criteria (DR, traffic trend and volume, language, country, link type, site age, price, site type, format). Returns paginated Markdown tables with summary stats, and a final CSV export. Trigger when the user says "find donors", "search Collaborator for donor sites", "give me a list of guest-post sites matching X criteria", "знайди донорів", "пошукай донорів на коллабораторі", "збери список донорів під критерії", or any variation of finding link prospects from Collaborator.pro.
---

# find-link-donors (EN)

Single-purpose skill: pull a list of candidate link-building donor sites from the **Collaborator.pro Public API** based on the user's criteria, then present them in paginated Markdown tables with summary stats and a final CSV export.

This skill does **not** depend on any other tool, CLI, or data source. Only the Collaborator.pro Public API + a local `.env` for the API key.

---

## TASK

Find link-building donor sites through the Collaborator.pro Public API.

## API

- **Base URL:** `https://collaborator.pro`
- **Endpoint:** `GET /api/public/creator/list`
- **Auth:** header `X-Api-Key: $COLLABORATOR_API`
- **Spec:** `https://collaborator.pro/es/api/public/default/schema` (OpenAPI 3.0)
- **Docs UI:** `https://collaborator.pro/api/public/default/docs`

## API KEY

Read `COLLABORATOR_API` from `/Users/andrei/PY/.env`.
If the file or the variable is missing — **ask the user**, do not guess the filename.

## IMPORTANT ACCESS RESTRICTION

The `/creator/list` endpoint is marked in the spec as **"available upon request from support"**.
If you get `401`/`403` — **do not loop**: stop and tell the user they need to request activation of the Creator API for their Collaborator account.

---

## USER INPUT

The user states criteria in free form. Typical set:

- minimum DR (Ahrefs)
- traffic trend (growing / stable / any)
- minimum organic traffic
- site language
- site country
- link type (dofollow / nofollow)
- minimum domain age
- maximum publication price
- site type (personal blog / corporate blog / mass media / online store / information site / portal / publisher)
- placement format (article, link insertion, etc.)

If a criterion is missing — **do not invent** it. Ask if needed.

---

## STEP 1 — RESOLVE DICTIONARIES (once, before the main request)

1. `GET /api/public/dictionary/languages` → find the ID for the language matching the user's input (e.g. "Ukrainian" / code `ua`).
2. `GET /api/public/dictionary/countries` → find the ID for the country.

**Show both IDs to the user** before launching the main request.

---

## STEP 2 — MAP USER CRITERIA → API PARAMETERS

| User criterion | API parameter |
|---|---|
| DR ≥ N | `ahrefs_dr_min=N` |
| Traffic growing | `is_traffic_grown=true` |
| Traffic ≥ N / month | `ahrefs_traffic_min=N` (absolute Ahrefs organic values; the `_traffic` parameter is in thousands and less reliable) |
| Language | `_language=<LANG_ID>` (from Step 1) |
| Country | `countries=<COUNTRY_ID>` (from Step 1) |
| Dofollow | `nofollow=0` (⚠️ `0 = Dofollow`) |
| Domain age ≥ N years | `_domain_age=N` |
| Price ≤ N USD | `_price_max=N` (+ verify currency in the response: `prices[].pricePublication` has a currency suffix; if not USD — filter client-side) |
| Site types | `_cre_type_id[]=<ID>` (repeats per type) |
| Format — article | `format_id=1` |

**Site-type reference (`_cre_type_id`):**
- `1` = Personal blog
- `2` = Corporate blog
- `3` = Mass media
- `4` = Online store
- `5` = Information site
- `6` = Portal
- `7` = Publisher

---

## STEP 3 — SORTING

The `sort` parameter exists but the spec **does not document its values**.
Try, in this order, until a request returns a correctly sorted batch:

1. `sort=-ahrefs_dr`
2. `sort=ahrefs_dr_desc`
3. `sort=-dr`

If none works — sort **client-side by DR (DESC)** after fetching the pages.

---

## STEP 4 — PAGINATION

The API restricts `per-page` to **[20, 40, 60, 100]** — `10` is not supported.

- Use `per-page=20`.
- **Display** results in chunks of **10 rows** per table.
- After every 10 — **pause and ask** "continue?".
- Use `pagination.totalCount` from the response to show progress:
  > "page X of Y, N total candidates, M remaining after client-side filters".

---

## STEP 5 — QUERY PLAN (SHOW BEFORE EXECUTING)

Before the first call to `/creator/list` — show the user:

- the full URL with all query parameters (**mask the API key as `***`**);
- which filters are applied server-side vs client-side;
- the expected price currency.

**Wait for an explicit `go`** from the user before the first request.

---

## STEP 6 — OUTPUT FORMAT

For every 10 valid donors — a Markdown table:

| # | Domain | Site Type | Country | Traffic | Site Age | Categories | Price | Link | Collaborator URL |
|---|--------|-----------|---------|---------|----------|------------|-------|------|------------------|

API response fields:

- **Domain** → `name`
- **Site Type** → `siteType`
- **Country** → `country`
- **Traffic** → `traffic` (string like `"353.1 k"`)
- **Site Age** → `siteAge`
- **Categories** → `categories` (truncate to the first 3)
- **Price** → `prices[0].pricePublication`
- **Link** → `prices[0].linkType` (highlight if ≠ `dofollow`)
- **Collaborator URL** → `url`

Do **not** output DR as a separate column — `CreatorFull` does not expose it as a field, only as a filter. If you want to surface it, note in the header: "DR ≥ N (per filter)".

**Under every table:**
- how many were dropped by client-side filters and why (topic / currency / dofollow / other);
- average price, median traffic;
- how many candidates remain in the pool (from `pagination.totalCount`).

---

## CONSTRAINTS & SAFETY

- **Rate limit:** pause `0.5–1` sec between pages.
- **5xx responses:** 2 retries with exponential backoff, then stop and report.
- **API key:** never print it in logs or in the query plan — mask as `***`.
- **Raw JSON of every page** must be saved to:
  `/Users/andrei/PY/collaborator-pro-skills/find-link-donors/output/donors_<YYYY-MM-DD>_p<N>.json`
  for downstream analysis.

---

## FINAL ARTIFACT

When the user says `stop` or the pool runs out — generate a consolidated CSV:

`/Users/andrei/PY/collaborator-pro-skills/find-link-donors/output/donors_<YYYY-MM-DD>.csv`

CSV columns:
- all columns from the table above;
- + a separate `prices_json` column containing the full JSON of the `prices` array.

Create the `output/` folder if it doesn't exist.

---

## WHAT THIS SKILL DOES NOT DO

- Does not run Python scripts, does not call any other CLI tool.
- Does not call any external API other than Collaborator.pro.
- Does not invent criteria the user did not state.
- Does not try to bypass `401`/`403` — it stops and reports.
