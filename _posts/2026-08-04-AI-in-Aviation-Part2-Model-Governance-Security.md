---
layout: post
title: AI in Aviation, Part 2 - Model Selection, Governance, and Security
---

## Introduction ##

In [Part 1]({{ site.baseurl }}/AI-in-Aviation-Part1-Data-Lake/) of this series I covered why aviation AI initiatives fail before they start: the data is scattered across PSS, AODB, MRO, ACARS, and crew systems, and no model does anything useful pointed at that mess directly. The fix was a governed data lake — S3 zones, Glue Data Catalog, and Lake Formation access control tied back to the account and identity structure from the [Landing Zone]({{ site.baseurl }}/AWS-Aviation-Landing-Zone/).

With that foundation in place, the next question is what actually gets to touch it. This is the second post in the four-part **AI in Aviation** series:

- **Part 1** — Building a data lake foundation on AWS so aviation data is usable, governed, and AI-ready rather than trapped in silos
- **Part 2 (this post)** — Model selection, AI governance, and security for aviation workloads
- **Part 3** — Standing up the AI platform: Amazon Bedrock, agents, and the tooling around them
- **Part 4** — Bringing it together: AI applications running on top of the data lake, with a working example

Aviation adds two constraints most industries don't have to weigh as heavily: safety-criticality and regulatory scrutiny. A model that summarises marketing copy badly is an inconvenience. A model that confidently misstates an engine health parameter, or auto-confirms a rebooking for a passenger who needed assistance, is a different category of problem. Model selection in this context isn't "which model is cheapest and fastest" — it's "which model, with what guardrails, is allowed to see what data, take what actions, and who is accountable when it gets something wrong."

