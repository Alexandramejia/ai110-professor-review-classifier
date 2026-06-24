# TakeMeter — Hunter CS Professor Review Classifier

A fine-tuned text classifier that sorts Hunter College CS professor reviews into three categories: helpful descriptions of the course, advice for future students, or emotional reactions. Built on DistilBERT, compared against a Groq zero-shot baseline.

---

## Where the Data Came From

Reviews were collected from Rate My Professors and Reddit (r/HunterCollege) for five Hunter CS professors. The thing that made this dataset interesting is how different the reviews actually are — some people write detailed breakdowns of a course, some are trying to warn the next person, and some are just venting. That variation is what makes classification worth doing.

**Professors covered:** Tiziana Ligorio, Saad Mneimneh, Tong Yi, Eric Schweitzer, Justin Tojeira

---

## Labels

**helpful_review** — The review describes the professor or course in a way that tells you what the class is actually like: teaching style, assignments, grading, workload.

- "The class is highly structured with weekly programming assignments and quizzes."
- "Saad teaches concepts in an overly complicated way and expects students to solve much harder problems on exams."

**course_advice** — The review is giving you direct advice or a warning. The main point is to help you decide whether to take it or survive it.

- "Practice constantly and use multiple resources because Saad's exams require deep understanding of the material."
- "Attend recitations and form study groups because discussing ideas with others can help you understand the material."

**emotional_reaction** — The review is mostly about how the person felt. Little actual information, just a reaction.

- "Worst professor ever."
- "I cried after failing this class. She is a nice person but not a good teacher."

---

## Dataset

- **Total examples:** 200
- **Sources:** Rate My Professors (~83%), Reddit r/HunterCollege (~17%)
- **Columns:** `professor`, `text`, `label`, `source`

| Label | Count | % |
|---|---|---|
| helpful_review | 96 | 48% |
| course_advice | 54 | 27% |
| emotional_reaction | 51 | 25.5% |

All 200 examples were labeled manually. No LLM pre-labeling.

**Three hard cases:**

1. *"Its going to be an uphill battle for sure, theres weekly quizzes, biweekly coding reviews (oral reviews you need to explain), program assignments daily pretty much, oh and final exam that will FAIL YOUR ASS IF YOU DON'T PASS IT LMAO."* — Lists real course structure, but the framing is a warning. **Labeled `course_advice`.**

2. *"Worst course experience. Reads off slides, assigns many programming assignments, and failing the final means failing the class."* — Opens with a strong negative verdict; the details that follow exist to support a complaint. **Labeled `emotional_reaction`.**

3. *"I failed the class once, but the second time the material finally clicked and Saad was helpful when I had questions."* — Personal experience, but it tells you something real about the course's learning curve. **Labeled `helpful_review`.**

---

## Training

- **Base model:** `distilbert-base-uncased`
- **Platform:** Google Colab (T4 GPU)
- **Split:** 70% train / 15% val / 15% test (stratified)
- **Epochs:** 3, **LR:** 2e-5, **Batch size:** 16

**One thing worth noting:** The first training run predicted `helpful_review` for everything, because it was 48% of the data and the model just defaulted to it. To fix this, class weights `[1.0, 1.8, 1.9]` were applied using a custom `WeightedTrainer` with `CrossEntropyLoss`. That pushed `course_advice` F1 from 0.00 to 0.62.

---

## Baseline

The zero-shot baseline used Groq's `llama-3.3-70b-versatile` with a prompt that included label definitions and one example per class. All 31 test examples were parseable. Run at `temperature=0`.

Prompt used:

```
You are classifying student reviews of Hunter College CS professors.
Assign each review to exactly one of the following categories.

helpful_review: The review describes the professor's teaching style, assignments,
grading, workload, or course structure. A reader would walk away knowing what the
class is actually like.
Example: "The class is highly structured with weekly programming assignments and quizzes."

course_advice: The review gives direct recommendations or warnings to future students.
The main point is to help someone decide whether to take the class or tell them how
to survive it.
Example: "Practice constantly and use multiple resources because Saad's exams require
deep understanding of the material."

emotional_reaction: The review primarily expresses how the writer felt without much
informational or advisory content.
Example: "Worst professor ever."

Respond with ONLY the label name. Do not explain your reasoning.
```

---

## Results

### Accuracy

| Model | Accuracy |
|---|---|
| Zero-shot baseline (Groq) | 71.0% |
| Fine-tuned DistilBERT | 61.3% |

