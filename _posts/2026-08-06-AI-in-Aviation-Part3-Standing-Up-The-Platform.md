---
layout: post
title: AI in Aviation, Part 3 - Standing Up the AI Platform on AWS
---

## Introduction ##

[Part 1]({{ site.baseurl }}/AI-in-Aviation-Part1-Data-Lake/) of this series built the data lake — S3 zones, Iceberg tables, Glue Data Catalog, and Lake Formation access control. [Part 2]({{ site.baseurl }}/AI-in-Aviation-Part2-Model-Governance-Security/) established how to choose models by risk tier and how AgentCore's Identity, Gateway, Runtime, Memory, and Observability components enforce that tiering for anything built as an agent. This post is where those two pieces actually get built.

This is the third post in the four-part **AI in Aviation** series:

- **Part 1** — Building a data lake foundation on AWS so aviation data is usable, governed, and AI-ready rather than trapped in silos
- **Part 2** — Model selection, AI governance, and security for aviation workloads
- **Part 3 (this post)** — Standing up the AI platform: Amazon Bedrock, agents, and the tooling around them
- **Part 4** — Bringing it together: AI applications running on top of the data lake, with a working example

Everything here is deliberately grounded in the tiering from Part 2. Nothing in this post is a generic "how to use Bedrock" walkthrough — every choice, from vector store to guardrail configuration to agent tool scope, is made with reference to which tier the use case sits in and what that tier demands.

