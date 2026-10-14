# iFood Case Study — Presentation Script (≈ 20 minutes)

**How to use this script**
- The plain text is what you **say**, in the first person. It is written to be read aloud: short sentences, with signposts.
- `[SHOW: … → In [n]]` tells you what to put on screen. **Show `iFood_Case_Study_Code.ipynb`** (code and results only), and read from this script or from `iFood_Case_Study_Solution.ipynb` (same code plus explanations).
  - `In [n]` is the cell number shown on the left of each cell in Jupyter; it is the same in both notebooks.
  - In the code notebook each section starts with a `# ═══ … ═══` comment, so you can also find your place by name.
  - A few cues point to text-only content (TL;DR, Q1e, Q4c design, Q8), which exists only in the explained notebook – the cue says which code cell to stay on.
- **Bold** numbers are the ones worth landing with a short pause.
- Boxes marked **If they ask…** are *not* part of the 20 minutes. Use them only when a question comes up.
- Timing per part is shown in the heading; total ≈ 20–22 min at a calm pace (~145–155 words per minute). If you need to cut, shorten Part 5 (keep the young-kids-vs-teens point) and Part 6 (keep model choice, threshold and drivers).

---

## Part 0 — Opening (≈ 1 min)

`[SHOW: code notebook, In [1] – setup and data load]` *(the TL;DR table itself is text, only in the explained notebook)*

Thank you for the case. I'd like to walk you through how I approached it, the decisions I made along the way, the alternatives I considered, and what the results mean for Marketing.

I'll follow the order of the questions. The thread running through all of them is one simple idea: **every analytical choice should be justified by the data, and every result should be translated into a business decision.**

The answer in one sentence: the last campaign went to everyone and lost money; a simple, well-validated response model could turn that loss into a profit by contacting fewer than one in five customers; and a lot of what the data seems to say only becomes true after fixing the data first.

---

## Part 1 — Framing the business problem (≈ 1 min)

`[SHOW: Q1c — campaign economics table → In [7]]`

Before modelling anything, I wanted to know what success means in money. The dataset has two columns, `Z_CostContact` and `Z_Revenue`, which are constant: 3 for every customer and 11 for every customer.

I could have simply dropped them as useless. That is the first decision, and I chose to **drop them as columns but keep them as business constants**. They match the public iFood case brief exactly: 2,240 customers times 3 is the 6,720 campaign cost, and 334 acceptances times 11 is the 3,674 revenue. So it costs **3 units to contact** a customer, and an acceptance brings **11 units**.

That gives the most important number of the whole case: **a campaign breaks even only if 27.3% of contacted customers accept** — 3 divided by 11. The last campaign converted about 15%, so it **lost about 45% of what it cost**. A 15% response rate sounds healthy; in this business it is a loss. That number frames everything that follows: the goal is not "more responses", it is "contact only the people worth contacting".

> **If they ask: "How sure are you that Z_Revenue is revenue?"**
> The constants reproduce the brief's cost and revenue figures to the unit, which is strong evidence, but it is still an inference, and I would confirm it with the data owner. I also flag that 11 is *revenue*, not margin. With margin data the break-even would be higher. For example, at a 40% margin it would be 3 / 4.4 ≈ 68%.

---

## Part 2 — Data audit and cleaning (≈ 3.5 min)

`[SHOW: Q1 cleaning log → In [15]; duplicates In [4]–[6]; dictionary audit In [9]; dates/income/Recency In [13]; impossible records In [14]]`

I treated Q1 as an audit, not a formality. My principle was: **remove only what is provably wrong, keep what is unusual but possible**, and log every step with row counts.

**Duplicates.** There are **184 exact duplicate rows**, 8% of the file. On top of that I found **19 pairs that are identical on all 38 columns except the target**: one copy says the customer accepted the campaign, the other says they didn't. And 2 pairs differ only in marital status. Two genuinely different customers matching on 36 to 38 columns – income to the unit, enrolment day, every spend category – is practically impossible, so these are copies of the same people.

For the 19 contradicting pairs, I considered four options:
- drop the customer entirely;
- assume they accepted;
- keep both copies;
- or keep one row and mark the label as *unknown*.

