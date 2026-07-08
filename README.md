# multi-tenancy-gitops-apps

The **applications layer** of a GitOps-managed Cloud Pak for Integration deployment: per-environment Kustomize overlays for the integration workloads, where an environment is a directory and a promotion is a pull request.

> Part of the IBM Client Engineering **Cloud Pak for Integration production-deployment demo** — the repo Argo CD watches for *what runs*, downstream of the operators and infra that make it possible to run.

## What this is

This is the desired-state catalog for the demo's runtime workloads. Argo CD continuously reconciles this repo onto the cluster; the Tekton pipelines in the factory repos don't `oc apply` anything — they **open pull requests here**. Merging a PR is the act of deploying. Moving a change from `dev` to `staging` to `prod` is a commit into the next directory. Every deployment is therefore reviewable, auditable, and revertable in git.

## What's inside

- **`mq/environments/`** — one overlay per environment (`dev`, `staging`, `prod`, plus `ci`), each holding an IBM MQ **`QueueManager`** CR (`mq.ibm.com/v1beta1`) for `qm01` with its MQSC static definitions. The image tag on each `QueueManager` is stamped by the `mq-infra` pipeline as it delivers a validated build.
- **`ace/environments/`** — one overlay per environment (`dev`, `staging`), each holding an App Connect Enterprise **`IntegrationRuntime`** CR (`appconnect.ibm.com/v1beta1`, v13.0) for the `createcustomer` flow plus its `configurations`. `IntegrationRuntime` is the current 2026 ACE CR — it replaces the deprecated `IntegrationServer`.
- **`cp4i-support/`** — the supporting cast, folded into git so the demo is reproducible end to end:
  - `soapserver/` — the SOAP backend the ACE `createcustomer` flow calls.
  - `ace-bar/` — the nginx **BAR host** that serves the `barURL` the `IntegrationRuntime` pulls from (RWX PVC so the ACE pipeline can publish to it).
  - `mq-spring-app/` — the MQ JMS app exposed through API Connect.
  - `rbac/` — the ClusterRole/RoleBindings letting Argo CD manage `IntegrationRuntimes` and the `ci` pipeline service account copy BARs into `tools`.
  - `argocd/` — Argo `Application`s for the workloads above.
  - `argocd-environments/` — Argo `Application`s that point Argo CD at the `mq/` and `ace/` environment overlays (e.g. `dev-mq-qm01`, `dev-ace-createcustomer`), each pinned to a `path` + `targetRevision`.
- **`cntk`, `shared`, `apic`** — Cloud-Native-Toolkit CI/CD scaffolding, shared config (e.g. an `scc` Helm chart), and a placeholder for API Connect app overlays.

## Why it's built this way

- **Separation of duties.** The *factories* (`mq-infra`, `ace-infra`) build and test images/BARs; this *apps* repo holds only desired state. Neither can silently mutate the cluster — the operator does, from what's committed here.
- **Promotion as a pull request.** Environments are directories, so promoting dev→staging→prod is a reviewable diff, not a console click. You get change control, approvals, and one-command rollback for free.
- **GitOps auditability.** Git is the single source of truth; Argo CD self-heals drift and prunes what's removed. What's in the cluster is always exactly what's in `master`.
- **Reproducible from scratch.** Because even the "supporting" manifests (backends, BAR host, RBAC) live in git, the whole demo can be rebuilt on a clean cluster instead of relying on hand-applied artifacts.
- **Current CR model.** ACE uses `IntegrationRuntime` and MQ uses the `v1beta1` `QueueManager` — the 2026 CP4I 16.x shapes, not the deprecated `IntegrationServer`/older APIs.

## How it fits the bigger picture

This is repo 4 of the multi-tenancy GitOps split, all under [github.com/ibmclientengineering](https://github.com/ibmclientengineering):

1. **multi-tenancy-gitops** — the bootstrap; Argo CD points at the others.
2. **multi-tenancy-gitops-infra** — cluster infrastructure (namespaces, storage, RBAC).
3. **multi-tenancy-gitops-services** — operators and shared services (the CP4I operators that own these CRs).
4. **multi-tenancy-gitops-apps** — *this repo*, the application workloads.

The delivery loop closes with the factory repos: **mq-infra** and **ace-infra** run the Tekton pipelines that build and validate the MQ image and ACE BAR, then open PRs that stamp the new artifact into the `mq/environments/**` and `ace/environments/**` overlays here. Argo CD sees the merge and rolls it out.

Maintained by IBM Client Engineering.
