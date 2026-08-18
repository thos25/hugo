---
title: "GitOps in Practice: What's Actually Automated, and What's Still Manual"
description: "An honest inventory of what ArgoCD really manages today, an AppProject idea that didn't survive contact with reality, a config-drift trap in a values.yaml file, and where GitOps is headed next."
date: 2026-08-16T09:00:00-06:00
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
 - Part8
---

## Recap

[Last post]({{< ref "kubernetes-argocd" >}}) covered getting ArgoCD installed, including a rendering bug that quietly left some Applications running outside GitOps entirely. To close out the series, I want to give an honest snapshot of what GitOps actually looks like day to day in this homelab right now — not the aspirational version, the real one.

## What's genuinely under GitOps today

Auto-sync is on everywhere, with both `prune` and `selfHeal` enabled on every Application — no exceptions. In practice that means: edit a manifest in git, push, and within a few minutes the cluster matches. Delete a resource from git, and ArgoCD removes it from the cluster. Edit something in the cluster directly instead of in git, and ArgoCD quietly reverts it back to match git the next reconciliation loop. That last one took some getting used to — the first time I `kubectl edit`'d something to test a quick change and watched ArgoCD undo it thirty seconds later, it was a good reminder of what "git is the source of truth" actually means in practice, not just in theory.

Coverage today is honest but incomplete: the workloads reachable from git are the media namespace apps and a handful of others — real, working, auto-syncing. The **platform layer isn't there yet** — cert-manager, Traefik, Longhorn, and the observability stack are all still installed and managed the way they were in the [installation]({{< ref "kubernetes-installation" >}}) and [configuration]({{< ref "kubernetes-configuration" >}}) posts: by hand, outside git. GitOps started with the easiest, most repetitive layer first and hasn't climbed down to the foundation yet. That's next.

## Sealed Secrets, running for real

Unlike the ArgoCD image updater below, this one's live: the Sealed Secrets controller itself is deployed as an ArgoCD Application, sourced straight from its upstream Helm repo rather than my own git repo:

```yaml
spec:
  source:
    repoURL: https://bitnami-labs.github.io/sealed-secrets
    chart: sealed-secrets
    targetRevision: 2.18.3
```

That's the same controller powering the encrypted-secret workflow from the [configuration post]({{< ref "kubernetes-configuration" >}}) — and it's a good example of ArgoCD managing infrastructure that isn't "my" code at all, just a chart I depend on.

## What I tried and walked back: per-project isolation

Early on I wanted ArgoCD's AppProject concept to segment my workloads — media apps in one project, platform in another, each with its own repo and resource restrictions. I wrote the AppProject, and then wrote the honest verdict directly into the file when I hit the wall:

```yaml
# This doesn't work currently. You can only tie 1 repo to 1 project.
# If I had my deployments in separate repos, then I'd create new projects for each.
```

All of my workload manifests live in one repo (`homelab-kubernetes`), so a project boundary drawn around "one repo" doesn't actually separate anything. That AppProject was never applied — every Application in the cluster still runs under the `default` project. Splitting workloads into separate repos per concern is the real fix, and it's a bigger reorganization than I've wanted to take on yet. I'm including the dead end here because "I tried this and the docs made it sound simpler than it turned out to be" seems more useful to write down than pretending the idea worked the first time.

## What's built but not live: image automation

I added the ArgoCD Image Updater's Application manifest, and annotated one workload (Pi-hole, using its `yyyy.mm.x` version scheme) to be managed by it:

```yaml
annotations:
  argocd-image-updater.argoproj.io/image-list: pihole=pihole/pihole
  argocd-image-updater.argoproj.io/pihole.update-strategy: latest
  argocd-image-updater.argoproj.io/pihole.allow-tags: regexp:^\d{4}\.\d{2}\.\d+$
  argocd-image-updater.argoproj.io/write-back-method: git
```

`write-back-method: git` is the part I like most about the design — instead of silently mutating the live Deployment, the updater would commit the new image tag back to git itself, so even automated version bumps stay visible in commit history. The catch: the Image Updater's own Application never actually got deployed, so none of this is running yet. The annotations are sitting there, correctly configured, waiting for their controller to exist. It's on the same list as the app-of-apps fix from the last post.

## The gotcha that's still live: a values.yaml nobody reads

This is the one I want to flag clearest, because unlike the others it's a silent landmine rather than an inert feature. ArgoCD's own Helm install has a `values.yaml` sitting at my repo root, with two settings I actually care about:

```yaml
configs:
  params:
    server.insecure: "true"
  cm:
    accounts.admin: "apiKey,login"
```

But the Application that installs ArgoCD points at `charts/argo-cd`, and that directory has no `values.yaml` of its own. My root-level file is never read by ArgoCD at all — it's a leftover from the very first `helm install -f values.yaml` I ran by hand, before ArgoCD was managing itself.

Right now those two settings are still active in the cluster, purely because ArgoCD's reconciliation merges into the existing ConfigMaps rather than replacing them wholesale — the values survive as leftover keys Helm's chart defaults don't know to remove. But that's fragile: if either ConfigMap ever gets deleted and recreated from scratch, both settings silently revert to upstream defaults. `server.insecure` reverting to `false` would be the visible one — my Traefik route talks plain HTTP to ArgoCD's server on the assumption that TLS is terminated upstream, so a default-secure ArgoCD would break the UI behind its own working certificate. The fix is simple, moving the file into `charts/argo-cd/values.yaml` where the Application actually looks for it, and it's genuinely the next thing I'm doing after publishing this post.

## Where this goes next

Looking at everything laid out across these gotchas, the real GitOps roadmap for this homelab is:

- Fix the app-of-apps rendering bug so every Application actually lives under `templates/`.
- Move ArgoCD's `values.yaml` to where its own Application will actually read it.
- Split workloads into separate repos (or at least separate paths with real project boundaries) so AppProjects can do something useful.
- Deploy the Image Updater for real, now that the annotations are already sitting there configured.
- Bring the platform layer — cert-manager, Traefik, Longhorn, observability — into git the same way the workloads already are.

None of that is a rewrite. It's closing gaps I only found by actually running this for months and watching where reality quietly diverged from what the repo said it should be — which, in hindsight, was the actual point of this whole series: not "here's a perfect homelab," but "here's what building one for real, mistakes included, actually looks like."

Thanks for following along.
