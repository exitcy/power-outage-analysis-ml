---
layout: page
title: Home
---

# How Long Will the Lights Stay Out? Predicting U.S. Major Power Outage Duration

**Patrick Wu**

## Introduction

This project analyzes the **U.S. major power outage** dataset (`outage.xlsx`), a collection of federally reported electric disturbance events in the continental United States from **January 2000 through July 2016**. Each row represents one major outage event. After loading the Excel file with `skiprows=5` to skip metadata header rows, the dataset contains **1,535 rows** and **57 columns**. Variables describe when and where an outage occurred, what caused it, how long it lasted, how many customers were affected, and contextual information about climate, electricity prices, and state-level demographics for the affected area.

**Research question:** *How long do major power outages last, and what factors—especially outage cause—help explain or predict that duration?*

This question matters to the general public because power outages directly affect daily life: hospitals rely on electricity for critical equipment, homes depend on power for heating and cooling, and communication networks fail when the grid goes down. For utility planners and emergency responders, knowing how long an outage is likely to last supports decisions about where to send repair crews, how to stage backup resources, and how to prioritize grid resilience investments. Longer outages mean longer disruption to businesses, schools, and essential services—so understanding the drivers of outage duration has real-world stakes beyond the spreadsheet.

The columns most relevant to our research question are listed below, with descriptions of what each represents in the data generating process (utilities and regulators reporting major events to the Department of Energy under OE-417 requirements):

| Column | Description |
| --- | --- |
| `OUTAGE.DURATION` | Length of the outage in minutes. This is our primary outcome for regression and the variable compared across cause groups in hypothesis testing. |
| `CAUSE.CATEGORY` | Broad cause label assigned to the event (e.g., severe weather, intentional attack, equipment failure). Used in hypothesis testing and as a predictor in the final duration model. |
| `U.S._STATE` | U.S. state where the outage occurred. Captures geographic context—different states have different grid infrastructure, weather exposure, and regulatory environments. |
| `CLIMATE.REGION` | NOAA-style climate region for the affected area. Encodes regional weather patterns that influence outage severity and restoration difficulty. |
| `MONTH` | Calendar month (1–12) when the outage started. Encodes seasonality (e.g., winter ice storms vs. summer heat events) in baseline and final models. |
| `ANOMALY.LEVEL` | Climate anomaly level associated with the event. An ordinal measure of how unusual local climate conditions were at the time of the outage. |
| `TOTAL.CUSTOMERS` | Total number of electricity customers in the affected area. A proxy for grid scale and population density; also used to define fairness groups. |
| `CUSTOMERS.AFFECTED` | Number of customers who lost power during the event. A severity measure with substantial missingness analyzed in the missingness section. |
| `YEAR` | Year of the outage (2000–2016). Useful for examining temporal trends in exploratory analysis. |
| `OUTAGE.START.DATE` | Date the outage began. Combined with start time during cleaning to construct timestamps when needed. |
| `OUTAGE.START.TIME` | Time the outage began. Paired with start date to record when the event started in the reporting system. |

## Data Cleaning and Exploratory Data Analysis

### Data Cleaning

Every cleaning step below is tied to how this dataset was actually produced: utilities and balancing authorities submit standardized OE-417 disturbance reports to the Department of Energy. Those reports arrive as structured Excel files with metadata rows, mixed text/numeric fields, and inconsistent string formatting across years and reporting entities.

1. **Load with `skiprows=5`.** The raw Excel file contains five header/metadata rows before the first data record. Skipping them ensures each row in our DataFrame corresponds to one reported outage event rather than documentation text. Without this step, summary statistics and row counts would be wrong.

2. **Drop the spurious variable-definition row.** After loading, the first row is sometimes a repeated column-name or variable glossary row (detected when all values are null or the row contains the word "variables"). Removing it prevents a non-event row from entering plots and models.

3. **Clean `OUTAGE.DURATION`.** In the raw reports, duration is sometimes stored as text with a `"mins"` suffix (e.g., `"3060 mins"`). We strip that suffix, trim whitespace, and coerce to numeric minutes. Invalid values become `NaN` and are dropped for modeling. This directly affects all duration analyses: without it, means, regression targets, and hypothesis tests would silently drop rows or fail to parse values.

