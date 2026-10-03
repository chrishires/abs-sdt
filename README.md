# Perception vs. Decision in MLB's ABS Challenge System

A Bayesian signal detection analysis of every pitch in the 2026 MLB regular season.

In 2026, MLB introduced the Automated Ball-Strike (ABS) challenge system: a batter, catcher, or pitcher can appeal an umpire's ball/strike call to a pitch-tracking system. Each team gets two challenges per game and keeps a challenge when it succeeds.

Public leaderboards track who *wins* challenges. This project asks a different question. For each player:

1. **Perception:** how accurately can they tell whether a call was wrong?
2. **Decision:** how much evidence do they need before they spend a challenge?

Separating the two requires accounting for the missed calls players *didn't* challenge, which outcome-based stats can't see.

## Key finding

**Players perceive the strike zone about equally well, and differ mainly in how much evidence they demand before challenging.**

Both quantities are estimated in inches, so their spread across players can be compared directly:

| Group | Typical criterion | Player-to-player SD of criterion | Typical perceptual noise (σ) | Player-to-player SD of σ |
|---|---|---|---|---|
| Batters (656) | 4.7 in | 1.0 in | 3.5 in | 0.3 in |
| Catchers (111) | 2.2 in | 0.3 in | 2.0 in | 0.2 in |

- Among batters, decision criteria vary about **3 times** as much as perception. Among catchers, about **twice** as much.
- Batters are very conservative. The typical batter doesn't reach a 50% chance of challenging until a pitch is 4.7 in outside the zone, but the median missed call is only 0.8 in outside. **82% of missed strike calls go unchallenged.**
- Catchers are both more aggressive and more perceptive than batters, which explains their higher success rate (59% vs. 49%). They still leave 71% of missed ball calls unchallenged.
- The model's criterion ranking recovers most of the outcome-based "recall" leaders without ever seeing challenge outcomes.
- Pitcher challenges are too rare to model: 179 on 256,734 called balls.

## Approach

**Signal detection theory.** Each player perceives a pitch's true distance from the zone edge with Gaussian noise and challenges when their percept exceeds a personal threshold. Integrating out the noise gives a probit:

$$P(\text{challenge} \mid d) = \Phi\left(\frac{d - c}{\sigma}\right)$$

Here *d* is the pitch's signed distance from the zone edge, *c* is the player's criterion (how much evidence they demand), and *σ* is their perceptual noise. Unlike the classic SDT convention of fixing σ = 1, both parameters are estimated in physical units (feet), which is what makes the cross-player comparison above possible.

**Hierarchical Bayesian model (PyMC).** Each player's *c* and log *σ* are drawn from league-wide distributions, with a non-centered parameterization. Partial pooling keeps estimates stable for players with few pitches. The evidence variable stays continuous: the steepness of each player's challenge curve is what identifies σ.

**Beyond precision.** Public analysis of ABS challenges measures success by precision (the share of challenges overturned) or by overturn counts. This project also measures **recall**, the share of missed calls a player challenged, which requires classifying every called pitch, not just the challenged ones.

## Data pipeline

`ETL.ipynb` builds a one-row-per-pitch table (2,429 games, 714,020 pitches) from the MLB Stats API's event-level game feed. The feed records what *happened*. The analysis needs the state of the game *before* each pitch, so the pipeline replays every game event by event and reconstructs:

- **Count:** rebuilt forward from 0-0, including 2026 pitch-clock violations, which add automatic balls and strikes that aren't pitches.
- **Runners, outs and score:** tracked by player ID across each half-inning, handling pinch runners and the extra-innings automatic runner.
- **Catcher:** the player actually behind the plate, including mid-game and mid-at-bat substitutions.
- **Challenges remaining:** two per team, kept on success, plus the extra-innings bonus challenge, which MLB's own summary field doesn't reflect.
- **Original call:** the umpire's call before any overturn, since that's what the player was reacting to.

**Every reconstruction is validated for every game** against an independent total MLB reports: the recorded count on each pitch, the next play's outs and runners, the final linescore, the challenge ledger, and the identity of each challenger.

**Pitch location.** Statcast reports where a pitch crosses the *front* of home plate, but ABS rules on where it crosses the *midpoint*, 8.5 in deeper. The pipeline re-projects each pitch to the midpoint using Statcast's constant-acceleration trajectory fit, and pads the zone by the ball's radius, because ABS calls a strike if any part of the ball touches the zone. This classification matches ABS's actual ruling on about 99% of challenged pitches, compared with about 85% using the front-of-plate coordinates.

## Repository contents

| File | Description |
|---|---|
| `ETL.ipynb` | Data pipeline: schedule, API extraction, game-state reconstruction and validation, geometry, outcome labels. Writes `pitch_data.parquet`. |
| `analysis.ipynb` | Baseline outcome metrics, batter and catcher SDT models, diagnostics, results and limitations. |
| `pixi.toml`, `pixi.lock` | Reproducible environment ([pixi](https://pixi.sh)). |

## Running it

1. **Install the environment** with [pixi](https://pixi.sh):
   ```bash
   pixi install
   ```
   The lockfile targets macOS on Apple Silicon. For another platform, run `pixi workspace platform add <platform>` first (for example `linux-64`).

2. **Get the data.** Either:
   - download `pitch_data.parquet` from the [latest release](../../releases/latest) into the repository root, or
   - rebuild it by running `ETL.ipynb`. This makes one API call per game, about 2,400 calls, and takes roughly an hour.

3. **Run `analysis.ipynb`** using the pixi environment's kernel. Sampling takes about 8 minutes for the batter model and 16 minutes for the catcher model on an M-series Mac.

## Limitations and next steps

- About 7% of the modeled pitches came when the player's team had no challenges left. They're currently counted as declined challenges, which biases criteria slightly upward. The fix is a filter and a refit.
- The criterion is fixed per player, not a function of game situation.
- **Next:** compute the run-expectancy value of a challenge and the option value of holding one, then compare each player's observed criterion with the economically optimal one.

## Author

Chris Hires
