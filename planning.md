# Project Planning: r/LetsTalkMusic Post Classifier

## Community

I chose r/LetsTalkMusic because it is a text-heavy community where post quality
varies significantly — some posts are deeply analytical while others are pure
personal reaction. This makes it a strong fit for a classification task because
the distinctions between post types are meaningful to the community itself:
regulars there actively value substantive discussion over low-effort opinions,
so the labels reflect a real social distinction, not an arbitrary one.

---

## Labels

### `opinion`
A post that expresses a personal feeling or preference about an artist, album,
or song without supporting reasoning or evidence.

- *"Random Access Memories is honestly one of the best albums ever made.
  It just hits different every time I listen to it."*
- *"I can't get into Radiohead no matter how hard I try.
  Thom Yorke's voice just annoys me."*

### `discussion`
A post that poses a question or open-ended topic inviting the community to
debate or share views.

- *"Do you think concept albums are becoming less relevant in the streaming era?
  Hard to experience them as a full work when people shuffle."*
- *"What's a critically acclaimed album you genuinely can't understand
  the hype for?"*

### `recommendation`
A post that primarily suggests music to listen to, or asks others for
suggestions, with minimal or brief reasoning.

- *"If you liked Sufjan Stevens' Illinois, check out The Microphones
  — very similar folk-orchestral layering."*
- *"Any experimental music recommendations? I've been mostly into metal
  but want to branch out."*

---

## Hard Edge Cases

**`opinion` vs. `discussion`:**
Some posts phrase an opinion as a question to seem like they're inviting debate,
but are really just stating a take. Example: *"Am I the only one who thinks
Tyler the Creator is overrated?"*

**Rule:** If the question is rhetorical and the post doesn't genuinely open
multiple sides of a debate, label it `opinion`.

Real example encountered during annotation: *"Do you think music is becoming
more genreless?"* — this was labeled `discussion` because it genuinely invites
multiple perspectives, not just agreement or disagreement with one position.

**`recommendation` vs. `discussion`:**
Some recommendation posts are phrased as questions, making them look like
discussion posts. Example: *"Any experimental music recommendations? I've been
a music fan all my life, mostly metal."*

**Rule:** If the post is primarily asking for or giving specific music
suggestions rather than inviting broad debate, label it `recommendation`.

Real example encountered during annotation: *"Can anyone recommend some websites
where people write in-depth articles about music?"* — labeled `recommendation`
because the goal is to get specific suggestions, not to debate a topic.

**`opinion` vs. `recommendation`:**
Some posts share an opinion about an artist while also nudging the reader to
listen to them. Example: *"A band I really enjoy is called Orbiter, and it
saddens me that they aren't more popular!"*

**Rule:** If the post's primary purpose is to point someone toward new music,
label it `recommendation` even if it contains opinion language.

---

## Data Collection Plan

- **Source:** r/LetsTalkMusic public posts and comment threads
- **Target:** 200+ examples, aiming for roughly equal representation per label
- **Collection method:** Manual copy-paste into a CSV spreadsheet with columns:
  `text`, `label`, `notes`
- **Final distribution:**
  - Discussion: 40.6%
  - Recommendation: 30.7%
  - Opinion: 28.7%

If any label was underrepresented after 200 examples (fewer than ~35), the plan
was to search the subreddit specifically for that post type. In practice,
discussion posts were slightly overrepresented, which contributed to the
model's class imbalance problem during training.

---

## Evaluation Metrics

The following metrics were used to evaluate both models:

- **Accuracy:** overall share of correct predictions — useful as a summary
  but insufficient alone since a model could score well by predicting the
  majority class
- **Per-class F1 score:** shows where the model struggles specifically,
  since some labels are harder to classify than others
- **Confusion matrix:** identifies which label pairs the model consistently
  confuses — essential for diagnosing failure modes

Accuracy alone is not enough because a model predicting `discussion` for every
post would score ~40% without learning anything. F1 and the confusion matrix
reveal whether the model has actually learned the boundaries between labels.

---

## Definition of Success

The fine-tuned model should achieve at least **75% overall accuracy** and no
individual label should have an F1 score below **0.65**. This would mean the
model is genuinely useful as a tagging tool — not perfect, but reliable enough
that a community moderator could use it to surface specific post types.

A baseline below 70% accuracy would confirm that fine-tuning added meaningful
value. In practice, the baseline scored 77.4% and the fine-tuned model scored
41.9%, meaning the success criteria were not met. This outcome is documented
and analyzed in the README.

---

## AI Tool Plan

### Label stress-testing
I gave Claude my label definitions and edge case descriptions and asked it to
generate 8–10 posts that sit at the boundary between `opinion` and `discussion`.
This surfaced cases like rhetorical questions dressed up as debates, which
helped tighten the annotation rule before collecting 200 examples.

### Annotation assistance
I used Claude to pre-label batches of 20–30 examples at a time by providing
my label definitions and a set of unlabeled posts. I reviewed and corrected
every pre-assigned label myself. All pre-labeled examples were marked in the
`notes` column with "pre-labeled by LLM."

### Failure analysis
After fine-tuning, I pasted the list of wrong predictions into Claude and asked
it to identify patterns. It flagged that most misclassified posts were short,
ambiguous in tone, and shared surface features across labels — particularly
that recommendation and discussion posts often contain question phrasing that
the model associated with opinion. I verified those patterns by re-reading the
examples myself before writing the evaluation.
