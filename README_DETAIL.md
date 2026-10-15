# Digital Product Passport (DPP) Solution Accelerator

A modular, reusable solution accelerator demonstrating how manufacturers can build
EU-compliant Digital Product Passports on the **Databricks Data Intelligence Platform**
with **Lakebase** as the operational backbone.

## Why This Matters

The EU Ecodesign for Sustainable Products Regulation (ESPR) mandates Digital Product
Passports, with batteries first (mandatory 18 Feb 2027), then iron & steel,
textiles, aluminium, tyres, furniture and further groups through the end of the
decade. Manufacturers must provide machine-readable, per-product lifecycle data
covering origin, materials, environmental impact, compliance, circularity, and
disposal.

The EU system is **decentralized**: the Commission runs a thin registry/portal +
standards, but the passport data is held and served by each **economic operator**.
This accelerator is that operator-side backend. See
[docs/positioning.md](docs/positioning.md) for the regulatory driver, the EU
architecture, and where this fits.

> **Scope of this accelerator.** v1.0 is a reference architecture and runnable
> demo, not a turnkey compliance product. In particular it serves passport data
> openly and does not yet implement the regulation's tiered access rights
> (public / restricted / authority-only) or DPP Registry registration and the
> Registry-issued unique identifier. Those are on the [roadmap](ROADMAP.md). The
> economic operator remains responsible for meeting the applicable legal
> requirements.

## Why Databricks

A DPP is not one workload — it is operational lookups, batch analytics, AI, and
secure sharing over the same governed data. Databricks lets a manufacturer build
the whole operator-side backend on **one platform**, instead of stitching
together a transactional database, a warehouse, an app host, and an AI service:

- **Lakebase** serves sub-second QR/passport lookups and supplier writes — OLTP
  next to the lakehouse, no separate operational database to integrate.
- **Unity Catalog** governs every layer (lineage, access, audit) — the basis for
  the public vs. restricted data tiers the regulation requires.
- **Spark Declarative Pipelines, AI Functions, Genie, Databricks Apps** cover
  ingestion, compliance analytics, AI insights, and the consumer/supplier apps.
- **Delta Sharing** addresses cross-company value-chain and cross-border data
  exchange without copying data.

One governed copy of the data, many workloads — which is hard to match by
assembling point products.

## Architecture

The solution uses a hybrid OLTP + analytics architecture:

- **Lakebase (PostgreSQL)** -- Operational store for per-product passport lookups,
  supplier data entry, compliance updates, audit trail writes, and the cached
  AI-insights serving table.
- **Spark Declarative Pipelines (SDP)** -- Snapshot ingestion from Lakebase into Bronze
  streaming tables, Silver conformance layer, and Gold business aggregates
  (passport-complete, multi-tier supplier traceability, carbon, compliance gaps,
  material composition trends, circularity metrics).
- **Databricks Apps** -- Consumer-facing passport viewer and supplier self-service portal,
  each bound to the Lakebase instance as a resource (service-principal auth).
- **AI/BI Dashboard** -- 4-page compliance and sustainability dashboard.
- **Genie Space** -- Natural language SQL exploration for compliance officers.
- **Foundation Model API** -- AI passport insights, pre-computed nightly via
  `ai_query` into `gold.dpp_passport_insights` and served from Lakebase cache.

![Architecture](docs/architecture.png)

> Diagram source: [`docs/architecture.svg`](docs/architecture.svg) (scalable vector,
> official Databricks icons; the PNG is rendered from it). A simplified
> [`docs/architecture.mmd`](docs/architecture.mmd) Mermaid version is also included.

## Modules

| Module | Status | Description |
|--------|--------|-------------|
| **1. Foundation** | Complete | Lakebase schema (12 tables incl. multi-tier `parent_supplier_id` + insights cache), deterministic synthetic data (500 products, 50 suppliers across 3 tiers), self-contained setup job (apply schema + seed via runtime OAuth) |
| **2. Passport Viewer** | Complete | Consumer-facing app with 7 tabs (Origin incl. upstream supplier chain, Materials, Environment, Compliance, Circularity, Disposal, AI Insights), QR codes, category filters, search, pagination |
| **3. Dashboard & Genie** | Complete | 4-page AI/BI compliance dashboard, Genie Space with curated questions |
| **4. Supplier Portal** | Complete | Supplier self-service app with cert renewal (Lakebase writes), audit log, search/filter |
| **5. AI/ML Enrichment** | Complete | AI passport insights (gap detection, carbon analysis, compliance alerts, circularity recommendations) pre-computed nightly via `ai_query` and served from Lakebase cache, with a live Foundation Model fallback |

## Databricks Products Showcased

