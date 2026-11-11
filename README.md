# Developer Documentation Hub (`dev-docs`)

Welcome to `dev-docs`, the central repository for project documentation, feature guides, task planning, and pull request records across all projects.

The rules in this repo are mandatory for every agent, chat thread and task inside `D:\projects\livik`. The folder's `CLAUDE.md` (one level up from this repo) points every agent here.

---

## Workspace Layout

```text
dev-docs/
  README.md                    <- Overall repository documentation (this file)
  templates/
    INSTRUCTIONS.md            <- Task lifecycle rules and workflow guide
    TEMPLATE-PR.md             <- Per-repo PR template (one copy per changed repo)
    TEMPLATE-FEATURE-INFO.md   <- Non-technical feature guide template
    TEMPLATE-FEATURE-DEV.md    <- Technical developer & AI agent guide template
    SKILL.md                   <- Claude agent skill for generating PR docs
  <project-name>/
    features/
      <feature-name>/          <- Feature documentation (business & developer guides)
    tasks/
      <branch-name>/
        <YYYY-MM-DD-slug>/     <- Task folder created per branch/task
          PLAN.md              <- Optional task plan
          PR-<repo>.md         <- One PR doc per changed repo (e.g. PR-frontend.md, PR-backend.md)
```

---

## Documentation Templates

All documentation in this repository follows the templates in [`templates/`](templates).

### 1. [`INSTRUCTIONS.md`](templates/INSTRUCTIONS.md)
The primary rulebook for developers and AI agents. It defines task lifecycles, folder naming conventions, git safety policies, and writing rules. Read this file first before starting any task.

### 2. [`TEMPLATE-PR.md`](templates/TEMPLATE-PR.md)
Used to create the PR description files. Make one copy per changed repository, named `PR-<repo>.md` (for example `PR-frontend.md` and `PR-backend.md`). There is no combined PR file.

### 3. [`TEMPLATE-FEATURE-INFO.md`](templates/TEMPLATE-FEATURE-INFO.md)
Used to document features for non-technical stakeholders (business owners, product managers, support teams). Focuses on real-world analogies, high-level process flows, key rules, and FAQs without technical jargon.

### 4. [`TEMPLATE-FEATURE-DEV.md`](templates/TEMPLATE-FEATURE-DEV.md)
Used to document technical details for developers and AI agents. Covers required data fields, database table/column mappings, validation checklists, architecture diagrams, and step-by-step implementation instructions.

### 5. [`SKILL.md`](templates/SKILL.md)
The reusable Claude Agent Skill definition that automates reading diffs and generating `PR-<repo>.md` files according to all repository guidelines. It is also installed as a working skill named `generate-pr` at `D:\projects\livik\.claude\skills\generate-pr\SKILL.md`. If you change one, change the other.

---

## Workflow & Guidelines

### Documenting a New Feature
1. Create a folder under `<project-name>/features/<feature-name>/`.
2. For business/non-technical documentation, copy [`TEMPLATE-FEATURE-INFO.md`](templates/TEMPLATE-FEATURE-INFO.md) into the folder and fill in the details.
3. For developer/AI implementation details, copy [`TEMPLATE-FEATURE-DEV.md`](templates/TEMPLATE-FEATURE-DEV.md) into the folder and fill in the details.
4. Remove all template instructions and comment blocks before finalizing.
5. A short `README.md` in the feature folder can point readers to the right file and give the current status.

### Documenting a Task or Pull Request
1. Create a task directory under `<project-name>/tasks/<branch-name>/<YYYY-MM-DD-short-slug>/`.
2. If the work requires upfront planning, create `PLAN.md`.
3. When the work is done and the user asks for the PR docs, create one `PR-<repo>.md` per changed repo using [`TEMPLATE-PR.md`](templates/TEMPLATE-PR.md).

### Core Writing & Git Rules
* **Simple English**: Write clearly so any teammate can understand without heavy jargon.
* **No Em Dashes**: Do not use em dashes anywhere. Use commas, periods, or brackets instead.
* **No Hard Wrapping**: Keep paragraphs on single lines and let the editor handle line wrapping.
* **One PR Document per Repo**: Each changed repo gets its own `PR-<repo>.md`. Never a combined file. Omit empty sections.
* **Git Safety**: Git operations for automated tools and AI agents are strictly read-only (`git status`, `git diff`, `git log`, `git show`). Never commit, push, or change branches. The user commits and opens PRs.
* **Honest Documentation**: Always report actual test results and list pre-existing bugs explicitly.

---

## Feature Docs in This Repo

| Project | Feature | Folder |
|---|---|---|
| erp-accounting | e-Invoice | [`erp-accounting/features/e-invoice/`](erp-accounting/features/e-invoice) |
| erp-accounting | e-Way Bill | [`erp-accounting/features/e-way bill/`](<erp-accounting/features/e-way bill>) |
| toughened-glass | AI photo reading of handwritten orders | [`toughened-glass/features/ai-extraction/`](toughened-glass/features/ai-extraction) |
| toughened-glass | Barcodes, label printing, Production Line | [`toughened-glass/features/barcode-scanner/`](toughened-glass/features/barcode-scanner) |

Note: some older task folders hold a single `PR.md` from before the one-file-per-repo rule. They are kept as they were.
