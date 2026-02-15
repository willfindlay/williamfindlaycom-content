---
title: "Migrating My Homelab to GitOps with Claude"
date: "2026-02-15T16:00:00Z"
description: "How I used Claude Code to turn a pile of ad-hoc kubectl applies into a fully declarative, ArgoCD-managed Kubernetes cluster in a single afternoon."
tags: ["homelab", "kubernetes", "gitops", "ai", "argocd"]
---

## How It Started

The homelab started the way they all do: I just wanted to run one thing. A single-node Talos OS cluster with Cilium for networking. I deployed a Docker registry so I could push my own images. That went fine, so I added a Discord bot. Then MongoDB. Then an Enshrouded game server for friends. Each one was a quick `kubectl apply`, maybe a service, maybe an ingress. Nothing fancy.

The problem with "nothing fancy" is that it accumulates. Six months in, I had a dozen services running, secrets created inline with `kubectl create secret`, TLS certificates I'd set up manually and half-forgotten about, and a directory on my laptop full of YAML files that may or may not have reflected what was actually running in the cluster. There was no version control. No way to rebuild the cluster from scratch if the node died. No audit trail for what changed when.

I kept telling myself I'd clean it up later. "Later" arrived when I realized I couldn't remember how the Docker registry's TLS chain was configured, and I needed to add a new domain.

The cleanup I had in mind was GitOps. One repository describing the entire cluster state, ArgoCD to reconcile it, sealed secrets for credentials, cert-manager for TLS automation. I'd done this professionally, but never bootstrapped it from scratch on my own infrastructure. It felt like a multi-weekend project: lots of boilerplate YAML, lots of cross-referencing Helm chart docs, lots of debugging certificate chains.

I got the entire thing working in an afternoon.

## Claude as an Infrastructure Collaborator

Claude Code is Anthropic's CLI tool that gives you an AI pair programmer in your terminal. It reads your files, runs commands, edits code, and holds a conversation about what you're building. It's not an infrastructure-as-code engine or a deployment tool. It's a collaborator that happens to be good at exactly the kind of work that makes GitOps migrations slow: translating running services into declarative manifests, cross-referencing Helm chart values, debugging cryptic Kubernetes error messages.

The actual architectural decisions (how to structure the app-of-apps pattern, which services need sealed secrets, where to draw the line between Helm charts and raw manifests) took maybe 10% of the total effort. The other 90% was mechanical. That ratio is what made an afternoon possible instead of multiple weekends.

One of the first things I did was create a `CLAUDE.md` file in the repo root describing the cluster architecture, conventions, and known constraints. Claude reads this file automatically at the start of every session. Instead of re-explaining the cluster layout each time, I could say "add a new application for the Docker registry UI" and Claude already knew the template patterns, the namespace conventions, the sync wave ordering, and the secret management approach. This file turned out to be one of the most valuable artifacts of the entire migration, but I'll come back to that.

## Building the Foundation

The first task was setting up the ArgoCD app-of-apps pattern. One root Helm chart that templates an Application custom resource for every service in the cluster. Each Application points to either an upstream Helm chart (with values stored in the repo) or a directory of raw manifests. Push to main, ArgoCD picks it up, cluster converges.

Claude scaffolded the root chart, the shared `values.yaml`, and the first few Application templates. I described what I wanted: multi-source Applications for Helm charts so values could live in the repo alongside the workloads, directory-source Applications for services deployed as raw manifests, auto-sync with pruning and self-heal enabled on everything. Claude produced the templates and I reviewed them against the ArgoCD docs.

The first real problem surfaced during the ArgoCD install itself. The ApplicationSets CRD has an annotation that exceeds Kubernetes' 256KB limit for client-side apply. `kubectl apply` fails with an error message that doesn't obviously point to the annotation size. Claude suggested `--server-side --force-conflicts`, which worked immediately. A small thing, but the kind of thing that eats an hour of searching if you haven't encountered it before.

