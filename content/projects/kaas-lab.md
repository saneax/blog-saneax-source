---
title: "Kubernetes-as-a-Service (KaaS) Lab"
date: 2025-10-30
description: "A virtualized environment for testing full KaaS lifecycle automation."
summary: "Hands-on exploration of cluster provisioning, network isolation, and tenant automation in a lab-scale Kubernetes setup."
draft: false
tags: ["kubernetes", "metal3", "networking", "automation"]
cover:
  image: "/images/covers/kaas-lab.webp"
  alt: "KaaS lab topology"
  caption: "Nested virtualization setup for KaaS automation testing."
---

This article walks through a **KaaS (Kubernetes-as-a-Service)** lab designed to emulate a production data center.  
It uses **nested virtualization**, **Metal3**, and **OVN** networking to automate cluster lifecycle management.

### Highlights
- Automated provisioning via Cluster API and Ansible  
- Load balancing with MetalLB  
- Tenant segmentation using OVN networks  
- Infrastructure definitions in Terraform and YAML  

**Goal:** Develop a modular, repeatable framework for self-service Kubernetes environments.

{{< comments >}}

