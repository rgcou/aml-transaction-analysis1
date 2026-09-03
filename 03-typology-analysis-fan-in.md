# Part III — Typology Analysis: Fan-In

> The full code for this phase is in the accompanying notebook. The complete write-up, including all charts and output screenshots, is in **III Typology Analysis.pdf** in this repository. This document explains the reasoning, the decisions and the findings.

## What I do in this analysis

**Step 1 — Degree.** I count how many different accounts send money to each receiving account, and keep only those receiving from 3 or more. This is the minimum degree that can be a fan-in.

**Step 2 — Time window.** I look at the gaps between incoming transactions to find the time scale of a "burst".

**Step 3 — Selecting a threshold for the model.** I use the accounts with the lowest time dispersion to measure how long their bursts last, and take that as my threshold.

**Step 4 — Clustering.** I run a DBSCAN model on each receiving account, using that threshold, so the model groups transactions that arrived close together in time. Each group is a candidate burst.

**Step 5 — Counting senders per burst.** For every cluster I count how many different senders it has, how long it lasted, and how many transactions it holds. A cluster is a fan-in candidate if it has 3 or more distinct senders.

**Step 6 — Conclusion.** I look at what my definition selected, and compare it with how AMLSim defines fan-in in its Patterns file. The two criteria are not the same, and explaining why they differ is the main finding of this work.

---

## 1. Degree and Definition

### 1.1. Introduction

Fan-in is a money laundering typology where one account receives money from many different accounts. But there is more than one way to define it precisely, and the definition I choose shapes everything I detect.

The literature offers two main definitions. The **structural** one (used by AMLSim, the simulator that generated this dataset) defines fan-in purely by shape: one account connected to k ≥ 2 different sending accounts, with no condition on time. The **temporal** one (used in detection work like MonLAD) adds speed: many sources sending into one account within a short time window, because launderers move money fast to reduce risk.

I choose the temporal definition. My reasoning is that a burst of many transactions from many accounts in a few minutes is more anomalous, and easier to separate from normal activity, than the same transactions spread across weeks. So throughout this work, "fan-in" means: **an account receiving money from 3 or more different accounts, concentrated in a short time window.**

I set the degree at 3 or more because receiving from 1 account cannot be fan-in by definition, and receiving from 2 is a weaker case I set aside (explained below). The time window is not fixed in advance: I derive it from the data itself in the sections that follow.

### 1.2. Why the time window matters in practice

For a bank or a fintech, the difference between the two definitions is not academic. A purely structural rule flags every account that receives from many sources, which includes shops, payroll accounts, and platforms collecting payments — a volume of alerts no compliance team can review. Adding a time condition targets something more specific: money that arrives from many sources and is moved fast.

The reason this could matter is that speed is part of the launderer's strategy, not an accident. Funds are split into small amounts across many transfers to stay under detection thresholds, and completed quickly because the sooner the money moves, the lower the risk of being caught. A detection rule built around short bursts is therefore aimed at the behaviour itself, not just at the shape of the network.

### 1.3. Obtaining a 3-degree subset

First I check how many different accounts each `to_account` is receiving transactions from. There are 576,176 receiving accounts out of the 712,688 accounts in the dataset.

Their composition by number of distinct senders:

| Distinct senders | Receiving accounts |
|---|---|
| 1 | 223,040 |
| 2 | 121,290 |
| 3 | 138,043 |
| … | … |
| 10 | 631 |

For my analysis I filter out the 344,330 / 576,176 accounts (almost 60%) that receive transactions from just 1 or 2 accounts, leaving **231,846 accounts**.

Justifying leaving out degree-2 fan-in is more complex. These accounts represent approximately 21% of receiving accounts. Money laundering with money coming from 2 different accounts into many different accounts could be possible, but it requires a lot of effort, making it less probable. Even if it is possible, I think it is more cost-effective to focus first on accounts that receive money from more than 2 accounts, setting that subset aside for separate study, as it would need more crossed information to be analysed and reach a meaningful conclusion.

Filtering the original dataframe accordingly leaves **5,318,124 / 6,924,049 transactions** (almost 77% of all transactions). This subset is what I call `focusGroup`.

---

## 2. Time Window: Timestamps and Time Gaps

### 2.1. Creating a time table

The total time window of `focusGroup` is approximately 16 days, distributed as follows:

| Cumulative | Date |
|---|---|
| 25% | Until September 2nd |
| 50% | Until September 5th |
| 75% | Until September 8th |
| 100% | Until September 17th |

Transactions are more concentrated (75%) in the first half of the studied time window, where they are evenly distributed.

But what matters here is not *when* transactions were made, but rather how time-concentrated they are with respect to a receiving account. To examine that, I need the time gap between transactions sent to each receiving account.

