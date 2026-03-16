# Design - Taxonomy

> **Last Updated**: 2026-03-16 by Keming He

Skill taxonomy, composition model, and priority order for the iac-adoption-skills repo.

> [!NOTE]
>
> Related design docs: [Design - Decisions](./design-decisions.md), [Design - Parked](./design-parked.md)

---

## Skill Categories

| Prefix | Category | Purpose |
| :--- | :--- | :--- |
| `orchestrator-` | Orchestrator | Generic workflow logic with self-discovery; sequences skills via frontmatter; carries cross-cutting references |
| `workflow-` | Workflow | Executes a specific step in the adoption process; independently invocable; produces artifacts for downstream consumption |
| `research-` | Research | Produces and maintains domain knowledge; dual-mode (operate + self-update via `SELF_UPDATE.md`); serves as reference provider to other skills via frontmatter search |
| `meta-` | Meta | Creates artifacts used by other skills; tooling for skill maintenance |

---

## Skill List

### `orchestrator-iac-adoption`

Generic workflow orchestrator for IaC adoption. Provides sequencing logic, decision points, and user context management. Does not maintain a static skill catalog - discovers available skills via platform-loaded frontmatter at runtime.

**References** (cross-cutting, always loaded first):

- `references/command-safety-matrix.md` - Read-only vs write/modify/delete command classification
- `references/file-organization-patterns.md` - Component naming, file splitting, comment separators
- `references/tagging-strategy.md` - Minimum viable tagging schema for IaC adoption
- `references/dual-responsibility-framework.md` - User vs agent responsibilities

**Parallelization guidance**: Orchestrator identifies parallelization opportunities during dispatch (e.g., independent resource groups, multi-cloud simultaneous research).

### `workflow-adoption-readiness`

User requirements gathering and setup validation. The bridge between orchestrator dispatch and the first discovery step.

**Requirements gathering**: Cloud platform(s), resource scope, adoption goals, known constraints (e.g., "we have EKS" informs discovery to handle EKS-managed EC2 intelligently).

**Setup validation**: Auth, CLI versions, backend config, `init`/`validate` status. Read-only verification.

**Output**: Structured context document consumed by all downstream workflow skills.

**Parallelization**: Validate terraform/tofu setup and cloud CLI auth in parallel.

### `workflow-iac-discovery`

Discover cloud resources via read-only CLI. Build dependency map for the target resource group. Generate structured discovery report (stakeholder-shareable).

**Searches for**: Cloud dependency mapping research (via frontmatter) to determine iteration ordering. Falls back to user-guided ordering if research skill is unavailable.

**Output**: Inventory list, dependency graph, discovery report document.

**Parallelization**: Per-service discovery queries in parallel within a resource group.

### `workflow-iac-capture`

Generate `import-[component].tf` files with import blocks. Run config generation (`plan -generate-config-out=temp-generated.tf`). Clean up generated config - resolve errors, split into `[component].tf` files, format. Result: hard-coded but authentic and plan-clean `.tf` files.

**Searches for**: Soft delete policies (via frontmatter) when destroy/reimport is relevant. Falls back to web search or user guidance.

**Active user check-in**: Before proceeding to optimize, present carrot (streamlined IaC) and stick (complex state migration later). Heuristic: recommend skipping optimize for simple groups.

**Parallelization**: Per-component config cleanup in parallel.

### `workflow-iac-optimize`

Transform captured config into production-grade IaC. Parameterize with variables, add `for_each`/`count`/dynamic blocks, extract shared values to locals, modularize where appropriate, add tagging, set up `lifecycle` rules.

**Result**: Portable, multi-env-capable, plan-clean `.tf` files.

**Parallelization**: Per-component optimization in parallel.

### `research-cloud-dependency-mapping`

Research and document per-cloud resource dependency relationships for IaC adoption iteration ordering. Each cloud researched independently with cloud-appropriate structure - no fixed layer model forced across clouds.