| Product | Role |
|---------|------|
| **Lakebase** | Operational passport store, supplier writes, audit trail |
| **Spark Declarative Pipelines** | Data ingestion and transformation (Bronze/Silver/Gold) |
| **Unity Catalog** | Data governance, lineage, DQ expectations |
| **Databricks Apps** | Passport Viewer + Supplier Portal |
| **AI/BI Dashboards** | Compliance and sustainability reporting |
| **Genie** | Natural language querying for compliance officers |
| **Foundation Model API** | Real-time AI passport analysis |
| **DABs** | Deployment and CI/CD |

## Adapting to your industry

The schema, SDP pipeline, apps, dashboard, and Genie space are **domain-agnostic** —
the six DPP categories (origin, materials, environmental impact, compliance,
circularity, disposal) are exactly what the EU ESPR mandates for *every* product
group. Only the **synthetic data** is industry-specific, and it is driven by a
selectable **profile**:

| Profile | Manufacturer | Categories | Example regulations |
|---------|--------------|------------|---------------------|
| `battery` (default) | VoltCore Energy AB | EV / Industrial / LMT / Portable Battery | **EU Batteries Regulation 2023/1542**, ESPR, UN 38.3; carbon footprint / recycled content / due diligence phased in post-2027 |
| `furniture` | NordicForm AB | Furniture, Textiles, Lighting, Storage | ESPR, REACH, RoHS, Timber |

The **battery** profile is the default: the battery passport is the first
mandatory EU Digital Product Passport (18 Feb 2027), so it is the highest-impact
lead. It models item-level passports, multi-tier `mine → cell → module → pack`
traceability, and battery **state of health** (dynamic data). Switch to
`furniture` with `--var profile=furniture` at deploy time.

Pick a profile at **deploy** time — the profile is baked into the setup job's
parameters when the bundle is deployed, so passing `--var` to `bundle run`
alone has no effect:

```bash
databricks bundle deploy -t dev --var="profile=battery" --var="dashboard_warehouse_id=<warehouse-id>"
databricks bundle run synthetic_data_setup -t dev
```

