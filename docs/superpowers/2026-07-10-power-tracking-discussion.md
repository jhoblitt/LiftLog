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
