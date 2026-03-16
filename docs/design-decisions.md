# Design - Decisions

> **Last Updated**: 2026-03-16 by Keming He

Consolidated design decisions for the iac-adoption-skills repo. Final answers only - no intermediate reasoning or superseded positions. Distilled from initial brain dump, design response, AWS dependency discovery, and skills breakdown sessions.

> [!NOTE]
>
> Related design docs: [Design - Taxonomy](./design-taxonomy.md), [Design - Parked](./design-parked.md)

---

## Top-Level Design Principles

These principles apply across all skills in this repo. Every skill's SKILL.md should reflect them.

### Parallelization-First Design

Every skill must be designed with parallelization in mind. Call out which steps are parallelizable in each SKILL.md.

- **Research skills**: Parallel agents per cloud (3 clouds = 3 agents minimum)
- **Workflow skills**: Parallel sub-agents for independent resource groups within an iteration
- **Cross-step**: Independent research refreshes can run in parallel with workflow execution

### No Required Dependencies, Graceful Degradation

No skill declares another skill as "required." Every skill works alone - just better with supporting skills. "Strongly recommended" signals importance without creating hard coupling. Enables independent skill testing.

**Auto-search fallback chain** (applies to all skills):

1. Search local skills/MCP/plugins/user docs automatically
2. If insufficient, ask user for references
3. Use web search and parallelize across agents

### Self-Discovery Over Static Catalogs

The platform loads all skill frontmatter into agent context. No skill maintains a catalog of other skills. Adding a new skill requires no updates to existing skills.

### Loose Coupling Via Frontmatter Search

Skills declare what knowledge they need without specifying where to find it. The agent resolves dependencies at runtime by searching available frontmatter. If a dependency is unavailable, the skill continues with degraded but functional behavior.

### Read-Only by Default

Safety model foundation. Write/modify/delete commands require explicit user allowlisting. The orchestrator's `references/command-safety-matrix.md` classifies all commands.

### Semi-Auto Heuristic

Pause for business/architectural decisions. Proceed autonomously for mechanical/deterministic operations (fmt, validate, error resolution in generated config).

---

## Settled Decisions

### Scope and Licensing

| Decision | Rationale |
| :--- | :--- |
| Separate repo (`iac-adoption-skills`) | Audience divergence from common-devx; independent iteration cadence; standalone discoverability |
| Apache 2.0, per-skill LICENSE file | Enterprise-friendly patent grant; aligns with HashiCorp ecosystem |
| Terraform + OpenTofu only (no Pulumi/SST) | Shared HCL, near-identical CLIs, manageable scope |
| Big 3 clouds only (AWS, GCP, Azure) | Covers vast majority of use cases |

### Naming Conventions

| Decision | Rationale |
| :--- | :--- |
| Category prefix on skill directories | `orchestrator-*`, `workflow-*`, `research-*`, `meta-*` for human readability and repo file clarity |
| No step-numbering in names | Numbering creates brittle coupling; ordering lives in orchestrator logic |
| Reference files: plain cloud/tool names | `aws.md`, `gcp.md`, `azure.md`, `terraform.md`, `opentofu.md` in `references/` |
| Uppercase meta files in `references/` | `SELF_UPDATE.md` (uppercase) distinguishes structural files from content files, consistent with `README.md`, `LICENSE`, `SKILL.md` convention |
| `-creation` suffix for meta skills | Follows common-devx convention (e.g., `readme-creation`, `commit-message-creation`) |

### Workflow Architecture

| Decision | Rationale |
| :--- | :--- |
| Iterative per-resource-group | Each resource group goes through full cycle; interruptable and resumable |
| Dynamic iteration ordering per cloud | No fixed vendor-neutral layer model; each cloud's dependency mapping research determines its own iteration order |
| Mode naming: Capture and Optimize | Replaces "tier 1" / "tier 2"; modes within each iteration, not project phases |
| Optimize-by-default per iteration | Default proposes optimize (carrot: streamlined IaC; stick: complex state migration later); user can skip with explicit warning |
| Active user check-in before optimize | Heuristic: if capture output is simple (few resources, no inter-resource references), recommend skipping |
| Avoid `moved` blocks by design | Optimize before first apply eliminates state migration need in the default path |
| Discovery report as deliverable | Structured stakeholder-shareable report between discovery and import |
| Provider version pinning | Captured during readiness validation, pinned in `required_providers` |

### Skill System Architecture

| Decision | Rationale |
| :--- | :--- |
| Generic orchestrator with self-discovery | SKILL.md provides workflow logic only; frontmatter-based discovery at runtime; no static catalog |
| Cross-cutting refs in orchestrator `references/` | Command safety matrix, file organization patterns, tagging strategy, dual responsibility framework all live in the orchestrator; always loaded first |
| Orchestrator SKILL.md is generic, references are domain-specific | SKILL.md does not change when references are updated; clean separation |
| Research skills: dual-mode (operate + self-update) | `SELF_UPDATE.md` documents refresh procedures and staleness indicators; on-demand triggers only |
| Cloud-decoupled research | Per-cloud reference files; each cloud researched independently with cloud-appropriate structure |
| Dependency mapping and CLI reference are separate research skills | Two lightweight skills beats one overloaded skill |
| Root README mirrors orchestrator for humans | README is human-facing entry point; orchestrator is agent-facing entry point; parallel structure, no content duplication |

### Post-Adoption Recommendations (Not Core Workflow)

| Decision | Rationale |
| :--- | :--- |
| Drift detection (scheduled `terraform plan`) | Document as "what comes next," not implemented by skill |
| Cost visibility (infracost) | Recommendation only |
| Pre-commit/CI tooling (trivy, tflint, terraform-docs) | Recommendation only |

---

## Remaining Open Items

### Functional decomposition of Terraform/OpenTofu operations

Preliminary decomposition table exists in [Design - Parked](design-parked.md) (section 1) with initial assessment of where tool knowledge should live. Needs validation when drafting workflow skill SKILL.md files.

### `workflow-adoption-readiness` output artifact shape

This skill produces a user context document consumed by downstream workflow skills. The structure and handoff mechanism need definition. What does this document contain? How do downstream skills reference it?
