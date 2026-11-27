# 2026-10-15 - cutout-rule-hole-counts (backend)

> **Linked PR (frontend):** https://github.com/liviktech/rubine-toughened-glass-frontend/pull/149

## Summary

The AI was still missing cutouts on sheets where glasses sit next to each other and reach the ground, the ceiling or the roof. This PR adds that rule to the prompt: when two or more glasses meet and a glass's edge at that meeting touches the ground, ceiling or roof, that glass has a cutout there, and it is counted in each glass that reaches it. It also records a comparison of four reads of the same sheet on three Gemini models: `gemini-3.5-flash` and `gemini-3.6-flash` performed well, `gemini-3.5-flash-lite` did not (see "Notes").

The prompt text was built with the real `buildOrderSheetInstruction()` and checked to be identical to the copy in the docs. The new rule was **not** tested live against a sheet on its own in this PR, see "Test plan".

## What changed

| File | Change |
|---|---|
| `src/shared/vision-ai/instruction.ts` | New rule in "HOLES AND CUTOUTS: THE BORDER TEST" for glasses that meet each other and touch the ground, ceiling or roof. The analysis step now also asks the model to say, for every joint between two glasses, whether it reaches the ground, ceiling or roof. Two new short examples (two glasses meeting at the floor, and a single glass alone on the floor). Nothing else in the prompt, the reply shape or the schema changed. |

## Why

The earlier prompt work (border test for holes and cutouts) did not cover cutouts that come from how glasses are fitted together. On installation views, where several panes stand side by side on a floor, the cutouts at the joints were left out or undercounted. The rule comes from the factory side: a glass whose edge touches the ground, ceiling or roof where it meets another glass has a cutout there.

## How

**One new rule, placed next to the shared-edge rule**: it says each glass that reaches the ground, ceiling or roof at a joint carries its own cutout, so it is counted in each glass and not once for the pair. A single glass standing alone on the floor does not get a cutout from this rule.

**Look at every joint**: the analysis step now lists, for each joint between two glasses, whether it reaches the ground, ceiling or roof, so the check is written down before the counts.

**When nothing is drawn**: if no mark is drawn but the situation is clear, the model still counts the cutout, adds `cutoutQty` to `unsureFields`, and says in `note` that it comes from this rule. This keeps staff informed and fits the existing rule that a flagged value gets reviewed.

**Kept the same on purpose**: the reply JSON shape, the schema, the four sides, the hole versus cutout border test, and the database. Stored readings are keyed by image bytes, so photos already read keep their old result until **Read again** is used.

## Decisions made

**Count the cutout even when no mark is drawn.** The alternative was to count it only when a mark is drawn. The rule was given as "these places will have the cutouts", so the model counts it and flags it for review. If this turns out to add cutouts that are not there, the sentence can be removed without touching anything else.

**One rule, not a change to the border test.** The border test is about what a single drawn mark is. This new rule is about where cutouts are expected between glasses, so it is its own bullet.

## Notes

**Model comparison on one sheet, four reads.** The same photo (a door with a transom and two fixed panels, 4 panes, sheet total 4) was read four times. The files are in `rubine-toughened-glass-backend/uploads/order-photos/2026/10/` (`f4bbf4e2-...reading copy*.json`). Two of the five files are exact copies of another read, so there are four reads. Times are India time.

| Run | Time | Model | Cutouts per pane (left fixed, transom, door, right fixed) | Total cutouts | Flagged unsure |
|---|---|---|---|---|---|
| 1 | 13:09 | `gemini-3.5-flash` | 4, 4, 3, 4 | 15 | 1 pane (door height) |
| 2 | 13:38 | `gemini-3.6-flash` | 4, 4, 3, 5 | 16 | 1 pane (door height) |
| 3 | 13:39 | `gemini-3.5-flash-lite` | 2, 2, 2, 2 | 8 | all 4 panes |
| 4 | 13:41 | `gemini-3.5-flash-lite` | blank, blank, blank, blank | 0 | 2 panes |

- **Result:** `gemini-3.5-flash` and `gemini-3.6-flash` performed well. `gemini-3.5-flash-lite` did not: the two reads disagree with each other (every pane at 2 cutouts and flagged, then every cutout count blank), so it is not reliable for this job.
- **What matched:** all four reads found the same 4 panes with the same sizes (inch values and mm), and all matched the sheet total of 4.
- **The only difference between the two good models:** the right fixed panel, 4 cutouts on `gemini-3.5-flash` and 5 on `gemini-3.6-flash`.
- **The model listed the panes in a different order** in runs 3 and 4, so panes were matched by size, not by row position.
- **Limits of this test:** one sheet and four reads. The "performed well" judgement is the team's review of the reads, not a score against a typed answer sheet. The files do not record the prompt text, the token counts or the read time in seconds, so this comparison cannot say which prompt version each run used, or compare cost and speed. The runs were made after the new rule was added to `instruction.ts`, but that is not recorded in the files.
- **Workbook:** a comparison workbook with a quick view, a field by field view and the raw rows was made from these files (`ai-extraction-runs-comparison.xlsx`, kept outside the repos).
- The feature docs were updated in the same task: `dev-docs/toughened-glass/features/ai-extraction/` (developer guide, prompt file, README and pricing page).

## Next steps

- Set `AI_MODEL` to `gemini-3.5-flash` or `gemini-3.6-flash` (the local `.env` was last seen on `gemini-3.5-flash-lite`).
- Run more sheets through both good models, with a typed answer sheet, to see how often the 4 versus 5 cutout difference on the right fixed panel happens, and to record time and token cost.
- Run the new rule on a sheet where glasses meet the floor and the ceiling on purpose, and check that a lone glass gets no extra cutout.
- Run the other vendors (OpenAI, Anthropic, Mistral, Qwen) on the same sheets. Only Gemini has been run.

## Test plan

**Automated tests (no automated test suite exists in this repo):**
- [x] `npx tsc --noEmit`: clean, no errors.

**Manual checks (local):**
- [x] Built the prompt with the real `buildOrderSheetInstruction()` and wrote it to the docs. The new rule, the analysis line and the two examples are all in the generated text, and the docs copy is identical to the code output.
- [x] Compared four real reads of one sheet on three Gemini models (see "Notes").
- [ ] A read made for this PR to prove the new rule on its own - not run in this PR. The four reads above were made after the rule was added but the files do not record the prompt used, so they are not proof of the rule by themselves.
- [ ] Other vendors with the new rule - not run, only Gemini is configured and keyed on this machine.
