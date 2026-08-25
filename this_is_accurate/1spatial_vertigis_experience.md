# 1Spatial / VertiGIS — Senior Software Engineer (2023 – Present)

Detailed notes on the architecture and implementation work behind **TMPA (1Streetworks)**, a geospatial traffic management planning platform, for CV deep-dives, interview prep and portfolio use. Source: internal Confluence documentation (System Architecture, and per-component pages).

---

## Product overview

TMPA (also known as 1Streetworks) is a web application used by traffic management planners to design, automate and manage traffic management (TM) plans for road works across the UK. It combines a geospatial mapping UI, an automated "auto-draft" plan generation engine driven by a business-rules engine, and a set of supporting microservices for map rendering, PDF generation, routing and reference-data management.

---

## System architecture

The system is built as a set of independently deployable, containerized services, coordinated around a central application and a shared set of PostgreSQL/PostGIS databases. Components fall into a few groups:

**Core application**
- **TMPA Application** — the central Spring Boot service. Hosts the static web UI and the REST API surface, is the sole authentication/authorization component (protecting all user and tenant data), and acts as a secure reverse proxy to external services (OS Data Hub, what3words) so API keys are never exposed to the browser. Deployed in a load-balanced cluster for high availability; all instances in an environment share the same database and file storage configuration.

**Rules engine integration**
- **1Integrate** (third-party, WildFly-based) — an "Interface" node exposing REST APIs to upload rules and run/monitor sessions, and one or more "Engine" nodes that each process one session at a time.
- **TMPA Rules** — the actual business rules that automate TM plan production, authored and tested by a dedicated rules team in 1Integrate, published via a repository sync tool, then packaged with Maven into a versioned zip (1Integrate backup format) bundled into a deployable JAR. Multiple rule versions can be live in production simultaneously, lowering the operational overhead of rule updates. A snapshot build runs nightly so the rules team can validate changes the next day.
- **Plan Runner Service** ("Auto Draft Service") — a Spring Boot microservice that clones, configures and monitors 1Integrate sessions, deploys/redeploys rules (including a dev-only forced redeploy on startup for nightly rule builds), and feeds user input through to 1Integrate, returning generated plans, failure reason codes, or advisories back to the application.

