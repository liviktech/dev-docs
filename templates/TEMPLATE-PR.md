<!--
TEMPLATE for a PR description. This template is per-repository.
Copy it into the task folder and rename it per repo as PR-<short repo name>.md (e.g. PR-frontend.md, PR-backend.md). Make one PR file per repo that changed. There are no common or combined PR files. See INSTRUCTIONS.md for rules.
Fill in every <placeholder>.
IMPORTANT: If any section has no points or content (e.g. Companion PR, Decisions made, Bugs found while testing, Notes, Next steps), OMIT the entire section from the final PR file instead of writing "None.".
Delete all of these comment blocks before you finish. The final file should have no comments left.
Keep the writing simple and plain. Say what is true. If you checked something live, say so. If you did not check it, say so. Do not oversell.
Do not commit or push. This file is just written to disk. The user opens the PR. Leave PR links as <...> for the user to fill in.
-->

# <YYYY-MM-DD> - <short-slug> (<repo side, for example frontend or backend>)

<!-- COMPANION PR block. Only keep this if the task changes multiple repos (e.g. both frontend and backend). Omit this section entirely if there is no companion PR. -->
**Companion PR**: `<other repo name>` - `<other PR link>` (<one line on what the other side does>). <Which one merges first and what breaks if the order is wrong.>

## Summary

<!-- A few plain sentences. What was missing or broken before. What this PR changes for this specific repo. Which case had to keep working, and whether you checked it live or just assumed it. -->
<summary>

<!-- If something is left out on purpose, say it here in bold. Omit this line if nothing is left out. -->
**<X> is not included on purpose** - see "<section name>" below. This was a choice, not a miss.

## What changed

<!-- One row per file or group of files in this repository. Mark new files as (new) and renamed files as (renamed). If a file was only touched for comments or a rename with no real change, say "no real change" so the reviewer knows not to worry about it. -->
| File | Change |
|---|---|
| `<path>` | <what changed and why> |
| `<path>` (new) | <what it is> |

## Why

<!-- The reason for the change. Point to the ticket or spec if one drove it. Say what was easy and what was the real work. -->
<why>

## How

<!-- How it works. One short bold lead-in per idea, then a plain explanation. Call out anything you kept the same on purpose, like a value passed through untouched or a function signature that did not change. -->
**<idea>**: <explanation>

## Decisions made

<!-- Choices a reviewer might question, or things that changed while you were building. Say what you picked, what you did not pick, and why. Omit this section entirely if there were no decisions made. -->
**<decision>.** <what you picked, what you did not, and why>

## Bugs found while testing (already there before, not fixed here)

<!-- Bugs you ran into that this PR does not fix, so they are not lost. For each one say what it is, why it was already there before this change, and who it affects. Omit this section entirely if no pre-existing bugs were found. -->
**1. <bug>.** <what it is, why it was already there, who it affects>

## Notes

<!-- Loose ends worth a look: things you checked live, local setup gotchas, why you left something as is. Omit this section entirely if there are no notes. -->
- <note>

## Next steps

<!-- Follow ups this PR leaves for later, like filing tickets for the bugs above. None of these block this PR. Omit this section entirely if there are no next steps. -->
- <next step>

## Test plan

<!-- Two groups. Use [x] for done and [ ] for not done or not possible, and say why if not possible. Name the exact test file and case. For manual checks, say what you actually saw, not what you hoped to see. -->
**Automated tests (<all passing / N passing, M pending>):**
- [ ] `<test file>`: <the case and what it should do>
- [ ] <build or type check or lint>: <result>

**Manual checks (local):**
- [ ] <what you ran, against what, and what you saw>
- [ ] <thing you could not check> - <why not, and what covers it instead>
