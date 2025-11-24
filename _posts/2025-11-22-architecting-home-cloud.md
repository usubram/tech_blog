---
layout: post
title: "The Architect’s Dilemma: Why I Replaced My Home Lab with a Private Cloud"
date: 2025-11-22
categories: [Architecture, DevOps]
tags: [TrueNAS, Proxmox, System Design]
description: "A hands-on architect’s journey building a private cloud at home, with lessons on zero trust, storage, high availability, and disaster recovery."
image: "/assets/img/architecture.svg"
og:
  title: "The Architect’s Dilemma: Why I Replaced My Home Lab with a Private Cloud"
  description: "A hands-on architect’s journey building a private cloud at home, with lessons on zero trust, storage, high availability, and disaster recovery."
  type: "article"
  image: "/assets/img/architecture.svg"
---
As a Distinguished Architect, I spend the majority of my professional life designing large-scale distributed systems for the retail sector. We talk about platforms that handle unpredictable load, require five-nines availability, and operate with strict cost discipline. The work is often abstract - diagrams, reference architectures, review boards, and design debates.

But here’s the uncomfortable truth: Architects become less effective when they stop being hands-on.
Real systems behave differently under load, during failures, or when constraints collide. The best way to stay sharp is to intentionally expose yourself to those constraints - not in theory, but on real hardware, real networks, and real services.
To bridge this gap, I rebuilt my home infrastructure from the ground up. I stopped treating it as a "home lab" and started treating it as a production environment. I built a fully segmented, observable, distributed private cloud, applying the same patterns I insist on in enterprise environments: decoupling, failure isolation, observability, and zero trust.


> **Personal Note:** This is a project I have been slowly working on and evolving over the past two years. Every component, decision, and lesson reflects real hands-on experience, not just theory.

<nav class="toc" style="background:rgba(44,83,100,0.13);border-radius:10px;padding:1.2em 1.5em 1em 1.5em;margin-bottom:2.2em;box-shadow:0 2px 8px rgba(44,83,100,0.10);">
  <strong style="color:#66d9ef;font-size:1.1em;">Table of Contents</strong>
  <ul style="margin-top:0.7em;margin-bottom:0.2em;">
    <li><a href="#the-architecture-at-a-glance">The Architecture at a Glance</a></li>
    <li><a href="#1-the-network-enforcing-zero-trust-at-the-edge">1. The Network: Enforcing Zero Trust at the Edge</a></li>
    <li><a href="#2-the-data--ai-node-storage-as-a-failure-domain">2. The Data & AI Node: Storage as a Failure Domain</a></li>
    <li><a href="#3-the-operational-node-high-availability-through-decoupling">3. The Operational Node: High Availability Through Decoupling</a></li>
    <li><a href="#4-disaster-recovery-hybrid-cloud--finops">4. Disaster Recovery: Hybrid Cloud & FinOps</a></li>
    <li><a href="#conclusion-the-hands-on-imperative">Conclusion: The Hands-On Imperative</a></li>
  </ul>
</nav>

Here is the architectural breakdown of that system - and why I believe every senior engineer should build something similar.

<h2 id="the-architecture-at-a-glance">The Architecture at a Glance</h2>
<div style="text-align:center; margin: 0 0 1.25rem 0;background-color:#2F4F4F;border-radius:8px; box-shadow: 0 6px 18px rgba(0,0,0,0.6); padding:12px">
  <img src="/assets/img/architecture.svg" alt="Home Cloud Architecture" style="max-width:100%; height:auto;" />
</div>



<h2 id="1-the-network-enforcing-zero-trust-at-the-edge">1. The Network: Enforcing Zero Trust at the Edge</h2>
Security architecture is defined by boundaries, not just firewalls. I moved away from consumer mesh routers to a software-defined network approach using OPNsense.

