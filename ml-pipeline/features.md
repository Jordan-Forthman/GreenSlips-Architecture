# Predictive Engine: Feature Dictionary

This document outlines the high-level feature engineering strategies used by the GreenSlips MLB predictive models. *Proprietary weighting, tuning constants, fitted coefficients, and exact mathematical transformations have been redacted.*

The engine is implemented in-process in C# (see [ADR-0006 summary in the main README](../README.md#-ai--machine-learning-pipeline)). Features are computed **as-of a date** throughout — every rolling aggregate is built from data that existed before the game it describes, so a backtest cannot see forward and a live projection and a historical one are produced by the same code path.

## 1. Identity & Provenance

* **Vendor ID Crosswalk:** Player identity is reconciled across the statistics vendor, the open Chadwick register, and the market vendor by deterministic normalized-name matching with a date-of-birth tiebreak. Each resolution records its method, confidence, and source; ambiguous identities go to a review queue rather than being silently guessed. Every downstream feature keys off the resolved identity, so a mis-join can never quietly contaminate a matchup.
* **Single-Source Provenance:** Each data domain has exactly one authoritative vendor, enforced as an architectural boundary. No feature blends two vendors' versions of the same quantity.

## 2. Event-Level Foundation

* **Plate-Appearance Base-Out State:** The event store records every plate appearance with its inning, half, base occupancy, and out count. This is the substrate the run-scoring simulation transitions over, rather than a season-average approximation.
* **Pitch Kinematics:** Per-pitch release velocity, spin, induced and horizontal break, extension, and release position, aggregated into per-pitcher arsenal profiles.
* **Plate-Discipline Events:** In-zone, swing, whiff, contact, and chase flags derived per pitch, aggregated into batter and lineup-slot chase profiles.
* **Contact Quality:** Exit velocity, launch angle, and expected-wOBA measures on balls in play, used to separate a hitter's process from small-sample outcome luck.

## 3. Form & Temporal Features

* **Rolling As-Of Windows:** Batter and pitcher rate vectors — strikeout, walk, hit-by-pitch, home-run, extra-base-hit, whiff, chase, and contact-quality rates — maintained over short, medium, and long lookback horizons (30 / 90 / 365 days) so recency and stability can be traded off per market.
* **Bayesian Shrinkage:** Player rates are shrunk toward league baselines as a function of sample size, preventing early-season and low-plate-appearance players from producing extreme projections. League baselines are themselves rebuilt nightly from the event store.
* **Pitcher Velocity Decay:** Within-start velocity curves fitted per pitcher, capturing how a given arm's stuff degrades with pitch count rather than assuming a league-average decline.
* **Times-Through-the-Order:** Starter-facing matchups are conditioned on how many times the batter has already seen the pitcher in that game.

## 4. Matchup & Platoon Features

* **Handedness Splits:** Batter-vs-hand and pitcher-vs-hand split rates maintained separately, oriented at projection time by the opposing starter's throwing hand and the batter's side.
* **Batter-vs-Pitcher Projection:** Matchup cells are projected for the upcoming slate by crossing each probable starter against the opposing expected lineup, refreshed through the day as lineups firm up. Direct historical head-to-head samples are far too small to use raw; they inform rather than determine the estimate.
* **Pitcher Archetypes:** Pitchers are clustered on arsenal kinematics into unsupervised archetypes, so a batter's performance can be generalised against *pitcher types* he has faced rather than only the specific arms he has seen.
* **Lineup Vulnerability:** Opposing batting orders are scored as a whole — aggregated chase susceptibility and platoon exposure by lineup slot — rather than treating each batter independently.
* **Expected Lineups:** Before official lineups are published, a frequency model projects the batting order from recent usage and carries an explicit conviction score. When the official lineup posts, the row is promoted from Expected to Confirmed and the conviction is resolved; downstream consumers can gate on lineup state.

## 5. Workload & Bullpen Features

* **Starter Hook Hazard:** A survival model estimates the probability a starter is removed at each point in a start, conditioned on game state and accumulated workload. This drives how many plate appearances the simulation assigns to the starter versus the bullpen.
* **Bullpen Availability & Fatigue:** Reliever appearances are logged and turned into rolling fatigue and availability status, so recent heavy usage propagates into the next day's projected bullpen quality instead of being ignored.
* **Leverage-Aware Reliever Selection:** Post-hook plate appearances are assigned to specific, availability-gated relievers by leverage rather than to a static team-average bullpen composite.

## 6. Environment & Park Features

* **Park Factors:** Per-venue run and event scaling from vendor sabermetric feeds, plus stored stadium geometry.
* **Air Density Carry:** Temperature, pressure, humidity, and venue elevation are combined into an atmospheric carry factor that shifts the batted-ball outcome distribution. Weather is resolved from a static stadium-coordinate table — never geocoded at runtime — and missing weather degrades to a neutral factor rather than failing.
* **Immutable Forecast Capture:** The pre-game *forecast* is frozen shortly before first pitch and stored separately from the post-game actual, so a model can later be evaluated against the information it actually had at inference time.
* **Run Environment Classification:** Games are classified into run-environment regimes from the combined park, weather, and matchup context.

## 7. Running-Game Features

* **Catcher Control:** Catcher pop time paired against baserunner sprint speed to estimate stolen-base success, with a neutral-degraded fallback when either measurement is absent.
* **Attempt Propensity:** Steal-attempt likelihood is driven by each runner's own empirical, league-shrunk attempt rate given opportunities, rather than inferred from raw speed — fast players who do not run are a known failure mode of speed-only models.

## 8. Market & Edge Features

* **Vig Removal:** Book prices are converted to implied probabilities and de-vigged through a pluggable strategy (proportional, power, and Shin closed-form implementations exist; the strategy is configuration-selected).
* **Line Movement:** Market lines are retained by supersession rather than overwrite, giving open / high / low / close series per game, market, book, and outcome — the substrate for line-movement features and closing-line-value evaluation.
* **Multi-Split Convergence:** A scanner flags props where several independent situational splits agree directionally, surfacing convergence as its own signal.
* **Correlation Structure:** Within-game outcome correlation is modelled with a Gaussian copula over the simulation's own draw matrix, so parlay pricing reflects real dependence instead of assuming independent legs.

## 9. Calibration & Validation

* **Probability Calibration:** Raw model probabilities are mapped through fitted calibration curves (isotonic and beta implementations) before anything downstream reads them. Calibration is applied as a bias correction and cannot, by construction, change the model's ranking skill.
* **Walk-Forward Backtesting:** Evaluation is walk-forward on a bounded historical window where both event data and historical closing lines are obtainable. Models are scored against the de-vigged closing line, not against raw win/loss, and closing-line value is a first-class gate.
* **Next-Day Grading:** Every persisted prediction is graded against settled labels the following morning, producing a continuous, auditable accuracy series independent of the backtest harness.
* **Dark-Launch Gating:** Each model family sits behind its own feature flag. With a flag off, that slice of the output falls back to a baseline, so a family can be developed, backtested, and enabled independently without a coordinated release.
