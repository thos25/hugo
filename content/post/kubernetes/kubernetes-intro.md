---
title: "Homelab Kubernetes Adventure"
description: "My journey into Kubernetes, and container orchestration"
date: 2026-01-07T22:09:17-06:00
image: https://www.ovhcloud.com/sites/default/files/styles/text_media_horizontal/public/2021-04/K8S-logo.png
math: 
license: 
hidden: false
comments: true
categories:
 - Homleab
 - Kubernetes
draft: false
tags:
 - Kubernetes
 - Series
 - Part1
---
# Homelab

## Intro

This is the first of a multi-post series detailing my adventure in Kubernetes.  I will go over what hardware i selected, what Kubernetes deployment method i chose, my first deployments, converting my docker compose manifests into Kubernetes manifests, storage, backups, GitOps with ArgoCD, and much more.  

This is my first time diving into Kubernetes, and the learning curve was high.  But i've learned a lot and i hope that by documenting what i went through, it'll help you on your journey to better understanding Kubernetes.  

# Problem Statement

![Modern Problems](https://en.meming.world/images/en/thumb/4/4a/Modern_Problems_Require_Modern_Solutions.jpg/300px-Modern_Problems_Require_Modern_Solutions.jpg)

My homelab situation was not great.  Here was the state of my environment, and why it needed to change:

- Outdated single-server hardware (limited resources, aging)
- Single point of failure — no redundancy or HA
- Services scattered in Docker Compose stacks, no orchestration
- No consistent persistent storage or volume strategy
- No automated backups or recovery plan
- Manual deployments, manual updating, leading to drift and falling behind
- No centralized monitoring, logging, or alerting
- Network complexity exposing services locally
- Limited capacity for scaling or resource isolation


This is essentially the topology of my homelab before Kubernetes:
![Sad Homelab](./homelab_before.png)

As you can tell, single server homelabs have their limitations.  When that single server goes down, everything goes down with it.  My services were running in Docker Compose stacks, which worked fine for a while, but as I added more services, managing and updating them became a nightmare.  There was no orchestration, so if a container crashed, it stayed down until I manually intervened.

I did, luckily, have my manifests in source control, but that's about where the `pro's` end.

# Solution

I improved my homelab by deploying Kubernetes: I purchased multiple hosts and built a cluster while learning core Kubernetes concepts and operations. The goal was to run my containerized apps on a resilient platform that provides orchestration, automated failover, rolling updates, scaling, and centralized management.

This is what the intended solution should look like:
![Happy Homelab](./homelab_after.png)

This solution addresses the problems I faced:
- **Redundancy and High Availability**: Multiple nodes ensure services remain available even if one node fails.
- **Orchestration**: Kubernetes manages container deployment, scaling, and recovery automatically.
- **Consistent Storage**: Implementing persistent volumes for reliable data storage.
- **Automated Backups**: Scheduled backups to protect against data loss.
- **GitOps**: Using ArgoCD for declarative, version-controlled deployments.
- **Centralized Monitoring**: Tools like Prometheus and Grafana for insights into system health

# More to come

I am going to break up this series into multiple posts so that it's easy to follow, so stay tuned for more!