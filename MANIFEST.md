# manifest.yaml — the addon input declaration

Every addon repository carries a `manifest.yaml` at its root. It is the
contract between the addon and the
[namazu-cloud-addon-runner](https://github.com/NamazuStudios/namazu-cloud-addon-runner):
what the addon is, who may consume it, and how its Terraform variables get
their values.

```yaml
description: >-
  Opt-in GPU node capacity: per-tenant Karpenter GPU tiers with quotas
  and capacity types. Everything gpu-specific resolves from the entry.
visibility: shared
display: GPU Node Capacity
input:
  # Generic facts only — things that exist regardless of addons:
  karpenter_node_role_name: $instance.karpenter_node_role_name
```

## Root fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `description` | Yes | string | Human-readable description. Used in logs, audit reports, and the namespace annotation (`addon.cloud.namazustudios.com/description-<id>_<variant>`). |
| `display` | No | string | A human-readable addon name overriding the addon id in the namespace annotations. |
| `visibility` | No | `shared` \| `private` | Who may consume the addon. Default: `shared` (any org fork). `private` restricts consumption to the org whose database id matches `owner_org`, enforced at lint and at apply time. |
| `owner_org` | if `private` | string | The owning organization's database unique id. |
| `input` | No | map | Input bindings: Terraform variable name → binding expression. |

## Binding expressions

The `input` map connects the addon's Terraform variables to values. Four
forms:

| Type | Syntax | Description |
|------|--------|-------------|
| Literal | `"value"`, `123`, `true`, `{...}` | A static value, passed as written. |
| Instance reference | `$instance.<output>[.<key>...]` | Read from the tenant baseline's outputs (generic platform facts: the cluster's Karpenter node role, the agent's IRSA role). Nested keys traverse objects and lists. |
| Addon reference | `$addon.<id>.<variant>.<output>...` | Read from another addon's statefile for this tenant. The variant names WHICH install — every install is the triple (addon, variant, tenant). Missing installs fail loudly naming the ref. |
| Sensitive override | `!$instance.<output>` | The `!` prefix acknowledges a sensitive output: the value rides `TF_VAR_<name>` environment only — never on disk, never logged. A `$` reference to a sensitive output is a hard error. |

Anything that does not start with a recognized unit token is a literal;
there is no escaping by design.

### Binding precedence

1. **Manifest binding** — highest priority, overrides everything.
2. **`--contract`** — the consuming fork's matrix entry (`contract:` lines
   → the runner's `contract` input). The per-tenant configuration channel:
   tier quotas, capacity types, repository names.
3. **Terraform variable default** — lowest priority.

A variable with no default, no binding, and no contract value fails the
plan with a clear, named error.

## How the runner uses it

At apply time the runner:

1. Reads `manifest.yaml` from the addon repo root and validates it.
2. Resolves each binding against statefiles (the tenant baseline's, and
   any referenced addon install's).
3. Emits `terraform.tfvars.json` (0600, deleted after the run) for
   non-sensitive values; sensitive values ride `TF_VAR_*` only.
4. Runs `terraform init` on the bijection backend key
   (`addon_<id>_<variant>.tfstate`) and applies.

## Author rules

1. **Never declare the contract variables.** `instance_id`,
   `workspace_id`, `organization_id`, `billable`, `state`, `product_sku`,
   `app_dns_name`, `architecture`, `tags`, `eks_cluster_name`,
   `eks_cluster_role_arn` are injected by the runner automatically.
2. **Never couple to another addon except through refs.** If addon B needs
   data from addon A, bind `$addon.A.<variant>.<output>` — never hardcode
   assumptions.
3. **Default-less variables must be bound.** Manifest bindings and
   `--contract` values are the only sources.
4. **One binding per variable.** Each key in `input` is a Terraform
   variable name, bound exactly once.
5. **Bind only generic facts from the baseline.** The tenant baseline
   carries zero addon knowledge — addon-specific configuration rides the
   consuming fork's matrix entry via `--contract`.

## Error surface

| Error | Cause |
|-------|-------|
| `manifest.yaml missing at addon repo root` | The repo has no manifest. |
| `Unbound variable '<name>'` | No default, no binding, no contract value. |
| `output '<name>' does not exist in the referenced statefile` | The baseline (or addon install) doesn't output the referenced key. |
| `... whose statefile is not available on this tenant` | A `$addon` reference to an install that isn't installed — install it first or correct the variant. |
| `output '<name>' is SENSITIVE — a '$' reference is rejected` | Sensitive output without the `!` acknowledgment. |
| `visibility violation: private to org '<owner_org>'` | A private addon consumed by a fork with a different `organization_id`. |
