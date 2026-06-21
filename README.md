# TakeMeter — r/anime Discourse Classifier

A fine-tuned text classifier that evaluates discourse quality in r/anime, distinguishing between **analysis**, **hot takes**, and **reactions**.

## Community Choice

r/anime (~8M members) is one of Reddit's largest anime discussion communities. Discourse ranges from one-word episode reactions to multi-paragraph thematic breakdowns. The community already informally distinguishes between low-effort reactions and substantive discussion — moderators enforce "low-effort" content rules, and users frequently call out "bad takes." This project formalizes those distinctions into a classifier.

Why it's a good fit: the community is text-heavy, posts vary widely in quality, and the quality distinctions are grounded in real community norms rather than imposed externally.

## Label Taxonomy

### 1. `analysis` — Structured argument with specific evidence
The post makes a claim and supports it with specific references: named episodes, scenes, character arcs, animation techniques, or thematic patterns. The post *argues* rather than asserts.

**Examples:**
- "Evangelion's hospital scene in End of Eva isn't shock value — it's the culmination of Shinji's instrumentality arc. Throughout episodes 25-26, we see him unable to connect without consuming, and that scene literalizes it."
- "Mob Psycho 100's animation drops in S2E5 aren't budget cuts — Bones intentionally uses rougher key frames during Mob's emotional breakdowns to mirror his loss of control."

### 2. `hot_take` — Bold opinion without supporting evidence
A confident, often provocative claim that asserts rather than argues. May reference an anime by name but doesn't provide specific evidence to evaluate the claim.

**Examples:**
- "Demon Slayer is carried entirely by Ufotable's animation. Without it, the story is mid shonen at best."
- "People who think Chainsaw Man Part 2 is better than Part 1 are just contrarians."

### 3. `reaction` — Immediate emotional response
An expression of feeling triggered by something just watched or read. Little to no argument — sharing an emotional state, not making a case.

**Examples:**
- "JUST FINISHED CLANNAD AFTER STORY AND I AM NOT OKAY 😭😭"
- "lmao the face Anya made in ep 8 killed me"

## Data Collection

**Source:** Reddit r/anime — episode discussion threads, recommendation threads, and standalone discussion posts.

**Labeling process:** Fully manual. Posts were copied from MAL reviews, episode discussion threads, and discussion forums, then read in full and labeled by hand against the taxonomy definitions above — no pre-labeling or AI-assisted labeling was used for the dataset itself. Ambiguous posts were checked against the decision rules in `planning.md` before assigning a label.

**Label distribution:**

| Label | Count | Percentage |
|-------|-------|------------|
| analysis | 73 | 36% |
| hot_take | 61 | 30% |
| reaction | 69 | 34% |
| **Total** | 203 (split 142 / 30 / 31 train/val/test) | 100% |

### Difficult-to-Label Examples

1. **Example:** "The manga readers have the most varied opinions on the anime, both negative and positive, but as an anime only until after the movie, it has been flawless perfection from start to finish. A real classic."
   - **Could be:** `hot_take` or `analysis`
   - **Decision:** `hot_take`. It names a real distinction (manga readers vs anime-only) but doesn't build an argument from it — "flawless perfection" is asserted, not supported. Removing the praise leaves no analytical structure, just a comparison setup. This is the "evidence-decorated hot take" pattern from `planning.md`.

2. **Example:** "I cant believe it is finally over. Im so sad to see it go. Ive loved this anime so much... Celty calms everything down. Takashi gets hit back and tortured. The trio is back together..."
   - **Could be:** `reaction` or `analysis`
   - **Decision:** `reaction`. It lists several specific plot beats, which can look like evidence, but the list isn't used to argue anything — it's a farewell recap driven by emotion ("I'm so sad to see it go"). The specificity is incidental context for the feeling, not the point of the post.