I chose **"label unknown"**. Nothing in the data tells me which copy is true, so a missing label is the honest representation. Their spend data is still valid, so they stay in the descriptive analysis, and every analysis of the response simply skips them. Assuming "yes" would have inflated the response rate to about 15.5% with no evidence, and keeping both would put the same person in the training and test sets with opposite answers.

**The data dictionary is wrong in three places**, so I recomputed every pre-computed column from its parts:
- `MntTotal` **excludes gold**, which understates revenue by about 8%;
- `MntRegularProds` produces negative spend;
- `AcceptedCmpOverall` does **not** include the latest campaign – so there is no target leakage, but "fixing" it to match the documentation would create some.

**Dates.** `Customer_Days` spans exactly the 699-day enrolment window of the source data, which means it was counted to **May 2020** – six years after the data was collected. Ages and tenure are about six years too high: comparisons are fine, absolute statements are not.

**Income.** There are **12 customers at exactly 7,500** – a placeholder – and 23 below 10,000. My options were to keep, impute or remove them. I **kept and documented them**: the values are possible, removing them would bias the base towards affluent customers, and imputing would invent data. Re-running the analysis without them doesn't change the conclusions.

**Impossible records.** Six records break hard logical rules, for example spending with zero purchases. I **removed** them rather than flagging or "repairing" them: at least one field in each is certainly wrong, and they are only 0.3% of the base.

Finally, **Recency is perfectly flat** from 0 to 99 days, so the extract was capped or filtered, and we don't know whether it was measured before or after the campaign. I'll come back to why that matters.

**Result:** 2,205 rows became **1,994 unique customers**, 1,975 with a trustworthy label, and **291 responders, 14.7%**.

`[SHOW: stay on In [15]]` *(the Q1e bullets are text, only in the explained notebook)*

My message to Marketing before they trust any dashboard is four points:
1. one record in ten is a duplicate;
2. two of three pre-computed totals are wrong;
3. dates are anchored to 2020;
4. income contains placeholders.

And one number belongs on every campaign dashboard: **27% is break-even**.

> **If they ask: "Why not just drop the 19 conflicting customers?"**
> Dropping them loses valid spend and demographic information for 19 people and gains nothing. Marking the label as unknown gives exactly the same modelling data, without throwing away the descriptive data.
>
> **If they ask: "Isn't removing records cherry-picking?"**
> Only if the rule is "looks strange". My rules were hard logical impossibilities, defined before looking at outcomes:
> - spending with zero purchases;
> - more discounted purchases than purchases in total;
> - spending 35% of two years' income.
>
> The six records also share a corrupted, bot-like pattern: incomes below 10k and often 14–20 web visits a month. All six were non-responders, and the response rate moved only from 14.69% to 14.73%.
>
> **If they ask: "Why does the Recency timing matter?"**
> If Recency was measured *after* the campaign, customers who accepted the offer have a recent purchase *because* they accepted. Recency then partly encodes the outcome (timing leakage). Real recency is right-skewed; a perfectly flat 0–99 distribution suggests an artificial extract, so I'd confirm the snapshot date.

---

## Part 3 — Feature engineering (≈ 1 min)

`[SHOW: Q2 feature checks and summary → In [16]–[17]]`

I built the requested fields and validated each one rather than just creating it. Two decisions matter.

**First, total purchases.** The specification adds discounted purchases to web, catalogue and store purchases. But a discounted purchase still happens through one of those channels, so this double-counts. I implemented it as specified **and** added a channel-only version. The difference is material: the median basket is **23.7 units by the specified formula versus 29.9 by the corrected one**, about 20% apart, which matters later because the free-delivery threshold in Q4c sits exactly at that median. Customers with zero purchases get "undefined" instead of zero, which would fake a low-value customer, or infinity.

**Second, tenure.** Because of the 2020 anchor, tenure "as delivered" is 6 to 8 years for customers who joined in 2012 to 2014. I report tenure **relative to the newest customer** – 0 to 1.9 years – and keep the delivered version for transparency.

I also collapsed the education and marital dummies back into single categories, and checked that they convert back to the original columns exactly.