I create a table with the receiving account, the timestamp of each incoming transaction, sorted, plus a column translating timestamps to minutes.

### 2.2. Calculating the gap between transactions

I then calculate the difference between each timestamp within the same account, in minutes:

| Statistic | Value |
|---|---|
| Mean | 564.41 min (≈ 9.4 hours) |
| Min | 0.0 min |
| 25% | 6 min |
| 50% | 22 min |
| 75% | 854 min (≈ 14 hours) |
| Max | 22,856 min (≈ 16 days) |

A minimum of 0.0 means at least one transaction was transferred in less than a minute.

---

## 3. Finding an Epsilon for Temporal Concentration

### 3.1. Three-stage approach

I started by thinking about which measure could tell me whether the incoming transactions to an account were concentrated in time.

Once I had created the `gap` attribute, I first considered building a table with one row per receiving account, containing dispersion measures of the gaps: range, standard deviation and percentiles.

**I discarded the percentile** after thinking about what it actually measures. The percentile orders the gaps by size, not by time. So if an account received two transactions close together, then a long pause, then two more close together, and so on across the month, the lower percentiles would show short gaps, but there was no burst — just recurring activity. The percentile tells me how short the short gaps are, but not whether they happened one after another.

I also thought that **standard deviation** is sensitive to extreme values. But I concluded it is still useful in one of its extremes: if dispersion is low, I know the transactions were concentrated, without needing anything else. The problem is with accounts where dispersion is high, because there I cannot tell whether there was a burst plus an isolated late transaction, or genuinely spread-out activity.

So I decided on a three-stage approach:

1. Use dispersion to identify the clearly concentrated accounts.
2. Try to identify a time pattern in this subset.
3. Use this information to build a clustering (DBSCAN) model to identify fan-in transactions.

DBSCAN needs a parameter (`eps`) that defines how close two transactions must be to belong to the same episode or cluster. I decided to use the accounts with low dispersion: in those accounts the concentration is visible, so I can measure how long their bursts typically last and use that as `eps`.

### 3.2. Timestamp dispersion for each receiving account

I create a table with the standard deviation of the timestamps (in minutes) for every receiving account:

| Statistic | Value |
|---|---|
| Mean | ≈ 4,441 min (≈ 3 days) |
| Min | ≈ 0.6 min |
| 25% | ≈ 4,276 min (≈ 3 days) |
| 50% | ≈ 4,460 min (≈ 3 days) |
| 75% | ≈ 4,671 min (≈ 3.25 days) |
| Max | ≈ 13,522 min (≈ 9.4 days) |

This shows a really low minimum and a very high maximum, but most values are close to the mean (75% of values are under ≈ 4,672 min) and the quartiles are not far from each other. So this is not giving me the information I was looking for, and I look at lower percentiles instead.

Even at 1% the value is 553 minutes (≈ 9.2 hours), so I isolate that 1% to look at it closer:

| Statistic | Value |
|---|---|
| Mean | ≈ 210 min (≈ 3.5 hours) |
| Min | ≈ 0.6 min |
| 25% | ≈ 9.7 min |
| 50% | ≈ 187 min (≈ 3 hours) |
| 75% | ≈ 385 min (≈ 6.4 hours) |
| Max | ≈ 553 min (≈ 9 hours) |

There is a relevant difference between percentiles 25 and 50 — 9.7 is ≈ 0.05 of 187 — so the threshold I am looking for might be between them.

Comparing the ratio each percentile has with respect to the next smaller one, there is an **eight-fold jump between percentiles 0.4 and 0.3**, which is between 4 and 8 times higher than the other visible jumps. I therefore filter the accounts belonging to percentile 0.4 of the timestamp standard deviation — that is, the 0.4% most time-concentrated receiving accounts.

### 3.3. Choosing a candidate

Looking at the gaps within this subset (`focusGroup_2`), **90% of the gaps are under six minutes**, before jumping to almost double (11). Six minutes is then a good first candidate for the epsilon value of the DBSCAN model.

---

## 4. Creating a DBSCAN Model

### 4.1. First trial

I run DBSCAN with `eps = 6` on a single row first. The result is `[0,0,0,0]`, meaning the model grouped those transactions inside the same cluster.

### 4.2. Creating a function

I then create a function applied to every group.

The function takes a `group` argument, which is every `to_account` group — that is, all the transactions made to each receiving account. Inside it, a variable `X` reshapes every group from a 1D array to a 2D array (a matrix of n rows and one column); this is needed because DBSCAN works with arrays of more than one dimension and every group is naturally 1D.

