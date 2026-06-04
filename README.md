

# Predicting U.S. Major Power Outage Duration

**By Patrick Wu**

## Introduction

This project analyzes the U.S. major power outage dataset (`outage.xlsx`), a collection of federally reported electric disturbance events in the continental United States from January 2000 through July 2016. Each row represents one major outage event. After loading the Excel file with `skiprows=5` to skip metadata header rows, the dataset contains 1,535 rows and 57 columns. Variables describe when and where an outage occurred, what caused it, how long it lasted, how many customers were affected, and contextual information about climate, electricity prices, and state-level demographics for the affected area.

**Research question:** *How long do major power outages last, and what factors, especially outage cause, help explain or predict that duration?*

This question matters because power outages disrupt daily life. Hospitals lose power for critical equipment, homes lose heating and cooling, and cell networks go down. For utility planners and emergency crews, knowing how long a blackout might last helps them deploy repair teams faster, stage backup generators, and strengthen vulnerable parts of the grid. Essentially, longer outages mean deeper disruptions to businesses and schools, so finding out what drives these delays has real-world stakes far beyond a spreadsheet.

The columns most relevant to the research question are listed below, with descriptions of what each represents in the data generating process:

| Column | Description |
| --- | --- |
| `OUTAGE.DURATION` | Length of the outage in minutes. This is our primary outcome for regression and the variable compared across cause groups in hypothesis testing. |
| `CAUSE.CATEGORY` | Broad cause label assigned to the event (e.g., severe weather, intentional attack, equipment failure). Used in hypothesis testing and as a predictor in the final duration model. |
| `U.S._STATE` | U.S. state where the outage occurred. Captures geographic context, different states have different grid infrastructure, weather exposure, and regulatory environments. |
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

The cleaning steps fix the issues built right into the raw data. The Department of Energy collects these OE-417 disturbance reports from utilities as Excel files. Because they span multiple years and entities, the raw files are messy, filled with metadata headers, mixed data types, and inconsistent text formatting.

1. **Load with `skiprows=5`.** The raw Excel file contains five header/metadata rows before the first data record. Skipping them ensures each row in our DataFrame corresponds to one reported outage event rather than documentation text. Without this step, summary statistics and row counts would be wrong.

2. **Drop the spurious variable-definition row.** After loading, the first row is sometimes a repeated column-name or variable glossary row (detected when all values are null or the row contains the word "variables"). Removing it prevents a non-event row from entering plots and models.

3. **Clean `OUTAGE.DURATION`.** In the raw reports, duration is sometimes stored as text with a `"mins"` suffix (e.g., `"3060 mins"`). We strip that suffix, trim whitespace, and coerce to numeric minutes. Invalid values become `NaN` and are dropped for modeling. This directly affects all duration analyses as without it, means, regression targets, and hypothesis tests would silently drop rows or fail to parse values.

4. **Standardize `CAUSE.CATEGORY` labels.** Cause strings may include inconsistent capitalization or trailing spaces from manual data entry across reporting years. We strip and lowercase labels (stored as `CAUSE.CATEGORY_CLEAN`) so that group comparisons, especially severe weather vs. intentional attack in hypothesis testing, match the intended categories rather than missing rows due to string mismatches.

5. **Filter for analysis subsets.** For cause-based plots and tables, rows missing `CAUSE.CATEGORY` are excluded because an unlabeled event cannot be interpreted in a cause-driven analysis. For duration modeling, rows missing `OUTAGE.DURATION` are excluded since duration is the response variable.

These steps ensure that downstream permutation tests, regression models, and fairness analyses operate on a consistent, event-level table where each row is one real outage with a numeric duration and interpretable cause label.

**Head of the cleaned DataFrame**:

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

Severe weather is the most common cause category, accounting for roughly half of all labeled outages in the dataset, while intentional attacks form the second-largest group. This imbalance motivates the hypothesis test comparing duration between these two high-frequency cause types.

The histogram below shows the distribution of `OUTAGE.DURATION` in minutes.

<iframe
  src="{{ '/assets/univariate_duration_histogram.html' | relative_url }}"
  width="800"
  height="600"
  frameborder="0"
