---
title: "Running Workloads: From Compose Files to Kubernetes Manifests"
description: "Converting Docker Compose stacks into Deployment/Service/Ingress manifests, the conventions I settled on, and a couple of experiments that didn't make it."
date: 2026-08-31T09:00:00-06:00
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
 - Part6
---

## Recap

Certificates, secrets, networking, and storage are all in place after the last four posts. This one is about the actual workloads — turning what used to be Docker Compose stacks into real Kubernetes manifests, and the conventions that came out of doing that a dozen times over.

## The structural pattern

Nothing fancy here: one folder per application, and inside it a single manifest file with a Deployment, Service, and Ingress concatenated together with `---`. No Helm charts for my own apps, no Kustomize overlays — just plain YAML, applied with `kubectl apply -f`. It's not the most sophisticated pattern, but it maps almost one-to-one onto how I used to think about a Compose service, which made the conversion a lot less intimidating than I expected going in.

A few conventions repeat across nearly every workload I run, mostly inherited straight from the [linuxserver.io](https://www.linuxserver.io/) image conventions I was already used to from Compose:

```yaml
env:
  - name: PUID
    value: "3000"
  - name: PGID
    value: "3000"
  - name: TZ
    value: "America/Chicago"
```

Consistent UID/GID across every app made the NFS permissions story from the [storage post]({{< ref "kubernetes-storage" >}}) dramatically simpler — one user, one group, every app writing to shared volumes without stepping on each other's file ownership.

## What's actually running

I keep a handful of apps in a dedicated media namespace that I'm intentionally not going to detail here — what they do and how they're wired up isn't the interesting part of this series, and it's not something I want to document publicly. What *is* relevant to a Kubernetes post: those workloads split across both storage tiers exactly like the last post described (small per-app config volumes on Longhorn, a large shared library on NFS), and one of them needs privileged networking capabilities beyond what the rest of the cluster requires — a good reminder that Kubernetes can still accommodate non-standard networking when you actually need it.

Outside of that namespace, a few things are worth naming specifically:

- **Pi-hole** — DNS for the network, covered in the [networking post]({{< ref "kubernetes-networking" >}}).
- **Home Assistant** — my home automation platform, with its own Ingress and a persistent NFS-backed config volume.
- **Heimdall** — a simple landing page/dashboard linking out to everything else I run.
- **Diun** — the update-tracking watcher from the [configuration post]({{< ref "kubernetes-configuration" >}}).

## What didn't survive

Not every experiment stuck around, and I don't think that's a bad thing to show. A Kubernetes Dashboard deployment went through three rounds of tweaking over about a week and then got deleted — Traefik plus Grafana plus `kubectl` covered what I actually needed, and running a second web UI just to look at the cluster stopped earning its keep. A chat-gateway service got built out over several commits — init container, security context, Sealed Secrets for its API tokens, the works — and was pulled a day later once I decided it wasn't worth the maintenance surface for how often I'd actually use it. Its PVC is still sitting there, unclaimed, which is its own small lesson about cleaning up after yourself.

Neither of those is a failure story exactly — they're closer to "built it, used it for a week, decided it wasn't worth carrying forward." That's a normal part of running a homelab, and I'd rather show that than pretend every manifest I've ever written is still in service.

## The maturity marker

The clearest sign that this stack has moved from "getting it working" to "operating it" is a single commit that touched twelve deployments at once, adding the failure-tolerance settings from the [configuration post]({{< ref "kubernetes-configuration" >}}) — shorter node-failure tolerations, paired with `strategy: Recreate` everywhere a `ReadWriteOnce` volume is involved. That's not a change you make on day one. It's a change you make after a node actually goes down and you sit there watching five minutes tick by before anything comes back.

## Up next

Everything so far has been applied by hand — `kubectl apply -f`, one file at a time, from whichever terminal I happened to be in. The next post is about why that stopped being good enough, and moving to ArgoCD.