3. **Example:** "It actually peaks at S2. The latest special is also phenomenal."
   - **Could be:** `hot_take` or `reaction`
   - **Decision:** `hot_take`. Short and could read as enthusiastic reaction, but it makes a comparative judgment (this season > others) rather than expressing a feeling. Per the short-post decision rule: judgment = hot_take, feeling = reaction.

## Fine-Tuning Approach

**Base model:** `distilbert-base-uncased` via HuggingFace Transformers

**Platform:** Google Colab (free T4 GPU)

**Training setup:**
- Epochs: 6
- Learning rate: 2e-5
- Batch size: 8 (train), 32 (eval)
- Warmup steps: 10
- Best checkpoint selected by validation macro F1

**Hyperparameter decision:** The starter notebook's defaults (3 epochs, batch size 16, warmup_steps=50) caused total model collapse — the model predicted `analysis` for nearly every test example (38.7% accuracy). The root cause: with only 142 training examples and batch size 16, each epoch is ~9 optimizer steps, so 3 epochs is only ~27 steps total — fewer than the 50 warmup steps configured. The learning rate scheduler never finished warming up, so the model never received a meaningful gradient signal before training stopped. Fixing this required reducing batch size to 8 (more steps per epoch), increasing epochs to 6, and reducing warmup_steps to 10 (well under the new ~108 total steps). I also switched checkpoint selection from `accuracy` to `f1_macro` since accuracy alone doesn't penalize collapse onto a majority class. After this fix, the model produced genuinely differentiated predictions across all three classes instead of collapsing.

## Baseline Comparison

**Baseline model:** Groq `llama-3.3-70b-versatile` (zero-shot)

**Prompt used:**
```
You are classifying posts from r/anime, a large anime discussion community.
Assign each post to exactly one of the following categories.

analysis: A structured argument with specific evidence — named episodes, scenes,
character arcs, animation techniques, or thematic patterns that another viewer
could verify or dispute. The post argues toward a conclusion rather than just
asserting one.
Example: "Mob Psycho 100's animation quality drops in S2E5 aren't budget cuts —
Bones intentionally uses rougher key frames during Mob's emotional breakdowns to
mirror his loss of control. Compare the clean lines in the Claw fights vs the
sketchy style when he hits ???%."

hot_take: A bold, confident opinion stated without supporting evidence. The post
asserts rather than argues — it may name a show or character but gives no
specific evidence someone could evaluate.
Example: "Demon Slayer is carried entirely by Ufotable's animation. Without it,
the story is mid shonen at best. Fight me."

reaction: An immediate emotional response to a specific moment or event, with
little to no argument. Often short, uses caps or exclamation marks, expresses a
feeling rather than a judgment.
Example: "JUST FINISHED CLANNAD AFTER STORY AND I AM NOT OKAY. That field scene
broke me completely"

Respond with ONLY the label name.
Do not explain your reasoning.

Valid labels:
analysis
hot_take
reaction
```

**How results were collected:** Each test example was sent to Groq with label definitions and the model was asked to output only the label name. Results parsed and compared against ground truth.

## Evaluation Report

### Overall Accuracy

| Model | Accuracy |
|-------|----------|
| Zero-shot baseline (Llama 3.3 70B) | 0.742 |
| Fine-tuned DistilBERT | 0.613 |

The fine-tuned model **underperformed the baseline by 12.9 points**, missing the project's success target of beating baseline by 10+ points. See Reflection section below for analysis of why.

### Per-Class Metrics (Fine-Tuned Model)

| Label | Precision | Recall | F1 |
|-------|-----------|--------|----|
| analysis | 0.60 | 0.82 | 0.69 |
| hot_take | 0.75 | 0.33 | 0.46 |
| reaction | 0.58 | 0.64 | 0.61 |
| **Macro Avg** | 0.64 | 0.60 | 0.59 |

Macro F1 of 0.59 falls short of the 0.65 target, and `hot_take` F1 (0.46) falls below the 0.50 floor set in `planning.md`.

### Per-Class Metrics (Zero-Shot Baseline)

