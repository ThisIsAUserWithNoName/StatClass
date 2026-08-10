## License
StatClass is open-source software released under the MIT License. See the [LICENSE](LICENSE) file for details.
# StatClass
Python CLI alternative to Stata, R, SAS.
# StatClass 3.17.0

StatClass is a command-line statistics program for CSV, tab-separated,
semicolon-separated, and whitespace-separated data. It is organized as a
small Python package while preserving the familiar command:

```bash
python stats.py DATAFILE [OPTIONS]
```

The program requires NumPy, pandas, SciPy, and Matplotlib. In Anaconda,
activate the desired environment and verify the dependencies:

```bash
conda activate base
python stats.py --dependencies
```

`--requirements` is an alias for the same command. It does not require a data
file. The output reports the versions actually installed in the active
Anaconda or Python environment, for example:

```text
StatClass dependency versions
statclass=3.17.0
python=3.12.8
numpy=2.2.1
pandas=2.2.3
scipy=1.15.0
matplotlib=3.10.0
```

Your installed version numbers may differ from this example.

If one is missing:

```bash
conda install numpy pandas scipy matplotlib
```

The artificial example datasets are distributed in the companion
`statclass_3.17.0_example_csvs.zip` archive. After unpacking it, set `DATA_DIR`
to that archive's `csv` directory before using a command that names a
`*_sample.csv` file:

```bash
DATA_DIR="/path/to/statclass_3.17.0_example_csvs/csv"
```

## Input size and memory preflight

StatClass does not impose an arbitrary global row or column limit, round input
values, or silently discard rows. Before pandas parses a data file, StatClass
first checks a conservative file-size lower bound. If that is safe, it uses the
file size and a bounded 128-row sample to estimate the number of rows, columns,
cells, and peak memory needed for the parser, retained raw table, and numeric
copy. It compares the estimate, with a 1.5x safety factor, with the memory
currently available to the process. Linux control-group limits are included
when present.

For an ordinary file, the text report shows the estimated peak load memory and
available memory. JSON retains the complete `input_resource_preflight` record,
including the sample basis, estimated dimensions, byte estimates, safety
factor, and status. This is explicitly a planning estimate, not an exact
measurement.

If the safe estimate exceeds available memory, loading stops before pandas
parses the table. The error reports the file size, estimated peak, required
headroom, and currently available memory, and confirms that no rows were
loaded, truncated, or silently discarded. If memory availability cannot be
detected portably, loading proceeds and the status is
`available_memory_unknown`; an actual Python `MemoryError` or operating-system
out-of-memory error still receives a specific diagnostic.

Computational limits remain method-specific. For example, a method with an
explicit quadratic-time limit may return `not_run` while summaries and
linear-time analyses on the same complete file still run. The program never
turns a full-data request into an unannounced subset analysis.

## Significance levels and numeric precision

`--alpha` accepts any finite real number strictly between 0 and 1. The default
is `.05`; custom values such as `.025`, `.0345159`, and scientific notation
such as `1e-4` are valid. StatClass rejects zero, one, negative values, `nan`,
infinities, and nonnumeric text instead of silently changing the input.

```bash
python stats.py --mean-ci 25 72.4 11.8 --alpha .0345159
```

The corresponding confidence or Bayesian credible level is `1 - alpha`. For a
two-sided equal-tail interval, each tail contains `alpha / 2`.

`--precision` controls only the maximum number of decimal places shown in the
human-readable terminal and text-file reports. It accepts integers from 1
through 15 and defaults to 6. Calculations use the unrounded floating-point
value, and numeric JSON fields retain full floating-point precision. Very small
nonzero results use scientific notation instead of being displayed as zero.

```bash
python stats.py --mean-ci 25 72.4 11.8 \
  --alpha .0345159 --precision 8
```

## Discrete moments and likelihood curvature

`--distribution-info` reports closed-form first raw, second raw, and second
central moments for every supported discrete distribution. This includes
Bernoulli, binomial, discrete uniform, geometric, hypergeometric,
negative-binomial, Poisson, and beta-binomial models. The formulas are printed
in the text report and retained as named fields in JSON. The usual identity
`E[X^2] = Var(X) + E[X]^2` is also recorded so results can be checked directly.

```bash
python stats.py --distribution-info beta-binomial 20 2 5
python stats.py --distribution-info negative-binomial 5 .3
```

Likelihood-based estimators now use a shared numerical-curvature layer where
an analytic information matrix is not available. It provides scaled central
finite differences, a comparison after halving the step, symmetry and rank
checks, eigenvalue and condition diagnostics, and positive-definiteness
checks. Covariance matrices are obtained with Cholesky linear solves rather
than an explicit inverse or pseudoinverse. Delta-method covariance matrices
are included for transformed beta-binomial, NB2 dispersion/size, and mixed
model variance parameters.

The text report shows a compact summary; JSON retains the information,
covariance, transformation, and numerical-Hessian details. If curvature is
singular, nearly singular, indefinite, or nonfinite, StatClass stops covariance
inference instead of manufacturing standard errors. When a variance parameter
is effectively on a zero boundary, ordinary two-sided Wald intervals are
withheld and the model-specific boundary likelihood-ratio result is used.

Expected and observed information are labeled explicitly. For the elementary
one-parameter discrete estimators, expected Fisher information is evaluated at
the MLE and equals observed information at an interior MLE.

## Numerical reliability and failure reporting

Likelihood optimizers now return a uniform numerical status in both text and
JSON output. It records the method, optimizer message, iteration and evaluation
counts when available, parameter and objective finiteness, the final gradient
infinity norm when an analytic gradient exists, and whether the stationarity
tolerance was reached. A reported optimizer success with a material remaining
gradient is retained as a visible warning. A failed optimizer, invalid
curvature, or unavailable Hessian-based inference is never presented as an
ordinary completed Wald result.

Information-matrix diagnostics include finiteness, symmetry error, numerical
rank, minimum and maximum eigenvalues, positive definiteness, condition number,
and a near-singular flag. Model-based covariance is computed only for finite,
positive-definite, well-conditioned information. The covariance label states
whether the source is expected or observed information and makes clear that
these likelihood-model covariances are not sandwich covariances. Transformed
parameters use the multivariate delta method, with boundary cases reported
separately.

Distribution probabilities and iterative power calculations validate their
numeric outputs. Nonfinite or materially out-of-range probabilities are errors;
only floating-point deviations within `1e-12` of 0 or 1 may be clipped as
roundoff. Sample-size searches report bracketing and bisection effort, and
detectable-effect and negative-binomial profile-limit calculations report
Brent convergence and the final residual. Known mathematical endpoint limits
are handled explicitly rather than allowing a dependency-specific `NaN` to
leak into a report.

Continuous interval probabilities use a lower-tail CDF difference, an
upper-tail survival-function difference, or a logarithmic version when needed.
This avoids cancellation such as subtracting two values that have both rounded
to one. If a positive probability is smaller than floating-point output can
represent, the report labels zero as underflow and retains the log probability
instead of presenting zero as exact.

The supported continuous families use their specialized PDF, CDF, survival,
log-CDF, log-survival, and quantile algorithms. StatClass does not perform
generic quadrature over a user-supplied density and therefore does not report a
quadrature error estimate for these built-in calculations.

Verification tests compare numerical Hessians with analytic likelihood
curvature for normal, binomial, Poisson, and logistic models. Separate tests
cover step-size stability, singular and nearly singular information,
delta-method transformation, boundaries, and explicit optimizer failure.

## Scope

StatClass is deliberately a command-line statistics suite. Version 3.16 is the
planned stopping point for the current scope: it does not add a GUI, time-series
analysis, survival analysis, stochastic-process models, an ODE/PDE solver, or
a general interpolation/approximation package. The numerical differentiation
routines support statistical estimation internally; they are not exposed as a
standalone numerical-analysis command. Nor does the program accept arbitrary
infinite series or sequences for convergence testing. The final estimation
layer is limited to the robust, regularized, smooth, monotone, and density
methods documented below; it is not a general machine-learning library.

### Distribution-relationship diagrams

The common-distribution relationship diagram is feasible as a conceptual map,
but not as an executable directed-path specification. StatClass supports all 20
distinct distribution families shown in that diagram: the two normal boxes are
one family, and double exponential is the Laplace distribution. StatClass also
supports logistic and Pareto distributions, which are not shown there.