></iframe>

Outage duration is strongly right-skewed, most events cluster at shorter lengths, but a long tail of multi-hour and multi-day outages pulls the mean well above the median. That skew motivates using RMSE, which penalizes large errors.

### Bivariate Analysis

The box plot below displays the relationship between outage cause category and outage duration in minutes.

<iframe
  src="{{ '/assets/bivariate_duration_by_cause.html' | relative_url }}"
  width="800"
  height="600"
  frameborder="0"
></iframe>

Severe weather causes much longer outages than intentional attacks. While weather events often last for thousands of minutes, attacks are usually over very quickly. On the other hand, fuel supply shortages cause the longest average outages but rarely happen. Overall, this shows that the cause of an outage is a great predictor for our model.

The scatter plot below shows outage duration versus total customers in the affected area (log scale), colored by cause category.

<iframe
  src="{{ '/assets/bivariate_duration_vs_customers.html' | relative_url }}"
  width="800"
  height="600"
  frameborder="0"
></iframe>

Bigger grids don't always mean longer outages. Instead, the cause of the outage matters most. However, severe weather can cause massive delays regardless of grid size, which proves that our final model needs to look at both grid size and the cause.

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

The data shows a massive gap in outage lengths based on their cause. Severe weather outages average 3,884 minutes, while intentional attacks average just 430 minutes, a nearly ninefold difference. Looking at typical middle-ground cases (the medians) confirms this isn't just skewed by a few extreme storms. Meanwhile, fuel supply emergencies drag on the longest (averaging over 13,000 minutes) but are rare, with only 38 events. Therefore ,the cause of an outage strongly predicts how long people lose power.

## Assessment of Missingness

### NMAR Analysis

The focus is on **`CUSTOMERS.AFFECTED`**, which has a missingness rate of **28.9%** (443 of 1,535 rows).

I suspect that the missing data in 'CUSTOMERS.AFFECTED' is NMAR (Not Missing At Random). This means the reason a value is missing depends on information we simply don't have. For instance, a utility company might leave out the customer count because they missed the reporting deadline, their damage survey wasn't done, or the event was a cyberattack redacted for security reasons. Because the missingness depends on these hidden workflows or the true scale of the outage itself, we can't explain it using just our visible data.

A statistical test alone cannot prove a dataset is NMAR. While the permutation tests show that missing values are linked to the 'CAUSE.CATEGORY' (which points to MAR), tests cannot distinguish between the two mechanisms. To truly rule out NMAR and explain the missing data,  extra variables: like internal utility ticket timestamps, filing dates, and regulatory audit flags are needed. These extra fields would let us model the missing data based on clear, observed filing rules rather than unobserved corporate decisions.

### Missingness Dependency

We test whether missingness in `CUSTOMERS.AFFECTED` depends on other observed columns using permutation tests (2,000 repetitions, α = 0.05). For each test we shuffle the missingness indicator while holding the comparison column fixed, then compare the observed statistic to the simulated null distribution.

#### Test 1: `CAUSE.CATEGORY` (expected dependence)

