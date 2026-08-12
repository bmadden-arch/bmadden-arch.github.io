---
layout: post
title: Reimagining the AODB - From Static Registry to AI-Augmented Decision Layer
---

## Introduction ##

Every airport runs on an AODB — the Airport Operational Database, the single source of truth for flight schedules, stand and gate allocation, and turnaround events that every other system on the airfield ultimately reads from or writes back into. It's foundational infrastructure, and it's also, in almost every implementation I've come across, a passive registry: it records what happened and what's scheduled to happen, but it doesn't reason about any of it. Stand conflicts get caught when a controller notices two aircraft assigned to the same piece of concrete, not before. Crew shortages get caught when a ground handler radios in short-staffed, not two hours out when the data already showed it coming.

This post looks at what changes when you give that registry a reasoning layer — real-time fusion of flight data with facility telemetry, predictive signals ahead of the conflict rather than after it, and generative AI that proposes concrete actions instead of just surfacing a dashboard of red numbers. It builds on two things I've covered previously: the [AWS Landing Zone for Aviation]({{ site.baseurl }}/AWS-Aviation-Landing-Zone/) account and identity foundation, and the data lake and AI platform patterns from the [AI in Aviation]({{ site.baseurl }}/AI-in-Aviation-Part1-Data-Lake/) series — in particular [Part 1's]({{ site.baseurl }}/AI-in-Aviation-Part1-Data-Lake/) data lake foundation and [Part 3's]({{ site.baseurl }}/AI-in-Aviation-Part3-Standing-Up-The-Platform/) Bedrock and agent platform. This post stands on its own rather than continuing that series, and the working example throughout is a real, deployed demo I've been building at Amach — not mockups.

The architecture diagrams in this post have been created using [Eraser.io](https://www.eraser.io/), then further enriched using ChatGPT's visual visionary theme for a more polished finish — thanks to Kevin Webb at Ingram Micro for showing me how to level up my architecture diagrams this way.


## Why Today's AODB Falls Short ##

The AODB's job, historically, has been to be correct and auditable — a system of record, not a system of insight. That's the right design goal for what it was built to do, but it leaves three gaps that matter more every year:

- **Reactive, not predictive.** A stand conflict is a query result, not an alert. Nothing in the traditional model flags a tight turnaround buffer as a risk before it becomes an actual overlap.
- **Siloed from facility data.** The AODB knows about flights, stands, and gates. It typically knows nothing about the HVAC unit overheating in the terminal it's about to route 200 passengers through, or the baggage belt motor showing early vibration signs — because that data lives in a completely separate building management system, if it's instrumented at all.
- **A human bottleneck at scale.** Exception handling — reassigning a stand, escalating a maintenance issue, adjusting ground crew — depends on someone noticing, cross-referencing several screens, and acting, which works until volume outpaces the number of people watching.

None of this is a criticism of AODB vendors or airport ops teams — it's a reasonable design point for systems built before airports had the sensor density or the compute to do anything else with the data. That's exactly what's changed.


## What an AI-Augmented AODB Looks Like ##

The shift isn't replacing the AODB — it's adding a reasoning layer on top of it that does three things a static registry structurally can't:

**Real-time fusion of flight and facility signals.** Stand allocation, turnaround timing, and ground crew capacity stop being evaluated in isolation from HVAC load, escalator and belt health, and passenger flow — because they're all downstream of the same physical event: an aircraft on the ground, generating demand across every system at once.

**Predictive signals ahead of the conflict, not after it.** Anomaly detection over facility telemetry and buffer-based risk scoring over ground-time windows both exist to answer one question earlier than a human would ask it: what's about to break, before it breaks.

**Generative AI as a reasoning layer over structured evidence — not a black box.** This is the part worth being precise about, because it's also the part most likely to be oversold. The model isn't given free rein over the AODB. It's given a narrow, well-defined set of computed signals — a conflict, a risk score, an alert — and asked to reason over them and propose a specific, evidenced action. That's a materially smaller and more auditable job than "manage the airport," and it's why the governance model matters as much as the model itself.

This brings up the governance point directly: **the human stays the decision-maker, not the model.** Every pattern below is "AI proposes, human decides," not "AI decides." That's not a limitation bolted on for compliance optics — it's the design that makes the rest of this trustworthy enough to actually deploy.


## Reference Architecture: The Data Platform ##

The concrete example throughout this post is a real Amach demo built against London Gatwick's live public flight data (EGKK), running on AWS. The data platform underneath it follows the same real-time/durable dual-path pattern from the [AI in Aviation]({{ site.baseurl }}/AI-in-Aviation-Part1-Data-Lake/) series, applied specifically to airport ground operations rather than in-flight telemetry.

Flight data comes from the FlightAware AeroAPI — real OOOI timings (out/off/on/in) and, where available, the actual runway used — pulled on a schedule and landed into **Amazon DynamoDB** for low-latency operational lookups. Facility telemetry — HVAC units, escalators, baggage belts, passenger-flow counters, across a fleet of simulated IoT/LoRaWAN-style sensors standing in for a real building management system — publishes over **AWS IoT Core** and lands in **Amazon Timestream for InfluxDB**, a purpose-built time-series store, with the raw payloads archived to **Amazon S3**. **Amazon EventBridge Scheduler** drives both cadences independently, since flight data and sensor telemetry have very different natural update frequencies.

The distinction that matters here — and the one the demo is explicit about rather than blurring — is that FlightAware genuinely provides delay and taxi timing, but has no visibility at all into ground movement: no taxiway identifiers, no real-time stand occupancy. That's A-CDM and surface-radar territory, which most airports either don't have or don't expose externally. So stand assignment, taxiway grouping, and congestion scoring in this demo are simulated, deterministically, driven by the real delay and taxi data — the same pattern the facility sensor fleet uses. Being explicit about which numbers are real and which are simulated is a small thing, but it's the difference between a demo that's credible under a technical audience's questions and one that isn't.

### Data ingestion and platform architecture ###

![_config.yml]({{ site.baseurl }}/images/blog/AODB-Future/Diagram1.png)


## The Decision Layer: "AI Proposes, Human Decides" ##

This is the part of the demo that argues most directly for the thesis of this post — an AODB Actions queue that reasons over live stand conflicts, predictive turnaround risk, and active facility alerts, and proposes concrete, evidenced actions for a human operator to approve or reject.

The pattern: **Amazon SageMaker** runs a Random Cut Forest model over facility telemetry for anomaly detection, and a separate, deterministic pass over live stand data flags two tiers of signal — an actual conflict (two flights already overlapping on the same stand) and an at-risk stand (a turnaround buffer under 20 minutes — not overlapping yet, but one delay away from becoming a real conflict, which is the higher-value, genuinely predictive case). Both, along with active alerts, are handed to **Amazon Bedrock**, with Claude as the target model, using forced tool-use against a single `propose_actions` tool — the model doesn't get to respond with prose, only with a structured action: a stand reassignment or a maintenance escalation, each with a rationale, specific evidence strings (flight idents, timestamps, stand IDs, gap minutes), and a confidence level. Every proposal lands in DynamoDB as "pending" until a human operator approves or rejects it from the frontend.

Two design decisions here are worth calling out explicitly, because they're the kind of detail that separates a demo built to impress from one built to actually deploy:

**It's deliberately conservative.** The system prompt instructs the model to propose a reassignment only when a genuinely free, compatible stand actually exists — not to force a recommendation because a conflict exists. Early in testing, against a snapshot with 55 stand conflicts and 7 predicted conflicts at Gatwick, three separate generation calls in a row returned zero proposals, because no terminal group had a genuinely free stand to reassign into at that moment. That's not a bug — it's the conservatism working exactly as designed: a system that always has an answer is more dangerous than one that sometimes says nothing. The screenshot further down shows the same queue later, once conditions had shifted — real proposals, including a predicted reassignment made before the conflict actually happened.

**v1 is deliberately log-only.** Approving a proposal updates its own status and records who decided it and when — it does not write back into the live flight or stand data. The underlying conflict keeps showing until its real cause changes, exactly as it would today. That's a conscious choice to avoid a second, competing source of truth for stand assignment sitting alongside the AODB itself, and it's the right sequencing: prove the reasoning is trustworthy in an audit trail before giving it write access to operational state.

The same pattern shows up a second time in the same demo, independently — which is the strongest evidence it's a reusable architectural approach rather than a one-off feature. A ground-crew management view models a finite, shared 25-team pool (across several real handling agents) against every flight's ground-time window, using genuine interval allocation rather than an unlimited team per flight. A Crew Demand Forecast panel computes concurrent demand against that pool over a rolling window, and — on request — Bedrock reasons over that pre-computed forecast to generate a short staffing note: where the peak is, by how much it exceeds capacity, and where there's slack to safely release a team. Same shape both times: **compute the signal deterministically, let the model reason over the evidence, keep a human in the loop for the decision.**

### AI decision loop — "AI proposes, human decides" ###

![_config.yml]({{ site.baseurl }}/images/blog/AODB-Future/Diagram2.png)


## Serving It Up ##

The consumption layer is intentionally simple. A React single-page app, served from Amazon S3 via **Amazon CloudFront**, polls a single **Amazon API Gateway** endpoint backed by one Lambda function — no websocket layer, since a 2-minute flight-data cadence and a 30-second sensor cadence don't need sub-second push updates, and polling is a much smaller operational surface to reason about. **Amazon Location Service** provides the live operations map directly to the browser.

Access is gated by a real **Amazon Cognito** user pool — Hosted UI, OAuth Authorization Code with PKCE, and mandatory TOTP MFA on every account, with a Pre Sign-Up Lambda restricting self-service sign-up to the right domain. API Gateway requires a valid Cognito ID token on every request, so the data itself is protected server-side, not just the dashboard shell in front of it. It's a detail that's easy to skip in a sales demo and easy to regret skipping the first time a prospect's security team asks how the login actually works.

### API, frontend, and identity architecture ###

![_config.yml]({{ site.baseurl }}/images/blog/AODB-Future/Diagram3.png)


## Art of the Possible ##

The screenshots below are taken directly from a live, running instance of the Amach demo against real Gatwick flight data — not mockups.

Everything starts from one live operations view, with real flight arrivals and departures, real facility sensor state, and the AI platform status all visible on a single screen:

![Amach Airport Command Centre — Live Operations overview at London Gatwick]({{ site.baseurl }}/images/blog/AODB-Future/screenshots/Amach-Live-Operations-Overview.png)

Flight Operations Impact shows exactly what's real and what's derived: FlightAware's genuine OOOI timeline drives real delay and taxi figures, propagated into a stand allocation grid where the conflicts and at-risk badges below are live, not staged:

![Amach Flight Operations Impact showing live stand conflicts across the Gatwick apron grid]({{ site.baseurl }}/images/blog/AODB-Future/screenshots/Amach-Flight-Ops-Stand-Conflicts.png)

Turnaround & Baggage is where the second instance of the "AI proposes, human decides" pattern lives — the finite ground-crew pool, individual crew assignment and reassignment against real flights, and the Crew Demand Forecast panel with a Bedrock-generated staffing note reasoning over the same pre-computed demand curve:

![Amach Turnaround Ops showing ground-crew management and the AI-generated Crew Demand Forecast staffing note]({{ site.baseurl }}/images/blog/AODB-Future/screenshots/Amach-Turnaround-Crew-Demand-Forecast.png)

And here's that same AODB Actions queue in action: stand reassignments — including one flagged **PREDICTED**, generated before the conflict actually happened — a maintenance escalation, and the running approval stats a real operator would see:

![Amach AODB Actions queue showing pending stand reassignment and maintenance escalation proposals, including a predicted reassignment, awaiting human approval]({{ site.baseurl }}/images/blog/AODB-Future/screenshots/Amach-AODB-Actions-Queue.png)


## What's Next ##

Worth being as honest about the gaps here as about what's working. Write-back from an approved AODB Actions proposal into the live flight and stand record is the obvious next step, once there's enough of an audit trail to trust doing it automatically for at least the highest-confidence cases. **AWS IoT SiteWise** asset models are part of the target architecture for a real facility rollout but aren't provisioned in this demo — Timestream for InfluxDB is sufficient for the dashboard and the anomaly-detection training today, and SiteWise's asset hierarchy modelling becomes worth the extra infrastructure once there's a real building management system behind it rather than simulated sensors. And the log-only decision itself is a v1 posture, not a permanent one — the path to governed autonomy for the highest-confidence, lowest-risk proposal types is a policy decision to make deliberately, once there's enough real approval-rate data to make it with evidence rather than optimism.


## Summary ##

The AODB doesn't need to be replaced to become materially more useful — it needs a reasoning layer on top of it that can fuse flight data with facility telemetry, flag risk before it becomes a conflict, and propose evidenced actions that a human still has to approve. The architecture that gets you there isn't exotic: a time-series store and an operational database for the data, SageMaker for anomaly detection, Bedrock with forced tool-use for structured, evidenced proposals, and a governance posture — log-only, human-approved, conservative by design — that earns the right to eventually do more.

If you're thinking through what an AI-augmented AODB or airport operations platform could look like for your own environment, feel free to reach out.