Every diagram node can therefore be evaluated directly. Several special cases
can be selected by their parameters, and ordinary data-column transformations
such as logarithms, reciprocals, powers, and standardization are available.
However, StatClass does not automatically derive the distribution of sums,
products, ratios, minima, maxima, or transformed random variables; enforce the
independence assumptions on those arrows; or verify dashed asymptotic limits.
Those capabilities would require a symbolic distribution-transformation,
convolution, order-statistic, simulation, and limit-analysis layer beyond the
chosen command-line scope. Because the diagram also contains self-loops, the
set of unrestricted directed walks is infinite. The manual may use such a
diagram to explain relationships, but should not imply that its arrows are
software operations.

## Analyses

- Summary statistics: sample size, missing values, mean, variance, standard
  deviation, quartiles, skewness, and excess kurtosis
- Shapiro-Wilk and D'Agostino-Pearson normality tests
- One-sample t-test
- Population-mean z-test with known population standard deviation
- Welch and Student independent-sample t-tests
- No-file one-sample t, population-mean z, independent two-sample t, and
  paired-difference t inference from sufficient summary statistics
- No-file calculator for 22 common distributions, with PMFs/PDFs,
  probabilities, interval probabilities, critical values, quantiles, support,
  moments, parameterization, and MGF information
- Direct confidence intervals for means (t or z), proportions (Wilson and
  exact), variances and standard deviations, and Poisson event rates
- No-file power, minimum sample-size, and detectable-effect calculations for
  one-sample means, paired means, two independent means, one and two
  proportions, and balanced one-way ANOVA
- Conjugate Bayesian inference for beta-binomial, shape-rate gamma-Poisson,
  normal means with known variance, and unknown normal means and variances
  using a normal-inverse-gamma prior; includes credible intervals, posterior
  prediction, CSV-column input, and prior-versus-posterior plots
- Logarithmic, square-root, reciprocal, power, Box-Cox, and Yeo-Johnson data
  transformations; z-score, min-max, robust median/IQR, and mean-centered
  standardization; z-score, modified-z-score, and Tukey IQR outlier diagnostics
  with case flags and augmented CSV export
- Principal component analysis using standardized correlation matrices or raw
  covariance matrices, with eigenvalues, explained variance, loadings,
  component scores, score export, scree plots, and score plots
- One-way MANOVA with Pillai, Wilks, Hotelling-Lawley, and Roy statistics,
  approximate F tests, group mean vectors, and follow-up univariate ANOVAs
- Linear discriminant analysis with empirical or equal priors, posterior class
  probabilities, canonical discriminant dimensions, confusion matrices,
  leave-one-out validation, prediction export, and model plots
- Welch and classical one-way ANOVA
- Two-way factorial ANOVA and ANCOVA with the factor interaction, Type III
  partial F-tests, diagnostics, and cell summaries
- Estimated marginal means with confidence intervals and Holm- or
  Bonferroni-adjusted pairwise comparisons
- Paired-sample t-test
- Mann-Whitney U test for two independent samples
- Wilcoxon signed-rank test for paired samples
- Kruskal-Wallis test for three or more independent samples
- Dunn pairwise post-hoc comparisons with Holm or Bonferroni adjustment
- Pearson, Spearman, and Kendall correlations
- Simple linear regression with coefficient and ANOVA tables
- Multiple linear regression with numeric and categorical predictors
- Huber robust linear regression for one or more numeric predictors, with
  IRLS convergence, robust weights, sandwich covariance, and matrix diagnostics
- Ridge, lasso, and elastic-net regression for numeric predictors, with
  deterministic K-fold tuning and explicit post-selection inference limits
- Cubic regression splines with coefficient and fitted-mean uncertainty, and
  roughness-penalized cubic smoothing splines with GCV tuning
- One-dimensional LOESS, Gaussian-kernel regression with leave-one-out
  bandwidth selection, and increasing/decreasing isotonic regression
- Gaussian kernel-density estimation with Scott, Silverman, or
  likelihood-cross-validated bandwidths and density-grid validation
- Raw polynomial terms and numeric/categorical interaction terms for multiple
  linear, binary logistic, multinomial logistic, Poisson, and negative-binomial
  regression
- Regression diagnostics: residual tests, VIF, standardized condition number,
  leverage, studentized residuals, Cook's distance, and Durbin-Watson
- Regression model comparison using common cases, AIC, BIC, changes in
  R-squared, and partial F-tests for nested models
- Binary logistic regression with odds ratios, pseudo R-squared measures,
  classification diagnostics, calibration testing, and influence measures
- Baseline-category multinomial logistic regression with relative-risk ratios,
  likelihood tests, and multicategory classification summaries
- Exact confidence intervals for a Poisson mean from an aggregate event count,
  without requiring an input file
- Poisson count regression with incidence-rate ratios, likelihood tests,
  dispersion and goodness-of-fit checks, count prediction metrics, and
  influence diagnostics
- NB2 negative-binomial count regression with estimated dispersion,
  incidence-rate ratios, and a boundary likelihood-ratio comparison with
  Poisson regression
- Fixed log offsets and automatically logged positive exposure variables for
  Poisson and negative-binomial models
- Casewise fitted-value, confidence-band, prediction-interval, and residual
  export for supported models
- One-factor repeated-measures ANOVA with Greenhouse-Geisser correction,
  estimated marginal means, and adjusted paired comparisons
- Gaussian random-intercept mixed-effects models with ML or REML estimation,
  fixed-effect inference, variance components, ICC, BLUPs, and marginal and
  conditional R-squared
- Headless PNG, PDF, and SVG statistical graphics: histograms with KDE,
  normal Q-Q plots, boxplots, grouped boxplots, scatterplots with fitted lines,
  correlation heatmaps, and analysis-aware model plots including PCA scree and
  score plots, MANOVA group profiles, and discriminant score/confusion plots
- Nonparametric bootstrap confidence intervals for one-sample statistics and
  independent-sample differences, with percentile, basic, and BCa methods
- Exact or Monte Carlo permutation tests for independent samples, paired
  sign-flips, and Pearson or Spearman correlation
- Frequency and two-way contingency tables
- Pearson chi-square tests of independence with Cramer's V, expected counts,
  standardized residuals, and assumption warnings
- Pearson chi-square goodness-of-fit tests
- Fisher's exact test for 2-by-2 tables
- Binomial likelihood and log-likelihood analysis with exact formulas,
  maximum-likelihood estimates, and evaluation at requested probabilities
- One-proportion z-tests with exact binomial comparison and likelihood details
- Two-proportion z-tests with risk difference, relative risk, and odds ratio
- No-file one- and two-proportion inference from aggregate success counts
- Discrete-distribution estimation for Bernoulli, binomial, geometric,
  negative-binomial, Poisson, discrete-uniform, hypergeometric,
  beta-binomial, and multinomial models, with raw-column and sufficient-summary
  interfaces, likelihood estimates, standard errors, exact or profile
  intervals where available, and model-specific tests

### Distribution calculator and direct confidence intervals

These commands do not require a data file. Each can be repeated, and different
calculator commands can be combined into one text or JSON report.

Show every supported distribution and its required parameter order with:

```bash
python stats.py --distributions
```

For a point or tail probability, provide the distribution, relation, value,
and parameters:

```bash
python stats.py --distribution-probability binomial eq 8 20 .3
python stats.py --distribution-probability poisson ge 8 5
python stats.py --distribution-probability normal le 1.96 0 1
python stats.py --distribution-probability t gt 2.1 20
python stats.py --distribution-probability chi-square gt 12.6 6
python stats.py --distribution-probability f le 3.2 4 20
```

Relations are `eq`, `le`, `lt`, `ge`, and `gt`. `eq` is available for all
discrete distributions. Inclusive and exclusive tails
are handled exactly for discrete values. For continuous distributions, `le`
and `lt` are equivalent, as are `ge` and `gt`.

The parameters after the value use this order:

