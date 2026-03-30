---
title: "Encoding Intent: A Practical Guide to Network-as-Code"
date: 2026-03-30
draft: false
tags: ["Network Automation", "Network Configuraiton As Code", "Intent Based Networking", "NetDevOps"]
summary: "Part 2 of 3 — From business requirements to structured configuration"
---
In the first article of this series, I made the case that intent-based networking represents a fundamental shift in how networks are designed and operated — moving from engineers writing configuration to engineers encoding intent and letting the system generate configuration from that. The promise is networks that are auditable, reproducible and ultimately capable of self-provisioning and self-healing.

But theory only takes you so far. In this article I want to show what this workflow actually looks like, end to end, using a realistic example. The organisation is ACME Investments — a fictional but credible financial services firm, subject to the usual regulatory pressures of MiFID II, FCA rules and PCI-DSS. Their infrastructure spans a primary London datacentre running an Arista EOS spine-leaf fabric and branch offices connected via dual WAN circuits.

The full chain we will walk through is:

**Business requirements → Design intents → Source of Truth YAML → Configuration templates → Device configs**

This article covers the first three layers. Everything from the Source of Truth downward — the Jinja2 templates, the Ansible playbooks, the rendered configurations — is the subject of a separate body of work that this series connects to.

Repository for this article:
https://github.com/ppklau/network_automation/tree/main/learning/intent_based_networking

---

## Layer 1: Business requirements as structured data

The starting point is not a spreadsheet, a Confluence page, or a Visio diagram. It is a YAML file — `requirements.yml` — committed to the same Git repository that holds every other network artefact. This is a deliberate choice: requirements are not background documentation, they are the root of the dependency tree. Everything else traces back to them.

For ACME, requirements fall into four categories: business continuity, security, network architecture and operations. Here is a representative sample:

```yaml
business_continuity:
  - id: REQ-BIZ-01
    statement: >
      Trading and order-management systems must maintain sub-millisecond
      latency between execution engines and market connectivity gateways.
    priority: critical
    driver: revenue
    kpi:
      latency_p99_us: 500

  - id: REQ-BIZ-02
    statement: >
      All production services must tolerate the failure of any single
      network device without loss of connectivity.
    priority: critical
    driver: availability
    kpi:
      uptime_percent: 99.99

security:
  - id: REQ-SEC-01
    statement: >
      Network traffic must be segmented into distinct security zones:
      Trading, Corporate and DMZ. No direct traffic between Trading
      and DMZ is permitted without inspection.
    priority: critical
    driver: regulatory   # FCA SYSC 8 / MiFID II Art 48

  - id: REQ-SEC-04
    statement: >
      ACL policies must be auditable and traceable to a named business
      requirement. No permit-any rules are permitted in production.
    priority: high
    driver: audit_compliance
```
<br>

Several things are worth noting about this format. Each requirement has an identifier (`REQ-BIZ-01`) that will be referenced throughout the downstream layers. The `driver` field distinguishes between requirements that exist because of regulatory obligation versus business decisions — important when someone asks why a particular design choice was made. The `kpi` field makes the requirement measurable.

The act of writing requirements in this structured format is itself valuable. It forces precision. "The network should be fast" becomes a requirement with a specific P99 latency threshold. "We need good security" becomes three distinct requirements with regulatory citations. Ambiguity is the enemy of automation and this format surfaces ambiguity early.

---

## Layer 2: Design intents — where architecture becomes code

The design intent layer is where network architects work. Each intent translates one or more business requirements into a specific, testable network behaviour. The key word is *testable*: every intent must have a verification statement — a description of what must be true in the network for the intent to be satisfied.

Here are the intents that flow from ACME's requirements:

```yaml
topology:
  - id: INTENT-TOPO-01
    title: Spine-leaf fabric at primary datacentre
    satisfies: [REQ-NET-01, REQ-NET-02, REQ-BIZ-01]
    description: >
      Deploy a two-tier spine-leaf fabric at lon-dc1. Two spine switches
      provide full-mesh connectivity to all leaf pairs. Any server-to-server
      path has exactly two hops (leaf → spine → leaf).
    implementation:
      fabric_type: spine_leaf
      spine_count: 2
      leaf_pairs: 2
      max_hops_east_west: 2
    test: "Assert path between any two leaf loopbacks is ≤ 2 hops"

  - id: INTENT-TOPO-02
    title: No single point of failure in fabric
    satisfies: [REQ-BIZ-02, REQ-NET-04]
    description: >
      Every leaf switch is deployed as an MLAG pair. Servers connect
      active-active across both peers. Loss of one peer causes zero
      server downtime.
    implementation:
      leaf_redundancy: MLAG
      server_bonding: LACP_active_active
    test: "Assert each leaf has an MLAG peer and peer-link is up"

segmentation:
  - id: INTENT-SEG-01
    title: Three VRFs map to three security zones
    satisfies: [REQ-SEC-01, REQ-SEC-02]
    description: >
      Three VRFs are instantiated in the VXLAN overlay: TRADING, CORPORATE,
      and DMZ. Inter-VRF routing is disabled by default. All cross-zone
      traffic must exit to a firewall for policy enforcement.
    implementation:
      vrfs: [TRADING, CORPORATE, DMZ]
      inter_vrf_routing: disabled
      firewall_exit: required_for_inter_zone
    test: "Assert no route leaking between VRFs without firewall exit"

  - id: INTENT-SEG-02
    title: ACLs enforce zone policy at leaf ingress
    satisfies: [REQ-SEC-02, REQ-SEC-04]
    implementation:
      acl_placement: leaf_svi_inbound
      default_action: deny
      permit_any: forbidden
      acl_entry_format:
        comment_format: "REQ-SEC-XX: <human description>"
    test: "Assert no ACL contains a permit-any rule; assert default deny exists"
```
<br>

