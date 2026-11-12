<div align="center">

# ☁️ Namazu Cloud Addons

**Per-tenant infrastructure, installed by PR, operated by Namazu Studios.**

[![License: AGPL v3](https://img.shields.io/badge/license-AGPL--3.0-blue.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/platform-Namazu_Cloud-4c1)](https://namazustudios.com/docs/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-EKS-326ce5)](https://kubernetes.io)
[![Terraform](https://img.shields.io/badge/IaC-Terraform-7b42bc)](https://www.terraform.io)

**Generative AI infrastructure for multi-tenant platforms — GPU pools that scale from zero to your cap, provisioned storage that kills cold starts, in-region model registries — one pull request per tenant.**

[The addon catalog](#-the-official-addon-catalog) ·
[Bring your own addon](#-bring-your-own-add-on) ·
[How it works](#-how-it-works) ·
[The process](#-the-process-terms-and-shared-responsibility)

</div>

---

## What is this?

**Namazu Cloud Addons** turn a shared multi-tenant Kubernetes cluster into a
platform your customers extend safely. An *addon* is Terraform that Namazu
Studios installs into a tenant's namespace under a **least-privilege
executor identity** — never touching the tenant's baseline deployment,
never touching another tenant's anything. The install manifest is a pull
request. The uninstall is the PR that removes the line. The sweep
guarantees it.

| You get | Without addons |
|---|---|
| One PR = install; removing the PR = guaranteed teardown | Hand-run terraform against shared clusters |
| Per-tenant state, per-tenant config, per-tenant billing labels | Shared tooling nobody dares to change |
| Least-privilege executor, org-qualified IAM, swept state keys | God-mode credentials |
| Code updates never move permissions | Permission drift |

## ⚡ Why it's powerful

- **Elastic by construction.** GPU pools are *ceilings, not reservations*:
  scale on demand from zero to your per-tenant cap, drain to zero when the
  batch is done. Empty pools cost zero.
- **Cold starts that don't hurt.** Provisioned per-namespace storage
  (gp3 up to **16,000 IOPS + 1,000 MiB/s**) loads multi-GB AI model weights
  and LoRA checkpoints from snapshots in **minutes, not half an hour**.
- **Reproducible AI images.** Your inference and worker images (5-8 GB
  CUDA builds) pinned by digest, in-region, pulled by every freshly
  provisioned GPU node in seconds.
- **Isolation you can audit.** Tenant-named state keys, org-qualified IAM
  names, namespace-guarded storage classes, per-namespace pull policies.
- **Operated, not babysat.** A scheduled sweep destroys anything the
  reviewed install list no longer names. Orphans cannot survive.

## 📦 The official addon catalog

Each one is a public repo — read every line before you buy:

| Addon | What your tenants get |
|---|---|
| **[gpu](https://github.com/NamazuStudios/namazu-cloud-addon-gpu)** | Dedicated Karpenter **GPU node pools** per tenant (g4dn/g5/g6/g7) — diffusion, LLM, and batch AI jobs scale on demand from zero to the tenant's cap, with spot/on-demand capacity types and self-reclaiming 4-GPU `-xl` pools for bursty renders |
| **[fast-storage-class](https://github.com/NamazuStudios/namazu-cloud-addon-fast-storage-class)** | A provisioned **fast EBS StorageClass per namespace** — 16,000 IOPS + 1,000 MiB/s for GenAI cold starts: model weights and LoRA checkpoints restore from snapshot in minutes, namespace-guarded so one tenant can't borrow another's class |
| **[organization-ecr-repository](https://github.com/NamazuStudios/namazu-cloud-addon-organization-ecr-repository)** | An **org-namespaced ECR repository** — your inference and worker images pinned by IMMUTABLE digest, in-region for fast node pulls, per-namespace pull control, OIDC-trusted CI push roles |
| **[agent-apn](https://github.com/NamazuStudios/namazu-cloud-addon-agent-apn)** | **AWS Partner Central wiring** for the co-selling agent: outbound assume on the tenant's agent role, optional co-sell opportunity-manager role, ECR pull |

Official addons activate for a **fixed fee**.

## 🛠 Bring your own add-on

Need something the catalog doesn't cover? The
**[example template](https://github.com/NamazuStudios/namazu-cloud-addon-example)**
(MIT — what you build is yours) is the starting point:

1. **Make a private clone** and write the Terraform you wish to install,
   plus its `manifest.yaml` input declaration.
2. **Open a paid support ticket** with Namazu Studios. **Each support
   ticket includes five (5) security audits and reviews.**
3. **Once approved, we install it to your namespaces at your request.**

Live maintenance is pull requests: **unlimited reviews, within reason,**
as long as the permissions boundary holds — boundary changes start a new
support cycle. Namazu Studios can also **author a custom addon for you,
for a fee scoped to the work required**. Full terms in
[ADDONS.md](https://github.com/NamazuStudios/namazu-cloud-addon-example/blob/main/ADDONS.md).

## ⚙️ How it works

```
merge to main (the PR IS the install manifest)
   │
   ├─► prepare-index   validates every entry (variant, pinned refs, no duplicates)
   │
   ├─► apply           one (addon, variant, tenant) triple per matrix entry:
   │                     clone @ pinned ref → resolve bindings from statefiles
   │                     → terraform apply on addon_<id>_<variant>.tfstate
   │                     → stamp namespace annotations
   │
   └─► sweep           destroys anything the review no longer names
```

The enforcer is the
**[namazu-cloud-addon-runner](https://github.com/NamazuStudios/namazu-cloud-addon-runner)**
(AGPL-3.0): shared **code, not shared authority** — the OIDC trust belongs
to the calling org's fork; the action can grant nothing. Org forks stay
**private** (they carry tenant ids and org-scoped IAM policies); the
generic capability is public and auditable.

## 🧠 Built for generative AI platforms

The pattern this system was extracted from: a multi-tenant GenAI platform
where every customer gets their own **GPU pool** (quota-capped,
batch-elastic), their own **provisioned weights volume** (snapshot →
serving in minutes), and their own **pinned inference image** — turned on
by a pull request and billed by attribution labels.

Whether your tenants run **diffusion**, **LLM serving**, **LoRA
fine-tuning**, or **batch video/music generation**, the addons give each
one dedicated capacity without giving each one a cluster.

## 📚 Documentation

- The manual: [namazustudios.com/docs — Custom Addons](https://namazustudios.com/docs/)
- The process + terms: [ADDONS.md](https://github.com/NamazuStudios/namazu-cloud-addon-example/blob/main/ADDONS.md)
- The design of record: the Namazu Cloud Addon System notebook in the
  Namazu Cloud portal

## License & provenance

**AGPL-3.0** — see [LICENSE](LICENSE). This overview repository is made
public for **transparency and visibility**: the addon system is deeply
connected to Namazu Cloud. Written for Namazu Studios and Namazu Cloud
use; derivative works are expected to come back upstream to
`NamazuStudios`, no exceptions.

**AGPL waiver for delivered work**: Namazu Studios, as the copyright
holder, **waives all AGPL obligations for software installed through a
purchase or other business arrangement between Namazu Studios and the
client** — the license terms govern the code as published; work delivered
under contract is delivered under that agreement.
