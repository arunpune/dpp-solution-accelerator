# DPP Solution Accelerator — Implementation Guide for New Programmers

This guide explains **what the codebase does, how its pieces fit together, and how to turn it from a
demo into a real Digital Product Passport (DPP) system** for an actual manufacturer.

The project currently maintains two separate deployments: the original **Battery** deployment and a
separate **Furniture** deployment. The Furniture deployment reuses the accelerator architecture but
has its own Unity Catalog catalog, Lakebase instance, bundle target, dashboard and Supplier Portal.
Keep these deployments isolated; in particular, never run the Furniture setup job against the Battery
Lakebase instance.

* Want a 5-minute overview or the deploy commands? Read [README.md](README.md).
* Want to understand the code and take it to production? Read this file.

> \*\*Honesty note.\*\* This repository is a \*reference architecture with synthetic data\*, not a
> turnkey compliance product (the README says so too). Part 9 lists the gaps I found while reading
> the code. Read it before you promise anything to a customer.

\---


## Current project deployment status (verified 12 November 2026)

This section records the actual Databricks deployment state. It is separate from the general
architecture and implementation guidance elsewhere in this document.

### Deployment separation

| Item | Battery (existing) | Furniture (separate deployment) |
|---|---|---|
| Repository | `dpp-solution-accelerator-main` | `dpp-solution-accelerator-furniture` |
| Bundle target | Existing Battery target | `furniture` |
| Bundle name | Existing Battery bundle | `dpp-furniture-accelerator` |
| Unity Catalog catalog/schema | `dpp_dev.dpp` | `dpp_furniture.dpp` |
| Lakebase instance/project | `dpp-passport` | `dpp-furniture-passport` |
| Lakebase host | `ep-still-mountain-d8r9kcmo.database.us-east-2.cloud.databricks.com` | `ep-lively-thunder-d8nhie5f.database.us-east-2.cloud.databricks.com` |

**Important isolation rule:** the Furniture setup job targets `dpp-furniture-passport` and the
`furniture` profile. It truncates and reseeds its target database. Do not change its instance
parameter to `dpp-passport`, and do not run the Furniture setup job against Battery data.

### Furniture deployment configuration

- Databricks CLI profile: `dpp-dev`
- Workspace: `https://dbc-7a77070e-e976.cloud.databricks.com`
- Workspace ID: `7474659510807566`
- Bundle target: `furniture`
- Bundle name: `dpp-furniture-accelerator`
- Bundle workspace path:
  `/Workspace/Users/mahajan_neel241@vit.edu/.bundle/dpp-furniture-accelerator/furniture`
- Unity Catalog: `dpp_furniture.dpp`
- Lakebase instance: `dpp-furniture-passport`
- Lakebase database: `databricks_postgres`
- Region: `us-east-2`

The Furniture bundle validates successfully with:

```powershell
databricks bundle validate -t furniture --profile dpp-dev
```

The pipeline configuration must not contain unsupported `mode:` or `development:` fields in
`resources/dpp_pipelines.yml` for the CLI/bundle version used in this workspace.

### Furniture data and pipeline verification

The Furniture synthetic setup job is:

- Job ID: `430900565120558`
- Job name: `[dev mahajan_neel241] DPP Furniture - Synthetic Data Setup`
- Entry point: `src/foundation/00_setup_lakebase.py`
- Parameters: instance `dpp-furniture-passport`, database `databricks_postgres`, profile `furniture`

A setup run completed successfully. The Furniture ingestion pipeline also completed successfully:

- Pipeline ID: `ea1322fa-6c7b-4834-ad53-339f962a8bc8`
- Bronze, Silver and Gold flows completed.
- Verified row counts: 500 in `bronze_product_passport`, 500 in
  `silver_product_passport`, 50 in `silver_supplier`, 500 in `gold_passport_complete`, and
  500 in `gold_supply_chain_summary`.
- `gold_supply_chain_lineage` and `gold_app_usage` were also confirmed to exist.

The AI insights precomputation job completed successfully:

- Job ID: `586587853091817`
- Successful run ID: `488542139039746`
- Source: `dpp_furniture.dpp.gold_passport_complete`
- Delta output: `dpp_furniture.dpp.dpp_passport_insights`
- Lakebase cache: `dpp.passport_insights`
- 500 product insight records were upserted.

### Furniture dashboard

The Furniture compliance dashboard is deployed, active, published and was opened successfully
in the browser.

- Dashboard ID: `01f1de63a90811059cc399aa0795cf3f`
- Dashboard title: `DPP Furniture Compliance & Sustainability`
- Warehouse ID: `c07d5aecc4629357`
- Definition: `resources/dashboards/dpp_compliance.lvdash.json`
- Resource declaration: `resources/dpp_dashboards.yml`
- Five pages and 19 datasets.
- Pages: Portfolio Overview, Compliance & Risk, Environmental Impact, Supply Chain Risk, and
  App Usage.
- Dashboard queries use the Furniture catalog, `dpp_furniture.dpp`.

Dashboard URL:
https://dbc-7a77070e-e976.cloud.databricks.com/dashboardsv3/01f1de63a90811059cc399aa0795cf3f/published?w=7474659510807566

### Furniture Supplier Portal

The Furniture Supplier Portal is deployed and currently working. Its supplier list is displaying
successfully after the Furniture Lakebase endpoint was re-enabled.

- App name: `dpp-furniture-supplier-portal`
- App ID: `9f86dd57-60dd-40d7-b3a1-ae6b265f6df6`
- Deployment ID: `01f1de64f2551847b67a500a69810f03`
- App status: `RUNNING`
- Compute status: `ACTIVE`
- Latest deployment status: `SUCCEEDED`
- Source path:
  `/Workspace/Users/mahajan_neel241@vit.edu/.bundle/dpp-furniture-accelerator/furniture/files/src/apps/supplier_portal`
- Lakebase resource binding: `dpp-furniture-passport`, database `databricks_postgres`

Supplier Portal URL:
https://dpp-furniture-supplier-portal-7474659510807566.aws.databricksapps.com

**Lakebase endpoint incident and resolution:** the Furniture endpoint
`projects/dpp-furniture-passport/branches/production/endpoints/primary` had
`spec.disabled=true`, which caused `asyncpg.exceptions.InvalidAuthorizationSpecificationError:
The endpoint has been disabled`. The endpoint was updated with `spec.disabled=false`; its status
then showed `disabled=false` and `current_state=ACTIVE`. The Supplier Portal supplier list now
loads. This was a Furniture-only change; the Battery endpoint was not modified.

### Remaining deployment work

- The Furniture Passport Viewer has **not** been deployed yet.
- The workspace currently has a maximum of three Databricks Apps. The three existing apps are
  `dpp-furniture-supplier-portal`, `dpp-passport-viewer` (Battery), and
  `dpp-supplier-portal` (Battery).
- Do not delete either Battery app to make room. Request an app quota increase or use another
  approved workspace before deploying the Furniture Passport Viewer.
- The original Battery deployment remains separate and must be preserved.

---

## Table of contents