The `satisfies` field is load-bearing. It creates an explicit, machine-readable dependency graph between intents and requirements. When an auditor asks "how does ACME's network satisfy MiFID II Article 48?", the answer is traceable through `REQ-SEC-01` → `INTENT-SEG-01` → the VRF configuration on every leaf switch. That traceability is not a retrospective documentation exercise — it is an intrinsic property of how the network was built.

The `test` field is equally important. It defines, in plain language, what automated verification looks like. In the CI/CD pipeline, these tests are implemented as Batfish assertions or Python checks — and they run on every proposed change before anything touches a live device.

---

## Layer 3: The Source of Truth

The Source of Truth (SoT) is the structured data model that captures all the device-specific values: IP addresses, ASNs, VLANs, hostnames, BGP peers, ACL entries. It is the input to the configuration generation pipeline. It is also the layer where the two higher layers are anchored — every significant block in the SoT carries an `intent:` annotation.

This is what a spine node looks like in ACME's `nodes.yml`:

```yaml
- hostname: spine01
  platform: arista_eos
  role: spine
  site: lon-dc1
  intent: [INTENT-TOPO-01, INTENT-RTG-01, INTENT-RTG-02]

  loopback:
    address: 10.0.254.1/32        # intent: INTENT-IP-01

  bgp:
    asn: 65001                    # intent: INTENT-RTG-01 (unique ASN per device)
    router_id: 10.0.254.1
    peers:
      - peer_ip: 10.0.255.0
        peer_asn: 65101
        description: "leaf01 underlay"
        address_families: [ipv4_unicast]
    evpn:
      enabled: true
      role: route_server           # intent: INTENT-RTG-02
      peers:
        - peer_ip: 10.0.254.11
          peer_asn: 65101
          description: "leaf01 EVPN"

  management:
    vrf: MGMT                      # intent: INTENT-MGMT-01
    address: 10.0.0.1/24
    ssh_vrf: MGMT
    syslog_servers:                # intent: INTENT-MGMT-02
      - 10.0.0.100
      - 10.0.0.101
    snmp:
      version: v3
      auth: SHA
      priv: AES128
```
<br>

And here is what a leaf node looks like — specifically its ACL configuration, where the traceability to business requirements is most visible:

```yaml
  acls:
    - name: ACL_TRADING_IN
      default_action: deny         # req: REQ-SEC-02
      entries:
        - seq: 10
          action: permit
          protocol: tcp
          src: 10.0.10.0/24
          dst: 10.0.10.0/24
          dst_port: any
          comment: "REQ-SEC-01: intra-trading east-west"
        - seq: 20
          action: permit
          protocol: tcp
          src: 10.0.10.0/24
          dst: any
          dst_port: 443
          comment: "REQ-BIZ-01: market data HTTPS feeds"
        - seq: 9999
          action: deny
          protocol: ip
          src: any
          dst: any
          comment: "REQ-SEC-02: explicit deny-all"
```
<br>

Every ACL entry carries a `comment` field containing a requirement ID and a human-readable description. This is not optional documentation — it is mandated by `INTENT-SEG-02`, which in turn satisfies `REQ-SEC-04` (ACLs must be traceable to a named requirement). When the Jinja2 template renders this into an Arista EOS configuration, the comment appears verbatim in the device config. The requirement lives in the device — not just in a document.

---

## The traceability chain in full

It is worth pausing to appreciate what this data model has achieved. We now have a complete, unbroken chain from business requirement to device configuration:

**REQ-SEC-01** (traffic must be zone-segmented, FCA/MiFID II)
→ **INTENT-SEG-01** (three VRFs, inter-VRF routing disabled)
→ **nodes.yml** (`vrfs: [TRADING, CORPORATE, DMZ]` on each leaf)
→ **Jinja2 template** (renders `vrf definition TRADING` in EOS config)
→ **Device configuration** (running on leaf01, leaf02, border-leaf01, border-leaf02)

