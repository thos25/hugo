---
title: "Installing Kubernetes: Picking MicroK8s for the Homelab"
description: "Why I chose MicroK8s over k3s or kubeadm, the three-node cluster I built, and how Ansible only showed up after the cluster already had workloads running on it."
date: 2026-07-28T09:00:00-06:00
image: 
math: 
license: 
hidden: false
comments: true
categories:
 - Homelab
 - Kubernetes
draft: false
tags:
 - Kubernetes
 - Series
 - Part2
---

## Recap

In [part one]({{< ref "kubernetes-intro" >}}) I talked about why I was moving off a single Docker Compose box and onto Kubernetes. This post covers the actual install: what I picked, what I built it on, and the (very small) role Ansible played.

## Picking a distribution

I didn't want to hand-roll `kubeadm` for my first real cluster, and I didn't want to fight with Talos's immutable, API-only model on day one either. I landed on **MicroK8s**. It's a single-snap install, it ships HA out of the box once you have three or more nodes, and the addon ecosystem (`microk8s enable <thing>`) meant I could turn on pieces of the platform — DNS, storage, observability — without writing them myself while I was still learning what each one actually did.

That trade-off is worth naming honestly: addons are convenient, but they're also opaque. A chunk of what's running in my cluster today (Longhorn, the observability stack) was never `kubectl apply`'d from a manifest I own — it just appeared because I typed `microk8s enable observability` one night. You'll see that gap show up again in the [storage]({{< ref "kubernetes-storage" >}}) and [GitOps]({{< ref "kubernetes-gitops" >}}) posts: things installed by an addon don't have a home in git, so my repo only holds the *edges* of those systems (an Ingress here, a ClusterIssuer there) rather than the whole install.

## The hardware

Three nodes, no dedicated control-plane/worker split:

```
[k8s_nodes]
k8-node1.thos.local
k8-node2.thos.local
k8-node3.thos.local
```

MicroK8s's HA mode means all three are control-plane and worker at once — as long as two of three are up, the cluster keeps a quorum and keeps scheduling. That was the whole point: no more "the one server is down and everything is down."

## Ansible showed up late, and only for two things

If you're picturing an Ansible playbook that bootstraps MicroK8s from bare metal — that's not what happened. The cluster was already running real workloads before Ansible entered the picture at all. When it did show up, it did exactly two things:

**`k8_setup.yml`** installs host prerequisites — and it's shorter than you'd expect:

```yaml
- hosts: k8s_nodes
  become: yes
  tasks:
    - name: Install open-iscsi and nfs-common
      apt:
        name:
          - open-iscsi
          - nfs-common
        state: present
```

Those two packages exist for exactly one reason: they're what Longhorn and NFS-backed volumes need on the host to attach storage. This playbook is a storage prerequisite, not a cluster bootstrap.

**`updates.yml`** is even simpler — `apt update && apt upgrade` across all three nodes, added about a month later once I got tired of patching each box by hand over SSH.

Everything else — MicroK8s itself, the addons, Helm installs — was done imperatively, one node/one terminal at a time. That's a deliberate callout for anyone following along: my "automation" story starts small and grows later, it didn't arrive fully formed.

## Prove it before you build on it

The pattern that shows up over and over in my early commits: stand up a piece of infrastructure, then immediately deploy something disposable to prove it actually works before trusting real workloads to it. The first thing I put behind Traefik wasn't a real app — it was three `hashicorp/http-echo` containers, just to confirm host-based routing worked. Cert-manager got the same treatment: an `nginx` pod at `test.joeyaxtell.com` existed for exactly one reason, to watch a real certificate get issued before I pointed anything real at the ClusterIssuer.

It's a small habit but it saved me a lot of debugging-two-things-at-once later. More on both of those in the [configuration]({{< ref "kubernetes-configuration" >}}) and [networking]({{< ref "kubernetes-networking" >}}) posts.

## Up next

With three nodes joined and quorum established, the next step was making the cluster trustworthy enough to run things on — TLS, secrets, and the conventions I settled on for every workload going forward.
