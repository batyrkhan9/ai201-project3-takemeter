# TakeMeter – Planning Document
**Community:** r/worldcup (FIFA World Cup Reddit community)
**Task:** Classify World Cup posts into discourse quality categories

---

## 1. Community Choice

I chose the FIFA World Cup Reddit community (r/worldcup) as my target community. This community is active during tournaments with thousands of posts and comments per match, generating a rich and highly varied corpus of discourse. The community includes fans from every country in the world, which produces a wide spectrum of post quality — from rigorous tactical and statistical analysis to pure emotional reactions to bold, unsupported opinions. This variance is exactly what a classification task needs: if every post were the same type of discourse, labeling would be trivial and a classifier would learn nothing meaningful.

The World Cup context is also particularly suited to this task because the discourse maps naturally onto three functionally distinct modes that the community itself implicitly recognizes: posts that argue something with evidence, posts that assert something without evidence, and posts that express a feeling in response to a specific event. These distinctions are not invented — they reflect how participants in the community actually evaluate each other's contributions ("that's just a hot take," "do you have stats for that?").

---

## 2. Label Taxonomy

### Label 1: `analysis`
**Definition:** The post makes a structured argument supported by specific, verifiable evidence — statistics, tactical observations, historical comparisons, or formation breakdowns. The claim is grounded and could be disputed or supported with additional data.

**Example 1:**
> "France's 4-3-3 has been consistently vulnerable on the left side of defense. Their fullback is being isolated in wide areas and teams with pace on the right wing are exploiting this."

**Example 2:**
> "Argentina's pressing intensity drops significantly after the 70th minute — their PPDA goes from 8.2 in the first half to 13.7 in the second, which explains why they concede late."

---

### Label 2: `hot_take`
**Definition:** A bold, confident opinion stated without supporting evidence. The post asserts rather than argues. The claim may be plausible but is not backed by data or structured reasoning — often provocative or contrarian in framing.

**Example 1:**
> "Mbappe is a fraud in knockout rounds. Stats don't lie — he vanishes every single time."

**Example 2:**
> "Germany is a finished footballing nation. That era is permanently over."

---

### Label 3: `reaction`
**Definition:** An immediate emotional response to a specific match event, result, or tournament moment. The post expresses a feeling rather than making an argument. Little to no reasoning is present — the post is about how the author feels right now.

**Example 1:**
> "I can't stop crying. Four years of waiting for this and it ends like that."

**Example 2:**
> "GOOOOOL! Are you kidding me?! What a strike from outside the box!"

---

## 3. Hard Edge Cases

### Primary edge case: `hot_take` vs `analysis`

The most difficult boundary is between a hot take that includes one supporting statistic and genuine analysis. Consider:

> "Mbappe is overrated — his xG in knockout rounds is 0.3 per game."

This could be `hot_take` (bold accusatory claim) or `analysis` (cites a specific metric). 

**Decision rule:** If removing the opinion framing leaves a genuine evidenced argument — one where the evidence would persuade a neutral reader without the emotional charge — label it `analysis`. If the evidence is cherry-picked, decorative, or selected to sound credible rather than to genuinely reason, label it `hot_take`. The one-stat accusatory post above is `hot_take`: the framing drives the conclusion, not the evidence.

### Difficult annotation examples from actual dataset:

**Example 1:**
> "Africa deserves more World Cup spots. The continent produces elite talent but gets limited representation relative to Europe."

This could be `hot_take` (bold claim about a policy issue) or `analysis` (implicitly referencing structural imbalance). **Decision:** `hot_take` — no specific data is cited; the claim is asserted rather than argued.

**Example 2:**
> "Penalty shootouts should be replaced with golden goal extra time. More football, more fair."

This is `hot_take` — an opinion about the rules stated with no supporting evidence or comparative analysis.

**Example 3:**
> "That tactical switch in the second half completely changed the match. Brilliant management."

