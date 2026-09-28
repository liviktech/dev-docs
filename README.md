# Developer Documentation Hub (`dev-docs`)

Welcome to `dev-docs`, the central repository for project documentation, feature guides, task planning, and pull request records across all projects.

---

## Workspace Layout

```text
dev-docs/
  README.md                    <- Overall repository documentation (this file)
  templates/
    INSTRUCTIONS.md            <- Task lifecycle rules and workflow guide
    TEMPLATE-PR.md             <- Single PR template covering all touched repos (frontend & backend)
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
          PR.md                <- Single PR doc covering all touched repos (frontend, backend, etc.)
```

---

## Documentation Templates

All documentation in this repository follows the templates in [`templates/`](file:///d:/Livik-projects/dev-docs/templates).

### 1. [`INSTRUCTIONS.md`](file:///d:/Livik-projects/dev-docs/templates/INSTRUCTIONS.md)
The primary rulebook for developers and AI agents. It defines task lifecycles, folder naming conventions, git safety policies, and writing rules. Read this file first before starting any task.

### 2. [`TEMPLATE-PR.md`](file:///d:/Livik-projects/dev-docs/templates/TEMPLATE-PR.md)
Used to create the single task PR description file (`PR.md`). Document all changes for both frontend and backend repositories together in this single file, grouping file tables by repository.

### 3. [`TEMPLATE-FEATURE-INFO.md`](file:///d:/Livik-projects/dev-docs/templates/TEMPLATE-FEATURE-INFO.md)
Used to document features for non-technical stakeholders (business owners, product managers, support teams). Focuses on real-world analogies, high-level process flows, key rules, and FAQs without technical jargon.

### 4. [`TEMPLATE-FEATURE-DEV.md`](file:///d:/Livik-projects/dev-docs/templates/TEMPLATE-FEATURE-DEV.md)
Used to document technical details for developers and AI agents. Covers required data fields, database table/column mappings, validation checklists, architecture diagrams, and step-by-step implementation instructions.

### 5. [`SKILL.md`](file:///d:/Livik-projects/dev-docs/templates/SKILL.md)
The reusable Claude Agent Skill definition that automates reading diffs and generating `PR.md` files according to all repository guidelines.

---

## Workflow & Guidelines

### Documenting a New Feature
1. Create a folder under `<project-name>/features/<feature-name>/`.
2. For business/non-technical documentation, copy [`TEMPLATE-FEATURE-INFO.md`](file:///d:/Livik-projects/dev-docs/templates/TEMPLATE-FEATURE-INFO.md) into the folder and fill in the details.
3. For developer/AI implementation details, copy [`TEMPLATE-FEATURE-DEV.md`](file:///d:/Livik-projects/dev-docs/templates/TEMPLATE-FEATURE-DEV.md) into the folder and fill in the details.
4. Remove all template instructions and comment blocks before finalizing.

### Documenting a Task or Pull Request
1. Create a task directory under `<project-name>/tasks/<branch-name>/<YYYY-MM-DD-short-slug>/`.
2. If the work requires upfront planning, create `PLAN.md`.
3. When the work is completed, create a single `PR.md` file using [`TEMPLATE-PR.md`](file:///d:/Livik-projects/dev-docs/templates/TEMPLATE-PR.md) covering both frontend and backend repository changes.

### Core Writing & Git Rules
* **Simple English**: Write clearly so any teammate can understand without heavy jargon.
* **No Em Dashes**: Do not use em dashes anywhere. Use commas, periods, or brackets instead.
* **No Hard Wrapping**: Keep paragraphs on single lines and let the editor handle line wrapping.
* **Single PR Document**: Combine frontend and backend repository changes into one `PR.md` per task.
* **Git Safety**: Git operations for automated tools and AI agents are strictly read-only (`git status`, `git diff`). Never auto-commit or push changes directly.
* **Honest Documentation**: Always report actual test results and list pre-existing bugs explicitly.
