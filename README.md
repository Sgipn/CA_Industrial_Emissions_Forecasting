# Forecasting California Industrial Petroleum CO2 Emissions

### Overview
This project compares three forecasting approaches, SARIMA, Lasso, and Gradient Boosting, on a long-run, trend-dominated emissions series, and looks closely at what "out-of-sample" evaluation actually means once the three models are given different amounts of information at prediction time. The core finding is less about which model wins outright and more about how easy it is to build a model comparison that quietly favors one approach without meaning to, and what a fair comparison looks like once that's corrected.

### Questions
- Is the series stationary, and what does that imply for model specification?
- How much of the series is trend versus seasonality?
- How accurately can a classical time-series model (SARIMA) versus feature-based models (Lasso, Gradient Boosting) forecast the series?
- What does "out-of-sample" mean when the models don't have access to the same information at prediction time, and does the answer to "which model is best" change depending on the forecasting protocol?

### Data
- Source: EIA state-level emissions series, EMISS.CO2-TOTV-IC-PE-CA.A (California, industrial petroleum)
- Frequency: annual, 1970-2021, forward-filled to monthly to give the models more observations to work with
- The forward-fill means within-year values are identical by construction, which matters for interpreting short-lag autocorrelation in the ACF/PACF diagnostics.

### Approach

##### Exploratory Analysis
- The raw series is visibly non-stationary: a long climb through the 1970s-80s, a plateau, then a slower decline after the mid-2000s
- An Augmented Dickey-Fuller test confirms this formally (ADF statistic -2.11, p = 0.24)
- ACF/PACF on the original series decay slowly rather than cutting off, consistent with a trending, non-stationary process
- A seasonal decomposition shows an essentially flat seasonal component, so no seasonal order was carried into the SARIMA specification

##### Modeling Part 1: SARIMA
- SARIMA(1,1,1), no seasonal terms: first-differencing for the non-stationarity, AR(1)/MA(1) for the remaining short-run autocorrelation
- Residual diagnostics are very clean when checked against the full series (Ljung-Box p is approximately 1.0), though this cleanliness is partly a function of being checked against data the model was fit on, see the evaluation section below

##### Feature Engineering
For the Lasso and Gradient Boosting models: Lag1, Lag24 (one month and two years back), MA3, MA6 (rolling 3- and 6-month averages), and Month as a sanity check on seasonality.

##### Modeling Part 2: Lasso and Gradient Boosting
- 80/20 time-based split: training on 1972 through early 2011 (471 months), holding out April 2011 through January 2021 (118 months) as a genuine test set
- Lasso tuned via 5-fold TimeSeriesSplit cross-validation
- Gradient Boosting fit on the same features, with SHAP used to check which features the model actually relies on

##### Model Evaluation
- Metrics: RMSE, MAE, MAPE, MBE
- A first pass at comparing the three models evaluated SARIMA on fitted values from the full series while evaluating Lasso and GBR only on the true holdout, an apples-to-oranges comparison that happened to not change the qualitative conclusion but was not a fair basis for it
- Two corrected comparisons were run instead: a genuinely blind forecast, where SARIMA is fit on training data only and forecasts forward with no updating, and a one-step-ahead comparison, where SARIMA is given the same access to true recent values that Lasso's lag features get for free
- Which model looks best depends on which of these two protocols matches the real forecasting task

##### Post-Hoc Analysis
- Actual-vs-predicted and residuals-over-time plots to see where each model diverges from the truth, particularly around the 2020 COVID-related disruption
- Cumulative forecast error to distinguish a model with a small, persistent bias (Lasso) from one that is more reactive but noisier (Gradient Boosting)

### Results

##### Predictive Performance
- SARIMA (in-sample, full series): RMSE 0.831, MAE 0.180, MAPE 0.61%
- Lasso (true out-of-sample test set): RMSE 0.192, MAE 0.120, MAPE 0.48%
- Gradient Boosting (true out-of-sample test set): RMSE 0.629, MAE 0.458, MAPE 1.88%
- SARIMA, blind 118-month forecast, no updating: RMSE 0.900, MAE 0.776, MAPE 3.09%, the forecast converges almost immediately to a flat line and stays there
- SARIMA, one-step-ahead with train-only parameters: RMSE 0.279, MAE 0.062, MAPE 0.25%, beating Lasso on MAE and MAPE while losing narrowly on RMSE

##### Residual Diagnostics
- SARIMA: essentially no leftover autocorrelation when checked in-sample (Ljung-Box p ≈ 1.0)
- Lasso: some leftover autocorrelation (Ljung-Box p = 0.0047), no strong evidence of heteroskedasticity (Breusch-Pagan p = 0.33)
- Gradient Boosting: substantial leftover structure (Ljung-Box p ≈ 2e-36), consistent with overfitting to training-window noise

##### Feature Importance
- Lasso's cross-validated regularization parameter (alpha = 0.0047) barely shrinks anything; all four engineered features survive, led by MA3 (coefficient 7.93), followed by MA6 (-1.62) and Lag1 (-1.62) at roughly equal and opposite magnitude, with Lag24 contributing little (-0.04)
- Gradient Boosting's SHAP values point to the same conclusion: MA3 and MA6 dominate, Lag1 contributes little, and Month confirms seasonality isn't doing meaningful work in this series

##### Stability
- Lasso's cumulative forecast error drifts down gradually and persistently, consistent with a mild, consistent negative bias from L1 shrinkage
- Gradient Boosting's cumulative error swings harder, with a steep decline through about 2016, a partial correction, and a sharp break around the 2020 disruption

### Takeaways
- SARIMA has the cleanest residual diagnostics of the three models, but that cleanliness is partly an artifact of being checked against data it was fit on
- Lasso is the most reliable choice for a genuine long-horizon, blind forecast, and its regularization path doubles as a feature-selection tool: four simple engineered features capture nearly all of the signal
- Gradient Boosting reacts most to real structural breaks like 2020 but pays for that flexibility with the weakest residual diagnostics of the group
- "Which model is best" is not a fixed answer; it depends on whether the real task looks like a blind long-range extrapolation or a rolling, one-step-ahead forecast that gets to update on new data as it arrives
- An evaluation that isn't explicit about what information each model has access to at prediction time can quietly favor one model over another without the difference being obvious from the final numbers alone

This project treats the evaluation protocol as part of the modeling decision, not an afterthought. In settings where different models have access to different amounts of information at prediction time, the fairest comparison is often not a single number but a description of which model wins under which forecasting task.
