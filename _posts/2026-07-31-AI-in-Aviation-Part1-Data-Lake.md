---
layout: post
title: AI in Aviation, Part 1 - Building the Data Lake Foundation
---

## Introduction ##

Every aviation operator I talk to at the moment is asking some version of the same question: where do we start with AI? Copilots for engineers, predictive maintenance for the fleet, disruption management assistants, dynamic crew rostering — the list of plausible use cases is long, and vendors are more than happy to sell you the model layer first.

The problem is that almost none of it works without a data foundation underneath it. Aviation is one of the most data-rich industries there is — flight plans, ACARS telemetry, ADS-B position reports, MRO records, AODB events, crew rosters, baggage systems, loyalty data, weather — and almost none of it lives in one place. It's scattered across decades of point-to-point integrations between reservation systems, maintenance platforms, airport systems, and ground handler APIs, each with its own format, latency, and owner. Point an AI model at that landscape directly and you get exactly what you'd expect: inconsistent answers, stale context, and a governance headache.

This is the first post in a four-part series on **AI in Aviation**, building on the [AWS Landing Zone for Aviation]({{ site.baseurl }}/AWS-Aviation-Landing-Zone/) architecture I covered previously. The series runs:

- **Part 1 (this post)** — Building a data lake foundation on AWS so aviation data is usable, governed, and AI-ready rather than trapped in silos
- **Part 2** — Model selection, AI governance, and security for aviation workloads
- **Part 3** — Standing up the AI platform: Amazon Bedrock, agents, and the tooling around them
- **Part 4** — Bringing it together: AI applications running on top of the data lake, with a working example

Before any of that model-layer work makes sense, the data has to be in a shape where it can be trusted, governed, and queried consistently. That's what this post covers: why silos are the real blocker to aviation AI, and how to architect a data lake on AWS that fixes it.

