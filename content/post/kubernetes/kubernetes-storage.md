---
title: "Storage in the Homelab: NFS for Bulk, Longhorn for Anything With a Database"
description: "The two storage tiers I run side by side, why containers with a local database dependency had to move off NFS after fighting SQLite locking issues, and the throwaway debug pod I built to poke at block volumes."
date: 2026-08-06T09:00:00-06:00
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
 - Part5
---

**Homelab Kubernetes Series:** [1. Intro]({{< ref "kubernetes-intro" >}}) · [2. Installation]({{< ref "kubernetes-installation" >}}) · [3. Configuration]({{< ref "kubernetes-configuration" >}}) · [4. Networking]({{< ref "kubernetes-networking" >}}) · **5. Storage** · [6. Workloads]({{< ref "kubernetes-workloads" >}}) · [7. ArgoCD]({{< ref "kubernetes-argocd" >}}) · [8. GitOps]({{< ref "kubernetes-gitops" >}})

## Recap

Traffic can reach a pod as of [the last post]({{< ref "kubernetes-networking" >}}). This one is about what happens after that: where the data actually lives, and why I ended up running two completely different storage systems instead of one.

## The split: NFS vs. Longhorn

I run storage on two tiers, and the dividing line comes down to one question: **does this workload need a single-writer block device, or can it live happily on shared network storage?**

**Tier one is NFS**, backed by a TrueNAS box on my network. I provisioned a handful of statically-defined PersistentVolumes against it: one for general app config, one for a large shared media library, and one for another large content volume, all `ReadWriteMany` and all set to `Retain` so deleting the claim never deletes the data.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: truenas-config-pv
  labels:
    type: config
spec:
  capacity:
    storage: 1Ti
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: 192.168.0.126
    path: /mnt/media/configs
  storageClassName: config
```

There's no dynamic NFS provisioner in the mix, so binding a PVC to the right PV is done the old-fashioned way, with label selectors:

```yaml
spec:
  selector:
    matchLabels:
      type: config
```

A handful of small apps — a dashboard, a home automation platform, an update-tracking service, a DNS server — all share that single config PV, each carving out its own directory with `subPath`. It's a simple pattern, and it works well for anything that just wants a folder to read and write config files from, shared across as many pods as need it.

**Tier two is Longhorn**, MicroK8s's built-in distributed block storage addon, used for `ReadWriteOnce` volumes where something wants exclusive, low-latency access to its own disk. Every app in my media namespace that needs a config volume gets its own small dedicated Longhorn PVC (a few gigabytes each) rather than sharing the NFS config volume the smaller apps use.

Longhorn itself lives entirely in the addon layer. `microk8s enable` turned it on, so the only thing I actually own in git for it is its Ingress, giving me a web UI at `longhorn.joeyaxtell.com`. That also means I don't have replica counts, backup targets, or a documented disaster-recovery story committed anywhere in the repo yet. It's a gap, and it's on the list.

## The migration that taught me the dividing line

I didn't start out with a clean rule for which tier a workload belonged on. I found the rule the hard way. A handful of containers in my media namespace keep their own local state in an embedded database, SQLite in every case that bit me. SQLite and NFS do not get along: file locking over a network filesystem is unreliable enough that those apps would intermittently stall or throw database-locked errors under normal use.

The fix was a straight storage migration: move each affected app's config volume off the shared NFS PVC and onto its own dedicated Longhorn PVC instead. The difference was immediate. No more lock contention, because Longhorn gives each pod exclusive block-level access to its own volume instead of arbitrating access across the network. That's the same reasoning behind putting every media app's config volume on Longhorn in the first place: anything with a real database wants a real disk, not a network share, even if the network share is easier to set up.

The rule of thumb I use now: **bulk, shared, or read-heavy data → NFS. Anything with an embedded database or that's picky about file locking → Longhorn.**

## A debug pod for poking at block storage

One annoyance with `ReadWriteOnce` Longhorn volumes: you can't just browse them from outside the pod that's using them the way you can mount an NFS share from anywhere. So I keep a small utility pod around for exactly that:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-toolbox
spec:
  restartPolicy: Never
  containers:
    - name: toolbox
      image: ubuntu:22.04
      command: ["sleep", "infinity"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: deluge-config-pvc # change this to the PVC you want to access
```

The workflow is three commands:

```
kubectl exec -it pvc-toolbox -- bash
kubectl cp /local/path/to/file pvc-toolbox:/data/path/in/pvc
kubectl delete pod pvc-toolbox
```

Point it at whatever PVC needs inspecting, exec in or `kubectl cp` files in and out, then throw the pod away. It's not fancy, but it's saved me from writing one-off debugging tools more times than I expected.

## Up next

With traffic routed and storage sorted, the next post covers what's actually running on top of all this — how I converted Compose stacks into Kubernetes manifests, the conventions I standardized on, and a couple of experiments that didn't survive contact with reality.

---

**◀ Previous:** [4. Networking]({{< ref "kubernetes-networking" >}})  |  **Next ▶:** [6. Workloads]({{< ref "kubernetes-workloads" >}})
