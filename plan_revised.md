# Tasks

Design: one joint 3-state HMM (`H`, `A`, `D`) decoded with Viterbi over hand-built features,
using tactical tracking and Smart Tagging, tuned on Metrica's open sample games.

## Pipeline

- [ ] Download the data bundle into `data/` and check it loads
- [ ] `mtah/io.py`: parse the raw ATD once and cache `ball[n,2]`, `players[n,46,2]`, `speed[n,46]`,
      `team[46]` to `.npz`, positions in metres
- [ ] Reproduce `baseline_smooth50.csv` exactly from the cache (v0)
- [ ] `mtah/pipeline.py`: one deterministic command from data to submission CSV, ending with
      `evaluation/validate_submission.py`
- [ ] Write `X` on ungraded frames only; never on graded frames
- [ ] v1: missing ball carries the last team forward instead of `D`
- [ ] v1: label 2nd-half frames 110,808–113,742 as `D`
- [ ] Upload v1 to the form as a fallback submission
- [ ] Pin dependencies and remove any randomness
- [ ] Reproduce the submission file from a clean clone

## Smart Tagging

- [ ] `mtah/tagging.py`: load set pieces with restart frame, type and team
- [ ] Load team-phase intervals (`BUILD UP`, `PROGRESSION`, `CREATING CHANCES`, `ATTACKING TRANSITION`,
      `DEFENDING IN ...`) with their team
- [ ] Handle `N/A` team tags (all throw-ins)

## Features: dead-ball evidence

- [ ] Ball outside the pitch, or last seen near a line before disappearing
- [ ] Sharp drop in mean player speed over a ~2 s window
- [ ] Ball stationary at a restart spot (corner, 6-yard box, centre spot, touchline)
- [ ] Set-piece tags: frames before a restart are likely dead, the restart frame is likely alive
- [ ] `GOALS` tag: dead until the following kick-off
- [ ] Check on the public 5 minutes how early `D` starts and how late it ends (understanding only,
      no fitting)

## Features: team evidence

- [ ] Infer direction of play per half from team centroids
- [ ] Gap between nearest home and nearest away player
- [ ] Pass detection (ball speed jumps from a player and arrives at a teammate)
- [ ] Team shape for frames without the ball (spread, centroid moving forward, pressing)
- [ ] Smart Tagging team phases as soft evidence

## Joint model

- [ ] `mtah/model.py`: per-frame emission scores for `H`, `A`, `D` built from both feature groups
- [ ] Transition costs: `H`↔`A` switch penalty, live→`D` and `D`→live costs
- [ ] `D`→`H`/`A` biased toward the restarting team (tag team, or nearest team for `N/A`)
- [ ] Minimum durations: stoppages (~2 s) and possession changes (~1 s on the ball or 2 passes),
      via expanded states or duration penalties
- [ ] Viterbi decode per half
- [ ] Handle frames with no ball: dead-ball and team evidence fall back to player-only features

## Validation

- [ ] Download Metrica's open sample games (tracking + events)
- [ ] Convert their events into H/A/D labels per frame
- [ ] Degrade their tracking: drop the ball on ~40% of frames in bursts, add jitter, split tracks
- [ ] Score with `evaluation/possession_metrics.evaluate()`
- [ ] Tune transition costs, minimum durations and emission weights on the sample games
- [ ] Ablations: measure what each feature adds, with `--bootstrap 1000`
- [ ] Log possession count and median length for every run

## Submission

- [ ] Technical document: approach, assets used and dropped, training data, possession rules,
      limitations, reproduction
- [ ] Make the repository public, with no match data in it
- [ ] Validate the exact final file
- [ ] Submit `<team>.csv` and `<team>.pdf` before Nov 17
