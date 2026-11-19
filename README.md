# China 250cc+ Motorcycle Brand Landscape

A fully static dashboard of the Chinese 250cc+ motorcycle market: 44 brands and 424 models on sale (plus 14 parallel imports, flagged separately and excluded from the totals). Cross-filter by segment / displacement band / body type / price band, search with multiple words, sort by price or displacement, and browse the brand wall, market overview, linked charts and data-scope notes.

The UI is a dark industrial theme with a light theme toggle, brand favourites (localStorage) and a right-hand brand drawer. The displacement, body-type and segment bar charts filter the model browser on click. The six price bands are bucketed by **entry price** and add up to exactly 438 (parallel imports included).

The **structural insights** section shows the same data three ways: the top 3 brands by model count in each body type (channel density), the top 5 brands by model count in each price band (price-band coverage), and a body-type × price-band matrix (11 rows × 6 columns, each cell holding the model count and shading by volume). Clicking a brand on a leaderboard or a non-empty cell jumps to the model browser with body type and price band applied; the matrix sums to 424, matching the statistical scope. The last row totals each price band, the rightmost column totals each body type; clicking a row total filters by body type, a column total by price band, and the bottom-right 424 clears everything. Hovering any cell lists the brands and model counts behind it. A row of 11 body-type tiles sits below the charts and filters on click. Price bands are horizontal chips with counts printed on them (1 / 47 / 80 / 82 / 182 / 46); clicking again clears. Search splits on spaces (AND across words) and highlights every hit. A zero-result search does not just say "nothing found": it computes, from live data, how many models each relaxation would return (clear search / drop body type / drop price band / drop displacement / include parallel imports), each one click away.

**Technical spec layer**: `data/specs.js` (`window.MOTO_SPECS`) is keyed by `brand|model` and covers **438 / 438 models (100%)**, about 30 fields per model — engine and drivetrain (power/rpm, torque/rpm, bore × stroke, compression, cooling, gearbox, clutch, final drive), dimensions and weight (L × W × H, seat height, wheelbase, tank, dry/kerb weight, ground clearance, frame), chassis and brakes (front/rear suspension and travel, discs and calipers, tyres and rims), and electronics (ABS, cornering ABS, TCS, quickshifter, cruise, TPMS, keyless, dash, top speed, fuel consumption). The compare drawer and the single-model drawer share the same `SPEC_GROUPS`; power-to-weight (power ÷ kerb weight) is computed live, best power/torque/tank and lowest kerb weight get an automatic "Best" tag (ties untagged), and missing fields show an em dash — never an estimate.

The first column of the model table is a checkbox: collect 2–3 models and a compare bar appears; "Start comparison" lines up list price, price nature (list vs entry), displacement, body type, price band, channel status, brand segment and per-row sources (plus the four spec groups when available). The lowest price and largest displacement are highlighted and tagged; a fourth model is refused. The header is sticky inside the container, and the checkbox and brand columns stay pinned during horizontal scroll. An A-Z index (by English brand initial, with counts per letter) sits above the brand wall; click to filter, click again to clear.

On narrow screens (≤760px) the filter block collapses to search + chips + result line, expanding to displacement / body type / price (sticky) only when opened, so the top half of a phone is not consumed by filters; price and body-type chips scroll horizontally on one line.

Every brand also has a static page at `brand/<id>.html` (no JS, dark theme) with a model table, price range, displacement range, doubtful-entry notes, same-category brand links and per-row sources, plus `Brand + ItemList(Product, offers) + BreadcrumbList` structured data. The home page carries `WebSite + Dataset`, and the scope section ends with a static index of all 44 brand pages. `sitemap.xml` covers the home page and all 44 brand pages.

The market section also carries a **250cc+ monthly output/sales trend**: January–September 2026 as paired monthly bars, with YoY on hover. The data comes from the China Chamber of Motorcycle Commerce monthly reports, fetched into `data/market_monthly.js` by `scripts/fetch_monthly.py`; re-run the script after each monthly release.

Live site: <https://zhouyundi2006.github.io/moto-dashboard/>

## Repository layout

```
index.html              single-page site (styles and logic inlined)
build_data.py           data build + validation: data/brands_*.json → data/dashboard.js / dashboard.json
build_site.py           assembles the publishable site: _site/index.html (version stamp injected) + data + assets + brand/*.html + sitemap.xml + robots.txt
data/brands_a~d.json    hand-maintained source data (the only source of truth; edit data here only)
data/dashboard.js       build output, loaded directly by the front end (window.MOTO_DATA)
data/dashboard.json     build output, same content, for humans and scripts
data/market_monthly.js  monthly output/sales series (generated by scripts/fetch_monthly.py, with per-month source links)
data/specs.js           technical spec library (window.MOTO_SPECS, 438 models covered)
data/logos.json         logo acquisition checklist
assets/logos/           localised official brand logos
assets/favicon.svg      site icon
assets/publish_api.py   fallback publishing channel (api.github.com when github.com is unreachable)
stats_server.py         optional: visit stats backend (Flask + SQLite)
scripts/                data-prep scripts (archive) + fetch_monthly.py (chamber monthly fetch)
.github/workflows/pages.yml  validate + publish to GitHub Pages
```

Model names stay in the form the manufacturer sells them; brand names, body types, segments and prices are in English.

## Data pipeline

Three steps: edit data → build → commit.

```bash
# 1. edit data/brands_*.json (the only files that need manual edits)
# 2. validate only, write nothing (CI runs this too; a failure blocks publishing)
python3 build_data.py --check
# 3. generate data/dashboard.js and data/dashboard.json
python3 build_data.py
```