The architecture diagrams in this post have been created using [Eraser.io](https://www.eraser.io/).


## The Silo Problem in Aviation ##

Ask an airline or airport how many operational systems of record they run, and the number is usually higher than they expect. A representative — far from exhaustive — list looks like this:

- **PSS / GDS** — passenger service systems and global distribution systems holding bookings and PNR data, typically exchanged as EDIFACT or NDC XML
- **AODB** — the Airport Operational Database, the airport's system of record for flight schedules, stand and gate allocation, and turnaround events
- **MRO systems** — platforms like AMOS or TRAX holding maintenance records, work orders, and parts inventory, usually on a completely separate technical stack from operations
- **ACARS and engine telemetry** — real-time and near-real-time aircraft messaging and sensor data, arriving as high-frequency streams during flight
- **ADS-B position data** — live aircraft tracking, often sourced from third parties like FlightAware
- **Crew management systems** — rostering, Flight Duty Time (FDT) tracking, and qualifications
- **Baggage handling systems** — BHS scan events across sortation and load points
- **Weather feeds** — METAR/TAF and forecast data from external providers
- **Commercial and loyalty platforms** — CRM, ancillary revenue, and frequent flyer systems

Each of these is a system of record for someone — engineering, ops, commercial, crewing — and each was typically integrated point-to-point with whichever downstream system needed it at the time. The result is dozens of brittle, bespoke integrations, no consistent view of "this aircraft, right now," and data that is either siloed for good operational reasons (regulatory, safety-critical) or siloed for no reason other than nobody has consolidated it.

AI does not fix this. AI makes it worse, faster — because now instead of one brittle integration you have a model confidently synthesising an answer from partial, stale, or duplicated data across five different sources, none of which it knows are inconsistent with each other.


## Why a Data Lake, Not More Point-to-Point Integration ##

The instinct when a new use case appears is often to build another direct integration: pull the engine telemetry, wire it to the new dashboard, done. That works exactly once. The tenth time you do it, you have ten brittle pipelines, ten sets of credentials to rotate, and no single place where "aircraft health" or "flight status" has one agreed definition.

A data lake inverts this. Producers publish once, to a governed landing zone. Consumers — dashboards, BI tools, ML pipelines, and eventually AI agents — read from a curated, catalogued, access-controlled layer rather than reaching back into operational source systems directly. This has a few concrete benefits specific to aviation:

- **Decoupling** — MRO, ops, and commercial teams keep their systems of record; the lake doesn't replace AMOS or the PSS, it becomes the shared, analysis-ready copy of the data they produce
- **One definition of the truth** — "aircraft health score," "on-time performance," and "connecting passengers at risk" get computed once, consistently, rather than recalculated slightly differently in every downstream tool
- **Governed access at the right granularity** — crew FDT and passenger PII need very different access controls to public flight status; a lake with proper cataloguing lets you enforce that centrally rather than per integration
- **A runway for AI** — every genuinely useful aviation AI use case (predictive maintenance, disruption triage, crew re-planning) depends on having clean, joined, historical data available — which is exactly what a data lake is for


## Data Lake Architecture: Zones ##

The architecture uses a standard three-zone (medallion) layout on **Amazon S3**, which keeps the model simple while giving each stage of processing a clear contract with the next.

- **Raw / Landing zone** — an immutable, as-received copy of source data, partitioned by source system and ingestion date. Nothing is transformed here; this is the audit trail and the reprocessing source if downstream logic changes
- **Curated / Conformed zone** — schema-applied, deduplicated, standardised data — consistent time zones, consistent units (knots vs km/h, litres vs gallons of oil consumption), decoded ACARS message content, resolved aircraft and flight identifiers across source systems
- **Consumption / Aggregated zone** — business-level marts and feature sets: fleet health scores, OTP rollups, connection risk tables — built for direct consumption by BI tools, ML training, and downstream AI applications in later parts of this series

Each zone is a separate S3 prefix (or separate bucket, depending on the account structure — more on that below), with lifecycle policies moving ageing raw data to S3 Glacier for cost-efficient long-term retention, which matters given aviation's regulatory record-keeping requirements.

### Data lake zone architecture ###

![_config.yml]({{ site.baseurl }}/images/blog/AI-in-Aviation/Diagram1.png)


## Ingestion Patterns ##

Aviation data arrives in three fundamentally different patterns, and the architecture needs to treat each on its own terms rather than forcing everything through one pipeline shape.

### Batch ###

PSS/GDS extracts, MRO work order exports, and AODB nightly reconciliation files typically arrive as scheduled file drops. **AWS Transfer Family** provides a managed SFTP endpoint for partners and legacy systems that only know how to push files, landing directly into the S3 raw zone. Where a source system supports it, **AWS Database Migration Service (DMS)** handles change data capture from operational databases (AODB, crew systems) rather than waiting for a nightly export.

### Streaming ###

ACARS telemetry and ADS-B position reports are naturally high-frequency and time-sensitive — this is where a batch pattern falls down. **Amazon Kinesis Data Streams** ingests the raw telemetry, with **Kinesis Data Firehose** buffering and writing it into the S3 raw zone in near real time. A parallel **AWS Lambda** stream processor can act on the same stream for time-critical logic — writing current aircraft state to **Amazon DynamoDB** for low-latency operational lookups, and publishing events to **Amazon EventBridge** when thresholds are breached (an EGT margin closing in on a critical limit, for example) — while the same data lands durably in the lake for historical analysis.

### API Pull ###

Weather and third-party flight tracking data (FlightAware AeroAPI and similar) don't push data to you — you have to go and get it. **Amazon EventBridge Scheduler** triggers a Lambda function on a fixed interval to pull and land this data alongside everything else, keeping it in the same governed, catalogued lake rather than living only inside whichever dashboard happens to call the API directly.

### Streaming ingestion detail ###

![_config.yml]({{ site.baseurl }}/images/blog/AI-in-Aviation/Diagram2.png)


## Storage and Table Format ##

**Amazon S3** is the durable store across all three zones, but the table format on top of it matters more than it used to. The curated and consumption zones use **Apache Iceberg** tables (via AWS Glue and Amazon S3 Tables) rather than plain Parquet-on-S3, for three reasons specific to aviation data:

- **Schema evolution** — engine telemetry schemas change as new sensor generations are introduced across a fleet; Iceberg handles this without breaking existing queries or requiring a full rewrite
- **ACID transactions** — multiple concurrent Glue jobs can safely write to the same table, which matters once you have parallel pipelines processing different source systems into a shared consumption table
- **Time travel** — being able to query a table as it existed at a specific point in time is genuinely useful for maintenance and safety record-keeping, where you may need to reconstruct exactly what the data showed at a given moment

Partitioning is source-appropriate: telemetry and flight data by aircraft registration and date, PSS/booking data by carrier and date, MRO records by aircraft and work order.


## Cataloguing and Governance ##

**AWS Glue Data Catalog** provides the single metadata catalog across all three zones — every table, regardless of which team or pipeline produced it, is discoverable in one place with consistent schema information.

Access control sits on top via **AWS Lake Formation**, using LF-Tags to classify data by sensitivity (crew personal data, passenger PII, commercial data, public operational data) and grant access at the table, column, or row level based on those tags rather than per-table grants that need to be maintained individually as new tables are added. This is where the data lake architecture connects directly back to the [Landing Zone]({{ site.baseurl }}/AWS-Aviation-Landing-Zone/) foundation: the data lake typically sits in its own account within the Workload OU, and Lake Formation permissions are granted to the same IAM Identity Centre personas already defined there — platform engineers, application engineers, operations, and security teams — so a crew scheduling analyst and an MRO engineer see different slices of the same lake without either team needing separate infrastructure.

This governance layer is also the foundation Part 2 of this series builds on directly, when we get into AI-specific governance — controlling what a model or agent is allowed to retrieve and reason over is the same access-control problem, applied to a new class of consumer.

### Governance and access architecture ###

![_config.yml]({{ site.baseurl }}/images/blog/AI-in-Aviation/Diagram3.png)


## Processing and Orchestration ##

**AWS Glue ETL** jobs (Spark-based) handle the bulk of the raw-to-curated and curated-to-consumption transforms — schema conformance, deduplication, unit standardisation. For heavier workloads, such as feature engineering across a full fleet's historical telemetry ahead of a predictive maintenance model, **Amazon EMR Serverless** provides the additional scale without managing clusters. Lightweight, low-latency transforms on the streaming path stay in **AWS Lambda**, as described above.

Pipeline dependencies — "don't build today's fleet health rollup until the overnight AMOS extract and the telemetry curation job have both completed" — are orchestrated with **Amazon Managed Workflows for Apache Airflow (MWAA)**, giving visibility into pipeline health and a place to manage retries and alerting as the number of source feeds grows.


## Consumption Layer ##

With the lake catalogued and governed, consumption fans out to whichever tool suits the audience: **Amazon Athena** for ad hoc SQL against the curated and consumption zones, **Amazon Redshift Serverless** for BI workloads that need consistent low-latency performance at scale, and **Amazon QuickSight** for the operational and executive dashboards — fleet health, OTP trends, disruption cost exposure — that most stakeholders actually interact with day to day.

This consumption layer is also where Part 3 and Part 4 of this series pick up: the same curated, governed tables become the retrieval source for Bedrock Knowledge Bases and the feature source for ML models, once the AI platform is in place.


## Art of the Possible: SkyConnect ##

To make this concrete, it's worth looking at what this pattern enables once it's built. **SkyConnect** is a demo Operational Command Centre we built at Amach on exactly this architecture, simulating a fictional airline running 18 aircraft across a four-route European network out of Dublin.

The data path is a small, self-contained version of everything described above: an ACARS simulator publishes engine telemetry to a **Kinesis Data Stream**, which fans out through **Kinesis Firehose** into an S3 data lake (catalogued hourly by a **Glue Crawler** and queryable via **Athena**) and through a **Lambda stream processor** into **DynamoDB** for live fleet state — the same dual hot-path/durable-path pattern from Diagram 2 above, running end to end. Live ADS-B position data from FlightAware AeroAPI is layered on top for real aircraft tracking. The screenshots below are taken directly from a live, running instance of the demo — not mockups.

Everything starts from one command overview, with every department reading from the same governed data foundation rather than its own disconnected tool:

![SkyConnect Operational Command Centre overview]({{ site.baseurl }}/images/blog/AI-in-Aviation/screenshots/SkyConnect-OCC-Overview.png)

**Predictive maintenance with lead time.** The fleet health grid ranks all 18 aircraft by risk. Drilling into EI-SKJ — flagged Critical at a 7% health score — shows exactly why: Engine 1's EGT margin has narrowed to 5.0°C against a critical threshold, oil consumption is running at 2.11 L/hr, and the aircraft is already sitting AOG with three open MEL/CDL items. None of that is visible from any single source system in isolation — it only becomes a coherent picture because engine telemetry, maintenance records, and airframe utilisation are joined in one place.

![SkyConnect predictive maintenance detail for aircraft EI-SKJ]({{ site.baseurl }}/images/blog/AI-in-Aviation/screenshots/SkyConnect-Predictive-Maintenance.png)

**Disruption cascades traced automatically.** A technical delay on SK552 (LHR → DUB) is followed through its knock-on effects in real time: the 6-minute delay pushes SK553's departure to 15:51, 23 of the 127 passengers on board need active re-accommodation, and the welfare triage automatically prioritises the passengers who need it most — an 86-year-old wheelchair user misconnecting from her onward flight, an unaccompanied 7-year-old, and a passenger travelling with an oxygen concentrator. All of that comes from joining flight, passenger, and delay data that would traditionally sit in three unconnected systems.

![SkyConnect disruption management desk showing the SK552 delay cascade]({{ site.baseurl }}/images/blog/AI-in-Aviation/screenshots/SkyConnect-Disruption-Desk.png)

This is also a useful illustration of where automation should stop and a person should start. Most airlines already run an automated rebooking engine that auto-accommodates the bulk of disrupted passengers onto the next available seat, and that's the right default — it's fast and it works for the overwhelming majority of cases. What SkyConnect adds isn't a replacement for that engine, it's a filter in front of it: an AI layer that scores every affected passenger against a welfare model — factoring in journey time, number of connections, and whether special assistance has actually been pre-arranged at the connecting airport — and pulls out only the small number who shouldn't be allowed to flow through standard automated rebooking untouched. For the 86-year-old wheelchair passenger, the fastest option on paper is a one-stop routing with a tight connection and no assistance booked on the second leg — exactly what the automated engine would confirm by default, and exactly the kind of itinerary that leaves a vulnerable passenger to navigate an unfamiliar terminal alone. The system flags that option as low-scoring, pulls her case out of the automated flow, and puts a direct flight in front of a human agent as the recommended choice instead — even though it costs the airline more in seat availability and fare difference than the connecting option would. That's a trade the airline is glad to make: no incident, no complaint, no story, and a passenger who had a good experience despite the disruption. It's a small example of a broader principle this series comes back to in Part 2: the value of AI here isn't doing the rebooking, it's knowing which cases should never have been left to full automation in the first place.

The same delay cascades into crew: Captain Aoife Brennan's Flight Duty Time is projected to breach the 8-hour limit if she operates SK553, and the platform surfaces a ready standby captain before the breach happens, rather than after.

![SkyConnect crew control showing an FDT breach and standby crew recommendation]({{ site.baseurl }}/images/blog/AI-in-Aviation/screenshots/SkyConnect-Crew-FDT.png)

None of the individual dashboards in SkyConnect are the point — the point is that they're all cheap to build once the data lake underneath them exists, and each one becomes materially more capable once an AI layer sits on top, which is exactly where this series is headed.


## Summary ##

Aviation AI initiatives that skip straight to the model layer are building on sand. The systems of record — PSS, AODB, MRO, ACARS, crew, weather — will always be siloed by design and by necessity; the fix isn't to eliminate those silos but to stop forcing every new use case to integrate with them individually.

The architecture in this post — S3 zones with Apache Iceberg, batch and streaming ingestion matched to the source, Glue Data Catalog and Lake Formation for governance, tied back into the account and identity structure from the [Landing Zone]({{ site.baseurl }}/AWS-Aviation-Landing-Zone/) — gives you exactly that: a single, governed, AI-ready copy of aviation data that every downstream use case can build on rather than re-plumb from scratch.

In Part 2, I'll cover model selection, AI governance, and security — how to decide which models to use for which aviation workloads, and how to control what they're allowed to see and do against the data lake foundation built here.

If you're working through a data lake or AI strategy for an aviation or similarly regulated environment and want to discuss the architecture, feel free to reach out.
