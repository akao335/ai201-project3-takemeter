# Project Planning: r/LetsTalkMusic Post Classifier

## Community

I chose r/LetsTalkMusic because it's a text-heavy community where post quality
varies significantly — some posts are deeply analytical while others are pure
personal reaction. This makes it a strong fit for a classification task because
the distinctions between post types are meaningful to the community itself:
regulars there actively value substantive discussion over low-effort opinions,
so the labels reflect a real social distinction, not an arbitrary one.

## Labels

### `opinion`
A post that expresses a personal feeling or preference about an artist, album,
or song without supporting reasoning or evidence.

- *"Random Access Memories is honestly one of the best albums ever made.
  It just hits different every time I listen to it."*
- *"I can't get into Radiohead no matter how hard I try.
  Thom Yorke's voice just annoys me."*

### `analysis`
A post that examines specific musical elements — production, lyrics, structure,
or genre influences — with concrete details or examples.

- *"What makes Kendrick's TPAB so special is how it uses jazz and funk not just
  as aesthetic choices but as structural metaphors — the live band recording
  gives it an anxious, unresolved tension that mirrors the album's themes."*
- *"Charli XCX's production on Brat leans heavily on hyperpop's distorted 808s
  but strips away the maximalism — the space in the mix is doing as much work
  as the sounds themselves."*

### `recommendation`
A post that primarily suggests music to listen to, with minimal or brief
reasoning.

- *"If you liked Sufjan Stevens' Illinois, you should check out The Microphones
  — very similar folk-orchestral layering."*
- *"Looking for ambient music to study to — someone recommended Brian Eno's
  Music for Airports and it's perfect."*

### `discussion`
A post that poses a question or open-ended topic inviting the community to
debate or share views.

- *"Do you think concept albums are becoming less relevant in the streaming era?
  Hard to experience them as a full work when people shuffle."*
- *"What's a critically acclaimed album you genuinely can't understand
  the hype for?"*

## Hard Edge Cases

**`opinion` vs. `analysis`:** Some posts mention specific musical elements but
don't actually analyze them — they use technical-sounding language to dress up
a preference. Example: *"The guitar work on this album is incredible, every
riff is perfectly placed"* names a musical element but makes no analytical
claim about how or why it works.

**Rule:** If a post names a specific element but doesn't explain how or why it
works, label it `opinion`. Analysis requires a claim, not just a reference.

**`discussion` vs. `opinion`:** Some posts phrase an opinion as a question to
seem like they're inviting debate, but are really just stating a take.
Example: *"Am I the only one who thinks Tyler the Creator is overrated?"*

**Rule:** If the question is rhetorical and the post doesn't genuinely open
multiple sides, label it `opinion`.

## Data Collection Plan

- Source: r/LetsTalkMusic public posts and comment threads
- Target: 200+ examples, aiming for ~50 per label
- Collection method: manual copy-paste into a CSV spreadsheet
- Columns: `text`, `label`, `notes`

If any label is underrepresented after 200 examples (fewer than ~35),
I will search the subreddit specifically for that post type — for example,
searching for "recommend" or "what should I listen to" to find more
recommendation posts.

## Evaluation Metrics

I will use the following metrics:

- **Accuracy:** overall share of correct predictions — useful as a quick
  summary but insufficient alone
- **Per-class F1 score:** since some labels may be harder to classify than
  others, per-class F1 shows where the model struggles specifically
- **Confusion matrix:** to identify which label pairs the model consistently
  confuses (e.g., opinion vs. analysis)

Accuracy alone is not enough because a model could score 60%+ by always
predicting the majority class. F1 and the confusion matrix reveal whether
the model has actually learned the boundaries between labels.

## Definition of Success

The fine-tuned model should achieve at least **75% overall accuracy** and
no individual label should have an F1 score below **0.65**. This would mean
the model is genuinely useful as a tagging tool — not perfect, but reliable
enough that a community moderator could use it to surface analytical posts
or filter low-effort ones. A baseline (zero-shot LLM) below 70% accuracy
would confirm that fine-tuning added meaningful value.

## AI Tool Plan

### Label stress-testing
I will give Claude my label definitions and edge case descriptions and ask it
to generate 8–10 posts that sit at the boundary between `opinion` and
`analysis`. If I can't cleanly classify them, I'll tighten my definitions
before annotating 200 examples.

### Annotation assistance
I will use an LLM to pre-label batches of 20–30 examples at a time by
providing my label definitions and a set of unlabeled posts. I will then
review and correct every pre-assigned label myself before saving it to the
CSV. All pre-labeled examples will be marked in the `notes` column with
"pre-labeled by LLM."

### Failure analysis
After fine-tuning, I will paste my list of wrong predictions into Claude and
ask it to identify patterns — recurring label pairs, post length, sarcasm,
or vague language. I will then verify those patterns by re-reading the
examples myself before writing up my evaluation.