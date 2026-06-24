# r/LetsTalkMusic Post Classifier

## Project Overview

This project fine-tunes a DistilBERT model to classify posts from
r/LetsTalkMusic into three categories: opinion, discussion, and recommendation.
The goal was to build a classifier that could automatically tag posts by type,
which could be useful for community tools like filtering, sorting, or surfacing
specific kinds of content.

The fine-tuned model underperformed the zero-shot baseline, which is documented
and analyzed honestly below. The failure reveals real constraints around data
volume and label design that are worth understanding.

---

## Labels

| Label | Definition |
|---|---|
| `opinion` | Expresses a personal feeling or preference without supporting reasoning |
| `discussion` | Poses a question or open-ended topic inviting community debate |
| `recommendation` | Primarily suggests or asks for music to listen to with minimal reasoning |

---

## Dataset

- **Source:** r/LetsTalkMusic public posts and comments
- **Total examples:** 207
- **Collection method:** Manual copy-paste into CSV, with some examples
  pre-labeled by Claude and reviewed manually
- **Label distribution:**

| Label | Count | Percentage |
|---|---|---|
| discussion | ~84 | 40.6% |
| recommendation | ~64 | 30.7% |
| opinion | ~59 | 28.7% |

---

## Baseline Results (Zero-Shot Groq LLM)

Overall accuracy: **77.4%** (evaluated on 31/31 examples)

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| opinion | 0.71 | 0.56 | 0.62 | 9 |
| discussion | 0.68 | 1.00 | 0.81 | 15 |
| recommendation | 0.50 | 0.14 | 0.22 | 7 |
| **weighted avg** | 0.65 | 0.68 | 0.62 | 31 |

The baseline struggled most with recommendation posts (F1: 0.22), likely
because short recommendation posts look similar to opinions on the surface.
Discussion posts were easiest (F1: 0.81) because their question format is
a strong signal even for a zero-shot model.

---

## Fine-Tuned Model Results

Overall accuracy: **41.9%** (evaluated on 31 examples)

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| opinion | 0.00 | 0.00 | 0.00 | 9 |
| discussion | 0.48 | 1.00 | 0.65 | 15 |
| recommendation | 0.00 | 0.00 | 0.00 | 7 |
| **weighted avg** | 0.23 | 0.48 | 0.32 | 31 |

The fine-tuned model performed significantly worse than the baseline,
regressing by 35.5 percentage points. The model collapsed to predicting
`opinion` for most examples across both runs, achieving 0.00 F1 on both
opinion and recommendation in the final evaluation.

---

## Confusion Matrix (Fine-Tuned Model)

|  | Predicted: opinion | Predicted: discussion | Predicted: recommendation |
|---|---|---|---|
| **True: opinion** | 8 | 1 | 0 |
| **True: discussion** | 7 | 5 | 0 |
| **True: recommendation** | 7 | 3 | 0 |

The confusion matrix shows the model predicted `recommendation` zero times
across all 31 test examples. It correctly identified 8/9 opinion posts and
5/15 discussion posts, but completely failed to learn the recommendation
boundary — every recommendation post was predicted as either opinion or
discussion.

---

## Failure Analysis

### Why the Fine-Tuned Model Failed

**1. Majority class collapse**
The model learned that `discussion` and `opinion` were the most frequent labels
and defaulted to splitting predictions between them, never predicting
`recommendation` at all. This is a classic failure mode when training data is
imbalanced and the model finds it more efficient to predict frequent classes
than to learn subtle boundaries.

**2. Insufficient training data**
With 207 total examples and a 70/15/15 split, the model trained on roughly
145 examples — approximately 40-50 per label. DistilBERT requires substantially
more examples to learn reliable decision boundaries, especially when label
differences are subtle.

**3. Label similarity in text**
The three labels are not visually distinct the way spam vs. not-spam would be.
An opinion post and a discussion post can both be one sentence long, casual in
tone, and about the same topic. Recommendation posts frequently contain question
phrasing ("Can anyone recommend...?") which the model associated with
discussion. The model couldn't find a reliable surface-level signal to separate
the three.

**4. Class weights did not resolve the collapse**
I attempted to apply weighted loss during training to penalize majority-class
predictions more heavily. However, the training loss barely moved across all
epochs (1.105 → 1.103), indicating the model was not learning regardless of
the weighting. This suggests data volume was the primary bottleneck, not
class imbalance alone.

### 3 Specific Wrong Predictions

**Example 1:**
- Text: *"Do you think music is becoming more genreless? I feel like it used
  to be easier to put artists into a specific category..."*