| Distribution | Parameters |
|---|---|
| `bernoulli` | success probability `P` |
| `binomial` | number of trials `N`, success probability `P` |
| `discrete-uniform` | largest integer `N`; support is 1 through `N` |
| `geometric` | success probability `P`; counts trials starting at 1 |
| `hypergeometric` | population size `N`, population successes `M`, draws `K` |
| `negative-binomial` | required successes `R`, success probability `P`; counts failures |
| `poisson` | mean `MEAN` |
| `beta` | shapes `ALPHA`, `BETA` |
| `beta-binomial` | trials `N`, shapes `ALPHA`, `BETA` |
| `cauchy` | location `THETA`, scale `SIGMA` |
| `chi-square` | degrees of freedom `DF` |
| `laplace` | location `MU`, scale `SIGMA`; alias `double-exponential` |
| `exponential` | scale `BETA` |
| `f` | numerator `DF1`, denominator `DF2` |
| `gamma` | shape `ALPHA`, scale `BETA` |
| `logistic` | location `MU`, scale `BETA` |
| `lognormal` | mean `MU` and SD `SIGMA` of `log(X)` |
| `normal` | mean `MEAN`, standard deviation `SD` |
| `pareto` | minimum `ALPHA`, shape `BETA` |
| `t` | degrees of freedom `DF` |
| `uniform` | lower endpoint `A`, upper endpoint `B` |
| `weibull` | shape `GAMMA`, textbook parameter `BETA` in `exp(-x^GAMMA/BETA)` |

Evaluate a PMF for a discrete distribution or a PDF for a continuous
distribution with `--distribution-density`:

```bash
python stats.py --distribution-density geometric 4 .25
python stats.py --distribution-density gamma 4 2 3
python stats.py --distribution-density weibull 1.5 2 4
```

No numerical derivative of the CDF is used to obtain a PDF. At a finite lower
support endpoint, the reported endpoint density follows the distribution's
right-hand support convention; at a finite upper endpoint it follows the
left-hand convention. A PDF may be infinite at an endpoint without assigning
positive probability to that single point. The JSON and text output identify
these endpoint cases.

Use `--distribution-info` for the distribution type, parameterization,
support, mean, variance, standard deviation, and MGF formula or existence
information:

```bash
python stats.py --distribution-info geometric .25
python stats.py --distribution-info cauchy 0 1
python stats.py --distribution-info gamma 2 3
python stats.py --distribution-info weibull 2 4
```

Undefined and infinite moments are identified explicitly. For example, the
Cauchy mean and variance are reported as undefined rather than as numerical
values.

For an interval probability, supply its lower and upper bounds before the
distribution parameters:

```bash
python stats.py --distribution-interval normal -1.96 1.96 0 1
python stats.py --distribution-interval binomial 4 8 20 .3
```

Every CDF is right-continuous. The supported continuous distributions have
continuous CDFs, so including or excluding either interval endpoint does not
change their probability. A discrete CDF is a right-continuous step function;
StatClass therefore shifts strict integer bounds and includes both endpoint
masses for a closed discrete interval. Quantiles use the generalized inverse
`inf{x: F(x) >= p}` of the right-continuous CDF.

For a critical value or quantile, supply a lower-tail cumulative probability.
For example, `.975` gives the usual two-sided 95% t critical value:

```bash
python stats.py --distribution-quantile normal .975 0 1
python stats.py --distribution-quantile t .975 20
python stats.py --distribution-quantile chi-square .95 6
python stats.py --distribution-quantile f .95 4 20
```

Direct confidence intervals use `--alpha` (default `.05`, giving 95%
intervals). Any finite value strictly between 0 and 1 is permitted:

```bash
# Mean with a sample SD: t interval
python stats.py --mean-ci 25 72.4 11.8

# Mean with a known population SD: z interval
python stats.py --mean-ci-z 100 102.5 15

# Proportion: Wilson and exact Clopper-Pearson intervals
python stats.py --proportion-ci 30 100

# Population variance and population standard deviation
python stats.py --variance-ci 25 139.24

# Poisson event rate: EVENTS divided by EXPOSURE
python stats.py --poisson-rate-ci 8 1000
```

A combined calculation can be saved to JSON just like a file-based analysis:

```bash
python stats.py \
  --distribution-probability poisson ge 8 5 \
  --distribution-quantile t .975 20 \
  --mean-ci 25 72.4 11.8 \
  --proportion-ci 30 100 \
  --json calculator_results.json
```

### Discrete-distribution estimation

Estimation is separate from the distribution calculator: calculator commands
evaluate a model whose parameters are already known, while these commands
estimate unknown parameters from observations. Every interval and test uses
the exact `--alpha` supplied by the user; it is never clipped or rounded before
calculation.

Wald inference is reported only where a regular continuous-parameter
approximation exists. Even there, it is an asymptotic approximation rather
than a universal default. StatClass reports a better-behaved comparison when
one is available and warns when the Wald standard error degenerates or its
interval leaves the parameter space.

| Model | Main estimate | Interval and test treatment |
|---|---|---|
| Bernoulli/binomial | common success probability | Wald plus Wilson and exact Clopper-Pearson intervals; optional Wald, score, exact binomial, and likelihood-ratio tests |
| Geometric | success probability; observations count trials through first success | Wald plus exact stopped-trial interval; optional Wald, score, exact negative-binomial-tail, and likelihood-ratio tests |
| Negative-binomial, known `R` | success probability; observations count failures before the `R`th success | same stopped-trial inference as geometric |
| Negative-binomial, unknown `R` and `p` | mean, size `R`, probability, and dispersion `1/R` | numerical MLE, Wald interval for the mean, profile-likelihood interval for `R`, and explicit Poisson-boundary handling |
| Poisson | event mean | Wald plus exact Garwood intervals; optional Wald, score, exact Poisson-tail, and likelihood-ratio tests; raw samples also receive an index-of-dispersion check |
| Discrete uniform on `1,...,N` | integer endpoint `N` | exact one-sided confidence set and optional exact likelihood-ratio endpoint test; ordinary Wald inference is invalid because `N` changes the support |
| Hypergeometric | integer number of population successes | discrete likelihood MLE and likelihood-ratio confidence set/test; exact inversion for a manageable single count, otherwise an identified asymptotic LR fallback; no Wald inference |
| Beta-binomial | positive shapes, mean probability, concentration, and intraclass correlation | numerical MLE and transformed observed-information Wald intervals; boundary comparison with binomial is reported without an ordinary chi-square p-value |
| Multinomial | category probabilities and covariance matrix | marginal Wald and Wilson intervals plus Bonferroni simultaneous Wilson intervals |

The raw-column forms are:

```bash
# VALUE after a Bernoulli column is the category counted as success
python stats.py survey.csv --estimate-bernoulli outcome Yes .50

# TRIALS may be a fixed positive integer or another column
python stats.py grouped.csv --estimate-binomial successes trials .40

python stats.py waits.csv --estimate-geometric trials_to_success .25
python stats.py counts.csv --estimate-poisson events 5

# Known R, then joint estimation of R and p
python stats.py counts.csv --estimate-negative-binomial failures 3 .40
python stats.py counts.csv --estimate-negative-binomial failures auto

python stats.py endpoints.csv --estimate-discrete-uniform observed 20
python stats.py samples.csv --estimate-hypergeometric observed 100 20 30
python stats.py grouped.csv --estimate-beta-binomial successes trials
python stats.py survey.csv --estimate-multinomial response
```

The final optional value in most commands is the null value. Omitting it
reports estimation and intervals without hypothesis tests. `auto` joint
negative-binomial estimation does not accept a null value because its
two-parameter inference is handled by profiling and boundary diagnostics.

When sufficient statistics contain all information needed by the likelihood,
no input file is required:

```bash
python stats.py --estimate-binomial-summary 37 100 .40
python stats.py --estimate-geometric-summary 20 73 .25
python stats.py --estimate-poisson-summary 25 119 5
python stats.py --estimate-negative-binomial-summary 20 93 2 .30
python stats.py --estimate-discrete-uniform-summary 30 17 20
python stats.py --estimate-hypergeometric-count 7 100 20 30
python stats.py --estimate-multinomial-counts red=12 blue=9 green=4
```

Summary Poisson input cannot supply a sample-variance dispersion check;
beta-binomial and joint negative-binomial estimation require the raw grouped or
sample observations. All results can be combined and written to JSON.

### Power and sample-size analysis

Power calculations also require no data file. Every supported design provides
three operations: calculate power for a proposed design, find the minimum
integer sample size for a target power, or find the smallest detectable effect
for fixed sample sizes. The default is a two-sided test with `--alpha .05`.