The fine-tuned model underperformed the baseline by ~10 points. The main reason: `emotional_reaction` was never predicted — 0 times — despite the class weights.

### Per-Class Metrics

**Baseline (Groq):**
| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| helpful_review | 0.75 | 0.80 | 0.77 | 15 |
| course_advice | 0.73 | 1.00 | 0.84 | 8 |
| emotional_reaction | 0.50 | 0.25 | 0.33 | 8 |

**Fine-tuned DistilBERT:**
| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| helpful_review | 0.58 | 1.00 | 0.73 | 15 |
| course_advice | 0.80 | 0.50 | 0.62 | 8 |
| emotional_reaction | 0.00 | 0.00 | 0.00 | 8 |

### Confusion Matrix (Fine-Tuned)

| | Predicted: helpful_review | Predicted: course_advice | Predicted: emotional_reaction |
|---|---|---|---|
| **True: helpful_review** | 15 | 0 | 0 |
| **True: course_advice** | 4 | 4 | 0 |
| **True: emotional_reaction** | 7 | 1 | 0 |

### Error Analysis

**"Attend lectures and seek help when needed because resources and office hours are readily available."**
True: `course_advice` → Predicted: `helpful_review`

The word "lectures" and "resources" show up a lot in `helpful_review` training examples. The imperative structure ("attend", "seek help") is the real signal for `course_advice`, but the model keyed on content words instead. Not enough training examples to learn that distinction.

**"The professor changed the final exam format at the last minute and many students struggled because of it."**
True: `emotional_reaction` → Predicted: `helpful_review`

This reads as a factual statement even though the intent is to complain. The model can't detect intent from surface text alone, and factual phrasing pushed it toward `helpful_review`. This is the hardest boundary in the dataset.

**"Very rude professor. 0 respect for students. Avoid taking."**
True: `emotional_reaction` → Predicted: `helpful_review`

Clear emotional reaction, but predicted `helpful_review` with 0.36 confidence — basically a random guess. The model never learned this class at all.

### Sample Predictions

| Text | True | Predicted | Confidence |
|---|---|---|---|
| "The class is highly structured with weekly programming assignments and quizzes." | helpful_review | helpful_review | 0.41 |
| "Saad genuinely cares about students and is willing to accommodate them when asked." | helpful_review | helpful_review | 0.40 |
| "Do all the homework, study heavily for exams, and meet with Saad for clarification if you want to improve your grade." | course_advice | course_advice | 0.38 |
| "Many students retake this course, so consider studying far harder than you would for most classes." | course_advice | course_advice | 0.39 |
| "This class was so stressful that it affected my mental health." | emotional_reaction | helpful_review | 0.37 |

---

## What the Model Actually Learned

The model got `helpful_review` right consistently and `course_advice` partially right after class weighting. It never predicted `emotional_reaction` once.

The problem is that `emotional_reaction` doesn't always look the way you'd expect. I defined it as reviews that "primarily express how the writer felt," but a lot of emotional reviews in this dataset use factual-sounding language — they describe what happened rather than saying "I felt X." A review like "The professor changed the final exam format at the last minute" reads like information even though the writer is venting. The model can't tell the difference.

What I wanted the model to learn was the *intent* behind a review. What it learned was vocabulary patterns. With 200 examples and 3 epochs, there wasn't enough signal for the subtler distinction.

The fix would be more `emotional_reaction` examples that use descriptive language, and tighter annotation guidelines for that specific boundary.

---

## Spec Reflection

The spec helped most during label design. Having to name a hard edge case before annotating forced a decision rule early: "if the post centers on telling the reader what to do, it's `course_advice`." That made annotation noticeably faster.

Where things diverged: the spec expects the fine-tuned model to meaningfully beat the baseline. Mine didn't. After class weighting fixed the worst of the imbalance problem, `emotional_reaction` still came out at F1 = 0.00. In hindsight, collecting more `emotional_reaction` examples that use descriptive surface language would have helped more than adjusting weights — the easy ones like "Worst professor ever" don't represent where the real boundary is.

---

## AI Usage

**Instance 1 — Groq classification prompt:** Asked Claude to draft a classification prompt based on my label definitions from planning.md. The structure worked, but I replaced all examples with ones from my actual dataset and adjusted the definitions to match my wording exactly.

**Instance 2 — Edge case identification:** After realizing my original hard edge cases weren't actually that hard, I asked Claude to find reviews from the CSV that created real labeling conflicts. It flagged three cases where surface language and intent pointed in different directions. I used those and wrote the decision rules myself based on how I had actually labeled them.
