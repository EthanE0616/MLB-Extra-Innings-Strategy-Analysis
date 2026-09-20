# MLB Extra-Inning Strategy Analysis

A baseball analytics project using Statcast data, statistical modeling, and empirical simulation to study how MLB teams should approach the automatic runner in extra innings.

## Project Motivation

Every baseball fan has some sort of "baseball intuition". Mine was thrown for a loop when I watched a compilation of several extra inning outings. I thought to myself: "If you are the away team and you score nothing in the top of the 10th, you are nearly guaranteed to lose, so just bunt and significantly increase your chance of scoring, and then play to prevent the run on defense". This dilemma sparked what would become an extensive analysis of this particular baseball strategy. 

While watching extra-inning games, I kept questioning managerial decisions with the automatic runner on second. Sometimes a team would bunt immediately and give away an out. Other times, a team would swing away when it seemed like moving the runner to third was the obvious choice. Sometimes this would get quite infuriating, so naturally I wanted to find out why.

The project asks a simple question with a surprisingly complicated answer:

> **When does a bunt, swing-away approach, or intentional walk actually maximize win probability in MLB extra innings?**

The key idea is that the best strategy should not necessarily maximize expected runs. It should maximize the probability of winning the game.

```math
\text{maximize } E[\text{runs}] \neq \text{maximize } P(\text{win})
```
That distinction becomes especially important under MLB's automatic-runner rule. One run in the top half creates a lead that still has to be defended. One run in the bottom half of a tie game ends the game immediately.

## Research Questions

The analysis focuses on four main questions:

1. **Should the visiting team bunt or swing away with the automatic runner on second and nobody out?**
2. **Does the optimal offensive strategy change in the bottom half of the inning?**
3. **Is bunting most valuable when exactly one run is needed to win?**
4. **Does intentionally walking the first batter provide a defensive advantage in the bottom half of a tied extra inning?**

A secondary analysis also explores whether hitter quality changes the bunt decision.

---

## Data