Every link in this chain is machine-readable. Every link can be queried. Every link can be tested. This is qualitatively different from a network where the relationship between a regulatory requirement and a running configuration exists only in an engineer's memory.

For a financial services firm subject to regulatory examination, this matters enormously. But it matters equally for any organisation that wants to operate its network with confidence — that changes are safe, that policies are enforced, that documentation reflects reality.

---

## From SoT to configuration: the generation pipeline

The SoT does not generate configuration by itself. It is the input to an Ansible playbook that drives Jinja2 templates — one template per platform and role. The template for an Arista EOS leaf node takes the structured YAML above and renders it into the specific CLI syntax that EOS expects.

The pipeline looks like this:

{{< mermaid >}}
graph TD
    %% Define Nodes
    Start(["nodes.yml + inventory.yml"])
    Storage["Config diff +<br>artefact storage"]
    Gate["MR approval gate"]
    Deployment["Napalm deployment"]

    subgraph Playbook
      Templates["Jinja2<br>templates"] --> Configs["Rendered<br>EOS / IOS Configs"]
    end

    subgraph Validation
      Batfish["Batfish<br>validation"] --> Report["JUnit XML report<br>to GitLab CI"]
    end

    %% Define Relationships
    Start --> Playbook
    Playbook --> Validation
    Validation --> Storage
    Storage --> Gate
    Gate --> Deployment

    %% Styling
    style Gate fill:#f96,stroke:#333,stroke-width:2px
    style Deployment fill:#9f9,stroke:#333,stroke-width:2px
{{< /mermaid >}}
<br>
The rendered configurations are build artefacts. Engineers do not edit them. If a configuration needs to change, the change is made in the SoT — and the pipeline generates a new configuration. This is the same discipline that software engineering adopted with compiled code decades ago: you change the source, not the binary.

Batfish sits in the middle of this pipeline as the validation engine. Before any configuration reaches a device, Batfish models the entire network — all devices, all routing protocols, all ACLs — and runs assertions against that model. Does every BGP session have a peer? Are there any routing loops? Does any ACL contain a permit-any? These assertions are the design intents, expressed as code.

---

## What this looks like for the branch office

ACME's London branch office runs a simpler topology — Cisco IOS, OSPF Area 0, a collapsed two-tier design. The SoT reflects this with a different structure, but the same intent annotations:

```yaml
- hostname: lon-branch-rtr01
  platform: cisco_ios
  role: wan_router
  site: lon-branch1
  intent: [INTENT-RTG-03, INTENT-SEG-01, INTENT-MGMT-01]

  ospf:
    process_id: 1
    router_id: 10.1.254.1
    area: 0                        # intent: INTENT-RTG-03
    default_route: inject

  management:
    address: 10.1.0.1/24
    ssh_source_interface: Loopback0   # intent: INTENT-MGMT-01 (OOB)
    syslog_servers:
      - 10.0.0.100
      - 10.0.0.101
    snmp:
      version: v3
      auth: SHA
      priv: AES128
```
<br>

The platform is different. The routing protocol is different. But the intent annotations are the same and the management policies — dual syslog, SNMPv3, OOB access — are enforced identically. This is one of the most significant operational benefits of the approach: platform heterogeneity stops being a source of inconsistency. The intent layer is vendor-agnostic. The template layer handles the translation.

---

## The repository structure

Everything discussed in this article lives in a single Git repository. The structure is deliberately simple:

```
network-as-code/
├── requirements.yml          # Layer 1: business requirements
├── design_intents.yml        # Layer 2: design intents
├── inventory.yml             # Site and device inventory
├── nodes.yml                 # Layer 3: source of truth
├── templates/
│   ├── arista_eos/
│   │   ├── spine.j2
│   │   └── leaf.j2
│   └── cisco_ios/
│       ├── wan_router.j2
│       └── access_switch.j2
├── playbooks/
│   └── generate_configs.yml
├── tests/
│   └── batfish_validate.py
└── .gitlab-ci.yml
```
<br>
Every file in this repository is version-controlled. Every change goes through a merge request. The pipeline runs automatically on every MR. The rendered configurations are stored as pipeline artefacts. The audit trail is the Git history.

This is network infrastructure managed with the same discipline as application software. The tools are the same. The workflow is the same. The cultural expectations — code review, automated testing, CI/CD — are the same.

---

## What this enables that was not possible before

By the end of this workflow, ACME has something that most network teams do not: a network that can be *reasoned about* completely from the data model. You can ask "which devices implement INTENT-SEG-01?" and get an answer from the YAML. You can ask "does any device have a management configuration that does not reference INTENT-MGMT-01?" and catch a gap before it becomes a compliance finding. You can ask "what would change if we added a new security zone?" and model the answer before writing a single line of configuration.

The third article in this series takes this further. We look at what happens when verification is automated across every intent — and how a new branch office can be added to the network by running a single command, with the system guaranteeing that the new site satisfies every design intent automatically.