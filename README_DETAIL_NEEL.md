# DPP Solution Accelerator --- Implementation Guide for New Programmers

This guide explains **what this codebase does, how its pieces fit
together, and how to turn it from a demo into a real Digital Product
Passport (DPP) system** for an actual manufacturer.

-   Want a 5-minute overview or the deploy commands? Read
    [README.md](README.md).
-   Want to understand the code and take it to production? Read this
    file.

> \*\*Honesty note.\*\* This repository is a \*reference architecture
> with synthetic data\*, not a turnkey compliance product (the README
> says so too). Part 9 lists the gaps I found while reading the code.
> Read it before you promise anything to a customer.

---

## Table of contents

1.  [The problem in plain language](#1-the-problem-in-plain-language)
2.  [Concepts you need first](#2-concepts-you-need-first)
3.  [How the system works (big
    picture)](#3-how-the-system-works-big-picture)
4.  [Code tour: what every folder and file
    does](#4-code-tour-what-every-folder-and-file-does)
5.  [The data model](#5-the-data-model)
6.  [Step 0 --- Run the demo end to
    end](#6-step-0--run-the-demo-end-to-end)
7.  [Implementing this for a real company (the main
    roadmap)](#7-implementing-this-for-a-real-company)
8.  [Day-to-day developer tasks
    (recipes)](#8-day-to-day-developer-tasks-recipes)
9.  [Known gaps and traps in the current
    code](#9-known-gaps-and-traps-in-the-current-code)
10. [Production readiness checklist](#10-production-readiness-checklist)
11. [Troubleshooting](#11-troubleshooting)
12. [Glossary](#12-glossary)

---

------------------------------------------------------------------------

## Current implementation status --- parallel furniture deployment (updated 4 Nov 2026)

This section records the work performed after the original guide was
written. The goal is to retain the existing battery deployment and
create an independent furniture deployment for **Furniture, Textiles,
Lighting, and Storage**. The original battery workspace resources and
Lakebase data must remain untouched.

### Deployment strategy

The furniture work is being carried out in a separate local project
copy:

``` text
C:\College work\TY\Machine Learning\dpp-solution-accelerator-furniture
```

The original battery project remains in:

``` text
C:\College work\TY\Machine Learning\dpp-solution-accelerator-main
```

The furniture copy has a separate bundle identity and target:

  ----------------------------------------------------------------------------------
  Setting                 Existing battery deployment  New furniture deployment
  ----------------------- ---------------------------- -----------------------------
  Bundle name             `dpp-solution-accelerator`   `dpp-furniture-accelerator`

  Target                  `dev`                        `furniture`

  Unity Catalog catalog   `dpp_dev`                    `dpp_furniture`
  (intended)                                           

  Lakebase instance       `dpp-passport`               `dpp-furniture-passport`
  (intended)                                           

  Data profile            `battery`                    `furniture`

  Manufacturer example    VoltCore Energy AB           NordicForm AB

  Product categories      Battery profile categories   Furniture, Textiles,
                                                       Lighting, Storage
  ----------------------------------------------------------------------------------

**Isolation rule:** do not run the furniture setup job against
`dpp-passport`. The setup script truncates the DPP tables before
reseeding them, so using the battery instance would replace the battery
operational data. Create and verify the separate furniture Lakebase
instance and catalog before running the furniture setup job.

### Local project changes already made

The following changes have been made in the furniture project copy (not
the original battery copy):

1.  Copied the project to `dpp-solution-accelerator-furniture`.
2.  Removed the copied `.databricks` deployment state and `.git`
    metadata from the furniture copy so it does not inherit the battery
    bundle's local deployment state.
3.  Changed the bundle name in `databricks.yml` to
    `dpp-furniture-accelerator`.
4.  Added a `furniture` target configured for the `furniture` profile,
    catalog `dpp_furniture`, schema `dpp`, and intended Lakebase
    instance `dpp-furniture-passport`.
5.  Changed the furniture app resource names to:
    -   `dpp-furniture-passport-viewer`
    -   `dpp-furniture-supplier-portal`
6.  Renamed the furniture pipeline and jobs for clarity:
    -   `DPP Furniture Ingestion Pipeline`
    -   `DPP Furniture - Synthetic Data Setup`
    -   `DPP Furniture - Pre-compute AI Insights`
7.  Changed the dashboard display name to
    `DPP Furniture Compliance & Sustainability`.
8.  Replaced dashboard SQL references from `dpp_dev.dpp` to
    `dpp_furniture.dpp` in
    `resources/dashboards/dpp_compliance.lvdash.json`.
9.  Updated the app YAML fallback instance names to
    `dpp-furniture-passport`.
10. Updated the Passport Viewer fallback Supplier Portal app name to
    `dpp-furniture-supplier-portal`.

These changes have been made in the working copy during the guided
setup. Before deployment, inspect the actual files and run validation
again; do not assume a resource binding or variable is correct merely
because its name was changed.

### Validation completed

  ------------------------------------------------------------------------------
  Check                   Result                  Notes
  ----------------------- ----------------------- ------------------------------
  Databricks CLI version  Passed                  CLI `v1.20.0`

  Databricks              Passed after re-login   Profile `dpp-dev`; secure OS
  authentication                                  keyring

  Furniture bundle        Passed with one warning Bundle
  validation                                      `dpp-furniture-accelerator`,
                                                  target `furniture`

  Dashboard JSON parse    Passed                  JSON parsed successfully after
                                                  catalog replacement

  Dashboard SQL catalog   Checked                 Search output showed
  replacement                                     `dpp_furniture.dpp` references
                                                  and no `dpp_dev.dpp` matches

  Synthetic-data tests    Passed                  `42 passed, 1 warning`
  ------------------------------------------------------------------------------

The bundle validation warning is:

``` text
The "development" property of pipelines is deprecated.
Use "mode: development" in databricks.yml instead.
```

This is a deprecation warning, not a validation failure. It can be
cleaned up after checking the pipeline resource schema and bundle
configuration.

The pytest warning is:

``` text
PytestRemovedIn10Warning:
Class-scoped fixture defined as instance method is deprecated.
```

It did not cause a test failure. The fixture can be changed to a
`@classmethod` if the test setup and fixture scope require class-level
state.

The 42 passing tests validate the synthetic-data generator and its
data-quality invariants. They do **not** yet prove that the furniture
Lakebase schema, deployed pipeline, dashboard queries, or apps work in
the Databricks workspace. Those still require deployment and integration
checks.

### Lakebase inventory checked --- creation still pending

The command
`databricks database list-database-instances --profile dpp-dev` returned
the existing battery instance:

  Property                  Existing instance
  ------------------------- -------------------
  Name                      `dpp-passport`
  PostgreSQL version        16
  Capacity                  `CU_1`
  Node count                1
  Retention window          7 days
  Native PostgreSQL login   Disabled
  State at last check       `STOPPED`

No furniture instance appeared in that listing. The furniture instance
has **not yet been created**. The next intended instance name is
`dpp-furniture-passport`, with an initial configuration matching the
existing instance (`CU_1`, one node, seven-day retention, native login
disabled), subject to workspace capacity, permissions, and current
pricing.

The CLI v1.20.0 help confirms that the relevant commands are available:

``` powershell
databricks database create-database-instance --help
databricks database list-database-instances --help
```

The creation command has been identified but **has not yet been run**:

``` powershell
databricks database create-database-instance dpp-furniture-passport `
  --capacity CU_1 `
  --node-count 1 `
  --retention-window-in-days 7 `
  --profile dpp-dev
```

Lakebase Database Instances are a Public Preview CLI feature in this CLI
version. Confirm the workspace supports the requested configuration and
review potential charges before creating the instance.

### Current status and next steps

**Completed:** separate local project copy, bundle identity/target
changes, furniture-specific resource naming, dashboard catalog text
replacement, app fallback updates, CLI authentication, bundle
validation, dashboard JSON parsing, and 42 generator tests.

**Not yet completed:**

1.  Create `dpp-furniture-passport` and verify it reaches the expected
    available/active state.
2.  Create or verify the separate Unity Catalog catalog `dpp_furniture`.
3.  Inspect the final furniture `databricks.yml` and resource YAMLs to
    confirm all Lakebase bindings, app names, job parameters, pipeline
    names, and catalog variables resolve to the furniture resources.
4.  Deploy the furniture bundle with the existing workspace's correct
    SQL warehouse ID supplied explicitly as `dashboard_warehouse_id`.
5.  Run the furniture synthetic-data setup job only after confirming it
    targets the new Lakebase instance.
6.  Run the furniture ingestion pipeline and precompute AI insights.
7.  Deploy and test both furniture apps and the furniture dashboard.
8.  Create or configure a separate furniture Genie space; the Genie YAML
    in this repository is a specification and is not automatically
    deployed as a native bundle resource.

### Commands used during validation

Run from the furniture project directory:

``` powershell
Set-Location "C:\College work\TY\Machine Learning\dpp-solution-accelerator-furniture"

databricks version
databricks auth describe --profile dpp-dev
databricks bundle validate -t furniture --profile dpp-dev
python -m pytest tests/test_synthetic_data.py -v
python -c "import json; json.load(open('resources/dashboards/dpp_compliance.lvdash.json', encoding='utf-8')); print('Dashboard JSON is valid')"
databricks database list-database-instances --profile dpp-dev
```

**Safety reminder:** never run the furniture setup job until the job's
resolved parameters and app resource bindings have been checked to point
to `dpp-furniture-passport` and `dpp_furniture`. Do not change, delete,
stop, or reseed the existing battery instance as part of the furniture
setup.

------------------------------------------------------------------------

## 1. The problem in plain language

The EU now requires many products to carry a **Digital Product
Passport**: a QR code on the product that leads to a web page/API with
the product's origin, materials, carbon footprint, compliance
certificates, repairability/recycling info and disposal instructions.
Batteries come first (mandatory **18 Feb 2027** per this repo's docs);
other product groups follow through the decade. Dates move, so always
confirm against the legal act for your product group.

The key design fact (see [docs/positioning.md](docs/positioning.md)):
the EU does **not** store your passport data. A thin EU registry points
to it. **You (the "economic operator": manufacturer, importer) must run
the backend that stores and serves the passports.** This repo is a
blueprint for that backend on Databricks.

What the system has to do, in programmer terms:

  ------------------------------------------------------------------------
  Need                    What it means           Where it lives here
                          technically             
  ----------------------- ----------------------- ------------------------
  Fast lookup when        Low-latency read of one Lakebase (Postgres) +
  someone scans a QR      product's full record   `passport\_viewer` app

  Suppliers keep their    Transactional writes +  `supplier\_portal` app +
  own data current        an audit trail          Lakebase triggers

  Prove compliance across Batch analytics, data   Bronze/Silver/Gold
  thousands of products   quality checks          pipeline + dashboard

  Ask questions in        Natural-language SQL    Genie space (spec only)
  English                                         

  Explain gaps per        LLM text generation,    `precompute\_insights`
  product                 cached                  job
  ------------------------------------------------------------------------

---

## 2. Concepts you need first

You do not need to be a Databricks expert, but you must understand these
ten ideas. Everything else in the repo is built from them.

1.  **Databricks workspace** --- your company's Databricks account
    environment (web UI + APIs).
2.  **Unity Catalog (UC)** --- Databricks' governance layer. Data is
    addressed as `catalog.schema.table`,
    e.g. `dpp\_dev.dpp.gold\_passport\_complete`.
3.  **Lakebase** --- a managed **PostgreSQL** database inside
    Databricks. Use it for *operational* (OLTP) work: "give me row X in
    5 ms", "update this certificate". The repo uses **Lakebase
    Provisioned**, created by name (`dpp-passport`).
4.  **Lakehouse tables (Delta)** --- columnar analytics tables in UC.
    Good for scanning millions of rows, bad for single-row lookups.
    Spark reads/writes them.
5.  **Medallion layers** --- a naming convention for the analytics
    pipeline: **Bronze** = raw copy, **Silver** = cleaned and validated,
    **Gold** = business-ready aggregates.
6.  **Spark Declarative Pipelines (SDP)** --- formerly Delta Live
    Tables. You *declare* tables as Python functions decorated with
    `@dlt.table`; the platform works out dependency order and runs them.
    Data-quality rules are `@dlt.expect...` decorators.
7.  **Databricks Apps** --- hosts a web app (here, FastAPI) inside your
    workspace, behind Databricks login (SSO), with its own **service
    principal** identity.
8.  **Service principal (SP)** --- a non-human identity. Each app and
    each production job runs as one.
9.  **Databricks Asset Bundles (DABs)** --- "infrastructure as code" for
    Databricks. `databricks.yml` and `resources/\*.yml` declare jobs,
    pipelines, apps and the dashboard; `databricks bundle deploy`
    creates them all.
10. **OAuth short-lived tokens instead of passwords** --- nothing in
    this repo stores a database password. Code asks the Databricks API
    for a token that expires in about an hour (see
    `\_resolve\_runtime\_lakebase` in the foundation scripts and
    `\_lakebase\_oauth\_token` in the apps).

---

## 3. How the system works (big picture)

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

**The most important architectural idea:** *Lakebase is the source of
truth; the lakehouse is a derived copy.* Data flows **Lakebase →
Bronze**, not the other way round. Apps read and write Lakebase
directly; analytics never write back (except the AI-insights cache). If
you remember one thing from this guide, remember that direction, because
it determines how you plug in real data (Part 7).

### Runtime sequence for each scenario

**A. A consumer scans a QR code**

1.  QR encodes `https://<viewer-app-url>/passport/<passport\_id>` (built
    in `/api/qr/{id}`).

2.  `GET /passport/{id}` → `\_get\_full\_passport()` runs several SQL
    queries against Lakebase (passport

    -   manufacturer, origins + suppliers, materials, impact,
        compliance, circularity, disposal).

3.  A recursive SQL query (`\_attach\_supplier\_chains`) follows
    `parent\_supplier\_id` to show the upstream supplier chain per
    component.

4.  Jinja2 renders `passport.html` with 7 tabs. The AI Insights tab
    calls `/api/insights/{id}` asynchronously (see D).

**B. A supplier renews a certificate**

1.  `GET /supplier/{id}/renew/{origin\_id}` shows a pre-filled form.
2.  `POST` validates the origin belongs to that supplier and the date
    parses, then runs two `UPDATE`s in **one transaction**
    (`product\_origin` and `supplier.updated\_at`).
3.  A Postgres **trigger** (`dpp.audit\_trigger\_func`) writes old/new
    row JSON into `dpp.passport\_audit\_log` automatically --- the app
    code never writes the audit row.
4.  Next pipeline run propagates the change to Silver/Gold; the
    dashboard updates.

**C. Nightly analytics**

1.  `dpp\_ingestion\_pipeline` reads every Lakebase table over JDBC into
    Bronze (full snapshot), cleans into Silver (with expectations),
    builds Gold.
2.  `precompute\_insights` job calls `ai\_query` over
    `gold\_passport\_complete` for every product, writes
    `gold.dpp\_passport\_insights` (Delta) and upserts JSON into
    Lakebase `dpp.passport\_insights`.

**D. AI Insights tab** `/api/insights/{id}` → (1) look in Lakebase
cache; (2) if missing, call the Foundation Model serving endpoint live;
(3) if that fails, return a friendly `pending` message. The page never
errors because of AI.

---

## 4. Code tour: what every folder and file does

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

### 4.1 `src/foundation/` --- building the operational store

  -------------------------------------------------------------------------------------
  File                                  Role                    Read this when...
  ------------------------------------- ----------------------- -----------------------
  `01\_setup\_lakebase\_schema.sql`     Idempotent DDL: 12      You change the data
                                        tables, indexes, audit  model
                                        trigger, grants         

  `02\_synthetic\_data\_generator.py`   `DPPDataGenerator`      You add an industry or
                                        class + `PROFILES` dict need test data
                                        (battery, furniture)    

  `03\_seed\_lakebase.py`               asyncpg helpers:        You load data into
                                        `\_connect()`,          Lakebase
                                        `insert\_table()`,      
                                        `truncate\_tables()`,   
                                        column lists in         
                                        `TABLE\_COLUMNS`        

  `00\_setup\_lakebase.py`              The job entry point:    You wonder what the
                                        split SQL → apply DDL → setup job does
                                        generate in memory →    
                                        truncate → insert       
  -------------------------------------------------------------------------------------

Things worth knowing:

-   **Numeric prefixes** (`00\_`, `01\_`) are not importable Python
    names, so `00\_setup\_lakebase.py` loads its siblings with
    `importlib.util.spec\_from\_file\_location`.
-   **Why generate in memory?** Serverless job tasks do not share a
    filesystem, so a "generate JSON file → seed JSON file" two-task
    design cannot work. One task does everything.
-   **`split\_sql()`** is a small hand-written parser that splits the
    DDL on `;` while respecting `$$ ... $$` function bodies and `--`
    comments, because asyncpg executes one statement at a time.
-   **`run\_async()`** runs the coroutine in a fresh thread/event loop
    because Databricks serverless already has a running loop, which
    makes `asyncio.run()` fail.
-   **Text codecs** (`\_init\_codecs`) let asyncpg accept ISO-date
    strings and UUID strings as-is.
-   **Determinism**: `SEED = 42` and a fixed
    `REFERENCE\_DATE = 2026-03-15`, so every run produces the same 500
    products / 50 suppliers (that is why the tests can assert exact
    properties).

### 4.2 `src/pipelines/` --- Bronze → Silver → Gold

  ------------------------------------------------------------------------------------------------------
  Layer                   File(s)                               What happens
  ----------------------- ------------------------------------- ----------------------------------------
  Bronze                  `ingest\_lakebase\_to\_bronze.py`     Gets a Lakebase OAuth token, then reads
                                                                each `dpp.\*` table via JDBC and adds
                                                                `\_ingested\_at` and `\_source\_table`.
                                                                11 `bronze\_\*` tables (includes
                                                                `app\_access\_log`).

  Silver                  `clean\_and\_conform.py`              Trims/uppercases/lowercases, and applies
                                                                expectations. `@dlt.expect\_or\_drop`
                                                                removes bad rows (null IDs, wrong
                                                                country code length, etc.); plain
                                                                `@dlt.expect` only records violations.
                                                                Adds helper flags: `is\_complete`,
                                                                `certification\_expiring\_soon`,
                                                                `is\_expired`, `needs\_verification`.

  Gold                    `passport\_complete.py`               One wide row per passport
                                                                (materials/compliance aggregated into
                                                                arrays). Feeds AI and apps/analytics.

  Gold                    `supply\_chain\_lineage.py`           `gold\_supply\_chain\_lineage`,
                                                                `gold\_supply\_chain\_summary`,
                                                                `gold\_supplier\_traceability` (flattens
                                                                ancestry with a bounded 2-hop join).

  Gold                    `compliance\_gap\_analysis.py`        Risk score (0-100) and category
                                                                CRITICAL/HIGH/MEDIUM/LOW per product.
                                                                Weights: non-compliant ×30, pending ×15,
                                                                expired ×20, expiring supplier cert ×10,
                                                                incomplete +15, unverified LCA +10.

  Gold                    `carbon\_footprint\_aggregates.py`,   `gold\_carbon\_by\_category`,
                          `material\_circularity\_metrics.py`   `gold\_carbon\_by\_product`,
                                                                `gold\_material\_composition\_trends`,
                                                                `gold\_circularity\_metrics`.

  Gold                    `app\_usage.py`                       Cleans the opt-in access log for the
                                                                dashboard's "App Usage" page.

  Job                     `precompute\_insights.py`             **Not part of the pipeline.** A job that
                                                                runs `ai\_query` with a JSON-schema
                                                                response format, writes a Delta table,
                                                                then upserts into Lakebase.
  ------------------------------------------------------------------------------------------------------

Important detail: the bronze functions are decorated `@dlt.table` but
read with batch `spark.read.format("jdbc")`, so each run **re-reads the
whole table** (a snapshot), regardless of the word "streaming" in
comments. That is fine for 500 products and a cost problem at millions
(Part 7.6).

### 4.3 `src/apps/` --- the two web apps

Both are FastAPI + Jinja2 + asyncpg and share \~200 lines of copy-pasted
database plumbing (the roadmap lists de-duplicating this).

**Passport Viewer** (`passport\_viewer/main.py`)

  -----------------------------------------------------------------------
  Route                               Purpose
  ----------------------------------- -----------------------------------
  `GET /`                             Searchable, filterable, paginated
                                      passport list (page size 24)

  `GET /passport/{id}`                The passport (7 tabs)

  `GET /api/passport/{id}`            Same data as JSON, for system
                                      integration

  `GET /api/qr/{id}`                  PNG QR code; builds the URL from
                                      the `X-Forwarded-Host` header

  `GET /api/insights/{id}`            Cached → live LLM → pending

  `GET /demo`                         "How it's built" explainer page

  `GET /health`                       Returns 503 if the DB is
                                      unreachable
  -----------------------------------------------------------------------

**Supplier Portal** (`supplier\_portal/main.py`): `/` supplier list,
`/supplier/{id}` dashboard, `/supplier/{id}/renew/{origin\_id}` (GET
form, POST write), `/supplier/{id}/audit`, `/health`.

**How the apps authenticate to Postgres** (the part new developers find
confusing):

1.  `resources/apps.yml` binds the Lakebase instance as an app
    *resource*.
2.  Databricks creates a Postgres role for the app's service principal
    and injects `PGHOST`, `PGPORT`, `PGDATABASE`, `PGUSER` --- but **no
    password**.
3.  The app passes a *callable* as asyncpg's `password`
    (`\_oauth\_password`). asyncpg calls it every time the pool opens a
    connection, which mints a fresh OAuth token. A long-lived pool
    therefore never holds an expired token.
4.  If `PGHOST` is absent (local dev) it falls back to
    `LAKEBASE\_USER` + `LAKEBASE\_PASSWORD`.

### 4.4 Config files you will touch most

-   `databricks.yml` variables: `catalog`, `schema`, `profile`,
    `lakebase\_instance\_name`, `lakebase\_database`,
    `dashboard\_warehouse\_id`, `prod\_sp\_name`. Targets: `dev`,
    `staging` (both `mode: development`), `prod` (`mode: production`,
    `run\_as` a service principal).
-   `src/apps/\*/app.yaml`: the app start command and env vars
    (`LAKEBASE\_INSTANCE`, `DPP\_SCHEMA`, `AI\_MODEL`,
    `ACCESS\_LOG\_ENABLED`). **DABs does not substitute `${var.\*}`
    inside `app.yaml` or `.lvdash.json`** --- if you rename the
    instance/catalog you must edit those files by hand.

---

## 5. The data model

Schema `dpp` in Lakebase (12 tables). Think of `product\_passport` as
the hub:

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

Rules enforced by the database itself (so your ingestion code cannot
violate them):

-   `passport\_status` ∈ `draft | active | expired | revoked`
-   `compliance\_status` ∈
    `compliant | pending | non\_compliant | expired`
-   `supplier.tier` 1--3; `risk\_score` 0--10; percentages 0--100
-   `product\_id` is `UNIQUE` (the business key); `passport\_id` is a
    UUID (the technical key)
-   Audit triggers exist on 7 tables: passport, origin, materials,
    impact, compliance, circularity, disposal.

The six EU "categories" map to tables: origin →
`product\_origin`/`supplier`; materials → `product\_materials`;
environmental impact → `environmental\_impact`; compliance →
`compliance\_records`; circularity → `circularity\_info`; disposal →
`disposal\_guidelines`. This is why the schema is "domain-agnostic": a
steel coil and a battery pack both have those six things.

**Industry profiles.** `PROFILES` in the generator is a dict of dicts. A
profile supplies the manufacturer, product catalog, materials,
certifications, carbon ranges, regulations, facilities, etc. The
`battery` profile additionally sets `phased\_regulations` (Art. 7/8/48
items always `pending`), `tier\_components` (mine → cell → module → pack
naming) and `has\_dynamic\_data` (state of health for rechargeable
categories only). Only the *seed data* is industry-specific; schema,
pipeline and apps are not.

---

## 6. Step 0 --- Run the demo end to end

Do this before changing anything. It proves your environment works and
teaches you the moving parts.

### 6.1 Zero-cost local check (no Databricks needed)

``` bash
python -m venv .venv \&\& source .venv/bin/activate
pip install faker pytest
pytest tests -q                       # expect: 42 passed
python src/foundation/02\_synthetic\_data\_generator.py   # writes JSON per table to src/foundation/data/
```

Open a couple of the JSON files to see what a "passport" looks like as
data.

### 6.2 Deploy to a workspace

Prerequisites: Databricks CLI ≥ 0.278 (check `databricks version`), a
workspace with Lakebase and Unity Catalog, permission to create a
catalog (or use an existing one), and a SQL warehouse.

``` bash
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

Why that order: the pipeline reads tables the setup job creates; the
apps need the schema; the AI job needs the Gold layer.

### 6.3 Verify

-   Open the viewer URL → you should see \~500 battery passports
    (profile `battery` is passed by the bundle).
-   Open one → click through all 7 tabs. AI Insights shows a "pending"
    note until `precompute\_insights` has run.
-   Open the supplier portal → pick a supplier → renew a certificate →
    open its audit log and confirm an `UPDATE` row appeared (this proves
    the trigger works).
-   Re-run the pipeline → open the dashboard → the change appears.

---

## 7. Implementing this for a real company

The demo invents its own data. A real project is mostly about
**replacing the data source, hardening access, and meeting regulatory
identity requirements**. Work in these phases; each one ends with a
checkable result.

### Phase 1 --- Scope and decisions (1--2 weeks, mostly not coding)

Answer these in writing *before* you write code; the answers change the
design.

1.  **Which product group and which delegated act?** (battery now;
    steel, textiles, etc. later.) Get the list of mandatory data
    attributes from the act, not from this repo.
2.  **Granularity**: model-level, batch-level or item-level passports?
    The schema supports one passport per `product\_id` (unique) with
    optional `serial\_number`, `batch\_lot\_number`. Batteries require
    item-level for some categories --- that implies a *lot* of rows
    (millions), which affects Part 7.6.
3.  **Who owns each data element?** Build a table: attribute → source
    system → owner → refresh frequency. Typical sources:

  -----------------------------------------------------------------------
  Passport section                    Typical real source
  ----------------------------------- -----------------------------------
  Product identity, GTIN, serial,     ERP / MES (e.g. SAP)
  batch, facility                     

  Materials / BOM, hazardous          PLM, supplier material declarations
  substances                          

  Carbon footprint, LCA               LCA tool, third-party verified
                                      reports

  Certificates, compliance            QMS, notified bodies, supplier
                                      portal

  Supplier hierarchy                  Procurement / supplier master data

  State of health, cycle count        Battery management system telemetry
  -----------------------------------------------------------------------

4.  **Access tiers**: which fields are public, which need "legitimate
    interest", which are authority-only. The current code serves
    everything publicly (Part 9).
5.  **Environments and ownership**: who runs production, who approves
    data changes, who is the named responsible economic operator?
6.  **Data residency / cross-border rules** and where the workspace must
    live.

**Done when:** you have the attribute-to-source table and a signed-off
access-tier list.

### Phase 2 --- Stand up a clean environment (days)

1.  Create **dev / staging / prod** workspaces or at least separate
    catalogs (`dpp\_dev`, `dpp\_staging`, `dpp` are the defaults in
    `databricks.yml`).
2.  Create a **service principal** for production (`dpp-prod-sp` by
    default, variable `prod\_sp\_name`). The `prod` target runs
    everything `run\_as` that SP. Give it: `CREATE`/`USE` on the target
    catalog, permission to create the pipeline/jobs/apps, and a Postgres
    role on the Lakebase instance (the Bronze ingestion authenticates as
    the identity running the pipeline and needs `SELECT` on `dpp.\*`).
3.  Create one Lakebase instance per environment; size it above `CU\_2`
    for production, and read Databricks' current docs on
    HA/backup/retention for Lakebase before go-live.
4.  Fork the repo into your own Git organisation. Replace
    `.github/workflows/databricks-ci.yml`: it currently hardcodes a
    Databricks-internal workspace URL and a custom runner label
    (`html\_publisher`); both will fail for you. Use OIDC/federated or
    service-principal auth rather than a personal access token.
5.  Run the demo (Part 6) in **dev** unchanged, to confirm the platform
    side works.

**Done when:** the unmodified demo runs in dev under your tenant.

### Phase 3 --- Map your product data onto the schema (1--2 weeks)

Do this **on paper/SQL first**.

1.  For each of the 10 data tables, list every column and decide: mapped
    from source X, derived, or unavailable. Look for *mandatory
    attributes the schema lacks* (battery Annex XIII technical
    parameters such as rated capacity, voltage, expected cycles are
    listed in ROADMAP item 3).
2.  Choose stable **business keys**. `product\_id` must be unique
    forever and ideally never reused. Decide how you will mint
    `passport\_id` --- the demo uses random UUIDs; for real use you will
    probably derive or register an identifier (Phase 6).
3.  Extend the schema **additively**: write
    `ALTER TABLE ... ADD COLUMN IF NOT EXISTS ...` (the existing DDL
    already does this for the state-of-health columns). Never rename or
    drop columns in place on a live passport system.
4.  If you add a table or column, follow the **change checklist** in
    Part 8.1 --- a new column has to be threaded through the seeder
    column list, Silver select list, Gold, apps and tests.

**Done when:** a reviewed mapping document exists and the DDL changes
are merged.

### Phase 4 --- Replace the synthetic data with real ingestion (the core work, 2--6 weeks)

The demo's `00\_setup\_lakebase.py` does
`generate → TRUNCATE ALL → insert`. **Never run that against real data**
(Part 9, gap 1). You need a *loader* that upserts.

**Design decision --- where is the system of record?**

  -------------------------------------------------------------------------------------------
  Option            How                                  Good when          Watch out
  ----------------- ------------------------------------ ------------------ -----------------
  **A. Write        Small ingestion services/jobs read   Source systems are You own
  directly into     ERP/PLM/etc. and                     few; you want the  validation and
  Lakebase**        `INSERT ... ON CONFLICT DO UPDATE`   audit triggers to  idempotency
  (matches this     into `dpp.\*`                        see every change;  
  repo)                                                  apps need fresh    
                                                         data               

  **B. Land in the  Ingest sources into Delta (Auto      Many sources,      The pipeline
  lakehouse first,  Loader / Lakeflow Connect / partner  heavy              direction becomes
  then sync to      tools), clean there, then use        transformations,   *reversed* from
  Lakebase**        Databricks' Lakebase sync /          data engineers     this repo;
                    reverse-ETL capability to publish    already own the    `bronze\_\*`
                    serving tables                       lakehouse          ingestion from
                                                                            Lakebase and the
                                                                            Supplier Portal's
                                                                            writes need a
                                                                            rethink. Check
                                                                            current sync
                                                                            feature limits in
                                                                            the docs.
  -------------------------------------------------------------------------------------------

For most first projects, start with **A** --- it keeps this repo's
pipeline and apps valid as-is.

**Option A recipe.**

1.  Write `src/foundation/04\_load\_from\_sources.py` (new). Reuse the
    helpers in `03\_seed\_lakebase.py`: `\_connect()`, `TABLE\_COLUMNS`,
    `\_init\_codecs`, `prepare\_value`. Replace `truncate + insert` with
    an upsert. Sketch:

``` python
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

Only `product\_passport.product\_id` (and PKs) have unique constraints
today. For child tables (`product\_materials`, `compliance\_records`,
...) either add a natural unique key (e.g.
`(passport\_id, regulation\_name)`) in the DDL, or use "delete children
for this passport then insert" inside one transaction. Pick one and
document it.

2.  Load in the FK order that `TABLE\_LOAD\_ORDER` already defines:
    manufacturer → supplier → passport → children. Insert **suppliers
    parents-first** (tier 1, then 2, then 3), because
    `parent\_supplier\_id` is a foreign key.
3.  Put the connection logic behind an identity that is *not a person*:
    run this as a job under the production service principal.
4.  **Validate before you write.** Reuse the invariants the unit tests
    assert: material percentages per product sum to \~100, no
    supplier-parent cycles, valid enums, FK integrity. Reject or
    quarantine bad records (a `rejected\_records` table is a good
    addition) instead of failing the whole load.
5.  **Schedule it** (a job with a cron schedule, or file-arrival
    trigger) and **make it idempotent** --- a re-run must not create
    duplicates or lose edits.
6.  **Decide conflict rules** between source feeds and Supplier Portal
    edits. Example: certificates are owned by the portal; materials are
    owned by PLM. Otherwise the nightly load will silently overwrite a
    supplier's renewal.
7.  Keep the synthetic generator --- but only for dev/test environments,
    never prod.

**Dynamic data (battery state of health).** Today
`state\_of\_health\_pct`, `cycle\_count`, `dynamic\_data\_updated\_at`
hold only the *latest* value, with no update pipeline and no history. If
you need it, add a time-series table (ROADMAP item 2), stream BMS
telemetry into Delta (Structured Streaming / Zerobus), and write the
latest value to Lakebase. Note that `clean\_and\_conform.py`'s
`silver\_circularity\_info` does **not** currently select those three
columns, so they never reach Silver or Gold; add them if analytics need
them.

**Done when:** a scheduled job loads real records into dev Lakebase; the
viewer shows a real product; re-running the job changes nothing.

### Phase 5 --- Adapt the pipeline and analytics (1--2 weeks)

1.  Run `dpp\_ingestion\_pipeline` against the real data. Watch the
    **expectation metrics** in the pipeline UI: `expect\_or\_drop`
    silently removes rows, so a spike in drops means your data violates
    a rule.
2.  Tune Silver rules to your regulation (e.g. what counts as
    "complete"? currently serial number + facility + country + QR URL).
3.  Replace the Gold risk weights in `compliance\_gap\_analysis.py`
    (30/15/20/10/15/10) with your compliance team's logic. The weights
    are illustrative.
4.  Edit the dashboard **datasets** in
    `resources/dashboards/dpp\_compliance.lvdash.json` if you changed
    the catalog/schema (it hardcodes `dpp\_dev.dpp.`):
    `sed -i 's/dpp\_dev\\.dpp\\./<catalog>.<schema>./g' resources/dashboards/dpp\_compliance.lvdash.json`
5.  Create the Genie space from
    `resources/genie/dpp\_compliance\_explorer.yml` (manual or REST
    API), adding table/column descriptions so it answers *your*
    vocabulary.
6.  Set `development: false` on the pipeline and decide whether to stay
    on `channel: PREVIEW` (the roadmap suggests moving to `CURRENT` once
    supported).

**Done when:** dashboard numbers reconcile against a hand-checked sample
of 20 products.

### Phase 6 --- Identity, QR codes and the EU registry (varies; biggest regulatory risk)

What the demo does: each seeded passport stores a *fictional*
`qr\_code\_url` (`https://dpp.voltcore.se/passport/<uuid>`), but the
viewer's `/api/qr/{id}` ignores it and encodes the app's own
`/passport/<uuid>` URL.

What a real system needs (the repo does **not** implement any of this;
see ROADMAP items 5 and 7):

1.  A **resolvable, permanent URL** you control
    (e.g. `https://dpp.yourcompany.com/p/<id>`) that keeps working for
    the product's lifetime, even if you later move off Databricks Apps.
    Put a stable hostname/reverse proxy in front of the apps; do not
    print the `\*.databricksapps.com` URL on products.
2.  The **unique product identifier scheme** and **data-carrier** format
    required by the applicable standards (the repo cites EN 18219/18220)
    and **registration** in the EU DPP Registry with its issued
    identifier. Track the legal status yourself: dates and implementing
    acts are still moving.
3.  Store the registry identifier and registration status on the
    passport (new columns) and make the QR resolve through it.
4.  A **backup/continuity** plan (ROADMAP item 6): the passport must
    stay available even if your company ceases trading.

**Done when:** legal/compliance sign off the identifier and carrier
design.

### Phase 7 --- Security and access control (do not skip)

The code is demo-grade here. Fix these *before* real data goes in:

1.  **Replace the `GRANT … TO PUBLIC`** at the end of
    `01\_setup\_lakebase\_schema.sql` with per-role grants. Viewer's SP:
    `USAGE` on schema + `SELECT` on tables. Supplier-portal SP: `SELECT`
    plus `INSERT/UPDATE` only on the tables it writes
    (`product\_origin`, `supplier`). The ingestion SP: write on all
    `dpp.\*`. Find each app's role name from the injected `PGUSER`.
2.  **Supplier authorization.** The portal has **no mapping from the
    logged-in user to a supplier** --- `/supplier/{supplier\_id}` and
    the renew POST trust the ID in the URL. Anyone who can open the app
    can edit any supplier. Add a
    `supplier\_user(supplier\_id, user\_email)` table, read the identity
    from the `X-Forwarded-Email` header Databricks Apps adds, and reject
    requests where the user is not linked to that supplier. Consider
    whether suppliers should instead be external users, which usually
    needs a different front door than workspace SSO.
3.  **Real audit identity.** The trigger stores `current\_user` as
    `changed\_by`. Because the apps connect as their service principal,
    every audit row says the app's SP, not the human. Pass the user
    through (e.g. `SET LOCAL app.user\_email = ...` in the same
    transaction and have the trigger read
    `current\_setting('app.user\_email', true)`).
4.  **Tiered public/restricted/authority access.** Today
    `/api/passport/{id}` returns *every* field to anyone who can reach
    the app. Implement an `access\_tier` classification, split the
    response into tiers, add authentication for the non-public ones, and
    consider Unity Catalog row/column controls for the lakehouse copy
    (ROADMAP item 1). Do not treat unit costs, supplier identities or
    due-diligence documents as public by default.
5.  **Public access path.** Databricks Apps sit behind workspace login.
    A consumer scanning a QR is not a workspace user. Decide how public
    passport data is exposed (a separate public-facing read-only
    service/API gateway in front of a *public-tier* dataset is the
    safest pattern) rather than opening the internal app.
6.  **Input validation.** A malformed (non-UUID) ID causes a Postgres
    error that surfaces as a 500; validate UUIDs up front and return 404
    (ROADMAP "Engineering hardening").
7.  **Privacy.** Leave `ACCESS\_LOG\_ENABLED=false` unless legal agrees;
    it stores user emails (personal data).
8.  **AI text.** LLM-generated insights are advisory text on a regulated
    document. Label them clearly as AI-generated, never show them as
    certified data, and keep them out of the legally binding passport
    view unless reviewed.

**Done when:** a written threat model exists and each item above is
closed or explicitly accepted.

### Phase 8 --- Make the apps production-grade (1--2 weeks)

1.  Edit `app.yaml` env values to match your instance/catalog by hand
    (no variable substitution).
2.  Extract the duplicated DB code in both apps into a shared module
    (ROADMAP item).
3.  Replace `asyncio.get\_event\_loop()` with `asyncio.to\_thread` in
    the insights path (ROADMAP item).
4.  Add caching for hot passports and think about load: each page view
    runs several SQL queries; a viral QR scan needs a pool size and
    Lakebase capacity plan (`max\_size=10` per app instance today).
5.  Pin dependency versions in `requirements.txt` (currently open-ended
    `>=`).
6.  Add integration tests (the current tests only cover the data
    generator). At minimum test: the viewer returns 200 for a seeded
    passport and 404 for an unknown one; the renew POST writes an audit
    row.

### Phase 9 --- Operations and rollout

1.  **CI/CD:** on pull request run `pytest`,
    `databricks bundle validate`, and a deploy to dev; on tag deploy to
    staging then prod via the `prod` target (`mode: production`, runs as
    the SP).
2.  **Scheduling:** schedules are `PAUSED` by default in
    `dpp\_jobs.yml`. Unpause the loader, pipeline and
    `precompute\_insights` with a deliberate order (load → pipeline →
    insights).
3.  **Cost control:** `precompute\_insights` calls the LLM for **every**
    passport on every run and overwrites the table. For large catalogs,
    only regenerate passports whose `updated\_at` changed, and check
    which model/endpoint your workspace allows (`AI\_MODEL`, default
    `databricks-meta-llama-3-3-70b-instruct`; endpoint availability
    varies).
4.  **Monitoring:** alert on job/pipeline failures, on the `/health`
    endpoint (it returns 503 if the DB is down), and on expectation-drop
    counts.
5.  **Backup and retention:** document RPO/RTO for Lakebase and for the
    Delta tables; pair with the continuity obligation in Phase 6.
6.  **Pilot:** go live with one product line, one supplier cohort and an
    internal compliance reviewer before opening it up.

### 7.6 Scaling notes

-   500 passports → trivial. Millions of item-level passports changes
    three things: the **full-snapshot Bronze read** becomes expensive
    (move to incremental change capture, Lakebase sync or Lakeflow
    Connect, as the README's production-hardening section suggests); the
    Gold `collect\_list` aggregation gets heavy; and
    `precompute\_insights`' driver-side `collect()` plus `executemany`
    upsert will not scale (batch it, or write through a sync).
-   Keep per-request lookups indexed by
    `passport\_id`/`product\_id`/`gtin` (indexes exist). Add indexes for
    any new lookup key (e.g. registry identifier).

---

## 8. Day-to-day developer tasks (recipes)

### 8.1 Add a column or table (the full checklist)

For a new column on, say, `environmental\_impact`:

1.  `01\_setup\_lakebase\_schema.sql` --- add the column in
    `CREATE TABLE` **and** an `ALTER TABLE … ADD COLUMN IF NOT EXISTS …`
    (so existing databases upgrade).
2.  `03\_seed\_lakebase.py` --- add it to
    `TABLE\_COLUMNS\["environmental\_impact"]`.
3.  `02\_synthetic\_data\_generator.py` --- emit it in
    `\_generate\_impact`.
4.  `silver/clean\_and\_conform.py` --- add it to the `select(...)`
    (**Silver lists columns explicitly; anything not listed is
    dropped**).
5.  Gold files that need it --- add to the select/aggregation.
6.  Apps --- query/template (`templates/passport.html`).
7.  `tests/test\_synthetic\_data.py` --- add assertions.
8.  Re-run: setup job (or your loader) → pipeline → apps.

If the table is **new**, also add it to `TABLE\_LOAD\_ORDER`, the
generator's `tables` dict in `00\_setup\_lakebase.py`, and (if it has
`passport\_id`) to the audit-trigger table list in the DDL.

### 8.2 Add a new industry profile (e.g. steel)

1.  In `02\_synthetic\_data\_generator.py` build: a product catalog,
    material DB, certifications, carbon ranges, lifecycle splits,
    regulations, facilities, component names, durability ranges and
    disposal components, using the `battery` or `furniture` entries as
    templates.
2.  Add the new key to `PROFILES`.
3.  Select it at **deploy time**:
    `databricks bundle deploy -t dev --var="profile=steel"` (passing
    `--var` only to `bundle run` has no effect, because the value is
    baked into the job parameters at deploy).
4.  Run `pytest` --- the tests check structure, FK integrity and
    distributions for the profile you pass. Note
    `00\_setup\_lakebase.py` defaults to `furniture` when run directly
    *without* `--profile` or `DPP\_PROFILE`; the bundle always passes
    the profile explicitly.

### 8.3 Change which catalog/schema you deploy to

``` bash
databricks bundle deploy -t dev --var="catalog=my\_catalog" --var="schema=dpp" \\
  --var="dashboard\_warehouse\_id=<id>"
sed -i 's/dpp\_dev\\.dpp\\./my\_catalog.dpp./g' resources/dashboards/dpp\_compliance.lvdash.json
```

Then update the Genie spec tables by hand.

### 8.4 Run the apps locally

``` bash
cd src/apps/passport\_viewer
pip install -r requirements.txt
export LAKEBASE\_INSTANCE=dpp-passport LAKEBASE\_USER=<pg user> LAKEBASE\_PASSWORD=<password>
export DATABRICKS\_HOST=https://<workspace>   # needed for the live AI path
uvicorn main:app --reload --port 8000
```

Without `PGHOST` the app uses the native-login fallback; that requires a
Lakebase instance created with `--enable-pg-native-login` and a password
you created yourself. This is a dev convenience only.

### 8.5 Inspect and debug the data

-   Operational data: connect to Lakebase with any Postgres client;
    schema is `dpp`.
-   Analytics: query `<catalog>.<schema>.gold\_\*` in a SQL warehouse.
-   Audit trail:
    `SELECT \* FROM dpp.passport\_audit\_log ORDER BY changed\_at DESC LIMIT 20;`
-   Why did a passport vanish from Silver? Check pipeline expectation
    metrics for `expect\_or\_drop` rules.

---

## 9. Known gaps and traps in the current code

These are things I confirmed by reading the source. None is a bug in the
*demo*, but each one is a trap in a *real* deployment.

  --------------------------------------------------------------------------------------------------------------------------------
  \#             Gap                             Where                               Impact                      What to do
  -------------- ------------------------------- ----------------------------------- --------------------------- -----------------
  1              **The setup job wipes all       `03\_seed\_lakebase.py`, called by  Re-running it on a real     Use it only in
                 tables.** `truncate\_tables()`  `00\_setup\_lakebase.py`            system **destroys all       dev. Build a
                 runs `TRUNCATE … CASCADE` on                                        passport data and audit     separate upsert
                 every table (including                                              history**.                  loader (Phase 4).
                 `passport\_audit\_log`) before                                                                  Consider removing
                 inserting.                                                                                      the job from the
                                                                                                                 prod target.

  2              **No tiered access.** All data  `passport\_viewer/main.py`          Cannot meet                 Phase 7 item 4;
                 is served openly, including the                                     restricted/authority-only   ROADMAP #1.
                 JSON API.                                                           requirements.               

  3              **Supplier Portal has no        `supplier\_portal/main.py`          Any user with app access    Phase 7 item 2.
                 user→supplier authorization.**                                      can edit any supplier's     
                 Supplier IDs come from the URL.                                     certificates.               

  4              **Audit `changed\_by` is the    trigger uses `current\_user`        The audit trail cannot say  Phase 7 item 3.
                 app's service principal**, not                                      *who* made a change.        
                 the human.                                                                                      

  5              **`GRANT … TO PUBLIC`.**        end of the DDL                      Every Postgres role can     Least-privilege
                                                                                     read and write everything.  grants.

  6              **Fictional QR URL stored in    generator `qr\_base\_url` vs        QR codes would not survive  Phase 6.
                 data; QR image uses a different `/api/qr/{id}`                      a change of host.           
                 URL.** No registry ID / URI.                                                                    

  7              **State-of-health fields stop   `clean\_and\_conform.py`            Not in Gold, dashboard, or  Add columns to
                 at Lakebase.** Silver's                                             AI prompts for the          Silver/Gold if
                 circularity select omits them.                                      lakehouse path.             needed.

  8              **Bronze is a full snapshot**,  `ingest\_lakebase\_to\_bronze.py`   Cost grows with data size.  Incremental
                 though comments say "streaming                                                                  capture (Phase
                 tables".                                                                                        7.6).

  9              **Hardcoded values DABs won't   listed files                        Silent mismatches after     Edit by hand, or
                 substitute**: catalog in the                                        renaming.                   template with a
                 dashboard JSON; instance/DB                                                                     script in CI.
                 name in both `app.yaml` files.                                                                  

  10             **`dashboard\_warehouse\_id`    `databricks.yml`                    Dashboard deploy will fail  Always pass
                 has a hardcoded default** (a                                        or point at a warehouse you `--var`.
                 specific workspace's ID)                                            cannot use.                 
                 although a comment mentions a                                                                   
                 placeholder.                                                                                    

  11             **CI workflow is tied to a      `.github/workflows/`                CI fails in your org.       Replace (Phase
                 Databricks-internal                                                                             2).
                 workspace/runner.** README also                                                                 
                 mentions a `ci.yml` that does                                                                   
                 not exist (files are                                                                            
                 `databricks-ci.yml`,                                                                            
                 `publish.yaml`).                                                                                

  12             **Setup script's default        foundation scripts                  Surprising data when run    Always pass
                 profile differs by entry                                            locally without             `--profile`.
                 point.** Generator default is                                       `--profile`.                
                 `battery`;                                                                                      
                 `00\_setup\_lakebase.py`'s                                                                      
                 argparse default is `furniture`                                                                 
                 unless `DPP\_PROFILE` is set.                                                                   

  13             **Invalid IDs cause 500s**      both apps                           Poor errors; log noise.     Validate UUIDs.
                 (Postgres UUID parse error).                                                                    

  14             **Certificate renewal updates   `renew\_certification`              Compliance status may lag   Decide the
                 only `product\_origin`** (and                                       the supplier's action.      business rule;
                 touches                                                                                         extend the
                 `supplier.updated\_at`). It                                                                     transaction.
                 does not update                                                                                 
                 `compliance\_records` or                                                                        
                 `supplier.last\_audit\_date`.                                                                   

  15             **Pipeline is                   `dpp\_pipelines.yml`                Not production posture.     Phase 5 item 6.
                 `development: true` on the                                                                      
                 `PREVIEW` channel.**                                                                            

  16             **AI cost and trust.** Every    `precompute\_insights.py`           Cost; hallucinated          Phase 9 item 3;
                 passport is sent to an LLM on                                       "insights" on a regulated   Phase 7 item 8.
                 each run; output is not                                             record.                     
                 validated beyond JSON schema.                                                                   

  17             **Only the data generator has   `tests/`                            Regressions go unnoticed.   Phase 8 item 6.
                 tests** (42 pass). No tests for                                                                 
                 apps, pipelines or SQL.                                                                         

  18             **`reyden\_toggle.py` is dead   `src/serving/`                      Confusion.                  Ignore or delete.
                 code** (not imported anywhere).                                                                 

  19             **Regulatory content is a       README, docs, generator             Law changes; the repo says  Get legal review.
                 snapshot** (dates, article                                          to confirm against the      
                 numbers, phase-in rules).                                           applicable act.             
  --------------------------------------------------------------------------------------------------------------------------------

---

## 10. Production readiness checklist

**Data and model**

-   [ ] Attribute-to-source mapping approved; mandatory attributes for
    your product group covered
-   [ ] Real-data loader is an idempotent upsert; the destructive setup
    job is not in the prod target
-   [ ] Validation + quarantine for bad source records
-   [ ] Source-vs-portal ownership rules documented

**Security and privacy**

-   [ ] Least-privilege Postgres grants (no `PUBLIC`)
-   [ ] Supplier user→supplier authorization implemented and tested
-   [ ] Audit rows capture the real human
-   [ ] Public vs restricted vs authority tiers designed and enforced
-   [ ] Public access path separate from the internal workspace apps
-   [ ] Access logging decision made with legal; AI text labelled

**Identity and regulation**

-   [ ] Permanent resolvable URL and data carrier agreed with compliance
-   [ ] EU Registry registration/identifier handled; continuity/backup
    plan

**Platform**

-   [ ] `prod` deploys under a service principal; CI replaced and green
-   [ ] Pipeline in production mode; schedules unpaused in the right
    order
-   [ ] Monitoring and alerts; backup/RPO/RTO documented; capacity
    tested
-   [ ] Dependency versions pinned; integration tests added

---

## 11. Troubleshooting

  ----------------------------------------------------------------------------------------------
  Symptom                 Likely cause                 Fix
  ----------------------- ---------------------------- -----------------------------------------
  `bundle deploy`         Default                      Pass
  complains about the     `dashboard\_warehouse\_id`   `--var="dashboard\_warehouse\_id=<id>"`
  warehouse / dashboard   is not yours                 (`databricks warehouses list`)

  `auth login` hangs or   Old `databricks` CLI first   `databricks version`; use ≥ 0.278
  old behaviour           on `PATH`                    

  Pipeline fails with     Pipeline ran before the      Run the setup job first (DDL is
  missing table/column    setup job, or schema is      idempotent)
                          outdated                     

  Pipeline can't connect  Running identity has no      Grant the identity access to the instance
  to Lakebase             Postgres role/permission     and `SELECT` on `dpp.\*`

  App shows DB errors /   Lakebase resource not bound, Check `resources/apps.yml` binding; run
  `/health` is 503        or schema not created        the setup job

  "No Lakebase            No `PGHOST` (binding         Deploy via the bundle, or set the
  credentials found"      missing) and no              native-login fallback
                          `LAKEBASE\_PASSWORD`         

  AI Insights tab says    Cache empty and live model   Run `precompute\_insights`; verify the
  "generated on a         call failed                  `AI\_MODEL` endpoint exists and is
  schedule"                                            permitted

  Dashboard empty         Dashboard datasets point at  Rewrite catalog in the `.lvdash.json`
                          a different catalog          

  Rows missing in         `expect\_or\_drop` rule      Inspect pipeline expectation metrics
  Silver/Gold             removed them                 

  App links point at the  `app.yaml` values not        Edit `LAKEBASE\_INSTANCE` /
  wrong place after       substituted                  `LAKEBASE\_DATABASE\_NAME`
  rename                                               

  Seed fails on re-run:   Data already present and not Use upsert, or truncate in dev only
  `uq\_product\_id`       truncated (custom loader)    
  ----------------------------------------------------------------------------------------------

---

## 12. Glossary

-   **DPP** --- Digital Product Passport.
-   **ESPR** --- EU Ecodesign for Sustainable Products Regulation
    (2024/1781).
-   **Economic operator** --- the manufacturer/importer legally
    responsible for the passport.
-   **ABAC** --- attribute-based access control.
-   **LCA** --- life-cycle assessment (how carbon/energy/water are
    computed).
-   **SVHC / REACH** --- EU chemicals rules (substances of very high
    concern).
-   **SoH** --- battery state of health.
-   **BMS** --- battery management system.
-   **OLTP / OLAP** --- transactional vs analytical workloads.
-   **CDC** --- change data capture (incremental sync of changes).
-   **DAB** --- Databricks Asset Bundle.
-   **SDP / DLT** --- Spark Declarative Pipelines / Delta Live Tables.
-   **SP** --- service principal (a machine identity).
-   **UC** --- Unity Catalog.
-   **URI (EU registry)** --- the Registry-issued unique identifier for
    a passport (not a generic web URI).

---

*This guide describes the repository as uploaded (v1.0.0). Regulatory
dates and product-group scope change; the operator remains responsible
for confirming requirements against the acts that apply.*
