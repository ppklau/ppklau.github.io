---
title: "Network Configuration As Code : Making Networking Scale"
date: 2026-03-21
draft: false
tags: ["Network Automation", "Network Configuraiton As Code", "Hands-On Learning", "NetDevOps"]
summary: "A simple introduction to network configuration as code."
---
> TL;DR
Stop configuring devices manually. Define your network once, generate configs automatically and validate before deployment.

## Introduction

Managing network configurations has traditionally been a manual, device-by-device process. Engineers connect to each device, apply configuration changes, verify them, and repeat.

This works — until it doesn’t.

As networks grow, this approach becomes slow, inconsistent and difficult to scale.

In this article, we’ll walk through a very simple demonstration of treating network configuration as code. The goal is not to build a production-ready system, but to illustrate the core principles and why they matter.

Repository for this demo :
https://github.com/ppklau/network_automation/tree/main/learning/network_as_code

---

## The Problem We’re Trying to Solve

Traditional workflows introduce several challenges:

### Manual, Device-by-Device Configuration

Engineers must connect to each device individually (SSH, console, etc.), apply changes and verify them.

- Time-consuming
- Repetitive
- Difficult to track

### Configuration Drift

Over time, devices drift away from their intended state:

- Emergency fixes applied inconsistently
- Changes not documented
- Devices no longer match design

### Human Error

Manual work inevitably leads to:

- Typos
- Missing commands
- Inconsistent configurations across devices

### Does Not Scale

What works for:

- 5 devices → manageable
- 50 devices → painful
- 500 devices → unsustainable

![Traditional Network Configuration Workflow](nac_traditional.png)

---

## A Better Approach: Network Configuration as Code

Instead of configuring devices directly, we define configurations in code and generate device-specific configurations automatically.

This approach is built on two simple ideas.

### 1. Single Source of Truth

All network data lives in one place.

This could be:

- YAML
- JSON
- A database

It defines things like:

- Hostnames
- IP addresses
- VLANs
- Interfaces

> Key Idea :
> The source of truth is the only place you edit network intent.

### 2. Templates for Configuration Generation

We use templates (e.g., Jinja2) to turn data into device configurations.

This gives us:

- Consistency across devices
- Repeatability
- Easier updates (change data → regenerate configs)

![Network Configuration As Code Workflow](nac_network_as_code.png)

---

## A Simple Demo

This repository (https://github.com/ppklau/network_automation/tree/main/learning/network_as_code) demonstrates the basic workflow:

1. Define network data (source of truth)
1. Use templates to generate configurations
1. Output device configs automatically

> Note :
> This is intentionally minimal. The goal is to illustrate the concept — not build a full production system.
> The source of truth is just a YAML file to keep things simple and understandable, more scalable solutions need to be deployed for anything beyond lab or development.

---

### Repository Structure

```
network_as_code/
├── ansible.cfg                   # Ansible runtime configuration
├── generate_configs.yml          # Master playbook
├── inventory/
│   ├── inventory.yml             # Ansible host groups & connection vars
│   └── nodes.yml                 # Source of Truth – devices, IPs, VLANs, VRFs
├── templates/
│   ├── arista_eos.j2             # Arista EOS template (VXLAN/MLAG/eBGP-EVPN)
│   └── cisco_ios.j2              # Cisco IOS template (OSPF/ACLs/802.1Q)
├── build/                        # Generated configs (git-ignored in prod)
│   ├── datacenter/
│   │   ├── spine/
│   │   ├── leaf/
│   │   └── border/
│   └── branch/
│       └── london/
└── README.md
```

---

### Sample Configuration Template

![Cisco Configuration Template](nac_cisco_template.png)

---

### Sample Generated Configuration

![Cisco Generated Configuration](nac_cisco_output.png)

---

## Built-in Benefits: Linting and Testing

Once configuration is treated as code, we unlock powerful capabilities.

### Linting

We can automatically check:

- Syntax correctness
- Style consistency
- Policy compliance

Before anything touches a real device.

### Automated Testing (e.g., Batfish)

We can validate the network before deployment:

- Reachability checks
- Routing validation
- Misconfiguration detection

> Shift Left :
> Catch issues early — before they become outages.

---

## Applying Configuration Safely

Generating configs is only part of the story — the next step is applying them.

Modern network devices increasingly support a “replace” configuration model.

### Replace vs Traditional Approach

Instead of merging line-by-line:

- The device calculates an internal diff
- Only required changes are applied
- Unnecessary changes are avoided

### Why This Matters

- Reduces service disruption
- Avoids unintended side effects
- Makes deployments predictable

> Key Insight :
> This allows us to treat network devices more like modern services:
> - Apply a new configuration
> - Let the system calculate the difference
> - Update safely and consistently
> - no configuration drift

![Replace vs Merge](nac_replace_vs_merge.png)

---

## Where This Leads

This simple demo is just the beginning.

Next steps typically include:

- A Single Source of Truth that scales
- Adding validation pipelines (lint + testing)
- Integrating CI/CD workflows
- Automating deployment
- Implementing rollback strategies

---

## Conclusion

Network configuration as code directly addresses the core challenges of traditional network management:

- Eliminates repetitive manual work
- Reduces configuration drift
- Minimizes human error
- Scales with your infrastructure

By combining:

- A single source of truth
- Templates for generation
- Automated validation

we bring modern software practices into network engineering.