The function returns a Pandas Series built with `pd.Series()`, which needs the series values and the index labels correlated to each value: the values are the cluster each transaction belongs to, associated to an index corresponding to its position within the group. Finally, the result applies the function to each group of transactions made to a single receiving account.

### 4.3. Is eps = 6 working?

98% of transactions ended up inside the first cluster of their group.

This means almost every transaction is inside a cluster — and not every transaction of a receiving account should be, because the clusters were built precisely to detect anomalously concentrated transactions. Having 98% inside the first cluster of every group is not what I was looking for, because money laundering should be anomalous in the time axis too.

This is explained by the **chaining effect** in DBSCAN: with a 6-minute radius, consecutive transactions link into one another and generate a single large cluster per group.

### 4.4. Revisiting the epsilon: from 6 to 2 minutes

The result above told me `eps = 6` was too permissive, so I went back to the percentile table that produced the original candidate.

Six minutes came from percentile 90 of the gaps in the most concentrated accounts (`focusGroup_2`). That is a wide cut: it accepts almost every gap in a subset that is already, by construction, the most time-concentrated in the dataset. Looking further down the same distribution, **percentile 70 sits at 2 minutes**.

I chose 2 minutes for two reasons. First, it comes from the same empirical source as the original candidate, so it is a stricter cut of the same distribution rather than an arbitrary number. Second, at this scale a burst means transactions arriving essentially back to back, which is closer to the behaviour I set out to detect in section 1.

With `eps = 2` the model produced five clustering groups and left **134,694 transactions outside any cluster**, which is much closer to what I was looking for: clusters that isolate concentrated episodes rather than absorbing all of an account's activity.

---

## 5. Cluster Analysis

I add the sending account attribute to the time table, so I can count the number of different sending accounts per cluster.

I then create a table (`cluster_summary`) grouping by receiving account and cluster, containing how many different senders there are per cluster, the cluster's time window, and how many transactions it holds. After that I filter to clusters with more than two senders, since I am identifying degree-3 fan-in.

There are **241,152 clusters**. Describing the table:

| Metric | Min | Mean | p75 | Max |
|---|---|---|---|---|
| Distinct senders | 3 | — | 4 | 785 |
| Duration (minutes) | 0 | ≈ 5 | ≈ 6 | 15 |
| Transactions | — | ≈ 14 | 17 | — |

The sender count goes from 3 to 785, which matches the criterion I chose, but p75 is 4 — so most clusters come from 3 or 4 different accounts, with outliers between p75 and p100.

The duration goes from 0 to 15 minutes, which is precisely what I wanted the DBSCAN model to identify: bursts of several transactions within a few minutes. Most clusters last 6 minutes or less.

In short, **most suspected accounts are receiving between 3 and 17 transactions, within 0 to 7 minutes, from 3 to 4 senders.**

---

## 6. Conclusion

This work defined fan-in as a burst: many different accounts sending money into one account within a short time window. Using this definition, I identified **211,099 receiving accounts out of the 231,846** that had 3 or more distinct senders — that is, **91%**. The time criterion filtered out less than 10% of the candidates.

In this dataset, receiving from several accounts within a short window is not rare: it is the normal case. Short bursts turned out to be common rather than exceptional, so the time window alone did not reduce the candidate set much. In a real institution the same rule would need to be combined with other signals — transaction amounts, customer profile, or the account's usual behaviour — before it could be used to raise alerts. This part of the work, though, has been focused on fan-in behaviour, not on every other signal worth looking at.

### Comparison with AMLSim's own labels

The dataset comes with a Patterns file listing the laundering structures the simulator planted. It contains **12 fan-in blocks**. That number is very far from mine, but the two are not measuring the same thing, and it is worth explaining why.

AMLSim defines fan-in structurally: one account connected to k ≥ 2 sending accounts, with no condition on time. Its blocks confirm this: the transactions in each one are spread across several days, not concentrated in minutes. My definition adds a temporal condition that theirs does not have. So the two criteria select different sets by construction, and neither one is the correct version of the other.

There is a second and more important difference worth mentioning.

In AMLSim, fan-in exists **both as a normal pattern and as a laundering pattern, with the same shape**. A shop receiving payments from many customers looks structurally identical to a laundering fan-in. The 12 blocks are the ones the simulator deliberately planted and labelled; the rest of the fan-in shapes in the data are ordinary activity. The difference between them is not in the transactions but in a label assigned when the data was generated.

This is what makes the structural approach reach its limit. My model identifies the fan-in shape combined with a temporal dimension, and it does so correctly. What no shape-based rule can do — neither mine nor a purely structural one — is separate laundering fan-ins from legitimate ones, because they share the same form. Doing that would require information beyond topology and timing: amounts, currencies, or tracing where the funds came from.

This is the core difficulty of typology-based detection in real anti-money-laundering work.
