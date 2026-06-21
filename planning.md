# TakeMeter Planning — r/anime Discourse Classifier

## Community Choice

r/anime is one of the largest anime discussion communities on Reddit (~8M members). Discourse ranges from one-word episode reactions to multi-paragraph thematic analyses. The community itself actively distinguishes between low-effort reactions and substantive discussion — moderators enforce rules around "low-effort content" and users frequently call out "bad takes." This makes it a natural fit for a classification task: the quality distinctions already exist in the community's norms, they just aren't formalized.

## Label Taxonomy

### 1. `analysis` — Structured argument with specific evidence

The post makes a claim about an anime and supports it with specific references: named episodes, scenes, character arcs, animation techniques, or thematic patterns. The reasoning could be wrong, but the post *argues* rather than asserts. Evidence is specific enough that another viewer could verify or dispute it.

**Examples:**
- "Evangelion's hospital scene in End of Eva isn't shock value — it's the culmination of Shinji's instrumentality arc. Throughout episodes 25-26, we see him unable to connect without consuming, and that scene literalizes it. Anno is forcing the audience into the same discomfort Shinji feels."
- "Mob Psycho 100's animation quality drops in S2E5 aren't budget cuts — Bones intentionally uses rougher key frames during Mob's emotional breakdowns to mirror his loss of control. Compare the clean lines in the Claw fights vs the sketchy style when he hits ???%."

### 2. `hot_take` — Bold opinion stated without supporting evidence

A confident, often provocative claim about an anime, character, or the medium itself. The post asserts rather than argues. It might reference an anime by name but doesn't provide specific evidence that would let someone evaluate the claim. Often framed to provoke reaction.

**Examples:**
- "Demon Slayer is carried entirely by Ufotable's animation. Without it, the story is mid shonen at best. Fight me."
- "People who think Chainsaw Man Part 2 is better than Part 1 are just contrarians. Denji's arc peaked at the end of Part 1 and everything since is Fujimoto being weird for the sake of weird."

### 3. `reaction` — Immediate emotional response to a specific moment or event

An expression of feeling triggered by something the poster just watched, read, or learned about. Little to no argument — the post is sharing an emotional state, not making a case. Often short, uses caps/exclamation marks, references a specific episode or scene without analyzing it.

**Examples:**
- "JUST FINISHED CLANNAD AFTER STORY AND I AM NOT OKAY. That field scene broke me completely 😭😭"
- "lmao the face Anya made in ep 8 killed me, Spy x Family is the only thing getting me through this week"

## Hard Edge Cases

### The "evidence-decorated hot take"

**Ambiguous post:** "Attack on Titan's ending ruined the entire series — Isayama spent 130 chapters building Eren as a free-will absolutist and then had him admit he didn't know why he did it. That's not subversion, that's bad writing."

