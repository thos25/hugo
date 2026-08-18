---
title: "Configuring the Cluster: TLS, Secrets, and Learning to Not Lose a Node"
description: "Standing up cert-manager with a Cloudflare DNS-01 solver, the three-stage evolution from plaintext secrets to Sealed Secrets, and the day I stopped waiting five minutes for a failed node to give up its pods."
date: 2026-07-31T09:00:00-06:00
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
 - Part3
---

## Recap

[Last time]({{< ref "kubernetes-installation" >}}) I had three MicroK8s nodes joined into a cluster. That's a cluster you can `kubectl get nodes` against — it is not yet a cluster you'd trust with anything real. This post covers the configuration work that closed that gap: certificates, secrets, and a couple of hard-won lessons about how the cluster behaves when a node actually dies.

## TLS: real certificates for an internal-only cluster

Every host I run is a subdomain of `joeyaxtell.com`, but none of them are reachable from the public internet — they all resolve to addresses inside my LAN. That rules out the usual HTTP-01 ACME challenge, which needs Let's Encrypt to reach your server directly. The fix is a **DNS-01 challenge through Cloudflare**, which proves domain ownership by writing a TXT record instead of serving an HTTP response:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-cloudflare
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: YOUR_EMAIL@EXAMPLE.COM
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
```

Worth pointing out: that's the **production** ACME endpoint, not staging. I went straight for real certificates from the first ClusterIssuer I ever wrote, which is a little bold in hindsight — Let's Encrypt's production rate limits are not generous if you get the config wrong and end up retrying in a loop. It worked out, but staging first is the safer habit.

I proved it worked the same way I proved everything else in this series: a disposable `nginx` pod at `test.joeyaxtell.com`, watched until a real certificate showed up in its Secret, then deleted.

## Secrets: three stages, and only the first one is embarrassing

My secrets story is basically a timeline of me learning why each previous approach doesn't scale:

**Stage 1 — plaintext templates committed to git.** The Cloudflare API token secret started life as a literal `kind: Secret` manifest with `api-token: YOUR_CLOUDFLARE_API_TOKEN` committed to the repo, meant to be hand-edited locally before applying. Never a real credential in git — but also never a pattern I'd want to repeat as the number of secrets grew.

**Stage 2 — imperative, out-of-band `kubectl create secret`.** For a while, secrets simply didn't exist in the repo at all — just a comment documenting the `kubectl create secret generic ... --from-literal=...` command I'd run once, by hand, and never again. It works, but it means the repo doesn't actually describe the cluster; there's a whole category of state that only exists in my shell history.

**Stage 3 — Sealed Secrets.** This is where I landed, and where new secrets go today. The [Bitnami Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) controller lets you encrypt a Secret client-side with its public key, commit the encrypted `SealedSecret` to git, and only the controller running in-cluster can decrypt it back into a real Secret. The workflow (yes, including the PowerShell-specific piping, since I do this from Windows):

```
kubeseal --fetch-cert --controller-namespace default --controller-name sealed-secrets-controller > cert.pem
Get-Content secret.yaml | kubeseal --cert cert.pem -o yaml > sealedsecret.yaml
```

That finally makes a secret's *existence* and its encrypted value both visible in git history, without the value ever being recoverable by anyone who doesn't hold the cluster's private key. It's not applied everywhere yet — that migration is still in progress workload by workload — but it's the pattern I reach for now.

## Update tracking: an opt-in watcher

Rather than a blanket "check everything for updates" policy, I run [Diun](https://github.com/crazymax/diun) with Kubernetes provider support enabled, watching every six hours. Workloads opt in individually by adding one annotation to their pod template:

```yaml
annotations:
  diun.enable: "true"
```

That opt-in model matters: I want to know when something has a new image available, but I don't want that noise on things I've deliberately pinned. When I locked one workload to a specific version and set `imagePullPolicy: IfNotPresent` so it would stop drifting, the very next thing I did was pull the `diun.enable` annotation back off it — no point getting paged about updates I've explicitly decided not to take yet.

## Configuring for node failure

The most useful configuration change I made didn't happen until months in, after I actually lost a node and watched what happened: nothing, for a long time. Kubernetes' default tolerance for an unreachable node is generous — by default, pods on a node that goes `NotReady` or `Unreachable` aren't rescheduled for **five minutes**. On a three-node homelab cluster, five minutes of a chunk of your services being down because one box hiccuped is a bad trade.

The fix, applied across every deployment in one pass once I understood the knob:

```yaml
tolerations:
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 30
  - key: "node.kubernetes.io/unreachable"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 30
```

That cuts the eviction wait from five minutes to thirty seconds. I paired it with `strategy: Recreate` on the same deployments — my persistent volumes are `ReadWriteOnce`, so a rolling update trying to attach the same volume to a second pod before the first releases it just hangs. `Recreate` tears the old pod down before standing the new one up, which is slower for a routine deploy but is the only strategy that actually works with single-writer storage.

## Up next

With TLS, secrets, and failure handling in place, the next post covers how traffic actually gets from my LAN to a pod — Traefik, MetalLB, and DNS.