4. **Standardize `CAUSE.CATEGORY` labels.** Cause strings may include inconsistent capitalization or trailing spaces from manual data entry across reporting years. We strip and lowercase labels (stored as `CAUSE.CATEGORY_CLEAN`) so that group comparisons—especially severe weather vs. intentional attack in hypothesis testing—match the intended categories rather than missing rows due to string mismatches.

5. **Filter for analysis subsets.** For cause-based plots and tables, rows missing `CAUSE.CATEGORY` are excluded because an unlabeled event cannot be interpreted in a cause-driven analysis. For duration modeling, rows missing `OUTAGE.DURATION` are excluded since duration is the response variable.

These steps ensure that downstream permutation tests, regression models, and fairness analyses operate on a consistent, event-level table where each row is one real outage with a numeric duration and interpretable cause label.

**Head of the cleaned DataFrame** (relevant columns, after cleaning and filtering rows with valid cause and duration):

| OUTAGE.DURATION | CAUSE.CATEGORY | U.S._STATE | CLIMATE.REGION | MONTH | TOTAL.CUSTOMERS |
| --- | --- | --- | --- | --- | --- |
| 3060.0 | severe weather | Minnesota | East North Central | 7.0 | 2595696.0 |
| 1.0 | intentional attack | Minnesota | East North Central | 5.0 | 2640737.0 |
| 3000.0 | severe weather | Minnesota | East North Central | 10.0 | 2586905.0 |
| 2550.0 | severe weather | Minnesota | East North Central | 6.0 | 2606813.0 |
| 1740.0 | severe weather | Minnesota | East North Central | 7.0 | 2673531.0 |

### Univariate Analysis

The histogram below shows the distribution of outage events across cause categories.

<iframe
  src="{{ '/assets/univariate_cause_histogram.html' | relative_url }}"
  width="800"
  height="600"
  frameborder="0"
></iframe>

Severe weather is the most common cause category, accounting for roughly half of all labeled outages in the dataset, while intentional attacks form the second-largest group. This imbalance motivates our hypothesis test comparing duration between these two high-frequency cause types. A second univariate plot in the project notebook examines the right-skewed distribution of `OUTAGE.DURATION` in minutes.

### Bivariate Analysis

The box plot below displays the relationship between outage cause category and outage duration in minutes.

<iframe
  src="{{ '/assets/bivariate_duration_by_cause.html' | relative_url }}"
  width="800"
  height="600"
  frameborder="0"
></iframe>

Severe weather outages show substantially higher median and upper-quartile durations than intentional attacks, with severe weather events frequently lasting thousands of minutes while attack-related outages cluster near very short durations. Fuel supply emergencies show the highest mean duration but occur far less frequently, suggesting cause category is a strong candidate predictor for our regression task. The notebook also includes a scatter plot of duration versus total customers (log scale) to explore grid-scale effects alongside cause.

### Interesting Aggregates

The table below summarizes outage count, mean duration, and median duration by cause category.

| CAUSE.CATEGORY | count | mean_minutes | median_minutes |
| --- | --- | --- | --- |
| severe weather | 744 | 3883.99 | 2460.0 |
| intentional attack | 403 | 429.98 | 56.0 |
| system operability disruption | 123 | 728.87 | 215.0 |
| public appeal | 69 | 1468.45 | 455.0 |
| equipment failure | 55 | 1816.91 | 221.0 |
| islanding | 44 | 200.55 | 77.5 |
| fuel supply emergency | 38 | 13484.03 | 3960.0 |

The aggregate table reveals a statistically meaningful pattern: severe weather outages (n = 744) have a mean duration of **3,884 minutes** compared to **430 minutes** for intentional attacks (n = 403)—a nearly ninefold difference in average outage length. The median tells a similar story (2,460 vs. 56 minutes), indicating the gap is not driven solely by a few extreme severe-weather outliers. Fuel supply emergencies have the highest mean duration (13,484 minutes) but only 38 events, so they contribute less to overall prediction variance. This table directly supports our core research question by showing that cause category is strongly associated with how long customers remain without power.

## Assessment of Missingness

### NMAR Analysis

We focus on **`CUSTOMERS.AFFECTED`**, which has a missingness rate of **28.9%** (443 of 1,535 rows).

