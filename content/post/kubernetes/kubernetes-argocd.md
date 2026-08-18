---
title: "Moving to ArgoCD: An App-of-Apps, and the Bug That Taught Me How Helm Rendering Works"
description: "Why kubectl apply stopped being good enough, installing ArgoCD as a self-managed Helm chart, and an app-of-apps gotcha that quietly left half my Applications running outside GitOps for months."
date: 2026-08-13T09:00:00-06:00
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
 - Part7
---

**Homelab Kubernetes Series:** [1. Intro]({{< ref "kubernetes-intro" >}}) · [2. Installation]({{< ref "kubernetes-installation" >}}) · [3. Configuration]({{< ref "kubernetes-configuration" >}}) · [4. Networking]({{< ref "kubernetes-networking" >}}) · [5. Storage]({{< ref "kubernetes-storage" >}}) · [6. Workloads]({{< ref "kubernetes-workloads" >}}) · **7. ArgoCD** · [8. GitOps]({{< ref "kubernetes-gitops" >}})

## Recap

Everything up through [the last post]({{< ref "kubernetes-workloads" >}}) got onto the cluster the same way: `kubectl apply -f`, from whichever terminal I happened to have open, applied in whatever order I remembered to run things. That works right up until it doesn't. This post is about why I moved to ArgoCD, and a bug in my own setup that I didn't catch for months.

## Why kubectl apply stopped being enough

A few things pushed me here. There was no record of what was actually deployed versus what I'd only ever run once from history. There was no reconciliation — if something drifted from the manifest, or someone (me, running a one-off `kubectl edit`) changed it directly, nothing would ever notice or fix it. And there was no single place to look and answer "is the cluster in the state I think it's in." GitOps fixes all three: git is the source of truth, and a controller in the cluster keeps reality in sync with it.

I followed [Micah Bird's homelab ArgoCD guide](https://www.micahbird.com/p/how-to-setup-argocd-the-homelab-way/) as the starting pattern, in a separate repo from the workload manifests themselves.

## Installing ArgoCD, and having it manage itself

ArgoCD is installed as a small Helm wrapper chart around the upstream `argo-helm` chart:

```yaml
apiVersion: v2
name: argo-cd
version: 3.2.1
dependencies:
  - name: argo-cd
    version: 9.1.7
    repository: https://argoproj.github.io/argo-helm
```

The interesting part isn't the install itself, it's what happens right after: ArgoCD manages its own upgrade going forward, as just another Application pointed at that same chart path in the same repo. Once it's up, I don't run `helm upgrade` by hand anymore. I bump the version in git and let ArgoCD reconcile itself.

Ingress needed one non-obvious detail. ArgoCD's UI and CLI both talk gRPC, which doesn't play nicely with a plain HTTP-routed IngressRoute, so the route needs a second, higher-priority rule specifically for gRPC traffic, upgraded to `h2c`:

```yaml
- kind: Rule
  match: Host(`argocd.joeyaxtell.com`) && Header(`Content-Type`, `application/grpc`)
  priority: 11
  services:
    - name: argo-cd-argocd-server
      port: 80
      scheme: h2c
```

Without that, the web UI loads fine but the CLI and any gRPC-based calls fail in confusing ways.

## App-of-apps

The pattern I landed on is the classic app-of-apps: one root Application watches a directory in git, and every file in that directory becomes another Application that ArgoCD manages:

```yaml
spec:
  source:
    repoURL: https://github.com/thos25/argo-cd.git
    path: apps/
    targetRevision: HEAD
  syncPolicy:
    automated:
      prune: true
      allowEmpty: true
      selfHeal: true
```

Add a workload, add one small Application file describing where its manifests live and where it should be deployed, and ArgoCD picks it up automatically. Each one follows the same shape: point at a path in my [homelab-kubernetes]({{< ref "kubernetes-workloads" >}}) repo, sync automatically, prune anything removed, and self-heal anything that drifts.

```yaml
spec:
  source:
    repoURL: https://github.com/thos25/homelab-kubernetes
    targetRevision: HEAD
    path: media/<app>/
    directory:
      recurse: true
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers: [/spec/replicas]
```

That last bit, `ignoreDifferences` on replica count, exists so that if I manually scale something down for maintenance, ArgoCD's self-heal doesn't immediately scale it back up and fight me.

## The bug: not every Application file is actually an Application

Here's the one I didn't catch for a while, and I think it's a genuinely useful lesson about how Helm rendering works. My `apps/` directory is itself a Helm chart — it has a `Chart.yaml` — which means ArgoCD renders it as a Helm chart. Helm has a rule I hadn't fully internalized: **only files under `templates/` get rendered as manifests.** Anything sitting at the chart root is just a file Helm can reference via `.Files`, but it never gets emitted.

I had new workload Applications sitting at the chart root instead of inside `templates/`. They were valid YAML, they were committed to git, and the root app-of-apps Application showed as `Synced` at the exact commit that added them. Every signal you'd normally trust said everything was fine. But because they weren't inside `templates/`, Helm silently never rendered them, and they never became real ArgoCD Applications. The workloads they described only exist in my cluster today because I `kubectl apply`'d them by hand at some point. GitOps for those apps is, right now, an illusion. If I'd edited that file in git expecting a sync to follow, nothing would have happened, because ArgoCD never rendered it as anything to sync.

The fix is a one-line move — `apps/pihole.yml` becomes `apps/templates/pihole.yml` — and it's on my list. I'm leaving the mistake in this post instead of quietly writing around it, because "the sync status looked healthy and I was still wrong" is exactly the kind of thing worth knowing to check for.

## Secrets and private repos

ArgoCD needs credentials to pull from a private GitHub repo, which it gets from an ordinary Secret carrying one special label:

```yaml
metadata:
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/[GITHUB_REPO_HERE]
  username: [GITHUB USERNAME HERE]
  password: [GITHUB PAT TOKEN HERE]
```

That label is what tells ArgoCD to treat this Secret as repository credentials rather than just an opaque Secret sitting in the namespace. The real values are created out-of-band, the same way the Cloudflare token was in the [configuration post]({{< ref "kubernetes-configuration" >}}) — the committed file is a template, never a credential.

## Up next

Getting ArgoCD installed is one thing; actually leveraging it day to day is another. The next post covers what's really running through it today, what I tried and had to walk back, and where the gaps still are.

---

**◀ Previous:** [6. Workloads]({{< ref "kubernetes-workloads" >}})  |  **Next ▶:** [8. GitOps]({{< ref "kubernetes-gitops" >}})
