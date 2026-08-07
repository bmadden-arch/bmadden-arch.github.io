---
layout: post
title: AI in Aviation, Part 4 - Bringing It Together on SkyConnect
---

## Introduction ##

Across this series, SkyConnect has been the running example: a real, deployed Operational Command Centre demo, built on the data lake architecture from [Part 1]({{ site.baseurl }}/AI-in-Aviation-Part1-Data-Lake/), with an AI layer designed against the tiering and governance model from [Part 2]({{ site.baseurl }}/AI-in-Aviation-Part2-Model-Governance-Security/) and built on the Bedrock and AgentCore platform from [Part 3]({{ site.baseurl }}/AI-in-Aviation-Part3-Standing-Up-The-Platform/).

This is the final post in the four-part **AI in Aviation** series:

- **Part 1** — Building a data lake foundation on AWS so aviation data is usable, governed, and AI-ready rather than trapped in silos
- **Part 2** — Model selection, AI governance, and security for aviation workloads
- **Part 3** — Standing up the AI platform: Amazon Bedrock, agents, and the tooling around them
- **Part 4 (this post)** — Bringing it together: AI applications running on top of the data lake, with a working example

SkyConnect's own backlog has carried an entry for exactly this for a while: **Bedrock, a Bedrock Knowledge Base, an AI chat bot, and AgentCore**. The Knowledge Base source documents have been sitting in the repository since early on — two engineering reference documents, written and ready, waiting for the platform to actually use them. This post is that backlog item, designed end to end using everything the previous three posts built.

