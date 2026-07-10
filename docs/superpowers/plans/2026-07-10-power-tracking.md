# Native Power (Watts) Tracking Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add optional per-set peak-power (watts) tracking to weighted exercises: a per-exercise `trackPower` toggle, a skippable popup on set completion, watts on the set tile, long-press editing, and a max-power-over-time stats graph.

**Architecture:** `power` becomes an optional field on `RecordedSet` (a completed set's record) and `trackPower` an optional-with-default-false flag on `WeightedExerciseBlueprint`. Both serialize as optional JSON fields inside schema version 2 (no migration — house precedent: cardio fields). UI hangs off `blueprint.trackPower`: the workout screen pops a `PowerDialog` on the not-done→done tap transition, the long-press rep dialog gains a power input (the second completion path), and stats grow a numeric max-power series.

**Tech Stack:** React Native/Expo, TypeScript, Redux Toolkit, react-native-paper, Tolgee i18n, vitest, oxlint/oxfmt + eslint, ts-json-schema-generator (workout-worker schemas → Kotlin codegen via symlink).

**Spec:** `docs/superpowers/specs/2026-07-10-power-tracking-design.md` (approved; includes adversarial-review revisions).

## Global Constraints

- Run ALL commands from `/home/jhoblitt/github/LiftLog/app` unless a step says otherwise.
- Branch: `feature/power-tracking`. Never push to `origin` (upstream LiamMorrow/LiftLog); the push remote is `jhoblitt`.
- Test commands: single file `npx vitest run <path>`; full suite `npx vitest run`. (`npm test` starts watch mode — do not use in automation.)
- Quality gates: `npm run typecheck` (tsgo), `npm run lint` (oxlint && eslint), `npm run format` (oxfmt --write .).
- After changing ANY `*JSON` type reachable from `app/src/models/workout-worker-messages.ts` or `app/src/models/storage/versions/latest/ai-plan.ts` (this includes `RecordedSetJSON` and `WeightedExerciseBlueprintJSON`): run `npm run json-schema` and commit the resulting `docs/schemas/**` changes IN THE SAME COMMIT as the type change. The Android module's Kotlin codegen reads these schemas through a symlink (`app/modules/workout-worker/android/src/main/resources/schema` → `docs/schemas/workout-worker`).
- Watts are whole integers ≥ 0. `power` exists only on completed sets (`RecordedSet`); `trackPower` defaults to `false` everywhere old data or missing JSON is involved.
- i18n: add ENGLISH keys only (`app/src/i18n/en.json`); Weblate handles other locales. Keys are lowercase dot-separated (`a–z 0–9 . _`). The Tolgee `TranslationKey` type derives from `en.json`, so a missing key fails `npm run typecheck`.
- There is NO component-test harness. UI tasks are verified by typecheck + lint + full vitest (models/store) and ultimately the sideload validation task. Do not invent a component harness.
- Commit messages: match repo style — `feat:`/`fix:`/`test:`/`docs:` prefix, capitalized subject (e.g. `feat: Add power field to RecordedSet`). End the body with `Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>`.
- Out of scope for this plan: the private backfill script, actually posting the GitHub Discussion, opening the upstream PR, and dropping the `docs/superpowers/*` commits (that happens at PR-prep time).

## File Structure

| File | Change |
|---|---|
| `app/src/models/storage/versions/latest/session.ts` | `RecordedSetJSON.power?` |
| `app/src/models/storage/versions/latest/blueprint.ts` | `WeightedExerciseBlueprintJSON.trackPower?` |
| `app/src/models/session-models/recorded-weighted-exercise.ts` | `RecordedSet.power`, `withPower`, preservation in `withRepCount`, `maxPower`, `latestRecordedPower` |
| `app/src/models/session-models/__test__/helpers.ts` | `filledPotentialSet` power arg |
| `app/src/models/blueprint-models/index.ts` | `WeightedExerciseBlueprint.trackPower` |
| `app/src/models/blueprint-diff.ts` | `exerciseTrackPower` change kind end-to-end |
| `app/src/components/presentation/foundation/editors/power-dialog.tsx` | NEW: watts entry dialog |
| `app/src/components/presentation/workout/weighted/potential-set-counter.tsx` | watts line on tile; pass-through props |
| `app/src/components/presentation/workout/weighted/weighted-exercise.tsx` | popup wiring; reps+power handler |
| `app/src/components/presentation/workout/weighted/potential-sets-addition-actions-dialog.tsx` | power input (second completion path) |
| `app/src/components/presentation/workout-editor/exercise-editor.tsx` | "Track Power" switch |
| `app/src/components/presentation/summary/exercise-summary.tsx` | power in chips |
| `app/src/store/stats/index.ts` | `NumericStatisticOverTime`, stats field |
| `app/src/store/stats/calculate-stats.ts` | max-power series |
| `app/src/components/presentation/stats/power-line-chart.tsx` | NEW: watts line chart |
| `app/src/app/stats/expanded-weighted-exercise.tsx` | Max Power card |
| `app/src/i18n/en.json` | 6 new keys |
| `docs/schemas/**` | regenerated |
| `docs/superpowers/2026-07-10-power-tracking-discussion.md` | NEW: discussion draft (working material) |

Specs touched: `recorded-weighted-exercise.spec.ts`, `blueprint-models/index.spec.ts`, `blueprint-diff.spec.ts`, `calculate-stats.spec.ts`.

---

### Task 1: Baseline verification and upstream discussion draft

**Files:**
- Create: `docs/superpowers/2026-07-10-power-tracking-discussion.md`

**Interfaces:**
- Consumes: nothing.
- Produces: a green baseline (all later tasks assume it) and the discussion text the user will post to https://github.com/LiamMorrow/LiftLog/discussions.

- [ ] **Step 1: Install dependencies and verify green baseline**

Run:
```bash
cd /home/jhoblitt/github/LiftLog/app
npm ci
npx vitest run
npm run typecheck
```
Expected: `npm ci` completes; vitest reports all test files passed, 0 failed; typecheck exits 0. If the baseline is red, STOP and report — do not build on a red baseline.

- [ ] **Step 2: Write the discussion draft**

Create `docs/superpowers/2026-07-10-power-tracking-discussion.md` (repo root path, not under app/) with exactly this content:

```markdown
# Feature request: optional per-set power (watts) tracking for weighted exercises

## Problem

Some strength equipment reports the peak power produced during a set — e.g.
Keiser functional trainers display max watts per set on their digital readout.
LiftLog has no concept of power, so the only way to keep the number is a
workaround: a parallel exercise like "Chest Press power" with 1-rep sets whose
"weight" is really watts. That distorts total-weight-lifted stats, clutters
the exercise list, and graphs the value in the wrong unit.

## Proposal

Optional, per-exercise power tracking:

- A **"Track Power" toggle** on weighted exercise blueprints (default off),
  next to the superset toggle in the exercise editor — same pattern as the
  cardio `trackDuration`/`trackDistance` flags.
- When a set of a power-tracked exercise is first marked done, a small
  **skippable dialog** asks for the max power (whole watts). Power is always
  optional — Skip just closes it.
- The recorded watts show as a small line on the **set tile** (`312 W`), and
  can be edited later via the set's long-press dialog (which also captures
  power when a set is completed from that dialog's rep entry).
- Stats: a **Max Power over time** chart on the exercise stats page, shown
  only for exercises that have recorded power.

## Data model

- `RecordedSetJSON` gains optional `power?: number` (integer watts).
- `WeightedExerciseBlueprintJSON` gains optional `trackPower?: boolean`.
- Both stay inside schema version 2 with no migration, following the
  precedent of the cardio per-set fields. Old data loads unchanged;
  `fromJSON` defaults `trackPower` to `false`.
- No backend changes: feed payloads are client-side-encrypted blobs and the
  C# models don't include recorded sessions.

## Compatibility caveat (calling it out explicitly)

Older app versions *read* newer data fine — unknown JSON fields are dropped at
parse. But an older client that loads-then-saves newer data (restoring a newer
backup on an old version and editing it, or a downgraded install) rewrites it
without the new fields, silently dropping recorded power in its local copy.
That is the app's existing behavior for every additive schema change (cardio
fields were added the same way), and shipped clients have no
unknown-version rejection, so it can't be prevented retroactively — but
flagging it so a different policy can be imposed if wanted.

## Implementation

I have a working branch implementing the above (model + JSON + regenerated
workout-worker schemas, plan-diff propagation for the toggle, UI, stats,
tests) that I'm validating on-device; happy to open a PR if this is a
direction you'd accept, and to adjust the design per feedback.
```

- [ ] **Step 3: Commit**

```bash
cd /home/jhoblitt/github/LiftLog
git add docs/superpowers/2026-07-10-power-tracking-discussion.md
git commit -m "docs: Draft upstream discussion post for power tracking (working material)

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 2: `RecordedSet.power` — model, JSON, schemas

**Files:**
- Modify: `app/src/models/storage/versions/latest/session.ts` (RecordedSetJSON, ~line 52)
- Modify: `app/src/models/session-models/recorded-weighted-exercise.ts` (RecordedSet class, ~line 212)
- Modify: `app/src/models/session-models/__test__/helpers.ts` (`filledPotentialSet`, ~line 83)
- Test: `app/src/models/session-models/recorded-weighted-exercise.spec.ts`
- Regenerated: `docs/schemas/workout-worker/*.json` (at minimum `RecordedSet.json`)

**Interfaces:**
- Consumes: existing `RecordedSet(repsCompleted: number, completionDateTime: OffsetDateTime)`.
- Produces: `RecordedSet` constructor `(repsCompleted: number, completionDateTime: OffsetDateTime, power: number | undefined = undefined)`; `RecordedSet.power: number | undefined`; `RecordedSetJSON.power?: number | undefined`; `filledPotentialSet(reps: number, time: OffsetDateTime, weight?: Weight, power?: number)`. All existing 2-arg `new RecordedSet(...)` call sites keep compiling (default param).

- [ ] **Step 1: Write the failing tests**

Append to `app/src/models/session-models/recorded-weighted-exercise.spec.ts` (it already imports `RecordedSet` and `tick`):

```ts
describe('RecordedSet.power', () => {
  it('defaults power to undefined', () => {
    expect(new RecordedSet(10, tick()).power).toBeUndefined();
  });

  it('stores power and handles it in with()', () => {
    const set = new RecordedSet(10, tick(), 312);
    expect(set.power).toBe(312);
    expect(set.with({ repsCompleted: 9 }).power).toBe(312);
    expect(set.with({ power: 400 }).power).toBe(400);
    expect(set.with({ power: undefined }).power).toBeUndefined();
  });

  it('round-trips power through JSON and defaults to undefined when absent', () => {
    const time = tick();
    const withPower = new RecordedSet(8, time, 250);
    const roundTripped = RecordedSet.fromJSON(withPower.toJSON());
    expect(roundTripped.equals(withPower)).toBe(true);
    expect(roundTripped.power).toBe(250);

    const withoutPower = new RecordedSet(8, time);
    expect(withoutPower.toJSON().power).toBeUndefined();
    expect(RecordedSet.fromJSON(withoutPower.toJSON()).power).toBeUndefined();
  });

  it('includes power in equality', () => {
    const time = tick();
    const a = new RecordedSet(10, time, 312);
    expect(a.equals(new RecordedSet(10, time, 312))).toBe(true);
    expect(a.equals(new RecordedSet(10, time, 300))).toBe(false);
    expect(a.equals(new RecordedSet(10, time))).toBe(false);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run src/models/session-models/recorded-weighted-exercise.spec.ts`
Expected: the new `RecordedSet.power` tests FAIL (`power` is undefined / not equal); pre-existing tests still pass.

- [ ] **Step 3: Implement**

In `app/src/models/storage/versions/latest/session.ts`, replace the `RecordedSetJSON` interface:

```ts
export interface RecordedSetJSON {
  /**
   * @asType integer
   */
  repsCompleted: number;
  completionDateTime: OffsetDateTimeJSON;
  /**
   * Peak power in whole watts achieved during the set, as reported by
   * equipment with a power readout (e.g. Keiser functional trainers).
   * @asType integer
   */
  power?: number | undefined;
}
```

In `app/src/models/session-models/recorded-weighted-exercise.ts`, replace the `RecordedSet` class:

```ts
export class RecordedSet {
  constructor(
    readonly repsCompleted: number,
    readonly completionDateTime: OffsetDateTime,
    readonly power: number | undefined = undefined,
  ) {}