**References**: `references/aws.md`, `references/gcp.md`, `references/azure.md`

**Parallelization**: 3 parallel research agents, one per cloud.

### `research-cloud-cli-reference`

Research and document read-only discovery CLI commands per cloud platform. The "how to query the cloud" knowledge that workflow skills need.

**References**: `references/aws.md`, `references/gcp.md`, `references/azure.md`

**Parallelization**: 3 parallel research agents, one per cloud.

### `research-soft-delete-policies`

Research cloud resource soft delete policies for Terraform/OpenTofu planning. Existing skill, needs restructure to fit taxonomy (per-cloud references, `SELF_UPDATE.md`, single-cloud template).

**References**: `references/aws.md`, `references/gcp.md`, `references/azure.md`

**Parallelization**: 3 parallel research agents, one per cloud (already designed this way).

### `research-shadow-it-market`

Research market data on unmanaged cloud resources, compliance drivers (SOC2, HIPAA, PCI), cost of manual IaC adoption, M&A scenarios. Stakeholder-ready output for promoting the repo and justifying IaC adoption projects.

**Parallelization**: Parallel research agents per topic area.

### `meta-self-update-setup`

Takes a target research skill as input. Analyzes what data the skill maintains. Generates a `references/SELF_UPDATE.md` with staleness indicators, refresh procedures, and validation steps. Ensures consistent self-update patterns across all research skills.

---

## Composition Model

```plaintext
Platform loads all skill frontmatter into agent context
         |
User --> orchestrator-iac-adoption
           |
           |-- cross-cutting refs always available (command safety, file patterns, etc.)
           |-- discovers available skills via frontmatter
           |-- captures user context, dispatches workflow:
           |
           |   1. workflow-adoption-readiness
           |      (validates setup, gathers user context, produces context doc)
           |      [parallel: cloud auth + IaC tool validation simultaneously]
           |
           |   Per iteration (one resource group at a time):
           |   2. workflow-iac-discovery
           |      (searches for research-cloud-dependency-mapping via frontmatter)
           |      [parallel: per-service discovery queries]
           |   3. workflow-iac-capture
           |      (searches for research-soft-delete-policies via frontmatter)
           |      [parallel: per-component config cleanup]
           |      --> active user check-in: proceed to optimize?
           |   4. workflow-iac-optimize (default, skippable)
           |      [parallel: per-component optimization]
           |
           |-- research skills: invoked on-demand or for self-update
           |-- all cross-skill refs resolved via frontmatter search, never filesystem paths
```

---

## Priority Order

| Priority | Skill | Rationale |
| :--- | :--- | :--- |
| P0 | `orchestrator-iac-adoption` | Entry point; carries cross-cutting references; defines the system |
| P0 | `workflow-adoption-readiness` | Bridge to first workflow step; produces context for all downstream skills |
| P0 | `research-cloud-dependency-mapping` | Strongly recommended dependency for discovery iteration ordering |
| P0 | `workflow-iac-discovery` | First iteration step; everything depends on good discovery |
| P1 | `workflow-iac-capture` | Core value delivery - resources under management |
| P1 | `workflow-iac-optimize` | Full value delivery - production-grade IaC |
| P1 | `research-soft-delete-policies` (restructure) | Exists but needs structural refactor to fit taxonomy |
| P2 | `research-cloud-cli-reference` | Enhances discovery; can build incrementally |
| P2 | `meta-self-update-setup` | Needed once multiple research skills exist |
| P3 | `research-shadow-it-market` | Advocacy/marketing, not core functionality |

---

## Open Design Questions

- What does `workflow-adoption-readiness` output artifact look like? Structured markdown? Key-value context? How do downstream skills consume it?
- Should `research-cloud-dependency-mapping` output adapt structure per cloud (different docs have different shapes), or should it follow a loose common template with GDC?
- Functional decomposition: preliminary table in [Design - Parked](design-parked.md) (section 1); needs validation during skill drafting
