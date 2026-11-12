# Plan: Nov 12 to Nov 17

**Goal:** beat the published floor (`spellF1` 0.2433) by a wide margin, with a repository that
rebuilds the exact same submission file.

**Constraints:** no hand labels on this match, no tuning on the public 5 minutes, and no match data
shared with cloud LLMs.

---

## Architecture

A two-stage pipeline in our own public repo (copy in `games.py` and `evaluation/`, and keep `data/`
gitignored):

```
mtah/
  io.py          parse ATD once, cache to .npz: ball[n,2], players[n,46,2], speed[n,46], team[46]
  tagging.py     Smart Tagging → set-piece restart frames, team-phase intervals
  features.py    per-frame features (nearest-player gap, ball speed, team shape, direction of play)
  deadball.py    stage 1: alive/dead per frame
  possession.py  stage 2: H/A inside each live-ball stretch (Viterbi)
  pipeline.py    one command: data → submission CSV → validate
validation/
  metrica_sample.py   legal tuning set built from other games
```

**Contract between the two stages:** `deadball.py` outputs a list of live-ball stretches
`[(start, end), ...]`, and `possession.py` assigns H or A to every frame inside them. Two people can
then work on the stages in parallel.

**Don't write `X` on graded frames.** The scorer fills `X` with our previous label anyway, so
writing that label ourselves scores the same on `spellF1` and better on `macroF1`. Use `X` only on
ungraded frames.

---

## Versions

Commit and score every version. Upload v1 to the form as insurance, since the response can be
edited until the deadline.

| Version | Change | Done when |
|---|---|---|
| **v0** | Reproduce the baseline from the cache | Matches `baseline_smooth50.csv` exactly |
| **v1** | Missing ball no longer means dead (carry the last team forward); 2nd half 110,808–113,742 = `D`; `X` outside the halves | Validates; possession count drops a lot |
| **v2** | Dead-ball detector | `D` intervals look right when we watch sampled stoppages on video |
| **v3** | Viterbi possession model with switch penalty and "controlled possession" rule | Possession count and lengths look like football |
| **v4** | Smart Tagging evidence + team-shape features for frames without the ball | Improves on the validation set (Workstream D) |

---

## Workstreams

The plan assumes 4 people. With 3, fold D into A and C.

### A: Data and pipeline (owns v0, v1, the final run)

- `io.py`: stream the raw ATD once (`starter/baseline.py` shows the format) and cache the arrays.
  Convert positions to metres with `GAMES[game]["pitch"]` (105.3 × 68.0 m).
- `pipeline.py`: a single deterministic command that ends by running
  `evaluation/validate_submission.py`.
- Pin dependencies, no randomness (or fixed seeds).
- Test from a clean clone on Nov 16 and again on Nov 17.

### B: Dead-ball detector (v2)

- Score how likely each frame is to be dead using:
  - the ball outside the pitch, or last seen near a line just before it disappears
  - mean player speed dropping sharply over a ~2 s window
  - the ball sitting still at a restart spot
- Use Smart Tagging `SET PIECES` start times as the *end* of a stoppage, then search backwards for
  where it began. Mark the whole stretch from a `GOALS` tag to the following kick-off as dead.
- Smooth with hysteresis and a minimum stoppage length (~2 s) so `D` doesn't flicker.
- Use the public 5 minutes for one thing only: checking what the ground truth means by "dead" (how
  early `D` starts before the ball crosses the line, how late it ends at a restart). Don't fit
  thresholds to it.

### C: Possession model (v3, v4)

- Work out direction of play for each half from the team centroids.
- Per-frame evidence for H vs A:
  - gap between the nearest home and nearest away player, using hysteresis
  - passes: ball speed jumps from one player and arrives at a teammate
  - team shape (spread, forward movement of the centroid, pressing) for frames without the ball
  - Smart Tagging phases with a team tag as soft evidence (`BUILD UP`, `PROGRESSION`,
    `CREATING CHANCES`, `ATTACKING TRANSITION` support that team; `DEFENDING IN ...` supports the
    other)
- Run a two-state Viterbi in each live-ball stretch. Its switch penalty sets the possession count.
- Control rule: a change counts only if the new team keeps the ball for about 1 s or makes 2
  passes. Loose touches and aerial duels shouldn't flip it.
- Start each stretch with the team taking the restart (from the tag's `Team`; for throw-ins tagged
  `N/A`, the team nearest the ball).

### D: Validation and technical document

- Download Metrica's open sample games (github.com/metrica-sports/sample-data: tracking + events).
- Turn their events into H/A/D labels per frame (`BALL LOST`, `RECOVERY`, `BALL OUT`, `SET PIECE`).
- Degrade the tracking to look like ours: drop the ball on ~40% of frames, in bursts; add jitter;
  split tracks.
- Score with `evaluation/possession_metrics.evaluate()` directly, which works on code arrays.
- This is where the switch penalty and thresholds get tuned. The rules allow it as long as the
  document describes it.
- From Nov 15, write the technical document alongside. It needs: approach, assets used and dropped,
  training data, possession rules, limitations, reproduction.

---

## Schedule

| Date | A | B | C | D |
|---|---|---|---|---|
| **Thu 12** | Get the data, build the cache, v0 | Explore stoppages in the video + Smart Tagging | Direction of play, nearest-player features | Download sample data, label conversion |
| **Fri 13** | v1, **upload v1** | Dead-ball v2 draft | Viterbi v3 draft | Degradation + scoring harness |
| **Sat 14** | Integrate v2 + v3 | Refine `D` boundaries | Control rule, restart initialisation | Tune switch penalty and `D` thresholds |
| **Sun 15** | Integrate v4, upload | Goals, kick-offs, edge cases | Team-shape + tagging evidence (v4) | Ablations with `--bootstrap`, start the doc |
| **Mon 16** | **Freeze code by evening**; test from a clean clone | Bug fixes only | Bug fixes only | Finish the doc |
| **Tue 17** | Re-run, validate the **exact** file, submit `<team>.csv` + `<team>.pdf` **by midday** | | | Final read of the doc |

---

## Rules for deciding

- Keep a change only if it helps on the validation set from Workstream D *or* has a clear football
  reason. Never because it gained on the public 5 minutes. A 6-point move there is one possession.
- Use the public window only to catch breakage: misaligned frames, swapped H and A, or a score
  collapse.
- Before trusting a gain, run `run_evaluation.py --bootstrap 1000` and check whether the intervals
  overlap.
- Track the possession count and median length in every run. If either drifts a lot from one
  version to the next, look into it before moving on.

---

## Risks

| Risk | Mitigation |
|---|---|
| Possession labels built from events in the sample games don't match the tracking-based definition here | Use the sample games for relative comparisons only; keep parameters close to football logic |
| The validation set isn't ready by Saturday | Fall back to fixed values: 1 s control, 2 s minimum stoppage |
| Integration breaks late | v1 is already uploaded; code freeze Monday evening |
| The submission file doesn't reproduce exactly | Clean-clone test on Monday and Tuesday; no randomness in the pipeline |
