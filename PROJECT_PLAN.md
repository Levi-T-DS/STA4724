# STA 4724: PCA Portfolio Construction and Backtesting

## 1. Established goal

**Can we use PCA fitted only on information available in the past to construct a portfolio that performs well in subsequent, unseen periods?**

Portfolio construction and backtesting are the main project. Descriptive plots and explained variance support that goal.

At each historical decision date, estimate relationships among returns, apply a defined allocation rule, hold the portfolio into the next period, and record what happens. Advance the date and repeat.

This is a plan only. No project code has been written or changed. The assignment rubric is still needed to check submission requirements.

## 2. Turn the goal into a testable experiment

| Approach | How the past informs the future | Additional rule needed |
| --- | --- | --- |
| PCA-based risk allocation | Historical co-movement estimates inform future portfolio risk | Convert the risk estimate into portfolio weights |
| PCA-assisted return forecasting | Historical component information becomes input to a return forecast | Define a forecasting model, horizon, and allocation rule |

**Recommended first experiment:** A PCA-based minimum-variance portfolio, tested against equal weighting and a comparable portfolio using ordinary sample covariance. This is a proposed starting strategy, not a claim that it will succeed.

**Primary hypothesis for this experiment:** The PCA risk estimate leads to lower out-of-sample volatility than the two comparison methods. Also evaluate returns, drawdowns, and costs. Lower volatility does not automatically establish better overall performance.

If success specifically means predicting winners or outperforming benchmark returns, include the forecasting experiment in Section 9 in the core scope. PCA identifies variation; it does not by itself specify whether prices will rise next.

## 3. Define the first portfolio rule

Use a **long-only portfolio that minimizes estimated variance using a PCA-based covariance estimate**.

At every rebalance:

1. Identify eligible stocks using information available on that date.
2. Fit PCA to the trailing training window of returns.
3. Retain enough components to explain a prespecified fraction of training variance. Start with 80% as a project convention, not a proven optimum.
4. Reconstruct the covariance contribution of those components and add each stock's residual variance on the diagonal.
5. Choose weights that minimize estimated portfolio variance under the constraints below.
6. Trade after the information cutoff, deduct costs, and hold until the next rebalance.

Proposed constraints: weights sum to 100%; each weight is between 0% and 10%; no shorting or leverage. Zero weights are allowed, so allocation also determines which stocks are held. At least 10 eligible stocks are needed. Predetermine a cash fallback for insufficient eligibility or optimization failure, and flag affected periods.

For the first version, use covariance PCA on **daily simple returns**, centered within the training window. This keeps risk estimation and portfolio accounting in the same units. Log-return analysis can remain a supporting comparison.

The proposed risk estimate is:

**Estimated covariance = retained PCA covariance + diagonal residual variances.**

With S the training sample covariance, V the retained directions, and Lambda their eigenvalues:

**Sigma_PCA = V Lambda V' + diag(diag(S - V Lambda V')).**

This models shared movements through retained components and treats remaining cross-stock residual covariances as zero. Keeping residual variances prevents discarded directions from appearing riskless. Check numerical validity and predetermine any small numerical floor.

Choose weights w to minimize **w' Sigma_PCA w** subject to the constraints. PCA direction weights themselves are not allocations: they can be negative and need not sum to one.

Standardized-return PCA can be a later comparison. Convert its reconstructed covariance back to return units using training-window standard deviations before allocating funds.

## 4. Data and historical eligibility

The notebook's saved outputs show 487 retained stocks but only 59 monthly return observations across 2021-2025. Centering permits at most 58 nonzero components across that entire sample. Rolling monthly windows would have even fewer observations.

**Proposed implementation change:** Obtain daily prices, start with a rolling window of 252 daily returns, and rebalance monthly. Obtain enough earlier history to initialize the first window. More observations help estimation but do not guarantee stability.

Record the data source, retrieval date, price adjustment convention, trading calendar, ticker history, and exclusions. Initially require 253 valid consecutive prices for 252 daily returns. Decide eligibility at each cutoff, not from completeness over the entire backtest.