  static fromJSON(json: RecordedSetJSON): RecordedSet {
    return new RecordedSet(json.repsCompleted, fromOffsetDateTimeJSON(json.completionDateTime), json.power);
  }

  equals(other: RecordedSet | undefined): boolean {
    if (!other) {
      return false;
    }
    if (other === this) {
      return true;
    }
    return (
      this.repsCompleted === other.repsCompleted &&
      this.completionDateTime.equals(other.completionDateTime) &&
      this.power === other.power
    );
  }

  with(other: Partial<RecordedSet>): RecordedSet {
    return new RecordedSet(
      'repsCompleted' in other ? other.repsCompleted! : this.repsCompleted,
      'completionDateTime' in other ? other.completionDateTime! : this.completionDateTime,
      'power' in other ? other.power : this.power,
    );
  }

  toJSON(): RecordedSetJSON {
    return {
      repsCompleted: this.repsCompleted,
      completionDateTime: toOffsetDateTimeJSON(this.completionDateTime),
      power: this.power,
    };
  }
}
```

In `app/src/models/session-models/__test__/helpers.ts`, replace `filledPotentialSet`:

```ts
export function filledPotentialSet(
  reps: number,
  time: OffsetDateTime,
  weight = new Weight(100, 'kilograms'),
  power?: number,
) {
  return new PotentialSet(new RecordedSet(reps, time, power), weight);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run src/models/session-models/recorded-weighted-exercise.spec.ts`
Expected: PASS (all).

- [ ] **Step 5: Regenerate schemas and typecheck**

Run:
```bash
npm run json-schema
npm run typecheck
git -C /home/jhoblitt/github/LiftLog status --short docs/schemas
```
Expected: typecheck exits 0; `docs/schemas/workout-worker/RecordedSet.json` shows as modified (an optional `power` integer property). Other schema files may be rewritten byte-identically; only genuinely changed ones appear.

- [ ] **Step 6: Commit**

```bash
cd /home/jhoblitt/github/LiftLog
git add app/src/models/storage/versions/latest/session.ts app/src/models/session-models/recorded-weighted-exercise.ts app/src/models/session-models/__test__/helpers.ts app/src/models/session-models/recorded-weighted-exercise.spec.ts docs/schemas
git commit -m "feat: Add optional power field to RecordedSet

Peak power in whole watts, as displayed by equipment such as Keiser
functional trainers. Optional within schema version 2; absent fields
default to undefined so existing data is unaffected.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 3: `RecordedWeightedExercise` power operations

**Files:**
- Modify: `app/src/models/session-models/recorded-weighted-exercise.ts` (`withRepCount` ~line 107; new members near `maxWeight` ~line 148)
- Test: `app/src/models/session-models/recorded-weighted-exercise.spec.ts`

**Interfaces:**
- Consumes: `RecordedSet.power` / 3-arg constructor from Task 2; existing `withSet`, `IndexOutOfBoundsError`, `TemporalComparer`, `Enumerable`.
- Produces: `withPower(setIndex: number, power: number | undefined): RecordedWeightedExercise` (no-op when the set is uncompleted; throws `IndexOutOfBoundsError` on a bad index); `maxPower: number | undefined` (getter); `latestRecordedPower: number | undefined` (getter); `withRepCount` preserves existing power when editing reps of a completed set.

- [ ] **Step 1: Write the failing tests**

Append to `app/src/models/session-models/recorded-weighted-exercise.spec.ts`:

```ts
describe('RecordedWeightedExercise power operations', () => {
  function makeExercise() {
    return new RecordedWeightedExercise(
      makeWeightedBlueprint(),
      [
        filledPotentialSet(10, tick(), undefined, 250),
        filledPotentialSet(10, tick(), undefined, 312),
        new PotentialSet(undefined, new Weight(100, 'kilograms')),
      ],
      undefined,
    );
  }

  it('withPower sets power on a completed set', () => {
    const result = makeExercise().withPower(0, 400);
    expect(result.getSet(0).set?.power).toBe(400);
    expect(result.getSet(1).set?.power).toBe(312);
  });

  it('withPower(undefined) clears power', () => {
    expect(makeExercise().withPower(1, undefined).getSet(1).set?.power).toBeUndefined();
  });

  it('withPower is a no-op on an uncompleted set', () => {
    const exercise = makeExercise();
    const result = exercise.withPower(2, 400);
    expect(result.getSet(2).set).toBeUndefined();
    expect(result.equals(exercise)).toBe(true);
  });

  it('withRepCount preserves existing power when editing reps', () => {
    const result = makeExercise().withRepCount(1, 8, tick());
    expect(result.getSet(1).set?.repsCompleted).toBe(8);
    expect(result.getSet(1).set?.power).toBe(312);
  });

  it('withRepCount(undefined) clears the whole set including power', () => {
    expect(makeExercise().withRepCount(1, undefined, tick()).getSet(1).set).toBeUndefined();
  });

  it('withCycledRepCount decrement preserves power', () => {
    const result = makeExercise().withCycledRepCount(1, tick());
    expect(result.getSet(1).set?.repsCompleted).toBe(9);
    expect(result.getSet(1).set?.power).toBe(312);
  });

  it('withNothingCompleted drops power with the set', () => {
    const result = makeExercise().withNothingCompleted();
    expect(result.potentialSets.every((x) => x.set === undefined)).toBe(true);
  });

  it('maxPower returns the max across sets, or undefined when none recorded', () => {
    expect(makeExercise().maxPower).toBe(312);
    const noPower = new RecordedWeightedExercise(
      makeWeightedBlueprint(),
      [filledPotentialSet(10, tick())],
      undefined,
    );
    expect(noPower.maxPower).toBeUndefined();
  });

  it('latestRecordedPower returns the power of the most recently completed set that has one', () => {
    const exercise = new RecordedWeightedExercise(
      makeWeightedBlueprint(),
      [
        filledPotentialSet(10, tick(), undefined, 250),
        filledPotentialSet(10, tick(), undefined, 312),
        filledPotentialSet(10, tick()),
      ],
      undefined,
    );
    expect(exercise.latestRecordedPower).toBe(312);
    expect(makeExercise().withPower(1, undefined).latestRecordedPower).toBe(250);
    const noPower = new RecordedWeightedExercise(
      makeWeightedBlueprint(),
      [filledPotentialSet(10, tick())],
      undefined,
    );
    expect(noPower.latestRecordedPower).toBeUndefined();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run src/models/session-models/recorded-weighted-exercise.spec.ts`
Expected: FAIL — `withPower is not a function`, `maxPower`/`latestRecordedPower` undefined, and `withRepCount preserves existing power` fails (current code constructs `new RecordedSet(reps, time)` with no power).

- [ ] **Step 3: Implement**

In `app/src/models/session-models/recorded-weighted-exercise.ts`:

Replace `withRepCount` (~line 107):

```ts
  withRepCount(setIndex: number, reps: number | undefined, time: OffsetDateTime): RecordedWeightedExercise {
    return this.withSet(setIndex, (s) =>
      s.with({
        set: reps === undefined ? undefined : new RecordedSet(reps, time, s.set?.power),
      }),
    );
  }
```

Add after `withWeight` (~line 137):

```ts
  withPower(setIndex: number, power: number | undefined): RecordedWeightedExercise {
    return this.withSet(setIndex, (s) => (s.set ? s.with({ set: s.set.with({ power }) }) : s));
  }
```

Add after the `maxWeight` getter (~line 157):

```ts
  get maxPower(): number | undefined {
    const powers = this.potentialSets.map((x) => x.set?.power).filter((x): x is number => x !== undefined);
    return powers.length ? Math.max(...powers) : undefined;
  }

  get latestRecordedPower(): number | undefined {
    return Enumerable.from(this.potentialSets)
      .where((x) => x.set?.power !== undefined)
      .orderByDescending((x) => x.set?.completionDateTime, TemporalComparer)
      .firstOrDefault()?.set?.power;
  }
```

(`withCycledRepCount`'s decrement branch already routes through `RecordedSet.with()`, which preserves power since Task 2 — the test proves it.)

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run src/models/session-models/recorded-weighted-exercise.spec.ts`
Expected: PASS (all).

- [ ] **Step 5: Commit**

```bash
cd /home/jhoblitt/github/LiftLog
git add app/src/models/session-models/recorded-weighted-exercise.ts app/src/models/session-models/recorded-weighted-exercise.spec.ts
git commit -m "feat: Add power operations to RecordedWeightedExercise

withPower sets/clears watts on a completed set (no-op when
uncompleted), withRepCount now preserves recorded power when editing
reps, and maxPower/latestRecordedPower support stats and the entry
dialog placeholder.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 4: `WeightedExerciseBlueprint.trackPower` — model, JSON, schemas

**Files:**
- Modify: `app/src/models/storage/versions/latest/blueprint.ts` (`WeightedExerciseBlueprintJSON`)
- Modify: `app/src/models/blueprint-models/index.ts` (`WeightedExerciseBlueprint`, ~lines 482–563)
- Test: `app/src/models/blueprint-models/index.spec.ts`
- Regenerated: `docs/schemas/**` (workout-worker `WeightedExerciseBlueprint.json`; `ai-plan/AiPlan.json` may also change — commit whatever regenerates)

**Interfaces:**
- Consumes: nothing new.
- Produces: `WeightedExerciseBlueprint` constructor gains a 9th parameter `readonly trackPower: boolean = false`; `trackPower` participates in `empty()`, `fromJSON` (`json.trackPower ?? false`), `equals`, `toJSON`, `with`; `WeightedExerciseBlueprintJSON.trackPower?: boolean`. All existing 8-arg constructor call sites keep compiling.

- [ ] **Step 1: Write the failing tests**

Append to `app/src/models/blueprint-models/index.spec.ts` (add `WeightedExerciseBlueprint` to its imports from `'./index'` if not already imported):

```ts
describe('WeightedExerciseBlueprint.trackPower', () => {
  it('defaults to false', () => {
    expect(WeightedExerciseBlueprint.empty().trackPower).toBe(false);
  });

  it('defaults to false when absent from JSON', () => {
    const json = WeightedExerciseBlueprint.empty().toJSON();
    delete (json as { trackPower?: boolean }).trackPower;
    expect(WeightedExerciseBlueprint.fromJSON(json).trackPower).toBe(false);
  });

  it('round-trips through JSON', () => {
    const blueprint = WeightedExerciseBlueprint.empty().with({ trackPower: true });
    expect(WeightedExerciseBlueprint.fromJSON(blueprint.toJSON()).trackPower).toBe(true);
  });

  it('participates in equality', () => {
    const off = WeightedExerciseBlueprint.empty();
    const on = off.with({ trackPower: true });
    expect(off.equals(on)).toBe(false);
    expect(on.equals(off.with({ trackPower: true }))).toBe(true);
  });

  it('with() sets and clears the flag', () => {
    const on = WeightedExerciseBlueprint.empty().with({ trackPower: true });
    expect(on.trackPower).toBe(true);
    expect(on.with({ trackPower: false }).trackPower).toBe(false);
    expect(on.with({ sets: 5 }).trackPower).toBe(true);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run src/models/blueprint-models/index.spec.ts`
Expected: FAIL — `trackPower` is undefined (not `false`), equality test fails.

- [ ] **Step 3: Implement**

In `app/src/models/storage/versions/latest/blueprint.ts`, add to `WeightedExerciseBlueprintJSON` after the `progressiveOverload` member:

```ts
  /**
   * When true, the app prompts for and records the peak power (watts) of
   * each completed set, for equipment with a power readout
   * (e.g. Keiser functional trainers).
   */
  trackPower?: boolean;
```

In `app/src/models/blueprint-models/index.ts`, update `WeightedExerciseBlueprint`:

Constructor — add the 9th parameter:

```ts
  constructor(
    readonly name: string,
    readonly sets: number,
    readonly repsPerSet: number,
    readonly progressiveOverload: ProgressiveOverload,
    readonly restBetweenSets: Rest,
    readonly supersetWithNext: boolean,
    readonly notes: string,
    readonly link: string,
    readonly trackPower: boolean = false,
  ) {}
```

`empty()`:

```ts
  static empty() {
    return new WeightedExerciseBlueprint('', 3, 10, new NoProgressiveOverload(), Rest.medium, false, '', '', false);
  }
```

`fromJSON` — add as the last constructor argument:

```ts
      json.trackPower ?? false,
```

`equals` — add to the returned conjunction, after the `this.link === other.link` line:

```ts
      this.link === other.link &&
      this.trackPower === other.trackPower
```

`toJSON` — add after `link`:

```ts
      trackPower: this.trackPower,
```

`with` — add as the last constructor argument:

```ts
      other.trackPower ?? this.trackPower,
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `npx vitest run src/models/blueprint-models/index.spec.ts`
Expected: PASS (all).

- [ ] **Step 5: Regenerate schemas, typecheck, full test run**

Run:
```bash
npm run json-schema
npm run typecheck
npx vitest run
git -C /home/jhoblitt/github/LiftLog status --short docs/schemas
```
Expected: all green; `docs/schemas/workout-worker/WeightedExerciseBlueprint.json` modified (optional boolean `trackPower`); `docs/schemas/ai-plan/AiPlan.json` may be modified too — include it if so.

- [ ] **Step 6: Commit**

```bash
cd /home/jhoblitt/github/LiftLog
git add app/src/models/storage/versions/latest/blueprint.ts app/src/models/blueprint-models/index.ts app/src/models/blueprint-models/index.spec.ts docs/schemas
git commit -m "feat: Add trackPower flag to WeightedExerciseBlueprint

Optional-with-default-false within schema version 2, so existing
programs and shared data are unaffected.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 5: Plan-diff propagation for `trackPower`

**Files:**
- Modify: `app/src/models/blueprint-diff.ts` (new change interface after `ExerciseSupersetChange` ~line 137; `ExerciseFieldChange` union ~line 227; `diffWeightedExercises` after the superset block ~line 475; apply match ~line 908; `getChangeDescription` superset area; `getChangeLabelKey` ~line 1140)
- Modify: `app/src/i18n/en.json` (one key)
- Test: `app/src/models/blueprint-diff.spec.ts`

**Interfaces:**
- Consumes: `WeightedExerciseBlueprint.trackPower` (Task 4).
- Produces: `ExerciseTrackPowerChange` (`kind: 'exerciseTrackPower'`, `oldValue`/`newValue: boolean`) flowing through diff → filter → apply → label/description, so mid-session toggle changes reach the program via the finish-workout "update your plan?" flow.

- [ ] **Step 1: Write the failing test**

Append inside the top-level `describe('diffSessionBlueprints', ...)` block of `app/src/models/blueprint-diff.spec.ts` (it already defines `createWeightedExercise` and imports `diffSessionBlueprints`, `applySessionBlueprintDiff`, `getChangeLabelKey`, `SessionBlueprint`, `WeightedExerciseBlueprint`):

```ts
  it('detects, labels, and applies a trackPower-only change', () => {
    const oldExercise = createWeightedExercise('Bench Press');
    const newExercise = oldExercise.with({ trackPower: true });
    const original = new SessionBlueprint('Push Day', [oldExercise], '');
    const modified = new SessionBlueprint('Push Day', [newExercise], '');

    const diff = diffSessionBlueprints(original, modified);

    expect(diff.hasChanges).toBe(true);
    expect(diff.modifiedExercises).toHaveLength(1);
    const changes = diff.modifiedExercises[0]!.changes;
    expect(changes).toHaveLength(1);
    expect(changes[0]).toEqual(
      expect.objectContaining({
        kind: 'exerciseTrackPower',
        oldValue: false,
        newValue: true,
      }),
    );
    expect(getChangeLabelKey(changes[0]!)).toEqual({
      key: 'plan.diff.track_power.label',
    });

    const result = applySessionBlueprintDiff(original, diff);
    expect((result.exercises[0] as WeightedExerciseBlueprint).trackPower).toBe(true);
  });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run src/models/blueprint-diff.spec.ts`
Expected: FAIL — `changes` is empty (`toHaveLength(1)` fails): `equals` sees the difference but `diffWeightedExercises` emits nothing. This is exactly the phantom-empty-diff bug from the spec.

- [ ] **Step 3: Implement**

In `app/src/models/blueprint-diff.ts`:

(a) After the `ExerciseSupersetChange` interface (~line 137), add:

```ts
interface ExerciseTrackPowerChange extends BaseChange {
  kind: 'exerciseTrackPower';
  type: 'modified';
  exerciseName: string;
  exerciseIndex: number;
  oldValue: boolean;
  newValue: boolean;
}
```

(b) In the `ExerciseFieldChange` union, after `| ExerciseSupersetChange`:

```ts
  | ExerciseTrackPowerChange
```

(c) In `diffWeightedExercises`, after the `supersetWithNext` block (~line 475):

```ts
  if (oldEx.trackPower !== newEx.trackPower) {
    changes.push({
      id: generateChangeId(),
      kind: 'exerciseTrackPower',
      type: 'modified',
      exerciseName,
      exerciseIndex,
      oldValue: oldEx.trackPower,
      newValue: newEx.trackPower,
    });
  }
```

(d) In `applySessionBlueprintDiff`'s change match, after the `exerciseSuperset` arm (~line 908):

```ts
        .with({ kind: 'exerciseTrackPower' }, (c) =>
          exercise instanceof WeightedExerciseBlueprint ? exercise.with({ trackPower: c.newValue }) : exercise,
        )
```

(e) In `getChangeDescription`, after the `exerciseSuperset` arm:

```ts
    .with({ kind: 'exerciseTrackPower' }, (c) =>
      t(c.newValue ? 'plan.diff.generic_enabled.body' : 'plan.diff.generic_disabled.body'),
    )
```

(f) In `getChangeLabelKey`, after the `exerciseSuperset` arm:

```ts
    .with({ kind: 'exerciseTrackPower' }, () => ({
      key: 'plan.diff.track_power.label',
    }))
```

In `app/src/i18n/en.json`, insert between the `"plan.diff.track_incline.label"` and `"plan.diff.track_resistance.label"` lines:

```json
  "plan.diff.track_power.label": "Power",
```

- [ ] **Step 4: Run test and typecheck to verify**

Run:
```bash
npx vitest run src/models/blueprint-diff.spec.ts
npm run typecheck
```
Expected: tests PASS; typecheck exits 0. If typecheck reports any OTHER `.exhaustive()` match over `DiffChange` (e.g. in a component), add an `exerciseTrackPower` arm there following the same superset pattern — the three arms above are the only known sites.

- [ ] **Step 5: Commit**

```bash
cd /home/jhoblitt/github/LiftLog
git add app/src/models/blueprint-diff.ts app/src/models/blueprint-diff.spec.ts app/src/i18n/en.json
git commit -m "feat: Propagate trackPower through the plan diff

Without this, a mid-session toggle change makes blueprints unequal
while producing an empty diff, so the finish-workout plan-update flow
shows a phantom prompt and never persists the toggle.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 6: `PowerDialog` component

**Files:**
- Create: `app/src/components/presentation/foundation/editors/power-dialog.tsx`
- Modify: `app/src/i18n/en.json` (two keys)

**Interfaces:**
- Consumes: `Button` gesture wrapper, react-native-paper `Dialog`/`Portal`/`TextInput`, Tolgee `T`.
- Produces: `PowerDialog` default export with props `{ open: boolean; power: number | undefined; placeholder: number | undefined; onClose: () => void; updatePower: (power: number | undefined) => void; }`. Skip (or dismiss) calls only `onClose`. Save calls `updatePower(watts)` — `undefined` when the field is empty — then `onClose`. Input accepts only integers ≥ 0.

- [ ] **Step 1: Add the i18n keys**

In `app/src/i18n/en.json`:
- Insert immediately before the `"exercise.select_reps.title"` line:

```json
  "exercise.select_power.title": "Max Power",
```

- Insert immediately after the `"generic.save.button": "Save",` line:

```json
  "generic.skip.button": "Skip",
```

- [ ] **Step 2: Create the component**

Create `app/src/components/presentation/foundation/editors/power-dialog.tsx`:

```tsx
import { spacing } from '@/hooks/useAppTheme';
import { T } from '@tolgee/react';
import { useEffect, useState } from 'react';
import { View } from 'react-native';
import Button from '@/components/presentation/foundation/gesture-wrappers/button';
import { Dialog, Portal, TextInput, useTheme } from 'react-native-paper';
import { KeyboardAvoidingView } from 'react-native-keyboard-controller';

interface PowerDialogProps {
  open: boolean;
  power: number | undefined;
  placeholder: number | undefined;
  onClose: () => void;
  updatePower: (power: number | undefined) => void;
}

export default function PowerDialog(props: PowerDialogProps) {
  const theme = useTheme();
  const [text, setText] = useState(props.power?.toString() ?? '');

  useEffect(() => {
    setText(props.power?.toString() ?? '');
  }, [props.open, props.power]);

  const parsed = Number(text);
  const isValid = !text || (Number.isInteger(parsed) && parsed >= 0);

  const onSaveClick = () => {
    if (!isValid) {
      return;
    }
    props.updatePower(text ? parsed : undefined);
    props.onClose();
  };

  return (
    props.open && (
      <Portal>
        <KeyboardAvoidingView
          behavior={'height'}
          style={{ flex: 1, pointerEvents: props.open ? 'box-none' : 'none' }}
        >
          <Dialog visible={props.open} onDismiss={props.onClose}>
            <Dialog.Title>
              <T keyName="exercise.select_power.title" />
            </Dialog.Title>
            <Dialog.Content>
              <View style={{ gap: spacing[2] }}>
                <TextInput
                  testID="power-input"
                  selectTextOnFocus
                  mode="outlined"
                  inputMode="numeric"
                  keyboardType="number-pad"
                  submitBehavior="blurAndSubmit"
                  returnKeyType="done"
                  autoFocus
                  value={text}
                  error={!isValid}
                  placeholder={props.placeholder?.toString()}
                  onChangeText={setText}
                  right={<TextInput.Affix text="W" />}
                  style={{ backgroundColor: theme.colors.elevation.level3 }}
                />
              </View>
            </Dialog.Content>
            <Dialog.Actions>
              <Button onPress={props.onClose} testID="power-skip">
                <T keyName="generic.skip.button" />
              </Button>
              <Button onPress={onSaveClick} testID="power-save" disabled={!isValid}>
                <T keyName="generic.save.button" />
              </Button>
            </Dialog.Actions>
          </Dialog>
        </KeyboardAvoidingView>
      </Portal>
    )
  );
}
```

- [ ] **Step 3: Verify (no component harness — typecheck + lint)**

Run:
```bash
npm run typecheck
npm run lint
```
Expected: both exit 0.

- [ ] **Step 4: Commit**

```bash
cd /home/jhoblitt/github/LiftLog
git add app/src/components/presentation/foundation/editors/power-dialog.tsx app/src/i18n/en.json
git commit -m "feat: Add PowerDialog for entering max power in watts

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 7: Set tile watts display + completion popup wiring

**Files:**
- Modify: `app/src/components/presentation/workout/weighted/potential-set-counter.tsx`
- Modify: `app/src/components/presentation/workout/weighted/weighted-exercise.tsx`

**Interfaces:**
- Consumes: `PowerDialog` (Task 6); `withPower`, `latestRecordedPower` (Task 3); `blueprint.trackPower` (Task 4).
- Produces: `PotentialSetCounterProps` gains `trackPower: boolean` (watts line renders when true, in all contexts including readonly). `WeightedExercise` opens `PowerDialog` when a tap transitions a set from not-done to done on a power-tracked exercise. NOTE: this task does NOT change the `onUpdateReps` signature — Task 8 does.

- [ ] **Step 1: Add the watts line to `PotentialSetCounter`**

In `app/src/components/presentation/workout/weighted/potential-set-counter.tsx`:

(a) Add to `PotentialSetCounterProps` after `isReadonly: boolean;`:

```ts
  trackPower: boolean;
```

(b) In the bottom section of the tile, insert the watts line after the closing `</TouchableRipple>` of the weight ripple (the one with `testID="repcount-weight"`), still inside the surrounding `<View>`:

```tsx
            {props.trackPower && (
              <Text
                testID="repcount-power"
                style={{ color: colors.onSurface, textAlign: 'center', ...font['text-sm'] }}
              >
                {props.set.set?.power !== undefined ? `${props.set.set.power} W` : '– W'}
              </Text>
            )}
```

- [ ] **Step 2: Wire the popup in `WeightedExercise`**

Replace the body of `app/src/components/presentation/workout/weighted/weighted-exercise.tsx` with:

```tsx
import PotentialSetCounter from '@/components/presentation/workout/weighted/potential-set-counter';
import PowerDialog from '@/components/presentation/foundation/editors/power-dialog';
import { spacing } from '@/hooks/useAppTheme';
import { RecordedWeightedExercise } from '@/models/session-models';
import { useState } from 'react';
import { View } from 'react-native';
import ExerciseSection from '@/components/presentation/workout/exercise-section';
import { OffsetDateTime } from '@js-joda/core';
import { Updater } from '@/utils/types';

interface WeightedExerciseProps {
  recordedExercise: RecordedWeightedExercise;
  previousRecordedExercises: RecordedWeightedExercise[];
  toStartNext: boolean;
  isReadonly: boolean;
  showPreviousButton: boolean;

  timeProvider: () => OffsetDateTime;
  updateExercise: (update: Updater<RecordedWeightedExercise>) => void;
  resetSetTimer: () => void;
  onEditExercise: () => void;
  onRemoveExercise: () => void;
}

export default function WeightedExercise(props: WeightedExerciseProps) {
  const { updateExercise, timeProvider, resetSetTimer } = props;
  const { recordedExercise } = props;
  const [powerDialogIndex, setPowerDialogIndex] = useState<number | undefined>(undefined);

  const trackPower = recordedExercise.blueprint.trackPower;
  const setToStartNext = recordedExercise.potentialSets.findIndex((x) => !x.set);

  return (
    <ExerciseSection
      recordedExercise={props.recordedExercise}
      previousRecordedExercises={props.previousRecordedExercises}
      toStartNext={props.toStartNext}
      isReadonly={props.isReadonly}
      showPreviousButton={props.showPreviousButton}
      updateExercise={props.updateExercise}
      onEditExercise={props.onEditExercise}
      onRemoveExercise={props.onRemoveExercise}
    >
      <View style={{ flexDirection: 'row', gap: spacing[2], flexWrap: 'wrap' }}>
        {recordedExercise.potentialSets.map((set, index) => (
          <PotentialSetCounter
            isReadonly={props.isReadonly}
            key={index}
            maxReps={recordedExercise.blueprint.repsPerSet}
            trackPower={trackPower}
            onTap={() => {
              const previousSet = set.set;
              const newSet = recordedExercise.withCycledRepCount(index, timeProvider()).getSet(index).set;
              updateExercise((ex) => ex.withCycledRepCount(index, timeProvider()));
              // We only want to reset the timer when switching between unfilled and filled
              // Otherwise, keep the same time
              if (!previousSet || !newSet) {
                resetSetTimer();
              }
              if (trackPower && !previousSet && newSet) {
                setPowerDialogIndex(index);
              }
            }}
            previousRepCount={props.previousRecordedExercises.at(0)?.potentialSets[index]?.set?.repsCompleted}
            onUpdateReps={(reps) => {
              updateExercise((ex) => ex.withRepCount(index, reps, timeProvider()));
              resetSetTimer();
            }}
            onUpdateWeight={(w, applyTo) => updateExercise((ex) => ex.withWeight(index, w, applyTo))}
            set={set}
            toStartNext={props.toStartNext && setToStartNext === index && !props.isReadonly}
            weightIncrement={recordedExercise.blueprint.progressiveOverload.weightIncrement}
          />
        ))}
      </View>
      <PowerDialog
        open={powerDialogIndex !== undefined}
        power={
          powerDialogIndex !== undefined ? recordedExercise.potentialSets[powerDialogIndex]?.set?.power : undefined
        }
        placeholder={recordedExercise.latestRecordedPower}
        onClose={() => setPowerDialogIndex(undefined)}
        updatePower={(power) => {
          const index = powerDialogIndex;
          if (index !== undefined) {
            updateExercise((ex) => ex.withPower(index, power));
          }
        }}
      />
    </ExerciseSection>
  );
}
```

(Notes: the stray `useState(false);` line in the current file is replaced by the real dialog state. The popup only opens on the `!previousSet && newSet` transition, so rep-decrement taps never re-open it. `isReadonly` never reaches here with taps enabled — `onTap` is disabled in the counter.)

- [ ] **Step 3: Fix the other `PotentialSetCounter` call site**

`PotentialSetCounter` is also rendered by `app/src/components/presentation/workout-editor/progressive-overload.tsx` (preview usage). Run:

```bash
npm run typecheck
```

Expected: an error there for the missing `trackPower` prop. Add `trackPower={false}` to that `<PotentialSetCounter ... />` usage, then re-run `npm run typecheck` — exits 0.

- [ ] **Step 4: Lint and full test run**

Run:
```bash
npm run lint
npx vitest run
```
Expected: both green.

- [ ] **Step 5: Commit**

```bash
cd /home/jhoblitt/github/LiftLog
git add app/src/components/presentation/workout/weighted/potential-set-counter.tsx app/src/components/presentation/workout/weighted/weighted-exercise.tsx app/src/components/presentation/workout-editor/progressive-overload.tsx
git commit -m "feat: Show watts on set tiles and prompt for power on completion

Power-tracked exercises render a watts line under the set weight and
open the PowerDialog when a tap first completes a set. Skipping the
dialog leaves the set completed with no power recorded.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 8: Long-press dialog power entry (second completion path)

**Files:**
- Modify: `app/src/components/presentation/workout/weighted/potential-sets-addition-actions-dialog.tsx`
- Modify: `app/src/components/presentation/workout/weighted/potential-set-counter.tsx` (prop pass-through, signature)
- Modify: `app/src/components/presentation/workout/weighted/weighted-exercise.tsx` (handler)
- Modify: `app/src/i18n/en.json` (one key)

**Interfaces:**
- Consumes: `withRepCount` power preservation + `withPower` (Task 3); `trackPower` prop (Task 7).
- Produces: `updateRepCount`/`onUpdateReps` signatures become `(reps: number | undefined, power: number | undefined) => void` through the dialog → counter → exercise chain. The dialog shows a power input whenever `showPower` is true (pre-filled from the set), so completing a set by typing reps captures power in the same save with no popup chaining. When `trackPower` is false the handler never touches power.

- [ ] **Step 1: Add the i18n key**

In `app/src/i18n/en.json`, insert immediately before the first `"exercise.progressive_overload.` line:

```json
  "exercise.power.label": "Power",
```

- [ ] **Step 2: Rewrite the dialog**

Replace the entire contents of `app/src/components/presentation/workout/weighted/potential-sets-addition-actions-dialog.tsx` with:

```tsx
import { spacing, useAppTheme } from '@/hooks/useAppTheme';
import { PotentialSet } from '@/models/session-models';
import { T } from '@tolgee/react';
import { useEffect, useState } from 'react';
import { View } from 'react-native';
import { KeyboardAvoidingView } from 'react-native-keyboard-controller';
import IconButton from '@/components/presentation/foundation/gesture-wrappers/icon-button';
import { Dialog, Portal, Text, TextInput } from 'react-native-paper';
import Button from '@/components/presentation/foundation/gesture-wrappers/button';

interface PotentialSetAdditionalActionsDialogProps {
  open: boolean;
  set: PotentialSet;
  repTarget: number;
  showPower: boolean;
  updateRepCount: (reps: number | undefined, power: number | undefined) => void;
  close: () => void;
}

export default function PotentialSetAdditionalActionsDialog({
  close,
  open,
  set,
  updateRepCount,
  repTarget,
  showPower,
}: PotentialSetAdditionalActionsDialogProps) {
  const { colors } = useAppTheme();
  const originalReps = set?.set?.repsCompleted;
  const originalPower = set?.set?.power;

  const [repCountText, setRepCountText] = useState<string>(originalReps?.toString() ?? '');
  const [powerText, setPowerText] = useState<string>(originalPower?.toString() ?? '');
  const parsedRepCount = Number(repCountText);
  const isValid = !repCountText || (Number.isInteger(parsedRepCount) && parsedRepCount >= 0);
  const parsedPower = Number(powerText);
  const isPowerValid = !powerText || (Number.isInteger(parsedPower) && parsedPower >= 0);
  useEffect(() => {
    setRepCountText(originalReps?.toString() ?? '');
  }, [originalReps]);
  useEffect(() => {
    setPowerText(originalPower?.toString() ?? '');
  }, [originalPower]);

  const powerValue = () => (powerText && isPowerValid ? parsedPower : undefined);

  const save = () => {
    if (!isValid || !isPowerValid) {
      return;
    }

    updateRepCount(repCountText ? parsedRepCount : undefined, powerValue());
    close();
  };
  return (
    open && (
      <Portal>
        <KeyboardAvoidingView behavior={'height'} style={{ flex: 1, pointerEvents: open ? 'box-none' : 'none' }}>
          <Dialog visible={open} onDismiss={close}>
            <Dialog.Title>
              <T keyName="exercise.select_reps.title" />
            </Dialog.Title>
            <Dialog.Content>
              <TextInput
                label={<T keyName="exercise.reps.label" />}
                inputMode="numeric"
                value={repCountText}
                selectTextOnFocus
                error={!isValid}
                onChangeText={setRepCountText}
                autoFocus
              />

              <View style={{ flexDirection: 'row', flexWrap: 'wrap' }}>
                {Array.from({ length: repTarget + 3 }).map((_, i) => (
                  <IconButton
                    key={i}
                    mode="outlined"
                    icon={() => <Text>{i}</Text>}
                    onPress={() => {
                      setRepCountText(i.toString());
                      updateRepCount(i, powerValue());
                      close();
                    }}
                  />
                ))}
                <IconButton
                  mode="contained"
                  iconColor={colors.error}
                  containerColor={colors.errorContainer}
                  icon={'close'}
                  onPress={() => {
                    setRepCountText('');
                    setPowerText('');
                    updateRepCount(undefined, undefined);
                    close();
                  }}
                />
              </View>
              {showPower && (
                <TextInput
                  label={<T keyName="exercise.power.label" />}
                  inputMode="numeric"
                  value={powerText}
                  selectTextOnFocus
                  error={!isPowerValid}
                  onChangeText={setPowerText}
                  right={<TextInput.Affix text="W" />}
                  style={{ marginTop: spacing[2] }}
                />
              )}
            </Dialog.Content>
            <Dialog.Actions>
              <Button onPress={close}>{<T keyName="generic.cancel.button" />}</Button>
              <Button disabled={!isValid || !isPowerValid} onPress={save}>
                {<T keyName="generic.save.button" />}
              </Button>
            </Dialog.Actions>
          </Dialog>
        </KeyboardAvoidingView>
      </Portal>
    )
  );
}
```

- [ ] **Step 3: Thread the new signature through `PotentialSetCounter`**

In `app/src/components/presentation/workout/weighted/potential-set-counter.tsx`:

(a) Change the props member:

```ts
  onUpdateReps: (reps: number | undefined, power: number | undefined) => void;
```

(b) Update the dialog usage at the bottom of the component:

```tsx
      <PotentialSetAdditionalActionsDialog
        open={isRepsDialogOpen}
        repTarget={props.maxReps}
        set={props.set}
        showPower={props.trackPower}
        updateRepCount={(reps, power) => props.onUpdateReps(reps, power)}
        close={() => setIsRepsDialogOpen(false)}
      />
```

- [ ] **Step 4: Apply reps+power together in `WeightedExercise`**

In `app/src/components/presentation/workout/weighted/weighted-exercise.tsx`, replace the `onUpdateReps` prop:

```tsx
            onUpdateReps={(reps, power) => {
              updateExercise((ex) => {
                let next = ex.withRepCount(index, reps, timeProvider());
                if (trackPower) {
                  next = next.withPower(index, reps === undefined ? undefined : power);
                }
                return next;
              });
              resetSetTimer();
            }}
```

(When `trackPower` is false, `withRepCount`'s built-in preservation keeps any legacy power value intact and the handler never overwrites it.)

- [ ] **Step 5: Typecheck, lint, full tests**

Run:
```bash
npm run typecheck
npm run lint
npx vitest run
```
Expected: all green (model behavior is covered by Task 3's tests; the dialog itself has no harness).

- [ ] **Step 6: Commit**

```bash
cd /home/jhoblitt/github/LiftLog
git add app/src/components/presentation/workout/weighted/potential-sets-addition-actions-dialog.tsx app/src/components/presentation/workout/weighted/potential-set-counter.tsx app/src/components/presentation/workout/weighted/weighted-exercise.tsx app/src/i18n/en.json
git commit -m "feat: Capture power from the long-press rep dialog

Entering a rep count in the long-press dialog is the second way a set
gets completed, so the dialog now carries a power input for
power-tracked exercises and applies reps and watts in one save.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 9: Exercise editor "Track Power" toggle

**Files:**
- Modify: `app/src/components/presentation/workout-editor/exercise-editor.tsx` (`WeightedExerciseEditor` SegmentedList items)
- Modify: `app/src/i18n/en.json` (one key)

**Interfaces:**
- Consumes: `WeightedExerciseBlueprint.trackPower` + `with({ trackPower })` (Task 4); existing `SegmentedListSwitch`, `updateExercise`.
- Produces: a "Track Power" switch in the weighted exercise editor (used by both the program editor and the mid-session exercise editor, which share this component).

- [ ] **Step 1: Add the i18n key**

In `app/src/i18n/en.json`, insert between the `"exercise.track_incline.label"` and `"exercise.track_resistance.label"` lines:

```json
  "exercise.track_power.label": "Track Power",
```

- [ ] **Step 2: Add the switch**

In `app/src/components/presentation/workout-editor/exercise-editor.tsx`, inside `WeightedExerciseEditor`'s `<SegmentedList items={[...]}>`, insert between the superset `SegmentedListSwitch` (key `2`) and the progressive-overload `SegmentListFormElement` (key `3`):

```tsx
            <SegmentedListSwitch
              key="trackPower"
              label={t('exercise.track_power.label')}
              icon={'flashOn'}
              value={exercise.trackPower}
              testID="exercise-track-power"
              onValueChange={(trackPower) => updateExercise({ trackPower })}
            />,
```

(String key avoids colliding with the numeric keys already used in this array. The `flashOn` icon name is already used elsewhere — `app/src/app/settings/index.tsx`.)

- [ ] **Step 3: Typecheck and lint**

Run:
```bash
npm run typecheck
npm run lint
```
Expected: both exit 0.

- [ ] **Step 4: Commit**

```bash
cd /home/jhoblitt/github/LiftLog
git add app/src/components/presentation/workout-editor/exercise-editor.tsx app/src/i18n/en.json
git commit -m "feat: Add Track Power toggle to the weighted exercise editor

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 10: Power in exercise summaries

**Files:**
- Modify: `app/src/components/presentation/summary/exercise-summary.tsx` (`WeightAndRepsChipData`, `getWeightAndRepsChips`, `FilledChips`)

**Interfaces:**
- Consumes: `RecordedSet.power` (Task 2).
- Produces: completed-set chips in summaries (previous-exercise viewer, session summaries, feed cards) append `312 W` when a set has power. Planned chips are untouched (power is never planned).

- [ ] **Step 1: Implement**

In `app/src/components/presentation/summary/exercise-summary.tsx`:

(a) Add to `WeightAndRepsChipData`:

```ts
interface WeightAndRepsChipData {
  repsCompleted: number | undefined;
  repTarget: number;
  weight: Weight;
  power: number | undefined;
}
```

(b) In `getWeightAndRepsChips`, add `power` to the mapped object:

```ts
function getWeightAndRepsChips(exercise: RecordedWeightedExercise): WeightAndRepsChipData[] {
  return exercise.potentialSets.map((set) => ({
    repsCompleted: set.set?.repsCompleted,
    repTarget: exercise.blueprint.repsPerSet,
    weight: set.weight,
    power: set.set?.power,
  }));
}
```

(c) In `FilledChips`'s weighted branch, add a power fragment after the `showWeight` fragment (inside the same `<Chip>`):

```tsx
        {chip.power !== undefined ? (
          <SurfaceText font="text-2xs" color="onSurface">
            {chip.power} W
          </SurfaceText>
        ) : undefined}
```

- [ ] **Step 2: Typecheck, lint, full tests**

Run:
```bash
npm run typecheck
npm run lint
npx vitest run
```
Expected: all green.

- [ ] **Step 3: Commit**

```bash
cd /home/jhoblitt/github/LiftLog
git add app/src/components/presentation/summary/exercise-summary.tsx
git commit -m "feat: Show recorded power in exercise summary chips

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 11: Max-power stats calculation

**Files:**
- Modify: `app/src/store/stats/index.ts` (new interface + field)
- Modify: `app/src/store/stats/calculate-stats.ts` (accumulator, per-session push, numeric helper, result mapping)
- Test: `app/src/store/stats/calculate-stats.spec.ts`

**Interfaces:**
- Consumes: `RecordedWeightedExercise.maxPower` (Task 3).
- Produces: `NumericStatisticOverTime { statistics: TimeTrackedStatistic<number>[]; currentValue: number; maxValue: number; minValue: number; }` exported from `@/store/stats`; `WeightedExerciseStatistics.maxPowerPerSessionStatistics: NumericStatisticOverTime | undefined` (undefined when the exercise has no recorded power in range).

- [ ] **Step 1: Write the failing tests**

Append to `app/src/store/stats/calculate-stats.spec.ts` (top-level; the file already imports `Session`, `PotentialSet`, `RecordedWeightedExercise`, `RecordedSet`, `Weight`, `LocalDate`, `LocalDateRange`, `calculateStats` and defines `makeBlueprint`, `makeSessionBlueprint`, `makeOffset`, `makeSession`):

```ts
describe('max power statistics', () => {
  function makePoweredSession(date: LocalDate, powers: (number | undefined)[]): Session {
    const blueprint = makeBlueprint('Chest Press', powers.length, 10);
    const sessionBlueprint = makeSessionBlueprint('Keiser Day', [blueprint]);
    const baseTime = makeOffset(date);
    const potentialSets = powers.map(
      (power, i) =>
        new PotentialSet(new RecordedSet(10, baseTime.plusSeconds(i * 60), power), new Weight(40, 'kilograms')),
    );
    const exercise = new RecordedWeightedExercise(blueprint, potentialSets, undefined);
    return new Session('session-' + date.toString(), sessionBlueprint, [exercise], date, undefined, undefined);
  }

  const range: LocalDateRange = {
    from: LocalDate.of(2025, 4, 1),
    to: LocalDate.of(2025, 4, 30),
  };

  it('collects best power per session over time', () => {
    const stats = calculateStats(
      [
        makePoweredSession(LocalDate.of(2025, 4, 7), [250, 312, 290]),
        makePoweredSession(LocalDate.of(2025, 4, 14), [280, 330, undefined]),
      ],
      'kilograms',
      range,
    );

    const exerciseStats = stats.weightedExerciseStats.find((x) => x.exerciseName === 'Chest Press')!;
    const power = exerciseStats.maxPowerPerSessionStatistics!;
    expect(power.statistics.map((x) => x.value)).toEqual([312, 330]);
    expect(power.maxValue).toBe(330);
    expect(power.minValue).toBe(312);
    expect(power.currentValue).toBe(330);
  });

  it('skips sessions with no power and is undefined when the exercise never has power', () => {
    const stats = calculateStats(
      [
        makePoweredSession(LocalDate.of(2025, 4, 7), [undefined, undefined, undefined]),
        makePoweredSession(LocalDate.of(2025, 4, 14), [undefined, 300, undefined]),
        makeSession(LocalDate.of(2025, 4, 21), 'Squat', 100),
      ],
      'kilograms',
      range,
    );

    const chestPress = stats.weightedExerciseStats.find((x) => x.exerciseName === 'Chest Press')!;
    expect(chestPress.maxPowerPerSessionStatistics!.statistics.map((x) => x.value)).toEqual([300]);

    const squat = stats.weightedExerciseStats.find((x) => x.exerciseName === 'Squat')!;
    expect(squat.maxPowerPerSessionStatistics).toBeUndefined();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run src/store/stats/calculate-stats.spec.ts`
Expected: FAIL — `maxPowerPerSessionStatistics` does not exist (TypeScript error surfaces as a failing run).

- [ ] **Step 3: Implement**

In `app/src/store/stats/index.ts`:

(a) After the `WeightedStatisticOverTime` interface, add:

```ts
export interface NumericStatisticOverTime {
  statistics: TimeTrackedStatistic<number>[];
  currentValue: number;
  maxValue: number;
  minValue: number;
}
```

(b) Add to `WeightedExerciseStatistics` after `totalVolumeStatistics`:

```ts
  maxPowerPerSessionStatistics: NumericStatisticOverTime | undefined;
```

In `app/src/store/stats/calculate-stats.ts`:

(c) Add `NumericStatisticOverTime` to the `@/store/stats` import list.

(d) Add to the `ExerciseStatAcc` interface after `totalVolumeStatistics`:

```ts
    maxPowerStatistics: TimeTrackedStatistic<number>[];
```

and to the `exerciseStatsMap.set(key, { ... })` initializer:

```ts
          maxPowerStatistics: [],
```

(e) After the `exerciseStats.totalVolumeStatistics.push({ ... });` block, add:

```ts
      const maxPower = ex.maxPower;
      if (maxPower !== undefined) {
        exerciseStats.maxPowerStatistics.push({
          dateTime: lastSet.set!.completionDateTime,
          value: maxPower,
        });
      }
```

(f) In the final `exerciseStats` mapping, add after `totalVolumeStatistics: ...`:

```ts
      maxPowerPerSessionStatistics: ex.maxPowerStatistics.length
        ? unsortedStatsToNumericStatisticOverTime(ex.maxPowerStatistics)
        : undefined,
```

(g) Next to `unsortedStatsToWeightedStatisticOverTime` (bottom of file), add:

```ts
function unsortedStatsToNumericStatisticOverTime(
  unsortedStats: TimeTrackedStatistic<number>[],
): NumericStatisticOverTime {
  const statistics = Enumerable.from(unsortedStats)
    .orderBy((x) => x.dateTime.toString())
    .toArray();
  const values = statistics.map((x) => x.value);
  return {
    statistics,
    currentValue: values.at(-1) ?? 0,
    maxValue: values.length ? Math.max(...values) : 0,
    minValue: values.length ? Math.min(...values) : 0,
  };
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run:
```bash
npx vitest run src/store/stats/calculate-stats.spec.ts
npm run typecheck
```
Expected: PASS; typecheck exits 0.

- [ ] **Step 5: Commit**

```bash
cd /home/jhoblitt/github/LiftLog
git add app/src/store/stats/index.ts app/src/store/stats/calculate-stats.ts app/src/store/stats/calculate-stats.spec.ts
git commit -m "feat: Calculate max power per session in exercise stats

Sessions without recorded power are skipped; exercises with no power
at all get an undefined series so the UI can omit the chart.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 12: Max-power stats UI

**Files:**
- Create: `app/src/components/presentation/stats/power-line-chart.tsx`
- Modify: `app/src/app/stats/expanded-weighted-exercise.tsx`
- Modify: `app/src/i18n/en.json` (one key)

**Interfaces:**
- Consumes: `NumericStatisticOverTime` + `maxPowerPerSessionStatistics` (Task 11); `lineGraphProps`, `useFormatDate`.
- Produces: `PowerLineChart({ statistics: NumericStatisticOverTime })` named export; a "Max Power" card on the expanded exercise stats page, rendered only when power data exists.

- [ ] **Step 1: Add the i18n key**

In `app/src/i18n/en.json`, insert immediately after the `"stats.exercise.max_weight.title"` line:

```json
  "stats.exercise.max_power.title": "Max Power",
```

- [ ] **Step 2: Create the chart component**

Create `app/src/components/presentation/stats/power-line-chart.tsx` (a numeric clone of `weight-line-chart.tsx` — no unit conversion, no negative handling since watts are ≥ 0, ` W` suffix):

```tsx
import { NumericStatisticOverTime } from '@/store/stats';
import { LineChart, lineDataItem } from 'react-native-gifted-charts';
import { View } from 'react-native';
import { spacing, useAppTheme } from '@/hooks/useAppTheme';
import { useEffect, useState } from 'react';
import { lineGraphProps } from '@/components/presentation/stats/line-graph-props';
import { useFormatDate } from '@/hooks/useFormatDate';
import { Text } from 'react-native-paper';

export function PowerLineChart({
  statistics: { statistics, maxValue, minValue },
}: {
  statistics: NumericStatisticOverTime;
}) {
  const formatDate = useFormatDate();
  const { colors } = useAppTheme();
  const points: lineDataItem[] = statistics.map((stat): lineDataItem => {
    const label = formatDate(stat.dateTime.toLocalDate(), {
      day: 'numeric',
      month: 'short',
    });
    return {
      value: stat.value,
      label,
      focusedDataPointLabelComponent: () => <FocusedDatapointLabelComponent value={stat.value} label={label} />,
    };
  });
  const [width, setWidth] = useState(0);
  // On android the area chart renders poorly unless it is delayed until after initial render
  const [areaChart, setAreaChart] = useState(false);
  useEffect(() => {
    setAreaChart(!!width);
  }, [width]);
  return (
    <View onLayout={(e) => setWidth(e.nativeEvent.layout.width)}>
      <LineChart
        {...lineGraphProps(colors, width, points.length)}
        showFractionalValues={false}
        dataPointLabelWidth={70}
        showReferenceLine1
        areaChart={areaChart}
        delayBeforeUnFocus={10_000}
        referenceLine1Position={maxValue}
        dataSet={[
          {
            data: points,
            strokeDashArray: [1],
            dataPointsColor: colors.primary,
            color: colors.primary,
            dataPointsRadius: 5,
            startFillColor: colors.primary,
            endFillColor: colors.primary,
            startOpacity: 0.1,
            endOpacity: 0.1,
          },
        ]}
        showDataPointLabelOnFocus
        noOfSections={4}
        height={100}
        yAxisOffset={Math.max(Math.floor(minValue) - 10, 0)}
      />
    </View>
  );
}

function FocusedDatapointLabelComponent(props: { value: number; label: string }) {
  const { colors } = useAppTheme();
  return (
    <View
      style={{
        alignItems: 'center',
        paddingVertical: spacing[1],
        backgroundColor: colors.surface,
        borderRadius: 4,
        borderColor: colors.outline,
        borderStyle: 'solid',
        borderWidth: 1,
      }}
    >
      <Text>{props.label}</Text>
      <Text>{props.value.toFixed(0)} W</Text>
    </View>
  );
}
```

- [ ] **Step 3: Add the card to the expanded exercise page**

In `app/src/app/stats/expanded-weighted-exercise.tsx`:

(a) Add the import:

```ts
import { PowerLineChart } from '@/components/presentation/stats/power-line-chart';
```

(b) In `LoadedStatsFilled`, insert after the `stats.exercise.max_weight.title` card:

```tsx
      {stats.maxPowerPerSessionStatistics && (
        <StatCardWithTitle title={t('stats.exercise.max_power.title')}>
          <PowerLineChart statistics={stats.maxPowerPerSessionStatistics} />
        </StatCardWithTitle>
      )}
```

- [ ] **Step 4: Typecheck, lint, full tests**

Run:
```bash
npm run typecheck
npm run lint
npx vitest run
```
Expected: all green.

- [ ] **Step 5: Commit**

```bash
cd /home/jhoblitt/github/LiftLog
git add app/src/components/presentation/stats/power-line-chart.tsx app/src/app/stats/expanded-weighted-exercise.tsx app/src/i18n/en.json
git commit -m "feat: Add Max Power chart to exercise stats

Shown only for exercises with at least one recorded power value.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 13: Full verification sweep

**Files:**
- Possibly modified: formatting-only changes from `npm run format`; regenerated `docs/schemas/**` stragglers.

**Interfaces:**
- Consumes: everything above.
- Produces: a branch where format, lint, typecheck, the full test suite, and schema generation are all clean — the bar CONTRIBUTING.md sets for PRs.

- [ ] **Step 1: Format**

Run: `npm run format`
Then: `git -C /home/jhoblitt/github/LiftLog status --short`
Expected: ideally no changes. If oxfmt reformatted files THIS branch touched, stage and fold them into a `chore: Format` commit; if it reformatted files this branch never touched, leave those unstaged (pre-existing drift — do not commit).

- [ ] **Step 2: Lint, typecheck, tests, schemas**

Run:
```bash
npm run lint
npm run typecheck
npx vitest run
npm run json-schema
git -C /home/jhoblitt/github/LiftLog status --short
```
Expected: lint/typecheck/tests green; after `json-schema`, `git status` shows NO modifications under `docs/schemas` (already regenerated in Tasks 2 and 4). If anything appears, commit it as `chore: Regenerate JSON schemas`.

- [ ] **Step 3: Review the series**

Run: `git -C /home/jhoblitt/github/LiftLog log --oneline main..feature/power-tracking`
Expected: roughly 14 commits — 3 docs (spec, spec revision, discussion draft) + 1 per implementation task. Confirm no commit mixes docs-working-material with implementation.

- [ ] **Step 4: Push to the fork**

```bash
git -C /home/jhoblitt/github/LiftLog push -u jhoblitt feature/power-tracking
```
(Needs `dangerouslyDisableSandbox` in this harness — `push -u` writes `.git/config`.)

---

### Task 14: Android sideload build and manual validation

**Files:** none (build + on-device validation).

**Interfaces:**
- Consumes: the completed branch; a USB-connected Android device with developer mode + USB debugging enabled (user-provided).
- Produces: the validated go/no-go for opening the upstream PR.

- [ ] **Step 1: Build and install on the device**

The project is CNG (no committed `android/` dir); its own device script uses the `debugOptimized` variant, which is the right one for validation (release signing needs the user's keystore via `plugins/android-signing.js`):

```bash
cd /home/jhoblitt/github/LiftLog/app
npx expo prebuild --platform android
npx expo run:android --variant debugOptimized --device
```

Expected: gradle build succeeds (Kotlin codegen consumes the updated schemas via the resources symlink) and the app installs and launches on the device. If no device is attached, STOP and ask the user to connect one (or run these two commands themselves with `! <command>`).
Alternative manual install: the APK lands under `app/android/app/build/outputs/apk/debugOptimized/`; `adb install -r <apk>`.

- [ ] **Step 2: Manual validation checklist (user walks the device, per spec §9)**

- [ ] Edit a program exercise → "Track Power" switch appears for weighted exercises, default off; turn it on and save.
- [ ] Plan-diff propagation: during a workout, edit the exercise and flip Track Power; on finishing, the "update your plan?" flow lists a "Power" change and saving it persists the toggle to the program (next workout still prompts).
- [ ] Tap a set to complete it → Max Power dialog appears; entering `312` and Save shows `312 W` on the tile.
- [ ] Tap another set and Skip → set stays completed, tile shows `– W`.
- [ ] Tap a completed set again (rep decrement) → no dialog re-opens; watts unchanged.
- [ ] Long-press a set → Power field pre-filled; edit and Save updates the tile.
- [ ] Long-press an uncompleted set, type reps AND watts, Save → set completes with power in one step.
- [ ] Long-press quick rep buttons → complete with whatever watts are in the field.
- [ ] Clear (×) in the long-press dialog → set uncompleted, power gone.
- [ ] Non-power exercises: tiles and dialogs look exactly as before (no watts line, no power field, no popup).
- [ ] Previous-exercise viewer and history/session summaries show `312 W` chips for powered sets.
- [ ] Stats → exercise with power → "Max Power" chart present with correct values; exercise without power → no Max Power card.
- [ ] Total-weight/volume stats do NOT include watts anywhere (the original kludge distortion is gone).
- [ ] Restart the app mid-workout → recorded watts survive (persistence round-trip).

- [ ] **Step 3: Report results**

Summarize any failures for fixes; when the checklist is green, the branch is ready for PR prep (drop the three `docs/superpowers` working-material commits via rebase, then open the draft PR — outside this plan).

---

## Self-Review Notes

- Spec coverage: data model (§1→Tasks 2–4), popup (§2→6–7), long-press second path (§3→8), display (§4→7,10), editor (§5→9), plan-diff (§5a→5), stats (§6→11–12), i18n (§7→5,6,8,9,12), edge cases (§8→model tests in 2–3), testing+sideload (§9→13–14), sequencing/deliverables (§10→1,13,14; backfill explicitly out of scope).
- Cross-version caveat (§ data model) needs no code — it is documented in the spec and the discussion draft (Task 1).
- Type consistency: `withPower(setIndex, power)`, `maxPower`, `latestRecordedPower`, `trackPower`, `NumericStatisticOverTime`, `maxPowerPerSessionStatistics`, `onUpdateReps(reps, power)` are used with identical names/signatures across tasks.
