# Quickstart Onboard

Take a factory-standard quickstart and produce a complete, ready-to-deploy RHDP lab. One command, full pipeline.

## What it does

You point `/onboard` at a quickstart repo. It runs 7 phases and produces everything needed for RHDP submission:

### Phase 1: SCAN
- Runs **NovaScan** (capacity planning, tier recommendation) and **DarkScope** (security scanning) in parallel
- Recommends tier (pilot/partner/dedicated), resource quotas, and MaaS model assignment
- Reports security grade (A-F) with finding counts

### Phase 2: REMEDIATE
- Classifies each security finding as auto-fixable, needs-human, or false-positive
- Proposes and applies code-level fixes for auto-fixable findings
- Re-scans to verify grade improved

### Phase 3: AGNOSTICV CATALOG
- Generates a **directly submittable** AgnosticV catalog:
  - `common.yaml` — full schema with `__meta__.components`, cluster pool refs, workload ordering (authentication -> LiteLLM -> app -> showroom), deployer EE image, asset UUID
  - `dev.yaml` — dev environment overrides (shorter TTL, single user)
  - `description.adoc` — catalog UI description (business-first)
  - `info-message-template.adoc` — post-provision access info
- Runs 14-point validation checklist matching AgnosticV submission requirements

### Phase 4: SHOWROOM LAB CONTENT
- Produces a **complete Showroom lab with real content** — not placeholder stubs
- Every module follows the **"See, Learn, Do"** structure:
  - **See**: Architecture explanation before commands (3-5 paragraphs)
  - **Learn**: Trade-offs, design decisions, why this approach
  - **Do**: Hands-on exercise with verification after every step
- Proper AsciiDoc formatting: `role="execute"` for terminal commands, `subs="attributes+"` for RHDP variables
- Minimum 5 modules, 50 lines per module — addresses the "too light" feedback
- Progressive complexity: users build the stack piece by piece

### Phase 5: VERIFY
- 20+ quality checks across 4 categories:
  - **S.\***: Structure (site.yml, ui-config, antora.yml, nav, supplemental-ui)
  - **D.\***: Content depth (scenario framing, See section, key takeaways, minimum length)
  - **E.\***: AsciiDoc formatting (role="execute", attributes, heading nesting, no secrets)
  - **B.\***: Cross-module consistency (progressive build, terminology, acronyms)
- Antora build verification (zero warnings, all pages generated)

### Phase 6: LAB GRADING (optional)
- Generates FTL solve/validate Ansible playbooks for E2E testing
- Requires live environment (OCP cluster)

### Phase 7: PUBLISH
- Local file output or PR with full report
- Launchpad callback for provisioning intelligence

## Output

Two artifacts ready to push:

| Artifact | Purpose |
|----------|---------|
| `showroom-<name>/` | Complete Showroom lab repo (Antora content, ui-config, supplemental-ui, Intel branding) |
| AgnosticV catalog files | `common.yaml`, `dev.yaml`, `description.adoc`, `info-message-template.adoc` — drop into AgV repo and submit |

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

## Dependencies

| Tool | Purpose | Location |
|------|---------|----------|
| NovaScan | Capacity planning, tier recommendation | `~/Documents/novascan` |
| DarkScope | Security scanning | `~/Documents/darkscope` |
| MaaS model map | Model assignment for showroom labs | `~/Documents/intel-quickstarts-triforce/quickstart-to-showroom/templates/maas-model-map.yaml` |
| Showroom templates | Supplemental-ui, page templates | `~/Documents/intel-quickstarts-triforce/quickstart-to-showroom/templates/` |