- **Null Hypothesis (H₀):** Missingness in `CUSTOMERS.AFFECTED` is independent of `CAUSE.CATEGORY`. The proportion of missing customer-impact values is the same across cause groups (after accounting for group size).
- **Alternative Hypothesis (Hₐ):** Missingness in `CUSTOMERS.AFFECTED` depends on `CAUSE.CATEGORY, at least one cause group has a different missingness rate.
- **Test statistic:** Variance of group-wise missingness rates (proportion missing `CUSTOMERS.AFFECTED` within each cause category, including outages with unlabeled cause as `__MISSING__`).
- **Observed statistic:** **0.098105** | **p-value:** **0.000500**

The grouped bar chart below compares the distribution of `CAUSE.CATEGORY` when customer impact is missing versus observed. When impact is missing, intentional attacks and public appeals make up a much larger share of events than when impact is recorded, suggesting reporting practices differ by cause type.

<iframe
  src="{{ '/assets/missingness_cause_by_impact_status.html' | relative_url }}"
  width="800"
  height="600"
  frameborder="0"
></iframe>

<iframe
  src="{{ '/assets/missingness_permutation_null.html' | relative_url }}"
  width="800"
  height="600"
  frameborder="0"
></iframe>

**Decision:** Because p < α, we **reject H₀**. Missingness in `CUSTOMERS.AFFECTED` is not independent of `CAUSE.CATEGORY`. The observed statistic (red dashed line) lies far in the right tail of the permutation null distribution above.

#### Test 2: `TOTAL.CUSTOMERS` (control, expected no dependence)

- **Null Hypothesis (H₀):** Missingness in `CUSTOMERS.AFFECTED` is independent of `TOTAL.CUSTOMERS`. The mean grid scale (total customers in the affected area) is the same whether customer impact is missing or observed.
- **Alternative Hypothesis (Hₐ):** Missingness in `CUSTOMERS.AFFECTED` depends on `TOTAL.CUSTOMERS`, mean grid scale differs between rows with missing versus observed impact.
- **Test statistic:** Absolute difference in mean `TOTAL.CUSTOMERS` between rows with missing versus observed `CUSTOMERS.AFFECTED`.
- **Observed statistic:** **59,563.64** | **p-value:** **0.807096**

<iframe
  src="{{ '/assets/missingness_total_customers_by_impact_status.html' | relative_url }}"
  width="800"
  height="600"
  frameborder="0"
></iframe>

<iframe
  src="{{ '/assets/missingness_total_customers_permutation_null.html' | relative_url }}"
  width="800"
  height="600"
  frameborder="0"
></iframe>

**Decision:** Because p > α, we **fail to reject H₀**. There is no statistically significant evidence that missingness in `CUSTOMERS.AFFECTED` depends on `TOTAL.CUSTOMERS`. The box plot and permutation null distribution are consistent with similar grid scales whether impact is reported or not.

#### Summary

| column_tested | test_type | statistic | p_value | decision (α = 0.05) |
| --- | --- | --- | --- | --- |
| CAUSE.CATEGORY | categorical missingness-permutation | 0.098105 | 0.000500 | Reject H₀ (depends) |
| TOTAL.CUSTOMERS | numeric missingness-permutation | 59563.64 | 0.807096 | Fail to reject H₀ (no dependence) |

Because missingness depends on at least one observed column (`CAUSE.CATEGORY`) but not on `TOTAL.CUSTOMERS`, the pattern is more consistent with MAR than MCAR, though, as argued in the NMAR section above, this does not rule out an NMAR component tied to unobserved reporting processes.

## Hypothesis Testing

We test whether outages caused by severe weather have a different average duration than those caused by intentional attacks, the two most common cause categories and the pair with the largest apparent duration gap in our EDA.

- **Null Hypothesis (H₀):** The duration of power outages caused by severe weather and those caused by intentional attacks come from the same underlying distribution (equal population means).
- **Alternative Hypothesis (Hₐ):** Power outages caused by severe weather have a different average duration than power outages caused by intentional attacks.

**Test statistic:** Absolute difference in sample means, |x̄_severe − x̄_attack| = **3,454.01 minutes**.

**Significance level:** α = 0.05.

**Method:** Permutation test with 3,000 repetitions, shuffling duration labels between the two groups while holding group sizes fixed (n_severe = 744, n_attack = 403).

**Results:**
- Mean severe-weather duration: **3,883.99 minutes**
- Mean intentional-attack duration: **429.98 minutes**
- **p-value = 0.00033**

**Decision:** Because p < α, we **reject H₀**. There is statistically significant evidence that average outage duration differs between severe weather and intentional attack events.

**Justification:** We chose a permutation test because it doesn't assume our data follows a normal distribution, which is crucial since outage durations are heavily skewed with long tails. This test directly answers whether cause affects duration, handles our large sample sizes well, and matches our initial data findings.

The plot below displays the test results. The bell-shaped curve shows what the differences in outage lengths would look like by pure chance (the null distribution). Our actual observed difference (the red dashed line) sits far to the right, proving that severe weather and intentional attacks cause significantly different outage lengths.

<iframe
  src="{{ '/assets/hypothesis_permutation_null.html' | relative_url }}"
  width="800"
  height="600"
  frameborder="0"
></iframe>

## Framing a Prediction Problem

**Prediction problem:** Given contextual information available early in a major outage event, predict how long the outage will last (in minutes).

**Problem type:** Regression, as the response variable is continuous.

**Response variable:** `OUTAGE.DURATION` (minutes). We selected this target because it is the direct quantitative answer to our research question. Our hypothesis test showed that cause category is strongly associated with duration, and planners need a minute-level forecast to schedule crew deployments and communicate restoration timelines to the public.

**Evaluation metric:** **Root Mean Squared Error (RMSE)** on held-out test data (20% split, `random_state=80`). RMSE penalizes large prediction errors more heavily than mean absolute error, which matters when underestimating a multi-day outage is costlier than a small timing error. We report R² as a secondary metric to describe variance explained, but R² alone does not measure error magnitude in minutes, so RMSE is our primary score. Accuracy and F1 are not applicable because this is a regression task, not classification.

**Time-of-prediction justification:**

We frame the prediction as occurring shortly after an outage is reported and classified, when a utility planner knows:

| Feature | Available at prediction time? | Rationale |
| --- | --- | --- |
| `MONTH` | Yes | Outage start month is recorded at event onset. |
| `U.S._STATE` | Yes | Location is known immediately. |
| `CLIMATE.REGION` | Yes | Derived from state/region mapping in the dataset. |
| `ANOMALY.LEVEL` | Yes | Climate context for the event period is observable at start. |
| `TOTAL.CUSTOMERS` | Yes | Grid scale for the affected area is a fixed infrastructure attribute. |
| `CAUSE.CATEGORY` | Yes (final model) | Initial cause classification is assigned early in the reporting process. |
| `OUTAGE.DURATION` | **No** | This is the target, we cannot use the answer to predict itself. |
| `OUTAGE.RESTORATION.DATE/TIME` | **No** | Restoration timestamps are only known after the outage ends (look-ahead bias). |
| `CUSTOMERS.AFFECTED` | **No** | Often missing or updated after initial report; not reliably known at onset. |
| Post-outcome economic fields | **No** | Price, sales, and GSP figures describe the billing period, not early outage conditions. |

The baseline model intentionally excludes `CAUSE.CATEGORY` to establish a lower bound using only location, season, climate, and grid scale. The final model adds cause category because it is classified early and our hypothesis test showed it is strongly predictive, without using any post-outcome restoration information.

## Baseline Model

**Model:** `LinearRegression` wrapped in a single scikit-learn `Pipeline` (preprocessing + estimator).

**Features used:**

| Feature | Type |
| --- | --- |
| `MONTH` | Quantitative |
| `ANOMALY.LEVEL` | Ordinal |
| `TOTAL.CUSTOMERS` | Quantitative |
| `CLIMATE.REGION` | Nominal |
| `U.S._STATE` | Nominal |

**Feature type breakdown:** 2 quantitative, 1 ordinal, 2 nominal.

**Encoding and transformations:**
- **Quantitative/ordinal numerics** (`MONTH`, `ANOMALY.LEVEL`, `TOTAL.CUSTOMERS`): `SimpleImputer(strategy='median')` → `StandardScaler()`.
- **Nominal categoricals** (`CLIMATE.REGION`, `U.S._STATE`): `SimpleImputer(strategy='most_frequent')` → `OneHotEncoder(handle_unknown='ignore')`.

**Train/test split:** 80/20 hold-out, `random_state=80`.

**Performance on unseen test data:**
- **Test RMSE: 5,179.97 minutes**
- **Test R²: 0.0048**

**Is the baseline good?** No. The baseline model explains almost none of the variation in outage lengths, yielding an R² score of approximately 0. In fact, it performs barely better than just guessing the average duration for every single outage (baseline error is 5,194.47 minutes).This failure makes sense: a simple linear model that ignores the outage cause can't capture the massive differences between severe weather and attacks that we uncovered earlier. This baseline simply sets a realistic starting point before we bring in engineered features and a 'RandomForestRegressor'.

## Final Model

**New features engineered on top of baseline encodings:**

1. **`log1p(TOTAL.CUSTOMERS)`:** Customer counts are right-skewed across utilities and states. In the data generating process, a small rural co-op and a large metropolitan utility operate at vastly different scales; a log transform lets tree splits separate "small grid" from "large grid" effects without being dominated by a few extreme values.

2. **`QuantileTransformer` on `ANOMALY.LEVEL`:** Anomaly level is an ordinal climate severity score. Transforming it to a normal-like scale helps the forest use rank-based thresholds when raw magnitudes are sparse or unevenly spaced across events.

3. **`sin(2π · MONTH / 12)`:** Month is cyclical: December (12) is adjacent to January (1) in the calendar but far apart as a raw integer. Seasonal outage drivers repeat annually, so a sine feature encodes that cycle in a way linear month cannot.

4. **`CAUSE.CATEGORY`:** Severe weather and intentional attacks differ by thousands of minutes on average in our hypothesis test. Cause is assigned early in the reporting process and reflects  different physical recovery workflows (e.g., storm damage repair vs. localized vandalism), making it a high-value feature from a data generating process perspective.

**Algorithm:** `RandomForestRegressor` inside a single sklearn `Pipeline`.

**Hyperparameter tuning:** `GridSearchCV` with 5-fold cross-validation on training data only, scoring = `neg_root_mean_squared_error`.

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

The final model reduces test RMSE by **891.24 minutes** (~17%) and explains roughly **32%** of duration variance compared to near-zero for the baseline. The improvement comes primarily from incorporating cause category and nonlinear seasonality/scale features that the linear baseline could not represent, consistent with the strong cause–duration relationship in our hypothesis test and EDA.

<iframe
  src="{{ '/assets/final_model_residuals.html' | relative_url }}"
  width="800"
  height="600"
  frameborder="0"
></iframe>

## Fairness Analysis

We ask whether the final duration model predicts equally well for outages in high-population grid areas versus lower-population areas, an equity concern because underestimating duration where more customers are served could lead to under-allocation of repair resources.

**Group definitions:**
- **Group X (high-impact):** Outages where `TOTAL.CUSTOMERS` is at or above the training-set median (**3,957,980 customers**).
- **Group Y (low-impact):** Outages where `TOTAL.CUSTOMERS` is below the training-set median.

**Evaluation metric:** RMSE (same regression metric as the prediction problem), computed separately on the held-out test set for each group.

**Hypotheses:**
- **Null Hypothesis (H₀):** The model is fair. RMSE for high-impact and low-impact outages are roughly the same; any observed difference is due to chance.
- **Alternative Hypothesis (Hₐ):** The model is unfair. RMSE for high-impact outages is greater than RMSE for low-impact outages (worse predictions where more people are served).

**Test statistic:** RMSE_high − RMSE_low = **2,101.78 minutes**.

**Significance level:** α = **0.05**.

**Method:** Permutation test (2,000 repetitions) shuffling group labels on the test set while keeping model predictions fixed.

**Results (test set):**
- n(high-impact) = 145, RMSE = **5,230.23 minutes**
- n(low-impact) = 151, RMSE = **3,128.44 minutes**
- **p-value = 0.1179**

**Decision:** Because p > α, we **fail to reject H₀**. There is no statistically significant evidence at the 5% level that RMSE differs between high- and low-impact groups.

**Interpretation:** On paper, the model has a higher error rate for large, high-impact outages off by about 2,102 minutes more than for smaller outages. While this gap is worth keeping an eye on from a policy standpoint, our permutation test shows it isn't statistically significant. Because our sample size is relatively small (around 145 to 151 events per group) and outage lengths vary wildly, the gap could just be due to random noise. Ultimately, we can't prove the model is unfair based on this test alone, but we also can't claim it treats all grid sizes perfectly equally.

<iframe
  src="{{ '/assets/fairness_permutation_null.html' | relative_url }}"
  width="800"
  height="600"
  frameborder="0"
></iframe>