We believe `CUSTOMERS.AFFECTED` is plausibly **NMAR** (Not Missing At Random). In the data generating process, utilities report customer impact counts to federal regulators after an event occurs. Whether that count is recorded depends on factors we do not fully observe in this public dataset: internal utility assessment timelines, whether the event met formal reporting thresholds at the time of filing, whether damage surveys were complete, and whether certain cause types (e.g., intentional attacks or cyber incidents) led to delayed or redacted impact figures for security reasons. A missing value therefore may depend on the true (unobserved) customer impact itself or on unobserved reporting workflow—not just on the columns we can see.

Looking at the data alone is not enough to confirm NMAR. Our permutation tests below show that missingness depends on observed `CAUSE.CATEGORY` (consistent with MAR), but they cannot distinguish MAR from NMAR because both mechanisms can produce dependency on observed variables. To move toward MAR—and to explain missingness more fully—we would need **additional data** such as: internal utility outage ticket timestamps, OE-417 submission dates relative to restoration, regulatory completeness audit flags, and standardized reporting-threshold documentation by utility and year. With those fields, missingness could be modeled conditional on observed filing status rather than unobserved reporting decisions.

### Missingness Dependency

We test whether missingness in `CUSTOMERS.AFFECTED` depends on other columns using permutation tests (2,000 repetitions, α = 0.05):

| column_tested | test_type | statistic | p_value |
| --- | --- | --- | --- |
| CAUSE.CATEGORY | categorical missingness-permutation | 0.098105 | 0.000500 |
| TOTAL.CUSTOMERS | numeric missingness-permutation | 59563.64 | 0.807096 |

**Interpretation:** At α = 0.05, we reject the independence hypothesis for `CAUSE.CATEGORY` (p = 0.0005): the proportion of missing `CUSTOMERS.AFFECTED` values varies significantly across cause categories. We fail to reject independence for `TOTAL.CUSTOMERS` (p = 0.807): grid scale does not appear to drive whether customer impact is reported. Because missingness depends on at least one observed column, the pattern is more consistent with **MAR** than **MCAR**, though—as argued above—this does not rule out an NMAR component tied to unobserved reporting processes.

<iframe
  src="{{ '/assets/missingness_permutation_null.html' | relative_url }}"
  width="800"
  height="600"
  frameborder="0"
></iframe>

The plot above shows the empirical null distribution of the test statistic (variance of group-wise missingness rates) from permuting missingness labels for `CAUSE.CATEGORY`. The observed statistic (red dashed line) falls far in the right tail, confirming that missingness is not independent of cause category.

## Hypothesis Testing

We test whether outages caused by **severe weather** have a different average duration than those caused by **intentional attacks**—the two most common cause categories and the pair with the largest apparent duration gap in our EDA.

- **Null Hypothesis (H₀):** The duration of power outages caused by severe weather and those caused by intentional attacks come from the same underlying distribution (equal population means).
- **Alternative Hypothesis (Hₐ):** Power outages caused by severe weather have a different average duration than power outages caused by intentional attacks.

**Test statistic:** Absolute difference in sample means, |x̄_severe − x̄_attack| = **3,454.01 minutes**.

**Significance level:** α = **0.05**.

**Method:** Permutation test with 3,000 repetitions, shuffling duration labels between the two groups while holding group sizes fixed (n_severe = 744, n_attack = 403).

**Results:**
- Mean severe-weather duration: **3,883.99 minutes**
- Mean intentional-attack duration: **429.98 minutes**
- **p-value = 0.00033**

**Decision:** Because p < α, we **reject H₀**. There is statistically significant evidence that average outage duration differs between severe weather and intentional attack events.

**Justification:** A permutation test is appropriate here because it makes no normality assumption about duration distributions, which are right-skewed with long tails. The test directly targets our research question about cause and duration, uses large sample sizes in both groups, and aligns with the aggregate patterns found in EDA.

<!-- AUTHOR REMINDER: Do not use language that implies absolute conclusions (e.g., "proves true" or "proved false"). This is a statistical test on observational data, not a randomized controlled trial—we cannot establish either hypothesis as 100% true or false. -->

<iframe
  src="{{ '/assets/hypothesis_permutation_null.html' | relative_url }}"
  width="800"
  height="600"
  frameborder="0"
></iframe>

## Framing a Prediction Problem

**Prediction problem:** Given contextual information available early in a major outage event, predict how long the outage will last (in minutes).