The architecture diagrams in this post have been created using [Eraser.io](https://www.eraser.io/).


## What's Already There ##

Two things exist today that make this backlog item tractable rather than a blank-page exercise.

The first is the data lake itself: the real, deployed pipeline from Part 1 — ACARS telemetry through Kinesis and Firehose into the S3 raw zone, a Glue Crawler cataloguing it hourly, DynamoDB holding live fleet state, all queryable through Athena. This is running infrastructure, not a diagram.

The second is a pair of engineering reference documents already sitting in the repository, written for a Bedrock Knowledge Base that hasn't been built yet:

- **SK-ENG-001, CFM56-5B Engine Health Monitoring Reference** — the actual EGT margin, oil consumption, vibration, and oil pressure thresholds SkyConnect's engineering team uses to classify an engine as Healthy, Watch, Alert, or Critical, plus the currently active Airworthiness Directives and which tail numbers they apply to
- **SK-ENG-002, Maintenance Policy** — the escalation chain (who gets notified, within what timeframe, for what kind of finding), the AOG response procedure, and the cost of an AOG event (€80,000–€120,000 per day)

These are exactly the kind of unstructured, policy-heavy documents that live telemetry alone can't answer questions about. A dashboard can show that EI-SKJ's Engine 1 EGT margin is 5.0°C. Only SK-ENG-001 says that's below the 25°C critical threshold, and only SK-ENG-002 says what happens next.


## Applying the Tiering to the Backlog ##

Before building anything, the four backlog items get sorted through the Part 2 risk framework, because they don't all belong in the same tier:

| Backlog item | Tier | Why |
|---|---|---|
| AI chat bot — general OCC Q&A | **Tier 1** | Read-only questions against live dashboard data ("what's SK552's status," "summarise today's OTP"). Wrong or imprecise answers are inconvenient, not dangerous |

| Bedrock Knowledge Base — maintenance advisory | **Tier 2** | Answers inform real engineering decisions about aircraft status and AD compliance. Requires grounding checks and a human engineer in the loop |

| AgentCore — disruption / crew / welfare orchestration | **Tier 2** | The recurring example from Parts 1–3: recommends, a human decides |

| Anything that would autonomously action a finding — opening an AMOS work order, grounding an aircraft, notifying the VP Engineering | **Out of scope for AI action** | SK-ENG-002 defines this as a human escalation chain with named accountable roles and hard time limits. The AI's role is to prepare and route that notification correctly and fast, never to send it unsupervised |

That last row matters as much as the tier assignments themselves. Nothing in this build gives a model or agent the ability to trigger an unplanned engine removal or ground an aircraft. That accountability stays exactly where SK-ENG-002 already puts it — with named human roles on defined clocks.


## Building the Knowledge Base from What Already Exists ##

Following the Part 3 pattern, the Bedrock Knowledge Base is pointed at two data sources, not one: the unstructured engineering documents (SK-ENG-001 and SK-ENG-002, sitting in S3 alongside the data lake) and the curated telemetry tables from Part 1. Retrieval spans both, which is what makes a grounded answer actually useful rather than just accurate-but-useless in isolation.

Asked "what's the status of EI-SKJ and what do I need to do," the response draws Engine 1's live EGT margin and oil consumption from the curated telemetry table, the definition of Critical status and the applicable Airworthiness Directive from SK-ENG-001, and the notification requirements — Maintenance Control immediately, Airworthiness Manager for AD-related findings — from SK-ENG-002, combined into one grounded response with both sources cited. Neither source alone answers the question completely; the Knowledge Base's job is to combine them without inventing anything that isn't in either.

This is the same live telemetry from Part 1 — Engine 1's EGT margin at 5.0°C, oil consumption at 2.11 L/hr, three open MEL/CDL items — that a grounded response would cite directly, alongside the SK-ENG-001 threshold and SK-ENG-002 escalation text it doesn't have access to today:

![SkyConnect predictive maintenance detail for aircraft EI-SKJ, the live telemetry the maintenance Knowledge Base would ground its response in]({{ site.baseurl }}/images/blog/AI-in-Aviation/screenshots/SkyConnect-Predictive-Maintenance.png)

### Knowledge Base spanning telemetry and engineering documents ###

![_config.yml]({{ site.baseurl }}/images/blog/AI-in-Aviation/Diagram10.png)


## The Chat Bot: A Tier 1 Front End on the OCC ##

The chat bot is a thin conversational layer embedded in the OCC dashboard, backed by the fast, low-cost model tier from Part 3, answering exactly the kind of question the dashboard already displays but in free-form language rather than requiring a shift manager to know which tab it's on: current OTP, which aircraft are airborne, what's on the disruption desk right now, a shift-handover summary.

Its guardrail configuration is deliberately narrow in one specific direction: it's scoped to refuse — and hand off — anything that looks like a maintenance judgement or an escalation decision. "Is EI-SKJ safe to fly" is not a Tier 1 question, and the chat bot's denied-topic configuration routes it to the maintenance advisory agent rather than attempting a confident answer outside its tier.


## The Maintenance Advisory Agent: Tier 2, Grounded, Human-Reviewed ##

This is where the AgentCore pattern from Part 3 does the real work. Built as a scoped agent — Gateway access limited to reading the curated telemetry tables and the Knowledge Base, no write access to AMOS — it takes a duty engineer's question about a specific aircraft and produces a grounded brief: current parameter values against threshold, applicable AD status, and the exact notification SK-ENG-002 requires, drafted and ready.

For EI-SKJ specifically, that means a drafted notification to Maintenance Control citing the 5.0°C EGT margin against the 25°C critical threshold, a note that AD SK-2024-CFM-001 already applies to this airframe at its current cycle count, and a flag that the Airworthiness Manager notification requirement is triggered. The agent prepares it. The duty engineer reviews it, edits it if needed, and sends it. That last step is deliberately not automated — it's the same human-in-the-loop boundary from Part 2, applied to a use case where the cost of an unreviewed false positive (or worse, a missed true positive) is measured in the €80,000–€120,000 per day SK-ENG-002 puts on an AOG event.


## Reconnecting the Disruption and Crew Orchestration ##

The multi-agent orchestration pattern from Part 3 — an orchestrator sequencing a flight operations agent, a crew agent, and a passenger welfare agent, each scoped to its own domain — is the same architecture already walked through in detail across this series, now simply running against the real SK552/SK553 disruption scenario from Part 1: Captain Brennan's FDT projection, the welfare-scored re-accommodation that keeps a vulnerable passenger out of standard automated rebooking, all consolidated for a human reviewer rather than acted on autonomously. Nothing new needs to be designed here — it's the direct product of Parts 2 and 3, wired into the same OCC front end as the chat bot and the maintenance advisory agent.


## Closing the Loop on the Backlog ##

| Backlog item | Delivered by |
|---|---|
| Amazon Bedrock | Part 3 — model access, tiered selection, Guardrails configuration |

| Bedrock Knowledge Base | Part 1's data lake as the source, Part 3's KB architecture, this post's dual structured/unstructured grounding |

| AI chat bot | This post — Tier 1, scoped, hands off anything outside its tier |

| Amazon Bedrock AgentCore | Part 2's governance model, Part 3's build-out, this post's maintenance advisory and disruption/crew/welfare agents |

Each post in this series did one piece of work that made the next one possible: Part 1 gave the AI something real and governed to be grounded in. Part 2 decided, before any of it was built, what each use case was allowed to do and who stays accountable. Part 3 turned that decision into actual AWS configuration. This post is where it stops being architecture and starts being a feature a duty engineer or a shift manager would actually use.


## Summary ##

The point of this series was never that AI in aviation is hard because the models aren't good enough. It's that the data is scattered, the stakes of a wrong answer vary enormously by use case, and most of the engineering effort belongs in the foundation — the data lake, the governance model, the platform — rather than in prompt engineering. Get that foundation right, as SkyConnect's own backlog shows, and the AI features themselves are often the smallest part of the build.

If you're working through a data lake, AI governance, or AI platform strategy for an aviation or similarly regulated environment — or want to talk through any part of this series in more depth — feel free to reach out.
