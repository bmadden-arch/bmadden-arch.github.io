---
layout: post
title: CloudFront VPC Origins - A Better Way to Protect API Gateway
---

## Introduction ##

Welcome back to another post. If you've been following along, you may recall that back in November 2024 I published a post on [using Amazon CloudFront to protect Amazon API Gateway]({{ site.baseurl }}/Amazon-CloudFront-API-SEC-Part1/). In that post I set up a Regional API Gateway with a mock endpoint and placed an **Amazon CloudFront** distribution in front of it with geographic restrictions. I promised a Part 2 covering custom origin headers with an API Gateway Resource Policy, and a Part 3 adding a Lambda Authoriser for request-level validation.

Those posts never materialised — and for good reason. On 20 November 2024, the very same month Part 1 went live, AWS announced **CloudFront VPC Origins**. This new feature fundamentally changed the approach to protecting API Gateway behind CloudFront, making the custom header workaround I had planned for Part 2 largely unnecessary for new architectures.

Rather than writing a now-outdated walkthrough, this post covers what VPC Origins is, how it changes the game for API Gateway security, and why it is the better approach going forward.

The architecture diagrams in this post have been created using [Eraser.io](https://www.eraser.io/).

## A Quick Recap ##

In Part 1 we created a **Regional REST API** in Amazon API Gateway with a simple mock integration. We then set up a CloudFront distribution pointing to the API Gateway invoke URL as the origin, configured with HTTPS only, geographic restrictions limited to Ireland and the United Kingdom, and a price class covering North America and Europe.

The problem with that setup was straightforward — whilst CloudFront provided a layer of caching, DDoS protection (via **AWS Shield**), and geographic restriction, the underlying API Gateway endpoint was still publicly accessible on the internet. Anyone who discovered the invoke URL could bypass CloudFront entirely and hit the API directly.

The planned Part 2 would have addressed this by injecting a custom origin header in CloudFront (e.g., `X-Custom-Header: secure-access`) and adding an API Gateway Resource Policy that denied any request not carrying that header. Part 3 would have added a **Lambda Authoriser** to validate the `x-forwarded-for` header for an additional layer of request-level verification.

Even with all three layers in place, the API Gateway endpoint would still have been reachable from the public internet — requests without the correct header would be denied at the application level, but the endpoint itself would remain exposed. This is security through policy enforcement rather than genuine network isolation.

![_config.yml]({{ site.baseurl }}/images/blog/CloudFront-VPC-Origins/BlogImage1.png)

**Notes:**
- The Regional API Gateway invoke URL is publicly accessible on the internet
- Even with custom headers and resource policies, the endpoint is reachable
- Red dotted arrow from Threat Actor highlights the security gap

## What Are CloudFront VPC Origins? ##

**Amazon CloudFront VPC Origins**, announced on 20 November 2024, is a feature that allows CloudFront to connect directly to resources inside private VPC subnets. Prior to this, any origin behind CloudFront — whether an Application Load Balancer, EC2 instance, or API Gateway endpoint — needed to be publicly accessible on the internet, even if access was restricted through security groups, custom headers, or resource policies.

With VPC Origins, the origin resource never needs a public IP address or internet gateway. CloudFront establishes a secure, private connection to the origin through the AWS internal network. The supported origin types are:

- **Application Load Balancer (ALB)** in private subnets
- **Network Load Balancer (NLB)** in private subnets
- **EC2 instances** in private subnets

There is no additional cost for using VPC Origins — it is included with CloudFront at no extra charge.

**Benefits of CloudFront VPC Origins**

- Complete Network Isolation: Origins remain in private subnets with no public internet exposure
- Reduced Attack Surface: Eliminates the possibility of bypassing CloudFront to access the origin directly
- No Additional Cost: Included with CloudFront at no extra charge
- Simplified Security Model: No need for custom header workarounds or resource policies to prevent direct access
- Native VPC Integration: Works with existing VPC networking constructs including security groups and NACLs

## VPC Origins and API Gateway ##

Here is where it gets interesting. **Amazon API Gateway** itself is not a supported VPC Origin target — VPC Origins works with ALBs, NLBs, and EC2 instances. However, API Gateway supports a **Private** endpoint type that is only accessible through an interface VPC Endpoint. By combining a Private API Gateway with an internal ALB and a VPC Endpoint, you can create an architecture where the entire API is completely unreachable from the public internet.

The architecture works as follows:

1. **Amazon CloudFront** receives the client request and routes it through a configured VPC Origin
2. The **VPC Origin** connects to an internal **Application Load Balancer** sitting in private subnets within your VPC
3. The ALB forwards the request to a **Target Group** configured with the IP addresses of the VPC Endpoint's elastic network interfaces
4. The **VPC Endpoint** (for the `execute-api` service) provides private connectivity to API Gateway
5. The **Private API Gateway** processes the request and returns the response back through the same chain

A custom header — `x-apigw-api-id` — is added to the CloudFront origin configuration containing your API Gateway's ID. This tells the VPC Endpoint which API to route the request to.

> **Note:** The ALB must have a certificate issued by a trusted certificate authority (such as DigiCert, GlobalSign, or similar). Self-signed certificates are not supported for VPC Origins.

The critical difference from the original approach is that the Private API Gateway has no public endpoint whatsoever. There is no invoke URL exposed to the internet. The only path to the API is through CloudFront, through the VPC Origin, through the internal ALB, and through the VPC Endpoint — all of which stay entirely within the AWS private network.

![_config.yml]({{ site.baseurl }}/images/blog/CloudFront-VPC-Origins/BlogImage2.png)

**Notes:**
- Nothing inside the VPC is reachable from the public internet
- CloudFront is the only public-facing entry point
- The dotted line with ✕ shows the blocked access attempt

## Comparing the Approaches ##

### Original Approach ###

- Regional API Gateway with a **public** endpoint
- CloudFront distribution with custom origin header injection
- API Gateway Resource Policy denying requests without the header
- Lambda Authoriser for `x-forwarded-for` validation
- The API Gateway invoke URL is **still reachable** on the public internet — requests are denied at the application layer, but the endpoint remains exposed

### VPC Origins Approach ###

- Private API Gateway with **no public endpoint**
- CloudFront distribution with VPC Origin pointing to an internal ALB
- VPC Endpoint providing private connectivity to API Gateway
- The API is **physically unreachable** from the public internet
- Network-level isolation rather than application-level policy enforcement

The distinction matters. With the original approach, you are relying on headers, policies, and authorisers to reject unwanted traffic — the door is locked, but it is still visible. With VPC Origins, the door does not exist on the public internet. There is no endpoint to discover, no invoke URL to probe, and no attack surface to exploit.

## Considerations ##

VPC Origins is not without trade-offs, and it is worth being upfront about them:

- **Additional Infrastructure**: The architecture requires an ALB, VPC Endpoint, target groups, and certificate management — more moving parts than a simple Regional API Gateway
- **ALB Requirement**: You must use an ALB (not NLB) when the origin connection requires TLS, as VPC Origins does not currently support NLB with TLS passthrough
- **Same Account**: The VPC Origin must reside in the same AWS account as the CloudFront distribution
- **No Private Custom Domains**: You cannot use the private custom domain name that API Gateway provides with this approach — only the default endpoint is supported
- **Target Group IP Management**: The VPC Endpoint's elastic network interface IPs can change. You will need automation to keep the target group in sync
- **Certificate Management**: The ALB requires a certificate from a trusted certificate authority — this adds ongoing certificate lifecycle management

The original custom header approach still has its place. If you have an existing Regional API Gateway, a simpler use case, or need cross-account architectures, the Part 1 approach with a custom header and resource policy remains valid. VPC Origins is the better choice for greenfield deployments where genuine network isolation is a priority.

## Conclusion ##

AWS releasing CloudFront VPC Origins rendered the custom header approach I had planned for Part 2 largely obsolete for new architectures. Rather than bolting security onto a public endpoint through headers and policies, VPC Origins allows you to keep the entire API private by design — with CloudFront as the only public-facing entry point.

The trade-off is additional infrastructure complexity, but for organisations that require genuine network isolation — particularly in regulated industries or where the attack surface must be minimised — it is well worth it.

I may follow up with a hands-on post walking through the implementation of this architecture using **OpenTofu**, covering the VPC Endpoint, internal ALB, target group, and CloudFront VPC Origin configuration as code. Stay tuned.