---

## Part 4 — Exploratory analysis and storytelling (≈ 2 min)

`[SHOW: Q3a histograms + bimodality test → In [18]]`

**Distributions.** Income and age are symmetric. Spend is not: the **mean is 608 but the median is 398**, and on a log scale it is clearly bimodal. A two-component mixture fits much better than one by BIC, giving roughly 42% light buyers and 58% heavy buyers. The practical consequence: "the average customer" doesn't exist, so I report medians and segments and use log-transformed spend in models.

`[SHOW: Q3b Kruskal-Wallis table → In [20]; segment tables In [19]; spend mix In [22]]`

**Who spends?** With 2,000 customers almost every difference is statistically significant, so I ranked the segmentations by **effect size**: a rank-based Kruskal-Wallis test with η². Spend is skewed, so classic ANOVA would be the wrong tool. The answer is very clear:
- **children explain about 26%** of the variation in spend ranks;
- education about 3%;
- marital status essentially nothing.

**Childless households are 28% of customers but 52% of revenue.** On the mix, wine is half of all revenue and meat a quarter, and the small "basic education" group buys a completely different basket, with hardly any wine.

> **If they ask about the product mix in more detail:**
> - Childless households over-index on meat (index 122) and fish (111).
> - Families put a *larger share of a smaller wallet* into wine (112–123) and gold (119–144).
> - PhDs over-index on wine (121).
> - "Basic" education buys fruit, fish, sweets and gold at three times the average share, and wine at only 18% of the average share.
> - Marital status changes the mix by less than ±10%.

`[SHOW: Q3c income vs spend chart → In [24]; robustness without low incomes In [25]]`

**Income versus spend.** The correlation is strong, Spearman **0.86**, but the relationship is convex: about **+25% spend for +10% income**. At the same income, families spend much less. I checked this without the suspicious low incomes, and everything holds except the exact elasticity, which moves from 2.4 to 2.7, so I quote it as a range.

`[SHOW: the three Q3d slides, one by one → In [26], In [27], In [28]]`

For the Head of Marketing I made three slides, each with one message:
1. **Concentration:** the top 20% of customers bring in **52% of revenue**; the bottom half only 9%. Protecting the top fifth matters more than another mass discount.
2. **The children gap:** at the same income, families spend **20 to 72% less**. Income-only targeting overestimates families.
3. **Track record:** the 79% of customers who never accepted an earlier offer respond at **8%**, far below the 27% break-even, while previous acceptors respond at **30 to 82%**. That is the case for stopping mass mailing.

---

## Part 5 — Statistical testing and experimentation (≈ 3.5 min)

`[SHOW: Q4a wine → In [30]; response → In [31]; at equal income → In [32]; per income band (CMH) → In [33]]`

**Claim 1: families buy less wine.** I checked normality first – wine spend is heavily skewed and zero-inflated – so my primary test is **Mann-Whitney**. Because the business cares about averages, I add a Welch t-test and bootstrap confidence intervals as robustness. And I report effect sizes, not just p-values:
- mean wine spend is **232 versus 500, minus 54%**, with a confidence interval of −301 to −235;
- Cliff's delta is −0.44: a random childless customer out-spends a random parent on wine **72% of the time**.

**Claim 2: families respond less.** It is a binary outcome, so I used a chi-square test, with Fisher as a check. Response is **9.6% versus 27.8%**, a relative risk of 0.34 – about three times less likely. Both claims survive a Holm correction for multiple testing.

**But I didn't stop there,** because families earn less, and income drives both spending and responsiveness. So I re-estimated both effects **holding income and age constant**, with two different ways of controlling for income, and I split children into young kids and teenagers. That changed the story in a useful way:
- most of the **wine** gap is really an **income** gap; what remains is driven by **young children**, minus 38 to 52%, while teenagers don't reduce wine spend at all;
- the **responsiveness** gap is driven by **teenage households**, an odds ratio around 0.35 to 0.4, while young children make no difference once income is accounted for.

I also checked the stratified analysis honestly. The effect is **absent in the lowest income quartile**, where everyone responds at about 10%, and strong everywhere else. So I report it per income band rather than as one pooled number.

