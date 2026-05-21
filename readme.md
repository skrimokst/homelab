# Homelab

A personal homelab combining bare-metal services, virtualised storage, and Kubernetes workloads — built to understand infrastructure better hands-on, and to replace subscription services with self-hosted alternatives where it makes sense.

> **Status:** A relatively young lab (a few months old), actively being built out. Setup and maintenance are scripted and idempotent; the actual configurations and scripts are private. This repo documents the architecture and the bootstrap pattern.

## Why it exists

Two main reasons:

- **To get better at infrastructure.** It's a homelab — the point is to build, manage, and maintain real infrastructure hands-on, and to understand the trade-offs by living with the consequences of my own decisions.
- **To reduce reliance on subscription services.** Self-hosting alternatives where the effort is worth it, and keeping my own data on my own hardware.

## Overview

The lab runs across a mix of bare metal and virtualised infrastructure:

- **OPNsense** — networking and firewall (bare metal)
- **Proxmox** — hypervisor (bare metal)
- **Home Assistant** — home automation (bare metal)
- **TrueNAS** — storage (virtualised under Proxmox)
- **k3s** — application workload orchestration, backed by TrueNAS storage

Running on the k3s cluster: a self-hosted **Gitea** instance with Actions, **Grafana** for observability, **Renovate** for automated dependency hygiene, and a personal media/photo library.

## The bootstrap pattern

The lab is designed so that rebuilding it from scratch is a known, repeatable procedure rather than an exercise in remembering what was installed where. The whole environment bootstraps from a minimal starting point:

```
Proxmox (bare metal)
  └── Gitea LXC
        └── Gitea Actions workflows
              └── provision & maintain every other service
```

From that minimal install — Proxmox plus a Gitea LXC — Gitea Actions workflows handle provisioning and ongoing maintenance of everything else. New services, dependency updates, configuration changes: all flow through workflows in private repositories.

This isn't GitOps in the strict sense — there's no ArgoCD or Flux continuously reconciling declarative state — but it's the same family of ideas: Git as the source of truth, automation handling the actual changes, and a small, bounded manual surface area.

## Secrets and certificates

Two pieces of the security story that I've deliberately invested in, both currently around 80% rolled out:

- **Secrets** are managed mostly through **Vaultwarden**. The remaining gap is services that don't expose a backend CLI or SDK to manage secrets programmatically — those still need a manual touch.
- **Certificates** are issued and renewed by **cert-manager** in k3s, backed by an internal **step-ca** certificate authority. Each service gets its own short-lived (24h) certificate, automatically rotated, rather than the single wildcard certificate I relied on previously. Shorter-lived per-service certs mean a smaller blast radius and no long-lived shared secret sitting around.

## Why this shape

A few reasons the architecture ended up this way:

- **Disaster recovery should be a known procedure, not a panic.** Automated paths cover provisioning and maintenance, but some procedures are inherently manual — recovering from hardware failure, or rebuilding when the Proxmox host itself fails. Those live as written runbooks, so the manual paths are as defined as the automated ones, and a failure means following a procedure rather than spending an afternoon on archaeology.
- **Idempotent provisioning is non-negotiable.** Every script can be re-run safely. This is what makes the lab genuinely useful as an R&D environment: I can break things deliberately and put them back.
- **Maintenance should feel like normal engineering work.** Renovate raises dependency-update PRs, workflows run, services update. The mental model is the same as professional software development, with the lab as the target.

## What I've learned

- **The "Gitea bootstraps everything" pattern is worth the up-front cost.** Setting up the workflows is real work, but it's paid back many times over by how small every subsequent change becomes.
- **TrueNAS as a storage VM (rather than bare metal) was the right call.** It folds storage into the Proxmox snapshot/backup story instead of leaving it as a separate concern.
- **Observability matters even at home.** Grafana has caught slow-burning problems — disks filling up, services restarting more often than they should — that I'd otherwise not have noticed for weeks.
- **Claude Code as the primary build tool let me ramp on unfamiliar infrastructure fast.** I owned the architecture and decisions; the agent drove much of the implementation. Working this way on genuinely new-to-me tooling (k3s, step-ca, cert-manager) compressed the learning curve significantly — I could make informed decisions without first grinding through every implementation detail by hand.

## Open questions / honest caveats

- **Backups aren't implemented yet — a matter of sequencing, not oversight.** The lab is young and currently holds little irreplaceable data. As it matures and accumulates content worth protecting, the plan is a proper local-plus-remote setup: local for fast restores, remote/offsite for genuine disaster recovery.
- **The "rebuilds from nothing" property only fully holds for the services, not the storage layer.** TrueNAS is deliberately excluded from the automated bootstrap, because rebuilding a storage pool involves disk mirroring and data-integrity considerations that don't fit a "just run the workflow" model.
- **Moving to a proper reconciliation tool is a natural next step.** ArgoCD or Flux (undecided which) would add continuous drift correction, rather than the event-driven provisioning model I have now. The current approach is the same family of ideas, but it reacts to changes rather than continuously reconciling toward a declared state.
- **Secret and certificate management are both around 80% complete.** The remaining work is the awkward long tail — services without good automation hooks for secrets, and finishing the cert-manager/step-ca rollout across everything.