This post *names* specific evidence (130 chapters, Eren's admission) but the reasoning is assertion-as-analysis. The evidence is selected to support a pre-determined verdict ("bad writing") rather than building toward a conclusion.

**Decision rule:** If the post provides evidence that *builds toward* a conclusion through reasoning — connecting multiple points, considering counterarguments, or explaining *why* the evidence supports the claim — label it `analysis`. If the evidence is decorative — a single reference used to make an assertion sound authoritative — label it `hot_take`. The key question: could you remove the opinion framing and still have an argument? If removing "that's bad writing" leaves a coherent analytical structure, it's `analysis`. If removing the opinion leaves just a plot summary + judgment, it's `hot_take`.

### The "analytical reaction"

**Ambiguous post:** "Just watched Violet Evergarden ep 10 and I'm crying. The way they used the time skip to show Ann growing up while her mother's letters kept arriving — it hit different because we knew the whole time what Ann didn't."

This is emotionally immediate (reaction) but includes specific structural observation (analysis).

**Decision rule:** What is the *primary purpose* of the post? If the poster is processing an emotional experience and the specifics are incidental context, label it `reaction`. If the poster is making an observation about how the show achieves its effect, label it `analysis`, even if the tone is emotional. The post above names a specific technique (time skip + dramatic irony) and explains *why* it works → `analysis`.

### Short ambiguous posts

Posts under ~20 words are often hard to classify because they lack enough signal. Example: "AoT ending was trash." This is a `hot_take` (bold claim, no evidence), but it's borderline with `reaction` (could be an immediate emotional response). **Decision rule:** If the post makes a *judgment* (good/bad/overrated/trash), it's `hot_take`. If it expresses a *feeling* (crying/laughing/shocked/hyped), it's `reaction`. Judgment = hot_take, feeling = reaction.

## Data Collection Plan

**Source:** Reddit's r/anime — specifically episode discussion threads, recommendation threads, and standalone discussion posts. These contain the widest range of discourse quality.

**Method:** Manual collection via browsing r/anime. Copy post text into a spreadsheet, assign labels per the definitions above. This keeps me close to the data and forces me to read every example carefully.

**Target distribution:**
- `analysis`: ~70 examples (35%)
- `hot_take`: ~70 examples (35%)
- `reaction`: ~60 examples (30%)

Analysis posts are rarer in the wild (most r/anime comments are reactions or hot takes), so I'll specifically seek out "Writing" and "Discussion" flaired threads to find enough.

**If a label is underrepresented:** I'll search for analysis-heavy threads (WT! posts, essay-style discussion threads, rewatch threads where users write detailed episode breakdowns). If still short, I'll pull from r/TrueAnime or r/CharacterRant which skew more analytical.

## Evaluation Metrics

**Overall accuracy** — baseline measure, but not sufficient on its own. With 3 roughly balanced classes, random guessing gives ~33%. Need to meaningfully beat that.

**Per-class F1 score** — the key metric. F1 captures both precision and recall per label. This matters because:
- High precision + low recall for `analysis` would mean the model is too conservative — only labeling obvious deep-dives and missing shorter analytical posts.
- High recall + low precision for `hot_take` would mean the model dumps anything opinionated into hot_take, including real analysis.
- F1 balances both failure modes and lets me compare across classes.

**Confusion matrix** — to identify which specific label pairs the model confuses. I expect `hot_take` ↔ `analysis` to be the hardest boundary based on the edge cases above.

**Macro-averaged F1** — single summary number that weights all classes equally regardless of distribution.

## Definition of Success

**Good enough for a useful tool:** Macro F1 ≥ 0.65 on the test set, with no single class F1 below 0.50. This means the model is learning all three distinctions at least somewhat, not just collapsing two labels together.

**Fine-tuning should beat baseline:** The fine-tuned model should exceed the zero-shot Groq baseline by at least 10 percentage points on overall accuracy. If it doesn't, either the labels are too easy (LLM already gets them) or the training data is too noisy.

**Stretch goal:** Macro F1 ≥ 0.75, which would indicate the model is reliably distinguishing all three categories.

## AI Tool Plan

### Label stress-testing
Before annotating, I'll give Claude my label definitions and edge case descriptions and ask it to generate 10 posts that sit at the boundary between `analysis` and `hot_take`. If I can't cleanly classify the generated posts using my decision rules, I'll tighten the definitions before committing to 200 annotations.

### Annotation assistance
I plan to use an LLM to pre-label a batch of ~50 posts after I've manually labeled the first 100. I'll provide my label definitions and examples, then review and correct every pre-assigned label. I'll track which examples were pre-labeled in a `prelabeled` column in my CSV (1 = LLM pre-labeled then reviewed, 0 = manually labeled from scratch). All pre-labeled examples will be fully reviewed and corrected where needed.

### Failure analysis
After training, I'll paste all misclassified examples into Claude and ask it to identify patterns: are errors concentrated in short posts? sarcastic posts? a specific label pair? I'll verify any patterns it surfaces by re-reading the examples myself, and discard patterns that don't hold up on inspection.
