# Task lifecycle - read this first

This file explains how a task is set up, what you make at each step, and the rules you must never break. Feed this file to the agent at the start of a task.

## Repos

A task can change one repo or several (for example frontend and backend repos). Every PR document is created strictly for its specific repository (`PR-<repo>.md`). There are no common or combined PR files. Find out which repos a task touches from the actual changes, and give each changed repo its own short name (for example the folder name or role like frontend/backend). When a task changes more than one repo, make one PR file per changed repo (`PR-<repo>.md`).

## Folder layout - one folder per task

Each branch has its own folder under `tasks/`. Inside it, the agent creates one folder per task automatically. The agent derives the folder name from today's date and a short slug based on the actual diff. The user never names the folder manually.

```
dev-docs/
  templates/
    INSTRUCTIONS.md            <- this file
    TEMPLATE-PR.md             <- per-repo PR template
    TEMPLATE-FEATURE-INFO.md   <- feature doc template for business/non-technical audience
    TEMPLATE-FEATURE-DEV.md    <- feature doc template for developers and AI agents
    SKILL.md                   <- Claude agent skill for PR documentation generation
  <project-name>/
    tasks/
      <branch-name>/
        <YYYY-MM-DD-short-slug>/   <- agent names this from the date + diff
          PLAN.md                  <- optional, make this if the work needs upfront planning
          PR-<repo>.md             <- YOU make one of these per changed repo (e.g. PR-frontend.md, PR-backend.md)
    features/
      <feature-name>/              <- user names this folder
        README.md                  <- main doc for the feature, or multiple files if needed
```

The slug is 2 to 4 words in kebab-case that describe what changed (for example `user-auth`, `payment-null-fix`, `dashboard-card-redesign`). The agent picks it from reading the diff. Keep it short and honest.

The templates in `templates/` are the source of truth for shape and wording. Always copy from them. Do not invent a new shape.

## How to make the PR md files for each repo

There is one PR template, `TEMPLATE-PR.md`. Every PR file is created strictly for its specific repository. Copy the template once per repo that changed and rename it `PR-<short repo name>.md`, where the short name is usually the repo's folder name or role (e.g., `PR-frontend.md` and `PR-backend.md`). Do not create any common or combined PR files across repositories.

In each copy, fill in the title line with today's date, the slug, and that repo's role (for example frontend, backend, api, worker). Fill the rest from that repo's own diff and its own test run. Each PR file is about one repo only.

If any section has no points or content (such as Decisions made, Bugs found while testing, Notes, or Next steps), omit that section entirely from the PR file. Do not write "None." or leave empty section headers.

## The steps of a task

### 1. Intake
The user describes what they are working on or points the agent at the diff. Read the actual changes across all touched repos to understand what is in scope. If there is a written brief or description, read that too.

### 2. Plan (optional, only when the user asks or the work is complex)
If the user asks for a plan, or the work is large enough that a plan helps, create the task folder (`<branch-name>/<YYYY-MM-DD-short-slug>/`) and make `PLAN.md` inside it. The agent picks the slug from the diff. Put in the approach, the files you expect to touch in each repo, the risks and unknowns, what is out of scope on purpose, and how you will test. Keep it updated to match what really happened. If no plan is needed, skip this step and create the task folder only when making the PR files.

### 3. Build
Do the work. When real life differs from the plan, write it down. Those changes, and any "I tried one way then picked a safer way" moments, are what go into the PR sections "Decisions made" and "Bugs found while testing". Check things live when you claim they work. Keep a note of old bugs you run into but do not fix.

### 4. Report (only when the user asks)
Do not make the PR files on your own. When the task is done the user will ask for "the PR mds". Only then:

- Make one `PR-<repo>.md` from `TEMPLATE-PR.md` for each repo that changed (for example `PR-frontend.md` and `PR-backend.md`).

Fill them from the real diffs and the real test runs, not from what the plan hoped for. Delete every comment block from the copied templates. Replace every `<placeholder>`. If a section has no points, omit it entirely instead of writing "None.".

## Writing style for every file you make

- Use simple, plain English. No corporate or heavy technical wording. Write like you are explaining it to a teammate in a hurry.
- Never use em dashes anywhere. Use a comma, a period, or brackets instead.
- Do not hard wrap the text. Keep each paragraph on one line and let the editor wrap it. Do not break a sentence across lines to hit some width, and do not add a character limit per line.
- Be honest over impressive. "Checked live" and "could not check, here is why" are both fine. Never claim a check you did not run.
- Anything left out on purpose is said out loud in bold with the real reason, not left silent.
- Old bugs get their own section or list so they are not lost. You do not fix them here, you flag them.
- Omit empty sections entirely instead of writing "None." or leaving blank headers.
- Help the reviewer. Point out files that had no real change so they do not waste time on them. Name the exact test file and case in the test plan.
- PR files use headings, tables, and checkboxes and go in the PR description.

## Line endings (EOL) for new files

Every new file you create must use the same line ending style as the other files already in that repo. A repo may use CRLF or LF, so do not assume one. Before writing a file, check the line endings of nearby existing files in the same repo and match them. If different files in the repo use different styles, follow the ones closest to where your new file lives, or the repo's `.gitattributes` or editor config if it sets one. The goal is that your new file does not show up as changed just because of line endings.

## Hard rules - never break

- Git is read only. Only `git status` and `git diff` (and other read only checks like `git log`, `git show`, `git branch --list`). Never commit, push, branch, checkout, reset, stash, merge, or rebase. Leave the work as uncommitted changes for the user to review, commit, and open PRs themselves. This holds in every repo and every session.
- Never change the machine (installs, global settings) without asking first.
- The PR files are just files you write to disk. Writing them is fine. Turning them into a commit or opening the PR for the user is not.
