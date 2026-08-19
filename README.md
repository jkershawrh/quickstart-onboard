# Quickstart Onboard

Take a factory-standard quickstart and produce a complete, ready-to-deploy RHDP lab. One command, full pipeline. Zero external dependencies.

## What it does

You point `/onboard` at a quickstart repo. It runs 7 phases and produces everything needed for RHDP submission:

### Phase 1: SCAN (inline — no external tools)
- **Capacity scan**: greps for LLM framework imports, model references, local vs MaaS inference, infrastructure components, K8s resource requests
- **Security scan**: 7 check categories — secrets, config, containers, dangerous code, infrastructure exposure, supply chain CVEs
- Recommends tier (pilot/partner/dedicated) and cluster size
- Assigns the **smallest MaaS model that works** for the use case
- **MaaS overprovisioning fix**: when the model runs on shared MaaS, clusters are sized for the app only — not the model

### Phase 2: REMEDIATE
- Classifies security findings as auto-fixable, needs-human, or false-positive
- Proposes and applies fixes for auto-fixable findings (secrets -> env vars, missing .gitignore, unpinned tags)
- Re-scans to verify grade improved

### Phase 3: AGNOSTICV CATALOG
- Generates a **directly submittable** AgnosticV catalog:
  - `common.yaml` — full schema with `__meta__.components`, cluster pool refs, workload ordering (authentication -> LiteLLM -> app -> showroom), deployer EE image, asset UUID, `reportingLabels`
  - `dev.yaml` — dev environment overrides (shorter TTL, single user)
  - `description.adoc` — catalog UI description (business-first)
  - `info-message-template.adoc` — post-provision access info
- Runs 14-point validation checklist matching AgV submission requirements

### Phase 4: SHOWROOM LAB CONTENT
- Produces a **complete Showroom lab with real content** — not placeholder stubs
- Every module follows the **"See, Learn, Do"** structure:
  - **See**: Architecture explanation before commands (3-5 paragraphs)
  - **Learn**: Trade-offs, design decisions, why this approach
  - **Do**: Hands-on exercise with verification after every step
- Addresses the "too light" feedback: no naked commands, scenario framing, key takeaways
- Proper AsciiDoc: `role="execute"` for terminal commands, `subs="attributes+"` for RHDP variables
- Minimum 5 modules, 50 lines per module, progressive complexity

### Phase 5: VERIFY
- 20+ quality checks across 4 categories:
  - **S.\***: Structure (site.yml, ui-config, antora.yml, nav, supplemental-ui)
  - **D.\***: Content depth (scenario framing, See section, key takeaways, minimum length)
  - **E.\***: AsciiDoc formatting (role="execute", attributes, heading nesting, no secrets)
  - **B.\***: Cross-module consistency (progressive build, terminology, acronyms)
- Antora build verification

### Phase 6: LAB GRADING (optional)
- FTL solve/validate Ansible playbooks for E2E testing
- Requires live environment (OCP cluster)

### Phase 7: PUBLISH
- Local file output or PR with full scan + content report
- Launchpad callback for provisioning intelligence

## Output

Two artifacts ready to push:

| Artifact | Purpose |
|----------|---------|
| `showroom-<name>/` | Complete Showroom lab repo (Antora content, ui-config, supplemental-ui, Intel branding) |
| AgnosticV catalog files | `common.yaml`, `dev.yaml`, `description.adoc`, `info-message-template.adoc` — drop into AgV repo and submit |

## Dependencies

**None.** All scanning (capacity + security), content generation, and validation run inline. No NovaScan, DarkScope, or RHDP marketplace plugins required.

## Usage as a Claude Code skill

Copy `onboard.md` to your `.claude/commands/` directory:

```bash
mkdir -p ~/.claude/commands
cp onboard.md ~/.claude/commands/onboard.md
```

Then invoke:

```
/onboard https://github.com/rh-ai-quickstart/edge-ai-cpu-inference
/onboard ~/Documents/my-quickstart
/onboard .
```

## Pipeline position

**repo** --`/intake`--> **quickstart** --`/onboard`--> **ready-made lab on RHDP**

| Skill | What it does | Repo |
|-------|-------------|------|
| `/intake` | Repo to factory-standard quickstart | [quickstart-intake](https://github.com/jkershawrh/quickstart-intake) |
| `/onboard` | Quickstart to ready-made RHDP lab | [quickstart-onboard](https://github.com/jkershawrh/quickstart-onboard) |

## Key design decisions

- **MaaS = small cluster.** When the model runs on MaaS (shared RACM infrastructure), the user's cluster only needs resources for the application. Previous versions overprovisioned by sizing clusters as if the model deployed locally.
- **Smallest model that works.** Start with granite-2b-cpu for chat/QA, only upgrade to larger models when accuracy or function calling requires it.
- **See, Learn, Do.** Every lab module explains architecture before commands, trade-offs before steps. Addresses the "showrooms are too light" feedback.
- **Self-contained.** All scanning logic from NovaScan (capacity) and DarkScope (security) is inlined as grep patterns and lookup tables. No external tool dependencies.
