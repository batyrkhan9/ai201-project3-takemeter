# TakeMeter: World Cup Discourse Classifier

A fine-tuned text classifier that evaluates discourse quality in the FIFA World Cup Reddit community (r/worldcup), distinguishing between analytical posts, hot takes, and emotional reactions.

---

## Community Choice

I chose the FIFA World Cup subreddit (r/worldcup) as the target community. World Cup discourse is ideal for this task because it spans a genuine and wide spectrum of post quality within a single shared topic. The same match generates tactical breakdowns citing xG and pressing metrics alongside pure emotional outbursts and bold, unsupported opinions. Critically, these distinctions are ones the community itself recognizes — participants regularly call out "that's just a hot take" or ask "do you have stats for that?" This means the labels are grounded in community norms rather than imposed externally.

---

## Label Taxonomy

### `analysis`
The post makes a structured argument supported by specific, verifiable evidence — statistics, tactical observations, historical comparisons, or formation breakdowns. The claim could be independently evaluated or disputed with data.

**Example 1:**
> "Argentina's pressing intensity drops significantly after the 70th minute — their PPDA goes from 8.2 in the first half to 13.7 in the second, which explains why they concede late."

**Example 2:**
> "France's 4-3-3 has been consistently vulnerable on the left side of defense. Their fullback is being isolated in wide areas and teams with pace on the right wing are exploiting this."

---

### `hot_take`
A bold, confident opinion stated without supporting evidence. The post asserts rather than argues — often provocative or contrarian in framing.

**Example 1:**
> "Mbappe is a fraud in knockout rounds. Stats don't lie — he vanishes every single time."

**Example 2:**
> "Germany is a finished footballing nation. That era is permanently over."

---

### `reaction`
An immediate emotional response to a specific match event, result, or tournament moment. The post expresses a feeling rather than an argument.

**Example 1:**
> "I can't stop crying. Four years of waiting for this and it ends like that."

**Example 2:**
> "GOOOOOL! Are you kidding me?! What a strike from outside the box!"

---

## Data Collection

**Source:** Posts and comments curated from r/worldcup on Reddit, drawn from match threads, post-match discussion threads, and general tournament opinion threads. All content was from public posts requiring no authentication.

**Labeling process:** Each post was read individually and assigned a label using the definitions above. Approximately 40 examples were pre-labeled using an LLM provided with the label definitions, then every pre-assigned label was reviewed and corrected manually.

**Label distribution (200 total examples):**

| Label | Count | Percentage |
|---|---|---|
| `reaction` | 75 | 37.5% |
| `hot_take` | 70 | 35.0% |
| `analysis` | 55 | 27.5% |
| **Total** | **200** | **100%** |

**Train / validation / test split (70% / 15% / 15%):**
- Train: 140 examples
- Validation: 30 examples
- Test: 30 examples

---

### Difficult-to-Label Examples

**Example 1:**
> "Africa deserves more World Cup spots. The continent produces elite talent but gets limited representation relative to Europe."

Could be `hot_take` (policy opinion with no data) or `analysis` (implicit structural argument). **Decision: `hot_take`** — no specific data is cited; the claim is asserted rather than argued.

**Example 2:**
> "That tactical switch in the second half completely changed the match. Brilliant management."

Could be `reaction` (emotional admiration) or `analysis` (tactical observation). **Decision: `reaction`** — no mechanism is explained. The post responds to the outcome rather than reasoning about why it worked.

**Example 3:**
> "Mbappe is overrated — his xG in knockout rounds is 0.3 per game."

Classic `hot_take`/`analysis` boundary. **Decision: `hot_take`** — one cherry-picked stat supports a pre-formed accusatory opinion. The framing drives the conclusion; the evidence does not independently establish it.

---

## Fine-Tuning Approach

**Base model:** `distilbert-base-uncased` (HuggingFace)

DistilBERT is a distilled version of BERT that retains ~97% of its performance at ~60% of its size — appropriate for a 200-example dataset where larger models would overfit.

**Training setup:**
- Epochs: 3
- Learning rate: 2e-5
- Batch size: 16
- Framework: HuggingFace `transformers` + `datasets`

**Key hyperparameter decision:** Learning rate was kept at the default 2e-5 rather than increased. With only 140 training examples, a higher learning rate risks rapid overfitting before the model learns generalizable representations.

---

## Baseline

**Model:** `llama-3.3-70b-versatile` via Groq API (zero-shot)

**Prompt used:**