For one-sample means, Cohen's $d$ is the population mean difference divided by
the standard deviation:

```bash
python stats.py --power-one-mean 50 .5
python stats.py --sample-size-one-mean .5 .80
python stats.py --detectable-effect-one-mean 50 .80
```

Paired-mean calculations use the number of complete pairs and Cohen's $d_z$,
the mean paired difference divided by the standard deviation of the paired
differences:

```bash
python stats.py --power-paired-mean 30 .5
python stats.py --sample-size-paired-mean .5 .80
python stats.py --detectable-effect-paired-mean 30 .80
```

For two independent means, Cohen's $d$ uses the common within-group standard
deviation. These calculations use the equal-variance noncentral t model:

```bash
python stats.py --power-two-means 40 40 .6
python stats.py --sample-size-two-means .6 .80
python stats.py --detectable-effect-two-means 40 40 .80
```

The sample-size command returns both group sizes. Unequal allocation is set by
the ratio $N_2/N_1$:

```bash
python stats.py --sample-size-two-means .6 .80 --allocation-ratio 1.5
```

One-proportion calculations take an actual population probability and the null
probability. The detectable-effect command holds the null probability fixed:

```bash
python stats.py --power-one-proportion 100 .30 .20
python stats.py --sample-size-one-proportion .30 .20 .80
python stats.py --detectable-effect-one-proportion 100 .20 .80
```

For two independent proportions, the difference is $p_1-p_2$. The
detectable-effect command holds $p_2$ at `BASELINE_P` and solves for $p_1$:

```bash
python stats.py --power-two-proportions 100 100 .30 .45
python stats.py --sample-size-two-proportions .30 .45 .80
python stats.py --detectable-effect-two-proportions 100 100 .30 .80
python stats.py --sample-size-two-proportions .30 .45 .80 --allocation-ratio 1.5
```

Proportion power uses the normal approximation to the corresponding z-test.
The report identifies this approximation explicitly.

Balanced one-way ANOVA uses the number of groups, equal sample size per group,
and Cohen's $f$:

```bash
python stats.py --power-anova 4 25 .25
python stats.py --sample-size-anova 4 .25 .80
python stats.py --detectable-effect-anova 4 25 .80
```

Mean and proportion calculations accept `--alternative two-sided`, `greater`,
or `less`. Supply a negative Cohen's $d$ for a specified `less` mean effect.
For proportion calculations, the actual probability or $p_1$ should be below
the comparison probability for `less`. Detectable-effect calculations search
downward automatically for `less` and upward for `greater` or `two-sided`.

All power commands can be repeated or combined. For example:

```bash
python stats.py \
  --power-one-mean 50 .5 \
  --sample-size-two-means .6 .80 \
  --sample-size-one-proportion .30 .20 .80 \
  --power-anova 4 25 .25 \
  --json power_results.json
```

### Bayesian conjugate models

StatClass implements four conjugate Bayesian models. They use exact analytic
posterior and posterior-predictive distributions, so simulation is not needed.
The `--alpha` value determines the equal-tail credible and predictive level;
the default `--alpha .05` produces 95% intervals.

Their posterior updates use closed-form conjugate algebra. They do not use
derivatives, optimization, iterative root finding, numerical integration, an
ODE solver, or MCMC. Credible and predictive intervals use specialized inverse
CDF algorithms. Text and JSON output validate posterior-parameter finiteness
and support, interval finiteness and ordering, continuous or discrete support,
and predictive probabilities. Nonfinite, reversed, or out-of-support results
are errors rather than completed analyses.

For binomial data, give successes, trials, and the two parameters of the beta
prior. This example uses the uniform `Beta(1,1)` prior:

```bash
python stats.py --bayes-binomial 30 100 1 1
```

The posterior is `Beta(31,71)`. The report includes its mean, variance, mode,
credible interval, the probability of success on the next trial, and a beta-
binomial predictive distribution. Request predictions for more future trials
with:

```bash
python stats.py --bayes-binomial 30 100 1 1 --bayes-future-trials 20
```

The gamma-Poisson model uses the shape-rate parameterization. Supply total
events, total exposure, prior shape, and prior rate:

```bash
python stats.py --bayes-poisson 8 1 1 1
python stats.py --bayes-poisson 8 1000 1 1 --bayes-future-exposure 500
```

Its posterior is gamma and its future event-count distribution is negative
binomial. The output includes the posterior event-rate interval, predictive
mean, probability of zero future events, and predictive interval.

For a normal population mean with known population standard deviation, supply
`N MEAN SIGMA PRIOR_MEAN PRIOR_SD`:

```bash
python stats.py --bayes-normal-known 25 72.4 15 70 10
```

When both the normal mean and variance are unknown, use the normal-inverse-
gamma model. Supply `N MEAN SD PRIOR_MEAN PRIOR_KAPPA PRIOR_ALPHA PRIOR_BETA`:

```bash
python stats.py --bayes-normal-unknown 25 72.4 11.8 70 1 2 100
```

This model reports the four posterior hyperparameters, marginal Student-t
inference for the mean, inverse-gamma inference for the variance, and the
Student-t posterior-predictive distribution for a future observation.

The same models work with data columns:

```bash
python stats.py diabetes.csv \
  --bayes-binomial-column diabetes Yes 1 1

python stats.py "$DATA_DIR/count_sample.csv" \
  --bayes-poisson-column visits 1 1 \
  --bayes-poisson-exposure-column exposure

python stats.py data.csv \
  --bayes-normal-known-column pressure 15 100 20

python stats.py data.csv \
  --bayes-normal-unknown-column pressure 100 1 2 100
```

For a Poisson count column without an exposure column, each usable row counts
as one exposure unit. Add `--bayes-plots` to save prior-versus-posterior density
plots. The normal-inverse-gamma model creates separate mean and variance plots:

```bash
python stats.py --bayes-binomial 30 100 1 1 \
  --bayes-plots --plot-dir plots
```

All four no-file Bayesian commands may be repeated or combined in one report.

### Transformations, standardization, and outlier diagnostics

Transform a numeric column by supplying its name, a method, and a parameter
when the method requires one:

```bash
python stats.py data.csv --transform pressure log
python stats.py data.csv --transform pressure log10
python stats.py data.csv --transform pressure sqrt
python stats.py data.csv --transform pressure reciprocal
python stats.py data.csv --transform pressure power 2
```

The natural-log, base-10-log, and Box-Cox methods require strictly positive
values. Square roots require nonnegative values, and reciprocals do not permit
zero. A power transformation requires its exponent.

Box-Cox and Yeo-Johnson estimate their power parameter by maximum likelihood
when it is omitted:

```bash
python stats.py data.csv --transform pressure box-cox
python stats.py data.csv --transform pressure yeo-johnson
```

Supply the parameter explicitly when desired:

```bash
python stats.py data.csv --transform pressure box-cox 0
python stats.py data.csv --transform pressure yeo-johnson 1
```

Yeo-Johnson permits zero and negative data. The report gives the estimated or
specified parameter, formula, usable observations, missing values, and summary
statistics before and after transformation.

Four standardization methods are available:

```bash
python stats.py data.csv --standardize pressure z-score
python stats.py data.csv --standardize pressure min-max
python stats.py data.csv --standardize pressure robust
python stats.py data.csv --standardize pressure center
```

The z-score method uses the sample mean and sample standard deviation. Min-max
scaling maps the observed minimum to 0 and maximum to 1. Robust scaling uses
the median and interquartile range. Centering subtracts the sample mean without
rescaling.

Univariate outliers can be diagnosed using ordinary z-scores, modified
z-scores based on the median absolute deviation, or Tukey IQR fences:

```bash
python stats.py data.csv --outliers pressure z-score
python stats.py data.csv --outliers pressure modified-z-score
python stats.py data.csv --outliers pressure iqr
python stats.py data.csv --outliers pressure all
```

The default absolute z-score threshold is 3, the modified-z threshold is 3.5,
and the Tukey fence multiplier is 1.5. They may be changed independently:

```bash
python stats.py data.csv \
  --outliers pressure all \
  --outlier-z-threshold 2.5 \
  --outlier-modified-z-threshold 3 \
  --outlier-iqr-multiplier 2
```

The report lists the center, scale, lower and upper fences, number and
proportion of outliers, row numbers, original values, and diagnostic scores.
Outliers are reported and flagged; the observations are never silently removed
or altered.