**Problem type:** **Regression** — the response variable is continuous.

**Response variable:** `OUTAGE.DURATION` (minutes). We selected this target because it is the direct quantitative answer to our research question. Our hypothesis test showed that cause category is strongly associated with duration, and planners need a minute-level forecast to schedule crew deployments and communicate restoration timelines to the public.

**Evaluation metric:** **Root Mean Squared Error (RMSE)** on held-out test data (20% split, `random_state=80`). RMSE penalizes large prediction errors more heavily than mean absolute error, which matters when underestimating a multi-day outage is costlier than a small timing error. We report R² as a secondary metric to describe variance explained, but R² alone does not measure error magnitude in minutes—so RMSE is our primary score. Accuracy and F1 are not applicable because this is a regression task, not classification.

**Time-of-prediction justification (avoiding data leakage):**

We frame the prediction as occurring **shortly after an outage is reported and classified**, when a utility planner knows:

| Feature | Available at prediction time? | Rationale |
| --- | --- | --- |
| `MONTH` | Yes | Outage start month is recorded at event onset. |
| `U.S._STATE` | Yes | Location is known immediately. |
| `CLIMATE.REGION` | Yes | Derived from state/region mapping in the dataset. |
| `ANOMALY.LEVEL` | Yes | Climate context for the event period is observable at start. |
| `TOTAL.CUSTOMERS` | Yes | Grid scale for the affected area is a fixed infrastructure attribute. |
| `CAUSE.CATEGORY` | Yes (final model) | Initial cause classification is assigned early in the reporting process. |
| `OUTAGE.DURATION` | **No** | This is the target—we cannot use the answer to predict itself. |
| `OUTAGE.RESTORATION.DATE/TIME` | **No** | Restoration timestamps are only known after the outage ends (look-ahead bias). |
| `CUSTOMERS.AFFECTED` | **No** | Often missing or updated after initial report; not reliably known at onset. |
| Post-outcome economic fields | **No** | Price, sales, and GSP figures describe the billing period, not early outage conditions. |

The baseline model intentionally excludes `CAUSE.CATEGORY` to establish a lower bound using only location, season, climate, and grid scale. The final model adds cause category because it is classified early and our hypothesis test showed it is strongly predictive—without using any post-outcome restoration information.

## Baseline Model

**Model:** `LinearRegression` wrapped in a single scikit-learn `Pipeline` (preprocessing + estimator).

**Features used (5 total):**

| Feature | Type |
| --- | --- |
| `MONTH` | Quantitative |
| `ANOMALY.LEVEL` | Ordinal |
| `TOTAL.CUSTOMERS` | Quantitative |
| `CLIMATE.REGION` | Nominal |
| `U.S._STATE` | Nominal |

**Feature type breakdown:** 2 quantitative, 1 ordinal, 2 nominal (5 features total, satisfying the ≥2 feature requirement).

**Encoding and transformations (all inside one Pipeline):**
- **Quantitative/ordinal numerics** (`MONTH`, `ANOMALY.LEVEL`, `TOTAL.CUSTOMERS`): `SimpleImputer(strategy='median')` → `StandardScaler()`.
- **Nominal categoricals** (`CLIMATE.REGION`, `U.S._STATE`): `SimpleImputer(strategy='most_frequent')` → `OneHotEncoder(handle_unknown='ignore')`.

**Train/test split:** 80/20 hold-out, `random_state=80`.

**Performance on unseen test data:**
- **Test RMSE: 5,179.97 minutes**
- **Test R²: 0.0048**

**Is the baseline good?** No. The model explains essentially none of the variance in outage duration (R² ≈ 0) and performs only marginally better than always predicting the training-set mean duration (mean-prediction reference RMSE = 5,194.47 minutes). This is expected: a linear model without cause category cannot capture the large nonlinear differences between outage types shown in EDA and hypothesis testing. The baseline establishes a fair lower bound before adding engineered features and `RandomForestRegressor`.

## Final Model

**New features engineered on top of baseline encodings:**

1. **`log1p(TOTAL.CUSTOMERS)`** — Customer counts are right-skewed across utilities and states. In the data generating process, a small rural co-op and a large metropolitan utility operate at vastly different scales; a log transform lets tree splits separate "small grid" from "large grid" effects without being dominated by a few extreme values.