**The recommendation:** Marketing is right, but "has children" is too blunt. **Young-kid households and teen households are two different audiences.** And since this is observational data, "families respond less" may really mean "families respond less *to the offers we sent*". Only an experiment with family-relevant offers can separate those two.

`[SHOW: Q4b test → In [34]; power table → In [35]; Bayesian view → In [36]]`

**Complainers.** 3 of 19 complainers responded, **15.8% versus 14.7%** – no difference. But the important question is whether this test *could* have found one. It couldn't:
- even if **none** of the 19 had responded, Fisher's p-value would be **0.096**, so the power to detect any reduction is effectively zero;
- the textbook normal approximation claims up to 93% power, which is a nice example of why approximations fail with three expected events;
- to detect a halving we'd need about **150 complainers**.

My advice to the stakeholder: this data cannot answer the question, and no re-analysis will. We need more data – and meanwhile a sensible policy: no promotions to customers with an open complaint.

> **If they ask: "So what would you do to get an answer?"**
> 1. Track complainers' responses over the next campaigns, with a pre-registered test.
> 2. Use all six campaigns per customer as repeated outcomes (a mixed model or GEE).
> 3. Sharpen the question: was the complaint before or after the offer, and was it resolved?
>
> A Bayesian view summarises what we know today: a 34% probability that complainers respond less, with a credible interval from −9 to +23 percentage points.

`[SHOW: Q4c numbers from the data → In [37]]` *(the full design is text, only in the explained notebook)*

**A/B test for "free delivery over 30":**
- Randomise **customers**, stratified by segment and region.
- The primary metric is **contribution margin per customer**, because an offer can sell more and earn less.
- Guardrails: delivery times and "basket padding" just above 30.
- About **6,000 customers per arm**, or 3,000 with CUPED, for at least two full weeks.
- One thing from the data: **half the customers already average more than 30 per order**, so for them the offer is pure cost.

> **If they ask for the full design:**
> - **H0 / H1:** no change vs a change in contribution margin per customer, tested two-sided (an offer can also lose money).
> - **Assignment:** a hash of customer ID plus an experiment salt gives sticky, reproducible assignment; 50/50 split; an A/A test and sample-ratio-mismatch checks first.
> - **Guardrails:** delivery cost per order, delivery time / SLA, cancellations, complaints, unsubscribes, post-test retention.
> - **Duration:** at least two weekly cycles, no peeking (or a sequential design).
> - **Threats:**
>   - interference through shared courier capacity – the control group's deliveries slow down too;
>   - contamination through shared accounts and promo codes;
>   - novelty effects and orders pulled forward;
>   - seasonality.

---

## Part 6 — Predictive model (≈ 5 min)

`[SHOW: Q5 excluded-features table and split → In [38]]`

**Leakage first.** Before choosing a model, I decided what must *not* go into it:
- the constant columns;
- redundant sums;
- the 19 customers with unknown labels.

The duplicates were already removed, which is essential: copies split across training and test inflate every metric.

**Set-up.** A stratified **80/20 split**, with the test set used **once**, at the very end. All preprocessing – logs for skewed money, scaling, one-hot encoding – lives **inside a pipeline**, so it is re-fitted in each cross-validation fold.

`[SHOW: models and search spaces → In [39]]` *(the candidates table is text, only in the explained notebook)*

**Which models, and why these.** I chose candidates that span the main families for tabular response modelling:
- a **baseline** that predicts the average rate, as the floor;
- **logistic regression**, the industry standard for response models, with calibrated probabilities and explainable coefficients;
- **random forest**, for non-linearities and interactions;
- **gradient boosting**, usually the most accurate family on tabular data – scikit-learn's equivalent of XGBoost.

**How I validated:** a grid search for logistic regression and a random search for the trees, each with 5-fold cross-validation. Then **nested cross-validation** for an unbiased comparison, because the tuning score itself is optimistic.

