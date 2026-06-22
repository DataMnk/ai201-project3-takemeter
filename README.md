# TakeMeter — r/AmItheAsshole Discourse Classifier

Fine-tuned text classifier that labels top-level comments on r/AmItheAsshole as **judgment**, **reaction**, or **advice**, compared against a zero-shot Groq baseline (llama-3.3-70b-versatile).

## Community

Picked r/AmItheAsshole because the comments split into pretty distinct modes depending on who's responding. Some people just deliver a verdict, some are basically just reacting with no real stance, some skip judgment entirely and tell the poster what to actually do. This isn't something I invented — the subreddit's own voting guide already trains people to lead with NTA/YTA/ESH/NAH, so "verdict vs not" is a category that already exists in how this community talks.

## Label Taxonomy

| Label | Definition | Example |
|---|---|---|
| **judgment** | Clear verdict (NTA/YTA/ESH/NAH or basically that) with a reason tied to what happened in the post. | "NTA. Your roommates are very immature, and they lost the right to use your things because they were not responsible." |
| **reaction** | Emotional response, clarifying/INFO question, personal story, observation — no verdict, no real reasoning. | "And fuck cancer!" |
| **advice** | Comment's main point is telling the poster what to do going forward, not judging what already happened. | "Try billing them both since neither one will confess." |

**The edge case rule:** comment opens with a verdict tag then adds a suggestion ("NTA, but you need to find some new roommates") → that's judgment, since the verdict is the main move and the suggestion is just extra. Advice only applies when there's no tag and the whole comment is forward-looking.

## Data Collection & Labeling

- **Source:** top-level comments copied manually off 14 different AITA posts, big threads with lots of engagement, covering roommates, family, relationships, parenting, friendships, work, money.
- **Process:** skipped OP replies, the AutoMod bot, deleted/empty comments. Read each one against the definitions instead of skimming in bulk. Used Claude to help pre-label batches, but reviewed and could override every single label — details in AI Usage below.
- **Final dataset:** 312 examples — judgment: 147 (47%), reaction: 85 (27%), advice: 80 (26%). Nothing over 70%.
- **Why it grew from 200 to 312:** the first pass at 200 examples (judgment 100 / reaction 54 / advice 46) technically passed the "no label over 70%" check, but when I fine-tuned on it, advice completely collapsed — F1 of 0.00, the model basically never predicted it correctly. Turns out 32 training examples for that class just wasn't enough to learn anything from. So I went back and collected 112 more comments, specifically targeting posts where the comment threads lean hard into "here's what you should do" territory, which got advice up to 80 examples total.

### Three difficult labeling calls

1. *"NTA, but you need to find some new roommates when you can."* — Right on the line between judgment and advice. Went with judgment, since the verdict tag is doing the main work and the suggestion is secondary. Became the general rule for anything similar.
2. *"INFO: Did you ask them outright for the schedule?"* — No verdict, no recommendation, just a question. Called it reaction, since it's responding to the post but not really taking a position or offering a plan.
3. *"Did their best? No. Not true. Next conversation is why did they ghost you?"* — This one reads with a really judgmental tone but never actually states a verdict, no NTA/YTA, no explicit "you're right/wrong." Went with reaction to stay consistent with the rule (explicit stance, not just vibe), but honestly this is the exact kind of comment the model ends up getting wrong later (see error analysis below), so in hindsight my rule might be a little too strict here.

## Fine-Tuning Approach

- **Base model:** distilbert-base-uncased
- **Split:** 70/15/15 (train 218, val 47, test 47), stratified.
- **Hyperparameter change:** default is 3 epochs. After the first run on the 200-example set showed advice totally collapsing, I bumped num_train_epochs to 6 — figured with not that many examples per class, the model needed more passes to actually pick up the minority class pattern at all. Left learning rate (2e-5) and batch size (16) at the defaults.
- This ended up being a two-step thing: more epochs alone on the 200-example set got accuracy from 60% to 66.7%, then more epochs plus the bigger 312-example dataset got it to 72.3%, and advice's F1 went from 0.00 all the way to 0.70.

## Baseline

- **Model:** llama-3.3-70b-versatile via Groq, zero-shot, no examples given beyond what's in the prompt.
- **Prompt:** same label definitions and examples used for my own annotation, includes the verdict-tag decision rule, tells the model to output only the label name.
- **Collection:** ran all 47 test comments through individually at temperature 0, all 47 came back parseable.

## Evaluation Report

### Overall accuracy

| Model | Accuracy |
|---|---|
| Zero-shot baseline (Groq llama-3.3-70b) | **85.1%** |
| Fine-tuned DistilBERT | **72.3%** |

So the fine-tuned model is behind by about 13 points. Not really a surprise honestly — distilbert fine-tuned on ~218 examples is working with way less signal than a 70B model that already has a strong general sense of tone and intent baked in from pretraining. The interesting part isn't that it lost, it's how much it improved once advice actually had enough data to learn from.

### Per-class metrics

**Fine-tuned DistilBERT**

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| judgment | 0.692 | 0.818 | 0.750 | 22 |
| reaction | 0.692 | 0.692 | 0.692 | 13 |
| advice | 0.875 | 0.583 | 0.700 | 12 |

**Zero-shot baseline (Groq)**

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| judgment | 0.80 | 0.91 | 0.85 | 22 |
| reaction | 0.86 | 0.92 | 0.89 | 13 |
| advice | 1.00 | 0.67 | 0.80 | 12 |

### Confusion matrix (fine-tuned, test set)

| True \ Predicted | judgment | reaction | advice |
|---|---|---|---|
| **judgment** | 18 | 3 | 1 |
| **reaction** | 4 | 9 | 0 |
| **advice** | 4 | 1 | 7 |

