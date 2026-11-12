# Plan: Nov 13 to Nov 17

**Goal:** beat the published floor (`spellF1` 0.2433) by a wide margin, with a repository that
rebuilds the exact same submission file.

**Constraints:**

- No hand labels on this match, and no tuning on the public 5 minutes.
- No constants tied to specific frames of this match (RULES.md). Anything frame-specific comes from
  `games.py` or the Smart Tagging, never from a literal in our code.
- No match data shared with cloud LLMs. That includes coding assistants: nothing from `data/`, and
  no per-frame output derived from it, gets printed into a chat.

---

## What the plan is built on

Measured on public files only (the 5-minute ground truth and the committed baseline CSVs).
`baseline_raw.csv` is `D` exactly where the ball is missing, so it is a public map of ball
visibility.

**The ball goes missing in long blocks, and the public window hides it.**

- Missing on 12% of the public window, against 41% of each half.
- The share climbs through both halves and reaches 60–88% in the later 5-minute blocks.
- About 70 missing runs per half, none shorter than 3 s; 81–89% of missing frames are in runs
  longer than 10 s. A mean possession is about 14 s, so one blackout usually spans a turnover.

**Team changes carry the score; stoppage boundaries barely do.** The public ground truth,
perturbed and rescored:

| Variant | spellF1 |
|---|---|
| Perfect teams, no `D` at all | 0.81–0.85 |
| Every `D` boundary 1 s off | 1.00 |
| Every team switch 1 s late | 0.82 |
| Every team switch 2 s late | 0.68 |
| Perfect `D`, baseline's teams | 0.40–0.52 |
| Possessions under 1.5 s absorbed | 0.83 |
| One false 1 s flip in each of the 4 long possessions | 0.70 |

This is 17 possessions and 4 stoppages. It shows how the metric behaves; it is not an estimate for
the full match.

**The ground truth counts very short possessions.** The public window has possessions of 0.8 s and
1.3 s, and 6 of 17 are under 4 s. Missing one and inventing one cost about the same.

What follows:

1. Possession with no ball in sight is the core of the problem, not a late add-on.
2. Most of the effort goes on team calls and their timing. The first dead-ball detector is cheap.
3. The minimum length of a possession is tuned on other matches, not fixed from football intuition.
4. The validation set needs real per-frame labels, with the ball removed in long blocks.

---

## Architecture

A two-stage pipeline in our own public repo (copy in `games.py` and `evaluation/`; keep `data/`,
the `.npz` cache and every submission CSV gitignored):

```
mtah/
  io.py            parse ATD once, cache to .npz: ball[n,2], players[n,46,2], speed[n,46], team[46]
  tagging.py       Smart Tagging → restart frames, kick-offs, team-phase intervals
  features.py      per-frame features, split into ball-based and player-only
  deadball.py      stage 1: alive/dead per frame
  evidence_ball.py   H-vs-A evidence when the ball is visible
  evidence_shape.py  H-vs-A evidence from the players alone
  possession.py    stage 2: combine evidence, decode H/A inside each live stretch (Viterbi)
  pipeline.py      one command: data → submission CSV → validate
validation/
  build_labels.py  per-frame H/A/D for other matches
  degrade.py       make clean tracking look like ours
  score.py         run the pipeline on validation matches and report
```

**Contracts, so people can work in parallel:**

- `deadball.py` outputs live-ball stretches `[(start, end), ...]`.
- Each evidence module outputs one number per frame: the log-odds of H against A, and 0 where it
  has no opinion. `possession.py` adds them and decodes.
- Every module takes arrays, not file paths, so the same code runs on our match and on validation
  matches.

**Don't write `X` on graded frames.** The scorer fills `X` with our previous label, so writing that
label ourselves scores the same on `spellF1` and no worse on `macroF1`. Use `X` only on ungraded
frames.

---

## Versions

Commit and score every version. Upload v1 to the form as insurance, since the response can be
edited until the deadline.

