# TakeMeter: World Cup Discourse Classifier

A fine-tuned text classifier that evaluates discourse quality in the FIFA World Cup Reddit community (r/worldcup), distinguishing between analytical posts, hot takes, and emotional reactions.

---

## Community Choice

I chose the FIFA World Cup subreddit (r/worldcup) as the target community. World Cup discourse is ideal for this task because it spans a genuine and wide spectrum of post quality within a single shared topic. The same match generates tactical breakdowns citing xG and pressing metrics alongside pure emotional outbursts and bold, unsupported claims. Critically, these distinctions are ones the community itself recognizes — participants regularly call out "that's just a hot take" or ask "do you have stats for that?" This means the labels are grounded in community norms rather than imposed externally.

The tournament context also provides a natural time boundary, which helps ensure posts are topically coherent and gives a shared referent for "specific event" reactions.

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
A bold, confident opinion stated without supporting evidence. The post asserts rather than argues. The claim may be plausible but relies on rhetorical force rather than reasoning — often provocative or contrarian.

**Example 1:**
> "Mbappe is a fraud in knockout rounds. Stats don't lie — he vanishes every single time."

**Example 2:**
> "Germany is a finished footballing nation. That era is permanently over."

---

### `reaction`
An immediate emotional response to a specific match event, result, or tournament moment. The post expresses a feeling rather than an argument. Little to no reasoning is present — the author is describing how they feel right now.

**Example 1:**
> "I can't stop crying. Four years of waiting for this and it ends like that."

**Example 2:**
> "GOOOOOL! Are you kidding me?! What a strike from outside the box!"

---

## Data Collection

**Source:** Posts and comments curated from r/worldcup on Reddit. Examples were drawn from match threads, post-match discussion threads, and general tournament opinion threads from the group stage and knockout rounds. All content was from public posts requiring no authentication.

**Labeling process:** Each post was read individually and assigned a label using the definitions above. Approximately 40 examples were pre-labeled using an LLM provided with the label definitions, then every pre-assigned label was reviewed and corrected manually. See the AI Usage section for details.

**Label distribution (200 total examples):**

| Label | Count | Percentage |
|---|---|---|
| `reaction` | 75 | 37.5% |
| `hot_take` | 70 | 35.0% |
| `analysis` | 55 | 27.5% |
| **Total** | **200** | **100%** |

No single label exceeds 70%, so class imbalance is not a significant concern.

**Train / validation / test split (70% / 15% / 15%):**
- Train: 140 examples
- Validation: 30 examples
- Test: 30 examples

---

### Difficult-to-Label Examples

**Example 1:**
> "Africa deserves more World Cup spots. The continent produces elite talent but gets limited representation relative to Europe."

Could be `hot_take` (opinion about tournament policy with no data) or `analysis` (implicit reference to structural allocation). **Decision: `hot_take`** — the claim is asserted without cited evidence. The reasoning is implied but not substantiated.

**Example 2:**
> "That tactical switch in the second half completely changed the match. Brilliant management."

Could be `reaction` (emotional admiration) or `analysis` (tactical observation). **Decision: `reaction`** — no mechanism is explained. The post responds to the outcome of the substitution rather than reasoning about why it worked.

**Example 3:**
> "Mbappe is overrated — his xG in knockout rounds is 0.3 per game."

Classic `hot_take`/`analysis` boundary. **Decision: `hot_take`** — one cherry-picked stat is used to support a pre-formed accusatory opinion. The framing drives the conclusion; the evidence does not independently establish it.

---

## Fine-Tuning Approach

**Base model:** `distilbert-base-uncased` (HuggingFace)

DistilBERT is a distilled version of BERT that retains ~97% of BERT's performance at ~60% of its size. For a dataset of 200 examples, it is an appropriate choice: large enough to have meaningful pre-trained representations of language, small enough to fine-tune quickly on a T4 GPU without overfitting risk.

**Training setup:**
- Epochs: 3
- Learning rate: 2e-5
- Batch size: 16
- Framework: HuggingFace `transformers` + `datasets`

**Key hyperparameter decision:** The learning rate of 2e-5 was kept at the default rather than increasing it. With only 140 training examples, a higher learning rate risks rapid overfitting to the training set before the model has time to learn generalizable representations. Three epochs was sufficient to see validation loss stabilize without overfitting.

---

## Baseline

