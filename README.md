# LLMs Shortcut Their Tasks

**Task substitution in Qwen3.5-4B: when a model answers the familiar question instead of the one it was asked.**

Eren Alp · Fatima Fellowship, Fall 2026 · "Blind Spots of Frontier Models" challenge

- All data, raw model outputs, labels and figures:
  [Hugging Face bucket `Erenko-13/llm-task-shortcutting`](https://huggingface.co/buckets/Erenko-13/llm-task-shortcutting)
- Code repository: [GitHub `Erenko13/llm-task-shortcutting`](https://github.com/Erenko13/llm-task-shortcutting)
- Code: [`task_shortcutting_eval.ipynb`](task_shortcutting_eval.ipynb) (generation and judging, Colab A100) and
  [`analyze_results.ipynb`](analyze_results.ipynb) (analysis, CPU)

## The idea in one paragraph

When a prompt looks like a familiar task but differs in one key detail, a language model often solves the familiar,
easier task instead. The answer looks like a hallucination, but it isn't made up. It is a correct answer to a
different question: the original one. I tested this on Qwen3.5-4B with 7,578 graded responses on two public
benchmarks of original vs. minimally modified problems (altered riddles and GSM-Plus math), with thinking off and
on.

**Short answer:** yes for riddles, where the model has a strong memorised answer to fall back on; no for arithmetic
changes, where it recomputes. A broader form of the same thing, turning an unanswerable problem into an answerable
one, shows up in math too.

| setting | verdict |
|---|---|
| Altered riddles, thinking on | ✅ **Supported** |
| Altered riddles, thinking off | ⚠️ **Partially supported** |
| GSM-Plus, thinking on | ❌ **Not supported** (except for problems with a missing fact) |
| GSM-Plus, thinking off | ❌ **Not supported** (except for problems with a missing fact) |

## Q1 · The blind spot

**Task substitution: the model answers the familiar question instead of the one it was asked.**

When a prompt closely resembles a task the model has seen many times, but one detail that changes the answer is
different, the model often solves the familiar version and returns *its* answer. From the outside this looks like a
hallucination: a confident, wrong answer. But nothing is made up. It is the correct answer to a different, easier
question, namely the original one.

I kept running into this when using LLMs: answers that looked hallucinated often turned out to be a correct answer to
a more common, slightly different version of what I had asked. The model had quietly made my task easier.

Why I think it is a blind spot:

- **Benchmarks don't see it.** They score answers as right or wrong. A substituted answer counts as one more error,
  indistinguishable from a calculation mistake or a hallucination, so the failure mode itself is never measured.
- **The usual fixes for hallucination don't target it.** Retrieval, more knowledge and calibration all assume the
  model is missing something. Here it isn't: it applies knowledge it has to the wrong question, and knowing the
  original actually makes it *more* likely to fail (see below).
- **It is silent.** The model rarely says it assumed the usual version, so a user has no signal to double-check.
- **In real use, the changed detail is often the whole point:** an unusual clause in a standard contract, the edge
  case in a familiar piece of code, a patient who differs from the textbook case.

## What I did

**Model.** [`Qwen/Qwen3.5-4B`](https://huggingface.co/Qwen/Qwen3.5-4B) (post-trained, Apache-2.0, used text-only),
once with thinking off and once with thinking on. Sampling follows Qwen's recommendations for general tasks
(thinking off: temperature 0.7, top-p 0.8, max 4,096 tokens; thinking on: temperature 1.0, top-p 0.95, max 16,384
tokens; top-k 20 and presence penalty 1.5 in both). One sample per prompt.

**Data.** Two public English benchmarks where each problem comes as an original and a minimally modified version:

| dataset | items | what changes |
|---|---|---|
| [`marcodsn/altered-riddles`](https://huggingface.co/datasets/marcodsn/altered-riddles) | 700 altered riddles from 189 well-known originals | a constraint is added, the meaning shifts, the context is swapped, or a stereotype is probed, so that the classic answer no longer fits |
| [`qintongli/GSM-Plus`](https://huggingface.co/datasets/qintongli/GSM-Plus) | 150 GSM8K problems × 3 types = 450 items | a needed fact is removed ("made unanswerable"), the question is reversed, or the numbers are changed |

Every item stores two answers: `gold`, the answer to the modified problem, and `shortcut`, the original problem's
answer. Example: in "Janet's ducks lay 16 eggs per day … how much does she make?" the answer is 18. Delete the 16
and the correct answer becomes "cannot be determined"; answering 18 is substitution.

**Four conditions.** Each prompt is a fresh chat:

| condition | what the model sees | what it tests |
|---|---|---|
| `canonical` | the original problem | does it know the familiar version? |
| `perturbed` | the modified problem | **the main test:** does it notice the change? |
| `nudge` | a generic "read carefully, this may differ from a well-known problem" + the modified problem | does a warning help? |
| `contrast` | original and modified side by side, "solve only the modified one" | can it solve it when the change is visible? |

**Grading.** [`Qwen/Qwen3.5-9B`](https://huggingface.co/Qwen/Qwen3.5-9B) acts as judge (greedy, no thinking) and
labels each final answer `correct`, `shortcut` (= the original's answer), other, or no answer. A rule-based grader
(number matching for math, accepted-answer lists for riddles) checks it independently. Responses that hit the token
limit count as "no answer".

**Metrics.**

| metric | meaning |
|---|---|
| accuracy | share of `correct` answers in a condition |
| shortcut rate | share of `perturbed` answers that are exactly the original's answer |
| COR | shortcut rate counted only on items whose original the model solved (the dataset's "conditioned override rate") |
| substitution share of errors | of the wrong answers to `perturbed`, the share that is the original's answer. **The core evidence:** random hallucination would scatter, substitution lands on one specific wrong answer |
| recovery | accuracy in `nudge` / `contrast` minus accuracy in `perturbed` |
| silent | share of shortcut answers that never mention anything being different |

Confidence intervals (in brackets) are 95% cluster bootstrap intervals over original problems, 2,000 resamples.

## Results

| setting | acc canonical | acc perturbed | acc nudge | acc contrast | shortcut rate | COR | substitution share of errors |
|:--|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| Altered riddles, thinking off | 37% | 34% [30%, 38%] | 35% | 38% | 18% | 29% | 30% [25%, 35%] |
| Altered riddles, thinking on | 67% | 44% [40%, 49%] | 50% | 52% | 27% | 34% | **50% [44%, 56%]** |
| GSM-Plus, thinking off | 96% | 72% [69%, 75%] | 72% | 78% | 5% | 5% | 22% [14%, 31%] |
| GSM-Plus, thinking on | 96% | 83% [80%, 86%] | 82% | 91% | 3% | 3% | 18% [10%, 28%] |

![Answers to the modified problem, by modification type](https://huggingface.co/buckets/Erenko-13/llm-task-shortcutting/resolve/analysis/outcome_mix.png)

![Accuracy by condition with 95% CIs](https://huggingface.co/buckets/Erenko-13/llm-task-shortcutting/resolve/analysis/accuracy_by_condition_ci.png)

## What the results show

1. **On riddles, substitution is the main failure.** With thinking on, Qwen3.5-4B solves 67% of the original riddles
   but only 44% of the altered ones. Half of its errors (50% [44%, 56%]) are exactly the original riddle's answer,
   and 84% of these answers never mention that anything was changed. With thinking off the model knows fewer of the
   originals (37%), so there is less to substitute, but 30% of its errors are still the original's answer.

2. **Knowing the original is what causes it.** On altered riddles the model gives the original's answer about 2.5
   times as often when it solved that original correctly: 34% vs 13% with thinking on, 29% vs 12% with thinking off.
   That is what substitution predicts and random error doesn't. Riddles written by Qwen3.5 models and by other models
   give the same rates (27% vs 27% with thinking on), so it isn't an artifact of where the riddles came from.

3. **Thinking doesn't prevent it. On riddles it makes it worse.** Thinking raises accuracy on altered riddles
   (34% → 44%), but it raises the shortcut rate too (18% → 27%), and the substitution share of errors goes from 30%
   to 50%. On the same items, thinking fixed 22 substitutions and introduced 31. In 98% of the substituted answers,
   the thinking trace calls the riddle "classic" or "well-known". The model recognises the familiar version, reasons
   about it at length, and then gives the familiar answer anyway.

4. **Math: almost no substitution, except when a fact is missing.** When GSM8K problems get new numbers or a reversed
   question, accuracy stays at 89–96% and the model almost never returns the original's answer: it actually
   recomputes. When a needed fact is deleted, only 33% (thinking off) and 63% (thinking on) of answers say the
   problem can't be solved. 13% and 8% give the original problem's number, filling in the deleted fact from memory,
   and 40% and 29% invent some other assumption. I see this as a broader form of the same blind spot: the model
   turns the task into one it can answer.

5. **Warnings barely help; showing the difference helps a little.** A generic "read carefully" warning changes
   accuracy by −1 to +6 pp. Putting the original next to the modified problem helps more: +4 to +8 pp overall, and on
   unanswerable math problems from 33% to 51% (thinking off) and from 63% to 86% (thinking on). But the familiar
   answer is sticky. Of the riddles substituted with thinking on, only 26% are fixed when the original is shown
   alongside, and 45% *still* get the original's answer.

6. **The grading holds up reasonably well.** The judge agrees with the rule-based grader on 91% (GSM-Plus) and 94%
   (riddles) of the answers where the rules could decide.

## Verdicts

I fixed the criteria before looking at the full run. Each one is decided by a threshold:

1. **Knows the familiar version:** canonical accuracy ≥ 70% (yes), ≥ 40% (partly).
2. **The modification hurts:** accuracy drop from canonical to perturbed ≥ 15 pp (yes), ≥ 5 pp (partly).
3. **Errors land on the original's answer:** substitution share of errors ≥ 50% with CI lower bound ≥ 25% (yes),
   ≥ 25% (partly).
4. **Could solve it:** contrast accuracy − perturbed accuracy has a CI above 0 (yes), or is positive with a CI that
   includes 0 (partly).
5. **Substitution is silent:** ≥ 70% of shortcut answers never mention a change (yes), ≥ 40% (partly).

**Overall:** *supported* when 2 and 3 are yes and 4 is at least partly; *partially supported* when 3 is at least
partly; otherwise *not supported*.

| setting | criterion | evidence | verdict |
|:--|:--|:--|:--|
| Altered riddles, thinking off | 1. Knows the familiar version | accuracy on the originals 37% | ❌ no |
| | 2. The modification hurts | accuracy 37% → 34% (−3 pp) | ❌ no |
| | 3. Errors land on the original's answer | 30% of errors [25%, 35%]; shortcut rate 18%; COR 29% | ⚠️ partly |
| | 4. Could solve it (recovery) | contrast +4 pp [0%, 8%]; nudge +1 pp | ⚠️ partly |
| | 5. Substitution is silent | 67% of 249 shortcut answers never mention a change | ⚠️ partly |
| Altered riddles, thinking on | 1. Knows the familiar version | accuracy on the originals 67% | ⚠️ partly |
| | 2. The modification hurts | accuracy 67% → 44% (−22 pp) | ✅ yes |
| | 3. Errors land on the original's answer | 50% of errors [44%, 56%]; shortcut rate 27%; COR 34% | ✅ yes |
| | 4. Could solve it (recovery) | contrast +8 pp [4%, 11%]; nudge +6 pp | ✅ yes |
| | 5. Substitution is silent | 84% of 318 shortcut answers never mention a change | ✅ yes |
| GSM-Plus, thinking off | 1. Knows the familiar version | accuracy on the originals 96% | ✅ yes |
| | 2. The modification hurts | accuracy 96% → 72% (−24 pp) | ✅ yes |
| | 3. Errors land on the original's answer | 22% of errors [14%, 31%]; shortcut rate 5%; COR 5% | ❌ no |
| | 4. Could solve it (recovery) | contrast +6 pp [3%, 9%]; nudge +0 pp | ✅ yes |
| | 5. Substitution is silent | 33% of 36 shortcut answers never mention a change | ❌ no |
| GSM-Plus, thinking on | 1. Knows the familiar version | accuracy on the originals 96% | ✅ yes |
| | 2. The modification hurts | accuracy 96% → 83% (−13 pp) | ⚠️ partly |
| | 3. Errors land on the original's answer | 18% of errors [10%, 28%]; shortcut rate 3%; COR 3% | ❌ no |
| | 4. Could solve it (recovery) | contrast +8 pp [5%, 11%]; nudge −1 pp | ✅ yes |
| | 5. Substitution is silent | 55% of 29 shortcut answers never mention a change | ⚠️ partly |

## Detailed findings

**By modification type**

| setting | type | items | acc canonical | acc perturbed | acc nudge | acc contrast | shortcut rate | COR | substitution share of errors |
|:--|:--|--:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| Altered riddles, thinking off | bias probe | 144 | 38% | 32% [25%, 40%] | 33% | 30% | 26% | 39% | 41% [32%, 51%] |
| | constraint addition | 236 | 40% | 37% [32%, 43%] | 40% | 45% | 17% | 22% | 30% [22%, 38%] |
| | context swap | 139 | 37% | 38% [30%, 46%] | 32% | 40% | 10% | 22% | 18% [10%, 27%] |
| | meaning shift | 181 | 38% | 29% [23%, 36%] | 32% | 34% | 20% | 35% | 30% [21%, 39%] |
| Altered riddles, thinking on | bias probe | 144 | 66% | 39% [32%, 47%] | 49% | 47% | **33%** | 40% | 57% [46%, 68%] |
| | constraint addition | 236 | 68% | 48% [42%, 55%] | 55% | 59% | 26% | 34% | 51% [42%, 60%] |
| | context swap | 139 | 64% | 50% [42%, 58%] | 55% | 56% | 19% | 26% | 41% [29%, 52%] |
| | meaning shift | 181 | 69% | 39% [31%, 47%] | 41% | 44% | 29% | 34% | 50% [40%, 59%] |
| GSM-Plus, thinking off | made unanswerable | 150 | 96% | 33% [26%, 41%] | 33% | 51% | 13% | 13% | 25% [16%, 35%] |
| | numbers changed | 150 | 96% | 89% [84%, 94%] | 90% | 91% | 1% | 1% | 14% [0%, 36%] |
| | question reversed | 150 | 96% | 93% [89%, 97%] | 93% | 93% | 0% | 0% | 0% [0%, 0%] |
| GSM-Plus, thinking on | made unanswerable | 150 | 96% | 63% [55%, 71%] | 61% | 86% | 8% | 8% | 22% [12%, 33%] |
| | numbers changed | 150 | 96% | 90% [85%, 95%] | 90% | 90% | 1% | 1% | 7% [0%, 21%] |
| | question reversed | 150 | 96% | 96% [93%, 99%] | 95% | 97% | 1% | 1% | 17% [0%, 50%] |

The bias-probe riddles with thinking on have the highest shortcut rate: 33% of 144 items.

**Substitution vs. knowing the original** (shortcut rate on `perturbed`)

| setting | model solved the original | model did not |
|:--|:-:|:-:|
| Altered riddles, thinking off | **29%** (n=265) | 12% (n=435) |
| Altered riddles, thinking on | **34%** (n=470) | 13% (n=230) |
| GSM-Plus, thinking off | 5% (n=432) | 6% (n=18) |
| GSM-Plus, thinking on | 3% (n=432) | 6% (n=18) |

**What happens to substituted items when the model is warned or shown the original**

Each row follows the items that were substituted in `perturbed` and shows what the same items got in `nudge` and
`contrast`.

| setting | substituted items | condition | fixed | still the original's answer | other |
|:--|--:|:--|:-:|:-:|:-:|
| Altered riddles, thinking off | 129 | nudge | 16% | 44% | 40% |
| | | contrast | 20% | 34% | 46% |
| Altered riddles, thinking on | 188 | nudge | 28% | 48% | 24% |
| | | contrast | 26% | 45% | 30% |
| GSM-Plus, thinking off | 22 | nudge | 18% | 36% | 45% |
| | | contrast | 32% | 36% | 32% |
| GSM-Plus, thinking on | 14 | nudge | 21% | 64% | 14% |
| | | contrast | 64% | 29% | 7% |

![Substituted items under nudge and contrast](https://huggingface.co/buckets/Erenko-13/llm-task-shortcutting/resolve/analysis/recovery_of_substituted_items.png)

**Thinking on vs. off, on the same items**

- GSM-Plus (450 items): accuracy 72% with thinking off vs 83% with thinking on. Thinking fixed 9 substitutions and
  introduced 1; no thinking answers were left unfinished.
- Altered riddles (700 items): accuracy 34% with thinking off vs 44% with thinking on. Thinking fixed 22
  substitutions and introduced 31; 2% of thinking answers never finished.

**GSM-Plus problems with a missing fact** (150 items per mode)

| | says it can't be determined (correct) | gives the original's number | gives another number | no answer |
|:--|:-:|:-:|:-:|:-:|
| thinking off | 33% | **13%** | 40% | 13% |
| thinking on | 63% | **8%** | 29% | 1% |

**Is substitution silent?** (shortcut answers in `perturbed` and `nudge`)

| setting | shortcut answers | answer mentions a change | thinking mentions a change | thinking mentions a familiar version |
|:--|--:|:-:|:-:|:-:|
| Altered riddles, thinking off | 249 | 33% | – | – |
| Altered riddles, thinking on | 318 | 16% | 97% | 98% |
| GSM-Plus, thinking off | 36 | 67% | – | – |
| GSM-Plus, thinking on | 29 | 45% | 100% | 79% |

The "thinking mentions a change" column is inflated (see Limitations). The "familiar version" column is the more
reliable one.

**Response length** (median tokens on `perturbed`)

| setting | shortcut answers | correct answers |
|:--|:-:|:-:|
| Altered riddles, thinking off | 173 | 169 |
| Altered riddles, thinking on | 6,010 | 5,538 |
| GSM-Plus, thinking off | 657 | 313 |
| GSM-Plus, thinking on | 5,104 | 2,291 |

So substitution isn't about saving effort in the literal sense. On math, the shortcut answers are about twice as
long as the correct ones.

**Riddles written by Qwen vs. by other models**

| setting | written by | items | accuracy | shortcut rate |
|:--|:--|--:|:-:|:-:|
| thinking off | other models | 561 | 34% | 19% |
| thinking off | Qwen3.5 | 139 | 37% | 17% |
| thinking on | other models | 561 | 44% | 27% |
| thinking on | Qwen3.5 | 139 | 44% | 27% |

## Examples of substitution

**Altered riddle (meaning shift), thinking on**

- Original: *I open wide and tight I shut, Sharp am I and paper-cut fingers too, so do take care, I'm good and bad,
  so best beware.* → Scissors
- Modified: *I open wide and tight I shut, Sharp am I and **bite** fingers too, so do take care, I'm good and bad, so
  best beware.* → A mouth
- Model's answer: **A Knife**. It is still answering the original "sharp cutting tool" riddle, and the judge labelled it
  as the original's answer.

**Altered riddle (meaning shift), thinking on**

- Original: *I am the hole in the night, the ever watchful eye. I return in a cycle, to enlighten the sky.* → The moon
- Modified: *I am the hole in the night, the ever watchful **ear**. I return in a cycle, to **listen to** the sky.* →
  A radio telescope
- Model's answer: **The Moon**

**GSM-Plus (made unanswerable), thinking on**

- Original: *The combined age of Peter, Paul and Jean is 100 years old. Find the age of Peter knowing that Paul is 10
  years older than John and that Peter's age is equal to the sum of Paul and John's age.* → 50
- Modified: the clause "Paul is 10 years older than John" is removed → cannot be determined
- Model's answer: **50**

**GSM-Plus (made unanswerable), thinking off**

- Original: *John likes to have a glass of water with breakfast, lunch and dinner. Finally, he has **one** before he
  goes to bed as well. John does this every weekday, but on the weekends he likes to relax and have a soda with
  dinner instead. How many glasses of water does John drink in a week?* → 26
- Modified: "he has **some** before he goes to bed" → cannot be determined
- Model's answer: **26**. Its reasoning says "let's assume his … pre-bed routine remain[s] the same" and counts one
  glass, which is exactly the fact that was taken out.

## Q3 · Path forward

**Proposed fix: train on minimal pairs, using the familiar answer as the negative example.**

1. **Data curation: minimal-pair contrast sets.** Take task families the model has seen many times: classic riddles
   and puzzles, benchmark-style math word problems, textbook coding exercises, standard documents. Generate variants
   that change one decisive detail. An LLM proposes the edit, and the new answer is verified by code (math, code) or
   by a stronger model plus spot checks (riddles). Include *unanswerable* variants made by deleting a needed fact,
   whose answer is "cannot be determined, X is missing". Each record stores the modified prompt, its correct answer
   and the original's answer. Unmodified originals stay in the mix, so the model doesn't learn that every familiar
   problem is a trick: accuracy on the originals (96% on GSM8K here) must not drop.

2. **Fine-tuning: preference optimization with the substituted answer as "rejected".** Task substitution comes with
   a ready-made hard negative: the original's answer given to the modified prompt. That gives DPO-style pairs
   directly. The chosen response is the correct answer to the modified problem; the rejected one is the familiar
   answer with the familiar reasoning. For unanswerable variants, chosen = "cannot be determined, because … is
   missing" and rejected = the original problem's number. This targets exactly the error measured here, which plain
   fine-tuning on more riddles or more math would not. Since it's a narrow behaviour, a few thousand pairs per task
   family should be enough to test the idea.

3. **Reasoning: check against the text, not just recognise the template.** The thinking traces call the problem
   "classic" or "well-known" in 98% of substituted riddle answers, and thinking *increased* substitution on riddles
   (from 30% to 50% of errors). So recognising the familiar version isn't the problem. The problem is that the model
   never goes back and checks its answer against the actual wording. In RL with verifiable rewards, modified items
   are cheap to reward (the correct answer is known, and the familiar answer scores zero), and the reasoning can be
   shaped toward an explicit step: "this resembles X; here is what's different; does my answer satisfy every stated
   condition?" The data points the same way. A generic "read carefully" warning changed accuracy by −1 to +6 pp,
   while showing the original next to the modified problem raised it by 4 to 8 pp, and from 63% to 86% on
   unanswerable math problems with thinking on. Making the difference explicit helps; being told to be careful
   doesn't. The model should learn to produce that difference itself.

4. **Evaluation: report substitution next to accuracy.** Measure the shortcut rate and the substitution share of
   errors, not just accuracy. Success for the steps above means a lower shortcut rate on *held-out* task families
   (e.g. train on riddles and math, test on code and documents) while accuracy on the unmodified originals stays the
   same.

## Data quality

Every planned prompt was answered and graded in both modes: 189 + 3 × 700 riddle prompts and 150 + 3 × 450 math
prompts per mode, 7,578 responses in total.

| setting | responses | hit the token limit (counted as no answer) |
|:--|--:|:-:|
| Altered riddles, thinking off | 2,289 | 6% |
| Altered riddles, thinking on | 2,289 | 2% |
| GSM-Plus, thinking off | 1,500 | 5% |
| GSM-Plus, thinking on | 1,500 | 0% |

Where the rule-based grader could decide, the judge agrees with it on 91% of 2,908 GSM-Plus answers and 94% of 2,222
riddle answers. A manual audit sample is exported in `results/judge_audit.csv` but hasn't been labelled yet.

## Limitations

- GSM8K is very likely in the model's training data. That's part of what is being tested (reliance on memorised
  solutions), but it limits how far the results generalise.
- 139 of the altered riddles were generated by Qwen3.5 models. I report them separately; they behave the same as the
  rest.
- The judge is from the same model family as the model under test. The rule-based cross-check is there to catch
  systematic judge errors, and the manual audit is still open.
- "Silent", "mentions a change" and "familiar version" use keyword matching, so they are approximate. The change
  keywords include "assume", which appears in almost every long thinking trace, so "thinking mentions a change" is
  inflated. The familiar-version keywords ("classic", "well-known", …) are more specific.
- One sample per prompt, one model.

## Files

Everything below is in the
[Hugging Face bucket](https://huggingface.co/buckets/Erenko-13/llm-task-shortcutting).

| path | content |
|---|---|
| `data/eval_items.jsonl` | every prompt with its `gold` and `shortcut` answers |
| `results/generations_<mode>.jsonl` | raw model responses, including thinking traces |
| `results/judgments.jsonl`, `results/graded.csv` | judge outputs and final labels |
| `results/judge_audit.csv` | judgments for manual checking |
| `analysis/` | summary with CIs, recovery table, figures, the auto-generated report (`report.md`) |
| `task_shortcutting_eval.ipynb` | generation and judging: Colab A100, vLLM, under 3 hours for both modes |
| `analyze_results.ipynb` | all numbers, tables and figures in this README, from the bucket contents (CPU) |

Data licenses: altered-riddles is Apache-2.0; GSM-Plus is CC-BY-SA-4.0, and the items derived from it are
redistributed under the same license.

## Note on edits after the submission date

After the submission date I edited only the README files and the format of the final report: wording, layout and
the chart titles. The experiments, the data, the model outputs, the grading and all the numbers are unchanged from
the submitted run.