0. [Current project deployment status](#current-project-deployment-status-verified-12-november-2026)
1. [The problem in plain language](#1-the-problem-in-plain-language)
2. [Concepts you need first](#2-concepts-you-need-first)
3. [How the system works (big picture)](#3-how-the-system-works-big-picture)
4. [Code tour: what every folder and file does](#4-code-tour-what-every-folder-and-file-does)
5. [The data model](#5-the-data-model)
6. [Step 0 — Run the demo end to end](#6-step-0--run-the-demo-end-to-end)
7. [Implementing this for a real company (the main roadmap)](#7-implementing-this-for-a-real-company)
8. [Day-to-day developer tasks (recipes)](#8-day-to-day-developer-tasks-recipes)
9. [Known gaps and traps in the current code](#9-known-gaps-and-traps-in-the-current-code)
10. [Production readiness checklist](#10-production-readiness-checklist)
11. [Troubleshooting](#11-troubleshooting)
12. [Glossary](#12-glossary)

\---

## 1\. The problem in plain language

The EU now requires many products to carry a **Digital Product Passport**: a QR code on the product
that leads to a web page/API with the product's origin, materials, carbon footprint, compliance
certificates, repairability/recycling info and disposal instructions. Batteries come first
(mandatory **18 Feb 2027** per this repo's docs); other product groups follow through the decade.
Dates move, so always confirm against the legal act for your product group.

The key design fact (see [docs/positioning.md](docs/positioning.md)): the EU does **not** store your
passport data. A thin EU registry points to it. **You (the "economic operator": manufacturer,
importer) must run the backend that stores and serves the passports.** This repo is a blueprint
for that backend on Databricks.

What the system has to do, in programmer terms:

|Need|What it means technically|Where it lives here|
|-|-|-|
|Fast lookup when someone scans a QR|Low-latency read of one product's full record|Lakebase (Postgres) + `passport\_viewer` app|
|Suppliers keep their own data current|Transactional writes + an audit trail|`supplier\_portal` app + Lakebase triggers|
|Prove compliance across thousands of products|Batch analytics, data quality checks|Bronze/Silver/Gold pipeline + dashboard|
|Ask questions in English|Natural-language SQL|Genie space (spec only)|
|Explain gaps per product|LLM text generation, cached|`precompute\_insights` job|

\---

## 2\. Concepts you need first

You do not need to be a Databricks expert, but you must understand these ten ideas. Everything else in
the repo is built from them.

1. **Databricks workspace** — your company's Databricks account environment (web UI + APIs).
2. **Unity Catalog (UC)** — Databricks' governance layer. Data is addressed as
`catalog.schema.table`, e.g. `dpp\_dev.dpp.gold\_passport\_complete`.
3. **Lakebase** — a managed **PostgreSQL** database inside Databricks. Use it for *operational*
(OLTP) work: "give me row X in 5 ms", "update this certificate". The repo uses
**Lakebase Provisioned**. This project uses separate instances for the Battery (`dpp-passport`)
and Furniture (`dpp-furniture-passport`) deployments.
4. **Lakehouse tables (Delta)** — columnar analytics tables in UC. Good for scanning millions of
rows, bad for single-row lookups. Spark reads/writes them.
5. **Medallion layers** — a naming convention for the analytics pipeline:
**Bronze** = raw copy, **Silver** = cleaned and validated, **Gold** = business-ready aggregates.
6. **Spark Declarative Pipelines (SDP)** — formerly Delta Live Tables. You *declare* tables as
Python functions decorated with `@dlt.table`; the platform works out dependency order and runs
them. Data-quality rules are `@dlt.expect...` decorators.
7. **Databricks Apps** — hosts a web app (here, FastAPI) inside your workspace, behind Databricks
login (SSO), with its own **service principal** identity.
8. **Service principal (SP)** — a non-human identity. Each app and each production job runs as one.
9. **Databricks Asset Bundles (DABs)** — "infrastructure as code" for Databricks. `databricks.yml`
and `resources/\*.yml` declare jobs, pipelines, apps and the dashboard; `databricks bundle deploy`
creates them all.
10. **OAuth short-lived tokens instead of passwords** — nothing in this repo stores a database
password. Code asks the Databricks API for a token that expires in about an hour (see
`\_resolve\_runtime\_lakebase` in the foundation scripts and `\_lakebase\_oauth\_token` in the apps).

\---

## 3\. How the system works (big picture)

```
                    WRITE SIDE (operational, "system of record")
 ┌──────────────────┐   INSERT/UPDATE   ┌───────────────────────────────┐
 │ setup job        │ ────────────────► │  LAKEBASE (Postgres, schema   │
 │ (synthetic data) │                   │  `dpp`, 12 tables)            │
 └──────────────────┘                   │  + audit trigger              │
 ┌──────────────────┐   UPDATE (cert    │  + passport\_insights cache    │
 │ Supplier Portal  │ ────────────────► │                               │
 │ (FastAPI app)    │   renewal)        └──────┬───────────────▲────────┘
 └──────────────────┘                          │ SELECT        │ upsert insights
                                               │               │
 ┌──────────────────┐   SELECT (asyncpg)       │        ┌──────┴─────────────┐
 │ Passport Viewer  │ ◄────────────────────────┘        │ precompute\_insights│
 │ (FastAPI app)    │   QR → /passport/{id}             │ job (ai\_query)     │
 └──────────────────┘                                   └──────▲─────────────┘
                                                               │ reads Gold
                    ANALYTICS SIDE (derived, rebuildable)      │
 Lakebase ──JDBC snapshot──► BRONZE ──► SILVER ──► GOLD ───────┘
   (full table read per run)   raw      clean+DQ    10 business tables
                                                      │
                                         ┌────────────┴─────────┐
                                         ▼                      ▼
                                  AI/BI Dashboard         Genie space
                                  (5 pages)               (manual setup)
```

**The most important architectural idea:** *Lakebase is the source of truth; the lakehouse is a
derived copy.* Data flows **Lakebase → Bronze**, not the other way round. Apps read and write Lakebase
directly; analytics never write back (except the AI-insights cache). If you remember one thing from
this guide, remember that direction, because it determines how you plug in real data (Part 7).

### Runtime sequence for each scenario

**A. A consumer scans a QR code**

1. QR encodes `https://<viewer-app-url>/passport/<passport\_id>` (built in `/api/qr/{id}`).
2. `GET /passport/{id}` → `\_get\_full\_passport()` runs several SQL queries against Lakebase (passport

   * manufacturer, origins + suppliers, materials, impact, compliance, circularity, disposal).
3. A recursive SQL query (`\_attach\_supplier\_chains`) follows `parent\_supplier\_id` to show the
upstream supplier chain per component.
4. Jinja2 renders `passport.html` with 7 tabs. The AI Insights tab calls `/api/insights/{id}`
asynchronously (see D).

**B. A supplier renews a certificate**

1. `GET /supplier/{id}/renew/{origin\_id}` shows a pre-filled form.
2. `POST` validates the origin belongs to that supplier and the date parses, then runs two `UPDATE`s
in **one transaction** (`product\_origin` and `supplier.updated\_at`).
3. A Postgres **trigger** (`dpp.audit\_trigger\_func`) writes old/new row JSON into
`dpp.passport\_audit\_log` automatically — the app code never writes the audit row.
4. Next pipeline run propagates the change to Silver/Gold; the dashboard updates.

**C. Nightly analytics**

1. `dpp\_ingestion\_pipeline` reads every Lakebase table over JDBC into Bronze (full snapshot),
cleans into Silver (with expectations), builds Gold.
2. `precompute\_insights` job calls `ai\_query` over `gold\_passport\_complete` for every product,
writes `gold.dpp\_passport\_insights` (Delta) and upserts JSON into Lakebase
`dpp.passport\_insights`.

**D. AI Insights tab**
`/api/insights/{id}` → (1) look in Lakebase cache; (2) if missing, call the Foundation Model
serving endpoint live; (3) if that fails, return a friendly `pending` message. The page never errors
because of AI.

\---

## 4\. Code tour: what every folder and file does

```
databricks.yml                 Bundle root: variables + dev/staging/prod targets
resources/
  apps.yml                     Declares the 2 apps and binds Lakebase to each as a resource
  dpp\_jobs.yml                 Jobs: synthetic\_data\_setup, precompute\_insights
  dpp\_pipelines.yml            The SDP pipeline and the 8 files it contains
  dpp\_dashboards.yml           AI/BI dashboard resource
  dashboards/\*.lvdash.json     Dashboard definition (5 pages, 19 datasets)
  genie/dpp\_compliance\_explorer.yml   Genie space \*spec\* (not auto-deployed)
src/
  foundation/                  Lakebase setup + synthetic data
  pipelines/{bronze,silver,gold}/   SDP code (+ precompute job in gold/)
  apps/passport\_viewer/        Public-facing app
  apps/supplier\_portal/        Supplier-facing app
  serving/reyden\_toggle.py     OPTIONAL reference code, not used by anything
tests/test\_synthetic\_data.py   42 pytest tests for the generator
docs/                          Positioning, architecture diagrams
ROADMAP.md                     Deliberately-missing v2 features
```

### 4.1 `src/foundation/` — building the operational store

|File|Role|Read this when…|
|-|-|-|
|`01\_setup\_lakebase\_schema.sql`|Idempotent DDL: 12 tables, indexes, audit trigger, grants|You change the data model|
|`02\_synthetic\_data\_generator.py`|`DPPDataGenerator` class + `PROFILES` dict (battery, furniture)|You add an industry or need test data|
|`03\_seed\_lakebase.py`|asyncpg helpers: `\_connect()`, `insert\_table()`, `truncate\_tables()`, column lists in `TABLE\_COLUMNS`|You load data into Lakebase|
|`00\_setup\_lakebase.py`|The job entry point: split SQL → apply DDL → generate in memory → truncate → insert|You wonder what the setup job does|

Things worth knowing:

* **Numeric prefixes** (`00\_`, `01\_`) are not importable Python names, so `00\_setup\_lakebase.py` loads
its siblings with `importlib.util.spec\_from\_file\_location`.
* **Why generate in memory?** Serverless job tasks do not share a filesystem, so a
"generate JSON file → seed JSON file" two-task design cannot work. One task does everything.
* **`split\_sql()`** is a small hand-written parser that splits the DDL on `;` while respecting `$$ ... $$`
function bodies and `--` comments, because asyncpg executes one statement at a time.
* **`run\_async()`** runs the coroutine in a fresh thread/event loop because Databricks serverless
already has a running loop, which makes `asyncio.run()` fail.
* **Text codecs** (`\_init\_codecs`) let asyncpg accept ISO-date strings and UUID strings as-is.
* **Determinism**: `SEED = 42` and a fixed `REFERENCE\_DATE = 2026-03-15`, so every run produces the
same 500 products / 50 suppliers (that is why the tests can assert exact properties).

### 4.2 `src/pipelines/` — Bronze → Silver → Gold

|Layer|File(s)|What happens|
|-|-|-|
|Bronze|`ingest\_lakebase\_to\_bronze.py`|Gets a Lakebase OAuth token, then reads each `dpp.\*` table via JDBC and adds `\_ingested\_at` and `\_source\_table`. 11 `bronze\_\*` tables (includes `app\_access\_log`).|
|Silver|`clean\_and\_conform.py`|Trims/uppercases/lowercases, and applies expectations. `@dlt.expect\_or\_drop` removes bad rows (null IDs, wrong country code length, etc.); plain `@dlt.expect` only records violations. Adds helper flags: `is\_complete`, `certification\_expiring\_soon`, `is\_expired`, `needs\_verification`.|
|Gold|`passport\_complete.py`|One wide row per passport (materials/compliance aggregated into arrays). Feeds AI and apps/analytics.|
|Gold|`supply\_chain\_lineage.py`|`gold\_supply\_chain\_lineage`, `gold\_supply\_chain\_summary`, `gold\_supplier\_traceability` (flattens ancestry with a bounded 2-hop join).|
|Gold|`compliance\_gap\_analysis.py`|Risk score (0-100) and category CRITICAL/HIGH/MEDIUM/LOW per product. Weights: non-compliant ×30, pending ×15, expired ×20, expiring supplier cert ×10, incomplete +15, unverified LCA +10.|
|Gold|`carbon\_footprint\_aggregates.py`, `material\_circularity\_metrics.py`|`gold\_carbon\_by\_category`, `gold\_carbon\_by\_product`, `gold\_material\_composition\_trends`, `gold\_circularity\_metrics`.|
|Gold|`app\_usage.py`|Cleans the opt-in access log for the dashboard's "App Usage" page.|
|Job|`precompute\_insights.py`|**Not part of the pipeline.** A job that runs `ai\_query` with a JSON-schema response format, writes a Delta table, then upserts into Lakebase.|

Important detail: the bronze functions are decorated `@dlt.table` but read with batch
`spark.read.format("jdbc")`, so each run **re-reads the whole table** (a snapshot), regardless of the
word "streaming" in comments. That is fine for 500 products and a cost problem at millions (Part 7.6).

### 4.3 `src/apps/` — the two web apps

Both are FastAPI + Jinja2 + asyncpg and share \~200 lines of copy-pasted database plumbing (the roadmap
lists de-duplicating this).

**Passport Viewer** (`passport\_viewer/main.py`)

|Route|Purpose|
|-|-|
|`GET /`|Searchable, filterable, paginated passport list (page size 24)|
|`GET /passport/{id}`|The passport (7 tabs)|
|`GET /api/passport/{id}`|Same data as JSON, for system integration|
|`GET /api/qr/{id}`|PNG QR code; builds the URL from the `X-Forwarded-Host` header|
|`GET /api/insights/{id}`|Cached → live LLM → pending|
|`GET /demo`|"How it's built" explainer page|
|`GET /health`|Returns 503 if the DB is unreachable|

**Supplier Portal** (`supplier\_portal/main.py`): `/` supplier list, `/supplier/{id}` dashboard,
`/supplier/{id}/renew/{origin\_id}` (GET form, POST write), `/supplier/{id}/audit`, `/health`.

**How the apps authenticate to Postgres** (the part new developers find confusing):

1. `resources/apps.yml` binds the Lakebase instance as an app *resource*.
2. Databricks creates a Postgres role for the app's service principal and injects `PGHOST`, `PGPORT`,
`PGDATABASE`, `PGUSER` — but **no password**.
3. The app passes a *callable* as asyncpg's `password` (`\_oauth\_password`). asyncpg calls it every time
the pool opens a connection, which mints a fresh OAuth token. A long-lived pool therefore never
holds an expired token.
4. If `PGHOST` is absent (local dev) it falls back to `LAKEBASE\_USER` + `LAKEBASE\_PASSWORD`.

### 4.4 Config files you will touch most

* `databricks.yml` variables: `catalog`, `schema`, `profile`, `lakebase\_instance\_name`,
`lakebase\_database`, `dashboard\_warehouse\_id`, `prod\_sp\_name`. Targets: `dev`, `staging`
(both `mode: development`), `prod` (`mode: production`, `run\_as` a service principal).
* `src/apps/\*/app.yaml`: the app start command and env vars (`LAKEBASE\_INSTANCE`, `DPP\_SCHEMA`,
`AI\_MODEL`, `ACCESS\_LOG\_ENABLED`). **DABs does not substitute `${var.\*}` inside `app.yaml` or
`.lvdash.json`** — if you rename the instance/catalog you must edit those files by hand.

\---

## 5\. The data model

Schema `dpp` in Lakebase (12 tables). Think of `product\_passport` as the hub:

```
manufacturer 1───\* product\_passport 1───\* product\_origin \*───1 supplier ◄─┐ parent\_supplier\_id
                          │                                               └─(self-reference, tiers 1-3)
                          ├──\* product\_materials         (bill of materials, % by weight, REACH/SVHC flags)
                          ├──\* environmental\_impact      (carbon by lifecycle phase, energy, water, LCA method)
                          ├──\* compliance\_records        (regulation, status, certificate, expiry)
                          ├──\* circularity\_info          (repairability, recycled %, SoH/cycle count)
                          ├──\* disposal\_guidelines       (per component)
                          ├──1 passport\_insights         (cached AI JSON)
                          └──\* passport\_audit\_log        (filled by trigger)
app\_access\_log                                           (opt-in usage tracking, no FK)
```

Rules enforced by the database itself (so your ingestion code cannot violate them):

* `passport\_status` ∈ `draft | active | expired | revoked`
* `compliance\_status` ∈ `compliant | pending | non\_compliant | expired`
* `supplier.tier` 1–3; `risk\_score` 0–10; percentages 0–100
* `product\_id` is `UNIQUE` (the business key); `passport\_id` is a UUID (the technical key)
* Audit triggers exist on 7 tables: passport, origin, materials, impact, compliance, circularity, disposal.

The six EU "categories" map to tables: origin → `product\_origin`/`supplier`; materials →
`product\_materials`; environmental impact → `environmental\_impact`; compliance →
`compliance\_records`; circularity → `circularity\_info`; disposal → `disposal\_guidelines`.
This is why the schema is "domain-agnostic": a steel coil and a battery pack both have those six things.

**Industry profiles.** `PROFILES` in the generator is a dict of dicts. A profile supplies the
manufacturer, product catalog, materials, certifications, carbon ranges, regulations, facilities, etc.
The `battery` profile additionally sets `phased\_regulations` (Art. 7/8/48 items always `pending`),
`tier\_components` (mine → cell → module → pack naming) and `has\_dynamic\_data` (state of health for
rechargeable categories only). Only the *seed data* is industry-specific; schema, pipeline and apps
are not.

\---

## 6\. Step 0 — Run the demo end to end

Do this before changing anything. It proves your environment works and teaches you the moving parts.

### 6.1 Zero-cost local check (no Databricks needed)

```bash
python -m venv .venv \&\& source .venv/bin/activate
pip install faker pytest
pytest tests -q                       # expect: 42 passed
python src/foundation/02\_synthetic\_data\_generator.py   # writes JSON per table to src/foundation/data/
```

Open a couple of the JSON files to see what a "passport" looks like as data.

### 6.2 Deploy to a workspace

Prerequisites: Databricks CLI ≥ 0.278 (check `databricks version`), a workspace with Lakebase and
Unity Catalog, permission to create a catalog (or use an existing one), and a SQL warehouse.

```bash
databricks auth login --host https://<workspace>.cloud.databricks.com --profile dpp
export DATABRICKS\_CONFIG\_PROFILE=dpp

# 1. Lakebase instance (created outside the bundle)
databricks database create-database-instance dpp-passport --capacity CU\_2

# 2. Deploy jobs, pipeline, dashboard, apps
databricks bundle deploy -t dev --var="dashboard\_warehouse\_id=<warehouse-id>"

# 3. Order matters: schema+data -> pipeline -> apps -> AI cache
databricks bundle run synthetic\_data\_setup -t dev
databricks bundle run dpp\_ingestion\_pipeline -t dev
databricks bundle run dpp\_passport\_viewer -t dev
databricks bundle run dpp\_supplier\_portal -t dev
databricks bundle run precompute\_insights -t dev    # optional
databricks apps list                                # find the app URLs
```

Why that order: the pipeline reads tables the setup job creates; the apps need the schema; the AI job
needs the Gold layer.

### 6.3 Verify

* Open the viewer URL → you should see \~500 battery passports (profile `battery` is passed by the bundle).
* Open one → click through all 7 tabs. AI Insights shows a "pending" note until `precompute\_insights` has run.
* Open the supplier portal → pick a supplier → renew a certificate → open its audit log and confirm
an `UPDATE` row appeared (this proves the trigger works).
* Re-run the pipeline → open the dashboard → the change appears.

\---

## 7\. Implementing this for a real company

The demo invents its own data. A real project is mostly about **replacing the data source, hardening
access, and meeting regulatory identity requirements**. Work in these phases; each one ends with a
checkable result.

### Phase 1 — Scope and decisions (1–2 weeks, mostly not coding)

Answer these in writing *before* you write code; the answers change the design.

1. **Which product group and which delegated act?** (battery now; steel, textiles, etc. later.) Get
the list of mandatory data attributes from the act, not from this repo.
2. **Granularity**: model-level, batch-level or item-level passports? The schema supports one passport
per `product\_id` (unique) with optional `serial\_number`, `batch\_lot\_number`. Batteries require
item-level for some categories — that implies a *lot* of rows (millions), which affects Part 7.6.
3. **Who owns each data element?** Build a table: attribute → source system → owner → refresh
frequency. Typical sources:

|Passport section|Typical real source|
|-|-|
|Product identity, GTIN, serial, batch, facility|ERP / MES (e.g. SAP)|
|Materials / BOM, hazardous substances|PLM, supplier material declarations|
|Carbon footprint, LCA|LCA tool, third-party verified reports|
|Certificates, compliance|QMS, notified bodies, supplier portal|
|Supplier hierarchy|Procurement / supplier master data|
|State of health, cycle count|Battery management system telemetry|

4. **Access tiers**: which fields are public, which need "legitimate interest", which are authority-only.
The current code serves everything publicly (Part 9).
5. **Environments and ownership**: who runs production, who approves data changes, who is the named
responsible economic operator?
6. **Data residency / cross-border rules** and where the workspace must live.

**Done when:** you have the attribute-to-source table and a signed-off access-tier list.

### Phase 2 — Stand up a clean environment (days)

1. Create **dev / staging / prod** workspaces or at least separate catalogs (`dpp\_dev`, `dpp\_staging`,
`dpp` are the defaults in `databricks.yml`).
2. Create a **service principal** for production (`dpp-prod-sp` by default, variable `prod\_sp\_name`).
The `prod` target runs everything `run\_as` that SP. Give it: `CREATE`/`USE` on the target catalog,
permission to create the pipeline/jobs/apps, and a Postgres role on the Lakebase instance (the
Bronze ingestion authenticates as the identity running the pipeline and needs `SELECT` on `dpp.\*`).
3. Create one Lakebase instance per environment; size it above `CU\_2` for production, and read
Databricks' current docs on HA/backup/retention for Lakebase before go-live.
4. Fork the repo into your own Git organisation. Replace `.github/workflows/databricks-ci.yml`: it
currently hardcodes a Databricks-internal workspace URL and a custom runner label
(`html\_publisher`); both will fail for you. Use OIDC/federated or service-principal auth rather than a
personal access token.
5. Run the demo (Part 6) in **dev** unchanged, to confirm the platform side works.

**Done when:** the unmodified demo runs in dev under your tenant.

### Phase 3 — Map your product data onto the schema (1–2 weeks)

Do this **on paper/SQL first**.

1. For each of the 10 data tables, list every column and decide: mapped from source X, derived, or
unavailable. Look for *mandatory attributes the schema lacks* (battery Annex XIII technical
parameters such as rated capacity, voltage, expected cycles are listed in ROADMAP item 3).
2. Choose stable **business keys**. `product\_id` must be unique forever and ideally never reused. Decide
how you will mint `passport\_id` — the demo uses random UUIDs; for real use you will probably derive or
register an identifier (Phase 6).
3. Extend the schema **additively**: write `ALTER TABLE ... ADD COLUMN IF NOT EXISTS ...` (the existing
DDL already does this for the state-of-health columns). Never rename or drop columns in place on a
live passport system.
4. If you add a table or column, follow the **change checklist** in Part 8.1 — a new column has to be
threaded through the seeder column list, Silver select list, Gold, apps and tests.

**Done when:** a reviewed mapping document exists and the DDL changes are merged.

### Phase 4 — Replace the synthetic data with real ingestion (the core work, 2–6 weeks)

The demo's `00\_setup\_lakebase.py` does `generate → TRUNCATE ALL → insert`. **Never run that against
real data** (Part 9, gap 1). You need a *loader* that upserts.

**Design decision — where is the system of record?**

|Option|How|Good when|Watch out|
|-|-|-|-|
|**A. Write directly into Lakebase** (matches this repo)|Small ingestion services/jobs read ERP/PLM/etc. and `INSERT ... ON CONFLICT DO UPDATE` into `dpp.\*`|Source systems are few; you want the audit triggers to see every change; apps need fresh data|You own validation and idempotency|
|**B. Land in the lakehouse first, then sync to Lakebase**|Ingest sources into Delta (Auto Loader / Lakeflow Connect / partner tools), clean there, then use Databricks' Lakebase sync / reverse-ETL capability to publish serving tables|Many sources, heavy transformations, data engineers already own the lakehouse|The pipeline direction becomes *reversed* from this repo; `bronze\_\*` ingestion from Lakebase and the Supplier Portal's writes need a rethink. Check current sync feature limits in the docs.|

For most first projects, start with **A** — it keeps this repo's pipeline and apps valid as-is.

**Option A recipe.**

1. Write `src/foundation/04\_load\_from\_sources.py` (new). Reuse the helpers in `03\_seed\_lakebase.py`:
`\_connect()`, `TABLE\_COLUMNS`, `\_init\_codecs`, `prepare\_value`. Replace `truncate + insert` with an
upsert. Sketch:

```python
   # Upsert one table. `conflict\_cols` is the business key, e.g. \["product\_id"].
   async def upsert\_table(conn, table, records, conflict\_cols):
       cols = TABLE\_COLUMNS\[table]
       placeholders = ", ".join(f"${i+1}" for i in range(len(cols)))
       updates = ", ".join(f"{c}=EXCLUDED.{c}" for c in cols if c not in conflict\_cols)
       sql = (f"INSERT INTO dpp.{table} ({', '.join(cols)}) VALUES ({placeholders}) "
              f"ON CONFLICT ({', '.join(conflict\_cols)}) DO UPDATE SET {updates}")
       rows = \[tuple(prepare\_value(r.get(c), c) for c in cols) for r in records]
       await conn.executemany(sql, rows)
   ```

   Only `product\_passport.product\_id` (and PKs) have unique constraints today. For child tables
(`product\_materials`, `compliance\_records`, …) either add a natural unique key (e.g.
`(passport\_id, regulation\_name)`) in the DDL, or use "delete children for this passport then insert"
inside one transaction. Pick one and document it.

2. Load in the FK order that `TABLE\_LOAD\_ORDER` already defines: manufacturer → supplier →
passport → children. Insert **suppliers parents-first** (tier 1, then 2, then 3), because
`parent\_supplier\_id` is a foreign key.
3. Put the connection logic behind an identity that is *not a person*: run this as a job under the
production service principal.
4. **Validate before you write.** Reuse the invariants the unit tests assert: material percentages per
product sum to \~100, no supplier-parent cycles, valid enums, FK integrity. Reject or quarantine bad
records (a `rejected\_records` table is a good addition) instead of failing the whole load.
5. **Schedule it** (a job with a cron schedule, or file-arrival trigger) and **make it idempotent** — a
re-run must not create duplicates or lose edits.
6. **Decide conflict rules** between source feeds and Supplier Portal edits. Example: certificates are
owned by the portal; materials are owned by PLM. Otherwise the nightly load will silently overwrite
a supplier's renewal.
7. Keep the synthetic generator — but only for dev/test environments, never prod.

**Dynamic data (battery state of health).** Today `state\_of\_health\_pct`, `cycle\_count`,
`dynamic\_data\_updated\_at` hold only the *latest* value, with no update pipeline and no history. If you need
it, add a time-series table (ROADMAP item 2), stream BMS telemetry into Delta (Structured Streaming /
Zerobus), and write the latest value to Lakebase. Note that `clean\_and\_conform.py`'s
`silver\_circularity\_info` does **not** currently select those three columns, so they never reach Silver or
Gold; add them if analytics need them.

**Done when:** a scheduled job loads real records into dev Lakebase; the viewer shows a real product;
re-running the job changes nothing.

### Phase 5 — Adapt the pipeline and analytics (1–2 weeks)

1. Run `dpp\_ingestion\_pipeline` against the real data. Watch the **expectation metrics** in the pipeline UI:
`expect\_or\_drop` silently removes rows, so a spike in drops means your data violates a rule.
2. Tune Silver rules to your regulation (e.g. what counts as "complete"? currently serial number +
facility + country + QR URL).
3. Replace the Gold risk weights in `compliance\_gap\_analysis.py` (30/15/20/10/15/10) with your
compliance team's logic. The weights are illustrative.
4. Edit the dashboard **datasets** in `resources/dashboards/dpp\_compliance.lvdash.json` if you changed the
catalog/schema (it hardcodes `dpp\_dev.dpp.`):
`sed -i 's/dpp\_dev\\.dpp\\./<catalog>.<schema>./g' resources/dashboards/dpp\_compliance.lvdash.json`
5. Create the Genie space from `resources/genie/dpp\_compliance\_explorer.yml` (manual or REST API),
adding table/column descriptions so it answers *your* vocabulary.
6. Set `development: false` on the pipeline and decide whether to stay on `channel: PREVIEW`
(the roadmap suggests moving to `CURRENT` once supported).

**Done when:** dashboard numbers reconcile against a hand-checked sample of 20 products.

### Phase 6 — Identity, QR codes and the EU registry (varies; biggest regulatory risk)

What the demo does: each seeded passport stores a *fictional* `qr\_code\_url`
(`https://dpp.voltcore.se/passport/<uuid>`), but the viewer's `/api/qr/{id}` ignores it and encodes the
app's own `/passport/<uuid>` URL.

What a real system needs (the repo does **not** implement any of this; see ROADMAP items 5 and 7):

1. A **resolvable, permanent URL** you control (e.g. `https://dpp.yourcompany.com/p/<id>`) that keeps
working for the product's lifetime, even if you later move off Databricks Apps. Put a stable
hostname/reverse proxy in front of the apps; do not print the `\*.databricksapps.com` URL on products.
2. The **unique product identifier scheme** and **data-carrier** format required by the applicable
standards (the repo cites EN 18219/18220) and **registration** in the EU DPP Registry with its
issued identifier. Track the legal status yourself: dates and implementing acts are still moving.
3. Store the registry identifier and registration status on the passport (new columns) and make the
QR resolve through it.
4. A **backup/continuity** plan (ROADMAP item 6): the passport must stay available even if your
company ceases trading.

**Done when:** legal/compliance sign off the identifier and carrier design.

### Phase 7 — Security and access control (do not skip)

The code is demo-grade here. Fix these *before* real data goes in:

1. **Replace the `GRANT … TO PUBLIC`** at the end of `01\_setup\_lakebase\_schema.sql` with per-role grants.
Viewer's SP: `USAGE` on schema + `SELECT` on tables. Supplier-portal SP: `SELECT` plus `INSERT/UPDATE` only
on the tables it writes (`product\_origin`, `supplier`). The ingestion SP: write on all `dpp.\*`.
Find each app's role name from the injected `PGUSER`.
2. **Supplier authorization.** The portal has **no mapping from the logged-in user to a supplier**
— `/supplier/{supplier\_id}` and the renew POST trust the ID in the URL. Anyone who can open the app
can edit any supplier. Add a `supplier\_user(supplier\_id, user\_email)` table, read the identity from
the `X-Forwarded-Email` header Databricks Apps adds, and reject requests where the user is not
linked to that supplier. Consider whether suppliers should instead be external users, which usually
needs a different front door than workspace SSO.
3. **Real audit identity.** The trigger stores `current\_user` as `changed\_by`. Because the apps connect
as their service principal, every audit row says the app's SP, not the human. Pass the user through
(e.g. `SET LOCAL app.user\_email = ...` in the same transaction and have the trigger read
`current\_setting('app.user\_email', true)`).
4. **Tiered public/restricted/authority access.** Today `/api/passport/{id}` returns *every* field to
anyone who can reach the app. Implement an `access\_tier` classification, split the response into
tiers, add authentication for the non-public ones, and consider Unity Catalog row/column controls for
the lakehouse copy (ROADMAP item 1). Do not treat unit costs, supplier identities or due-diligence
documents as public by default.
5. **Public access path.** Databricks Apps sit behind workspace login. A consumer scanning a QR is not
a workspace user. Decide how public passport data is exposed (a separate public-facing read-only
service/API gateway in front of a *public-tier* dataset is the safest pattern) rather than
opening the internal app.
6. **Input validation.** A malformed (non-UUID) ID causes a Postgres error that surfaces as a 500;
validate UUIDs up front and return 404 (ROADMAP "Engineering hardening").
7. **Privacy.** Leave `ACCESS\_LOG\_ENABLED=false` unless legal agrees; it stores user emails (personal data).
8. **AI text.** LLM-generated insights are advisory text on a regulated document. Label them clearly as
AI-generated, never show them as certified data, and keep them out of the legally binding passport view
unless reviewed.

**Done when:** a written threat model exists and each item above is closed or explicitly accepted.

### Phase 8 — Make the apps production-grade (1–2 weeks)

1. Edit `app.yaml` env values to match your instance/catalog by hand (no variable substitution).
2. Extract the duplicated DB code in both apps into a shared module (ROADMAP item).
3. Replace `asyncio.get\_event\_loop()` with `asyncio.to\_thread` in the insights path (ROADMAP item).
4. Add caching for hot passports and think about load: each page view runs several SQL queries; a
viral QR scan needs a pool size and Lakebase capacity plan (`max\_size=10` per app instance today).
5. Pin dependency versions in `requirements.txt` (currently open-ended `>=`).
6. Add integration tests (the current tests only cover the data generator). At minimum test: the viewer
returns 200 for a seeded passport and 404 for an unknown one; the renew POST writes an audit row.

### Phase 9 — Operations and rollout

1. **CI/CD:** on pull request run `pytest`, `databricks bundle validate`, and a deploy to dev; on tag
deploy to staging then prod via the `prod` target (`mode: production`, runs as the SP).
2. **Scheduling:** schedules are `PAUSED` by default in `dpp\_jobs.yml`. Unpause the loader, pipeline and
`precompute\_insights` with a deliberate order (load → pipeline → insights).
3. **Cost control:** `precompute\_insights` calls the LLM for **every** passport on every run and
overwrites the table. For large catalogs, only regenerate passports whose `updated\_at` changed, and
check which model/endpoint your workspace allows (`AI\_MODEL`, default
`databricks-meta-llama-3-3-70b-instruct`; endpoint availability varies).
4. **Monitoring:** alert on job/pipeline failures, on the `/health` endpoint (it returns 503 if the DB is
down), and on expectation-drop counts.
5. **Backup and retention:** document RPO/RTO for Lakebase and for the Delta tables; pair with the
continuity obligation in Phase 6.
6. **Pilot:** go live with one product line, one supplier cohort and an internal compliance reviewer
before opening it up.

### 7.6 Scaling notes

* 500 passports → trivial. Millions of item-level passports changes three things: the **full-snapshot
Bronze read** becomes expensive (move to incremental change capture, Lakebase sync or
Lakeflow Connect, as the README's production-hardening section suggests); the Gold
`collect\_list` aggregation gets heavy; and `precompute\_insights`' driver-side `collect()` plus
`executemany` upsert will not scale (batch it, or write through a sync).
* Keep per-request lookups indexed by `passport\_id`/`product\_id`/`gtin` (indexes exist). Add indexes for
any new lookup key (e.g. registry identifier).

\---

## 8\. Day-to-day developer tasks (recipes)

### 8.1 Add a column or table (the full checklist)

For a new column on, say, `environmental\_impact`:

1. `01\_setup\_lakebase\_schema.sql` — add the column in `CREATE TABLE` **and** an
`ALTER TABLE … ADD COLUMN IF NOT EXISTS …` (so existing databases upgrade).
2. `03\_seed\_lakebase.py` — add it to `TABLE\_COLUMNS\["environmental\_impact"]`.
3. `02\_synthetic\_data\_generator.py` — emit it in `\_generate\_impact`.
4. `silver/clean\_and\_conform.py` — add it to the `select(...)` (**Silver lists columns explicitly; anything
not listed is dropped**).
5. Gold files that need it — add to the select/aggregation.
6. Apps — query/template (`templates/passport.html`).
7. `tests/test\_synthetic\_data.py` — add assertions.
8. Re-run: setup job (or your loader) → pipeline → apps.

If the table is **new**, also add it to `TABLE\_LOAD\_ORDER`, the generator's `tables` dict in
`00\_setup\_lakebase.py`, and (if it has `passport\_id`) to the audit-trigger table list in the DDL.

### 8.2 Add a new industry profile (e.g. steel)

1. In `02\_synthetic\_data\_generator.py` build: a product catalog, material DB, certifications, carbon
ranges, lifecycle splits, regulations, facilities, component names, durability ranges and disposal
components, using the `battery` or `furniture` entries as templates.
2. Add the new key to `PROFILES`.
3. Select it at **deploy time**: `databricks bundle deploy -t dev --var="profile=steel"` (passing
`--var` only to `bundle run` has no effect, because the value is baked into the job parameters at deploy).
4. Run `pytest` — the tests check structure, FK integrity and distributions for the profile you pass.
Note `00\_setup\_lakebase.py` defaults to `furniture` when run directly *without* `--profile` or
`DPP\_PROFILE`; the bundle always passes the profile explicitly.

### 8.3 Change which catalog/schema you deploy to

```bash
databricks bundle deploy -t dev --var="catalog=my\_catalog" --var="schema=dpp" \\
  --var="dashboard\_warehouse\_id=<id>"
sed -i 's/dpp\_dev\\.dpp\\./my\_catalog.dpp./g' resources/dashboards/dpp\_compliance.lvdash.json
```

Then update the Genie spec tables by hand.


### 8.4A Deploy and operate the Furniture target from Windows PowerShell

Run these commands from the Furniture repository:

```powershell
cd "C:\College work\TY\Machine Learning\dpp-solution-accelerator-furniture"

# Confirm the selected bundle target and validate its configuration.
databricks bundle validate -t furniture --profile dpp-dev

# Deploy bundle resources. A deployment may report a partial failure if a resource
# (for example, a new App) is blocked by the workspace app quota; check each resource.
databricks bundle deploy -t furniture --profile dpp-dev

# Run the synthetic setup only against the Furniture instance/profile.
databricks bundle run synthetic_data_setup -t furniture --profile dpp-dev

# Run the ingestion pipeline and AI insights job when needed.
databricks bundle run dpp_ingestion_pipeline -t furniture --profile dpp-dev
databricks bundle run precompute_insights -t furniture --profile dpp-dev

# Inspect the Furniture Supplier Portal.
databricks apps get dpp-furniture-supplier-portal --profile dpp-dev
databricks apps logs dpp-furniture-supplier-portal --tail-lines 100 --profile dpp-dev
```

The setup job is destructive to the target database because it truncates and reseeds tables.
Only run it when intentionally rebuilding the **Furniture demo** data. Do not run it against
production data or the Battery instance.

To inspect the Furniture Lakebase project and endpoint:

```powershell
databricks postgres list-projects --profile dpp-dev
databricks postgres get-endpoint projects/dpp-furniture-passport/branches/production/endpoints/primary --profile dpp-dev
```

If the endpoint is disabled, first inspect its current state. Only if you have confirmed that
the target is the Furniture endpoint, create a UTF-8 (no BOM) request file:

```powershell
$json = '{"spec":{"disabled":false}}'
[System.IO.File]::WriteAllText(
    "$PWD\enable_furniture_endpoint.json",
    $json,
    [System.Text.UTF8Encoding]::new($false)
)

databricks postgres update-endpoint projects/dpp-furniture-passport/branches/production/endpoints/primary spec.disabled --json "@enable_furniture_endpoint.json" --profile dpp-dev
```

After the update, run `get-endpoint` again and confirm `spec.disabled` is `false` and the
endpoint state is `ACTIVE`. Never substitute the Battery project name in these commands.

### 8.4 Run the apps locally

```bash
cd src/apps/passport\_viewer
pip install -r requirements.txt
export LAKEBASE\_INSTANCE=dpp-passport LAKEBASE\_USER=<pg user> LAKEBASE\_PASSWORD=<password>
export DATABRICKS\_HOST=https://<workspace>   # needed for the live AI path
uvicorn main:app --reload --port 8000
```

Without `PGHOST` the app uses the native-login fallback; that requires a Lakebase instance created with
`--enable-pg-native-login` and a password you created yourself. This is a dev convenience only.

### 8.5 Inspect and debug the data

* Operational data: connect to Lakebase with any Postgres client; schema is `dpp`.
* Analytics: query `<catalog>.<schema>.gold\_\*` in a SQL warehouse.
* Audit trail: `SELECT \* FROM dpp.passport\_audit\_log ORDER BY changed\_at DESC LIMIT 20;`
* Why did a passport vanish from Silver? Check pipeline expectation metrics for `expect\_or\_drop` rules.

\---

## 9\. Known gaps and traps in the current code

These are things I confirmed by reading the source. None is a bug in the *demo*, but each one is a trap
in a *real* deployment.

|#|Gap|Where|Impact|What to do|
|-|-|-|-|-|
|1|**The setup job wipes all tables.** `truncate\_tables()` runs `TRUNCATE … CASCADE` on every table (including `passport\_audit\_log`) before inserting.|`03\_seed\_lakebase.py`, called by `00\_setup\_lakebase.py`|Re-running it on a real system **destroys all passport data and audit history**.|Use it only in dev. Build a separate upsert loader (Phase 4). Consider removing the job from the prod target.|
|2|**No tiered access.** All data is served openly, including the JSON API.|`passport\_viewer/main.py`|Cannot meet restricted/authority-only requirements.|Phase 7 item 4; ROADMAP #1.|
|3|**Supplier Portal has no user→supplier authorization.** Supplier IDs come from the URL.|`supplier\_portal/main.py`|Any user with app access can edit any supplier's certificates.|Phase 7 item 2.|
|4|**Audit `changed\_by` is the app's service principal**, not the human.|trigger uses `current\_user`|The audit trail cannot say *who* made a change.|Phase 7 item 3.|
|5|**`GRANT … TO PUBLIC`.**|end of the DDL|Every Postgres role can read and write everything.|Least-privilege grants.|
|6|**Fictional QR URL stored in data; QR image uses a different URL.** No registry ID / URI.|generator `qr\_base\_url` vs `/api/qr/{id}`|QR codes would not survive a change of host.|Phase 6.|
|7|**State-of-health fields stop at Lakebase.** Silver's circularity select omits them.|`clean\_and\_conform.py`|Not in Gold, dashboard, or AI prompts for the lakehouse path.|Add columns to Silver/Gold if needed.|
|8|**Bronze is a full snapshot**, though comments say "streaming tables".|`ingest\_lakebase\_to\_bronze.py`|Cost grows with data size.|Incremental capture (Phase 7.6).|
|9|**Hardcoded values DABs won't substitute**: catalog in the dashboard JSON; instance/DB name in both `app.yaml` files.|listed files|Silent mismatches after renaming.|Edit by hand, or template with a script in CI.|
|10|**`dashboard\_warehouse\_id` has a hardcoded default** (a specific workspace's ID) although a comment mentions a placeholder.|`databricks.yml`|Dashboard deploy will fail or point at a warehouse you cannot use.|Always pass `--var`.|
|11|**CI workflow is tied to a Databricks-internal workspace/runner.** README also mentions a `ci.yml` that does not exist (files are `databricks-ci.yml`, `publish.yaml`).|`.github/workflows/`|CI fails in your org.|Replace (Phase 2).|
|12|**Setup script's default profile differs by entry point.** Generator default is `battery`; `00\_setup\_lakebase.py`'s argparse default is `furniture` unless `DPP\_PROFILE` is set.|foundation scripts|Surprising data when run locally without `--profile`.|Always pass `--profile`.|
|13|**Invalid IDs cause 500s** (Postgres UUID parse error).|both apps|Poor errors; log noise.|Validate UUIDs.|
|14|**Certificate renewal updates only `product\_origin`** (and touches `supplier.updated\_at`). It does not update `compliance\_records` or `supplier.last\_audit\_date`.|`renew\_certification`|Compliance status may lag the supplier's action.|Decide the business rule; extend the transaction.|
|15|**Pipeline is `development: true` on the `PREVIEW` channel.**|`dpp\_pipelines.yml`|Not production posture.|Phase 5 item 6.|
|16|**AI cost and trust.** Every passport is sent to an LLM on each run; output is not validated beyond JSON schema.|`precompute\_insights.py`|Cost; hallucinated "insights" on a regulated record.|Phase 9 item 3; Phase 7 item 8.|
|17|**Only the data generator has tests** (42 pass). No tests for apps, pipelines or SQL.|`tests/`|Regressions go unnoticed.|Phase 8 item 6.|
|18|**`reyden\_toggle.py` is dead code** (not imported anywhere).|`src/serving/`|Confusion.|Ignore or delete.|
|19|**Regulatory content is a snapshot** (dates, article numbers, phase-in rules).|README, docs, generator|Law changes; the repo says to confirm against the applicable act.|Get legal review.|

\---

## 10\. Production readiness checklist

**Data and model**

* \[ ] Attribute-to-source mapping approved; mandatory attributes for your product group covered
* \[ ] Real-data loader is an idempotent upsert; the destructive setup job is not in the prod target
* \[ ] Validation + quarantine for bad source records
* \[ ] Source-vs-portal ownership rules documented

**Security and privacy**

* \[ ] Least-privilege Postgres grants (no `PUBLIC`)
* \[ ] Supplier user→supplier authorization implemented and tested
* \[ ] Audit rows capture the real human
* \[ ] Public vs restricted vs authority tiers designed and enforced
* \[ ] Public access path separate from the internal workspace apps
* \[ ] Access logging decision made with legal; AI text labelled

**Identity and regulation**

* \[ ] Permanent resolvable URL and data carrier agreed with compliance
* \[ ] EU Registry registration/identifier handled; continuity/backup plan

**Platform**

* \[ ] `prod` deploys under a service principal; CI replaced and green
* \[ ] Pipeline in production mode; schedules unpaused in the right order
* \[ ] Monitoring and alerts; backup/RPO/RTO documented; capacity tested
* \[ ] Dependency versions pinned; integration tests added

\---

## 11\. Troubleshooting

|Symptom|Likely cause|Fix|
|-|-|-|
|`bundle deploy` complains about the warehouse / dashboard|Default `dashboard\_warehouse\_id` is not yours|Pass `--var="dashboard\_warehouse\_id=<id>"` (`databricks warehouses list`)|
|`auth login` hangs or old behaviour|Old `databricks` CLI first on `PATH`|`databricks version`; use ≥ 0.278|
|Pipeline fails with missing table/column|Pipeline ran before the setup job, or schema is outdated|Run the setup job first (DDL is idempotent)|
|Pipeline can't connect to Lakebase|Running identity has no Postgres role/permission|Grant the identity access to the instance and `SELECT` on `dpp.\*`|
|App shows DB errors / `/health` is 503|Lakebase resource not bound, schema not created, or endpoint disabled|Check `resources/apps.yml`; verify the correct endpoint with `databricks postgres get-endpoint`; do not rerun the destructive setup job just to fix a connection error|
|`asyncpg` says "The endpoint has been disabled"|The Lakebase endpoint has `spec.disabled=true`|Verify the exact project/branch/endpoint, enable only the intended endpoint with `postgres update-endpoint`, then confirm `current_state=ACTIVE`|
|"No Lakebase credentials found"|No `PGHOST` (binding missing) and no `LAKEBASE\_PASSWORD`|Deploy via the bundle, or set the native-login fallback|
|AI Insights tab says "generated on a schedule"|Cache empty and live model call failed|Run `precompute\_insights`; verify the `AI\_MODEL` endpoint exists and is permitted|
|Dashboard empty|Dashboard datasets point at a different catalog|Rewrite catalog in the `.lvdash.json`|
|Rows missing in Silver/Gold|`expect\_or\_drop` rule removed them|Inspect pipeline expectation metrics|
|App links point at the wrong place after rename|`app.yaml` values not substituted|Edit `LAKEBASE\_INSTANCE` / `LAKEBASE\_DATABASE\_NAME`|
|Seed fails on re-run: `uq\_product\_id`|Data already present and not truncated (custom loader)|Use upsert, or truncate in dev only|

\---

## 12\. Glossary

* **DPP** — Digital Product Passport.
* **ESPR** — EU Ecodesign for Sustainable Products Regulation (2024/1781).
* **Economic operator** — the manufacturer/importer legally responsible for the passport.
* **ABAC** — attribute-based access control.
* **LCA** — life-cycle assessment (how carbon/energy/water are computed).
* **SVHC / REACH** — EU chemicals rules (substances of very high concern).
* **SoH** — battery state of health.
* **BMS** — battery management system.
* **OLTP / OLAP** — transactional vs analytical workloads.
* **CDC** — change data capture (incremental sync of changes).
* **DAB** — Databricks Asset Bundle.
* **SDP / DLT** — Spark Declarative Pipelines / Delta Live Tables.
* **SP** — service principal (a machine identity).
* **UC** — Unity Catalog.
* **URI (EU registry)** — the Registry-issued unique identifier for a passport (not a generic web URI).

\---

*This guide combines the original v1.0.0 reference-architecture description with the project deployment
state verified on 12 November 2026. Deployment status can change. Regulatory dates and product-group
scope change; the operator remains responsible for confirming requirements against the acts that apply.*

