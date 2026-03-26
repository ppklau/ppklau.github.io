---
title: "From Commands to Intent: Why Network Engineering Is At An Inflection Point"
date: 2026-03-27
draft: false
tags: ["Network Automation", "Network Configuraiton As Code", "Intent Based Networking", "NetDevOps"]
summary: "Part 1 of 3 — The case for intent-based networking"
---
There is a pattern that repeats itself across every generation of infrastructure engineering. We start by doing things manually, at great cost and with great expertise. Then we automate the repetition. Then we raise the abstraction layer — so that humans describe *what* they want and machines figure out *how* to deliver it. Compute went through this cycle with virtualisation and cloud. Storage went through it with software-defined everything. Networking is going through it now.

The term for this shift is **intent-based networking** (IBN). It sounds like a vendor marketing term — and it has certainly been adopted as one — but the underlying idea is both older and more profound than any product announcement. It is a fundamentally different way of thinking about how networks are designed, operated and governed.

This article is the first in a three-part series. Here I want to make the case for *why* this matters, what it actually means and where it leads. The second article will show what this looks like in practice, using a realistic financial services example. The third will explore what becomes possible when verification and generation are automated — and where AI enters the picture.

---

## The problem with how we operate networks today

Ask a network engineer to describe their daily work and you will hear a familiar story. A change request arrives — a new VLAN, a firewall rule, a routing adjustment. The engineer interprets the request, works out what it means in terms of device configuration, logs into the relevant devices and types commands. They do this carefully, methodically and with significant institutional knowledge in their head. Then they document what they did, after the fact, in a change record that describes *what happened* rather than *why it was needed*.

This is not a criticism of how engineers work. It is a description of how the craft has evolved. CLI-driven configuration management is extraordinarily powerful and, in the hands of experienced engineers, remarkably effective. But it has structural weaknesses that become more expensive as networks grow in scale and complexity.

**The intent is lost in translation.** A business requirement — "trading systems must be isolated from corporate users" — passes through several human interpretation steps before it becomes a set of ACL entries on a leaf switch. Each step is an opportunity for drift between what the business actually needs and what the network actually does.

**State diverges from documentation.** Networks accumulate configuration over time. Rules are added, modified, never cleaned up. The running configuration of a production device often reflects decisions made years ago, by people who have since left, for reasons nobody can reconstruct. The documentation, if it exists, lags behind.

**Change is expensive and risky.** Because configuration is manual and institutional knowledge is tacit, every change carries risk. The blast radius of a misconfiguration is hard to predict. Testing is limited. Rollback is manual. The result is change windows, lengthy approval processes and a culture of "if it ain't broke, don't touch it" — which is how technical debt accumulates.

**Compliance is a retrospective activity.** Auditors ask whether the network enforces certain policies. Engineers produce evidence — ACL exports, topology diagrams — but the link between a regulatory requirement and the specific configuration that satisfies it exists only in someone's head.

These are not edge cases. They are the normal operating conditions of most enterprise networks.

---

## What intent-based networking actually means

Intent-based networking reorganises the relationship between human decision-making and machine configuration. Instead of engineers writing device configuration directly, they express *what the network should do* at a level of abstraction that maps to business and architectural decisions. The translation from intent to configuration is automated, verifiable and repeatable.

The word "intent" is doing important work here. An intent is not a high-level configuration template. It is a statement about desired behaviour, expressed at the level of the network's purpose rather than its implementation.

Consider the difference:

**Configuration-level thinking:**
```
interface Vlan100
  vrf TRADING
  ip access-group ACL_TRADING_IN in
```
<br>

**Intent-level thinking:**
```yaml
- id: INTENT-SEG-01
  title: Trading zone is isolated from all other zones
  satisfies: [REQ-SEC-01, REQ-SEC-02]
  implementation:
    vrfs: [TRADING, CORPORATE, DMZ]
    inter_vrf_routing: disabled
    firewall_exit: required_for_inter_zone
  test: "Assert no route leaking between VRFs without firewall exit"
```

The configuration snippet tells a device what to do. The intent statement tells the network *why* this decision was made, *what business requirement it satisfies* and *how to verify it is still true*. The configuration is an output — generated from the intent, not written by hand.

This distinction matters enormously. When intent is captured as structured, version-controlled data, something powerful becomes possible: the network can be *reasoned about* by both humans and machines. You can ask "does the current configuration still satisfy this business requirement?" and get a deterministic answer. You can propose a change and automatically verify that it does not violate any stated intent. You can generate the configuration for a new site by encoding the intent once and applying it everywhere.

---

## The workflow: from requirement to configuration

Intent-based networking is not a single tool or product. It is a workflow — a way of organising the relationship between different layers of decision-making. A well-structured IBN workflow has three distinct layers.

**Layer 1: Business requirements.** These are the statements that come from the organisation — from compliance, from the CISO, from the trading desk, from operations. They describe what the business needs the network to do, expressed in business language. "All trading systems must maintain sub-millisecond latency." "Internet-facing services must be hosted in a DMZ." "All configuration changes must be auditable." These requirements live in version control alongside the technical artefacts they generate and every downstream decision traces back to them.