To add your own industry (medical devices, textiles, electronics, …), add a new
entry to `PROFILES` in `src/foundation/02_synthetic_data_generator.py` — product
catalog, materials, certifications, carbon ranges, lifecycle splits, regulations,
disposal components — and select it with `--profile`. No schema or app changes
needed. The [EU battery DPP requirements](https://single-market-economy.ec.europa.eu/events/digital-product-passport-batteries-2026-05-27_en)
map directly onto the existing six categories.

> Deeper battery-passport features (tiered access control, daily State-of-Health
> dynamic data, full Annex XIII technical parameters, value-chain aggregation) are
> intentionally out of scope for v1.0 and captured in [ROADMAP.md](ROADMAP.md),
> each mapped to the relevant EU requirement and Databricks capability.

## Step-by-step setup

Follow these steps in order to set up the project in a Databricks workspace.

### 1. Prerequisites

Before you start, confirm you have:

- Databricks workspace access with permission to create a Lakebase instance
- Unity Catalog enabled and a catalog available for pipeline output
- A SQL warehouse to use for the dashboard
- Databricks CLI version 0.278.0 or later
- Python 3.10 or later
- Git installed locally

> If a system-installed `databricks` CLI is older than the required version, update your PATH or call the newer binary explicitly (for example `/usr/local/bin/databricks`).

### 2. Clone the repository

```bash
git clone <repo-url>
cd dpp-solution-accelerator
```

### 3. Authenticate to Databricks

Use OAuth so the bundle can deploy and run against your workspace without storing secrets in the repo.

```bash
databricks auth login --host https://<your-workspace>.cloud.databricks.com --profile dpp
export DATABRICKS_CONFIG_PROFILE=dpp
```

Optional verification:

```bash
databricks version
```

### 4. Create the Lakebase instance

This project expects a Lakebase database instance to exist before the app bundle is deployed. Create it once from the CLI:

```bash
databricks database create-database-instance dpp-passport --capacity CU_2
```

The default database name is `databricks_postgres`, which is already used by the bundle configuration. If you need a custom database name, update the `lakebase_database` variable before deployment.

### 5. Identify the SQL warehouse

The dashboard needs a workspace warehouse ID.

```bash
databricks warehouses list -o json
```

Copy the warehouse ID you want to use. You will pass it as `dashboard_warehouse_id` in the next deployment step.

### 6. Deploy the bundle

This step provisions the jobs, pipeline, dashboard, and both Databricks Apps defined under `resources/`.

```bash
databricks bundle deploy -t dev --var="dashboard_warehouse_id=<warehouse-id>" --var="profile=battery"
```

Notes:

- `profile` defaults to `battery`, but you can switch to `furniture` if needed.
- The bundle reads `databricks.yml` and the resource files under `resources/`.
- The apps are bound to the Lakebase instance as a resource during deployment, so there is no separate `databricks apps create` step.

### 7. Run the setup job

After the bundle is deployed, run the setup job to create the schema and seed Lakebase with the synthetic DPP dataset.

```bash
databricks bundle run synthetic_data_setup -t dev
```

This job applies the foundation schema, generates synthetic product and supplier data, and writes it to the configured Lakebase database.

### 8. Run the ingestion pipeline

Run the DPP ingestion pipeline to load the Bronze, Silver, and Gold tables used by the apps and dashboard.

```bash
databricks bundle run dpp_ingestion_pipeline -t dev
```

### 9. Start the two apps

Once the data is in place, start the consumer and supplier apps.

```bash
databricks bundle run dpp_passport_viewer -t dev
databricks bundle run dpp_supplier_portal -t dev
databricks apps list
```

Use the returned app URLs to open the viewer and supplier portal.

### 10. Optional: precompute AI insights

If you want the Passport Viewer's AI Insights tab to show cached results immediately, run the optional insights job.

```bash
databricks bundle run precompute_insights -t dev
```

Without this step, the app still works, but the AI Insights area will show that it is waiting for scheduled generation.

### 11. Optional: create the Genie space

Genie is not deployed automatically by the bundle. The project includes the curated Genie spec in `resources/genie/`.

Open the Databricks UI and create a Genie space manually, then attach the relevant Gold and Silver tables and paste in the description/questions from the bundle spec.

### 12. Validate the installation

After the setup job and pipeline have completed successfully:

- Open the passport viewer app URL and confirm the product data loads.
- Open the supplier portal and confirm supplier records and product updates are visible.
- Open the dashboard and confirm it renders using the configured SQL warehouse.

### Deployment notes

- SQL warehouse: `dashboard_warehouse_id` must be set on every deployment. There is no universal default.
- Catalog: the bundle writes tables to `${var.catalog}` (default `dpp_dev`). If you need a different catalog, set it with `--var="catalog=<your_catalog>"`.
- Lakebase database: the default `databricks_postgres` is usually sufficient. Override it only if you created a custom Lakebase database.
- App authentication: the apps bind to the Lakebase instance as a Databricks resource, so they use service-principal-based access without manual secrets.
- Existing deployments: rerun the setup job before the ingestion pipeline if you are upgrading the schema.

> If you cannot use resource-based database binding in a local or restricted environment, use the native-login fallback described in the main README, but the default bundle flow is the recommended setup path.

## Project Structure

```
dpp-solution-accelerator/
  databricks.yml              DAB root config + variables + targets
  resources/                  DAB resources: pipelines, jobs, apps, dashboards
  src/
    foundation/               00 setup (schema+generate+seed), 01 DDL, 02 generator, 03 seeder
    pipelines/                SDP code: bronze / silver / gold + precompute_insights
    serving/                  Reyden toggle (reference pattern)
    apps/
      passport_viewer/        Consumer-facing passport app (FastAPI + Jinja2)
      supplier_portal/        Supplier self-service portal (FastAPI + Jinja2)
  tests/                      Unit tests (pytest)
  docs/                       architecture.svg/.png (diagram) + architecture.mmd (Mermaid)
  LICENSE.md  NOTICE.md  SECURITY.md  CONTRIBUTING.md  CHANGELOG.md  ROADMAP.md
```

## Multi-tier supply chain traceability

Suppliers form a hierarchy via `supplier.parent_supplier_id`: a tier-2 supplier's
parent is the tier-1 (direct) supplier it delivers to, and tier-3 rolls up to
tier-2. This models the tier-1 → tier-N question regulators and OEMs ask
("where does this component ultimately come from?").

- **Schema**: self-referential FK on `dpp.supplier`.
- **Gold**: `gold_supplier_traceability` flattens each supplier's full ancestry
  (bounded iterative parent walk).
- **App**: the Passport Viewer's Origin tab renders the upstream chain per
  component via a recursive CTE.

## Production hardening

This accelerator uses **snapshot ingestion** (full Lakebase → Bronze read per
pipeline run) for simplicity. For production scale, replace it with continuous
change data capture:

- **Lakebase Sync / reverse ETL** to keep the operational store and the lakehouse
  aligned without full snapshots.
- **Lakeflow Connect (CDC)** for incremental change capture from the source systems.
- Use **secrets + least-privilege Postgres roles** (the apps already support a
  native-login fallback), schedule the pre-compute job, and pin model/runtime versions.
- Consider **Reyden** (high-concurrency serving, optional toggle in `src/serving/`)
  for sub-second QR lookups at scale.

## App access logging (optional)

To track who opens the apps internally, set `ACCESS_LOG_ENABLED=true` in each
app's `app.yaml`. When on, both apps record the accessing user (from the
Databricks Apps SSO `X-Forwarded-*` headers), path, and timestamp to
`dpp.app_access_log`. It is **off by default**: user emails are personal data
(GDPR), so enable it only where appropriate and with the right notices. View
the data by querying the Lakebase table directly, or surface it in the AI/BI
dashboard. Note this logs *authenticated app usage*; requests from users who
lack access are handled by Databricks before reaching the app and are not
recorded here.