Use `--save-transformed` to preserve the original table and append every
requested transformed value, standardized value, and Boolean outlier flag:

```bash
python stats.py data.csv \
  --transform pressure log \
  --standardize temperature z-score \
  --outliers pressure all \
  --save-transformed processed_data.csv \
  --json processing_report.json
```

Each option may be repeated for additional columns. Derived column names are
made unique automatically, and missing input values remain missing in the
corresponding derived column.

### Multivariate analysis

The included `multivariate_sample.csv` contains three groups and four numeric
variables suitable for all three multivariate procedures.

Principal component analysis (PCA) reduces a group of correlated numeric
variables to a smaller set of components. By default, StatClass standardizes
each variable and analyzes the correlation matrix:

```bash
python stats.py "$DATA_DIR/multivariate_sample.csv" --pca x1 x2 x3 x4
```

Use the covariance matrix when the variables share meaningful units and their
original variances should affect the solution:

```bash
python stats.py "$DATA_DIR/multivariate_sample.csv" \
  --pca x1 x2 x3 x4 --pca-no-standardize
```

Retain a chosen number of components and export complete-case scores with:

```bash
python stats.py "$DATA_DIR/multivariate_sample.csv" \
  --pca x1 x2 x3 x4 \
  --pca-components 2 \
  --save-pca-scores pca_scores.csv
```

The report gives eigenvalues, proportions and cumulative proportions of
explained variance, component coefficients, and variable-component
correlations. `--model-plots` adds a scree plot and a PC1-versus-PC2 score plot
when two components are retained.

One-way MANOVA tests whether the population mean vectors are equal across the
levels of one grouping variable. List the group first, followed by at least two
numeric responses:

```bash
python stats.py "$DATA_DIR/multivariate_sample.csv" --manova group x1 x2 x3 x4
```

StatClass reports Pillai's trace, Wilks' lambda, the Hotelling-Lawley trace,
Roy's largest root, their approximate F tests, group summaries, hypothesis and
error SSCP matrices in JSON, and follow-up univariate ANOVAs. The usual
MANOVA assumptions—independence, within-group multivariate normality, and
comparable covariance matrices—remain substantive checks rather than facts
created by the command.

Linear discriminant analysis (LDA) predicts a categorical class from two or
more numeric features:

```bash
python stats.py "$DATA_DIR/multivariate_sample.csv" \
  --discriminant group x1 x2 x3 x4
```

Empirical class proportions are the default priors. Equal priors are available
when each class should receive the same prior weight:

```bash
python stats.py "$DATA_DIR/multivariate_sample.csv" \
  --discriminant group x1 x2 x3 x4 \
  --discriminant-priors equal
```

The output includes class means, classification functions, canonical
discriminant dimensions, apparent classification performance, and
leave-one-out cross-validation. Export predictions, posterior probabilities,
and canonical scores with:

```bash
python stats.py "$DATA_DIR/multivariate_sample.csv" \
  --discriminant group x1 x2 x3 x4 \
  --save-discriminant discriminant_predictions.csv \
  --model-plots --plot-dir plots
```

`--lda` is an alias for `--discriminant`. Missing or nonfinite values are
removed rowwise for each multivariate analysis and the report states how many
rows were removed.

### No-file summary-data inference

A data file is unnecessary when the relevant sample counts or sufficient
statistics are already known. StatClass accepts these values directly.

For a one-sample t-test, supply the sample size, sample mean, and sample
standard deviation, in that order:

```bash
python stats.py --one-sample-summary 25 72.4 11.8 --mu 70
```

For a population-mean z-test, replace the sample standard deviation with the
known population standard deviation:

```bash
python stats.py --z-summary 100 102.5 15 --mu 100
```

For two independent samples, provide `N`, `MEAN`, and `SD` for sample 1,
followed by the same three values for sample 2. Welch's test is the default;
add `--equal-variance` for Student's pooled-variance test:

```bash
python stats.py --two-sample-summary 50 10.2 2.1 45 9.4 2.3
python stats.py --two-sample-summary 50 10.2 2.1 45 9.4 2.3 --equal-variance
```

A summarized paired test uses the sample size, mean difference, and sample
standard deviation of the pairwise differences:

```bash
python stats.py --paired-summary 30 2.4 5.1 --mu 0
```

One- and two-proportion tests accept success counts and total trial counts:

```bash
python stats.py --one-proportion-counts 30 100 --p0 .25
python stats.py --two-proportion-counts 30 100 45 120
```

These commands report test statistics, degrees of freedom where applicable,
p-values, two-sided confidence intervals, decisions, and effect sizes. The
options `--alpha`, `--alternative`, `--mu`, `--p0`, `--output`, and `--json`
work without a file whenever they apply. Each analysis option may be repeated.

### Paired samples, correlation, and regression

```bash
python stats.py data.csv --paired Before After
```

Rows with either paired value missing are removed as pairs. Output includes the
number of complete pairs, paired means, mean difference, t-statistic, degrees
of freedom, p-value, confidence interval, and Cohen's dz.

Run Pearson, Spearman, and Kendall correlations together:

```bash
python stats.py data.csv --correlation Height Weight
```

Run only one method:

```bash
python stats.py data.csv --correlation Height Weight --correlation-method spearman
```

Available methods are `pearson`, `spearman`, `kendall`, and `all`.

The dependent variable is listed first:

```bash
python stats.py data.csv --regression Outcome Predictor
```

Regression output includes the intercept and slope estimates, standard errors,
t-statistics, p-values, confidence intervals, R-squared, adjusted R-squared,
residual standard error, overall F-test, ANOVA table, and residual summary.

### Multiple linear regression and diagnostics

The dependent variable is listed first, followed by all predictors:

```bash
python stats.py "$DATA_DIR/regression_sample.csv" \
  --multiple-regression Outcome X1 X2 Group
```

Text predictors such as `Group` are detected automatically and reference-coded
with the first observed category as the reference. To treat a numeric code as
categorical, identify it explicitly:

```bash
python stats.py data.csv \
  --multiple-regression Outcome Age TreatmentCode \
  --categorical-predictors TreatmentCode
```

The report contains coefficient estimates, standard errors, t-tests,
confidence intervals, standardized coefficients, R-squared, adjusted
R-squared, the overall F-test, AIC, BIC, and an ANOVA table. Diagnostics include
residual normality tests, the Breusch-Pagan test, Durbin-Watson, VIF and
tolerance, a standardized-design condition number, leverage, internally
studentized residuals, and Cook's distance. Missing values are removed by
complete-case analysis for each model.

Request two or more models in reduced-to-full order and compare adjacent models:

```bash
python stats.py "$DATA_DIR/regression_sample.csv" \
  --multiple-regression Outcome X1 \
  --multiple-regression Outcome X1 X2 Group \
  --compare-models
```

Models are refitted on their common complete cases for a fair comparison. AIC,
BIC, and changes in R-squared are reported for any models with the same outcome.
When the first predictor set is nested within the second, StatClass also reports
a partial F-test for the added terms.

### Robust, regularized, smooth, and monotone estimation

These procedures are the final bounded extension of the regression scope.
They estimate relationships in noisy data; none is an exact interpolating
polynomial. They currently accept numeric predictors only. Use ordinary
`--multiple-regression` when categorical reference coding, polynomial terms,
or interactions are required.

Huber regression reduces the influence of large residuals while retaining a
linear coefficient model:

```bash
python stats.py "$DATA_DIR/robust_regularized_sample.csv" \
  --robust-regression Outcome X1 X2 X3 NoisePredictor
```

The default Huber tuning constant is `1.345`; change it with
`--robust-tuning`. Fitting uses iteratively reweighted least squares (IRLS).
The report includes convergence, iteration count, residual scale, number and
minimum weight of downweighted cases, standardized-design rank and condition
number, and a robust sandwich covariance when it is estimable. Coefficient
z-tests and confidence intervals use that sandwich covariance. If the design
or the robust covariance is singular, the fit or the affected inference is
withheld explicitly rather than computed with a pseudoinverse.

Ridge, lasso, and elastic-net regression put the method before the dependent
and predictor columns:

