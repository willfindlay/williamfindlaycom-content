---
title: "findlab-gitops"
date: "2026-02-15T12:00:00Z"
description: "Single-node Talos OS homelab cluster managed entirely via ArgoCD GitOps."
tags: ["kubernetes", "gitops", "argocd", "helm", "talos", "homelab"]
status: "active"
featured: true
---

findlab-gitops is the GitOps repository that declares the entire state of my homelab Kubernetes cluster. The cluster runs on a single Talos OS node and uses ArgoCD's app-of-apps pattern to manage every workload from a single root Helm chart. Push to `main` and ArgoCD picks up the change within minutes. No `kubectl apply` allowed.

The cluster hosts a private Docker registry with htpasswd auth, a MongoDB instance via the community operator, a Discord bot backed by Postgres and Redis, an Enshrouded game server, and this website. Infrastructure services include cert-manager for automated Let's Encrypt TLS (DNS-01 via Cloudflare), Sealed Secrets for encrypting secrets at rest in Git, Reflector for mirroring TLS certs across namespaces, and a Cloudflare Tunnel for exposing services to the internet without any public IP or port forwarding.

All persistent data lives on local-path volumes backed up nightly to a USB drive via a privileged CronJob. The backup system uses hardlink-based snapshots with 30-day retention and exits gracefully if the drive is disconnected. Secrets never touch Git in plaintext. Every secret goes through `kubeseal` with cluster-wide scope before it enters the repo.
