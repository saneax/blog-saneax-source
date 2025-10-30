---
title: "Multi-Region OpenStack with AI Workload Support"
date: 2025-10-30
description: "A lab setup for deploying multi-region OpenStack clusters to support Kubernetes and AI inference workloads."
summary: "End-to-end exploration of deploying and federating OpenStack clusters for multi-tenant compute and inference use cases."
draft: false
tags: ["openstack", "ai", "kubernetes", "infrastructure"]
cover:
  image: "/images/covers/multi-region-openstack.webp"
  alt: "Multi-region OpenStack architecture"
  caption: "Federating OpenStack regions for AI workloads."
---

This study documents how to design and deploy a **multi-region OpenStack environment** capable of hosting Kubernetes clusters and serving **AI inference workloads** efficiently on CPUs.

### Key Learnings
- Federation of OpenStack regions via Keystone + Magnum  
- CPU-only inference pipelines with containerized models  
- Network isolation using OVN and VLAN overlays  
- Storage integration using Ceph and object gateways  

**Outcome:** An end-to-end reproducible environment for AI infrastructure testing using open-source tools.

{{< comments >}}
