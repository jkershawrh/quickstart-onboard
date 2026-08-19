# Onboard — Quickstart to Ready-Made Lab

Take a factory-standard quickstart and produce a complete, ready-to-deploy RHDP lab with real content — not stubs. The lab teaches users HOW to build what the quickstart builds using a "See, Learn, Do" structure. Output is a Showroom repo ready to push.

## Usage

- `/onboard ~/Documents/my-quickstart` — onboard local quickstart
- `/onboard https://github.com/org/quickstart` — clone and onboard
- `/onboard .` — onboard current directory

## Instructions

---

### Phase 1: SCAN

1. Parse `$ARGUMENTS` as a repo path or GitHub URL.
   - If URL: clone to temp dir with `git clone --depth 1`
   - If path: resolve and validate (exists, is directory)
   - Validate against shell injection (alphanumeric, dots, hyphens, underscores, tildes, slashes only)

2. Read the quickstart thoroughly — README, source code, Helm charts, scripts, tests — to understand:
   - What business problem it solves
   - The architecture (components, data flow, deployment method)
   - What technologies it uses (models, frameworks, Intel hardware)
   - What a user would learn by building it step by step

3. Run the capacity scan and security scan. All checks run inline — no external tools needed.

#### Capacity Scan

Detect what the quickstart needs by grepping the source:

**LLM framework detection** — grep for these imports/references:
- `openai`, `langchain`, `llama_index`, `vllm`, `transformers`, `torch`, `openvino`, `optimum`
- If found: the quickstart uses LLM inference

**Model detection** — grep for model name patterns:
- `granite`, `llama`, `qwen`, `mistral`, `phi`, `deepseek`, `falcon`, `gemma`, `nomic`
- Check Containerfiles, compose files, values.yaml, and Python source
- Note the model size (2B, 7B, 8B, 14B, 70B) — this determines resource needs