From there, migrating each service followed a pattern: describe the running service to Claude, have it generate the workload manifests, Application template, and sealed secrets, review everything, commit, and watch ArgoCD pick it up. Sixteen applications went through this process. The per-service turnaround was fast enough that the bottleneck was waiting for ArgoCD to sync, not writing the manifests.

## Sealed Secrets and the Annotation Problem

Moving secrets into the repo meant sealing them with Bitnami's Sealed Secrets controller. The workflow is straightforward: create a plaintext secret, pipe it through `kubeseal`, commit the encrypted output, and the controller decrypts it in-cluster.

Claude handled the mechanical parts well. "Seal this secret for the Docker registry with cluster-wide scope" would produce the right command, remind me to shred the plaintext afterward, and place the sealed output in the correct workload directory. But we hit a problem that took a few rounds to sort out.

`kubeseal` strips custom metadata from the input. If your secret template has ArgoCD sync-wave annotations, Helm ownership labels, or a specific `type` field, `kubeseal` drops all of it. The first time Claude sealed a secret, it produced valid output that was missing its `argocd.argoproj.io/sync-wave: "-1"` annotation. ArgoCD tried to create dependent resources before the secret existed, and things failed in a way that wasn't immediately obvious.

I caught it during review, and we added it to `CLAUDE.md` as a documented gotcha. After that, Claude would automatically re-add stripped annotations and labels after sealing. The Docker registry secret was particularly fiddly because it had Helm ownership annotations (`meta.helm.sh/release-name`, `app.kubernetes.io/managed-by: Helm`) that needed to be preserved for Helm to recognize it as a managed resource.

This became a recurring pattern. Claude would get something 90% right on the first pass, I'd catch an edge case during review, we'd document it in the project instructions, and then it wouldn't happen again. The knowledge accumulated in `CLAUDE.md` rather than in my head.

## The Prettier Incident

I had a pre-commit hook that ran Prettier on staged files. Claude, following good practice, formatted files before committing. Prettier looked at `{{ .Values.spec.source.repoURL }}` in a Helm template and reformatted it to `{ { .Values.spec.source.repoURL } }`. The root Application template broke.

The fix was a one-line revert, but the lesson mattered. We added a convention to `CLAUDE.md`: never format YAML files with Prettier or any other formatter. The pre-commit hook got a `SKIP_FORMAT_CHECK=1` escape hatch for YAML files.

What's worth noting is the feedback loop. Claude broke something, I noticed during review, we documented the constraint, and it never recurred. It's the same dynamic you'd have onboarding a new team member to a project with non-obvious conventions. The difference is that the "onboarding doc" is also the enforcement mechanism; Claude reads `CLAUDE.md` at the start of every session, so documented gotchas are always in context.

## Debugging Certificate Renewal

Setting up cert-manager with Let's Encrypt DNS-01 challenges through Cloudflare was the most complex piece of the migration. The chain involves a ClusterIssuer, Certificate resources for each domain, a sealed Cloudflare API token, and Reflector to mirror the resulting TLS secrets into the namespaces that need them.

The initial setup went smoothly. Claude produced the ClusterIssuer, Certificate resources, and Reflector annotations. Certificates were issued, TLS worked. Then a certificate needed renewal, it failed, and I discovered cert-manager's exponential backoff behavior.

cert-manager tracks failed issuance attempts. If your first few renewal attempts fail (say, because of a stale ACME account key), the backoff timer pushes the next retry hours or days into the future. The error message says "backing off due to previously failed issuance" with a timestamp far in the future. The fix is to patch the Certificate's status subresource to clear the backoff state, which is not something you'd guess from the logs alone.

Claude and I debugged this interactively. I'd ask it to check the cert-manager logs, it would read and interpret the output, I'd ask what could cause that specific error, and we'd work through the possibilities. The root cause turned out to be a stale ACME account key, but along the way we also discovered that failed challenges can leave orphaned `_acme-challenge` TXT records in Cloudflare DNS. These orphaned records cause cert-manager's Cloudflare client to cache an empty zone ID, which makes all subsequent challenges fail with a different and more confusing error.

