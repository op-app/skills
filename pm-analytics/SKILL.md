---
name: pm-analytics
description: Recipes for product-management analyses — funnels, retention cohorts, A/B test readouts, feature usage, growth loops, and roadmap prioritization (RICE/ICE). Use when the user asks PM-flavored questions ("how many users converted", "what's our week-1 retention", "did the experiment win", "which feature drove the most engagement", "should we ship X or Y first"). Provides concrete pandas patterns and canvas-assembly guidance the agent can adapt with run_code, create_chart, and create_text.
---

# PM Analytics Patterns

Recipes the agent can adapt for the most common product-management questions. Each section gives a pandas template plus the canvas layout that lands the answer.

## When to load this skill

Triggers on: funnel, conversion, drop-off, retention, cohort, churn, A/B test, experiment, lift, feature adoption, MAU/WAU/DAU, growth, prioritization, RICE, ICE, opportunity sizing, north-star metric.

For deep retention work (curves, aha moments, activation events), also load the `retention-analysis` skill.

## Output principle (read first)

PMs want the **so-what**, not the data dump. Every PM analysis should land as:

1. **One headline number** — text node at the top with the punch-line ("Week-1 retention dropped from 38% to 24% after the Aug 12 release")
2. **One supporting chart** — funnel bars / retention curve / experiment ridgeline
3. **One drilldown table** — the rows behind the chart, so the user can audit and copy values

Group all three with `group="<name of the analysis>"` so they live together on the canvas.

## Funnel analysis

### When
"What's our signup → activation → retention conversion?", "Where do we lose users?", "Did the new onboarding lift step-3 completion?"

### Recipe

```python
# events: long-format with columns [user_id, event_name, ts]
steps = ["visit", "signup", "activate", "convert"]

step_users = {s: events[events.event_name == s].user_id.nunique() for s in steps}

funnel = pd.DataFrame({
    "step": steps,
    "users": [step_users[s] for s in steps],
})
funnel["pct_of_top"] = (funnel.users / funnel.users.iloc[0] * 100).round(1)
funnel["step_conv_pct"] = (funnel.users.pct_change().fillna(0) * 100 + 100).round(1)
# step_conv_pct: % of the previous step who reached this step. The first row
# is 100 by convention.
funnel.loc[0, "step_conv_pct"] = 100.0
```

### Canvas

- **Chart**: horizontal bar (`Plot.barX`), one bar per step, labelled with both `users` and `pct_of_top`. Use `fill: "step"` for distinct colors. Sort by step order (NOT by value — funnels lose meaning when reordered).
- **Text insight**: name the biggest cliff and quantify it.
  > "60% of signups never activate — the activation step is the funnel's biggest cliff (1.2k users dropped here vs 0.4k at every other step combined)."
- **Table**: the funnel DataFrame as-is, so the user can see the raw counts.

### Common gotchas

- **Counted twice** — using row-counts (`.shape[0]`) instead of `nunique()` inflates conversion if users re-trigger events.
- **Cohort drift** — when steps span weeks, mixing today's "visit" cohort with last-month's "convert" cohort biases the conversion rate. Filter to a single cohort window or compute conversion **per signup-week**.
- **Censoring** — users who signed up yesterday haven't had time to convert. Drop the trailing N days where N = your typical convert lag, or annotate the chart.

## Retention cohorts

### When
"What's our week-N retention?", "Has retention improved since the v2 launch?", "Which signup channel retains best?"

### Recipe

```python
# events: [user_id, ts]; users: [user_id, signup_ts, channel]
events["signup_week"] = events.user_id.map(users.set_index("user_id").signup_ts.dt.to_period("W"))
events["weeks_since_signup"] = ((events.ts - events.user_id.map(users.set_index("user_id").signup_ts)).dt.days // 7)

cohort_size = users.groupby(users.signup_ts.dt.to_period("W")).size()

# Active in week N = had at least one event in that week-since-signup
active = (
    events.groupby(["signup_week", "weeks_since_signup"]).user_id.nunique()
    .unstack(fill_value=0)
)

retention_pct = active.div(cohort_size, axis=0).round(3) * 100
# retention_pct: index = signup cohort, columns = weeks since signup, values = % retained
```

### Canvas

- **Chart**: a multi-series line (Plot.lineY with `stroke: "cohort"`) — x is `weeks_since_signup`, y is `retention_pct`, one line per cohort. Recent cohorts on top with stronger color, older cohorts faded — set `stroke` to cohort and let tableau10 pick.
- **Table**: cohort table (rows = signup_week, cols = W0..W12) so the user can scan exact %s.
- **Text insight**: compare two cohorts of interest. "Week-4 retention rose from 18% (Jul cohort) to 26% (Sep cohort) — the new onboarding looks like a real lift, not noise."

### Common gotchas