**Layer 2: Design intents.** This is where architecture happens. Network architects translate business requirements into specific, testable network behaviours. An intent like "all leaf switches must be deployed as MLAG pairs, with active-active server bonding" is a design decision — one that satisfies the business requirement for high availability. Crucially, intents are machine-readable and testable. Each one has a verification statement: a description of exactly what must be true in the network for the intent to be considered satisfied.

**Layer 3: Source of Truth and configuration generation.** The Source of Truth (SoT) is a structured data model — typically YAML or a dedicated database — that captures the specific values (IP addresses, ASNs, VLANs, hostnames, peers) for every device in the network. Jinja2 templates or similar tooling transforms this data into vendor-specific configuration. Critically, the SoT is annotated with intent references: every block of data traces back to the design intent it implements and every intent traces back to a business requirement.

The result is an unbroken chain of traceability: *this ACL entry exists because of this design intent, which was created to satisfy this business requirement, which was approved by this stakeholder.* That chain has value far beyond compliance — it is the foundation of autonomous operation.

---

## Why now? The convergence that makes this possible

Intent-based networking as a concept has existed for years. What has changed is the maturity of the open-source tooling that makes it practical without a six-figure vendor contract.

**Network automation frameworks** like Ansible and Nornir are now widely understood and deployed. They provide the execution layer — the ability to render configuration from templates and push it to devices at scale.

**Network modelling tools** like Batfish allow you to validate network behaviour against a model of the network *before* any configuration is deployed. Batfish can answer questions like "does this proposed change create a routing black hole?" or "does any ACL contain a permit-any rule?" without touching a live device. This is the verification layer that makes automated change safe.

**GitLab and GitHub CI/CD pipelines** provide the governance layer — merge requests as change control, pipeline stages as automated gate checks, artefacts as the audit trail.

**Structured data formats** — YAML, JSON, YANG — are now the lingua franca of network configuration. The industry has largely accepted that human-readable structured data, version-controlled in Git, is a better source of truth than a device's running configuration.

None of these tools is new. What is new is that they are mature, well-documented and can be assembled into a coherent workflow without significant custom engineering. The activation energy required to adopt intent-based networking has dropped substantially.

---

## The destination: self-healing, self-provisioning networks

The practical benefits of intent-based networking — auditability, reproducibility, reduced change risk — are significant on their own. But they are the beginning, not the end. The real value emerges when you ask what becomes possible once intent is machine-readable and configuration is generated rather than written.

**Self-provisioning.** When a new site, a new VLAN, or a new security zone is required, the engineer no longer writes configuration. They declare the intent — the new site, its IP prefix, its role in the network — and the system generates configuration that is guaranteed to satisfy every stated design intent. The same patterns, the same security policies, the same management configuration, applied consistently and without the possibility of an engineer forgetting a step.

**Automated design verification.** Every proposed change to the Source of Truth triggers an automated pipeline that verifies the change against every stated intent before a single packet moves. Does the new device have MLAG configured? Are all ACLs traceable to a requirement? Does the proposed routing change break any reachability intent? These are not questions for a human reviewer — they are automated assertions, run in seconds.

**Self-healing.** When the network's actual state drifts from its intended state — a configuration that has been manually altered, a device that has lost its configuration — the system detects the drift and can correct it autonomously. The intent is the source of truth, not the running configuration. Deviation from intent is an event to be remediated, not accepted.

**AI-driven closed loops.** This is where the next generation of network operations lives. When intent is structured and machine-readable, it becomes the input to AI systems that can reason about the network at a level of abstraction that no human can sustain at scale. An AI system that understands the intent layer can propose configuration changes to improve performance, predict the blast radius of a failure, or generate the intent for a new business requirement from a natural language description. The engineer's role shifts from writing configuration to governing intent — from craftsperson to architect.

---

## The mindset shift that matters most

Intent-based networking is not primarily a technology choice. It is an organisational and architectural decision about how network decisions are made, recorded and enforced.

The most important shift is this: **configuration is an output, not an input.** Engineers do not write configuration for devices. They write intent — structured, version-controlled statements about what the network should do — and the system generates configuration from that intent. The configuration is a build artefact, like a compiled binary. You do not edit a compiled binary; you change the source and rebuild.

This reframing has consequences at every level of the organisation. Change management becomes a code review. Compliance becomes a test suite. Documentation becomes the data model itself. Onboarding becomes learning the intent schema rather than learning device-specific CLI syntax.

For organisations operating regulated infrastructure — financial services, healthcare, critical national infrastructure — the implications are particularly significant. The ability to demonstrate, at any point in time, that every aspect of the network configuration is traceable to an approved business requirement and that this traceability is enforced automatically, is a qualitatively different compliance posture than anything achievable with traditional methods.

---

## What comes next

The second article in this series walks through what this workflow looks like in practice, using ACME Investments — a fictional financial services firm — as a worked example. We will look at how business requirements are structured, how design intents are written and how a Source of Truth YAML file connects both layers to the configuration that runs on the devices.

The third article explores what becomes possible when verification and generation are automated: how a new branch office can be added to the network by running a single command and what automated design verification looks like in a CI/CD pipeline.

The destination is a network that provisions itself, verifies itself and heals itself. The path there starts with encoding intent.