# 2026-11-23 - question-images-plan-done (frontend)

## Summary

This PR changes four things in the mobile app. Question images from the backend now show in practice and mock exams. The Hub has a new "getting started" layout for students who have not finished anything yet, and no longer shows streaks or the exam focus card. The study plan has a new look (today as big cards, later days in one grouped card). Long questions on the Stats page get a "Read more" link.

The plan screen also refreshes after a practice session finishes, so a planned session the backend ticked off automatically (see the backend PR) shows as done on the Hub and the Plan tab.

I have not run the app on a device or an emulator. I ran the type check, lint and a full Android bundle (`expo export`), and they passed.

**Lottie loading animation fix is not included on purpose** - see "Notes" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/services/question-image.ts` (new) | The `QuestionImage` type (id, url, width, height, altText) |
| `src/services/practice-api.ts`, `src/services/mock-exam-api.ts` | New fields from the backend: `stemImages`, `passageImages`, option `images`, `explanationImages` |
| `src/components/ui/remote-image.tsx` (new) | One picture at full width. Reserves the space from the stored size, shows a soft block while loading, a "tap to try again" on failure, and can be tapped |
| `src/components/ui/image-viewer.tsx` (new) | Full screen view of one image on a dark background, closes on tap or the close button |
| `src/components/question/question-images.tsx` (new) | Stacks a list of images and opens the viewer. Draws nothing when there are none |
| `src/components/question/question-stem.tsx` (new) | Question text with its pictures, used by four screens |
| `src/components/practice/answer-option.tsx` | Options can show images under the text. Not tappable to enlarge, because a tap selects the option |
| `src/components/practice/passage-card.tsx`, `explanation-card.tsx` | Show passage and explanation images |
| `src/app/practice/[sessionId].tsx`, `src/components/mock-exam/exam-runner.tsx`, `exam-review-item.tsx`, `src/components/practice/review-item.tsx` | Use `QuestionStem` and pass the images to options, passage and explanation |
| `src/hooks/use-practice.ts` | After a session is submitted, the study plan query is refreshed |
| `src/components/hub/getting-started-hub.tsx` (new) | The new Hub layout: header, "Up next" card, Today tiles and plan, Target score and Daily goal tiles |
| `src/app/(tabs)/hub.tsx` | Shows the new layout until the student has a finished practice session or mock exam. Removed the streak chip and text and the exam focus card from the whole Hub |
| `src/components/hub/focus-card.tsx` | deleted, nothing used it any more |
| `src/components/plan/plan-day-section.tsx` | Today: heading with a "done" chip. Other days: one card with the day name, done count and dividers |
| `src/components/plan/plan-item-row.tsx` | Two looks: `featured` (card with round checkbox, Start, Skip, bin) and `compact` (row with small Start and a "..." menu) |
| `src/components/plan/plan-check.tsx`, `plan-item-info.tsx` (new) | The round done checkbox, and the subject, topic and time block |
| `src/app/(tabs)/plan.tsx` | Passes `featured` for today and `compact` for later days |
| `src/components/ui/pill-button.tsx` (new) | Small rounded button used for Start and Skip |
| `src/components/ui/text-button.tsx` | New `link` colour option |
| `src/components/ui/expandable-text.tsx` (new) | Text cut to a few lines with "Read more" and "Show less", only when the text is really longer |
| `src/components/dashboard/question-lists.tsx` | Question text on the Stats page uses the expandable text |
| `src/utils/plan-format.ts` | New `itemParts` that splits a session into subject and topic |

## Why

Admins can upload images for questions, but students could not see them, so some questions were not answerable. The Hub, plan and Stats changes come from design and usability requests: the old plan list looked crowded, the Hub showed a streak and focus card that were no longer wanted, and long questions on Stats were cut off with no way to read them.

## How

**Images**: Each picture is drawn at the full width of its box. The height is reserved from the stored width and height, so the page does not jump when the image loads. Tall images are capped at a maximum height and shown whole, not cropped. `expo-image` caches them on the phone. Tapping a question or explanation image opens it full screen.

**Hub choice**: The new layout shows when there is no finished practice and no finished mock exam. The Hub waits for the mock exam history first, so it does not flash one layout and switch to the other. Everything on it comes from real data: the unfinished session, today's plan, the target score and the daily minutes from onboarding.

**Plan look**: Today's sessions are separate cards with big actions. Later days are rows in a single card with a small Start and a menu for Skip and Remove.

**Read more**: The text is drawn twice, once cut and once invisible and uncut, because a cut text cannot tell how long it would have been. The link only shows when the full text needs more lines than the limit.

**Kept the same on purpose**: The "Today's Plan" block on the Hub reuses the same plan row, so it picks up the new look. Option text and letters, and the way answers are sent, did not change.

## Decisions made

**Streak is removed from the Hub, not hidden.** The mockup had a Streak tile, but the request was to remove the streak logic, so the Streak tile was replaced by a Target score tile. The backend still returns `streakDays`, the app just does not use it on the Hub.

**Only real data in the new Hub.** The mockup had a live class, a unit, a bell and a 45 of 60 minute goal. The app has no data for those, so they were left out. Daily goal shows the planned minutes per day, not minutes studied today, because the app does not record that.

**Option images cannot be enlarged.** A tap on an option selects the answer, so a second tap target inside it would be confusing.

**The "..." menu uses a system alert.** I did not build a custom menu for Skip and Remove.

## Notes

- The app crashed on open on one build with "Can't find ViewManager LottieAnimationView". That comes from the earlier study plan work adding `lottie-react-native`, not from this PR. The installed app has to be rebuilt after that package was added. The Android build also kept an old autolinking cache that had to be deleted (`android/build/generated/autolinking/autolinking.json`), and `./gradlew clean` failed with a CMake error. I did not change code for this.
- `src/components/auth/auth-shell.tsx` had an uncommitted brand colour change on my machine earlier. It is not part of this PR.

## Next steps

- Run the app on a device and check: a question with images in practice and in a mock exam review, the new Hub for a student with nothing finished, the plan screen, and "Read more" on Stats.
- Check a real uploaded image from the CDN or bucket.
- Check that Hub and plan show a planned session as done after a perfect practice session, once the backend PR is deployed.

## Test plan

**Automated tests (no test suite in this app, only type check, lint and bundle):**
- [x] `npx tsc --noEmit`: no errors
- [x] `npx expo lint`: no warnings
- [x] `npx expo export --platform android`: bundles without errors

**Manual checks (local):**
- [ ] Question, option, passage and explanation images on a device - not done, I could not run the app. The backend sends them, covered by the backend PR tests with a fake CDN
- [ ] New Hub layout and the switch to the old layout after the first finished session - not done on a device
- [ ] New plan layout with Start, Skip and the "..." menu - not done on a device
- [ ] "Read more" on long questions in Stats, including that it does not show on short ones - not done on a device
- [ ] Plan item turning to done after a perfect practice session - not done, needs the backend change running against the app
