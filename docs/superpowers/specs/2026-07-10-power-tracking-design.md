# Native power (watts) tracking for weighted exercises

**Date:** 2026-07-10
**Status:** Approved design, pre-implementation
**Destination:** Upstream PR to LiamMorrow/LiftLog, developed and sideload-tested on the jhoblitt/LiftLog fork first. A GitHub Discussion (feature-request vetting per CONTRIBUTING.md) is posted before/alongside implementation.

## Problem

Keiser functional trainers (and similar equipment) display the peak power (watts)
achieved in a set. LiftLog has no concept of power, forcing workarounds such as
parallel "&lt;exercise&gt; power" exercises whose "weight" is really watts — which
distorts total-weight-lifted stats and clutters graphs.

## Feature summary

- A per-exercise **"Track power"** toggle on weighted exercise blueprints
  (default off).
- When a set of a power-tracked exercise first transitions from not-done to
  done, a **skippable popup** asks for the max power (whole watts). Power is
  always optional.
- Recorded watts display on the **set tile** (small line under the weight,
  `312 W`, or `– W` when unset) in all contexts (workout, history, feed,
  shared).
- Power is editable after the fact via the set's **long-press dialog**.
- Stats: a **max-power-over-time** graph per exercise.

## Decisions made during brainstorming

| Question | Decision |
|---|---|
| Destination | Upstream PR quality; discussion first, build in parallel on fork |
| Entry UX | Auto-popup on the not-done → done transition (skippable) |
| Opt-in | Per-exercise blueprint toggle |
| Value | Single integer watts per set (peak power) |
| Stats v1 | Max power per session over time graph |
| Historical kludge data | Private one-off backup-rewrite script, NOT part of the PR |
| Display | Watts line on set tile |
| Validation | Sideload release build on Android before opening upstream PR |

## 1. Data model

- `RecordedSetJSON` (app/src/models/storage/versions/latest/session.ts) gains
  `power?: number | undefined` — integer watts, ≥ 0 (`@asType integer`).
- `RecordedSet` (app/src/models/session-models/recorded-weighted-exercise.ts)
  gains `readonly power: number | undefined`; `fromJSON`/`toJSON`/`equals`/
  `with` updated. Power lives only on a completed set, so power on an
  uncompleted set is structurally impossible.
- `WeightedExerciseBlueprintJSON` gains `trackPower?: boolean`;
  `WeightedExerciseBlueprint` gains `readonly trackPower: boolean`, default
  `false` (`?? false` in `fromJSON`, `false` in `empty()`; included in
  `equals`/`with`/`toJSON`).
