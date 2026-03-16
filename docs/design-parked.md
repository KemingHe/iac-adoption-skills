# Design - Parked Details

> **Last Updated**: 2026-03-16 by Keming He

Implementation-level details that are too granular for current planning but must not be lost. Each section includes a "revisit when" marker. Resolved items are removed as they are absorbed into skill artifacts.

> [!NOTE]
>
> Related design docs: [Design - Decisions](./design-decisions.md), [Design - Taxonomy](./design-taxonomy.md)

---

## 1. Functional Decomposition: IaC Tool Operations Per Skill

Maps which Terraform/OpenTofu operations each workflow skill needs. Informs where tool knowledge should live.

**Note on TF/Tofu redundancy**: Commands are nearly identical between Terraform and OpenTofu. The table uses `terraform` as the canonical reference. Each skill should note OpenTofu compatibility (substitute `tofu` for `terraform`). The `workflow-adoption-readiness` skill determines which tool is in use and sets context for downstream skills.

| Skill | IaC Tool Operations | Cloud CLI Operations | Parallelizable |
| :--- | :--- | :--- | :--- |
| `workflow-adoption-readiness` | `terraform version`, `terraform init`, `terraform validate`, `terraform fmt -check` | `aws sts get-caller-identity`, `gcloud auth list`, `az account show` (auth verification per cloud) | Tool validation and cloud auth checks in parallel |
| `workflow-iac-discovery` | None directly | `aws describe-*`, `aws list-*`, `gcloud * list`, `az * list` (read-only per-service queries) | Per-service discovery queries in parallel |
| `workflow-iac-capture` | `terraform plan -generate-config-out=temp-generated.tf`, `terraform fmt`, `terraform validate`, `terraform plan` (verify clean) | None (uses TF plan output, not cloud CLIs) | Per-component config cleanup in parallel |
| `workflow-iac-optimize` | `terraform fmt`, `terraform validate`, `terraform plan` (verify clean), potentially `terraform test` | None | Per-component optimization in parallel |

**Observations**:

- `workflow-iac-discovery` is the only skill that uses no IaC tool operations - it is purely cloud CLI driven
- `workflow-iac-capture` and `workflow-iac-optimize` share the same IaC verification cycle (`fmt`, `validate`, `plan`)
- The shared verification cycle could be documented once in the orchestrator's references and referenced by both skills
- Cloud CLI operations are concentrated in two skills: readiness (auth checks) and discovery (resource queries)
- `terraform plan -generate-config-out` is unique to capture and is the most complex IaC operation in the workflow

**Where tool knowledge should live** (preliminary assessment):

- Common IaC verification cycle (`fmt`, `validate`, `plan`): Orchestrator `references/` - shared across workflow skills
- Cloud CLI discovery commands: `research-cloud-cli-reference` skill - per-cloud, independently updatable
- Cloud CLI auth commands: `workflow-adoption-readiness` - small set, stable, inline in skill
- `plan -generate-config-out` specifics: `workflow-iac-capture` - unique to this skill, inline

**Revisit when**: Drafting workflow skill SKILL.md files.

---

## 2. Constants and Large Data Escalation Path

**Agreed escalation order**:

1. JSON/YAML data files via `file()` + `jsondecode()`/`yamldecode()` for large static data
2. `[...]-locals.tf` split for the moderate case
3. Constants module (`modules/[...]-constants/*.tf`) only for data shared across multiple root modules

**Revisit when**: Building `workflow-iac-capture` or `workflow-iac-optimize`.

---

## 3. File Organization Patterns

**Key patterns** (to codify in `orchestrator-iac-adoption/references/file-organization-patterns.md`):

- Component naming: `[component].tf`, `[component]-[subcomponent].tf`, max 3 levels
- Locals split: `[component].tf` alongside `[component]-locals.tf` when locals are large
- File split threshold: approximately 200 lines or 3+ logically distinct resource groups (needs validation)
- Comment separator: `#` followed by space and 76 dashes (total 78 chars), then section label, then another separator
- Example `.tf` asset files to illustrate

**Revisit when**: Building `workflow-iac-capture` (generates `.tf` files). Will live in `orchestrator-iac-adoption/references/`.

---

## 4. Soft Delete Skill Restructure

**Current structure**: Monolithic template in `assets/`, completed research in `docs/soft-delete-policies-research.md`.

**Proposed structure** (with naming convention):

```plaintext
research-soft-delete-policies/
  SKILL.md                    # Dual-mode: operate + self-update
  README.md
  LICENSE
  assets/
    single-cloud-template.md  # Template for one cloud's research
  references/
    aws.md                    # Completed AWS research
    gcp.md                    # Completed GCP research
    azure.md                  # Completed Azure research
    SELF_UPDATE.md            # Refresh triggers and procedures
```

**Key changes**:

- Rename directory from `soft-delete-policies-research/` to `research-soft-delete-policies/`
- Research output inside skill (self-contained), not in `docs/`
- Per-cloud split with standardized filenames
- `SELF_UPDATE.md` uppercase for structural distinction
- `docs/soft-delete-policies-research.md` content absorbed into per-cloud references; cross-cloud comparison can live in SKILL.md summary or as a generated artifact

**Revisit when**: P1 priority - after P0 skills are in place.

---

## 5. Dual Responsibility Framework

**User responsibilities**: Cloud auth configured, IaC tool initialized, backend configured, basic resource knowledge.

**Agent responsibilities**: Read-only recon, explain commands before executing, pause at decision points, handle mechanical ops autonomously.

Will be codified in `orchestrator-iac-adoption/references/dual-responsibility-framework.md`.

**Revisit when**: Building the orchestrator skill.

---

## 6. Sensitive Values in Generated Config

Imported resources may contain secrets as plaintext in generated config. Must detect, flag for user, recommend `sensitive = true` or vault/SSM integration.

**Revisit when**: Building `workflow-iac-capture`.

---

## 7. Lifecycle Rules for Imported Resources

Imported resources should get `lifecycle { prevent_destroy = true }` during adoption. Some may need `ignore_changes` for externally-managed attributes.

**Revisit when**: Building `workflow-iac-capture`.

---

## 8. Import Block vs Import Command Decision

Modern TF/Tofu uses `import` blocks (declarative, CI/CD-friendly). Older versions need `terraform import` CLI command. Skill must detect version and fall back gracefully.

**Revisit when**: Building `workflow-iac-capture` or `workflow-adoption-readiness` (version detection).

---

## 9. Root README Planning

Root `README.md` is the human-facing entry point, mirroring `orchestrator-iac-adoption` (agent-facing entry point). Structure should parallel but not duplicate. User has a `/readme-creation` skill for generation.

**Key sections**: What and who, skill taxonomy, how to use, how skills compose, contributing, license.

**Revisit when**: Taxonomy finalized and P0 skills exist.
