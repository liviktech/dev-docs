<!--
TEMPLATE for a PR description. This one template works for every repo.
Copy it into the task folder and rename it per repo as PR-<short repo name>.md, where the short name is usually the repo's folder name. Make one PR file per repo that changed. See INSTRUCTIONS.md for the naming and when to make more than one.
Fill in every <placeholder>. Keep sections even if the answer is short. If a section truly does not apply, write "None." instead of deleting it.
Delete all of these comment blocks before you finish. The final file should have no comments left.
Keep the writing simple and plain. Say what is true. If you checked something live, say so. If you did not check it, say so. Do not oversell.
Do not commit or push. This file is just written to disk. The user opens the PR. Leave PR links as <...> for the user to fill in.
-->

# <YYYY-MM-DD> - <short-slug> (<repo side, for example backend or storefront>)

<!-- COMPANION PR block. Only keep this if the task changes both repos. If it only changes one repo, delete this block. Say which PR is the other one, what it does in one line, and which must merge first and why. -->
**Companion PR**: `<other repo name>` - `<other PR link>` (<one line on what the other side does>). <Which one merges first and what breaks if the order is wrong.>

## Summary

<!-- A few plain sentences. What was missing or broken before. What this PR changes. Which case had to keep working, and whether you checked it live or just assumed it. -->
<summary>

<!-- If something is left out on purpose, say it here in bold. Delete this line if nothing is left out. -->
**<X> is not included on purpose** - see "<section name>" below. This was a choice, not a miss.

## What changed

<!-- One row per file or group of files. Mark new files as (new) and renamed files as (renamed). If a file was only touched for comments or a rename with no real change, say "no real change" so the reviewer knows not to worry about it. -->
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

<!-- Choices a reviewer might question, or things that changed while you were building. Say what you picked, what you did not pick, and why. Write "None." if there were none. -->
**<decision>.** <what you picked, what you did not, and why>

## Bugs found while testing (already there before, not fixed here)

<!-- Bugs you ran into that this PR does not fix, so they are not lost. For each one say what it is, why it was already there before this change, and who it affects. Write "None." if there were none. -->
**1. <bug>.** <what it is, why it was already there, who it affects>

## Notes

<!-- Loose ends worth a look: things you checked live, local setup gotchas, why you left something as is. -->
- <note>

## Next steps

<!-- Follow ups this PR leaves for later, like filing tickets for the bugs above. None of these block this PR. -->
- <next step>

## Test plan

<!-- Two groups. Use [x] for done and [ ] for not done or not possible, and say why if not possible. Name the exact test file and case. For manual checks, say what you actually saw, not what you hoped to see. -->
**Automated tests (<all passing / N passing, M pending>):**
- [ ] `<test file>`: <the case and what it should do>
- [ ] <build or type check or lint>: <result>

**Manual checks (local):**
- [ ] <what you ran, against what, and what you saw>
- [ ] <thing you could not check> - <why not, and what covers it instead>