The architecture diagrams in this post have been created using [Eraser.io](https://www.eraser.io/).


## A Risk-Tiered Approach to Model Selection ##

The instinct with a new platform capability is to pick one model and use it everywhere. That's the wrong starting point for aviation. Different use cases carry entirely different consequences when a model gets something wrong, and the right response is to tier use cases by risk and match the model, the guardrails, and the human oversight level to the tier — not to find one model that's cautious enough to be safe everywhere and capable enough to be useful everywhere, because that model doesn't exist.

A workable tiering looks like this:

- **Tier 1 — Advisory and analytics.** Summarising OTP trends, drafting a shift handover report, answering "what were yesterday's top three delay causes." Wrong or imprecise output here is inconvenient, not dangerous. This tier has the most latitude on model choice and the least overhead in guardrails.
- **Tier 2 — Operational decision support.** Recommending which disrupted passengers need manual re-accommodation rather than automated rebooking, flagging an aircraft for unscheduled maintenance review, suggesting a standby crew swap ahead of an FDT breach. These outputs influence real decisions with real cost and welfare implications. Human-in-the-loop is mandatory — the model recommends, a person decides — exactly the pattern the passenger welfare scoring in [Part 1]({{ site.baseurl }}/AI-in-Aviation-Part1-Data-Lake/) used to keep a vulnerable passenger out of standard automated rebooking.
- **Tier 3 — Safety-adjacent.** Anything touching airworthiness-adjacent maintenance judgement or crew legal compliance directly. This tier demands the highest scrutiny: full auditability, mandatory human sign-off, and a documented case for why a model is involved at all rather than a deterministic rules engine.

This tiering is a governance decision before it's a technical one, and it should be made by the same people who'd be accountable if it went wrong — not defaulted to whichever tier is most convenient for the engineering team building the feature.

### Model selection framework ###

![_config.yml]({{ site.baseurl }}/images/blog/AI-in-Aviation/Diagram4.png)


## Choosing Models on Amazon Bedrock ##

**Amazon Bedrock** is the right starting point for aviation AI workloads because it decouples the application from any single model provider. Claude, Amazon Nova, Llama, and Mistral models are all available behind the same API, which matters more in this context than it might elsewhere: as models improve or as pricing shifts, you want to swap the model behind a Tier 1 summarisation feature without re-architecting the application around it.

Selection criteria differ by tier:

- **Context window and retrieval quality** — Tier 2 and 3 use cases typically involve retrieval-augmented generation against the curated data lake (more on this in Part 3), joining flight, maintenance, and crew context in a single reasoning pass. A larger, more reliable context window matters more here than raw throughput.
- **Reasoning quality on multi-step scenarios** — tracing a disruption cascade from a delayed inbound flight through crew FDT to connecting passenger risk, as in the SkyConnect example from Part 1, is a multi-hop reasoning task, not a single lookup. Cheaper, smaller models tend to degrade badly on this kind of chained inference.
- **Cost and latency at operational scale** — a Tier 1 use case running against every flight event across a network of any size needs a fast, inexpensive model. Reserve the strongest (and most expensive) models for the Tier 2/3 use cases where getting it right matters more than getting it cheap.
- **Data residency** — for European operators, inference needs to happen in-region. Bedrock's regional availability needs to be checked against the same data residency requirements that shaped the account and network design in the Landing Zone.

In practice this leads to a portfolio, not a single model: a fast, low-cost model (Nova Micro/Lite class) handling high-volume Tier 1 classification and summarisation, and a stronger reasoning model (Claude class) reserved for the Tier 2 and Tier 3 use cases where the cost per request is justified by the stakes of getting it right.


## AI Governance: Extending Access Control to a New Consumer ##

Governance for AI in this architecture isn't a separate system bolted on afterwards — it's the same access control problem from [Part 1]({{ site.baseurl }}/AI-in-Aviation-Part1-Data-Lake/), extended to a new class of consumer. The Lake Formation LF-Tags that scope a crew scheduling analyst to crew data and keep an MRO engineer out of commercial data apply exactly the same way to a Bedrock Knowledge Base or an agent's retrieval role. An AI service role is just another persona in that permission model — it should never have broader access to the lake than the least-privileged human user it's acting on behalf of.

On top of that data-access layer, three additional controls matter specifically for aviation:

- **Bedrock Guardrails** — configurable denied topics, PII redaction, and grounding checks that verify a response is actually supported by retrieved content rather than generated from the model's general knowledge. Grounding checks matter enormously here: a maintenance advisory tool must not state an EGT margin or an MEL status that isn't actually present in the retrieved telemetry or work order.
- **Human-in-the-loop as a governance control, not a UX nicety** — for Tier 2 and Tier 3 use cases, the model's role is to surface and rank, not to act. This is the same pattern as the welfare-scored re-accommodation from Part 1: the AI pulls the exception cases out of standard automation and puts them in front of a person, rather than being trusted to close the loop itself.
- **Full invocation audit trail** — every model call (prompt, retrieved context, output, and the human decision that followed, where applicable) logged and retained. This lands in the same Log Archive account and centralised logging pipeline established in the Landing Zone, so AI activity is subject to the same audit and retention posture as everything else in the environment, rather than living in a separate, less scrutinised system.

Once a use case moves beyond a single request/response model call into something that reasons across multiple steps and calls tools — which is where most Tier 2 and Tier 3 use cases end up — the governance surface changes shape, and this is specifically what **Amazon Bedrock AgentCore** is built to address:

- **AgentCore Gateway** turns existing systems — the data lake's query interface, an MRO work order API, a rebooking engine — into explicitly scoped, agent-callable tools. An agent's tool inventory becomes a deliberate, reviewable allowlist rather than an open-ended set of API credentials handed to a model
- **AgentCore Observability** provides end-to-end tracing of an agent's reasoning steps and tool calls per session, feeding CloudWatch. This is the concrete mechanism behind the audit trail requirement above — not just "what did the model output," but the full chain of tool calls and intermediate reasoning that led there, which matters when reconstructing why an agent made a Tier 2 recommendation
- **AgentCore Memory** lets agents retain context across sessions, which is genuinely useful for continuity but needs the same governance discipline as everything else in the lake: memory that accumulates crew or passenger details outside the LF-Tag-governed data model is a shadow copy of exactly the data you built Lake Formation to control. Memory retention scope and expiry should be a deliberate governance decision per use case, not left at a permissive default

Standing up a use case in Tier 2 or 3 should go through a sign-off process involving whoever owns safety and compliance accountability for that domain, not just the engineering team that built it. That approval — which use cases exist, at which tier, with which guardrails — is itself a governance artefact worth keeping under version control alongside the infrastructure that enforces it.

### AI governance architecture ###

![_config.yml]({{ site.baseurl }}/images/blog/AI-in-Aviation/Diagram5.png)


## Security Architecture for AI Workloads ##

AI workloads don't get a separate security model — they run inside the same Landing Zone account and network structure as everything else, which is deliberate: it means AI-specific risk doesn't require reinventing the controls that already exist.

- **Account placement** — Bedrock, any SageMaker workloads, and agent orchestration sit in a dedicated account within the Workload OU, subject to the same SCPs, Config conformance packs, and GuardDuty coverage as every other workload account in the [Landing Zone]({{ site.baseurl }}/AWS-Aviation-Landing-Zone/).
- **Data access via IAM, scoped by Lake Formation** — Knowledge Bases and agents read the curated and consumption zones through IAM roles with Lake Formation permissions, never through direct S3 bucket policies. This keeps the LF-Tag model as the single source of truth for who and what can see which data, human or machine.
- **Private network path** — Bedrock VPC endpoints (AWS PrivateLink) keep model invocation traffic off the public internet entirely, consistent with the Cloud WAN network segmentation covered in the Landing Zone.
- **Least-privilege per agent, via AgentCore Identity** — rather than one broad IAM role shared across every agent capability, **AgentCore Identity** issues each agent its own scoped credentials, and — critically for Tier 2/3 use cases — supports OAuth-based delegation so an agent can act with the specific permissions of the human operator it's assisting, rather than a standing service-level permission set. A maintenance agent assisting an MRO engineer inherits that engineer's actual access boundary, not a broader one provisioned for convenience.
- **Session isolation via AgentCore Runtime** — each agent invocation runs in its own isolated, session-scoped execution environment. In a platform serving multiple departments (ops, MRO, crew) or, in a multi-airline deployment, multiple operators from shared infrastructure, this prevents context or state from one session leaking into another — the same isolation principle the Landing Zone applies at the account level, now applied at the agent session level.
- **Prompt injection and exfiltration risk** — an agent with tool access to crew or passenger data can, in principle, be manipulated by adversarial content embedded in the data it retrieves (a crafted maintenance log comment, a passenger free-text field) into leaking data across an LF-Tag boundary or invoking a tool it shouldn't. Guardrails, AgentCore Gateway's scoped tool allowlisting, and output filtering are the mitigations — and this is precisely why Tier 2/3 agents shouldn't hold write access to anything they don't need for the specific task in front of them.

Before any Tier 2 or 3 use case reaches production, it should go through evaluation and red-teaming specifically targeting these failure modes — not just accuracy testing against a golden dataset, but adversarial testing of what the model does when the retrieved context is misleading or the input is deliberately crafted to provoke an unwanted action.

### AI security architecture within the Landing Zone ###

![_config.yml]({{ site.baseurl }}/images/blog/AI-in-Aviation/Diagram6.png)


## Worked Example: Revisiting the Welfare-Scored Re-Accommodation ##

It's worth returning to the SkyConnect passenger welfare example from Part 1 through this lens, because it's a clean illustration of the framework holding together end to end.

The use case sits squarely in **Tier 2**: it influences a real operational decision (how a disrupted passenger gets rebooked) with a real welfare and cost dimension, but it isn't safety-adjacent in the Tier 3 sense. The governance model that follows from that tier placement is exactly what Part 1 described — the model scores and flags, a human agent decides — and the guardrails that follow from Tier 2 placement would include a grounding check to ensure the welfare score is actually derived from real assistance-booking and itinerary data rather than a plausible-sounding guess, plus full logging of every scored decision and the human outcome that followed it, landing in the same audit trail as the rest of the platform.

None of that requires a separate governance system for the AI feature. It requires the AI feature to be honest about which tier it belongs to, and the platform to already have the tiering, guardrails, and logging in place to enforce it. Built as an agent, this use case would run as a single AgentCore-hosted agent with a Gateway allowlist limited to the itinerary, assistance-booking, and welfare-scoring tools it actually needs — nothing broader.


## Summary ##

Model selection for aviation AI is a portfolio decision driven by risk tier, not a single model choice made once. Governance is not a separate concern bolted onto the model layer — it's the Lake Formation access control model from Part 1, extended to cover AI as a new class of consumer, with grounding checks and human-in-the-loop decision points added specifically because the cost of a confident wrong answer is higher here than in most domains. Security for AI workloads reuses the account, network, and identity architecture from the Landing Zone rather than requiring a parallel security model, and for anything built as an agent, **Bedrock AgentCore** — Identity, Gateway, Runtime, Observability, and Memory — is what makes that reuse practical rather than aspirational.

In Part 3, I'll walk through actually standing up this platform on AWS — Bedrock configuration, Knowledge Bases against the Part 1 data lake, and building agents within the tiering and guardrails established here.

If you're working through model selection or AI governance for an aviation or similarly regulated environment and want to discuss the approach, feel free to reach out.