**The universe must also respect time.** Prefer S&P 500 membership known at each historical date, including companies that later exited. A universe fixed from a genuine historical starting list is another option, provided later failures and exits remain accounted for.

If only today's constituent list is available, label the study a retrospective demonstration with survivorship and selection bias. Past-only PCA does not repair a future-selected universe.

Do not backfill from future prices or remove earlier holdings because later data are missing. Predetermine treatment of halts, delistings, acquisitions, and unavailable holding-period prices. Flag unresolved outcomes rather than silently dropping holdings or assuming zero returns. Document limitations of revised historical data and avoid counting dividends twice when adjusted returns already include them.

## 5. Walk-forward backtest

**Past returns -> fit PCA -> estimate risk -> choose weights -> trade -> observe subsequent returns -> advance the date.**

| Moment | Action |
| --- | --- |
| End of December 2023 | Fit using the latest 252 daily returns available through that close and calculate target weights |
| Next trading session's close | Execute under a stated closing-price execution approximation and charge costs |
| Until the next execution | Track holdings and allow weights to drift with returns |
| End of January 2024 | Refit using the latest past-only window and calculate new targets |
| Following session's close | Rebalance from actual drifted holdings to the new targets |

This one-session delay avoids assuming a trade at the same close needed to calculate the signal. Existing holdings earn returns until execution; new weights earn returns only afterward. At inception, hold cash until the first trade. Use the same timeline for all strategies.

