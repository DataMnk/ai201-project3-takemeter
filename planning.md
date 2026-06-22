# TakeMeter — Planning Document

## Community

Going with r/AmItheAsshole for this one. People post a conflict, ask if they were wrong, and the comments respond in pretty different modes depending on the commenter. Some just drop a verdict (NTA, YTA, whatever) with a reason attached. Some are basically just reacting, no real stance taken. And some skip the whole judgment thing and just tell the poster what to do next. That's a real distinction, not something I'm making up — the subreddit literally has a voting guide that trains people to lead comments with NTA/YTA/ESH/NAH, so there's already a built-in sense in the community of what counts as "delivering a verdict" vs everything else.

## Labels

Going with three:

- **judgment** — comment gives a clear verdict (NTA/YTA/ESH/NAH or basically the same idea stated plainly) and backs it with a reason tied to what actually happened in the post.
  - "NTA. Your roommates are very immature, and they lost the right to use your things because they were not responsible."
  - "YTA. Look at it from your parent's point of view. They wanted to remember a special night for you and then posted a picture of you that you didn't like. How controlling of you."

- **reaction** — emotional response, a clarifying question, a personal story, just an observation. No verdict, no real reasoning behind a stance.
  - "And fuck cancer!"
  - "INFO: Did you ask them outright for the schedule?"

- **advice** — the comment's main thing is telling the poster what to do going forward, not judging what already happened.
  - "Try billing them both since neither one will confess."
  - "Tell your husband he needs to deal with this with you NOW! Or it's going to snowball and they will run right over you."

## Hard edge case I'm anticipating

The one I already know is going to be annoying: comments that open with a verdict tag and then tack on a suggestion. Like "NTA, but you need to find some new roommates when you can." That's half judgment, half advice.

My rule going in: if it opens with an explicit verdict tag, it's judgment, even if a suggestion follows. The verdict is the main move, the suggestion is just extra. I'll only call something advice if there's no tag and the whole point of the comment is forward-looking action.

Second thing I expect to run into: "INFO:" style questions. Those don't take a position at all, so my plan is to call those reaction — it's a response to the post, not a verdict and not really a plan either.

## Data collection plan

Plan is to manually copy top-level comments off big AITA threads — posts with a lot of engagement so there's enough comment variety to pull from. Going to spread it across different kinds of conflicts (roommates, family stuff, relationships, money, whatever comes up) so the model isn't just learning one flavor of conflict. Skipping anything from the OP themselves, the AutoMod bot, and deleted/empty comments, obviously.

If I get to 200 examples and one label is way underrepresented, the plan is to go find more posts specifically with threads that lean toward that label — like, if advice is light, target posts where people are clearly telling the OP what to do (addiction situations, stuff where the OP needs to confront someone directly, money disputes), since those tend to pull a lot of advice-style comments.

## Evaluation metrics

Accuracy alone isn't going to tell me enough here, mostly because the classes probably won't be perfectly even, so a model that just guesses the majority class a lot could still post a decent-looking accuracy without actually learning anything. So I'm planning to look at:
- overall accuracy, just to compare the two models head to head
- per-class precision/recall/F1, since that's what'll actually show me if one label is failing even when the overall number looks okay
- confusion matrix, to see which labels are getting mixed up and in which direction

## Definition of success

For this to actually be useful, I'd want per-class F1 at least 0.70 across all three labels — a classifier that silently fails on one of three categories isn't something I'd trust for anything real. Bare minimum "good enough to call it working" would be no class sitting at F1 = 0, since that at least means the model is trying to learn all three distinctions instead of just defaulting to whichever label shows up most.

## AI Tool Plan

- **Label stress-testing**: planning to ask Claude to generate some comments that sit right on the boundary between judgment and advice, before I commit to annotating everything. If it spits out stuff I genuinely can't classify cleanly, that's my signal to tighten the definitions first.
- **Annotation assistance**: planning to have an LLM pre-label batches of real comments I've copied over, using my label definitions. I'll review every single label myself though, not just accept whatever it suggests — that'll get disclosed properly in the README's AI usage section.
- **Failure analysis**: once I've got wrong predictions from the fine-tuned model, plan is to hand that list to Claude and ask if it sees a pattern before I write up my own analysis. Then I go back and actually check the pattern myself against the real examples instead of just taking its word for it.