> **If they ask: "Why not other models?"**
> - **SVM:** no native probabilities.
> - **k-NN:** struggles with mixed features and imbalance.
> - **Naive Bayes:** assumes independent features, but the spend categories are strongly correlated.
> - **Neural networks:** 233 positive examples are far too few.
> - **XGBoost:** not in the allowed libraries; HistGradientBoosting is the same algorithm family.
>
> **If they ask: "Why grid search for one and random search for the others?"**
> Logistic regression has three small hyper-parameters, so an exhaustive grid is cheap. The trees have five or six, and random search finds good regions much faster. Nested CV uses 5 outer and 3 inner folds, with paired comparisons on identical folds, and the test set gets bootstrap confidence intervals.

`[SHOW: nested CV table → In [42]; tuning results In [40]]`

**The result is clear.** In nested CV the precision-recall AUC is **0.71 for logistic regression, 0.64 for gradient boosting and 0.60 for the random forest**, against 0.15 for the baseline. Logistic regression wins on **all five** outer folds against both. The simple model wins because the data is small – 233 positives in training – and the signal is mostly monotonic: more previous acceptances and more recent purchases mean more response. The trees spend their flexibility on noise.

`[SHOW: 1-SE rule → In [41]; all models on equal terms → In [47]; profit uncertainty → In [48]; stability table → In [49]]`

**Which logistic regression?** The best cross-validation score came from a lightly regularised model. I applied the **one-standard-error rule** instead: among all settings statistically indistinguishable from the best, take the simplest.
- The result is an **elastic net with about 100 times stronger regularisation** – needed here, because there are only **7 events per parameter** and the spend features are strongly correlated.
- Its coefficients are **about 40% less variable** across bootstrap resamples, with the same accuracy.
- I use no class weights, so the probabilities stay calibrated.

To be transparent: the lighter model, with its own data-driven threshold, earned somewhat more on the test set – **216 versus 191 units** – but the confidence interval runs from minus 30 to plus 87, so the difference is not significant. I chose stability and a transparent rule over a gain that may be noise.

> **If they ask: "What exactly is the final model?"**
> - Logistic regression with an elastic-net penalty: l1_ratio 0.5, C = 0.32 (penalty strength ≈ 3), no class weights.
> - The L1 part set two redundant features to zero (young kids at home, average order value). The L2 part spreads weight across correlated spend features, whose variance inflation factors run up to 24.
> - Events per parameter: 233 / 32 = 7.3, below the rule of thumb of 10 for an unpenalised model – exactly why it is regularised.
> - Versus the best-CV variant: largest coefficient 1.15 vs 1.79, mean bootstrap SD 0.155 vs 0.271, test AP 0.662 vs 0.666. Sign flips are equally rare in both (≈ 7%).
> - On profit: out-of-fold on training, 917 vs 977; on test, +191 vs +216 (95% CI of the difference −30 to +87, so not significant).
>
> **If they ask: "Did you check the assumptions of logistic regression?"** *(all assumption checks: code notebook In [59])*
> - **Linearity of log-odds:** a spline (non-linear) version scores 0.698 vs 0.702, so the linear form is adequate.
> - **Independence:** duplicates were removed.
> - **Low events per parameter and collinearity:** handled by regularisation.
> - **Calibration:** a predicted 26% comes out as 29%; Brier score 0.080 vs 0.125 for the baseline.

`[SHOW: Q5b test metrics → In [43]; accuracy and thresholds → In [44]]`

**Evaluation – and why not accuracy.** With 15% responders, predicting "nobody responds" is **85% accurate** and finds no one. So I used:
- **PR-AUC 0.66** on test, against 0.15 for random;
- **ROC-AUC 0.91**;
- **lift**: the top 10% of scores respond at **4.4 times** the average rate.

> **If they ask: "Why is accuracy wrong beyond the imbalance?"**
> It treats a wasted contact (−3 units) the same as a missed responder (−8 units of profit). PR-AUC focuses on the customers we would actually contact, and the Brier score checks that the probabilities themselves are right, which the 27% rule needs.

`[SHOW: who-to-contact table → In [45]; gains/profit/calibration charts → In [46]]`