| Version | Change | Done when |
|---|---|---|
| **v0** | Reproduce the baseline from the cache | Matches `baseline_smooth50.csv` exactly |
| **v1** | Missing ball no longer means dead (carry the last team forward); `D` from the start of each half to its `Kick Off (Start)` tag, cross-checked against the `games.py` anchors; `X` outside the halves | Validates; possession count drops a lot |
| **v2** | Viterbi possession model on ball-based evidence, plus the cheap dead-ball detector | Beats v1 on the validation set where the ball is visible |
| **v3** | Player-only evidence for the stretches without the ball, plus Smart Tagging team phases | Beats v2 on the validation set with the ball removed in long blocks |
| **v4** | Whatever the validation set says is the largest remaining error: stoppage recall, switch timing, or learned evidence (see optional track) | Gain is outside the bootstrap interval |

---

## Workstreams

The plan assumes 4 people, with two of them on possession. With 3, fold A into D's harness work and
keep B and C intact.

### A: Pipeline and dead ball (owns v0, v1, the final run)

- `io.py`: stream the raw ATD once (`starter/baseline.py` shows the format) and cache the arrays.
  Convert positions to metres with `GAMES[game]["pitch"]`.
- `pipeline.py`: a single deterministic command that ends by running
  `evaluation/validate_submission.py`. Pin dependencies, no randomness.
- `deadball.py`, kept simple on purpose because boundary errors of a second cost almost nothing:
  - start of each half until the kick-off tag
  - a `GOALS` tag until the following kick-off
  - the ball outside the pitch, or last seen near a line before it disappears
  - mean player speed dropping sharply over ~2 s
  - Smart Tagging `SET PIECES` starts as the *end* of a stoppage, searching backwards for its start
  - hysteresis and a minimum stoppage length so `D` doesn't flicker
- Smart Tagging has only 59 set pieces (5 free kicks), so it finds restarts with good precision and
  poor recall. The other signals have to carry stoppages it misses.
- What matters most here is not missing a stoppage that sits between two possessions of the same
  team, since that merges two real possessions into one.
- Test from a clean clone on Nov 16 and again on Nov 17.

### B: Possession without the ball (v3)

The largest single piece of work, covering roughly 40% of the match.

- Player-only features, all built per team because tracks have no identities and fragment:
  - team spread and compactness (the team with the ball spreads, the other squeezes)
  - centroid position and velocity along the direction of play
  - depth and movement of each team's back line
  - how many players of each team are moving towards the same spot (pressing)
  - where the players' movement converges, as a rough guess at the ball's location
- Smart Tagging phases with a team tag as soft evidence (`BUILD UP`, `PROGRESSION`,
  `CREATING CHANCES`, `ATTACKING TRANSITION` support that team; `DEFENDING IN ...` supports the
  other). Evidence, not an answer: a phase can cover two possessions.
- Calibrate each feature into log-odds on the validation matches, with the ball hidden.
- Check on the validation matches how well these features find a turnover *inside* a blackout. That
  number, more than any other, tells us how high the score can go.

### C: Possession with the ball, and the decoder (v2)

- Direction of play per half, from `attack_direction` in the ATD metadata, checked against team
  centroids.
- Ball-based evidence:
  - gap between the nearest home and nearest away player
  - passes: the ball leaves one player fast and arrives at a teammate
  - ball speed and height proxies, to discount aerial duels and deflections
- `possession.py`: a two-state Viterbi in each live stretch, over the summed evidence from B and C.
- Start each stretch with the team taking the restart (from the tag's `Team`; for throw-ins tagged
  `N/A`, the team nearest the ball).
- No fixed "1 s or 2 passes" rule. The switch penalty and any minimum possession length are tuned
  on the validation set, because the ground truth keeps possessions under a second.
- Viterbi sees the future, so a switch should land on the frame where control changed, not where
  the evidence became convincing. Verify there is no lag (see "Rules for deciding").

### D: Validation and technical document

- **Friday morning, 30 minutes:** confirm that the DFL open tracking dataset (Bassek et al., 7
  matches, kloppy's sportec loader) carries per-frame ball possession and ball status. If it does,
  that is H/A/D directly.
- **Fallback:** Metrica's open sample games, turning events into per-frame labels (`BALL LOST`,
  `RECOVERY`, `BALL OUT`, `SET PIECE`). Usable, but the conversion adds its own errors.