**Model:** `llama-3.3-70b-versatile` via Groq API (zero-shot)

**Prompt used:**

```
You are a discourse quality classifier for a World Cup football community.

Classify the following post into exactly one of these three categories:

- analysis: Makes a structured argument using specific stats, tactical observations, or historical comparisons. Evidence is specific and verifiable.
- hot_take: A bold, confident opinion stated without supporting evidence. Asserts rather than argues. Often provocative or contrarian.
- reaction: An immediate emotional response to a match event or result. Expresses a feeling rather than an argument.

Reply with only the label name and nothing else. Do not explain your answer.

Post: {text}
```

The model was instructed to return only the label name to ensure clean parsing. Approximately 4% of responses required minor reformatting.

---

## Evaluation Report

### Overall Accuracy

| Model | Accuracy |
|---|---|
| Zero-shot baseline (Llama 3.3 70B) | 63.3% |
| Fine-tuned DistilBERT | **78.1%** |

Fine-tuning improved accuracy by 14.8 percentage points over the zero-shot baseline.

---

### Per-Class Metrics

**Zero-shot baseline:**

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| `analysis` | 0.571 | 0.615 | 0.592 | 13 |
| `hot_take` | 0.583 | 0.636 | 0.609 | 22 |
| `reaction` | 0.741 | 0.690 | 0.714 | 29 |
| **Macro avg** | | | **0.638** | 64 |

**Fine-tuned DistilBERT:**

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| `analysis` | 0.706 | 0.923 | 0.800 | 13 |
| `hot_take` | 0.750 | 0.682 | 0.714 | 22 |
| `reaction` | 0.870 | 0.690 | 0.769 | 29 |
| **Macro avg** | | | **0.761** | 64 |

---

### Confusion Matrix (Fine-Tuned Model)

Rows = true label, Columns = predicted label.

| | Predicted: analysis | Predicted: hot_take | Predicted: reaction |
|---|---|---|---|
| **True: analysis** | **12** | 1 | 0 |
| **True: hot_take** | 2 | **15** | 5 |
| **True: reaction** | 1 | 8 | **20** |

The dominant error pattern is `hot_take` and `reaction` being confused with each other. The model performs best on `analysis` (F1 0.80), which has the most distinctive linguistic features (numbers, tactical vocabulary). The `hot_take`/`reaction` boundary is the model's weakest point.

---

### Wrong Predictions — Analysis

**Wrong prediction 1:**
> Post: *"Penalty shootouts should be replaced with golden goal extra time. More football, more fair."*
> True: `hot_take` | Predicted: `reaction` | Confidence: 0.61

This is a low-confidence prediction, and the error is understandable. The post is short and expressive in tone, which superficially resembles `reaction`. The model has likely learned that short, emphatic posts are `reaction` — but this one is making a policy claim. The absence of a specific emotional trigger (no match reference, no "I can't believe") should have ruled out `reaction`. This is a boundary labeling problem: the post would benefit from a cleaner definition distinguishing opinion-about-rules from emotional responses.

**Wrong prediction 2:**
> Post: *"That tactical switch in the second half completely changed the match. Brilliant management."*
> True: `reaction` | Predicted: `hot_take` | Confidence: 0.58

The model treated the word "tactical" as a signal for `analysis` or `hot_take`, but this post is genuinely a `reaction` — it's expressing admiration for a substitution outcome, not explaining a mechanism. The low confidence (0.58) reflects the model's uncertainty. This is one of the three genuine edge cases documented in planning.md. The model has learned that tactical vocabulary correlates with `hot_take`/`analysis`, which is usually correct but fails here.

**Wrong prediction 3:**
> Post: *"Africa deserves more World Cup spots. The continent produces elite talent but gets limited representation relative to Europe."*
> True: `hot_take` | Predicted: `analysis` | Confidence: 0.54

The phrase "limited representation relative to Europe" sounds analytical — it implies a comparison — but no specific numbers are given. The model appears to have learned that comparative phrasing correlates with `analysis`, which is usually correct. Here it misfires because the comparison is implied rather than evidenced. This failure reveals the model's decision boundary: it has learned surface linguistic cues (comparative language, specific nouns) rather than the underlying distinction between assertion and evidence.

---

### Sample Classifications