**Who to contact.** I compared several rules on the test set:
- **contacting everyone** loses 547 units;
- the **"accepted before" rule of thumb** earns 75;
- a **0.5 threshold** earns 167;
- a **fixed top-20% budget** earns 181;
- my choice, the **break-even threshold of 27.3%**, earns **191**, contacting **18%** of customers and reaching **64% of responders**.

Per thousand customers, that turns **minus 1,385 into plus 484**.

I chose break-even because it comes from the economics, not the data – contact when 11 times the probability exceeds 3 – so it cannot over-fit. It is valid because the model is calibrated: the data-driven optimum lands on the same value, 0.26.

> **If they ask: "When would you use a different rule?"**
> - A **fixed budget** (top X%) when spend is set before the campaign.
> - A **lower threshold** for reach-driven campaigns (brand, retention), where each extra responder has a known marginal cost.
> - A **higher threshold** (e.g. 0.5) when contact fatigue or channel capacity is the constraint.
> - Eventually, an **uplift threshold**.
>
> The break-even rule also updates itself: threshold = cost / value of a response. With a 40% margin it would be 3 / 4.4 ≈ 68%.

`[SHOW: coefficients + grouped permutation importance → In [50]; drivers in business numbers → In [51]]`

**Top drivers.** I used *grouped* permutation importance – correlated columns shuffled together – for two different model families, and they agree almost perfectly. The top three drivers, in observed response rates:
1. **Campaign history:** 8% with no earlier acceptance, 30% with one, 52% with two, 82% with three or more.
2. **Recency:** 29% for customers who bought in the last 20 days versus 6% for 80 to 100 days.
3. **Channel:** heavy catalogue buyers 31% versus 4% for non-catalogue buyers.

In plain English: *contact people who said yes before, bought recently, and shop by catalogue – and let the model's score decide.*

`[SHOW: feature-set variants → In [52]]`

**Feature set.** I kept **all features** (PR-AUC 0.69) and tested the alternatives:
- **Without Recency: 0.62.** That variant is my ready fallback in case Recency turns out to be measured after the campaign.
- **Without demographics: 0.65.** It wouldn't remove the bias anyway, because income and basket mix act as proxies. I'd rather keep them and audit outcomes by group.

> **If they ask about the other feature variants:**
> Without both Recency and demographics: 0.58. Without campaign history: 0.55 – it is the strongest legitimate signal, so brand-new customers need a separate cold-start rule.

`[SHOW: Q5d fairness table → In [53]]`

**Risks before production.**
- **Fairness:** the model selects **34% of childless households but only 8 to 13% of families**. Discounts would flow to the richest customers, who would probably buy anyway.
- **Incrementality:** the model predicts who *accepts*, not who accepts *because of* the offer. That calls for a randomised holdout and uplift modelling.

> **If they ask for more risks:**
> - **Equal opportunity:** among actual responders, the model catches 79% of childless customers but only 50% of families with two or more children.
> - **Feedback loop:** keep 5–10% random contacts so the model keeps learning about customers it would not pick.
> - **Economics:** use margin, including the discount cost, not revenue.
> - **Governance:** GDPR profiling (transparency, opt-out), plus monitoring of calibration and drift.
> - **Small data:** only 291 responders, hence the wide test CIs.

---

## Part 7 — Segmentation (≈ 2 min)

`[SHOW: Q6 scaling options → In [54]; k-selection charts and table → In [55]]`

For segmentation I used only **behaviour**: six spend categories, four purchase channels and web visits. I deliberately left demographics out so I could **validate** the segments against them afterwards. I also left out Recency, because a persona shouldn't change from one day to the next.

**Scaling.** I compared three options and chose **log plus standardisation**, so that going from 10 to 100 counts as much as going from 100 to 1,000, and a few big spenders don't dominate the distances.

`[SHOW: stay on In [55] (the k table under the charts)]`

**How many segments.** I looked at the elbow, silhouette, two other indices, **bootstrap stability**, and two other algorithms as checks. There is no sharp natural number of clusters – silhouette is highest at two, but two is just "light versus heavy buyers". I chose **four**:
- it is stable – refits on resampled data reproduce it, with an adjusted Rand index of 0.87;
- every segment is at least 20% of customers;
- each segment calls for a *different* action.