The project uses pitch-level **MLB Statcast data** accessed through [`pybaseball`](https://github.com/jldbc/pybaseball).

The notebook is configured for:

- **Start date:** April 1, 2021
- **End date:** October 1, 2025
- **Primary sample:** innings 10 and later
- **Starting state:** runner on second, zero outs

The analysis uses variables including:

- game and inning identifiers
- inning half
- batter and pitcher IDs
- base state
- outs
- plate-appearance events
- pitch descriptions
- batting-team score
- post-play batting-team score
- home and away scores
- Statcast wOBA values

Raw Statcast files are stored as daily CSVs in `data/raw/`.

### Why `post_bat_score` matters

Walk-off innings can end immediately after the winning play. Using only the score before the play can therefore miss the final run. The notebook uses `post_bat_score` when constructing half-inning outcomes so walk-off runs are counted correctly.

---

## Methods

### 1. Constructing the Extra-Inning Sample

Pitch-level Statcast data are converted into one starting-state observation per qualifying extra-inning half.

A qualifying starting state requires:

- inning 10 or later
- runner on second
- zero outs

The notebook then attaches:

- runs scored during the half-inning
- final game result
- batting-team win indicator
- score differential from the batting team's perspective

---

### 2. Identifying Bunt Attempts

A major methodological choice is to analyze **bunt attempts**, not only successful sacrifice bunts.

A plate appearance is classified as a bunt attempt if any pitch description contains `"bunt"`.

This matters because a batter who:

1. attempts a bunt,
2. fouls it off,
3. later strikes out,

still represents a managerial decision to bunt.

Using only final events labeled `sac_bunt` would incorrectly move failed bunt attempts into the swing-away group.

---

### 3. Controlling for Hitter Quality

Managers do not randomly choose who bunts.

To partially account for this selection effect, the notebook calculates each batter's **season-to-date wOBA before the game being analyzed**.

Early-season wOBA is shrunk toward league average:

\[
\text{Adjusted wOBA}
=
\frac{
PA \times wOBA +
100 \times wOBA_{\text{league}}
}{
PA + 100
}
\]

This prevents very small early-season samples from being treated as fully reliable estimates of hitter quality.

---

### 4. Bunt vs. Swing-Away Analysis

The project compares bunt attempts and swing-away plate appearances in tied extra innings using:

- raw win rates
- average runs scored
- probability of scoring at least one run
- probability of scoring at least two runs
- two-proportion tests
- logistic regression

The main regression includes a **bunt × inning-half interaction**:

\[
\text{logit}(P(\text{win})) =
\beta_0 +
\beta_1 \text{Bunt} +
\beta_2 \text{Bottom} +
\beta_3(\text{Bunt}\times\text{Bottom}) +
\beta_4 \text{Inning} +
\beta_5 \text{Adjusted wOBA}
\]

Standard errors are clustered by game.

---

### 5. Empirical Strategy Engine

The project then moves from individual plate appearances to complete extra-inning strategy.

For the visiting team, the model uses the empirical distribution of runs scored under each offensive strategy.

If the away team scores \(r\) runs, the home team needs \(r+1\) runs to win.

The home-team win probability is calculated as:

\[
P(\text{Home Win})
=
\sum_r
P(\text{Away scores } r)
\times
P(\text{Home wins}\mid\text{needs }r+1)
\]

The bottom-half probabilities are estimated by:

- number of runs needed
- offensive strategy

Small samples are stabilized with **Beta(1,1) smoothing**.

The strategy engine compares:

**Visitor**
- Bunt attempt
- Swing away

**Home**
- Always swing away
- Bunt only when exactly one run is needed
- Bunt when one or two runs are needed

---

### 6. Game-Level Bootstrap

To quantify uncertainty, the notebook performs a **cluster bootstrap at the game level**.

Entire games are resampled rather than individual half-innings, preserving dependence between observations from the same game.

The strategy probabilities are recalculated for every bootstrap sample.

This avoids the slow nested Monte Carlo approach and measures uncertainty from the historical sample itself.

---

### 7. Intentional-Walk Analysis

Intentional walks are treated as a **defensive strategy**.

The primary intentional-walk analysis focuses on tied bottom-half situations, where one run wins the game.

The notebook compares:

- intentionally walking the first batter
- pitching to the hitter

using:

- Fisher's exact tests
- logistic regression
- predicted scoring probabilities for a league-average hitter

---

## Key Findings

### 1. The value of a bunt is state dependent

The analysis does **not** support a universal rule that bunting is always good or always bad in extra innings.

The strongest pattern is that strategy changes with the value of the next run.

### 2. Top half: preserve outs

The visiting team's point estimate favored **swinging away** rather than bunting.

The estimated visitor advantage from swinging away was approximately:

**+5.10 percentage points**

with a 95% bootstrap interval of:

**-4.31 to +13.35 percentage points**

Because the interval crosses zero, the evidence favors swinging away but is not strong enough to support an absolute "never bunt" rule.

### 3. Bottom half, one run needed: bunting becomes more valuable

The strongest result in the project comes when exactly one run wins the game.

A home-team policy of **bunting only when one run is needed** improved estimated win probability by approximately:

**+4.52 percentage points**

with a 95% bootstrap interval of:

**+0.05 to +9.09 percentage points**

This supports the idea that sacrificing an out can become worthwhile when the offense only needs to manufacture a single run.

### 4. When multiple runs are needed, preserve outs

The data do not support bunting simply because the team is batting in the bottom half.

When two or more runs are needed, the value of preserving outs and maintaining multi-run upside becomes more important.

### 5. Intentional walks showed no clear defensive benefit

In tied bottom-half situations, intentionally walking the first hitter did not significantly reduce the probability that the offense scored.

The adjusted model also found no statistically significant defensive benefit from the intentional walk.

### Main Strategic Takeaway

> **Bunt when one run is the whole game. Preserve outs when the offense may need more.**

The most important variable is not simply whether the team is home or away.

It is the **value of the next run**.

---

## Repository Structure

```text
extra-innings-strategy/
│
├── extra_innings_strategy_cleaned.ipynb
├── README.md
│
├── data/
│   ├── raw/
│   │   └── statcast_YYYY-MM-DD.csv
│   └── processed/
│       └── extra_innings_strategy.csv
│
├── figures/
│   ├── bunt_win_probability_effect.png
│   ├── home_bunt_bootstrap_ci.png
│   └── intentional_walk_scoring_probability.png
│
└── results/
    ├── bunt_comparison.csv
    ├── strategy_matrix.csv
    └── bootstrap_strategy_summary.csv
```

### Note on raw data

The raw Statcast files can be large and should generally **not be committed to GitHub**.

A `.gitignore` entry such as the following is recommended:

```gitignore
data/raw/
.ipynb_checkpoints/
__pycache__/
.DS_Store
```

The notebook can download the raw data automatically if needed.

---

## Running the Notebook

### 1. Clone the repository

```bash
git clone YOUR_REPOSITORY_URL
cd YOUR_REPOSITORY_NAME
```

### 2. Create a Python environment

Using `venv`:

```bash
python -m venv .venv
```

Activate it on macOS/Linux:

```bash
source .venv/bin/activate
```

Activate it on Windows:

```bash
.venv\Scripts\activate
```

### 3. Install dependencies

```bash
pip install pandas numpy matplotlib scipy statsmodels pybaseball jupyter
```

### 4. Open the notebook

```bash
jupyter notebook extra_innings_strategy_cleaned.ipynb
```

You can also open the notebook in VS Code or JupyterLab.

### 5. Download Statcast data

The notebook expects daily Statcast CSV files in:

```text
data/raw/
```

If the files do not already exist, change:

```python
DOWNLOAD_RAW_DATA = False
```

to:

```python
DOWNLOAD_RAW_DATA = True
```

and run the download section.

The notebook will:

1. query Statcast through `pybaseball`
2. save daily CSV files to `data/raw/`
3. skip dates that have already been downloaded

Once the files are available locally, set `DOWNLOAD_RAW_DATA = False` so future runs do not repeatedly query Statcast.

### 6. Run the notebook from top to bottom

The notebook will automatically create:

```text
data/processed/
figures/
results/
```

and save the main processed dataset, result tables, and figures used in the analysis.

---

## Reproducibility

The project uses a fixed random seed:

```python
RANDOM_SEED = 42
```

The game-level bootstrap currently uses:

```python
n_bootstrap = 500
```

Increasing the number of bootstrap samples can improve the stability of the uncertainty estimates at the cost of additional runtime.

---

## Limitations

This project is observational, not experimental. Managers do not randomly choose when to bunt or issue an intentional walk, so unobserved selection bias remains possible even after controlling for hitter quality.

Other limitations include:

- relatively small bunt samples
- very small bunt samples after splitting by hitter quality
- walk-off censoring in bottom-half run totals
- no explicit controls for runner speed or bunting skill
- no pitcher-quality or platoon adjustments
- no bullpen-strength adjustment
- no on-deck hitter or full-lineup context
- no park, weather, or defensive-positioning controls

The results should therefore be interpreted as evidence for **state-dependent strategy**, not as universal causal rules.

---

## Future Work

A natural extension is a full extra-inning decision engine that incorporates:

- batter quality
- pitcher quality
- handedness
- runner speed
- batter bunting ability
- on-deck hitter quality
- bullpen strength
- defensive positioning

A future version could take a live game state such as:

```text
Bottom 10th
Tie game
Runner on second
0 outs
Weak hitter at the plate
Elite runner
Strong hitter on deck
```

and return estimated win probabilities for:

```text
Bunt
Swing away
Intentional-walk response
```

The goal would be to move from analyzing what teams historically did to estimating what they **should do** in a specific game state.

---

## Tools Used

- Python
- pandas
- NumPy
- Matplotlib
- SciPy
- statsmodels
- pybaseball
- MLB Statcast

---

## Author

This project began as a personal baseball question: after watching enough extra-inning games and disagreeing with managerial decisions, I wanted to see what the data actually said.

The result is an attempt to combine baseball intuition with statistical modeling and simulation to better understand one of the most interesting strategic situations created by MLB's automatic-runner rule.
