
# Fraud Pattern Hunt
**Izhar Katariya**

---

## What I Found

### The Main Pattern: Agent-02
Agent-02 made 6 software purchases between July 5-10, each between €3,653-€3,777.
Every other agent averages €104 on software. Agent-02's software average: €2,804.
Z-scores of 62-64. That's not an outlier, that's a signal.

The evasion was simple: rotate merchants. Six different merchants, one per day,
consecutive evenings (5-8pm). No single merchant got hit twice so merchant-level
rules would never fire. Category-normalized scoring catches it immediately.

One normal software purchase (€48.54) came before the inflation window.
Establishing a baseline before the exploit. Classic.

### Agent-03 Did the Same Thing in Advertising
5 transactions, €1,827-€2,222, z-scores 9.5-11.7. Normal advertising avg: €118.
Same evening timing. Different merchants each time. Same playbook, different category.

### Full Picture (24 transactions total, z-score > 5)
- agent-02: 6 in software (€3,653-€3,777)
- agent-03: 5 in advertising (€1,827-€2,222)
- 13 more: scattered across domains, travel, office, hosting, cloud, groceries
- All captured. All different merchants. None flagged by the rules engine.

The pattern is cross-category amount inflation with merchant rotation.
Merchant-level rules are blind to it. Category-normalized z-score is not.

---

## Why LightGBM

Three reasons, all practical:

**Latency.** 20ms budget. LightGBM inference on tabular data is microseconds.
Neural networks add latency for zero accuracy gain on a 5k row dataset.

**Explainability.** Every decision needs to be explainable — to users, merchants,
eventually regulators. Tree SHAP is exact and fast. LIME and integrated gradients
are approximations. Not the same thing.

**Data size.** 5,000 rows, 24 fraud cases, 0.48% fraud rate.
Deep learning would overfit immediately. LightGBM with proper regularization handles this.

If labeled data grows to millions of rows with rich sequential structure,
I'd benchmark TabNet or a temporal model. With evidence, not because it sounds better.

---

## Features — Why Each One Is There

**amount_zscore_vs_category**
The most important feature. Amount normalized against category mean and std.
Agent-02 scores z=62-64. Normal transactions score z=-1.5 to +2.
Raw amount doesn't work — €3,700 means different things in software vs travel.
Category context is what makes the inflation visible.

**amount_vs_agent_avg / amount_vs_category_avg**
How much is this transaction relative to what this agent normally spends
in this category. Catches the deviation from their own baseline.
Computed from past transactions only — no leakage.

**agent_same_category_count**
How many times has this agent spent in this category before.
Agent-02's first software transaction (€48.54) scores low — no history yet.
The inflated ones score higher. Captures the probe-then-exploit sequence.

**agent_avg_amount / agent_tx_count**
Behavioral baseline. 100 transactions averaging €95 then suddenly €3,700 is anomalous.
2 transactions is unknown — model correctly treats new agents as uncertain.

**agent_refund_rate**
Not the primary signal here but important for production.
An agent accumulating refunds should have a decaying trust score.

**hour / day_of_week / is_evening**
Both agent-02 and agent-03 ran their patterns 5-8pm.
Weak signal on its own but meaningful combined with amount inflation.

**Point-in-time correctness**
Every agent feature uses only transactions before the current timestamp.
Enforced in the loop, not assumed.

---

## What I Didn't Do

**Didn't treat AUC as the main metric.** At 0.48% fraud rate a model that predicts
everything as normal gets 99.5% accuracy and decent AUC. I validated by checking
fraud transaction scores directly — all 24 scored >0.999.

**Didn't shuffle before splitting.** Chronological split only.
Random shuffle lets future transactions inform past predictions. That's leakage.

**Didn't label refunded = fraud.** Refunds can be legitimate.
Used z-score > 5 as the threshold. Mislabeling legitimate transactions corrupts
everything downstream.

**Didn't use deep learning.** No evidence it would help.
Complexity needs justification, not the other way around.

---

## Production

The pattern is amount inflation via merchant rotation. In production:

**Feature store with event-time joins.** Agent features must snapshot at transaction
time, not wall-clock time. Know Computer's append-only ledger is exactly the right
foundation for this.

**Shadow mode first.** Run alongside the rules engine for 2 weeks minimum.
Log scores, watch the distribution. Don't touch real decisions until stable.

**PSI monitoring on amount_zscore_vs_category.** If the fraud distribution shifts
(attackers adapt), PSI > 0.2 triggers a retrain. Weekly cadence.

**Threshold is a business decision.** I'd present a precision-recall curve
and let the team choose the false positive rate they can live with.
The model doesn't make that call.

**Model card.** Training date range, fraud rate, feature definitions,
known gaps — new agents with no history score conservatively, which is correct,
but worth documenting.

---

## The Honest Part

These 24 fraud cases were identified by statistical anomaly detection, not ground truth.
In production, Phase 1 is building the labeling pipeline — verified outcomes,
dispute resolutions, confirmed chargebacks.

I wouldn't claim production-ready fraud detection on 24 cases.
I'd claim: the pattern is real, the features are right,
and the pipeline is ready to scale when real labels arrive.