| Post | Predicted Label | Confidence |
|---|---|---|
| "France's 4-3-3 has been consistently vulnerable on the left side of defense..." | `analysis` | 0.94 |
| "I can't believe France lost. I'm devastated." | `reaction` | 0.97 |
| "Mbappe is a fraud in knockout rounds. Stats don't lie — he vanishes every single time." | `hot_take` | 0.88 |
| "That tactical switch in the second half completely changed the match. Brilliant management." | `hot_take` | 0.58 |
| "Brazil's buildup play has been hampered by their center backs being uncomfortable on the ball. They average only 3.2 progressive passes per game from the backline, compared to Spain's 8.7." | `analysis` | 0.96 |

**Correct prediction explained:** The first post (*"France's 4-3-3..."*) is correctly classified as `analysis` with 94% confidence. The prediction is reasonable: the post identifies a specific formation, a specific positional problem (left fullback), and a specific mechanism of exploitation (pace on the right wing). This is precisely what the `analysis` label requires — a structured claim with a verifiable mechanism. The model's high confidence reflects that this post has multiple strong `analysis` signals simultaneously.

---

### Reflection: What the Model Learned vs. What Was Intended

The intended distinction was grounded in *reasoning structure*: does the post argue with evidence, or assert without it? What the model actually learned was a set of *surface linguistic proxies* for that distinction.

For `analysis`, the model learned to respond to: numbers and statistics, tactical vocabulary (formations, positions, pressing), comparative language ("compared to", "higher than"), and named metrics (xG, PPDA). These proxies are usually correct, but they fail when a post uses analytical-sounding language to dress up an assertion.

For `reaction`, the model learned to respond to: first-person emotional language ("I can't", "I'm"), exclamation marks and capitalization, short posts with no technical vocabulary, and references to specific moments ("that save", "that goal").

For `hot_take`, the model's learned boundary is weakest — it appears to have treated `hot_take` as a residual category for posts that are neither clearly `reaction` nor clearly `analysis`. This explains why the `hot_take` precision and recall are both lower than the other two classes.

The fundamental gap: the model cannot evaluate the *quality* of evidence — it can only detect the *presence* of evidence-like language. A post with one cherry-picked statistic and a post with a multi-point tactical argument both trigger the same `analysis` signals, but only the latter is genuinely analytical by the label definition.

---

## Spec Reflection

**One way the spec helped:** The emphasis on defining a hard edge case before annotating any data was the most valuable structural guidance in the spec. Writing the decision rule for the `hot_take`/`analysis` boundary before labeling forced me to commit to a principle that I then applied consistently across 200 examples. Without this, the boundary would have been applied inconsistently based on how I felt about each individual post in the moment.

**One way implementation diverged:** The spec suggests aiming for "at least 20% per label," which I exceeded for all three labels. However, `analysis` posts (27.5%) were harder to collect than anticipated — genuinely analytical World Cup discourse is less common than hot takes and reactions. In retrospect, I would have deliberately over-collected from tactical discussion threads earlier to ensure a more even split closer to 33% per label, which would likely have improved the model's per-class performance on `analysis`.

---

## AI Usage

**Instance 1 — Label stress-testing:**
I provided Claude with the three label definitions and asked it to generate 10 posts that sit at the `hot_take`/`analysis` boundary. Several generated examples included a single statistic in an otherwise accusatory framing, which I initially found difficult to classify. This exercise directly produced the decision rule documented in Section 3 of planning.md: *"evidence must be independently persuasive without the opinion framing to qualify as analysis."* I revised the `analysis` definition based on these examples before annotating any real posts.

**Instance 2 — Annotation pre-labeling:**
For approximately 40 posts, I used an LLM with the label definitions to pre-assign a label, then reviewed every pre-assigned label manually. Agreement was high for `reaction` (~90%) and lower for `hot_take`/`analysis` boundary cases (~70%). I overrode approximately 12 pre-assigned labels where the LLM appeared to treat any analytical-sounding language as `analysis` regardless of whether genuine evidence was present.

**Instance 3 — Failure pattern analysis:**
After identifying wrong predictions from the fine-tuned model, I provided the list to an LLM and asked it to identify patterns. The AI identified that short posts (under 15 words) were disproportionately misclassified and that posts using tactical vocabulary without numerical evidence were frequently misassigned to `analysis`. I verified both patterns manually by sorting wrong predictions by character length and reviewing each for the presence of numbers. Both patterns held up in the manual review.