Fit all preprocessing, PCA, component counts, and allocation inputs using information available at the cutoff. Do not fit PCA on the full dataset and reuse it for earlier decisions. See [scikit-learn's guidance on preprocessing leakage](https://scikit-learn.org/stable/common_pitfalls.html).

### Separate development from evaluation

Proposed schedule for daily 2021-2025 data:

- **2021:** Initial estimation history.
- **2022-2023:** Development backtest for debugging and choosing a small number of settings.
- **2024-2025:** Reserved evaluation after freezing the strategy definition.

During evaluation, monthly refitting may use earlier evaluation observations once they have occurred. The update procedure stays fixed; manually changing settings after viewing final performance is not allowed.

The notebook already explored the full date range. Disclose that exposure instead of describing these dates as completely untouched. Reserve an additional unexamined period if feasible. Two evaluation years also provide limited evidence across market conditions.

## 6. Benchmarks

Use identical dates, eligibility, execution timing, and cost assumptions:

| Portfolio | Purpose |
| --- | --- |
| Equal-weight eligible universe, rebalanced monthly | Simple allocation benchmark |
| Sample-covariance minimum-variance portfolio | Same objective and constraints; isolates the effect of replacing the risk estimate |
| PCA-covariance minimum-variance portfolio | Proposed strategy |

A broad-market total-return benchmark can add context. Its different weighting method means it cannot replace the controlled comparisons.

An optional shrinkage-covariance benchmark tests whether another way of stabilizing covariance also helps. See [scikit-learn's covariance documentation](https://scikit-learn.org/stable/modules/covariance.html). Record failures and apply the same predetermined fallback to both optimized strategies.

## 7. Define what "works" means before evaluation

For the risk-allocation experiment, the primary outcome is **realized out-of-sample volatility relative to both core benchmarks**. Also report:

| Measure | Question |
| --- | --- |
| Cumulative return and annualized compounded return | How much did the portfolio grow? |
| Annualized volatility | Did allocation reduce fluctuations? |
| Maximum drawdown | How severe was the worst peak-to-trough loss? |
| Sharpe ratio with a documented risk-free series | How did excess returns compare with volatility? |
| Turnover and costs | How expensive was maintaining the allocation? |
| Largest weight and effective number of holdings | How concentrated was the portfolio? |

Calculate performance from simple returns and weights actually held before each return. A weighted sum of stock log returns is not generally the portfolio log return. Do not assume weights stay constant between monthly rebalances.

**Illustrative costs:** Charge 10 basis points per dollar traded on both buys and sells, including initial purchases. Measure traded value from drifted holdings: a sale and a purchase both count. Show 0 and 25 basis points as sensitivity cases. These are experimental assumptions, not measured execution costs. Charge any final liquidation consistently.

Show gross and net results. Lower volatility with much lower returns is a tradeoff. An improvement that disappears after costs is a meaningful negative finding.

Use only a few prespecified sensitivity checks, such as 252 versus 504 training days or 70% versus 80% retained variance. Compare common dates with sufficient history. Select settings during development, not by finding the best final-period result.

Report separate evaluation years and consider a paired block bootstrap of strategy and benchmark returns for uncertainty ranges. A favorable equity curve alone is weak evidence of persistent advantage.

Timing errors, universe selection, repeated strategy selection, and costs are established backtesting concerns; see [Portfolio Optimization, Section 8.2](https://portfoliooptimizationbook.com/book/8.2-seven-sins.html).

## 8. Corrections needed during implementation

- Calculate return correlations; the current correlation matrix uses price levels.
- Separate component directions from scores: the table labeled eigenvectors currently contains scores.
- Replace full-period missing-data filtering with dated eligibility decisions.
- Replace full-period PCA with repeated historical-window fits.
- Track information cutoffs, decision dates, execution dates, holdings, and returns separately.
- Use the allocation rule above for the first experiment instead of the unfinished "closest asset to principal direction" idea. Stock return histories and component directions have different dimensions and cannot be compared directly with SSE.

See the [PCA API reference](https://scikit-learn.org/stable/modules/generated/sklearn.decomposition.PCA.html) for the distinction between directions, scores, and explained variance.

## 9. If we also want to predict future returns

Add a separate forecasting experiment:

1. At each cutoff, fit PCA inside the trailing training window.
2. Use that transformation to construct historical component-score features.
3. Define a target, such as each stock's next-month simple return.
4. Fit a simple regularized forecasting model using only examples whose entire target period has finished by the cutoff. Exclude examples whose outcomes overlap a later validation or evaluation period.
5. Forecast the next period and combine forecasts with the risk estimate under the same portfolio constraints. Prespecify the return-versus-risk tradeoff during development.
6. Compare against the risk-only strategy and the same forecasting model without PCA, using time-ordered development splits and identical costs.

A large eigenvalue, positive component weight, or low reconstruction error does not imply positive future returns. Refit the forecasting model consistently with each PCA basis because component signs and ordering can change.

This directly tests whether PCA helps forecast returns. First complete and audit the portfolio simulator so forecasting results can be evaluated correctly.

## 10. Work plan and deliverables

| Phase | Work | Completion evidence |
| --- | --- | --- |
| 1 | Set hypothesis, universe, allocation, timing, dates, and metrics | Written experiment specification |
| 2 | Audit daily data and historical eligibility | Data-quality report and dated eligibility records |
| 3 | Work through one historical decision | PCA inputs, risk estimate, weights, and constraint checks |
| 4 | Establish equal-weight accounting | Manually checked trades, weight drift, returns, and costs |
| 5 | Backtest both optimized strategies | Dated holdings, trades, costs, returns, and failures |
| 6 | Finish development and freeze settings | Record of settings and variants tried |
| 7 | Run reserved evaluation without changing rules | Performance table and uncertainty discussion |
| 8 | Explain whether and why PCA helped | Report and presentation as required by the rubric |

Essential figures: wealth after costs, drawdowns, rolling volatility, turnover, and retained component count over time. Include a small stock-weight or sector-exposure illustration to explain what PCA learned.

Essential audit checks: changing future data must not change earlier decisions; weights satisfy constraints; trades follow their information cutoff; costs reconcile with traded value; holdings reproduce portfolio returns.

Rolling refitting is part of the main design. Exponential decay and alternative scaling are later extensions.

**Report-ready project statement:** "We investigate whether principal component analysis of historical stock returns can improve portfolio construction. At each rebalance date, we fit PCA using only previously available observations, construct a constrained portfolio from the resulting risk estimate, and evaluate its subsequent performance against simple and non-PCA benchmarks in a walk-forward backtest that accounts for trading costs."