**The Gateway:** [OPNsense](https://opnsense.org/) handles all routing, providing me with enterprise-grade deep packet inspection and granular control over traffic flow.

**Segmentation:** Using [Netgear](https://www.netgear.com/) managed switches and Access Points, I enforce strict VLAN segmentation. IoT devices (which are notoriously insecure) are isolated on their own tag, completely separated from my management network and critical data.

**Secure Ingress:** Instead of exposing random ports, I utilize [HAProxy](https://www.haproxy.org/) for controlled ingress with SSL termination. For administrative access, I run a self-hosted VPN, ensuring that my management surface is never exposed to the public internet.

**The Architectural Lesson:** When you live inside a Zero Trust network of your own making, you develop an intuition for where trust boundaries break and how lateral movement happens. This intuition transfers directly to production design reviews.

<h2 id="2-the-data--ai-node-storage-as-a-failure-domain">2. The Data & AI Node: Storage as a Failure Domain</h2>
This is the backbone of the system: a custom workstation build centered on the [ASUS Pro WS W680M-ACE SE](https://www.asus.com/motherboards-components/motherboards/workstation/pro-w680m-ace-se/). I chose this board specifically for its [ECC memory](https://en.wikipedia.org/wiki/ECC_memory) support and BMC capabilities, allowing for true headless server management.

**The OS:** [TrueNAS Scale](https://www.truenas.com/).

**The Storage Layer:** I provisioned a 40TB raw capacity [ZFS](https://openzfs.org/) pool (4x 10TB disks). ZFS provides end-to-end data integrity, copy-on-write protection against ransomware, and self-healing against bit-rot.

**The Compute Layer:** TrueNAS Scale isn't just a NAS; it is a hyper-converged infrastructure. I utilize its container ecosystem to run data-heavy workloads directly next to the storage:

*   **[Immich](https://immich.app/):** A self-hosted photo and video management platform. Running this on the NAS eliminates network latency for high-bandwidth asset retrieval.
*   **Private AI ([Llama](https://www.llama.com/llama-downloads)):** I host local Large Language Models (LLMs) on this node. This allows me to experiment with AI inference while keeping all data completely private and sovereign.

**The Architectural Lesson:** ZFS teaches you humility. It makes failure explicit. Furthermore, orchestrating AI workloads locally forces you to understand memory pressure and storage throughput in ways that managed cloud providers often hide from you.

<h2 id="3-the-operational-node-high-availability-through-decoupling">3. The Operational Node: High Availability Through Decoupling</h2>
To ensure resilience, I decoupled my "Always-On" services from my storage. I offload these to a separate, power-efficient node ([Beelink Mini PC](https://www.bee-link.com/)) running the [Proxmox VE](https://www.proxmox.com/proxmox-ve) hypervisor.

**Service Reliability:** This node hosts [Home Assistant](https://www.home-assistant.io/) (automation) and [Frigate](https://frigate.video/) (NVR).

**Why Split the Nodes?** By running Frigate and Home Assistant here, I ensure that my security cameras and smart home automation remain online even if I take the main TrueNAS server down for maintenance. This is a classic "failure domain separation" pattern.

**Observability:** I run a full [Prometheus](https://prometheus.io/) + [Grafana](https://grafana.com/) stack on this node, scraping metrics from the OPNsense router, the TrueNAS box, and the Proxmox node itself to visualize the health of the entire distributed system in real-time.

**The Architectural Lesson:** Running Proxmox and TrueNAS as separate nodes teaches the value of SLA design. You learn quickly that storage maintenance shouldn't cause an outage in operational security (cameras).

<h2 id="4-disaster-recovery-hybrid-cloud--finops">4. Disaster Recovery: Hybrid Cloud & FinOps</h2>
Backups are not optional, and RAID is not a backup. To achieve true disaster resilience, I implemented a hybrid cloud strategy using [AWS S3](https://aws.amazon.com/s3/).

**The Strategy:** I mirror critical encrypted datasets to S3 Standard for immediate disaster recovery.

**Cost Control (FinOps):** I configured aggressive AWS Lifecycle Policies to automatically transition aging backups into Glacier Deep Archive. This allows me to retain terabytes of historical data for pennies on the dollar, while keeping recent backups instantly accessible.

**The Architectural Lesson:** Implementing lifecycle policies yourself forces you to confront the real FinOps trade-offs that enterprise organizations struggle with: cold data growth, retrieval penalties, and the true cost of redundancy.

<h2 id="conclusion-the-hands-on-imperative">Conclusion: The Hands-On Imperative</h2>
There is a vast difference between using the cloud and building the cloud.

When you have to manually configure the VLAN tags on a switch, tune the HAProxy ACLs, or troubleshoot a ZFS scrub at 2 a.m., you gain an instinct about distributed systems that cannot be learned from a whiteboard.

My advice to fellow Architects and Engineers is simple: Build your own cloud. Break it. Fix it. Observe it. It is the best R&D you will ever do.