```bash
python stats.py "$DATA_DIR/robust_regularized_sample.csv" \
  --regularized-regression ridge Outcome X1 X2 X3 NoisePredictor
python stats.py "$DATA_DIR/robust_regularized_sample.csv" \
  --regularized-regression lasso Outcome X1 X2 X3 NoisePredictor
python stats.py "$DATA_DIR/robust_regularized_sample.csv" \
  --regularized-regression elastic-net Outcome X1 X2 X3 NoisePredictor \
  --elastic-net-l1-ratio .5
```

Predictors are standardized internally and the intercept is not penalized.
By default, a deterministic K-fold search selects the positive penalty by
mean squared validation error; `--cv-folds` and `--random-seed` control the
folds. Supply `--regularization-strength POSITIVE_NUMBER` to bypass the search
(`--regularization-alpha` is retained as an alias, but is not the significance
level `--alpha`).
The JSON report retains every candidate and validation score. Ridge uses a
penalized linear solve; lasso and elastic net use coordinate descent with a
visible convergence status. A rank-deficient unpenalized design is reported,
but a positive penalty can still produce a stable fit. Ordinary OLS standard
errors and p-values are not reported for penalized coefficients because they
would ignore shrinkage and data-driven penalty selection.

A cubic regression spline estimates a finite-dimensional piecewise-polynomial
mean curve and provides ordinary coefficient inference plus pointwise
fitted-mean confidence intervals:

```bash
python stats.py "$DATA_DIR/smooth_estimation_sample.csv" \
  --regression-spline NonlinearOutcome X --spline-knots 5
```

Interior knots are predictor quantiles. The truncated-power basis is formed
from a standardized predictor, and the report gives the original-scale knot
locations, basis rank, condition number, coefficients, and uncertainty.
`--spline-knots` requests from 1 through 50 interior knots; an analysis is not
run if the sample or number of distinct values cannot support that basis.

A smoothing spline uses cubic B-splines with an integrated squared-second-
derivative roughness penalty:

```bash
python stats.py "$DATA_DIR/smooth_estimation_sample.csv" \
  --smoothing-spline NonlinearOutcome X --spline-knots 5
```

The default penalty is selected by generalized cross-validation (GCV), with
the complete search retained in JSON. `--smoothing-lambda` supplies a fixed
nonnegative penalty instead. Output includes the selected strength, GCV,
effective degrees of freedom, penalized-system condition number, and
approximate pointwise confidence intervals for the fitted mean. A selected
edge of the GCV grid is reported as a warning.

LOESS fits local polynomials with nearest-neighborhood tricube weights and
optional robust bisquare reweighting:

```bash
python stats.py "$DATA_DIR/smooth_estimation_sample.csv" \
  --loess NonlinearOutcome X --loess-span .65 --loess-degree 1
```

The span is greater than 0 and at most 1; the local degree is 1 or 2. The
default two robust iterations can be changed with
`--loess-robust-iterations`. A locally singular requested polynomial is
reduced in degree and counted in the warnings. LOESS reports fitted values,
residual metrics, robust weights, and an approximate effective degrees of
freedom, but not a single global coefficient table or unsupported p-values.

Gaussian Nadaraya-Watson regression estimates a local weighted mean:

```bash
python stats.py "$DATA_DIR/smooth_estimation_sample.csv" \
  --kernel-regression NonlinearOutcome X
```

Its bandwidth is selected by leave-one-out squared-error cross-validation.
Use `--kernel-bandwidth POSITIVE_NUMBER` to supply one. The candidate grid,
selected error, effective degrees of freedom, fitted values, and residuals are
reported. The calculation is intentionally limited to 5,000 complete cases
because it is quadratic in sample size.

Isotonic regression estimates a monotone step function with the
pool-adjacent-violators algorithm:

```bash
python stats.py "$DATA_DIR/smooth_estimation_sample.csv" \
  --isotonic-regression MonotoneOutcome X --isotonic-direction increasing
```

Directions are `increasing`, `decreasing`, and `auto`. `auto` fits both and
chooses the smaller training residual sum of squares; this is a descriptive
choice, not an external validation result. Duplicate predictor values are
combined with frequency weights. Output includes the direction, number of
distinct predictor values, monotone block count, fitted values, residuals,
and algorithm status.

Gaussian kernel-density estimation is available separately from regression:

```bash
python stats.py "$DATA_DIR/smooth_estimation_sample.csv" \
  --kernel-density DensityValue --kde-bandwidth cv --kde-grid-size 300
```

Bandwidth choices are `cv`, `scott`, and `silverman`. The default `cv` uses
leave-one-out maximum-likelihood selection and is limited to 3,000 finite
values; the rule-based methods handle larger samples. The text report gives
the selected bandwidth and checks that the reported grid is finite,
nonnegative, and integrates approximately to one. JSON retains the full
density grid and bandwidth search.

All of these options may be combined or repeated. Complete cases are selected
per requested model. `completed_with_warnings` is distinct from `completed`,
and `not_run` includes a reason plus a failed numerical status. Prediction and
residual rows work with `--save-predictions`, `--save-residuals`, and
`--model-plots`. Confidence statements use the exact value of `--alpha`; no
calculation input is rounded or clipped.

### Polynomial and interaction terms

Add curvature with raw polynomial powers from degree 2 through 5:

```bash
python stats.py "$DATA_DIR/regression_sample.csv" \
  --multiple-regression Outcome X1 Group \
  --polynomial X1 2
```

Add an interaction while retaining both main effects:

```bash
python stats.py "$DATA_DIR/regression_sample.csv" \
  --multiple-regression Outcome X1 Group \
  --interaction X1 Group
```

The two named columns must already be predictors in the same model. Numeric by
numeric interactions add one product. Numeric by categorical interactions add
one product per nonreference dummy, and categorical by categorical
interactions cross their nonreference dummies. Polynomial powers are raw,
rather than orthogonal, powers. Large VIFs can therefore be expected when a
predictor is far from zero; centering the predictor in the data first can make
the main-effect interpretation and collinearity diagnostics more useful.

Both options may be repeated. They apply to every compatible requested
multiple linear, binary logistic, multinomial logistic, Poisson,
negative-binomial, or random-intercept mixed model that contains the named
predictors.

### Binary logistic regression

The outcome may contain text or numeric categories. Specify the category to be
modeled as the event:

```bash
python stats.py "$DATA_DIR/logistic_sample.csv" \
  --logistic Purchased Income Age Segment \
  --event Yes
```

Text predictors are reference-coded automatically. The output includes log-odds
coefficients, standard errors, Wald z-tests, confidence intervals, odds ratios,
the likelihood-ratio test, deviance, AIC, BIC, McFadden, Cox-Snell, and
Nagelkerke pseudo R-squared. Classification output includes the confusion
matrix, accuracy, sensitivity, specificity, precision, F1, ROC AUC, and the
Brier score. Calibration and influence output includes the Hosmer-Lemeshow
test, VIF, condition number, leverage, standardized Pearson residuals, and
Cook's distance.

Change the classification cutoff when needed:

```bash
python stats.py "$DATA_DIR/logistic_sample.csv" \
  --logistic Purchased Income Age Segment \
  --event Yes \
  --classification-threshold .40
```

### Multinomial logistic regression

Use multinomial logistic regression when the outcome has three or more
unordered categories:

```bash
python stats.py "$DATA_DIR/logistic_sample.csv" \
  --multinomial-logistic Risk Income Age Segment \
  --outcome-reference Low
```

If `--outcome-reference` is omitted, the first observed outcome category is the
baseline. StatClass reports a separate coefficient equation for each other
category versus the baseline, including relative-risk ratios and confidence
intervals. It also reports the likelihood-ratio test, deviance, AIC, BIC,
McFadden pseudo R-squared, a multicategory confusion matrix, accuracy, macro
precision, macro recall, macro F1, category-specific metrics, and the
multiclass Brier score.

### Poisson mean confidence intervals

An observed aggregate event count does not require a data file. For example,
to estimate the Poisson parameter after observing 8 events, use:

```bash
python stats.py --poisson-count 8
```

The command reports the model `X ~ Poisson(M)`, the likelihood formula, the
MLE `M = 8`, and the exact 95% confidence interval. Change the confidence level
through the significance level; for example, an exact 90% interval uses:

```bash
python stats.py --poisson-count 8 --alpha .10
```

`--poisson-mean-count` is an alias for `--poisson-count`. Zero events are
supported, and repeated `--poisson-count` options analyze several aggregate
counts in one command.