### Three wrong predictions, actually analyzed

**1.** *"Yes, that is what I would have done as soon as I knew there was one. The family here makes no sense whatsoever that they'd create a schedule and not share it, then passive aggressively get angry later..."*
True: judgment → Predicted: reaction (confidence 0.45)
There's no NTA/YTA tag anywhere in here, but the comment clearly takes a side ("the family here makes no sense") — by my own logic that's a stance, just delivered through narrative agreement instead of a tag. The model seems to be keying off the missing verdict marker and defaulting to reaction. Kind of a labeling inconsistency on my end too, honestly — I labeled this one judgment based on tone, but my rule technically wants a tag.

**2.** *"It says a lot that her fiancé had to tell you not to come. A proposal is not a family event. It's between the couple. Why would you be there? Why did you need to be told not to be there?"*
True: judgment → Predicted: reaction (confidence 0.43)
This is basically all rhetorical questions, and the model seems to strongly associate question-shaped comments with reaction (makes sense, since INFO questions are reaction by definition). But here the questions are doing judgmental work, implying a verdict without ever stating one outright. The model can't really tell the difference between a rhetorical question that's making a point and an actual clarifying question.

**3.** *"The main character energy is absolutely wild. She really tried to make a literal cancer diagnosis all about her financial drama. OP needs to drop her and those enabling friends ASAP."*
True: advice → Predicted: judgment (confidence 0.48)
First two sentences are pure commentary, no verdict, just describing what the other person did. Only the last clause ("OP needs to drop her... ASAP") is actually advice. Looks like the model weighted the dominant tone of the comment over that final actionable bit, and read commentary-heavy advice as judgment instead.

**The pattern:** all three of these fail because the comment implies its stance through tone, rhetorical structure, or narrative description instead of an explicit verdict word or a clean direct command. The model does fine when the structure is simple — tag plus reason, or straight-up imperative — and falls apart when the actual function of the comment is buried in phrasing rather than an obvious marker.

### Sample Classifications

| Comment | Predicted | Confidence |
|---|---|---|
| "NTA. Your roommates are very immature, and they lost the right to use your things." | judgment | 0.91 |
| "And fuck cancer!" | reaction | 0.88 |
| "Try billing them both since neither one will confess." | advice | 0.79 |
| "INFO: Did you ask them outright for the schedule?" | reaction | 0.82 |
| "Did their best? No. Not true. Next conversation is why did they ghost you?" | judgment | 0.34 |

The first one gets predicted correctly with high confidence because it's exactly the structural pattern the model saw the most of in training — verdict tag immediately followed by a reason. That combo is probably the cleanest, most repeated signal in the whole dataset, so it's not surprising the model latched onto it hard.

## Reflection: What the Model Learned vs. What I Meant For It to Learn

I wanted the model to pick up on communicative function — is this comment delivering a verdict, just reacting, or recommending something — no matter how it's phrased. What it actually picked up on looks a lot more like surface markers: is there a verdict abbreviation somewhere, is there a question mark, is there an imperative verb near the end. Honestly that's a reasonable shortcut for a model that's only seen a few hundred examples, but it's exactly why it breaks on the comments where my own labeling required judgment calls beyond just spotting a marker — stuff that takes a stance through tone or rhetorical question instead of an explicit tag. The advice class improving the most once I added data (0.00 to 0.70 F1) kind of confirms that the earlier failure wasn't because advice is some uniquely hard concept, it was just that there wasn't enough of it around for the model to find any pattern at all.

## Spec Reflection

The "read 30-40 posts before locking in your labels" advice from the spec was actually useful — the verdict-tag rule only showed up after seeing real "NTA, but..." comments in the wild, not from just thinking about the taxonomy on paper. Where I ended up diverging from the spec: it kind of assumes one annotation pass is enough, but my first 200-example pass produced a class the model literally couldn't learn at all. The spec treats error pattern analysis as an optional stretch goal, but in practice I had to do a rough version of that after the very first training run just to figure out if my dataset itself was broken, way before I got to the kind of analysis the spec describes for the final writeup.

## AI Usage

1. **Label stress-testing (Claude):** before annotating anything, asked Claude to generate AITA-style comments sitting right on the boundary between judgment and advice. That's what surfaced the "NTA, but..." pattern and led to the verdict-tag-wins rule.
2. **Annotation assistance (Claude):** copied real comment threads off AITA posts and asked Claude to label them against my definitions. Reviewed every suggested label against the actual text and overrode a bunch — a few "INFO:" questions got suggested as judgment and I changed them to reaction, and some sarcastic one-liners suggested as reaction got bumped to judgment where they actually carried an implicit clear verdict.
3. **Failure diagnosis (Claude):** after the first training run came back with advice F1 = 0.00, described the per-class numbers to Claude and asked what might be going on. It pointed at the class being underrepresented (32 of 140 training examples) rather than the label definition itself being broken. Checked that against the actual support numbers in the report before deciding to go collect more advice-heavy comments instead of redefining the label.
4. **Groq prompt drafting (Claude):** had Claude draft the zero-shot system prompt for the baseline, based on the label definitions from this doc. Reviewed it against my own definitions before using it — it matches the taxonomy as written above.

## Files in this repo

- `planning.md` — design doc written before collecting data
- `takemeter_dataset.csv` — final labeled dataset (312 examples: text, label, notes)
- `evaluation_results.json` — final accuracy numbers for both models
- `confusion_matrix.png` — confusion matrix for the fine-tuned model
- `README.md` — this file

##Video
https://www.loom.com/share/41701cac019f454cac120e8ca849a9e3
