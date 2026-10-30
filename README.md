# Perception vs. Decision in MLB's ABS Challenge System

A Bayesian signal detection analysis of every pitch in the 2026 MLB regular season.

In 2026, MLB introduced the Automated Ball-Strike (ABS) challenge system: a batter, catcher, or pitcher can appeal an umpire's ball/strike call to a pitch-tracking system. Each team gets two challenges per game and keeps a challenge when it succeeds.

Public leaderboards track who *wins* challenges. This project asks a different question. For each player:

1. **Sensitivity:** how closely do their challenge decisions track where the pitch actually was?
2. **Decision criterion:** how much evidence do they need before they spend a challenge?

Separating the two requires accounting for the missed calls players *didn't* challenge, which outcome-based stats can't see.

## Key finding

**Players' challenge decisions track pitch location with roughly the same precision. What separates them is how much evidence they demand before challenging.**

| Group | Typical criterion *c* | Player-to-player SD of *c* | Typical sensitivity *d′* | Player-to-player SD of *d′* |
|---|---|---|---|---|
| Hitters (653) | 4.29 in | 1.03 in | 1.46 | 0.16 (≈11%) |
| Catchers (111) | 2.02 in | 0.39 in | 4.59 | 0.44 (≈10%) |

- **Hitters are conservative, and their thresholds vary widely.** The typical hitter doesn't reach a 50% chance of challenging until a pitch looks 4.3 in off the plate, while the average missed call is about 1 in off. Roughly a third of hitters sit more than an inch above or below that average.
- **Sensitivity barely varies.** Within each group, *d′* clusters tightly around its mean, so differences in challenge behavior come mostly from where each player sets their criterion.
- **Catchers are more aggressive, and more alike.** The typical catcher challenges once a ball looks about 2 in inside the zone.
- **Pitcher challenges are too rare to model:** 179 on 256,738 called balls.

*d′* is compared within each group only. Hitters' and catchers' signal and noise distributions differ for geometric reasons (a called ball can only be so far inside the zone), so their *d′* values aren't on the same scale.

## Approach

**Signal detection theory.** Each player acts on a noisy version of a pitch's true distance from the zone edge and challenges when it exceeds a personal threshold. Integrating out the noise gives a probit:

$$P(\text{challenge} \mid d) = \Phi\left(\frac{d - c}{\sigma}\right)$$

Here *d* is the pitch's signed distance from the zone edge, *c* is the player's criterion, and *σ* sets how sharply their challenge probability rises around it. Because challenges are optional, *σ* captures judgment as well as perception, so *d′* measures how closely challenge decisions track pitch location rather than eyesight alone. Unlike the classic SDT convention of fixing σ = 1, both parameters are estimated in physical units.

**Hierarchical Bayesian model (PyMC).** Each player's *c* and log *σ* are drawn from league-wide distributions, with a non-centered parameterization. Partial pooling keeps estimates stable for players with few pitches. The evidence variable stays continuous: the steepness of each player's challenge curve is what identifies σ.

**Opportunities.** Pitches thrown after a team has used up its challenges are excluded, since not challenging isn't a decision there. This matches how Baseball Savant counts challenge opportunities.

## Data pipeline

`ETL.ipynb` builds a one-row-per-pitch table (2,429 games, 714,033 pitches) from the MLB Stats API's event-level game feed. The feed records what *happened*. The analysis needs the state of the game *before* each pitch, so the pipeline replays every game event by event and reconstructs:

- **Count:** rebuilt forward from 0-0, including 2026 pitch-clock violations, which add automatic balls and strikes that aren't pitches.
- **Runners, outs and score:** tracked by player ID across each half-inning, handling pinch runners and the extra-innings automatic runner.
- **Catcher:** the player actually behind the plate, including mid-game and mid-at-bat substitutions.
- **Challenges remaining:** two per team, kept on success, with the extra-innings rule that no team enters an extra inning with zero challenges. MLB's own summary field doesn't reflect that rule.
- **Original call:** the umpire's call before any overturn, since that's what the player was reacting to.

**Every reconstruction is validated for every game** against an independent total MLB reports: the recorded count on each pitch, the next play's outs and runners, the final linescore, the challenge ledger, and the identity of each challenger.

**Pitch location.** Statcast reports where a pitch crosses the *front* of home plate, but ABS rules on where it crosses the *midpoint*, 8.5 in deeper. The pipeline re-projects each pitch to the midpoint using Statcast's constant-acceleration trajectory fit, and pads the zone by the ball's radius, because ABS calls a strike if any part of the ball touches the zone. This classification matches ABS's actual ruling on 99.7% of the season's 10,556 challenged pitches.

## Repository contents

| File | Description |
|---|---|
| `ETL.ipynb` | Data pipeline: schedule, API extraction, game-state reconstruction and validation, geometry, outcome labels. Writes `pitch_data.parquet`. |
| `part1.ipynb` | Part 1 of the blog post, as a Quarto notebook: the SDT explainer, both models, results and player comparisons. |
| `images/` | Images used in the post. |
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

3. **Run `part1.ipynb`** using the pixi environment's kernel. Sampling takes about 8 minutes for the hitter model and 16 minutes for the catcher model on an M-series Mac. To render the post, use [Quarto](https://quarto.org).

## Limitations and next steps

- The criterion is fixed per player, not a function of game situation such as count, score or inning.
- Because challenges are optional, *σ* and *d′* mix perception with judgment; the model can't separate the two.
- Pitchers aren't modeled. A competing-risks model, with pitcher and catcher as two possible challengers on the same pitch, would be the principled way to include them.
- **Next (Part 2):** compute the run-expectancy value of a challenge and the option value of holding one, then compare each player's decisions with the economically optimal ones.

## Author

Chris Hires
