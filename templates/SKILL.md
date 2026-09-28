---
name: generate-pr
description: Generates a single PR markdown document (PR.md) covering all changed repositories (frontend and backend) based on dev-docs INSTRUCTIONS.md and TEMPLATE-PR.md rules.
---

# Generate PR Documentation Skill

Use this skill whenever asked to generate PR documentation, create `PR.md` files, or summarize task diffs across frontend and backend repositories for pull requests.

## Workflow Rules & Lifecycle

### 1. Inspection & Safety
- **Git is Read-Only**: Use only `git status`, `git diff`, `git log`, and `git show`. Never commit, push, checkout, stash, merge, or rebase.
- Read actual diffs across all touched repositories (frontend, backend, etc.) to understand the full scope of changes.

### 2. Folder & File Structure
- Create a folder under: `<project-name>/tasks/<branch-name>/<YYYY-MM-DD-short-slug>/`
  - `YYYY-MM-DD`: Current date.
  - `short-slug`: 2 to 4 kebab-case words derived from the diff (e.g., `user-auth-flow`, `payment-null-check`).
- Create a **single** `PR.md` file in the task folder covering all touched repositories. Do not create separate PR files for frontend and backend.

---

## Writing Style Constraints

- **Plain English**: Write simply as if explaining to a teammate. Avoid corporate or heavy technical jargon.
- **No Em Dashes**: Never use em dashes anywhere. Use a comma, a period, or brackets instead.
- **No Hard Wrapping**: Keep each paragraph on a single line. Let the editor handle wrapping.
- **No Template Comments**: Delete all HTML comment blocks (`<!-- ... -->`) from the generated `PR.md` file.
- **PR Link Placeholders**: Keep PR link placeholders as `<... PR link>` for the user to replace when opening the PRs.
- **Intentional Exclusions**: Anything left out on purpose must be explicitly stated in bold with the reason.
- **Pre-existing Bugs**: Flag pre-existing bugs found during testing without attempting to fix them in this PR.

---

## Generated File Structure (`PR.md`)

The generated `PR.md` must follow this exact template structure:

```markdown
# <YYYY-MM-DD> - <short-slug>

<!-- COMPANION PR block (only if external PR dependency exists outside this task) -->
**Companion PR**: `<external repo name>` - `<PR link>` (<one line description>). <Merge order and risk if reversed.>

## Summary
<2-3 plain sentences explaining what was missing/broken before, what this PR changes across frontend and backend, and what cases were verified live.>

**<Scope item> is not included on purpose** - see "<section name>" below. This was a choice, not a miss.

## What changed

### <Frontend Repo Name>
| File | Change |
|---|---|
| `<path>` | <what changed and why> |
| `<path>` (new) | <what it is> |
| `<path>` | no real change |

### <Backend Repo Name>
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
**<Decision 1>.** <What was picked, what was rejected, and why.>

## Bugs found while testing (already there before, not fixed here)
**1. <Bug 1>.** <What it is, why it existed before, and who it affects.>

## Notes
- <Key local setup notes or live test observations.>

## Next steps
- <Follow-up tasks or ticket items that do not block this PR.>

## Test plan
**Automated tests (<passing count>):**
- [x] `<test file>`: <case tested and result>
- [ ] `<lint/type check>`: <result>

**Manual checks (local):**
- [x] <exact action performed and observed result>
- [ ] <untested scenario> - <reason why and what covers it>
```