The architecture diagrams in this post have been created using [Eraser.io](https://www.eraser.io/).


## Enabling Bedrock and Selecting Models per Tier ##

Model access in **Amazon Bedrock** is opt-in per model, per account, per region — the first practical step is requesting access to the specific model families the Part 2 tiering actually needs, rather than enabling everything by default. For most aviation deployments that's a fast, low-cost model family (Amazon Nova Micro or Lite) for Tier 1, and a stronger reasoning model family (Claude) for Tier 2 and Tier 3.

A few decisions worth making deliberately at this stage:

- **On-demand vs Provisioned Throughput** — Tier 1 use cases running against high flight-event volume benefit from Provisioned Throughput for predictable latency and cost at scale; lower-volume Tier 2/3 use cases are usually better served on-demand, where the fixed cost of provisioned capacity isn't justified
- **Cross-region inference profiles** — for resilience, routing inference across multiple regions within the same data residency boundary established in the [Landing Zone]({{ site.baseurl }}/AWS-Aviation-Landing-Zone/) rather than depending on a single region's capacity
- **IAM invocation roles scoped per tier** — the IAM role a Tier 1 summarisation Lambda assumes to call Bedrock should not be the same role a Tier 2 agent uses; separating them means a compromised or misconfigured Tier 1 workload can't reach the models or guardrail configurations reserved for higher-stakes use cases

None of this is exotic AWS configuration — the discipline is in refusing to let model access sprawl beyond what the tiering from Part 2 actually calls for.


## Grounding Models in the Data Lake: Bedrock Knowledge Bases ##

Retrieval-augmented generation is what turns a general-purpose model into one that actually knows about a specific aircraft's maintenance history or a specific flight's disruption state, and **Amazon Bedrock Knowledge Bases** is the managed RAG layer that sits directly on top of the Part 1 data lake.

- **Data source** — the Knowledge Base points at the curated and consumption zones of the S3 data lake, the same Iceberg tables described in Part 1, rather than a separate copy of the data maintained just for AI. One governed dataset, multiple consumption paths
- **Embeddings and vector store** — Amazon Titan Text Embeddings (or a Cohere embedding model, depending on multilingual requirements for international operations) generates vector representations, stored in Amazon OpenSearch Serverless's vector engine for larger-scale deployments, or Aurora PostgreSQL with pgvector where the team already runs Aurora and wants to avoid standing up a new service
- **Sync strategy** — the Knowledge Base ingestion job runs on a schedule aligned with the Glue ETL cadence from Part 1, so retrieval reflects curated data that's actually gone through conformance and quality checks, not raw landing-zone data
- **Metadata filtering for governed retrieval** — this is the part that matters most for aviation. Knowledge Base queries support metadata filters, and those filters should mirror the LF-Tag classification from Part 1's Lake Formation model. A crew-scoped agent's retrieval calls carry a filter restricting results to non-PII, crew-permitted content; a maintenance agent's calls filter to engineering data. The access control isn't reinvented for AI — it's projected from Lake Formation into the retrieval layer

### Knowledge Base architecture on the data lake ###

![_config.yml]({{ site.baseurl }}/images/blog/AI-in-Aviation/Diagram7.png)


## Configuring Guardrails per Tier ##

**Amazon Bedrock Guardrails** get created and attached per use case, not once globally, because a Tier 1 report-summariser and a Tier 3 maintenance-judgement assistant need meaningfully different configurations:

- **Denied topics and content filters** — broader and stricter for Tier 2/3, where an off-topic or speculative response carries real consequence; lighter for Tier 1, where the cost of over-blocking a legitimate summarisation request outweighs the marginal risk
- **PII redaction** — applied consistently across all tiers wherever passenger or crew data is in scope, independent of the tier's other settings
- **Contextual grounding checks** — this is the setting that matters most for Tier 2 and 3. A grounding check verifies a response is actually supported by the Knowledge Base content retrieved for that query, and rejects or flags responses that go beyond it. For a maintenance advisory use case, this is what stops a model from stating an EGT margin or MEL status that isn't present in the retrieved telemetry — the mechanism described conceptually in Part 2, configured concretely here
- **Guardrail versioning** — guardrail configurations are versioned resources in Bedrock; treat them the same way as the infrastructure they protect, promoted through review rather than edited directly in production

Every Tier 2 and Tier 3 use case should have its guardrail configuration reviewed as part of the same sign-off process from Part 2 that approves the use case's tier placement in the first place — the guardrail is the enforcement mechanism for that approval, not a separate technical decision made independently of it.


## Building an Agent with Bedrock AgentCore ##

For any Tier 2 or Tier 3 use case that reasons across multiple steps and calls tools — the disruption triage and welfare-scored re-accommodation example from Parts 1 and 2 being the running example — the build sits on **Bedrock AgentCore**:

- **AgentCore Gateway** — register the specific tools the agent needs as explicit, scoped entries: a read-only query against the flight/passenger consumption tables in the data lake, a read against the assistance-booking system, and (deliberately) no write access to the rebooking engine itself, since Part 2 established that this use case surfaces recommendations to a human rather than acting autonomously
- **AgentCore Identity** — the agent's runtime identity is configured with OAuth delegation, so when an operations agent invokes it, the agent's data access reflects that specific operator's permissions rather than a standing broad service role
- **AgentCore Runtime** — the agent is deployed as a versioned, session-isolated workload; each disruption event gets its own session, with no state carried over between unrelated disruptions or between different operators using the same agent definition
- **AgentCore Memory** — configured with a short retention window scoped to the active disruption event only. There's no business need for this agent to remember Mary O'Sullivan's welfare score after her re-accommodation is resolved, so memory retention is set to expire with the session rather than defaulting to indefinite retention
- **AgentCore Observability** — every reasoning step and tool call in the session is traced and shipped to CloudWatch, landing in the same Log Archive account from the Landing Zone that holds every other audit trail in the environment

Put together, this is the concrete implementation of the governance and security architecture from Part 2 — not a new design, just the actual AWS resources that make it real.

### AgentCore build-out for a Tier 2 disruption agent ###

![_config.yml]({{ site.baseurl }}/images/blog/AI-in-Aviation/Diagram8.png)


## Multi-Agent Orchestration for Cross-Domain Scenarios ##

The disruption cascade from Part 1 — a late inbound flight affecting crew FDT and connecting passengers — spans three domains that shouldn't be handled by one broad agent with access to everything. The better pattern is a narrow **orchestrator agent** that sequences calls to separate, tightly scoped domain agents:

- A **flight operations agent**, scoped to flight and delay data only
- A **crew agent**, scoped to roster and FDT data, with no passenger data access
- A **passenger welfare agent**, scoped to passenger and assistance-booking data, with no crew data access

The orchestrator holds the workflow logic — "check crew impact, check passenger impact, consolidate for the human reviewer" — but each domain agent's Gateway tool allowlist stays exactly as narrow as that domain requires. This keeps the least-privilege principle from Part 2 intact even as the use case gets more complex: adding a new domain means adding a new narrowly-scoped agent, not widening an existing one's access.

### Multi-agent orchestration pattern ###

![_config.yml]({{ site.baseurl }}/images/blog/AI-in-Aviation/Diagram9.png)


## Deploying the Platform as Infrastructure as Code ##

Consistent with the rest of this series, none of the above gets configured by hand in the console. Knowledge Base definitions, Guardrail configurations, AgentCore Gateway tool registrations, and agent definitions are all managed as **OpenTofu** modules, deployed through the same CI/CD pipeline described in the Landing Zone post — `tofu validate` and `tofu plan` on every pull request, `tofu apply` gated on review.

This matters more here than for typical infrastructure, because the "infrastructure" in this case includes the governance controls themselves. A guardrail configuration or a Gateway tool allowlist is a security boundary, and changes to it should go through the same review and audit trail as a change to an IAM policy or a security group — not be treated as a lower-stakes application configuration change.

Before any Tier 2 or 3 agent is promoted past a development environment, it goes through the evaluation and red-teaming step from Part 2 — testing not just whether it produces correct answers, but whether it stays within its guardrails and tool scope when the input is adversarial or the retrieved context is misleading. Bedrock's evaluation tooling supports this as a repeatable step in the same pipeline, rather than a one-off manual exercise before launch.


## Summary ##

The platform is now assembled: models selected and access-scoped per the Part 2 tiering, Bedrock Knowledge Bases grounding retrieval in the governed data lake from Part 1 with LF-Tag-aligned metadata filtering, Guardrails enforcing the right level of scrutiny per tier, and AgentCore providing the identity, tool scoping, session isolation, memory discipline, and observability that Tier 2 and 3 agents require — all deployed as version-controlled infrastructure rather than console configuration.

None of this is useful in isolation. It's a platform, waiting for an application to run on it.

In Part 4, I'll bring this together with a working example — an AI application built on the data lake and platform from this series, using the disruption and predictive maintenance scenarios from SkyConnect as the concrete walkthrough.

If you're working through standing up an AI platform for an aviation or similarly regulated environment and want to discuss the approach, feel free to reach out.