Three merges two different audiences, and five only splits the low spenders into "very low" and "low" with no new action.

`[SHOW: segment profiles → In [56]; segments in PCA space → In [57]]`

The four personas:
1. **Budget Young Families** – 36% of customers but only 4% of revenue. No paid outreach; use free owned channels and family bundles.
2. **Deal-Seeking Teen Parents** – wine is 70% of their basket and they buy on deal. Deal-led wine offers online, model-selected.
3. **Established Family Foodies** – a third of revenue across all categories. Cross-sell fresh and premium food with loyalty points, not discounts.
4. **Affluent Childless Connoisseurs** – **22% of customers and 47% of revenue**, catalogue-first, almost never buy on deal. A VIP and retention programme, no discounts.

The key insight: **even the best segment responds at 25%, below the 27% break-even**. So no segment is worth contacting in full. **Segments decide the message and the channel; the model decides who gets contacted.**

> **If they ask: "Why does k-means agree only moderately with the Gaussian mixture?"**
> Adjusted Rand 0.59 with the mixture and 0.71 with Ward. Both share the core structure – spend level times family stage – but draw some boundaries differently. That is expected when segments are regions of a continuum rather than separate islands, which is also why I relied on stability and business usefulness rather than any single index.

---

## Part 8 — Making it repeatable: SQL and reporting (≈ 1 min)

`[SHOW: Q7 SQL queries and results → In [58]]`

For the SQL question I wrote two queries: the textbook one, and a production one that de-duplicates, recomputes the total with gold, and builds the education field from the flags. I ran both in SQLite and asserted that the production query matches pandas exactly.

The interesting result: **fixing the data changes the ranking**. On the raw table, Master edges out Graduation on average spend; after the fixes, Graduation is ahead. A weekly report on raw data would have shown the wrong order.

For the weekly report: ingestion into a warehouse, transformations in dbt, scheduled orchestration, and a dashboard showing the response rate **against the 27% break-even**. The key point: **every issue I found in Q1 becomes an automated test that stops the pipeline** before bad data reaches a dashboard.

> **If they ask for details of the pipeline:**
> 1. **Ingestion:** ELT or CDC into a raw layer; campaign contacts and responses stored as time-stamped events, which also fixes the Recency timing question.
> 2. **dbt models:** staging → intermediate → marts; totals always from line items; age and tenure relative to the snapshot date; metric definitions in a semantic layer.
> 3. **Tests as quality gates:**
>    - unique customer ID;
>    - one response per customer and campaign;
>    - totals equal the sum of their parts;
>    - no negative spend;
>    - exactly one education flag per customer;
>    - valid age and income ranges, and heaping alerts;
>    - deal purchases ≤ purchases;
>    - freshness and drift (PSI).
> 4. **Orchestration:** Airflow, Dagster or dbt Cloud weekly, with Slack alerts. The dashboard shows "data as of" and the status of the last quality check.
> 5. **Model monitoring:** PR-AUC and calibration per closed campaign, plus a random holdout.

---

## Part 9 — Recommendations and close (≈ 1 min)

`[SHOW: Q8 – text only; on the code notebook you can end on In [59] (assumption checks) or on the Q3d slides In [26]–[28]]`

To summarise for the CMO:
1. **Stop mass campaigns.** Contact only customers the model rates above the 27% break-even. In testing that turned a loss of **1,385 into a profit of about 480 per thousand customers**.
2. **Build a VIP programme** for affluent customers without children – **22% of customers, 47% of revenue** – with premium selections, not discounts.
3. **Fix the data pipeline.** One record in ten is a duplicate, and two of three pre-computed totals are wrong. Every dashboard is affected.

And three next steps:
- a randomised holdout, to measure true incrementality and move to uplift modelling;
- an experiment with family-relevant offers, to see whether families are really less responsive or just poorly served;
- confirming the snapshot dates, which decides whether Recency can stay in the model.

Throughout, I tried to keep the simplest method that isn't worse, to validate honestly, and to state the uncertainty along with the result. I'm happy to go deeper into any part.

---

# Appendix — Answers to likely questions