**Local inference detection** — does the quickstart run a model LOCALLY (in the user's cluster) or call a remote API?
- Local indicators: `vllm serve`, `--model` in container args, model download in Containerfile, `VLLM_CPU_KVCACHE_SPACE`, GPU resource requests
- Remote/MaaS indicators: API URL environment variables (`MODEL_ENDPOINT`, `OPENAI_API_BASE`), no model downloads, no GPU requests

**Infrastructure detection** — grep for:
- Databases: `postgres`, `redis`, `mongodb`, `sqlite`, `pgvector`
- Queues: `kafka`, `rabbitmq`, `celery`
- Frontends: `gradio`, `streamlit`, `react`, `angular`, `vue`, `open-webui`
- Helm charts: check `chart/` directory exists

**Resource extraction** — read K8s manifests and Helm values for explicit CPU/memory requests.

**Tier recommendation:**
- **dedicated**: local model inference detected OR total CPU > 20 cores OR total memory > 32 GB
- **partner**: uses LLM frameworks/models but via MaaS (no local inference)
- **pilot**: no LLM usage detected — simple app, MaaS API key only

**MaaS cluster sizing fix:** When the model runs on MaaS (not locally), the cluster only needs resources for the APPLICATION — frontend, backend, databases, embeddings. Do NOT size the cluster for model hosting. This was causing overprovisioning.

| Scenario | Cluster size | Workers | Why |
|---|---|---|---|
| MaaS-only (no local model) | small | 2 | App + showroom only |
| MaaS + embeddings (e.g., nomic) | small | 3 | App + embedding model pod |
| Local CPU model (< 8B) | medium | 3-4 | Model needs ~16 GB RAM |
| Local CPU model (>= 8B) | large | 4+ | Model needs 32+ GB RAM |
| Local GPU model | dedicated | GPU nodes | Gaudi/CUDA required |

**MaaS model selection** — match quickstart to the smallest model that works:

| Use case | Start with | Upgrade to | Only if |
|---|---|---|---|
| Chat / Q&A / classification | granite-2b-cpu | granite-3-2-8b-instruct-cpu | Accuracy matters (RAG, compliance) |
| Function calling / tool use | qwen3-14b | — | Only model with working FC |
| RAG generation | granite-3-2-8b-instruct-cpu | qwen3-14b | Need FC in RAG agent mode |
| Embeddings | nomic-embed-text-v1-5 | — | Standard choice |

Known broken models (do not assign):
- `granite-4-0-h-tiny-cpu` — 30s timeouts, no responses
- `deepseek-r1-distill-qwen-14b` — function calling does not work despite being listed as FC-capable

#### Security Scan

Run these checks by grepping the repo (exclude `.git/`, `node_modules/`, `venv/`):

**Secrets detection** (CRITICAL) — grep for:
- AWS keys: `AKIA[0-9A-Z]{16}`
- API keys: `sk-[A-Za-z0-9]{20,}`, `hf_[A-Za-z0-9]{20,}`, `ghp_[A-Za-z0-9]{20,}`
- JWTs: `eyJ[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}`
- Private keys: `-----BEGIN.*PRIVATE KEY-----`
- Passwords in code: `password\s*[:=]\s*["'][A-Za-z0-9]`
- Connection strings: `postgresql://.*:.*@`, `mongodb://.*:.*@`, `redis://.*:.*@`
- High-value patterns: `api_key\s*[:=]\s*["'][A-Za-z0-9]`, `secret\s*[:=]\s*["'][A-Za-z0-9]`

**Config issues** (HIGH) — check:
- `.gitignore` exists and covers `.env`, `*.pem`, `*.key`, `__pycache__`, `venv/`
- No `.env` files committed with real (non-placeholder) values
- No secrets in YAML/JSON config files

**Container security** (MEDIUM) — grep Containerfiles for:
- `USER root` or no USER directive (runs as root)
- `COPY .` (copies everything including secrets)
- `:latest` tag on base images (unpinned)
- `--privileged` in compose files
- `--tls-verify=false` in scripts

**Dangerous code patterns** (MEDIUM) — grep Python files for:
- `eval(`, `exec(`, `__import__(`
- `pickle.loads`, `yaml.load(` (without SafeLoader)
- `subprocess.*shell=True`
- `f"SELECT.*{` (SQL injection via f-string)

**Infrastructure exposure** (HIGH) — grep for:
- Internal RHDP hostnames: `sandbox.opentlc.com`, `infra.demo.redhat.com`, `redhatworkshops.io` hardcoded (not in attributes)
- Private IPs: `10\.\d+\.\d+\.\d+`, `172\.(1[6-9]|2[0-9]|3[01])\.\d+\.\d+`, `192\.168\.\d+\.\d+`

**Supply chain** (LOW) — check pinned dependency versions against known CVEs:
- `requests < 2.32.0` (CVE-2024-35195)
- `flask < 2.3.0` (CVE-2023-30861)
- `pyyaml < 6.0` (arbitrary code execution)
- `cryptography < 41.0` (multiple CVEs)

**Grading formula:**
- Each finding has severity: critical (10 pts), high (7), medium (4), low (2), info (0)
- Sum all points
- Grade: <= 10 = A, <= 30 = B, <= 60 = C, <= 90 = D, > 90 = F
- Blockers: grade D or F, any critical finding

4. Present combined scan report:

```
SCAN REPORT: <quickstart-name>
==============================
CAPACITY:
  Tier: pilot / partner / dedicated
  Model hosting: MaaS (shared) / Local (in-cluster)
  Cluster size: small / medium / large
  Workers: N
  Estimated resources: X CPU, Y GB RAM per seat
  MaaS model: <model-name>

SECURITY: Grade [A-F]
  Critical: N  High: N  Medium: N  Low: N
  Blockers: [list or "none"]
```

5. Ask: "Continue to remediation? [Y/n]"

### Phase 2: CLASSIFY + REMEDIATE

6. For each critical/high security finding, classify as:
   - **auto_fixable**: clear code-level fix (hardcoded secret -> env var, missing .gitignore entry, unpinned image tag) — propose edit
   - **needs_human**: architectural issue (running as root by design, SQL injection in core logic) — report only
   - **false_positive**: test fixture, placeholder value, example in docs — skip with explanation

7. For auto_fixable findings:
   - Show finding: check name, severity, file:line, what was found
   - Show proposed fix using the Edit tool
   - Ask: "Apply fix? [Y/n/skip all]"

8. If fixes applied, re-run the security grep checks to verify grade improved. Report delta.

### Phase 3: AGNOSTICV CATALOG CONFIG

9. Generate the full AgnosticV catalog for RHDP deployment. The output must be directly submittable — no manual field additions needed.

Generate a UUID for the asset and check for collisions if an AgV repo path is known.

#### common.yaml

```yaml
---
# Auto-generated by /onboard — QUICKSTART_NAME
# Submit to: agnosticv/CATALOG_PATH/common.yaml

__meta__:
  catalog:
    display_name: "DISPLAY_NAME"
    # Short description for catalog card (1 sentence, business-first)
    description: >-
      SHORT_DESCRIPTION
    # Category: one of Workshops, Demos, Labs, Sandboxes, Brand_Events
    category: Workshops
    multiuser: true
    labels:
      - partner-intel
      - ai
      - quickstart
    keywords:
      - KEYWORD1
      - KEYWORD2
      - intel
      - ai
    icon: intelligent-automation
    reportingLabels:
      # primaryBU: one of Hybrid_Platforms, Application_Services, Ansible, RHEL, Cloud_Services, AI
      primaryBU: AI
      product: Red Hat OpenShift AI

  components:
    - name: cluster
      item: agd-v2/ocp-cluster-cnv-pools
      parameter_values:
        cluster_size: small
        ocp_version: "4.21"
        num_users: 30
        # Worker scaling: ceil(num_users / USERS_PER_WORKER) + 1 spare
        worker_instance_count_formula: "ceil(num_users / 10) + 1"
      propagate_provision_data:
        - openshift_api_url
        - openshift_cluster_admin_token
        - openshift_cluster_ingress_domain
        - openshift_console_url

  deployer:
    # Execution environment image — use latest chained build
    execution_environment: quay.io/agnosticd/ee-multicloud:chained-latest
    type: agnosticd

  asset_uuid: "GENERATE-UUID-HERE"

config: openshift-workloads
cloud_provider: none

# --- Workloads (order matters: authentication -> app workloads -> showroom) ---
workloads:
  # 1. Authentication — sets up user accounts
  - name: ocp4_workload_authentication
    role: ocp4_workload_authentication
    vars:
      ocp4_workload_authentication_idm_type: htpasswd
      ocp4_workload_authentication_htpasswd_user_count: "{{ num_users }}"
      ocp4_workload_authentication_htpasswd_user_password: openshift
      ocp4_workload_authentication_remove_kubeadmin: true

  # 2. LiteLLM virtual keys — MaaS model access (required for AI quickstarts)
  - name: ocp4_workload_litellm_virtual_keys
    role: ocp4_workload_litellm_virtual_keys
    vars:
      ocp4_workload_litellm_virtual_keys_litellm_base_url: "https://litellm-prod.apps.maas.redhatworkshops.io"
      ocp4_workload_litellm_virtual_keys_model_list:
        - MAAS_MODEL
      ocp4_workload_litellm_virtual_keys_num_users: "{{ num_users }}"
      ocp4_workload_litellm_virtual_keys_budget_per_key: 5.0
      ocp4_workload_litellm_virtual_keys_max_budget: 150.0

  # 3. Quickstart application workload (customize per quickstart)
  # - name: ocp4_workload_QUICKSTART_NAME
  #   role: ocp4_workload_QUICKSTART_NAME
  #   vars:
  #     # Add quickstart-specific deployment vars here

  # 4. Showroom — lab content delivery
  - name: ocp4_workload_showroom
    role: ocp4_workload_showroom
    vars:
      ocp4_workload_showroom_content_git_repo: "https://github.com/rh-ai-quickstart/showroom-QUICKSTART_NAME.git"
      ocp4_workload_showroom_content_git_repo_ref: main
      ocp4_workload_showroom_template: showroom_template_nookbag
```

Fill in all placeholder values (DISPLAY_NAME, SHORT_DESCRIPTION, QUICKSTART_NAME, MAAS_MODEL, keywords) from the quickstart's README and NovaScan results. The workload order is critical: authentication first, then LiteLLM keys, then app workload, then showroom last.

Adjust based on NovaScan tier:
- **pilot**: `cluster_size: small`, `num_users: 10`, no GPU workloads
- **partner**: `cluster_size: medium`, `num_users: 30`, CPU or endpoint-GPU
- **dedicated**: `cluster_size: large`, `num_users: 50+`, direct-GPU if needed

#### dev.yaml

```yaml
---
# Development environment overrides
# Shorter TTL, fewer users, for testing

__meta__:
  catalog:
    display_name: "DISPLAY_NAME (Dev)"
    labels:
      - dev
      - partner-intel

  components:
    - name: cluster
      parameter_values:
        num_users: 1
        worker_instance_count_formula: "2"

  deployer:
    # Dev can use latest or pinned EE
    execution_environment: quay.io/agnosticd/ee-multicloud:chained-latest

# Override TTL for dev
lifetime: 4h
max_lifetime: 8h
```

#### description.adoc

```asciidoc
= DISPLAY_NAME

SHORT_DESCRIPTION_BUSINESS_FIRST

== What you will learn

* Learning outcome 1 (business-oriented)
* Learning outcome 2
* Learning outcome 3
* Learning outcome 4
* Learning outcome 5

== Prerequisites

* A web browser
* No prior OpenShift experience required

== Time estimate

This lab takes approximately 90 minutes to complete.

== Technology

* Red Hat OpenShift AI
* QUICKSTART_TECHNOLOGY
* Powered by Intel Xeon processors

== Support

If you encounter issues, contact the lab facilitator or visit https://red.ht/rhdp-ticket
```

#### info-message-template.adoc

```asciidoc
= DISPLAY_NAME

Your lab environment is ready.

== Access

[cols="1,3"]
|===
| OpenShift Console | {{ openshift_console_url }}
| Terminal (Showroom) | Open the Showroom tab in the catalog
| Username | {{ user }}
| Password | {{ password }}
|===

== Quick Start

. Open the Showroom tab to access the lab guide
. Follow the modules in order — each builds on the previous
. The terminal panel on the right executes commands directly on the cluster

== Powered by

Red Hat OpenShift AI running on Intel Xeon processors.
```

#### AgnosticV Validation Checklist

Before reporting, verify the generated catalog passes these checks:

- [ ] `__meta__.catalog.category` is one of: `Workshops`, `Demos`, `Labs`, `Sandboxes`, `Brand_Events`
- [ ] `__meta__.catalog.reportingLabels.primaryBU` is one of: `Hybrid_Platforms`, `Application_Services`, `Ansible`, `RHEL`, `Cloud_Services`, `AI`
- [ ] `__meta__.components[].item` references a valid pool path (e.g., `agd-v2/ocp-cluster-cnv-pools`)
- [ ] `__meta__.components[].parameter_values.ocp_version` is a supported version (4.18, 4.20, 4.21)
- [ ] Workload order: authentication before showroom, LiteLLM before app workload
- [ ] Showroom workload has `ocp4_workload_showroom_content_git_repo` pointing to the correct showroom repo
- [ ] Showroom template is `showroom_template_nookbag` (underscores, not hyphens)
- [ ] No hardcoded `/prod` suffix on pool paths (ordering system handles this)
- [ ] If multiuser: `num_users` set, authentication configured for user count, worker scaling formula present
- [ ] If AI quickstart: `ocp4_workload_litellm_virtual_keys` workload present with model list
- [ ] No sensitive data (real passwords, tokens) — use Jinja2 variables or `user_data` secrets
- [ ] `asset_uuid` is generated and unique
- [ ] `dev.yaml` overrides TTL and user count for testing
- [ ] `description.adoc` leads with business problem, not technology

10. Present all generated files for review. After confirmation, write to output directory.

### Phase 4: SCAFFOLD SHOWROOM LAB

This is the core phase. Produce a complete Showroom lab with real content — not placeholder stubs. Every module has substance.

#### Output Structure

```
showroom-<name>/
├── content/
│   ├── antora.yml
│   ├── supplemental-ui/          # Send-to-terminal buttons, Intel branding
│   └── modules/ROOT/
│       ├── nav.adoc
│       ├── images/               # Architecture diagram, screenshots
│       └── pages/
│           ├── index.adoc        # Welcome + architecture
│           ├── 01-accessing-cluster.adoc
│           ├── 02-module-01.adoc # First hands-on module
│           ├── ...
│           └── NN-conclusion.adoc
├── default-site.yml
├── ui-config.yml
├── benchmark-config.yaml
└── README.adoc
```

#### antora.yml — RHDP Variables

```yaml
asciidoc:
  attributes:
    user: user1
    password: openshift
    guid: '%GUID%'
    openshift_api_url: 'https://api.cluster-%GUID%.%GUID%.sandbox.opentlc.com:6443'
    openshift_console_url: 'https://console-openshift-console.apps.cluster-%GUID%.%GUID%.sandbox.opentlc.com'
    openshift_cluster_ingress_domain: 'apps.cluster-%GUID%.%GUID%.sandbox.opentlc.com'
    rhoai_dashboard_url: 'https://rhods-dashboard-redhat-ods-applications.apps.cluster-%GUID%.%GUID%.sandbox.opentlc.com'
    project_name: '<quickstart-name>'
    quickstart_repo: 'https://github.com/rh-ai-quickstart/<name>.git'
    maas_api_url: 'https://litellm-prod.apps.maas.redhatworkshops.io/v1'
    maas_model: '<selected-model>'
    intel_platform: 'Intel Xeon'
```

#### ui-config.yml

```yaml
type: showroom
default_width: 30
persist_url_state: true
view_switcher:
  enabled: true
  default_mode: split
tabs:
  - name: Terminal
    path: /wetty
    port: 443
  - name: App UI
    url: 'https://<app-route>.${openshift_cluster_ingress_domain}'
  - name: OCP Console
    url: '${openshift_console_url}'
```

#### Module Structure — "See, Learn, Do"

Every lab module (except index, accessing-cluster, and conclusion) MUST follow this structure. This is the fix for "too light" content.

```asciidoc
= Module N: [Action-Oriented Title]

[Scenario framing — 2-3 sentences setting up WHY this module matters.
Connect to the business problem. What situation is the user in?]

== What you will learn

* [Learning outcome 1 — specific, measurable]
* [Learning outcome 2]
* [Learning outcome 3]

== See: [What you're building]

[Explain the architecture or component this module focuses on.
Show a diagram or describe the data flow. 3-5 paragraphs.
The user should understand the DESIGN before touching the keyboard.]

image::module-N-architecture.png[Architecture for module N,link=self,window=_blank]

[Optional: explain the trade-offs, why this approach was chosen,
what alternatives exist. This is the "learn" depth that prevents
the lab from being "too light."]

== Do: [Hands-on exercise]

=== Step 1: [Verb-first instruction]

[1-2 sentences explaining what this step does and why.]

[source,bash,role="execute",subs="attributes+"]
----
oc new-project {project_name}
----

[Expected output or what to observe. Help the user verify they did it right.]

=== Step 2: [Next step]

[Continue with clear instructions. Every command block must use
role="execute" for the send-to-terminal button.]

[source,bash,role="execute",subs="attributes+"]
----
helm install {project_name} chart/ \
  --set model.endpoint={maas_api_url} \
  --set model.name={maas_model}
----

=== Verify

[How to confirm this module's work succeeded.]

[source,bash,role="execute",subs="attributes+"]
----
oc get pods -n {project_name}
----

[Describe what the user should see. Be specific — pod names, status, counts.]

== Key takeaway

[2-3 sentences summarizing what the user built and why it matters.
Connect back to the business problem. What can they now do that they couldn't before?]
```

#### Content Depth Rules

These rules address the "too light" feedback directly:

1. **Every module needs the "See" section.** Don't jump straight to commands. Explain what the user is building and why before they type anything. 3-5 paragraphs minimum.

2. **Explain trade-offs.** Why this model and not another? Why Helm and not raw manifests? Why this architecture? Users learn from understanding decisions, not just following steps.

3. **Connect to the business problem.** Each module's scenario framing ties back to the quickstart's business value. "You're a platform engineer who needs to..." not "In this module we will deploy..."

4. **Verify after every major step.** Don't make users wait until the end to find out something failed. Show them how to check after each step.

5. **No naked commands.** Every `[source,bash,role="execute"]` block must have 1-2 sentences before it explaining what it does and why. A lab is not a script.

6. **Key takeaway is mandatory.** Every module ends with what the user learned and why it matters. This is what separates a lab from a README.

7. **Progressive complexity.** Module 1 is simple (access cluster, clone repo). Module 2 introduces the first component. Each subsequent module adds a layer. The user builds the full stack piece by piece.

8. **Minimum 5 modules** for a workshop lab. Fewer than that means the content is too compressed.

#### AsciiDoc Formatting Rules

- Terminal commands: `[source,bash,role="execute",subs="attributes+"]` — the `role="execute"` adds the send-to-terminal button
- Code display (not executable): `[source,yaml]` or `[source,python]` — no `role="execute"`
- RHDP variables: use `{attribute_name}` with `subs="attributes+"` — these are filled at provision time
- Images: `image::filename.png[Descriptive alt text,link=self,window=_blank]`
- Admonitions: `NOTE:`, `TIP:`, `IMPORTANT:`, `WARNING:`, `CAUTION:`
- Lists: blank line before the first item, no blank lines between items
- Headings: `=` for title, `==` for sections, `===` for subsections — no skipping levels
- Links: descriptive text, not bare URLs or "click here"
- No sensitive data (passwords, API keys, tokens) in content — use `{attributes}`

#### Index Page

The welcome page must include:
- Lab title and 2-3 sentence description
- "What you will learn" bullet list (5-8 items)
- Prerequisites (provisioned cluster, browser, terminal)
- Time estimate (typically 60-90 minutes)
- Architecture diagram: `image::architecture.png[Architecture diagram,link=self,window=_blank]`

#### Conclusion Page

- "What you accomplished" — bullet list mapping to the learning outcomes from index
- Cleanup commands with `role="execute"`
- Next steps (quickstart source, RHDP catalog, Red Hat OpenShift AI docs)
- Intel co-branding: "Powered by {intel_platform}"

#### MaaS Model Selection

Check `~/Documents/intel-quickstarts-triforce/quickstart-to-showroom/templates/maas-model-map.yaml` for assigned models. Selection rules:
- CPU inference: granite-2b-cpu, granite-3-2-8b-instruct-cpu, phi3-mini-cpu
- Function calling required: qwen3-14b, llama-scout-17b
- RAG with embeddings: nomic-embed-text-v1-5 for embeddings, granite-3-2-8b for generation
- Branding: "Powered by Intel" only. Never "Intel Gaudi" in user-facing content. "Intel Xeon 6" is acceptable.

#### Intel Co-Branding

- Footer: "Powered by Red Hat OpenShift AI running on {intel_platform} processors"
- Use supplemental-ui from `~/Documents/intel-quickstarts-triforce/quickstart-to-showroom/templates/supplemental-ui/` for Intel-branded header
- Intel logo in supplemental-ui if available

### Phase 5: VERIFY LAB QUALITY

After generating all content, run these quality checks. Fix any failures before reporting.

#### Structure Checks (S.*)
- [ ] S.1: `default-site.yml` exists and references correct content source
- [ ] S.2: `ui-config.yml` has Terminal, App UI, and OCP Console tabs
- [ ] S.3: `antora.yml` has all required RHDP attributes
- [ ] S.4: `nav.adoc` lists all pages in correct order
- [ ] S.5: All `.adoc` files referenced in nav.adoc exist
- [ ] S.6: `supplemental-ui/` directory exists with send-to-terminal assets

#### Content Checks (D.*)
- [ ] D.1: Every module has scenario framing (business context, not just tech)
- [ ] D.2: Every module has "What you will learn" section
- [ ] D.3: Every module has "See" section (architecture/explanation before commands)
- [ ] D.4: Every module has "Key takeaway" section
- [ ] D.5: No module is less than 50 lines (too thin)
- [ ] D.6: Index page has architecture diagram reference
- [ ] D.7: Conclusion has cleanup commands and next steps

#### Formatting Checks (E.*)
- [ ] E.1: All terminal commands use `[source,bash,role="execute",subs="attributes+"]`
- [ ] E.2: No bare `[source,bash]` without `role="execute"` for commands users should run
- [ ] E.3: All RHDP variables use `{attribute_name}` syntax, not hardcoded values
- [ ] E.4: Images use `link=self,window=_blank` attribute
- [ ] E.5: Heading levels nest correctly (no `=` to `===` skip)
- [ ] E.6: No sensitive data (passwords, tokens, API keys) in plain text
- [ ] E.7: Every command block has explanatory text before it

#### Cross-Module Checks (B.*)
- [ ] B.1: Modules build progressively (each adds a layer)
- [ ] B.2: No module references something not yet introduced
- [ ] B.3: Acronyms expanded on first use
- [ ] B.4: Consistent terminology across modules
- [ ] B.5: nav.adoc matches actual page files

Fix any failures. Then attempt an Antora build if `npx` is available:

```bash
cd OUTPUT_DIR && npx antora default-site.yml
```

Verify zero warnings, all pages generated, no unresolved attributes.

### Phase 6: LAB GRADING (optional)

If the user requests E2E test automation and has a live environment:

Generate per-module Ansible playbooks:
- `runtime-automation/module-XX/solve.yml` — performs all student tasks programmatically
- `runtime-automation/module-XX/validate.yml` — checks each task was completed

One module at a time. Test cycle: fresh validate (expect fail) -> solve -> validate after solve (expect pass).

Skip this phase if not requested or no live environment.

### Phase 7: PUBLISH

11. Ask: "Publish mode? [local / PR]"

    **Local**: Write files, show file list, done.

    **PR**: Create branch (`onboard/<repo-name>`), commit all generated files, push, `gh pr create` with body including:
    - NovaScan tier recommendation
    - DarkScope security grade
    - AgnosticV catalog summary
    - Lab module count and content summary
    - Quality check results

12. Run `/preflight` on the result as final verification.

13. Publish onboard outcome to Launchpad via POST `/callbacks/`:
    - Event type: `demo-onboarded`
    - Includes: catalog_item_id, tier, models, security_grade, config_path, pr_url
    - Launchpad unreachable: warning, not failure

---

## Pipeline Position

**repo** --`/intake`--> **quickstart** --`/onboard`--> **ready-made lab on RHDP**

## Arguments

$ARGUMENTS - Path to quickstart repo or GitHub URL. Examples:
- `/onboard ~/Documents/my-quickstart`
- `/onboard https://github.com/rh-ai-quickstart/edge-ai-cpu-inference`
- `/onboard .`

## Dependencies

None. All scanning, security checks, and content generation run inline. No external tools (NovaScan, DarkScope) or RHDP marketplace plugins required.

## Important Rules

- **Labs are not deployment scripts.** Every module explains WHY before HOW. "See, Learn, Do" is mandatory, not optional.
- **"Too light" is a failure.** If a module is just commands with no explanation, it fails D.3 and must be rewritten.
- **Business problem first.** Scenario framing connects to the quickstart's business value, not just the technology.
- **No stubs.** Every module has real content. No `[TODO]` placeholders in the output.
- **Progressive build.** Users construct the stack piece by piece and understand each layer before adding the next.
- **Verify after every step.** Don't let users go 3 steps before finding out step 1 failed.
- **Intel branding.** "Powered by Intel" only. No "Intel Gaudi" in user-facing content.
- **MaaS = small cluster.** When the model runs on MaaS, size the cluster for the app only — not the model. This prevents overprovisioning.
- **Smallest model that works.** Start with the smallest MaaS model that meets the use case. Don't default to qwen3-14b when granite-2b-cpu will do.