**Geospatial data & rendering**
- **Spatial Data Service** — a read-only Spring Boot REST service over a shared "reference data" PostGIS database, used for things like computing the spatial extent of a plan (from OS highways speed-limit data), looking up streets by USRN (not supported by the free "Names API" used for general location search), and fetching full permit details on demand (kept out of the tile service for performance).
- **Martin Service** — a vector-tile server (the open-source [Martin](https://github.com/maplibre/martin) project) serving OSM features (bridges, tunnels, schools, hospitals), OS MasterMap data (speed limits, roads, streets), NaPTAN transit stop data, and Street Manager permit data to the UI. Serves most layers from a prebuilt MBTiles file (found to significantly outperform live PostGIS queries during testing) while permit data — which changes far more frequently — is served live from PostGIS with time-window filtering. The MBTiles file is copied into the container's local storage on startup to avoid concurrency issues against shared storage.
- **GraphHopper Service** — a routing engine built on OpenStreetMap data, used to plan diversion routes for road vehicles and to suggest diversion sign placements. Ships as a self-contained Docker image with a prebuilt graph database for Britain and Ireland, rebuilt nightly by CI so scaling is just adding more load-balanced nodes.
- **PDF Generator Service** — a Spring Boot microservice that renders TM plan documents to PDF by driving headless Chrome one page at a time, using Redis both as a job queue (across a pool of "page generator" threads, each effectively a Chrome tab) and for cross-instance semaphores/mutexes. Assembles completed multi-page PDFs and hands them to the TMPA Application for storage. Memory-hungry and bursty (up to ~2GB RAM per page render observed), so the page-thread pool size is a key deployment/capacity-planning lever.

**Reference-data ingestion pipeline**
- **OS Downloader** — downloads versioned monthly data packages from the Ordnance Survey Data Hub, skips redundant downloads, verifies checksums, and atomically publishes each dataset into a shared volume (`/data/os-downloads/{data_package_id}/{published_date}/`) so partial downloads are never mistaken for complete ones.
- **Spatial Importer** and **Rules Data Importers** (3 containers: MasterMap, Road, Speed Limits) — load the downloaded OS data into PostgreSQL/PostGIS via `ogr2ogr`, reshaping it to match the schema the rules engine was originally built against. Run on a quarterly cadence aligned with OS MasterMap releases; each run is verified in dev with the Bulk Test Framework before being promoted to production, and the previous database version is retained to allow rapid rollback.
- **Martin Import** — builds the MBTiles file consumed by the Martin Service, pulling in OS data plus OpenStreetMap (Geofabrik Britain & Ireland extract), NaPTAN, and TfL bus stop data, versioned by internal schema version and OS download date.
- **Street Manager Open Data SNS Function** — an Azure serverless function subscribed to Street Manager's AWS SNS open-data feed (permit submitted/granted/work-start events etc.), validating and persisting each event as a file on a shared volume, ordered by received timestamp.
- **Work Importer** — a scheduled Spring Boot application that reconciles the "reference data" database with permits from Street Manager (via the SNS function's output), the Scottish Roadworks Register, and Geoplace SWA codes. Deliberately single-instance (not clustered) since concurrent writers were found to cause optimistic-locking conflicts.

**Testing**
- **Bulk Test Framework** — a standalone Spring Boot application (own database, own Azure AD-secured environments for dev/test/demo use) built for the rules team to define test suites of thousands of test cases, submit them against a live TMPA deployment, and analyze pass/fail results at scale — the primary regression safety net for rules changes.

**External integrations**
- **OS Data Hub** — OS Features API (consumed server-side by the 1Integrate engine for geometry/metadata), OS Vector Tile API and OS Names API (both proxied through the TMPA Application, which also logs usage to Logstash for OS billing/reporting).
- **what3words** — proxied through the TMPA Application to protect the API key; used for address lookup and for tagging plan locations.
- **Bing Maps** — used directly from the browser (not proxied) for satellite imagery, relying on referrer-domain restriction to protect the API key.

**CI/CD**
- **Bitbucket Pipelines** for test execution and static analysis on every change.
- **Jenkins** for building snapshot and release artefacts.
- **Nexus** as the internal Maven dependency repository and Docker registry, with production images promoted to **Azure Container Registry**.
- **Azure DevOps pipelines** for deploying and managing the live environments.
- Database schema changes across all services are managed with **Flyway** migrations, maintained in source control and applied automatically on startup.

---

## What I built / owned

- Designed and helped build the overall microservices architecture: roughly ten containerized Spring Boot services deployed in load-balanced clusters behind the central TMPA Application gateway, which handles authentication, multi-tenant data isolation, and secure proxying of external APIs.
- Built the integration layer between the application and 1Integrate (the rules engine), including session lifecycle management (clone/configure/monitor), automated "auto-draft" plan generation, and versioned rules packaging/deployment via Maven.
- Designed the geospatial reference-data pipeline: OS Data Hub ingestion with checksum verification and atomic publishing, PostGIS loading via `ogr2ogr`, and a quarterly release process with dev verification (via the Bulk Test Framework) before production promotion and rollback support.
- Built the PDF generation service — headless-Chrome page rendering parallelized via a Redis-backed job queue — and the vector-tile serving layer (Martin/PostGIS/MBTiles), including the performance tradeoff analysis between serving tiles from prebuilt MBTiles vs. live PostGIS.
- Built the bulk automated regression-testing framework used by the rules team to validate rules changes against thousands of generated test plans before release.
- Integrated third-party geospatial services (OS Data Hub, what3words, Bing Maps, GraphHopper) including the secure reverse-proxy pattern used to keep API keys off the client.
- Full-stack feature development: React/TypeScript/Material UI frontend, Java/Spring Boot backend; upgraded major libraries (Jackson, MUI); automated testing with JUnit and Testcontainers.
- Built and maintained the CI/CD toolchain: Bitbucket Pipelines, Jenkins, Nexus, Azure DevOps/Azure Container Registry.
- Participated in architecture discussions, code reviews and mentoring.

---

## Technologies

Java, Spring Boot, React, TypeScript, Material UI, PostgreSQL, PostGIS, Redis, Docker, Flyway, Maven, Jenkins, Bitbucket Pipelines, Azure DevOps, Azure Container Registry, AWS SNS, Martin (MapLibre), GraphHopper, 1Integrate, JUnit, Testcontainers.