- True label: `discussion`
- Predicted: `opinion` (confidence: 0.35)
- Why it failed: This post opens a genuine debate, but it begins with a
  personal observation that reads like an opinion. The model latched onto
  the observational tone rather than recognizing it as an invitation for debate.
  Low confidence (0.35) confirms it was genuinely uncertain.

**Example 2:**
- Text: *"Any experimental music recommendations? I've been a music fan all
  my life, mostly metal. A couple of years ago I started getting into
  progressive instrumental metal/rock..."*
- True label: `recommendation`
- Predicted: `opinion` (confidence: 0.35)
- Why it failed: The post starts by describing personal listening history,
  which looks like opinion text. The model never learned to recognize the
  recommendation request pattern — it predicted `recommendation` zero times
  in the entire test set.

**Example 3:**
- Text: *"Can anyone recommend some websites where people write articles
  or short essays about music in an in-depth analysis?"*
- True label: `recommendation`
- Predicted: `opinion` (confidence: 0.35)
- Why it failed: Despite containing the word "recommend" explicitly, the model
  still predicted opinion. This confirms the model learned no signal for the
  recommendation class at all — even surface keyword patterns were ignored.

---

## Sample Classifications

| Post (truncated) | True Label | Predicted | Confidence |
|---|---|---|---|
| "Do you think music is becoming more genreless?..." | discussion | opinion | 0.35 |
| "Do songs become classics because they're great, or because enough time passes?..." | discussion | opinion | 0.35 |
| "Anyone have immense trouble finding music you like?..." | recommendation | opinion | 0.35 |
| "A band I really enjoy is called Orbiter, it saddens me they aren't more popular..." | recommendation | opinion | 0.35 |
| "Whose history is classical music? Why are there almost no women in music history?..." | discussion | opinion | 0.36 |

Every prediction carried near-identical low confidence (~0.35-0.36). This
uniformity is itself a signal — a well-trained model would show higher
confidence on clear examples and lower confidence on ambiguous ones. Flat
confidence across all examples confirms the model never learned a meaningful
decision boundary.

---

## Reflection: What the Model Captured vs. What I Intended

I intended the model to learn the distinction between someone sharing a
personal take, someone asking the community a question, and someone pointing
others toward new music. What the model actually captured was much simpler:
it learned to predict the two most common labels and ignored recommendation
entirely.

The gap between intention and learned behavior came down to two things. First,
the labels are semantically subtle — the difference between a recommendation
post and a discussion post often comes down to intent rather than surface
wording, which is not enough signal for a model with so little training data.
Second, 145 training examples is simply not enough for DistilBERT to learn
three-way classification on a nuanced social media task.

A future version of this classifier would need at least 300-400 examples per
label, stricter label definitions that produce more surface-level distinguishable
text, and possibly a stronger pretrained model with more domain knowledge of
informal writing. Alternatively, reducing to two labels (e.g. opinion vs.
discussion only) would give the model a simpler boundary to learn from the
same data volume.

---

## Spec Reflection

**One way the spec helped:** The requirement to define a hard edge case before
annotating forced careful thinking about the opinion vs. discussion boundary
early. The rule — "if a post phrases an opinion as a rhetorical question, label
it opinion" — kept annotation consistent even when posts were ambiguous and
likely reduced label noise in the training data.

**One way my implementation diverged:** The spec assumes fine-tuning will
outperform the baseline. In this case it did not, which required a different
kind of analysis — diagnosing why the model failed rather than explaining why
it succeeded. This was ultimately more instructive than a clean result would
have been, because it surfaced real constraints around data volume and label
design that a successful run might have hidden.

---

## AI Usage

**1. Label stress-testing**
Claude was given the label definitions and asked to generate boundary cases
between opinion and discussion. It produced examples like rhetorical questions
dressed as debates, which helped write the annotation rule that a question
inviting only agreement or disagreement counts as opinion, not discussion.
The generated examples were reviewed and two were added to planning.md as
illustrative edge cases.

**2. Annotation assistance**
Claude was used to pre-label batches of 20-30 examples by providing label
definitions and unlabeled posts. Every pre-assigned label was reviewed and
corrected manually. Pre-labeled examples were marked in the notes column of
the CSV. Roughly 30% of pre-labels were corrected, particularly on the
opinion vs. recommendation boundary.

**3. Failure analysis**
The list of wrong predictions was pasted into Claude with a request to identify
patterns. It flagged that most misclassified posts shared question phrasing or
personal observation language regardless of their true label, and that
recommendation posts in particular had no distinctive surface features the
model could latch onto. These patterns were verified by re-reading the examples
— the observation held up and is reflected in the failure analysis above.

# Demo Link
https://www.loom.com/share/937a3c0b8a884f5292965e6354643190