2. **`QuantileTransformer` on `ANOMALY.LEVEL`** — Anomaly level is an ordinal climate severity score. Transforming it to a normal-like scale helps the forest use rank-based thresholds when raw magnitudes are sparse or unevenly spaced across events.

3. **`sin(2π · MONTH / 12)`** — Month is cyclical: December (12) is adjacent to January (1) in the calendar but far apart as a raw integer. Seasonal outage drivers (winter ice, summer heat) repeat annually, so a sine feature encodes that cycle in a way linear month cannot.

4. **`CAUSE.CATEGORY` added as a predictor** — Severe weather and intentional attacks differ by thousands of minutes on average in our hypothesis test. Cause is assigned early in the reporting process and reflects fundamentally different physical recovery workflows (e.g., storm damage repair vs. localized vandalism), making it a high-value feature from a data generating process perspective.

**Algorithm:** `RandomForestRegressor` inside a single sklearn `Pipeline`.

**Hyperparameter tuning:** `GridSearchCV` with 5-fold cross-validation on **training data only**, scoring = `neg_root_mean_squared_error`.

**Best hyperparameters:**
- `max_depth`: None
- `min_samples_leaf`: 10
- `n_estimators`: 100

**Validation:** 5-fold CV mean RMSE = 5,482.67 minutes; best pipeline refit on full training set before test evaluation.

**Performance comparison on the same held-out test split:**

| Model | Test RMSE (minutes) | Test R² |
| --- | --- | --- |
| Baseline (`LinearRegression`) | 5,179.97 | 0.0048 |
| Final (`RandomForestRegressor`) | 4,288.72 | 0.3178 |

The final model reduces test RMSE by **891.24 minutes** (~17%) and explains roughly **32%** of duration variance compared to near-zero for the baseline. The improvement comes primarily from incorporating cause category and nonlinear seasonality/scale features that the linear baseline could not represent—consistent with the strong cause–duration relationship in our hypothesis test and EDA.

<iframe
  src="{{ '/assets/final_model_residuals.html' | relative_url }}"
  width="800"
  height="600"
  frameborder="0"
></iframe>

## Fairness Analysis

We ask whether the **final duration model** predicts equally well for outages in high-population grid areas versus lower-population areas—an equity concern because underestimating duration where more customers are served could lead to under-allocation of repair resources.

**Group definitions:**
- **Group X (high-impact):
** Outages where `TOTAL.CUSTOMERS` is at or above the training-set median (**3,957,980 customers**).
- **Group Y (low-impact):** Outages where `TOTAL.CUSTOMERS` is below the training-set median.

**Evaluation metric:** RMSE (same regression metric as the prediction problem), computed separately on the held-out test set for each group.

**Hypotheses:**
- **Null Hypothesis (H₀):** The model is fair. RMSE for high-impact and low-impact outages are roughly the same; any observed difference is due to chance.
- **Alternative Hypothesis (Hₐ):** The model is unfair. RMSE for high-impact outages is **greater** than RMSE for low-impact outages (worse predictions where more people are served).

**Test statistic:** RMSE_high − RMSE_low = **2,101.78 minutes**.

**Significance level:** α = **0.05**.

**Method:** Permutation test (2,000 repetitions) shuffling group labels on the test set while keeping model predictions fixed.

<!-- AUTHOR REMINDER: The permutation test must use the final, unmodified fitted model—no retraining during the fairness test. -->

**Results (test set):**
- n(high-impact) = 145, RMSE = **5,230.23 minutes**
- n(low-impact) = 151, RMSE = **3,128.44 minutes**
- **p-value = 0.1179**

**Decision:** Because p > α, we **fail to reject H₀**. There is no statistically significant evidence at the 5% level that RMSE differs between high- and low-impact groups.

**Interpretation:** The point estimate shows higher error for high-impact outages (RMSE difference ≈ 2,102 minutes), which warrants monitoring from a policy perspective. However, with test-set groups of roughly 145–151 events each and high duration variance, the permutation test does not find this gap statistically significant. We cannot conclude the model is unfair based on this test alone, but we also cannot claim perfect equity across grid scales.

<iframe
  src="{{ '/assets/fairness_permutation_null.html' | relative_url }}"
  width="800"
  height="600"
  frameborder="0"
></iframe>
