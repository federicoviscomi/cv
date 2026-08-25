# TMPA / 1Streetworks — Interview Prep Notes

Pulled from the architecture docs (no source code access), focused on the kind of "why did you do X" and "tell me about a time" questions this system sets you up well for. Cross-check specifics (exact numbers, who decided what) against your own memory before using these in an interview — the docs describe the system, not always your personal role in each decision.

---

## The "why microservices, not a monolith" story

Service boundaries line up with genuinely different resource/scaling profiles rather than being arbitrary:

- **PDF Generator** is memory-hungry and bursty (~2GB RAM per page render observed) — it needed its own deployable unit so it could be capacity-planned independently of everything else, with the page-thread pool size tuned against available memory per node.
- **GraphHopper** and **Martin** are read-heavy, mostly-static data services — they're shipped as self-contained Docker images with data baked in (or copied in at startup) so they scale by just adding load-balanced nodes, no shared-state coordination needed.
- **Work Importer** is deliberately the *opposite* — single-instance only. Running more than one caused optimistic-locking exceptions against the reference DB, because the import model assumes one sequential writer. Good example of knowing when *not* to scale horizontally, and being able to say why in one sentence.
- **Plan Runner** exists specifically to isolate the rest of the system from a legacy third-party engine (1Integrate) that has its own very different operational model (WildFly, one engine = one session at a time).

If asked "how did you decide on service boundaries" — this is a stronger answer than "microservices are best practice": boundaries followed resource shape and failure isolation, not just domain lines.

## Decisions backed by actually measuring something

- **MBTiles vs. live PostGIS for tile serving**: they tested both modes (Martin supports either) and found prebuilt MBTiles significantly outperformed live queries — so static/slow-changing layers (OSM, OS MasterMap, NaPTAN) are served from a periodically rebuilt MBTiles file, while permit data (which changes constantly) is deliberately kept on live PostGIS queries with time-window filtering because baking it into MBTiles would mean serving stale permits. That's a real "we picked the boundary between two strategies based on data volatility, not for consistency's sake" story.
- **what3words cost asymmetry**: converting a location to a w3w address is free/unlimited, converting a w3w address back to coordinates costs money — the integration is shaped around only calling the paid direction when the user explicitly saves a plan feature or uses the w3w lookup tab. Good example of a system design decision driven by a vendor's pricing model, not just engineering taste.
- **OS Data Hub usage logging**: the reverse-proxy for the OS Vector Tile/Names APIs logs every request to Logstash specifically so the team can reconcile usage against what they're billed by OS. Worth mentioning if asked about cost-awareness or observability with a business purpose, not just debugging.

## Security pattern, and a deliberate exception to it

- Standard pattern: never expose third-party API keys to the browser. OS Data Hub and what3words are both proxied through the TMPA Application, which holds the credentials server-side.
- Bing Maps is the deliberate exception — imagery tiles are fetched directly by the browser, with only referrer-domain restriction protecting the key. If asked about it: this is a real tradeoff (avoiding proxy latency/load for a high-volume tile service, accepting a weaker but "good enough" protection for a lower-sensitivity credential) rather than an oversight — worth being ready to justify it that way rather than being caught defending it as if you'd forgotten to proxy it.

## Working around a legacy/third-party dependency

- The 1Integrate REST client's default timeout (30s) was fine for most calls but hopeless for deploying the rules bundle (~150MB), which routinely exceeds it — production runs with a much longer timeout specifically for that call. A good concrete "found a limitation in a dependency and adapted the config/architecture around it" answer if asked for one.
- Rules are versioned and multiple versions can be deployed to a single 1Integrate instance simultaneously — this exists specifically to decouple a rules release from an application release, reducing coordinated-deploy risk between two different teams (app team vs. rules team) working against the same runtime.

## Data pipeline: integrity and rollback

- OS Downloader publishes each dataset atomically (download to a temp location, verify checksums, then move into place) — the presence of the dated directory *is* the completeness signal, so nothing downstream can ever pick up a partially-downloaded dataset.
- Quarterly data releases go to dev first, get verified with the Bulk Test Framework, and only then get promoted to production — with the previous database version retained specifically so a bad release can be rolled back quickly. This is a solid answer if asked how you'd safely roll out a change to data that drives automated decision-making (the rules engine).
- Multiple services (Work Importer, Spatial Data Service, Martin Service) read/write the same "reference data" database — a good prompt to talk about the coordination cost of a shared schema across services owned at different times, and why Flyway-managed, source-controlled migrations mattered there.

## Eventing / consistency requirements

- Street Manager delivers permit events over AWS SNS with no ordering guarantee. Rather than building out ordering/dedup machinery, the team leaned on a real observation: only the *current* state of a permit matters to this system, not its history — so out-of-order delivery is a non-issue. This is a clean example of matching the complexity of your consistency handling to what the domain actually requires, instead of defaulting to the "safest-looking" solution.
- The serverless ingestion function does the absolute minimum (validate + persist raw event to a shared volume) and hands processing off to Work Importer entirely separately — a deliberate decoupling of ingestion from processing, which also means a Work Importer outage can't cause events to be lost.

## Testing a rules/business-logic engine at scale

- The Bulk Test Framework is a purpose-built application (not just a test suite) because validating "does this rules change still produce sane traffic management plans" needs to run thousands of real scenario-based test cases against a live TMPA deployment, not just unit-test the rules logic in isolation. Good to have ready if asked how you test something that's fundamentally about generated, hard-to-assert-on output (a generated plan) rather than a simple input/output function.

## CI/CD shape — and why it's not just "one tool"

Bitbucket Pipelines (test + static analysis) → Jenkins (build snapshot/release artefacts) → Nexus (Maven + Docker registry) → Azure DevOps pipelines + Azure Container Registry (deploy to live environments). Worth being able to explain simply: build tooling and deployment tooling are split, "latest" nightly images flow to dev automatically, and QA/production only ever get versioned release artefacts — so you can talk about promotion strategy and blast-radius control even without knowing every tool's internals cold.

---

## Things you may want to firm up before an interview (not fully covered by the docs)

- Rough scale: how many customers/tenants, how many plans generated per day/month, team size you worked in.
- Specific incidents you personally debugged (the docs describe *designed* behavior, not war stories — those are usually the strongest interview answers).
- Anything about the frontend architecture in more depth — the docs are almost entirely backend/infrastructure-focused.