| Label | Precision | Recall | F1 |
|-------|-----------|--------|----|
| analysis | 0.89 | 0.73 | 0.80 |
| hot_take | 0.67 | 0.67 | 0.67 |
| reaction | 0.69 | 0.82 | 0.75 |
| **Macro Avg** | 0.75 | 0.74 | 0.74 |

### Confusion Matrix (Fine-Tuned Model)

| Predicted → | analysis | hot_take | reaction |
|-------------|----------|----------|----------|
| **analysis** | 9 | 1 | 1 |
| **hot_take** | 2 | 3 | 4 |
| **reaction** | 4 | 0 | 7 |

`hot_take` is the model's weakest class by far — only 3/9 correctly identified, most often confused with `reaction` (4 cases). This lines up with the taxonomy's hardest boundary noted in `planning.md`: short, assertive posts can look like either a judgment (`hot_take`) or a feeling (`reaction`) without more context than a 256-token DistilBERT model retains well.

### Wrong Prediction Analysis

1. **Post:** "The manga readers have the most varied opinions on the anime... it has been flawless perfection from start to finish. A real classic."
   - **True label:** hot_take | **Predicted:** analysis (confidence: 0.55)
   - **Why it failed:** The post mentions a real distinction (manga readers' opinions vs. anime-only) which gives it surface texture resembling evidence. The model likely keyed on this comparative framing and the discussion of "opinions" without distinguishing that the post never actually builds an argument from it — it just asserts "flawless perfection." Low confidence (0.55) shows the model itself was near the decision boundary, mirroring the human labeling difficulty on this exact post (see Difficult-to-Label Examples).

2. **Post:** "It actually peaks at S2. The latest special is also phenomenal."
   - **True label:** hot_take | **Predicted:** reaction (confidence: 0.44)
   - **Why it failed:** Short posts with strong positive language ("phenomenal") seem to get pulled toward `reaction` by the model regardless of whether they're a feeling or a judgment. This is the same hot_take/reaction boundary that's weakest in the confusion matrix (4 of 9 hot_take posts misrouted to reaction) — the model appears to be using emotional intensity of word choice as a proxy for "reaction," when the taxonomy's actual distinction is judgment vs. feeling, not intensity.

3. **Post:** "I cant believe it is finally over. Im so sad to see it go... Celty calms everything down. Takashi gets hit back and tortured. The trio is back together..."
   - **True label:** reaction | **Predicted:** analysis (confidence: 0.56)
   - **Why it failed:** The opposite failure mode — this post lists several specific plot events, which the model likely read as evidence-bearing (similar to genuine `analysis` posts that reference specific scenes). But the list isn't argumentative; it's an emotional farewell recap. With only 256 max tokens and no explicit reasoning markers ("because," "this shows that"), the model can't reliably tell "specific references used to argue" apart from "specific references used to express feeling."

### Sample Classifications

| Post (truncated) | True Label | Predicted Label | Confidence | Correct? |
|-------------------|-----------|-----------------|------------|----------|
| "The manga readers have the most varied opinions... flawless perfection from start to finish." | hot_take | analysis | 0.55 | ❌ |
| "Part 1 of season 2 is probably the slowest part of the show... the payoff is huge." | analysis | hot_take | 0.47 | ❌ |
| "No don't watch it. It sucks" | hot_take | reaction | 0.70 | ❌ |
| "I don't think she got darker, it's the lighting/gloom choice to contrast Alucard and Integra..." | analysis | reaction | 0.41 | ❌ |
| "Holy bajeez. I can count on one hand the amount of times any show... has shot up THIS much in quality between releases." | reaction | analysis | 0.74 | ❌ |

Note: confidence scores on wrong predictions cluster low (0.41–0.74, mostly under 0.6), meaning the model is rarely confidently wrong — its errors are concentrated near genuine decision boundaries rather than confident misreads.

## Reflection: What the Model Learned vs. What Was Intended

The taxonomy was designed around a single underlying distinction per pair of labels: `analysis` vs. `hot_take` is "argues vs. asserts," and `hot_take` vs. `reaction` is "judgment vs. feeling." The fine-tuned model did not learn either distinction cleanly.

For `analysis` vs. `hot_take`, the model appears to have learned a shallower proxy: posts that mention specific show/character details (episode numbers, character names, plot events) get pulled toward `analysis`, regardless of whether those details are used to build an argument or just decorate an opinion or recap a feeling. This explains why `reaction` posts with detailed plot recaps (error #3) and `hot_take` posts with surface-level comparisons (error #1) both got misclassified as `analysis` — the model is pattern-matching on the *presence* of specifics, not on whether the post *reasons* from them.

For `hot_take` vs. `reaction`, the model seems to be using emotional word intensity as a proxy ("phenomenal," "sucks") rather than the intended distinction between making a judgment and expressing a feeling. Both label pairs the model confuses are exactly the boundaries flagged as hardest in `planning.md` before any training happened — the edge cases were correctly anticipated, but a 66M-parameter model fine-tuned on 142 examples doesn't have the capacity or data volume to learn the more abstract distinction (argues-vs-asserts, judges-vs-feels) that the larger zero-shot baseline picks up from its general language understanding.

This also explains the overall result: the zero-shot baseline (Llama 3.3 70B) outperformed the fine-tuned DistilBERT by 12.9 accuracy points. A model with vastly more parameters and pretraining exposure to argumentative vs. emotional language generalizes the taxonomy's *intent* better from just a prompt than a small model can learn from ~140 labeled examples. With more training data (the project spec's 200+ minimum is a floor, not necessarily enough for this task's difficulty), I'd expect the gap to narrow, but closing it entirely with DistilBERT and a dataset this size may not be realistic.

## Spec Reflection

**How the spec helped:** Writing the decision rules for hard edge cases in `planning.md` *before* collecting data forced me to think through the taxonomy's actual boundaries in advance. This paid off directly during error analysis — the model's two main failure modes (analysis-vs-hot_take confusion, hot_take-vs-reaction confusion) are the exact two edge cases I'd already identified and written decision rules for. Having those rules documented made it fast to diagnose *why* specific posts were misclassified instead of just observing that they were.

**Where implementation diverged:** The spec's default training hyperparameters (3 epochs, batch size 16, warmup_steps=50) assumed a dataset large enough that 50 warmup steps wouldn't dominate the whole training run. With only 142 training examples, that assumption broke — total training steps came out lower than the warmup step count, causing the model to collapse to predicting a single class. I had to diagnose and fix this (batch size 8, epochs 6, warmup_steps 10) before getting any usable signal at all, which wasn't something the starter notebook's defaults anticipated for a dataset this small.

## AI Usage

I used Claude (Claude Code) as a guide throughout this project for planning, debugging, and writing this report. All taxonomy decisions, data collection sourcing, and label assignments were mine.

### Instance 1: Debugging model collapse
When the first fine-tuning run collapsed to predicting `analysis` for almost every test example (38.7% accuracy), I described the symptom and confusion matrix to Claude. It identified that `warmup_steps=50` exceeded the actual total training steps (~27, given 142 examples / batch size 16 / 3 epochs), meaning the LR scheduler never finished warming up. I applied the suggested fix (batch size 8, more epochs, lower warmup_steps) myself in Colab.

### Instance 2: Drafting the evaluation report
After running the final baseline and fine-tuned evaluations, I pasted the raw `evaluation_results.json`, confusion matrix image, classification reports, and wrong-prediction printouts to Claude and asked it to help draft this README's evaluation, error-analysis, and reflection sections. I reviewed and can speak to all the claims made, but the prose drafting in those sections was AI-assisted.

### Disclosure: Annotation
No AI assistance was used for labeling the 203 posts in the dataset — every label was assigned by manually reading the post against the taxonomy in `planning.md`.