### Poisson count regression

Use Poisson regression when the outcome consists of nonnegative integer
counts. The outcome is listed first:

```bash
python stats.py "$DATA_DIR/count_sample.csv" \
  --poisson visits exposure age group \
  --categorical-predictors group
```

Polynomial and interaction terms work the same way as in the other regression
models:

```bash
python stats.py "$DATA_DIR/count_sample.csv" \
  --poisson visits exposure age group \
  --categorical-predictors group \
  --polynomial exposure 2 \
  --interaction exposure group
```

The report includes log-count coefficients, Wald z-tests, confidence
intervals, incidence-rate ratios (IRRs), the likelihood-ratio test, AIC, BIC,
McFadden pseudo R-squared, deviance and Pearson goodness-of-fit approximations,
dispersion statistics, MAE, RMSE, observed and predicted zero rates, VIF,
condition number, leverage, standardized Pearson residuals, and Cook's
distance. A Pearson dispersion above 1.5 produces an overdispersion warning;
in that case Poisson standard errors may be too small.

When observations have different amounts of time, population, distance, or
other opportunity for events, supply a positive exposure variable:

```bash
python stats.py "$DATA_DIR/overdispersed_count_sample.csv" \
  --poisson claims age risk_score group \
  --categorical-predictors group \
  --exposure insured_years
```

StatClass uses the natural logarithm of `insured_years` as a fixed offset with
coefficient 1. If the data already contain the logarithm, use
`--offset log_insured_years` instead. `--offset` and `--exposure` are mutually
exclusive and apply to every requested count model in the command.

### Negative-binomial count regression

Use NB2 negative-binomial regression when counts are overdispersed relative to
Poisson regression:

```bash
python stats.py "$DATA_DIR/overdispersed_count_sample.csv" \
  --negative-binomial claims age risk_score group \
  --categorical-predictors group \
  --exposure insured_years
```

The NB2 variance function is `Var(Y|X) = mu + dispersion * mu^2`. StatClass
estimates the positive dispersion parameter jointly with the regression
coefficients. Output includes IRRs, Wald inference, a dispersion confidence
interval, likelihood-ratio testing, AIC, BIC, residual and influence
diagnostics, goodness-of-fit approximations, and count-prediction metrics. It
also fits the corresponding Poisson model and reports a boundary
likelihood-ratio comparison. Because the Poisson null value sets dispersion to
zero at the edge of the parameter space, that comparison uses the appropriate
50:50 point-mass/chi-square mixture approximation.

### Two-way ANOVA, ANCOVA, and estimated marginal means

The outcome is numeric and the next two columns are categorical factors. The
model always includes both main effects and their interaction:

```bash
python stats.py "$DATA_DIR/factorial_sample.csv" \
  --two-way-anova score method setting
```

Add one or more numeric covariates after the two factors to fit ANCOVA:

```bash
python stats.py "$DATA_DIR/factorial_sample.csv" \
  --ancova score method setting baseline
```

StatClass uses sum-to-zero effect coding and reports Type III partial F-tests,
an overall F-test, R-squared, adjusted R-squared, cell descriptives, residual
normality, Brown-Forsythe and Breusch-Pagan checks, and a homogeneity-of-slopes
check for ANCOVA. Estimated marginal means hold covariates at their sample
means and weight the levels of the other factor equally. Pairwise comparisons
use Holm adjustment by default; select Bonferroni with
`--emmeans-adjust bonferroni`.

### Predictions, confidence bands, and residual export

Write casewise model results without changing the human-readable report:

```bash
python stats.py "$DATA_DIR/regression_sample.csv" \
  --multiple-regression Outcome X1 X2 Group \
  --save-predictions fitted.csv \
  --save-residuals residuals.csv
```

The prediction file identifies the model and original row, then includes the
observed value, fitted value, residual, and intervals available for that model.
Linear models include confidence limits for the conditional mean and prediction
intervals for a new observation. Binary logistic and count models include
confidence limits for the fitted probability or mean count. Multinomial models
include every category probability and the predicted category. The compact
residual file keeps identifiers, observed and predicted values, and applicable
raw or Pearson residuals. These options also work with factorial, repeated-
measures, and mixed-effects analyses.

### Bootstrap confidence intervals

Bootstrap a sample mean, median, or standard deviation by listing the column
and statistic:

```bash
python stats.py data.csv \
  --bootstrap pressure mean \
  --bootstrap pressure median
```

Bootstrap a difference between two independent samples stored in separate
columns:

```bash
python stats.py "$DATA_DIR/nonparametric_sample.csv" \
  --bootstrap-difference Control Treatment_1 mean
```

BCa—the bias-corrected and accelerated interval—is the default. Percentile and
basic intervals are also available:

```bash
python stats.py data.csv \
  --bootstrap pressure standard-deviation \
  --bootstrap-method percentile \
  --resamples 20000 \
  --seed 2026
```

Output includes the observed estimate, bootstrap mean, standard error,
estimated bias, confidence interval, resample count, and random seed. BCa
output also includes the estimated bias-correction and acceleration constants.
Missing values are removed independently from each sample. The implementation
uses bounded-memory batches, allowing it to process substantially longer
columns without allocating the complete resampling matrix at once.

### Permutation tests

Compare two independent columns using a difference in means or medians:

```bash
python stats.py "$DATA_DIR/nonparametric_sample.csv" \
  --permutation-two-sample Control Treatment_1 mean
```

Use a paired sign-flip test for matched columns:

```bash
python stats.py "$DATA_DIR/nonparametric_sample.csv" \
  --permutation-paired Before After median
```

Test Pearson or Spearman association by permuting one member of each pair:

```bash
python stats.py data.csv \
  --permutation-correlation Height Weight spearman
```

StatClass calculates the complete exact permutation distribution whenever its
size does not exceed `--resamples`. Otherwise it performs the requested number
of Monte Carlo permutations and reports a Monte Carlo standard error. Monte
Carlo p-values use `(extreme + 1)/(permutations + 1)`, preventing a reported
p-value of zero. The general `--alternative two-sided`, `less`, or `greater`
option applies to every permutation test.

The default is 10,000 resamples with seed 12345. Use `--resamples` from 100
through 1,000,000 and `--seed` (or `--random-seed`) for another reproducible
sequence. When several resampling analyses appear in one command, StatClass
increments the seed so that each analysis receives a distinct reproducible
stream.

### Statistical plots

Plots are saved to files and never require a graphical desktop. This makes the
same commands usable in Anaconda, a terminal-only Linux session, Docker, or a
cloud job. Repeat `--plot` to request several graphics:

```bash
python stats.py "$DATA_DIR/regression_sample.csv" \
  --plot histogram Outcome \
  --plot qq Outcome \
  --plot boxplot Outcome \
  --plot grouped-boxplot Outcome Group \
  --plot scatter X1 Outcome \
  --plot correlation Outcome X1 X2 \
  --plot-dir plots
```

The supported explicit forms are:

```text
--plot histogram NUMERIC_COLUMN
--plot qq NUMERIC_COLUMN
--plot boxplot NUMERIC_COLUMN
--plot grouped-boxplot NUMERIC_OUTCOME GROUP
--plot scatter NUMERIC_X NUMERIC_Y
--plot correlation NUMERIC_COLUMN NUMERIC_COLUMN [NUMERIC_COLUMN ...]
```

Histograms include a kernel-density curve when the data permit one.
Scatterplots include an ordinary least-squares fitted line, and correlation
heatmaps are drawn directly with Matplotlib without requiring Seaborn.

Use `--model-plots` with fitted analyses to create appropriate diagnostics:

```bash
python stats.py "$DATA_DIR/regression_sample.csv" \
  --multiple-regression Outcome X1 X2 Group \
  --model-plots \
  --plot-dir plots
```

Linear, count, factorial, and mixed models receive observed-versus-predicted,
residual-versus-fitted, and residual Q-Q diagnostics when applicable. Binary
logistic models receive ROC and calibration plots; multinomial logistic models
receive a confusion-matrix plot. Factorial models receive interaction plots,
repeated-measures models receive subject-profile and marginal-mean plots, and
mixed models receive group-trajectory plots.

PNG is the default. Select another format or resolution with:

```bash
python stats.py data.csv \
  --plot histogram pressure \
  --plot-dir plots \
  --plot-format svg \
  --plot-dpi 200
```

