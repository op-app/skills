---
name: retention-analysis
description: Reference material for retention, aha moment, and user engagement analysis. Use when users ask about retention curves, churn, activation, onboarding, aha moments, or which behaviors predict long-term engagement. Provides analysis patterns the agent can adapt with run_code and create_cell.
---

# Retention & Aha Moment Analysis

Reference patterns for analyzing user retention and identifying behaviors that predict engagement. Adapt these to the user's data — don't follow them rigidly.

## Core Concepts

**Retention**: Did a user come back after N days? Measured as: user's last activity >= N days after their first activity.

**Aha moment**: A behavior performed early (in the "activation window") that correlates with long-term retention. Classic examples: Facebook's "7 friends in 10 days", Slack's "2000 messages sent".

**Cohort**: Users grouped by when they first appeared (signup week/month). Comparing retention across cohorts shows whether the product is improving.

## Typical Input Data

User event data with at minimum:
- User identifier (user_id, customer_id, account_id, etc.)
- Event/action name (event_name, event, action, activity, etc.)
- Timestamp (timestamp, created_at, time, date, etc.)

Always inspect the data first with `run_code` — check column names, date formats, event types, and user counts before writing analysis code.

## Analysis Patterns

### 1. Retention Curve

Calculate what percentage of users return after day 1, 7, 14, 30, etc.

```python
import pandas as pd

# Parse timestamps, get first/last per user
events['timestamp'] = pd.to_datetime(events['timestamp'])
user_first = events.groupby('user_id')['timestamp'].min().rename('first_seen')
user_last = events.groupby('user_id')['timestamp'].max().rename('last_seen')
users = pd.concat([user_first, user_last], axis=1).reset_index()
users['lifespan_days'] = (users['last_seen'] - users['first_seen']).dt.total_seconds() / 86400

data_end = events['timestamp'].max()

# For each retention day, only count users who've had enough time
retention_days = [1, 3, 7, 14, 30]
curve = []
for day in retention_days:
    eligible = users[(data_end - users['first_seen']).dt.total_seconds() / 86400 >= day]
    if eligible.empty:
        continue
    rate = (eligible['lifespan_days'] >= day).mean()
    curve.append({'day': day, 'retention_rate': round(rate, 4), 'eligible_users': len(eligible)})

retention_curve = pd.DataFrame(curve)
```

Key detail: only include users who've had enough calendar time to possibly be retained (eligible filter).

### 2. Cohort Retention

Group users by signup period and compare retention across cohorts.

```python
users['cohort'] = users['first_seen'].dt.to_period('W').dt.to_timestamp()
users['retained'] = users['lifespan_days'] >= 14  # pick your retention threshold

cohort_summary = users.groupby('cohort').agg(
    total_users=('user_id', 'count'),
    retained_users=('retained', 'sum'),
).reset_index()
cohort_summary['retention_rate'] = cohort_summary['retained_users'] / cohort_summary['total_users']
```

### 3. Behavior-Retention Correlation (Aha Moment)

For each behavior, compare retention of users who did it early vs those who didn't.

```python
# Define activation window and retention threshold
ACTIVATION_DAYS = 7
RETENTION_DAY = 14

# Get eligible users (enough time to measure retention)
users['eligible'] = (data_end - users['first_seen']).dt.total_seconds() / 86400 >= RETENTION_DAY
eligible = users[users['eligible']].copy()
eligible_ids = set(eligible['user_id'])
retained_ids = set(eligible[eligible['retained']]['user_id'])

# Get events in activation window
activation = events[events['user_id'].isin(eligible_ids)].merge(
    eligible[['user_id', 'first_seen']], on='user_id'
)
activation['days_since_first'] = (activation['timestamp'] - activation['first_seen']).dt.total_seconds() / 86400
activation = activation[activation['days_since_first'] <= ACTIVATION_DAYS]

# Per-behavior correlation
behavior_users = activation.groupby('event_name')['user_id'].apply(set).to_dict()
total = len(eligible_ids)

results = []
for behavior, users_with in behavior_users.items():
    n_with = len(users_with)
    n_without = total - n_with
    if n_with < 5 or n_without < 5:
        continue
    retained_with = len(users_with & retained_ids)
    retained_without = len(retained_ids) - retained_with
    rate_with = retained_with / n_with
    rate_without = retained_without / n_without
    results.append({
        'behavior': behavior,
        'users_with': n_with,
        'users_without': n_without,
        'retention_with': round(rate_with, 4),
        'retention_without': round(rate_without, 4),
        'lift': round(rate_with - rate_without, 4),
    })

correlations = pd.DataFrame(results).sort_values('lift', ascending=False)
```

### 4. Statistical Significance (Chi-Squared)

Test whether a behavior's retention difference is statistically significant.

```python
from scipy.stats import chi2_contingency

# For each behavior, build a 2x2 contingency table:
#              Retained   Not retained
# With behavior    a           b
# Without          c           d

for _, row in correlations.iterrows():
    a = row['retained_with'] if 'retained_with' in row else int(row['retention_with'] * row['users_with'])
    b = row['users_with'] - a
    c = row['retained_without'] if 'retained_without' in row else int(row['retention_without'] * row['users_without'])
    d = row['users_without'] - c
    chi2, p_value, _, _ = chi2_contingency([[a, b], [c, d]], correction=True)
    # p_value < 0.05 = statistically significant
```

If scipy isn't available, a manual chi-squared for 2x2 tables with Yates' correction:

```python
import numpy as np

def chi2_2x2(a, b, c, d):
    n = a + b + c + d
    if n == 0 or (a+b) == 0 or (c+d) == 0 or (a+c) == 0 or (b+d) == 0:
        return 0.0, 1.0
    chi2 = (n * (abs(a*d - b*c) - n/2)**2) / ((a+b)*(c+d)*(a+c)*(b+d))
    # p-value approximation for 1 df
    z = np.sqrt(chi2)
    t = 1.0 / (1.0 + 0.2316419 * z)
    p = t * (0.319381530 + t * (-0.356563782 + t * (1.781477937 + t * (-1.821255978 + t * 1.330274429))))
    p_value = 2 * p * np.exp(-0.5 * z * z) / np.sqrt(2 * np.pi)
    return chi2, max(0, min(1, p_value))
```

## Presenting Results

When putting results on canvas:
- **Retention curve**: Line chart — day on x-axis, retention rate on y-axis
- **Behavior correlations**: Horizontal bar chart — behavior on y-axis, retention lift on x-axis. Use `run_code` to check column names before writing chart code.
- **Cohort trends**: Line or bar chart by cohort period
- **Key findings**: Short text annotations near the related charts explaining what the data shows
- **Data tables**: Show the full correlation data so users can explore

## Common Pitfalls

- **Not filtering eligible users**: If a user signed up yesterday, they can't have 30-day retention. Always filter by calendar time.
- **Correlation vs causation**: A behavior correlating with retention doesn't mean it causes retention. Users who do more things in general are more likely to retain.
- **Small sample sizes**: Don't draw conclusions from behaviors performed by only a handful of users. Require minimum group sizes.
- **Timezone issues**: Parse timestamps consistently. Use `pd.to_datetime(col, utc=True)` when timezone-aware.