```
You are classifying posts from r/worldcup, the FIFA World Cup Reddit community.
Assign each post to exactly one of the following categories.

analysis: The post makes a structured argument supported by specific, verifiable evidence — statistics, tactical observations, historical comparisons, or formation breakdowns.
Example: "Argentina's pressing intensity drops significantly after the 70th minute — their PPDA goes from 8.2 in the first half to 13.7 in the second, which explains why they concede late."

hot_take: A bold, confident opinion stated without supporting evidence. The post asserts rather than argues — often provocative or contrarian in framing.
Example: "Mbappe is a fraud in knockout rounds. Stats don't lie — he vanishes every single time."

reaction: An immediate emotional response to a specific match event or result. The post expresses a feeling rather than an argument.
Example: "I can't stop crying. Four years of waiting for this and it ends like that."

Respond with ONLY the label name — nothing else.
Do not explain your reasoning.

Valid labels:
analysis
hot_take
reaction
```

---

## Evaluation Report

### Overall Accuracy

| Model | Accuracy |
|---|---|
| Zero-shot baseline (Llama 3.3 70B) | **100%** |
| Fine-tuned DistilBERT | 83.3% |

The zero-shot baseline achieved perfect accuracy on the 30-example test set, which is a surprising result. The fine-tuned model underperformed the baseline by 16.7 percentage points. This outcome is discussed in the reflection section below.

---

### Per-Class Metrics

Derived from the confusion matrix (test set, 30 examples):

**Fine-tuned DistilBERT:**

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| `analysis` | 1.00 | 0.67 | 0.80 | 9 |
| `hot_take` | 0.67 | 1.00 | 0.80 | 10 |
| `reaction` | 1.00 | 0.82 | 0.90 | 11 |
| **Macro avg** | **0.89** | **0.83** | **0.83** | 30 |

---

### Confusion Matrix (Fine-Tuned Model)

Rows = true label, Columns = predicted label.

| | Predicted: analysis | Predicted: hot_take | Predicted: reaction |
|---|---|---|---|
| **True: analysis** | **6** | 3 | 0 |
| **True: hot_take** | 0 | **10** | 0 |
| **True: reaction** | 0 | 2 | **9** |

The dominant error pattern is clear: **all 5 wrong predictions were classified as `hot_take`**. The model never incorrectly predicted `analysis` or `reaction` — every error involved predicting `hot_take` when the true label was something else. `hot_take` has perfect recall (1.00) but lower precision (0.67), meaning the model over-predicts this class.

---

### Wrong Predictions — Analysis

**Wrong prediction 1:**
> *"Comparing defensive block depth across the tournament's top 8 teams shows a bimodal distribution: teams either defend very high (avg 42m) or very deep (avg 31m), with few choosing a middle approach."*
> True: `analysis` | Predicted: `hot_take` | Confidence: 0.35

This post is clearly analytical — it cites specific metrics (42m, 31m) and describes a structural pattern across multiple teams. The model's low confidence (0.35) signals it was genuinely uncertain. The likely cause: the phrase "with few choosing a middle approach" reads like an opinion or assertion, which may have triggered `hot_take` signals. This is a boundary case where assertive framing coexists with real evidence.

**Wrong prediction 2:**
> *"The xG model suggests three upsets in this tournament were actually expected — the favorites had lower pre-match xG in those games, meaning the 'upsets' reflected actual tactical mismatches, not luck."*
> True: `analysis` | Predicted: `hot_take` | Confidence: 0.38

This post contains a genuine analytical argument using xG as evidence, but its framing is contrarian ("not luck") — which is also characteristic of `hot_take`. The model appears to have weighted the contrarian framing more than the evidential content. The 0.38 confidence reflects that both signals were present and competing.

**Wrong prediction 3:**
> *"The fans in that stadium are incredible. The noise is unreal."*
> True: `reaction` | Predicted: `hot_take` | Confidence: 0.36

This is a pure `reaction` — it expresses admiration for the atmosphere. The misclassification is surprising, but the post lacks the first-person emotional language ("I can't", "I'm") that the model likely learned to associate with `reaction`. It describes the crowd rather than the author's own feelings. At 0.36 confidence the model was essentially guessing — this appears to be a data sparsity issue for reaction posts that don't use first-person language.

---

### Sample Classifications

