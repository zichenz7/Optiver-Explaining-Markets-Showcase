# Research and validation

[Project overview](README.md) · [Public model card](https://explainingmarkets.ai/models/s_e01164bd47b2)

## Target and inputs

The output is a percentile in [0, 1] for next-day abnormal return, with 0.50 as neutral. Inputs are the event's focal ticker and earnings-call facts, a historical prior-surprise lookup, and valid option-implied absolute earnings-move statistics when available.

The v2 artifact was trained on **8,656 usable events across 2025Q4–2026Q3**. The deployed historical lookup is frozen through 2026Q3. An unknown ticker receives a neutral prior; it does not trigger a live historical lookup.

## Current model (v12, October 31, 2026)

For each event, v12 compares the current earnings-call facts with the latest eligible disclosure for the same company from an earlier quarter. The prior must be at most 365 days old. Both its event time and archive generation time must precede the current prediction cutoff. Both reports must also contain informative operating text.

For an eligible pair, a second Ridge ensemble uses the current TF-IDF features and their difference from the prior report. Its forecast and the v2 forecast receive equal weight. If the timing or text checks fail, v12 returns the v2 forecast.

Historical time splits and a Q4 replay informed candidate selection. Q4 outcomes informed that choice but were excluded from fitting the regression coefficients. Independent post-freeze validation is pending. The table below documents the earlier v2 comparison.

## Baseline model (v2)

1. **Encode facts.** Word unigrams and bigrams become sublinear, L2-normalized TF-IDF features, with a vocabulary capped at 30,000 features.
2. **Learn complementary signals.** A Ridge outcome head learns return-percentile residuals after a quarter-specific linear earnings-surprise adjustment. A second Ridge head distills the historical reference forecasts supplied in the official archive.
3. **Combine standardized signals.** Fixed ensemble weights combine the heads and a prior-surprise mean-reversion adjustment. The historical prior is constructed from completed training quarters.
4. **Scale the forecast around neutral.** A valid option-implied move scales the signal's distance from 0.50. This is a forecast-scaling heuristic, not a calibrated uncertainty estimate. Missing or invalid option statistics leave that scale unchanged. The output is clipped to [0, 1].

Distillation uses archived reference outputs during training; it does not call a live language model at prediction time. Earnings-preview text is not an input to the published v2 model. A separate preview-text candidate was evaluated and excluded.

## Quarterly walk-forward evaluation

**Archived comparison of v2 and v1.**

Each fold trains on all available earlier quarters and evaluates on the next quarter. The vocabulary, Ridge heads, signal standardization, and prior-surprise lookup are fitted from that fold's training data.

| Validation quarter | Training quarters | v1 ΔR² | v2 ΔR² | Absolute improvement |
| --- | --- | ---: | ---: | ---: |
| 2026Q1 | 2025Q4 | 0.0225 | 0.0310 | +0.0085 |
| 2026Q2 | 2025Q4–2026Q1 | 0.0183 | 0.0267 | +0.0084 |
| 2026Q3 | 2025Q4–2026Q2 | 0.0479 | 0.0611 | +0.0132 |

**Metric:** ΔR² is the increase in the evaluation regression's R² when the forecast is added alongside realized earnings-surprise percentile, compared with a regression using surprise alone. It measures incremental explanatory information under the competition's scoring setup; it is not a percentage return, directional accuracy, or deployable trading profit.

**Provenance:** these rounded figures are recorded historical research results retained in the private v2 training metadata and local model documentation. They were checked against those records for this showcase; the historical training experiment was not rerun for this documentation update. The public model card documents the method, but does not publish this result table.

**Interpretation:** these folds were used to compare candidates and select the ensemble, so they are validation folds, not an untouched final test set. The v2 artifact was subsequently refitted through 2026Q3. Current-quarter provisional observations and live leaderboard rank are intentionally not presented as final validation results.

**Option-data caveat:** option statistics first appear in the Q3 historical data. Where the earlier training set has no option-move reference, the research code uses the evaluation quarter's pre-event cross-sectional mean as a common scale. A uniform scale does not change the regression score before clipping, but this is still a retrospective normalization detail, not a fully sequential estimate. The deployed artifact uses a frozen historical reference.

## Decisions and trade-offs

| Decision | Benefit | Cost or assumption |
| --- | --- | --- |
| Linear lexical model | Deterministic, lightweight local inference | Limited semantic and numerical reasoning |
| Archived teacher distillation | Learns from reference forecasts without a live provider dependency | Inherits information and biases in the reference targets |
| Prior-surprise adjustment | Adds a structured temporal signal | Assumes historical mean reversion persists |
| Option-move scaling | Adjusts distance from neutral to event uncertainty | Depends on coverage and statistic quality |
| Frozen artifact and lookup | Makes the information boundary inspectable | Needs deliberate retraining as regimes and coverage change |

## Limitations

The model can miss negation, management tone, numerical relationships, and the distinction between reported facts and market expectations. Historical gains may not persist under a different company mix or market regime. Repeated model selection on a small number of quarters can overstate generalization. No transaction costs, execution model, portfolio construction, or trading P&L are established by this evaluation.