- `degrade.py`, matched to what our tracking looks like:
  - remove the ball in runs of 10 s and longer, covering about 40% of frames, more often late in
    each half
  - drop player identities and split tracks into fragments across ~46 slots
  - add position jitter and occasional missing or duplicated players
- Compare the validation labels' possession lengths with the public window's. If the validation
  labels have no possessions under 2 s, their notion of control is stricter than ours and the
  switch penalty tuned on them will be too high.
- `score.py` reports, for every run: `spellF1`, `macroF1`, possession count and median length,
  transition scores at 12 and 25 frames, and all of these separately for ball-visible and
  ball-hidden stretches.
- This is where the switch penalty and thresholds get tuned. The rules allow it as long as the
  document describes it.
- From Nov 15, write the technical document alongside: approach, assets used and dropped, training
  and validation data, possession rules, limitations, reproduction.

### Optional track: learned evidence

Only if someone has trained sequence models before, and only beside the rule-based version.

- Train a small model on the degraded validation matches to predict H/A/D from player positions
  alone, and feed its output into the same Viterbi as one more evidence term.
- Hold out whole matches for testing, never frames.
- Commit the trained weights and pin versions, or the submission will not reproduce.
- **Decision on Saturday evening:** it goes into the pipeline only if it beats B's hand-built
  features on a held-out match by more than the bootstrap interval. Otherwise it is dropped.

---

## Schedule

Thursday is gone, so v0 and v1 both land on Friday.

| Date | A | B | C | D |
|---|---|---|---|---|
| **Fri 13** | Cache, v0, v1, **upload v1** | Player-only features on the cache | Direction of play, ball-based evidence | Confirm dataset, labels, scoring harness |
| **Sat 14** | Cheap dead-ball detector | Calibrate shape evidence with the ball hidden | Viterbi, restart initialisation, **v2** | Degradation; first tuning of switch penalty |
| **Sun 15** | Integrate **v3**, upload | Tagging evidence, turnovers inside blackouts | Lag check, short-possession trade-off | Ablations with `--bootstrap`, start the doc |
| **Mon 16** | v4 if it earns its place; **freeze code by evening**; clean-clone test | Bug fixes only | Bug fixes only | Finish the doc |
| **Tue 17** | Re-run, validate the **exact** file, submit `<team>.csv` + `<team>.pdf` **by midday** | | | Final read of the doc |

---

## Rules for deciding

- Keep a change only if it helps on the validation set *or* has a clear football reason. Never
  because it gained on the public 5 minutes. A 6-point move there is one possession.
- Use the public window only to catch breakage (misaligned frames, swapped H and A, a score
  collapse) and to understand what the labels mean. It has the ball visible 88% of the time, so it
  says nothing about the half of the problem that matters most.
- Judge every change on the ball-hidden score as well as the overall one.
- Before trusting a gain, run `--bootstrap 1000` and check whether the intervals overlap.
- Watch for lag: if the transition score at 25 frames is much better than at 12, switches are
  landing late, and a 1 s lag costs about a fifth of the score.
- Track the possession count and median length in every run. If either drifts a lot from one
  version to the next, look into it before moving on.

---

## Risks

| Risk | Mitigation |
|---|---|
| Player-only evidence turns out too weak to find turnovers inside blackouts | Know by Saturday evening from the ball-hidden validation score; fall back to tagging phases plus carrying the last team, and spend the time on switch timing where the ball is visible |
| Validation labels define possession differently from this match | Compare possession-length distributions; use the validation set for relative comparisons; keep parameters defensible in football terms |
| The DFL dataset doesn't have per-frame labels, or the validation set isn't ready by Saturday | Fall back to Metrica sample events; failing that, fixed values (switch penalty equivalent to ~1 s, 2 s minimum stoppage) |
| Our degradation is easier than the real tracking | Match the missing-ball run lengths and the rise through each half; check track counts and fragment lengths against our cache |
| Integration breaks late | v1 is already uploaded; evidence modules share one contract; code freeze Monday evening |
| The submission file doesn't reproduce exactly | Clean-clone test on Monday and Tuesday; no randomness; trained weights committed if used |
| A frame-specific constant slips into the code | Review for literals before the freeze; everything frame-specific comes from `games.py` or the tagging |