This could be `reaction` (expressing admiration) or `analysis` (identifying a tactical observation). **Decision:** `reaction` — no mechanism is explained; it's an emotional response to the substitution's outcome rather than a structured observation about why it worked.

---

## 4. Data Collection Plan

**Source:** Posts and comments manually curated from r/worldcup on Reddit (public posts, no authentication required). Posts were drawn from match threads, post-match discussions, and general tournament discussion threads across the group stage and knockout rounds.

**Target per label:** ~67 examples per label (for approximately balanced distribution across 200 total examples)

**If a label is underrepresented:** Collect additional examples specifically targeting that label. For `analysis`, this meant specifically searching for tactical discussion threads and post-match stat summaries. For `hot_take`, looking at general opinion threads. For `reaction`, looking at live match threads.

**Final distribution:**
- `reaction`: 75 (37.5%)
- `hot_take`: 70 (35.0%)
- `analysis`: 55 (27.5%)
- **Total: 200**

No label exceeds 70%, so class imbalance is not a concern at this distribution.

---

## 5. Evaluation Metrics

**Primary metric: per-class F1 score**
Accuracy alone is insufficient because it can be inflated by a majority class. If 38% of examples are `reaction`, a model that predicts `reaction` for everything achieves 38% accuracy without learning anything. Per-class F1 captures both precision (how many predicted positives are correct) and recall (how many actual positives were found), giving a complete picture of performance for each label.

**Secondary metric: macro-averaged F1**
This averages F1 across all classes equally, regardless of class size. It is the right summary metric for this task because all three discourse categories are equally important — there is no "dominant" class we care more about predicting correctly.

**Supplementary: confusion matrix**
Reveals directional error patterns — which specific pairs of labels the model confuses and in which direction. This is essential for diagnosing whether label boundaries are unclear.

**Why not accuracy alone:** Accuracy rewards majority-class prediction and obscures per-class failures. A model that never predicts `analysis` would still score 72.5% accuracy on this dataset — which would be misleadingly high.

---

## 6. Definition of Success

**Minimum threshold:** Fine-tuned model must exceed the zero-shot Groq baseline by at least 5 percentage points on macro-averaged F1. If fine-tuning adds no value, the labels may be too easy (solvable by a general LLM without training) or too noisy (no learnable signal).

**Target for genuine usefulness:** Macro F1 ≥ 0.70 across all three classes. A community moderation tool using this classifier would need at minimum 70% per-class F1 to be useful — below that, the false positive rate for any single class becomes high enough to annoy users.

**Class-specific floor:** No individual class F1 below 0.55. A class with F1 below that threshold indicates the model has failed to learn that boundary and the classifier is unreliable for that category.

**Failure condition:** If the fine-tuned model performs worse than the zero-shot baseline, this indicates a training bug, label leakage, or fundamental data quality issue that must be investigated before submission.

---

## 7. AI Tool Plan

### Label stress-testing
Before finalizing the taxonomy, I used an LLM to generate 10 borderline posts sitting at the `hot_take`/`analysis` boundary. Several generated examples initially seemed like `analysis` but had no verifiable mechanism — just an assertion with a number attached. This led to the decision rule documented in Section 3: evidence must be independently persuasive without the opinion framing to qualify as `analysis`.

### Annotation assistance
For a subset of examples (approximately 40 posts), I used an LLM with the label definitions from Section 2 to pre-label before reviewing. Every pre-assigned label was reviewed and corrected where needed. Pre-labeling was fastest for clear `reaction` examples (high agreement) and least reliable for `hot_take`/`analysis` boundary cases (required manual override in ~30% of cases).

### Failure analysis
After generating wrong predictions from the fine-tuned model, I used an LLM to identify common patterns in the misclassified examples. The AI identified that short posts were disproportionately misclassified (consistent with the expected difficulty of distinguishing `hot_take` from `reaction` when posts lack enough context). I verified this pattern manually by reviewing post lengths for correct vs. incorrect predictions.

---

## Stretch Features

*(None attempted due to time constraints.)*