The full fix was a multi-step process: verify the Cloudflare API token, check for stale TXT records and delete them through the Cloudflare API, restart cert-manager to clear its cache, delete all outstanding CertificateRequests, and patch the Certificate status to reset the backoff timer. We wrote all of this up as a troubleshooting runbook in the project docs.

This kind of iterative debugging is where conversational AI genuinely helps. Infrastructure problems are rarely one-step. You check one thing, it reveals something else, you fix that, another issue surfaces. Being able to say "check the logs," then "what does that error mean," then "try deleting the account key and see if it re-registers" in natural language, with Claude executing the commands and interpreting the results, compresses the feedback loop.

## What Worked

The biggest time savings came from mechanical translation. Taking a service that I'd deployed by hand and producing the full set of declarative manifests (Deployment, Service, Ingress, PVC, Application template, sealed secrets with correct annotations) is straightforward but slow when done manually. Claude did it in minutes per service. Across sixteen applications, that's what compressed a multi-weekend project into an afternoon.

Claude's persistent memory across sessions proved useful for ongoing maintenance after the initial migration. It remembered that my WSL environment uses `pacman` instead of `apt`, that the `yq` binary is the Go version (not the Python wrapper), that the `argocd` CLI has a namespace issue and you need to use `kubectl annotate` as a workaround. These environment-specific details normally cost you five minutes of confusion every time you context-switch back to a project.

And `CLAUDE.md` evolved from "context for the AI" into genuine cluster documentation. Every debugging session, every gotcha, every architectural decision ended up recorded there. When I need to remember why Cilium isn't managed by ArgoCD (a CNI misconfiguration could break networking and prevent ArgoCD from self-correcting) or how the backup drive mounting works, the answer is in that file. It's the most thorough documentation I've ever maintained for a personal project, and it exists largely because I was writing it for Claude as much as for myself.

## What Didn't Work

Claude can't interact with web dashboards. A significant part of the Cloudflare Tunnel configuration (routing public hostnames to in-cluster services) happens in the Cloudflare Zero Trust dashboard. Claude could describe exactly what to configure, but I had to click through the UI myself.

Complex debugging sometimes went in circles before converging. When the Cloudflare DNS-01 challenge was failing with empty zone IDs, Claude's first instinct was to check the API token permissions. That was a reasonable hypothesis but the wrong one. It took a few rounds of "that's not it, try something else" before we landed on orphaned TXT records as the root cause. A human engineer with deep Cloudflare experience might have gotten there faster.

I also reviewed every manifest before committing and caught issues maybe 15% of the time. Incorrect indentation, missing annotations, wrong port numbers. The error rate was low enough that the workflow was still much faster than writing everything by hand, but high enough that skipping review would have been a bad idea.

## The End State

The cluster is now fully described in a single Git repository. Sixteen applications managed through the app-of-apps pattern. Sealed secrets for every credential. cert-manager handling TLS with automated renewal. Daily backup CronJobs that rsync persistent volumes to a USB drive with 30-day snapshot retention. A Cloudflare Tunnel providing internet access to selected services without any port forwarding or public IP.

Pushing a change to main and watching ArgoCD converge the cluster within three minutes is a different experience than `kubectl apply`. When something breaks, `git log` shows what changed and when. Rolling back is a `git revert`.

The migration itself took one afternoon. Most of the saved time came from boilerplate generation, not architectural guidance. The AI didn't decide how to structure the cluster; I did. But it compressed the gap between having a plan and having a running implementation down to almost nothing. The decisions were mine, and the YAML was Claude's, and that division of labour turned out to work well.

If you're considering a similar migration, start with the documentation. Write down your cluster architecture, your conventions, your known gotchas, before you write a single manifest. Whether you're using an AI tool or working alone, that document becomes the foundation everything else builds on. Mine just happened to be called `CLAUDE.md`.