`build_data.py` runs these checks and exits non-zero on any failure:

- brand `id` unique; `category` / `group` / `marketStatus` within the allowed values
- no duplicate model names within a brand
- `cc` between 245 and 3000 (bikes badged 250cc but measuring 246–249cc are in scope, hence the lower bound)
- `price` parses to a number, or is empty
- `type` non-empty, `source` required
- the file referenced by `logo` actually exists

It also derives `bucket` (displacement band), `priceMin` / `priceMax` / `priceFrom` / `priceUnit` (structured price) and per-dimension statistics.

## Data fields

Brand: `id` `name` `nameEn` `company` `category` `subCategory` `group` `marketStatus` `website` `logo` `logoDark` `models` `modelCount` `uncertain` `sources` `note`

Technical specs (`data/specs.js`, `models["brand|model"]`): `variant` (actual model year/trim the values were taken from) `price` `kw` `kwRpm` `nm` `nmRpm` `cc` `cyl` `cmp` `cool` `gear` `clutch` `drive` `frame` `lwh` `seat` `wb` `tank` `dry` `weight` `clearance` `suspF` `suspR` `brkF` `brkR` `tireF` `tireR` `abs` `tcs` `qs` `cruise` `tpms` `keyless` `dash` `src`; keys must match `dashboard.js` `brand name|model name` exactly, which `tools/check_specs.js` verifies at build time.

Model: `name` `cc` `type` `price` (display text as published) `priceMin` `priceMax` `priceFrom` (whether it is a "from" price) `priceUnit` (CNY) `source` `bucket`

About `marketStatus`:

- `official` — brands sold in mainland China through official channels; models count towards `stats.modelCount`
- `parallel` — only available as parallel imports (e.g. Royal Enfield); models are shown in a separate group and **excluded** from the statistics. `stats.modelCountAll` is the total including parallel imports.

## Monthly output/sales data

```bash
python3 scripts/fetch_monthly.py --check   # show what would be fetched, write nothing
python3 scripts/fetch_monthly.py           # fetch and overwrite data/market_monthly.js
```

The chamber publishes the previous month's report in the middle of each month; the script locates the data month as "publication month − 1" and only takes the sentence in the body beginning "（排量 250ml/250cc 以上，不含）… 产销 X 万辆和 Y 万辆 … 同比 A% 和 B%", keeping the original link on every record. Three self-checks are built in: months must be consecutive; month-on-month change may not exceed 80%; a single month above 200k units is treated as a cumulative figure leaking in and dropped (a 2023-07 "1–6 月 30.18 万辆" was caught this way); once all 12 months of a year are present, the sum is reconciled against the chamber's annual figure (2025: 95.37 / 95.23 万辆) and warns beyond a 0.6 万辆 gap. The chamber site is unreliable from overseas IPs, so fetching is not part of CI — it runs locally and only the data file is committed.

Defaults are `--pages 4`, `--years 1`, i.e. the current year only. For a cross-year view use `--years 2`, but note that the May and June 2025 releases carry identical numbers (10.05 / 10.12 万辆) while reporting −17.75% and +4.68% month-on-month respectively — mutually contradictory. Laying them end to end would turn a typo into a trend, so the chart stays within the current year and the annual comparison sits in the KPI at the top right.

## Deployment

Pushing to `main` triggers the GitHub Pages workflow. It first runs `build_data.py --check`, then `build_site.py` assembles `_site` (`index.html` with `?v=<UTC time>` version stamps on the two data files, 44 brand pages, `sitemap.xml`, `robots.txt`) and asserts the output is complete (home-page stamps, brand-page count, brand index substituted, monthly series month count); any failure stops the release. The raw research JSON and collection scripts are never published with the site.

## Visit stats backend (optional)

The front end ships `STATS_API = ""`; with no backend the counter block stays hidden — a purely static site cannot count visitors anyway. When needed, deploy `stats_server.py` separately:

```bash
pip install -r requirements.txt
ADMIN_TOKEN=$(openssl rand -hex 16) \
ALLOW_ORIGIN=https://zhouyundi2006.github.io \
PORT=5000 python stats_server.py
```

| Variable | Meaning |
| --- | --- |
| `ADMIN_TOKEN` | required. Password for `/admin`; the server refuses to start without one (no default password) |
| `ALLOW_ORIGIN` | front-end origin. If empty, **no** CORS header is sent and browsers block cross-origin reporting |
| `STATS_DB` | SQLite path, default `stats.db` |
| `TRUST_PROXY` | set to 1 to trust `X-Forwarded-For` unconditionally (by default only from private/loopback peers) |
| `GEO_ENABLE` | set to 0 to disable geo lookup (on by default, which sends the visitor IP to ip-api.com over plain http) |

Endpoints:

- `POST /api/visit` — returns immediately after the write; geo is filled in by a background thread (one record per IP per 60s)
- `GET /api/stats` — `{pv, uv24, uv_all}`
- `GET /admin` — visitor detail, needs the `X-Admin-Token` header or `?token=`

To wire the front end, point `STATS_API` at the backend; it **must** be https or mixed-content blocking applies.

## Known trade-offs

- The monthly series covers "January of the current year to date" only. The May and June 2025 chamber releases repeat the same 250cc+ figures and contradict their own month-on-month numbers, so stitching a longer series would turn a source error into a trend. A cross-year comparison would need an independent monthly source that can be cross-checked; broker research only publishes quarterly or cumulative figures so far.
- Brand `note` fields hold raw research logs (including working notes such as "official site returned 403") and the English build omits them rather than translating research shorthand; the underlying detail stays in the Chinese source data.