`--plot-format` accepts `png`, `pdf`, or `svg`; `--plot-dpi` accepts 50 through
600. Existing files are preserved by adding a numerical suffix to a new plot
when necessary. The text report and JSON output list every generated filename.

### Repeated-measures ANOVA

Repeated-measures data use long format: one row per subject and within-factor
level. This example has the columns `subject`, `time`, and `outcome`:

```bash
python stats.py "$DATA_DIR/longitudinal_sample.csv" \
  --repeated-measures outcome subject time
```

The current procedure supports one within-subject factor. It requires at most
one observation per subject and factor level and analyzes subjects having all
levels. Output includes the ordinary repeated-measures F-test, partial
eta-squared, Greenhouse-Geisser epsilon and corrected inference, marginal
means, and Holm-adjusted paired comparisons with Cohen's dz.

### Random-intercept mixed-effects models

The outcome comes first, the grouping variable second, and fixed predictors
follow:

```bash
python stats.py "$DATA_DIR/longitudinal_sample.csv" \
  --mixed-effects outcome subject time_index treatment age \
  --categorical-predictors treatment \
  --interaction time_index treatment
```

The default variance-component estimator is REML. Use `--mixed-method ml` for
maximum likelihood. The report includes fixed-effect estimates and confidence
intervals, residual and random-intercept variance, ICC, group-specific random-
intercept predictions, marginal and conditional R-squared, and conditional
casewise predictions. Polynomial and interaction options work as they do in
the other regression commands. Version 3.0 fits Gaussian random-intercept
models; random slopes and non-Gaussian mixed models are not yet included.

### Nonparametric tests

The independent samples are stored in separate columns. Compare two columns
with the Mann-Whitney U test:

```bash
python stats.py "$DATA_DIR/nonparametric_sample.csv" \
  --mann-whitney Control Treatment_1
```

For matched observations stored on the same rows, use the Wilcoxon signed-rank
test. Rows missing either member of a pair are removed together:

```bash
python stats.py "$DATA_DIR/nonparametric_sample.csv" --wilcoxon Before After
```

Compare three or more independent columns using Kruskal-Wallis:

```bash
python stats.py "$DATA_DIR/nonparametric_sample.csv" \
  --kruskal Control Treatment_1 Treatment_2
```

Add Dunn pairwise comparisons with Holm-adjusted p-values:

```bash
python stats.py "$DATA_DIR/nonparametric_sample.csv" \
  --kruskal Control Treatment_1 Treatment_2 \
  --posthoc
```

Holm is the default post-hoc adjustment. Select Bonferroni with
`--posthoc-adjust bonferroni`. The `--alternative` option applies to
Mann-Whitney and Wilcoxon; Kruskal-Wallis and Dunn comparisons are two-sided.

### Categorical tables and tests

Create one-way and two-way tables:

```bash
python stats.py "$DATA_DIR/categorical_sample.csv" --frequency Outcome
python stats.py "$DATA_DIR/categorical_sample.csv" --crosstab Treatment Outcome
```

Test whether two categorical variables are independent:

```bash
python stats.py "$DATA_DIR/categorical_sample.csv" --chi-square Treatment Outcome
```

For a goodness-of-fit test, expected probabilities must be listed in the same
order shown by the frequency table. They must be positive and sum to 1:

```bash
python stats.py "$DATA_DIR/categorical_sample.csv" \
  --frequency Color \
  --goodness-of-fit Color \
  --expected .40 .35 .25
```

Run Fisher's exact test when both variables have exactly two observed
categories:

```bash
python stats.py "$DATA_DIR/categorical_sample.csv" --fisher-exact Treatment Outcome
```

### Proportion tests

When the data have already been summarized as a success count and total trial
count, no data file is needed. For 30 successes in 100 trials, use:

```bash
python stats.py \
  --binomial-likelihood-counts 30 100 \
  --likelihood-p .20 .30 .50
```

The first count is the number of successes and the second is the total number
of trials. The report displays
`L(p) = C(100, 30) * p^30 * (1 - p)^70`, identifies the MLE as `p = .30`,
and evaluates the likelihood, log-likelihood, and relative likelihood at each
requested value. Use `--output` or `--json` normally if a saved report is
needed.

When individual observations are stored in a data file, the `--success` value
says which category is counted as a success. To calculate the same information
from a column, use:

```bash
python stats.py diabetes.csv \
  --binomial-likelihood diabetes --success Yes
```

Use `--likelihood-p` to evaluate the likelihood, log-likelihood, and relative
likelihood at one or more proposed population proportions:

```bash
python stats.py diabetes.csv \
  --binomial-likelihood diabetes --success Yes \
  --likelihood-p .20 .30 .50
```

Probabilities of 0 and 1 are permitted for likelihood evaluation. The
log-likelihood is retained even when the ordinary likelihood becomes too
small to represent accurately.

A one-proportion analysis reports both the large-sample z-test and the exact
binomial test, plus Wilson and Clopper-Pearson confidence intervals. It now
also reports the likelihood and log-likelihood at the null proportion, the
MLE, maximized log-likelihood, and relative likelihood:

```bash
python stats.py "$DATA_DIR/categorical_sample.csv" \
  --one-proportion Outcome --success Yes --p0 .50
```

For two independent proportions, provide a grouping column with exactly two
observed categories and a categorical outcome column:

```bash
python stats.py "$DATA_DIR/categorical_sample.csv" \
  --two-proportion Treatment Outcome --success Yes
```

Use `--alternative two-sided`, `less`, or `greater` for proportion tests and
Fisher's exact test. The default is `two-sided`.

## Combining analyses

Several analyses may be requested in one unattended command:

```bash
python stats.py data.csv \
  --paired Before After \
  --correlation Height Weight \
  --regression Outcome Predictor \
  --chi-square Treatment Response \
  --output report.txt \
  --json report.json
```

Use `python stats.py --help` for the complete command reference.

## Project structure

```text
stats.py                     command-line launcher
statclass_commands.txt       ready-to-copy command examples for every feature
statclass_3.17.0_example_csvs.zip  separately distributed, indexed example data
statclass/cli.py             arguments and analysis orchestration
statclass/data.py            loading, columns, and missing values
statclass/descriptives.py    summary and normality procedures
statclass/dependencies.py    installed dependency-version reporting
statclass/calculators.py     22 distributions, quantiles, moments, and direct CIs
statclass/discrete_estimation.py  estimation for discrete distribution families
statclass/power.py           power, sample-size, and detectable-effect analysis
statclass/bayesian.py        conjugate posterior and predictive calculations
statclass/transformations.py transformations, scaling, and outlier diagnostics
statclass/multivariate.py    PCA, one-way MANOVA, and linear discriminant analysis
statclass/inference.py       t-tests and z-tests
statclass/summary_inference.py  inference from sufficient summary statistics
statclass/anova.py           one-way ANOVA
statclass/nonparametric.py   rank tests and Dunn post-hoc comparisons
statclass/association.py     correlation procedures
statclass/regression.py      simple linear regression
statclass/multiple_regression.py  multiple OLS, diagnostics, and comparison
statclass/advanced_regression.py  robust, regularized, smooth, monotone, and KDE methods
statclass/logistic.py        binary and multinomial logistic regression
statclass/poisson.py         Poisson mean intervals, regression, and diagnostics
statclass/negative_binomial.py  NB2 count regression and Poisson comparison
statclass/numerical.py       numerical derivatives, information diagnostics, and covariance
statclass/factorial.py       two-way ANOVA, ANCOVA, and marginal means
statclass/longitudinal.py    repeated-measures ANOVA and mixed-effects models
statclass/plotting.py        headless explicit, model-aware, and Bayesian graphics
statclass/resampling.py      bootstrap intervals and permutation tests
statclass/design_terms.py    shared polynomial and interaction construction
statclass/categorical.py     frequency, crosstab, and categorical tests
statclass/proportions.py     binomial likelihood and proportion inference
statclass/reporting.py       text reports
statclass/utils.py           shared utilities
tests/test_statclass.py      numerical and command-line tests
tests/fixtures/              internal CSV fixtures used by automated tests
```

The loader retains the complete original table for categorical analyses and a
separate numeric representation for numerical analyses. Categorical-only CSV
files are supported.

## Running the tests

From the project directory:

```bash
python -m unittest discover -s tests -v
```

