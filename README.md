# F1 2026 Hungary & Netherlands: Undercut/Overcut Breakeven Model

A personal project that treats the undercut/overcut pit call as a cost-benefit problem instead of a physics one, using real FastF1 lap data from the 2026 Hungarian and Dutch Grands Prix.

The timing data doesn't give tyre wear or grip level, so everything here is built from lap times, tyre age, compound, and pit in/out timestamps instead. Degradation and pit loss are both regression outputs, not physical quantities.

## What's here

* `f1_undercut_breakeven_model.ipynb` - the whole pipeline, data pull through backtest
* `f1_breakeven_findings.docx` - write-up of the results and limitations

Running it prints the regression tables and shows two plots inline, the degradation curve and the pit-loss distribution. No output folder, everything stays in the notebook.

## What it does

* Fits tyre degradation as a fixed-effects panel regression (driver fixed effects, clustered standard errors) instead of assuming a flat per-lap wear rate
* Estimates pit-lane loss from each driver's own pace just before and after their stop, instead of pit lane length and speed limit
* Solves the breakeven gap as algebra on the regression coefficients, no simulated overtaking
* Adds a simple best-response rule for whether a rival should cover the undercut
* Finds real undercut/overcut episodes straight from the timing data (drivers close together on track, one pits before the other) instead of hand-picking famous battles, then backtests the model against them

## Findings

Degradation came out U-shaped, not linear, tyres actually get quicker for the first 10 to 12 laps of a stint before wear takes over. Pit loss is mostly the out-lap, not the in-lap, which is the opposite of what I expected. The breakeven model called the outcome correctly on 76% of the 34 real pit sequences it found across both races, though a good chunk of that is the model correctly predicting failed undercuts rather than picking winners, since pit loss usually swamps the maths over a short window.

## What's broken (and what's still broken)

First working version of the backtest scored 0% accuracy. Every prediction was wrong, and it came down to two bugs, both about checking the gap at the wrong moment:

1. The backtest checked the outcome after the rival had also pitted, by which point both drivers have paid pit loss once and it cancels out. The formula only counts pit loss once, so the backtest was quietly testing a different claim than the one the formula makes
2. The starting gap was measured on the exact lap the undercutting driver pits, but FastF1's cumulative time for that lap already includes the stop, so the "before the stop" gap already had some of the stop baked into it

Both fixed now, checked against synthetic data with a known true degradation curve and pit loss, notebook recovers the true numbers. Also hit a `LinAlgError: SVD did not converge` in the regression, caused by a few laps with missing `TyreLife` values in FastF1's own data. Fixed with a NaN check before fitting.

Explained but not yet fixed: some of the pit sequences flagged happen on lap 2. Verstappen crashed early in the Dutch Grand Prix, triggering a red flag, and several drivers used the stoppage to change tyres for free. Hadn't accounted for this before, worth investigating properly. That's not an undercut or overcut in any real sense, just teams taking a free pit stop while the field was stationary. The detection logic doesn't check for red flags, only `TrackStatus == '1'` laps get filtered out of the regression, not the episode finder, so these still show up as candidate episodes. Fix is to exclude pit stops that happen under a red flag when scanning for episodes, not done yet.
