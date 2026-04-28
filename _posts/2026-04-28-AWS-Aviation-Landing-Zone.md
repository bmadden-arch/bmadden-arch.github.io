---
layout: post
title: Building an AWS Landing Zone for Aviation
---

## Introduction ##

Aviation is one of the most operationally complex industries in the world. Systems need to be available around the clock, data flows across airlines, airports, ground handlers, and maintenance providers, and regulatory requirements — from IATA standards to regional data sovereignty rules — add significant compliance obligations on top of the usual security baseline.

When moving workloads to AWS, aviation organisations cannot simply lift-and-shift into a single account and call it a cloud strategy. The foundation matters enormously. A well-designed landing zone determines how scalable, secure, and auditable everything built on top of it will be — for years to come.

This post walks through the architecture of an AWS Landing Zone we have deployed for a major aviation operator, built to support multi-account governance, centralised networking, identity federation, and strong compliance posture from day one. The architecture is available as a professional services engagement via the [AWS Marketplace](https://aws.amazon.com/marketplace/pp/prodview-s7bklekkdkw3w).

The architecture diagrams in this post have been created using [Eraser.io](https://www.eraser.io/).

---

## Why Aviation Needs a Purpose-Built Landing Zone ##

Generic landing zone guides cover the fundamentals well, but aviation introduces specific pressures that shape architecture decisions:

- **Operational continuity**: Ground operations, passenger services, and cargo handling cannot tolerate extended outages. Resilience must be built into the foundation, not bolted on later.
- **Data segregation**: Airlines, MRO providers, and cargo operators often run on shared infrastructure but cannot share data. Account-level isolation enforces hard boundaries.
- **Regulatory compliance**: Aviation operators are subject to frameworks including GDPR, NIS2, and industry-specific mandates. Audit trails, encryption at rest, and configuration drift detection are non-negotiable.
- **Third-party integrations**: GDS connections, airport systems, ground handler APIs, and maintenance databases mean the network perimeter is complex and must be explicitly managed.
- **Team structure**: Operations teams, engineering teams, and commercial teams have entirely different access requirements. Federated identity with fine-grained role separation is essential.

These constraints directly influence the account structure, networking design, and security controls described below.

---

## Account Structure and AWS Organisations ##

The foundation of the landing zone is a multi-account structure built on **AWS Organisations**. Rather than attempting to isolate workloads within a single account using VPCs and IAM policies alone, the architecture uses accounts as the primary security boundary.

Accounts are grouped into **Organisational Units (OUs)** that reflect the operational structure of the business:

- **Management OU** — the root management account; used exclusively for billing, account vending, and organisational policies. No workloads run here.
- **Infrastructure OU** — contains the Network Account (Cloud WAN, Direct Connect, firewall) and the Shared Services Account (directory services, internal tooling).
- **Security OU** — contains the Log Archive Account (centralised CloudTrail and VPC Flow Logs) and the Security Tooling Account (Security Hub, GuardDuty, Config aggregation).
- **Workload OUs** — separate OUs for Production, UAT, and Development workloads. Each environment sits in its own account, with SCPs enforcing what is permitted at each tier.

**Service Control Policies (SCPs)** are applied at the OU level to enforce guardrails that cannot be overridden by account administrators — for example, preventing the disabling of CloudTrail, restricting regions to approved locations, and blocking the creation of internet gateways outside of approved accounts.

![_config.yml]({{ site.baseurl }}/images/blog/AWS-Aviation-Landing-Zone/Diagram1.png)

---

## Governance with AWS Control Tower ##

**AWS Control Tower** sits above the account structure and automates the enforcement of governance baselines. Rather than manually configuring each account, Control Tower provides:

- **Proactive guardrails** that prevent non-compliant resources from being created in the first place
- **Detective guardrails** (backed by AWS Config rules) that identify drift from the baseline and report to the Security Tooling Account
- **Account Factory** for provisioning new accounts with pre-applied baselines — critical when onboarding new workload teams or standing up environments for new business units
- **Audit and logging baselines** deployed automatically to every enrolled account, including CloudTrail organisation trails and Config recorders

For the aviation deployment, Control Tower guardrails were selected and tuned to align with the organisation's compliance requirements. Controls covering encryption enforcement, S3 public access blocking, and IAM password policy were enabled from the outset. Additional custom Config rules were layered on top via the Security Tooling Account to address aviation-specific requirements.

---

## Identity and Access Management ##

### AWS IAM Identity Centre ###

Federated identity is managed through **AWS IAM Identity Centre** (formerly AWS SSO). This provides a single point of authentication for all AWS account access across the organisation, with access granted through permission sets that map to job roles rather than individual users.

The architecture integrates IAM Identity Centre with the organisation's existing identity provider, enabling engineers, operations teams, and management to authenticate using their existing corporate credentials. Access is role-based:

- **Platform engineers** have elevated access to Infrastructure and Security accounts with limited production access
- **Application engineers** have scoped access to their workload accounts only, with no access to centralised security or logging accounts
- **Security and compliance teams** have read-only access to the Security Tooling Account and Log Archive Account
- **Operations teams** have production read access and specific operational permissions aligned to their runbooks

Permission sets are version-controlled and deployed through the Infrastructure as Code pipeline, meaning access changes are audited, reviewed, and applied consistently.

![_config.yml]({{ site.baseurl }}/images/blog/AWS-Aviation-Landing-Zone/Diagram2.png)

---

## Centralised Networking ##

### VPC Design ###

Each workload account has its own VPC with a consistent CIDR allocation drawn from a non-overlapping address plan. VPCs are structured with three subnet tiers per Availability Zone:

- **Public subnets** — for NAT Gateways and load balancer endpoints only; no compute runs here
- **Private application subnets** — for compute workloads (EC2, ECS, Lambda VPC attachments)
- **Private data subnets** — for RDS instances, ElastiCache, and other data tier resources

Subnet isolation is reinforced with Network ACLs and Security Groups following a least-privilege model.

### AWS Cloud WAN ###

Connectivity between accounts and to on-premises systems is built on **AWS Cloud WAN**, deployed from the centralised Network Account. Cloud WAN replaces the traditional Transit Gateway hub-and-spoke model with a managed global network backbone, giving greater flexibility for multi-region growth and simplifying the operational model for network segmentation.

The Cloud WAN deployment consists of:

- **Global Network** — the top-level container managed in the Network Account, providing a single pane of glass for all connectivity across the organisation
- **Core Network** — the policy-driven backbone, with **network segments** defining the isolation boundaries between Production, Non-Production, and Shared Services traffic. Production segments are explicitly configured to prevent lateral communication with Non-Production, enforcing the same isolation previously achieved through Transit Gateway route tables — but as a declarative policy rather than per-attachment configuration
- **Core Network Edges** — Cloud WAN gateway infrastructure deployed into each AWS Region in use, enabling consistent connectivity policies regardless of where workload accounts land
- **VPC Attachments** — workload account VPCs attach directly to Cloud WAN via attachment policies, with segment assignment driven by tags on the attachment. This means new accounts are automatically placed into the correct network segment as part of the account vending workflow, with no manual route table updates required
- **On-premises connectivity** — Direct Connect connections terminate via a Connect attachment into Cloud WAN, bringing on-premises traffic into the same policy-governed backbone as VPC traffic

Cloud WAN's network policy is version-controlled as code alongside the rest of the landing zone configuration, meaning segment rules, attachment policies, and routing behaviour are subject to the same review and audit process as any other infrastructure change.

If you'd like more depth on Cloud WAN architecture and implementation, I've covered it in detail in the [AWS Cloud WAN series]({{ site.baseurl }}/AWS-Cloud-WAN-Design-Part1/) on this blog.

![_config.yml]({{ site.baseurl }}/images/blog/AWS-Aviation-Landing-Zone/Diagram3.png)

### AWS Global Accelerator ###

Aviation operations are inherently distributed. Ground handling teams at remote airports, maintenance crews at outstations, and operational staff in regions with constrained internet infrastructure all need reliable, low-latency access to centrally hosted applications — and the public internet does not provide consistent performance across all of these locations.

**AWS Global Accelerator** addresses this by routing user traffic onto the AWS global backbone at the nearest AWS edge location, rather than traversing the public internet for the majority of the journey. As soon as a user's request enters the AWS network, it benefits from AWS's private, high-throughput fibre infrastructure and intelligent routing — avoiding the congestion, packet loss, and variable latency that characterise long-haul public internet paths.

For the aviation deployment this provides several concrete benefits:

- **Reduced latency for remote users** — operational staff at outstations or regions with poor internet connectivity see significantly more consistent application response times, because the unpredictable portion of the network path (public internet) is minimised to the last mile only
- **Automatic failover** — Global Accelerator continuously health-checks the application endpoints behind it and reroutes traffic within seconds if an endpoint becomes unhealthy, without requiring DNS TTL expiry. For operational applications where downtime directly impacts ground handling or passenger services, this is a meaningful improvement over DNS-based failover
- **Static anycast IP addresses** — applications are reachable via two static global IP addresses regardless of which region the underlying infrastructure is running in. This simplifies firewall allowlisting at partner organisations and airports, which often require fixed IPs to permit outbound traffic
- **DDoS protection** — Global Accelerator is integrated with **AWS Shield Standard** by default, providing always-on protection at the edge against volumetric network attacks without additional configuration

Global Accelerator sits in front of the Application Load Balancers in the Production workload accounts, with the accelerator itself managed from the Network Account alongside the Cloud WAN and DNS configuration. This keeps all external-facing network infrastructure under a single governance boundary.

### DNS with Amazon Route 53 ###

**Amazon Route 53 Resolver** provides centralised DNS across the multi-account environment. A centrally managed private hosted zone in the Shared Services Account is shared to workload accounts, enabling consistent internal DNS resolution. Route 53 Resolver rules forward on-premises domain queries to the corporate DNS servers, and inbound resolver endpoints allow on-premises systems to resolve AWS-hosted service endpoints.

---

## Security Architecture ##

### Centralised Logging ###

All accounts ship logs to the centralised **Log Archive Account**, which is protected from modification by even the management account. The logging pipeline captures:

- **AWS CloudTrail** — organisation-wide API activity trail, written to an S3 bucket in the Log Archive Account with Object Lock enabled to prevent tampering
- **VPC Flow Logs** — captured for all VPCs and delivered to the same S3 destination for network forensics
- **AWS Config** — configuration snapshots and change history from all enrolled accounts, aggregated into the Security Tooling Account

The S3 buckets in the Log Archive Account have server-side encryption enforced using **AWS KMS** with a customer-managed key, versioning enabled, and S3 Object Lock in Compliance mode to ensure logs cannot be deleted within the retention window.

### Threat Detection and Posture Management ###

**Amazon GuardDuty** is enabled at the organisation level and delegated to the Security Tooling Account, which receives findings from all member accounts. Findings feed into **AWS Security Hub**, which aggregates results from GuardDuty, AWS Config conformance pack evaluations, and Inspector findings into a unified security posture view.

**AWS Config conformance packs** aligned to standard frameworks are deployed across all accounts from the Security Tooling Account. Any detected drift — such as an unencrypted EBS volume, a security group with unrestricted ingress, or an IAM user with active access keys — is reported as a non-compliant finding and surfaced in Security Hub.

![_config.yml]({{ site.baseurl }}/images/blog/AWS-Aviation-Landing-Zone/Diagram4.png)

---

## Encryption and Key Management ##

**AWS KMS** with customer-managed keys (CMKs) is used for encryption across all services — S3, EBS, RDS, Secrets Manager, and CloudWatch Logs. Keys are created in the account that owns the data, with key policies enforcing that only appropriate roles and services can use them. The Log Archive Account's CMK is managed separately to ensure security administrators cannot decrypt audit logs using the same key material that application teams manage.

**AWS Secrets Manager** is used for all application credentials, database passwords, and API keys. Secret rotation is enabled where the target service supports it, and access to secrets is granted via IAM resource-based policies that follow the principle of least privilege.

---

## Automation and Operations ##

### Infrastructure as Code ###

The landing zone configuration is managed entirely through Infrastructure as Code using **OpenTofu**. Account provisioning is handled natively through **AWS Organisations** and **AWS Control Tower** — new accounts are created via the Control Tower Account Factory, with account enrolment triggering the automatic application of the governance baseline (CloudTrail, Config, guardrails) through Control Tower's built-in mechanisms. This keeps the provisioning path simple and auditable without introducing an additional orchestration layer.

All account-level customisations — additional Config rules, IAM roles, S3 bucket policies, Cloud WAN attachment tagging — are managed as OpenTofu modules applied through a CI/CD pipeline. The pipeline runs `tofu validate` and `tofu plan` on every pull request, with `tofu apply` gated on peer review and approval. This means infrastructure changes go through the same engineering workflow as application code: branching, review, and a traceable audit trail in source control.

### AI-Assisted Development ###

Across the landing zone build, AI coding tools were used to accelerate delivery and enforce consistency. Rather than writing boilerplate OpenTofu module structures, variable definitions, and output blocks from scratch, AI assistance generated the scaffolding that engineers then reviewed and refined — cutting the time from design decision to working code significantly.

The more meaningful benefit was code quality. AI review was used as a first pass on every module before peer review: catching missing variable validation, inconsistent naming conventions, outputs that didn't align with the module's interface contract, and security misconfigurations such as overly permissive IAM trust policies or missing encryption settings. Issues that would previously have surfaced during human review — or worse, post-deployment — were caught earlier in the workflow.

This approach treats AI as a force multiplier for the engineering team rather than a replacement for judgement. Engineers own the architecture decisions and review everything generated; AI handles the mechanical consistency checks and accelerates the path from intent to deployable code.

### Operational Monitoring ###

**Amazon CloudWatch** is configured with cross-account observability, allowing a central monitoring account to query metrics and logs from workload accounts without requiring direct console access. CloudWatch alarms are set on critical operational metrics — API error rates, Cloud WAN attachment drops, GuardDuty finding severity — with alerts routed to operations teams via SNS.

Metrics from CloudWatch are forwarded to **Amazon Managed Service for Prometheus (AMP)**, providing a scalable, long-retention metrics store that the engineering and operations teams can query using standard PromQL. This decouples the operational dashboarding layer from CloudWatch's native console, enabling richer querying and alerting across the multi-account environment without navigating between accounts individually.

**Amazon Managed Grafana (AMG)** sits in front of AMP as the single observability interface for the platform team. Dashboards are built in Grafana against the Prometheus data source, covering infrastructure health, network performance, security posture metrics, and workload-level indicators. IAM Identity Centre integration means Grafana access follows the same federated identity model as the rest of the platform — engineers see dashboards scoped to the accounts they have permission to access, and the security team has a dedicated view across all accounts without requiring separate credentials.

**AWS Lambda** is used for event-driven automation: auto-remediation of specific Config findings, automated snapshot scheduling for non-production databases, and account hygiene tasks such as disabling stale IAM users flagged by Access Analyzer.

The combination of centralised observability, automated alerting, and Lambda-driven remediation has a direct impact on **Mean Time to Resolution (MTTR)**. Issues that previously required an engineer to manually triage across multiple accounts — correlating CloudTrail events, checking Config history, and cross-referencing network logs — are surfaced automatically in Security Hub or CloudWatch with the relevant context already aggregated. For operational incidents, this has reduced the triage phase from hours to minutes. For security findings, automated Lambda remediation handles a significant proportion of common issues — misconfigured security groups, public S3 access, unencrypted volumes — without any manual intervention at all, freeing engineering time for higher-value work.

---

## What This Foundation Enables ##

With the landing zone in place, the aviation operator has a secure, auditable, and scalable cloud foundation from which to deploy and iterate on business workloads. The structured account model and network segmentation are prerequisites for the workloads that matter most in aviation:

- **Passenger disruption management systems** that need to integrate with airline reservation systems and airport APIs, with clear network boundaries between partners
- **Predictive maintenance platforms** that process aircraft sensor data at scale, requiring isolated compute environments and controlled data egress
- **Cargo tracking and supply chain integrations** with fine-grained access controls between carrier and handler systems
- **Ground operations tooling** where operational teams need scoped, role-based access and high availability across Availability Zones

Each of these use cases can be deployed into a pre-existing, governed account structure rather than building security and compliance controls from scratch for every workload.

### Speed to Market ###

One of the most tangible outcomes of the landing zone is the reduction in lead time to get new workloads running in a production-ready environment. Before the landing zone existed, standing up a new environment meant manually creating accounts, configuring IAM, building networking from scratch, and establishing logging and monitoring — a process measured in weeks. With the landing zone in place, a new account is provisioned through Control Tower with the full governance baseline applied automatically, Cloud WAN attachment tagging places it in the correct network segment, and the OpenTofu pipeline deploys account-level customisations in a single run. The result is a production-ready environment delivered in hours rather than weeks.

This speed compounds over time. Engineering teams onboarding new workloads no longer spend their first sprint building infrastructure plumbing — they start from a secure, compliant baseline and focus immediately on the application layer. For a business moving quickly, the ability to go from a decision to a deployed, governed environment in the same day is a meaningful competitive advantage.

### Reduction in Engineering Overhead ###

The automation and IaC approach has significantly reduced the ongoing engineering effort required to operate the platform. Routine tasks that previously required manual intervention — account hygiene, security finding remediation, baseline drift correction — are handled automatically. The centralised governance model means that a small platform team can maintain consistent security and compliance posture across every account in the organisation without having to touch each one individually.

AI-assisted development has compounded this further by reducing the time engineers spend on boilerplate and code review feedback cycles. Collectively, these automation gains translate to a material reduction in the engineering hours required per workload onboarding and per operational cycle — time that is reinvested into delivering business capability rather than maintaining infrastructure.

---

## Summary ##

A well-designed AWS Landing Zone is the most important infrastructure investment an aviation operator can make when adopting AWS at scale. The architecture described here — built on AWS Organisations, Control Tower, IAM Identity Centre, Cloud WAN, and centralised security tooling, managed through OpenTofu — provides the governance, network segmentation, and compliance foundation that aviation-grade workloads require.

The approach is available as a professional services engagement through the [AWS Marketplace](https://aws.amazon.com/marketplace/pp/prodview-s7bklekkdkw3w), tailored to the specific operational and regulatory context of the aviation industry.

If you are planning a cloud foundation for an aviation or similarly regulated environment and want to discuss the architecture or implementation approach, feel free to reach out.
