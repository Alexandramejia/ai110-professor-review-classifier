# planning.md: Hunter CS Professor Reviews

# Community

Hunter College Computer Science Professor Reviews

I picked this community because the reviews vary a lot. Some are basically mini course guides, some are just people venting, and some are trying to warn future students away. That variety makes it worth classifying. Reviews come from two places:

- **Rate My Professors (RMP)**: short and structured, usually pretty blunt
- **Reddit (r/HunterCollege)**: longer, more conversational, sometimes buried in a thread

---

# Labels

## helpful_review
Tells you what the class is actually like: how the professor teaches, what the assignments are, how grading works, what the workload feels like. Someone reading it would know what they're signing up for.

Signals: lecture format, assignment types, exam difficulty, grading breakdown, attendance policy.

**Example 1:** "The class is highly structured with weekly programming assignments and quizzes." *(Tiziana Ligorio, RMP)*

**Example 2:** "Saad teaches concepts in an overly complicated way and expects students to solve much harder problems on exams." *(Saad Mneimneh, RMP)*

---

## course_advice
Tells you what to do: take it, avoid it, or how to survive it. The writer's goal is to steer your decision or help you get through the class.

Signals: imperative language ("take this", "avoid", "make sure to"), tips, warnings, forward-looking framing ("if you're planning to take this...").

**Example 1:** "Practice constantly and use multiple resources because Saad's exams require deep understanding of the material." *(Saad Mneimneh, Reddit)*

**Example 2:** "Attend recitations and form study groups because discussing ideas with others can help you understand the material." *(Saad Mneimneh, Reddit)*

---

## emotional_reaction
Mostly just how the writer felt, positive or negative, without much substance. Short, reactive, not much you could actually do with it as a future student.

Signals: strong emotional language ("best professor ever", "horrible", "I loved/hated"), personal experience with no real takeaway, short exclamations.

**Example 1:** "Worst professor ever." *(Tong Yi, RMP)*

**Example 2:** "I cried after failing this class. She is a nice person but not a good teacher." *(Tong Yi, RMP)*

---

# Dataset

- **Total reviews:** 200
- **Professors:** Tiziana Ligorio, Saad Mneimneh, Tong Yi, Eric Schweitzer, Justin Tojeira
- **Label distribution:**
  - `helpful_review`: 96 (48%)
  - `course_advice`: 54 (27%)
  - `emotional_reaction`: 51 (25.5%)
- **Source split:** ~83% RMP, ~17% Reddit
- **Columns:** `professor`, `text`, `label`, `source`

`helpful_review` is almost double the other two classes. That imbalance could cause the model to over-predict it during training, so I'll probably need class weighting.

---

# Hard Edge Cases

Three reviews that were hard to label.

---

**Example 1**
> "Its going to be an uphill battle for sure, theres weekly quizzes, biweekly coding reviews (oral reviews you need to explain), program assignments daily pretty much, oh and final exam that will FAIL YOUR ASS IF YOU DON'T PASS IT LMAO."
> *(Tiziana Ligorio, Reddit)*

This lists real course components (quizzes, coding reviews, programming assignments, final) which sounds like `helpful_review`. But the whole thing is written as a warning, not a description.

**Decision: `course_advice`.** The writer is trying to prepare you, not inform you neutrally. The all-caps at the end makes that clear.

**Rule:** If a review describes the class but clearly frames it as "here's what you're walking into," it's `course_advice`. A `helpful_review` doesn't push you toward or away from anything.

---

**Example 2**
> "Worst course experience. Reads off slides, assigns many programming assignments, and failing the final means failing the class."
> *(Tiziana Ligorio, RMP)*

The opening is pure `emotional_reaction`, but the rest is factual: how she teaches, how grading works.

**Decision: `emotional_reaction`.** The facts are there to back up the complaint, not to help you make a decision. If you stripped the emotional opener, the rest wouldn't stand on its own as useful information.

**Rule:** If the descriptive parts of a review only exist to support an emotional verdict, label it `emotional_reaction`. A real `helpful_review` would still be useful even without the opinion.

---

**Example 3**
> "I failed the class once, but the second time the material finally clicked and Saad was helpful when I had questions."
> *(Saad Mneimneh, RMP)*

Sounds like a personal story (`emotional_reaction`), but it actually tells you something: the material is hard to get at first, and Saad is approachable.

**Decision: `helpful_review`.** The personal experience is being used to describe the class, not just share a feeling. A future student learns something from this.

**Rule:** If a personal story carries real information about the course or professor, it's `helpful_review`. If it's just sharing what happened with no value for anyone else, it's `emotional_reaction`.

---

# Evaluation Metrics

Just using accuracy isn't enough. `helpful_review` is 48% of the data. A model that always guessed that label would hit ~48% accuracy while being completely useless for the other two classes.

What I'm actually tracking:

- **Per-class F1**: one number per label that combines precision and recall. If a class has low F1, the model is basically ignoring it.
- **Overall accuracy**: mostly useful for comparing the fine-tuned model vs. the zero-shot baseline.
- **Confusion matrix**: to see which pairs are getting mixed up. I expect `helpful_review` and `course_advice` to bleed into each other the most since their vocabulary overlaps.

The direction of confusion matters too. The model calling `course_advice` reviews `helpful_review` is a different problem than the reverse.

---

# Definition of Success

For the classifier to be worth using as a first-pass tagger on new reviews:

- **Overall accuracy >= 70%** on the test set
- **Per-class F1 >= 0.60** for all three labels. Anything below that means the model isn't really learning that category.
- **At least +10 points over the zero-shot baseline.** If fine-tuning doesn't beat just prompting a model, it's not worth doing.

If those thresholds are met, the classifier could auto-tag new RMP reviews with a human checking anything the model is uncertain about. If any label's F1 is under 0.50, that category needs more examples or tighter definitions before it's usable.

---

# AI Tool Plan

**Label stress-testing:** I'll ask Claude to generate ~10 reviews on the `helpful_review`/`course_advice` boundary since that's the hardest pair to separate. If any come back ambiguous under my definitions, I'll tighten the decision rule before locking in the dataset.

**Annotation:** I didn't use any LLM to pre-label examples. All 200 were labeled by hand.

**Failure analysis:** After fine-tuning, I'll put the wrong predictions into Claude and ask it to find patterns, things like "short reviews get mislabeled" or "reviews about exams get confused between two labels." I'll check each pattern myself before writing anything up.