**Data & cleaning**
- *"Why 'label unknown' and not majority/most recent record?"* There's no timestamp or ID to tell which record is more recent, and each pair is one-to-one, so there's no majority.
- *"Why keep the 7,500 incomes?"* They are possible values. Removing them biases the sample, and imputing invents data. The robustness check shows the conclusions don't change; only the elasticity estimate moves (2.4 → 2.7), and I quote it as a range.
- *"What would you need from the data owner?"* Three things: a customer ID, the snapshot date (for Age, tenure and Recency timing), and confirmation of the campaign economics, ideally as margin.

**Statistics**
- *"Why Mann-Whitney and not a t-test?"* Spend is skewed and zero-inflated, with Shapiro-Wilk p < 10⁻¹¹ and skewness up to 1.6. I still ran Welch's t-test, because revenue depends on means, and it agrees. Mann-Whitney tests whether one group tends to have higher values; I report that directly as Cliff's δ, so I don't over-claim a "median difference".
- *"Why is chi-square valid for children but not for complaints?"* Chi-square needs an expected count of at least 5 per cell. For children the smallest is 82; for complaints it is 2.8, so I used Fisher's exact test there.
- *"Is the logistic regression in Q4 reliable?"* There are 26 to 97 events per parameter (≥ 10 is the rule of thumb), the controls are not collinear (VIF ≤ 1.3), and the conclusion is identical with a flexible income specification.
- *"Why Holm and not Bonferroni?"* Same guarantee against false alarms, and never less power.
- *"What is CUPED?"* It uses each customer's pre-period spend as a covariate, removing variance that has nothing to do with the treatment. With a correlation of 0.7 it roughly halves the required sample.

**Model**
- *"Why not XGBoost?"* It is not in the allowed libraries. HistGradientBoosting is the same algorithm family, and it lost to logistic regression here.
- *"Why not SMOTE or oversampling?"* PR-AUC measures ranking, which oversampling doesn't improve, and oversampling distorts the probabilities I need for the 27% rule. Class weights were in every search grid; for logistic regression they never won.
- *"Isn't your test set too small?"* It has 58 responders, which is why I report bootstrap confidence intervals (AP 0.55–0.76) and rely primarily on nested CV, which uses all training data. Both agree.
- *"How would you validate in production?"* Temporally: train on past campaigns, test on the next one. Cross-sectional CV is valid for this snapshot, but campaigns are sequential.
- *"What does 1-SE mean exactly?"* Take the best cross-validated score and subtract one standard error. Any setting above that line is statistically as good as the best; choose the simplest of them.
- *"Is the model fair?"* By design it isn't neutral: it selects childless and affluent customers more. That is why I audit outcomes by group, and recommend policy guardrails and uplift modelling before using it to allocate discounts.

**Segmentation**
- *"Why not include income in the clustering?"* To keep the segments behavioural, and to use income as an independent check. The segments line up with income (32k → 76k), which validates them.
- *"How stable are the segments?"* Bootstrap refits reproduce them with an adjusted Rand index of 0.87.

---

# Numbers to remember

| Topic | Number |
|---|---|
| Break-even response rate | **27.3%** (3 / 11) |
| Last campaign | ~15% response, **−45%** of cost |
| Cleaning | 2,205 → **1,994** customers; 184 duplicates; 19 conflicting labels; 6 impossible records |
| Revenue concentration | top 20% = **52%** of revenue; bottom 50% = 9% |
| Children & spend | childless = 28% of customers, **52%** of revenue |
| Children & response | 9.6% vs **27.8%**; driven by **teens** at equal income |
| Children & wine | −54% raw; driven by **young kids** (−38 to −52%) at equal income |
| Complaints | 3/19; even 0/19 → p = 0.096 → **no power** |
| Model | LR (1-SE): nested PR-AUC **0.71**; test ROC-AUC **0.91**, PR-AUC 0.66, lift 4.4× |
| Business impact | **−1,385 → +484** per 1,000 customers |
| Top drivers | past acceptance (8% → 82%), recency (29% vs 6%), catalogue (31% vs 4%) |
| VIP segment | **22%** of customers, **47%** of revenue |