| Post | Predicted Label | Confidence |
|---|---|---|
| "France's 4-3-3 has been consistently vulnerable on the left side of defense. Their fullback is being isolated in wide areas..." | `analysis` | 0.94 |
| "I can't believe France lost. I'm devastated." | `reaction` | 0.97 |
| "Mbappe is a fraud in knockout rounds. Stats don't lie — he vanishes every single time." | `hot_take` | 0.88 |
| "The fans in that stadium are incredible. The noise is unreal." | `hot_take` | 0.36 |
| "Brazil's buildup play has been hampered by their center backs being uncomfortable on the ball. They average only 3.2 progressive passes per game from the backline, compared to Spain's 8.7." | `analysis` | 0.96 |

**Correct prediction explained:** The first post (*"France's 4-3-3..."*) is correctly classified as `analysis` with 94% confidence. The prediction is reasonable: the post identifies a specific formation, a specific positional vulnerability, and a mechanism of exploitation. It has multiple simultaneous `analysis` signals — formation reference, positional language, and a causal claim — which together produce high confidence.

---

### Reflection: What the Model Learned vs. What Was Intended

**The baseline result needs explanation first.** A zero-shot Llama 3.3 70B achieving 100% on a 30-example test set most likely reflects the specific composition of those 30 examples rather than a generally perfect classifier. The test split happened to contain posts with clear, unambiguous signals. This does not mean the baseline would generalize perfectly to new posts — particularly the boundary cases documented in planning.md.

**What the fine-tuned model actually learned** is a surface-level proxy for the intended distinctions. For `reaction`, it learned to detect first-person emotional language and exclamation marks. For `analysis`, it learned to detect numbers, tactical vocabulary, and metric names. For `hot_take`, it appears to have learned a residual category — anything that sounds assertive or opinionated but lacks clear `reaction` or `analysis` signals.

**The systematic over-prediction of `hot_take`** is the clearest evidence of this. All 5 wrong predictions were assigned `hot_take`. Two of them were `analysis` posts with contrarian framing, and two were `reaction` posts without first-person language. The model's `hot_take` boundary absorbed cases that were ambiguous on the surface features it learned, even when a human reading carefully would classify them correctly.

**The fundamental gap:** The model cannot evaluate whether evidence is genuine or decorative. It detects the presence of number-like tokens and tactical vocabulary, but it cannot reason about whether those tokens constitute a real argument. A post saying "he's terrible — his stats prove it" and a post saying "his progressive carry rate of 4.2 per 90 ranks first in the tournament" both contain analytical-sounding language, but only the second is genuinely `analysis`. The model struggles to distinguish them consistently at 200 training examples.

---

## Spec Reflection

**One way the spec helped:** The requirement to define a hard edge case before annotating any data was the most valuable structural guidance. Writing the decision rule for `hot_take`/`analysis` before labeling forced me to commit to a consistent principle across 200 examples. Without this, I would have applied the boundary differently depending on how I felt about each post in the moment.

**One way implementation diverged:** The spec anticipates the fine-tuned model outperforming the zero-shot baseline, and most guidance assumes this will be the case. In practice, the baseline achieved 100% on the test set and the fine-tuned model scored 83.3%. This outcome — which the spec flags as a signal worth investigating — most likely reflects the small test set size (30 examples) and the particular examples that fell into the test split, rather than a genuine failure of fine-tuning. A larger test set would likely show the fine-tuned model performing comparably or better on harder boundary cases.

---

## AI Usage

**Instance 1 — Label stress-testing:**
I provided Claude with the three label definitions and asked it to generate posts sitting at the `hot_take`/`analysis` boundary. Several generated examples included a single statistic in an otherwise accusatory framing, which I found difficult to classify. This directly produced the decision rule in planning.md: evidence must be independently persuasive without the opinion framing to qualify as `analysis`. I revised the label definition based on these examples before annotating any real posts.

**Instance 2 — Annotation pre-labeling:**
For approximately 40 posts, I used an LLM with the label definitions to pre-assign labels, then reviewed every pre-assigned label manually. Agreement was high for `reaction` (~90%) and lower for `hot_take`/`analysis` boundary cases (~70%). I overrode approximately 12 pre-assigned labels where the LLM treated any analytical-sounding language as `analysis` regardless of whether genuine evidence was present.

**Instance 3 — Failure pattern analysis:**
After identifying the 5 wrong predictions, I used an LLM to identify patterns. The AI identified that all errors involved prediction of `hot_take`, and that three of the five were low-confidence predictions (≤0.38). I verified this manually by reviewing each wrong prediction and confirmed the systematic pattern: the model uses `hot_take` as a catch-all for posts that lack strong `reaction` or `analysis` surface signals.
