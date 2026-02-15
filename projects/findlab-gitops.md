---
title: "findlab-gitops"
date: "2026-02-15T12:00:00Z"
description: "Single-node Talos OS homelab cluster managed entirely via ArgoCD GitOps."
tags: ["kubernetes", "gitops", "argocd", "helm", "talos", "homelab"]
status: "active"
featured: true
---

The GitOps repository that declares the entire state of my homelab Kubernetes cluster. A single Talos OS node runs every workload, managed through ArgoCD's app-of-apps pattern. One root Helm chart generates all child Application CRs. Push to `main` and ArgoCD picks up the change within minutes. No `kubectl apply` allowed.

The cluster hosts a private Docker registry, a MongoDB instance, a Discord bot, an Enshrouded game server, and this website. All of it defined in YAML, all of it version-controlled.

## Features

- **App-of-apps pattern**: A single root Helm chart in `root-app/` templates out every ArgoCD Application CR. Adding a new service means adding one template file and a `workloads/` directory.
- **Zero-trust internet exposure**: A Cloudflare Tunnel provides inbound access without any public IP or port forwarding. The `cloudflared` pod makes outbound-only connections to Cloudflare's edge.
- **Automated TLS**: cert-manager issues Let's Encrypt certificates via DNS-01 challenges against the Cloudflare API. Reflector mirrors TLS secrets across namespaces so each workload gets its cert without duplication.
- **GitOps-safe secrets**: Every secret is sealed with `kubeseal --scope cluster-wide` before entering the repo. Plaintext never touches Git.
- **Nightly PV backups**: A privileged CronJob detects the USB backup drive by stable device ID, mounts it, and rsyncs all persistent volumes with hardlink-based snapshots. 30-day retention. Gracefully skips if the drive is disconnected.
- **Self-healing and pruning**: All applications auto-sync with `selfHeal` and `prune` enabled. Manual cluster changes get reverted. Deleted manifests get cleaned up. PVCs are excluded from pruning to prevent data loss.

## Stack

| Component | Technology |
|-----------|-----------|
| OS | Talos OS v1.7.5 |
| Kubernetes | v1.30.1 (single node) |
| GitOps | ArgoCD v3.3.0 |
| CNI | Cilium 1.16.1 |
| Secret management | Sealed Secrets v0.35.0 (`kubeseal`) |
| TLS automation | cert-manager + Let's Encrypt (DNS-01) |
| Ingress | Cilium IngressClass (LAN) + Cloudflare Tunnel (internet) |
| Storage | Rancher `local-path-provisioner` |
