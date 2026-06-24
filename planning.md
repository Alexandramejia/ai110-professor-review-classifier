# Community
Hunter College Computer Science Professor Reviews

# Sources
- **Rate My Professors (RMP)** — structured review platform; reviews tend to be brief and opinion-forward
- **Reddit (r/HunterCollege)** — freeform discussion; reviews are often longer, conversational, and context-rich

# Dataset

- **Total reviews:** 200
- **Professors covered:** 5 Hunter CS faculty
- **Label distribution:**
  - `helpful_review`: 96 (48%)
  - `course_advice`: 54 (27%)
  - `emotional_reaction`: 51 (25.5%)
- **Source split:** ~83% RMP, ~17% Reddit
- **Columns:** `professor`, `text`, `label`, `source`

Note: the dataset is moderately imbalanced — `helpful_review` appears nearly twice as often as the other two classes. This may require class weighting or oversampling during training.

# Labels

## helpful_review
Reviews that describe a professor's teaching style, assignments, grading, workload, or course structure. These are informational and descriptive — a student reading them would learn what the class is actually like.

Example signals: mentions of lecture format, assignment types, exam structure, grading criteria, attendance policy, pacing.

## course_advice
Reviews that provide direct recommendations or warnings to future students. The writer's intent is to guide a decision (take or avoid this class/professor).

Example signals: imperative language ("take this class", "avoid", "make sure to"), forward-looking framing ("if you're planning to", "for future students"), tips and strategies.

## emotional_reaction
Reviews that primarily express how the writer felt about the professor or course, without providing much actionable or descriptive content.

Example signals: strong positive/negative language ("best professor ever", "horrible", "I loved/hated"), short reactions without elaboration, exclamations.

# Goal
Train a multi-class text classifier that predicts whether a review is a `helpful_review`, `course_advice`, or `emotional_reaction`.