## EU ESPR Timeline

Dates below reflect the European Commission's
[DPP overview](https://single-market-economy.ec.europa.eu/single-market/digital-product-passport_en)
(product-group dates are set by the individual ESPR delegated / sector acts and
continue to move; treat them as indicative and confirm against the act that
applies to you).

- **20 July 2026:** EU central DPP Registry operational.
- **Q4 2026:** Iron & steel ESPR delegated act (one of the earliest groups).
- **18 Feb 2027:** Mandatory battery passports (EV, light-transport, industrial).
- **Q2 2027:** Implementing Act for Unique Identifiers (the Registry-issued URI).
- **Q2 2027:** Construction products (Construction Products Regulation).
- **2027 (H2):** Textiles, aluminium, tyres (ESPR delegated acts).
- **2028:** Furniture. **2029:** Mattresses; recycled-content rules.
- Toys and detergents come in through their own legislation (e.g. Toys Safety
  Regulation 2025/2509), not the ESPR delegated acts above.

**Regulatory references** (external, authoritative sources — for context only):

- European Commission — [Digital Product Passport overview](https://single-market-economy.ec.europa.eu/single-market/digital-product-passport_en)
- Ecodesign for Sustainable Products Regulation (ESPR) — [Regulation (EU) 2024/1781](https://eur-lex.europa.eu/eli/reg/2024/1781/oj)
- EU Batteries Regulation — [Regulation (EU) 2023/1542](https://eur-lex.europa.eu/eli/reg/2023/1542/oj)

> This accelerator is an independent Databricks solution accelerator. It is **not
> affiliated with, endorsed by, or produced in association with** the European
> Commission or any EU body. Links above are provided as references to the
> primary regulation and official guidance; always confirm requirements against
> the acts that apply to you.

## Target Audience

Any organization that must place an EU Digital Product Passport on the market,
and the Databricks field and partner teams who support them:

- **Manufacturers, brands, and importers** across all in-scope ESPR categories
  (batteries first, then textiles, furniture, tyres, electronics, iron and steel,
  aluminium), whether based in the EU or selling into it.
- **Solutions Architects, Delivery Solutions Architects, and partners** building
  DPP solutions on the Databricks Data Intelligence Platform.

Originally incubated in the Databricks Manufacturing Community of Practice
(MFG CoP), now applicable to any EU DPP operator-side use case worldwide.

## Dependencies and licenses

All third-party libraries are permissively licensed (Apache-2.0 / BSD / MIT).
The data generator (Faker) and all pipeline/app code run on Databricks
Serverless + Unity Catalog.

| Library | Purpose | License |
|---------|---------|---------|
| faker | Synthetic data generation | MIT |
| fastapi | App web framework | MIT |
| uvicorn | ASGI server | BSD-3-Clause |
| jinja2 | App templating | BSD-3-Clause |
| qrcode[pil] | QR codes (Pillow) | BSD / HPND |
| requests | HTTP (OAuth token mint) | Apache-2.0 |
| databricks-sdk | Databricks API | Apache-2.0 |
| pyspark / Lakeflow | Pipelines (runtime-provided) | Apache-2.0 |
| asyncpg | Lakebase (PostgreSQL) connectivity | Apache-2.0 |

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). Contributions are accepted pursuant to a CLA.

## Maintenance

- **Maintainer:** Daniel Dahlin ([@danielvelasquez123-dbx](https://github.com/danielvelasquez123-dbx)).
- **Support:** please open a [GitHub Issue](../../issues) for bugs, questions, or
  enhancement requests. Issues are triaged by the maintainer.
- **Validation cadence:** the end-to-end deploy (bundle deploy, pipelines, apps,
  dashboard, Genie) is re-validated **quarterly** and whenever a new Databricks
  Runtime LTS is released, on a fresh workspace with Serverless + Unity Catalog.
- **Automated checks:** unit tests and a byte-compile run on every push/PR and on
  a **weekly schedule** (see [`.github/workflows/ci.yml`](.github/workflows/ci.yml))
  to catch dependency or platform drift between manual validations.
- **Deprecation:** if the accelerator can no longer be maintained, a notice will be
  added here and to [CHANGELOG.md](CHANGELOG.md) before the repository is archived.

## License

&copy; 2026 Databricks, Inc. Licensed under the Databricks License — see
[LICENSE.md](LICENSE.md). Use is permitted in connection with your use of the
Databricks Platform Services. See [CHANGELOG.md](CHANGELOG.md) for release notes.