- **No storage migration, no version bump.** Both fields are optional; old data
  loads untouched. The backend needs no changes (feed payloads are
  client-side-encrypted blobs; no recorded-session model exists in C#).
- **Cross-version semantics (stated honestly):** older app versions *read*
  newer data fine (unknown JSON fields are dropped at `fromJSON`), but any
  older-client *rewrite* — restoring a newer backup on an old version and then
  saving, running a downgraded app against a newer DB, or a follower's device
  re-persisting a received feed session — silently drops `power`/`trackPower`
  from its local copy. This cannot be prevented from this PR: shipped clients
  have no unknown-version rejection (`migrateUntil` passes
  newer-than-known versions through unchanged) and no unknown-field
  preservation, so neither a version bump nor lossless round-tripping would
  protect against them retroactively. It is also the app's established
  behavior for every additive schema change (cardio exercises and their
  optional per-set fields, e.g. `steps`, were added within version 2 the same
  way). The upstream discussion post must call this caveat out explicitly so
  the maintainer can impose a different policy if desired.
- Rep-editing paths must preserve power:
  - `withCycledRepCount` decrement goes through `RecordedSet.with()` — safe
    once `with` carries power.
  - `withRepCount` currently constructs `new RecordedSet(reps, time)` — must
    preserve existing power when the set was already completed.
  - `withNothingCompleted` clears the whole `RecordedSet`, taking power with
    it — correct.
- New model members: `RecordedWeightedExercise.withPower(setIndex, power |
  undefined)` and a `maxPower: number | undefined` getter.

## 2. Completion popup

- `weighted-exercise.tsx` `onTap` already distinguishes the not-done → done
  transition (it inspects previous/new set for the rest timer). When that
  transition fires and `blueprint.trackPower` and not readonly, open a new
  `PowerDialog` for that set index after dispatching the rep update.
- `PowerDialog` (new, app/src/components/presentation/foundation/editors/):
  trimmed clone of `WeightDialog` — Paper Dialog, integer-only numeric
  TextInput, no unit toggle. Title "Max power"; **Skip** and **Save** actions.
  Input starts empty; placeholder shows this exercise's most recently recorded
  power in the current session, if any.
- Skip or dismiss leaves the set completed with `power: undefined`.
- Taps that cycle the rep count down or un-complete the set never re-open the
  dialog.

## 3. Editing after the fact (and the second completion path)

- `PotentialSetAdditionalActionsDialog` (long-press on a set) gains a power
  section when `trackPower`: a watts input pre-filled with the recorded value,
  clearable to unset. This is also how power is added to a set whose popup was
  skipped.
- The input is always shown for power-tracked exercises — **not** gated on the
  set already being completed — because this dialog is itself a completion
  path: entering a rep count for an uncompleted set completes it via
  `withRepCount`. Reps and power are applied together on save, so completing a
  set through this dialog captures power directly, with no popup chaining. If
  the dialog is saved with reps unset (set uncompleted), any entered power is
  discarded — power lives on `RecordedSet`, so power without completion is
  structurally impossible.
- The auto-popup (§2) therefore remains wired to the tap path only; both
  completion paths offer power entry at the moment of completion.

## 4. Display

- `PotentialSetCounter` lower section: when `trackPower`, render a small watts
  line under the weight (`312 W` / `– W`). Because the same component renders
  workout, history, feed, and shared views, display is automatic in readonly
  contexts.
- `ExerciseSummary` (previous-exercise viewer/summaries) gains power display so
  past-session power is visible mid-workout.

## 5. Exercise editor

- `WeightedExerciseEditor` (app/src/components/presentation/workout-editor/
  exercise-editor.tsx) gains a `SegmentedListSwitch` "Track power" (icon:
  `lightning-bolt`), mirroring the existing superset toggle, dispatching
  `updateExercise` with `blueprint.with({ trackPower })`. Precedent: the cardio
  editor's `trackDuration`/`trackDistance`/… switches.

### Plan-diff propagation

- `app/src/models/blueprint-diff.ts` maintains an exhaustive per-field diff of
  weighted-exercise blueprints (`ExerciseFieldChange` union: sets, reps,
  progressive overload, rest, superset, notes, link), with matching apply
  logic and i18n labels. It feeds the finish-workout "update your plan?" flow
  (`getPlanDiff` → diff-save modal): once `WeightedExerciseBlueprint.equals`
  learns `trackPower`, a mid-session toggle change would make blueprints
  unequal while producing an **empty** diff — a phantom "changes" prompt whose
  save applies nothing, so the toggle never reaches the program and every
  future workout re-prompts.
- Therefore `trackPower` must be added end-to-end: a new
  `ExerciseFieldChange` union member (`exerciseTrackPower`), diff detection,
  apply logic, and i18n diff label — with a test asserting a
  trackPower-only blueprint change yields a non-empty diff that round-trips
  through apply.

## 6. Stats

- `ExerciseStatAcc` (app/src/store/stats/calculate-stats.ts) accumulates
  `maxPowerStatistics: TimeTrackedStatistic<number>[]` — best watts across a
  session's sets; sessions with no power data are skipped.
- `WeightedExerciseStatistics` (app/src/store/stats/index.ts) gains a numeric
  max-power-over-time series (numeric analog of the Weight-typed series,
  following the `RepsBreakdownStatistics` precedent).
- `expanded-weighted-exercise.tsx` renders a "Max power" `StatCardWithTitle`
  with a `PowerLineChart` (clone of `weight-line-chart.tsx` without unit
  conversion), shown only when the exercise has at least one recorded power
  value.

## 7. i18n

- New keys in `app/src/i18n/en.json` only (Weblate handles other locales):
  e.g. `exercise.track_power.label`, `exercise.select_power.title`,
  `exercise.power.skip`, watts formatting, `stats.exercise.max_power.title`.

## 8. Error handling & edge cases

- Watts input: digits only, ≥ 0; empty input = unset (`undefined`), never `0`.
  `0` is stored only when explicitly typed.
- Turning `trackPower` off hides the UI but never deletes recorded power;
  turning it back on reveals it again.
- Older-version data (fields absent) is covered by `fromJSON` defaults;
  feed/backup round-trips covered by serialization tests.

## 9. Testing

- Model specs (`recorded-weighted-exercise.spec.ts`): power survives rep
  cycling/editing; completing an uncompleted set via `withRepCount` can set
  power in the same operation; JSON round-trip; `equals`; `withPower`;
  `maxPower`. `__test__/helpers.ts` `filledPotentialSet` learns an optional
  power arg.
- Blueprint specs: `trackPower` default, round-trip, equality.
- Blueprint-diff specs (`blueprint-diff.spec.ts`): a trackPower-only change
  produces a non-empty diff; applying it writes the toggle to the program
  blueprint.
- Stats specs (`calculate-stats.spec.ts`): max-power series across sessions;
  sessions without power skipped.
- Runner: vitest (`npm test` in app/). Lint/format: oxlint/oxfmt per
  CONTRIBUTING.md.
- Manual validation: release APK built from the fork branch, sideloaded onto
  the user's Android device, exercising the full flow (toggle → popup → tile →
  long-press edit → completing a set from the long-press rep dialog →
  plan-diff prompt after a mid-session toggle → stats graph → history) before
  the upstream PR opens. (The repo has no component-test harness, so dialog
  behavior is validated here rather than in vitest.)

## 10. Deliverables & sequencing

1. **GitHub Discussion draft** — feature pitch (problem, proposed UX,
   data-model sketch) posted to LiamMorrow/LiftLog discussions for vetting;
   implementation proceeds in parallel.
2. **Implementation** on `feature/power-tracking` in the jhoblitt/LiftLog
   fork, per this spec.
3. **Sideload testing** on Android from a fork build; iterate until the flow
   is right.
4. **Upstream PR** against LiamMorrow/LiftLog `main` once validated (and
   discussion feedback incorporated). Before opening the PR, drop this spec's
   commit from the branch (rebase it out) — the spec is working material, not
   PR content.
5. **Private backfill script** (separate from the PR): reads a LiftLog backup
   export and folds each "X power" kludge exercise into the real "X" exercise,
   then removes the kludge entries. Hard requirements, since recorded sessions
   embed their own blueprint copies and all power UI is gated on
   `blueprint.trackPower`:
   - Match kludge → target **within the same session** by exercise name
     (never by date across sessions); abort with a report on any ambiguity
     (e.g. duplicate same-named exercises in one session, or set-count
     mismatch between "X power" and "X").
   - Write `power` per set positionally (kludge set N's weight-as-watts →
     target set N) and set `trackPower: true` on the **embedded blueprint** of
     every rewritten recorded exercise, plus the current program blueprint —
     otherwise backfilled power is invisible and uneditable.
   - Never modify the source backup; write a new file, produce a dry-run
     diff report first, and verify the output re-imports cleanly (and that a
     re-export round-trips) before pointing the app at it.
