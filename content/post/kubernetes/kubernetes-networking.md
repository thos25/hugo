---
title: "Kubernetes Networking in the Homelab: Traefik, MetalLB, and Pi-hole"
description: "How traffic gets from my LAN to a pod: a MetalLB address pool, Traefik as the ingress layer, and Pi-hole handling DNS for every joeyaxtell.com subdomain on the network."
date: 2026-08-03T09:00:00-06:00
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
 - Part4
---

## Recap

By [the last post]({{< ref "kubernetes-configuration" >}}) I had certificates and secrets sorted out. None of that matters if traffic can't actually reach a pod. This post is about the path a request takes from a browser on my LAN to a container running in the cluster.

## MetalLB: giving the cluster real LAN addresses

Bare-metal Kubernetes has no cloud provider to hand out `LoadBalancer` IPs for you, which is the gap [MetalLB](https://metallb.universe.tf/) fills — it's a MicroK8s addon here, so there's no manifest of my own to show you, but the effect is that `type: LoadBalancer` Services get a real, dedicated IP out of a pool I control instead of sitting in `<pending>` forever.

Every application in the cluster shares one pattern: it sits behind Traefik, and Traefik itself gets exactly one of those addresses. The single exception is DNS. Pi-hole needs to *be* the DNS resolver for the network, on a fixed, memorable IP, answering on the actual DNS port — routing that through an HTTP reverse proxy doesn't make sense, so it gets its own dedicated `LoadBalancer` IP straight from the pool:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: pihole-dns
spec:
  type: LoadBalancer
  loadBalancerIP: 192.168.0.241
  ports:
    - name: dns-udp
      port: 53
      protocol: UDP
    - name: dns-tcp
      port: 53
      protocol: TCP
```

Every device on my network points its DNS at that one address.

## Traefik: one ingress, host-based routing

[Traefik](https://traefik.io/) is installed by hand via Helm rather than a MicroK8s addon:

```
helm install traefik traefik/traefik -f traefik_values.yaml --wait
```

The values file is intentionally small — a dashboard `IngressRoute` and one interesting forward-looking setting: the Kubernetes Gateway API provider is turned on.

```yaml
providers:
  kubernetesGateway:
    enabled: true
gateway:
  listeners:
    web:
      namespacePolicy:
        from: All
```

I'll be honest about where that actually stands today: every workload in the cluster is still routed with a classic `networking.k8s.io/v1` Ingress, not a Gateway API `HTTPRoute`. Turning on the provider was me leaving a door open for later rather than something I'm using yet. If you're setting this up fresh today, going straight to Gateway API is probably the better long-term bet — I just hadn't gotten there when I wrote these values.

What *is* the real pattern is one dedicated subdomain per app — `sonarr.joeyaxtell.com`, `pihole.joeyaxtell.com`, `grafana.joeyaxtell.com`, `argocd.joeyaxtell.com`, and so on — each with an Ingress that references the same TLS ClusterIssuer from [the last post]({{< ref "kubernetes-configuration" >}}). Traefik terminates TLS, matches the `Host()` rule, and forwards to the matching in-cluster Service.

One Ingress breaks that pattern on purpose: my Grafana Ingress runs plain HTTP with `tls: []`, since it's the observability stack's own dashboard and I haven't gotten around to giving it a certificate the way everything else has one. It's on the list.

## DNS: Pi-hole as the front door for hostnames

With Pi-hole holding a fixed LAN address and handling DNS for the whole network, every one of those `*.joeyaxtell.com` hostnames just needs a local DNS entry pointing back at Traefik's address. That's what makes "internal-only, real TLS cert, friendly hostname" all work together: Pi-hole resolves the name, Traefik terminates the certificate and routes by host header, and from a browser's perspective it looks exactly like a normal public site.

Pi-hole also happens to be the clearest example of the Sealed Secrets pattern from the last post in practice — its upstream resolvers and web admin password are stored as an encrypted `SealedSecret` and pulled in with `envFrom.secretRef`, rather than sitting in the deployment manifest as plain text.

## What's deliberately not here

No service mesh, no Tailscale or WireGuard overlay — everything described here is LAN-only, reachable because Pi-hole resolves the hostname and MetalLB/Traefik own the routing. A few workloads in my media namespace have their own networking requirements beyond this — I'll leave those out here, since they're app-specific rather than cluster networking concerns.

## Up next

Traffic can reach a pod now. Next up: what happens to the data once it gets there — the two storage tiers I ended up running side by side, and why.
