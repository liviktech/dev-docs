---
name: generate-pr
description: Generates per-repo PR markdown documents (PR-<repo>.md, e.g. PR-frontend.md, PR-backend.md) based strictly on repo diffs without common PR files, omitting empty sections entirely.
---

# Generate PR Documentation Skill

Use this skill whenever asked to generate PR documentation, create PR md files, or summarize task diffs for pull requests across frontend and backend repositories.

## Workflow Rules & Lifecycle

### 1. Inspection & Safety
- **Git is Read-Only**: Use only `git status`, `git diff`, `git log`, and `git show`. Never commit, push, checkout, stash, merge, or rebase.
- Read actual diffs across all touched repositories (frontend, backend, etc.) to understand the full scope of changes.

### 2. Folder & File Structure
- Create a folder under: `<project-name>/tasks/<branch-name>/<YYYY-MM-DD-short-slug>/`
  - `YYYY-MM-DD`: Current date.
  - `short-slug`: 2 to 4 kebab-case words derived from the diff (e.g., `user-auth-flow`, `payment-null-check`).
- Make **one PR file per changed repository**, named `PR-<short repo name>.md` (e.g., `PR-frontend.md` and `PR-backend.md`).
- Do **not** create any common or combined `PR.md` file. Every PR document is strictly per-repository.

---

## Writing Style Constraints

- **Plain English**: Write simply as if explaining to a teammate. Avoid corporate or heavy technical jargon.
- **No Em Dashes**: Never use em dashes anywhere. Use a comma, a period, or brackets instead.
- **No Hard Wrapping**: Keep each paragraph on a single line. Let the editor handle wrapping.
- **No Template Comments**: Delete all HTML comment blocks (`<!-- ... -->`) from the generated PR files.
- **Omit Empty Sections**: If a section has no points or content (e.g. Companion PR, Decisions made, Bugs found while testing, Notes, Next steps), omit the entire section from the final file instead of writing "None." or leaving blank headers.
- **PR Link Placeholders**: Keep PR link placeholders as `<... PR link>` for the user to replace when opening the PRs.
- **Companion PRs**: When multiple repositories change, each `PR-<repo>.md` file MUST begin with a Companion PR block referencing the other repository's PR, explaining what the other side does, which merges first, and the risk if merged out of order. Omit this section if only one repo changed.
- **Intentional Exclusions**: Anything left out on purpose must be explicitly stated in bold with the reason.
- **Pre-existing Bugs**: Flag pre-existing bugs found during testing without attempting to fix them in this PR.

---

## Generated File Structure (`PR-<repo>.md`)

Each generated `PR-<repo>.md` must follow this exact template structure:

```markdown
# <YYYY-MM-DD> - <short-slug> (<repo side, e.g. frontend or backend>)

<!-- COMPANION PR block (only if multiple repos changed. Omit entirely if single repo) -->
**Companion PR**: `<other repo name>` - `<other PR link>` (<one line description of what the other side does>). <Merge order and risk if reversed.>

## Summary
<2-3 plain sentences explaining what was missing/broken before, what this PR changes for this specific repo, and what cases were verified live.>

**<Scope item> is not included on purpose** - see "<section name>" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `<path>` | <what changed and why> |
| `<path>` (new) | <what it is> |
| `<path>` | no real change |

## Why
<Explanation of the reason for the change, pointing to ticket/spec if applicable.>

## How
**<Idea 1>**: <Plain explanation of implementation approach.>
**<Idea 2>**: <Plain explanation of implementation approach.>

## Decisions made
<!-- Omit section entirely if no decisions were made -->
**<Decision 1>.** <What was picked, what was rejected, and why.>

## Bugs found while testing (already there before, not fixed here)
<!-- Omit section entirely if no pre-existing bugs were found -->
**1. <Bug 1>.** <What it is, why it existed before, and who it affects.>

## Notes
<!-- Omit section entirely if no notes exist -->
- <Key local setup notes or live test observations.>

## Next steps
<!-- Omit section entirely if no next steps exist -->
- <Follow-up tasks or ticket items that do not block this PR.>

## Test plan
**Automated tests (<passing count>):**
- [x] `<test file>`: <case tested and result>
- [ ] `<lint/type check>`: <result>

**Manual checks (local):**
- [x] <exact action performed and observed result>
- [ ] <untested scenario> - <reason why and what covers it>
```