- **Censoring**, again — the most recent cohort has only week-0 data. Either trim or shade the truncated cells.
- **Defining "active"** — does opening the app count, or only completing a key event? State the definition in the text insight.
- **Tiny cohort denominators** look like noise. Show cohort size alongside or filter out cohorts with <50 users.

## A/B test readouts

### When
"Did the experiment win?", "What's the lift on conversion?", "Is the result significant?"

### Recipe

```python
# events: [user_id, variant, converted]   (variant ∈ {"control", "treatment"})
results = events.groupby("variant").agg(
    n=("user_id", "nunique"),
    converted=("converted", "sum"),
)
results["conv_rate"] = results.converted / results.n

# Two-proportion z-test
from statsmodels.stats.proportion import proportions_ztest
counts = results.converted.values
nobs = results.n.values
zstat, pval = proportions_ztest(counts, nobs)

abs_lift = results.loc["treatment", "conv_rate"] - results.loc["control", "conv_rate"]
rel_lift_pct = abs_lift / results.loc["control", "conv_rate"] * 100
```

### Canvas

- **Headline text**: lead with the verdict, the lift, and the p-value.
  > "Treatment lifted conversion +2.1pp (12.4% → 14.5%, **+17% relative**, p=0.003 with n=18k per arm). Ship it."

  If not significant, say so plainly: "No detectable lift (Δ +0.3pp, p=0.41). Recommend killing the variant or running 3× the volume."
- **Chart**: side-by-side bar of conv_rate per variant with error bars (95% CI). Use `Plot.errorY` if available, or two `Plot.barY` + `Plot.ruleY` for the CI segments.
- **Table**: n, converted, conv_rate, lift_pp, lift_rel, p — one row per variant.

### Common gotchas

- **Multiple testing** — if you ran 5 metrics, treat p<0.05 as p<0.01 (Bonferroni). State which metric was the pre-registered primary.
- **SRM (sample-ratio mismatch)** — if assignment was 50/50 but n_treatment / n_control is 0.92, the bucketing is broken; report the analysis as suspect.
- **Novelty effects** — week-1 lifts often shrink in week-4. If you have 4+ weeks, plot conversion-by-week-since-exposure.

## Feature adoption

### When
"How many users tried feature X?", "Is adoption growing?", "Which features drove engagement?"

### Recipe

```python
# events: [user_id, event_name, ts] for the feature(s) of interest
window_days = 28

active_users = events[events.ts >= events.ts.max() - pd.Timedelta(days=window_days)].user_id.nunique()

# Adoption = users who triggered the feature event ÷ active users in the window
feature_event = "applied_filter"
adopters = events[(events.event_name == feature_event) &
                  (events.ts >= events.ts.max() - pd.Timedelta(days=window_days))].user_id.nunique()
adoption_rate = adopters / active_users

# Stickiness = WAU / MAU among adopters (intent: do they keep using it?)
weekly_users = events[events.event_name == feature_event].set_index("ts").last("7D").user_id.nunique()
monthly_users = events[events.event_name == feature_event].set_index("ts").last("28D").user_id.nunique()
stickiness = weekly_users / monthly_users if monthly_users else 0
```

### Canvas

- **Headline**: "12% of monthly active users used filters in the last 28 days; among adopters stickiness is 42% (they come back weekly)."
- **Chart**: time series of daily adopters, smoothed with 7-day rolling. Annotate launch date with `Plot.ruleX`.
- **Table**: per-feature adoption + stickiness when comparing several features.

## Prioritization (RICE / ICE)

### When
"Should we ship X or Y first?", "Score these initiatives", "Roadmap for Q3"

### Recipe

```python
# initiatives: [name, reach, impact, confidence, effort]
# Reach: users/period reached. Impact: 0.25/0.5/1/2/3 (Minimal..Massive). Confidence: 0..1. Effort: person-months.
initiatives["RICE"] = (initiatives.reach * initiatives.impact * initiatives.confidence) / initiatives.effort
ranked = initiatives.sort_values("RICE", ascending=False).reset_index(drop=True)
ranked["rank"] = ranked.index + 1
```

### Canvas

- **Chart**: horizontal bar of RICE score, sorted descending, fill by initiative name.
- **Table**: full RICE breakdown so stakeholders can debate the inputs (the score is a conversation-starter, not the answer).
- **Text**: "Top 3 by RICE are X, Y, Z — together they cover 80% of the projected reach for ~40% of the engineering effort."

## Cross-cutting tips

- **Never lead with "I built a chart"** — lead with what it shows. The user can see that you built it.
- **One number per headline.** PMs round mercilessly. "12.4% → 14.5%" beats "0.12431 vs 0.14503".
- **Show the cohort, not the average.** "Conversion is 14%" hides that new users convert at 28% and old users at 6%. Break down before reporting.
- **Don't bury the assumption.** If you defined "active" as "any session ≥ 30s", say so in the text node.
- **When the answer is "not enough data", say it.** PMs respect that more than a fabricated trend.
