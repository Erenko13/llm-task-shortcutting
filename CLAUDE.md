# Project context

Fatima Fellowship (Fall 2026) application, "Blind Spots of Frontier Models" challenge:
1. Identify an under-explored blind spot of current models (drawn from own experience).
2. Evaluate it systematically on a 0.6B–6B open-weight Hugging Face model, run in Colab.
3. Propose a fix (data curation / fine-tuning / architecture; may be speculative).
Deliverable: a **public Hugging Face Bucket** with all artifacts (eval dataset, results, figures,
notebook) and a README answering Q1 and Q3.

## Hypothesis: task substitution
When a prompt looks like a familiar task but differs in one key detail, the model quietly solves the
familiar (easier) task instead. The error looks like hallucination, but it is a correct answer to a
different question. Evidence: wrong answers on perturbed items land on the *familiar* answer far more
than on anything else, and accuracy recovers when the change is pointed out (so it's not inability).

## Model
`Qwen/Qwen3.5-4B` (post-trained, Apache-2.0). Multimodal (~5B with vision encoder), used text-only.
Thinks by default; `enable_thinking=False` in the chat template turns it off (`/think` and `/no_think`
are not supported). Both modes are evaluated.

## Setup and workflow (user's choices)
- **Everything lives in one notebook: `task_shortcutting_eval.ipynb`.** No .py package, no git clone
  on the runtime, no Google Drive. The user rejected those options; don't reintroduce them.
- The user runs it with **Google Colab inside VS Code**: the kernel is remote and cannot see local files.
- **No local Python on this PC**, so notebook code can't be executed here; check it by reading.
  Edit the .ipynb with the notebook-editing tool and keep the JSON valid.
- Results persist via a Hugging Face Bucket (`HF_BUCKET` in the config cell), synced after every
  generation chunk. Re-running resumes where it stopped. The bucket stays private until the final publish cell.
- No Turkish/multilingual slice (user decided against it).

## Notebook structure
0 config + install + bucket login/sync → 1 dataset (source items, pair builder, grader, self-test)
→ 2 model (vLLM or HF engine, smoke test, resumable generation) → 3 grading → 4 analysis (summary
table with bootstrap CIs, two figures, examples) → 5 publish bucket.

Dataset: 167 minimal pairs × 4 conditions (`canonical`, `perturbed`, `nudge`, `explicit`) = 668 prompts.
Categories: modified puzzles (17), base-8/9 arithmetic (40), misspelled letter counting (40), verbatim
copy with word swap/typo (40), list length (20), code with changed spec (10). Each row has `gold` and
`shortcut` (= canonical answer). Grader labels: correct / shortcut / other_wrong / ambiguous / unparseable,
plus disclosure flags for the answer and the thinking trace.

Sampling: Qwen's recommended values with presence_penalty = 0 (non-think 0.7/0.8/20; think 0.6/0.95/20).

## Status / next steps
- [ ] First Colab run: self-test passes, then smoke test with `PAIRS_PER_CATEGORY = 2`
- [ ] Full run, both modes
- [ ] Interpret results; README answers to Q1 (blind spot, tied to user's background) and Q3 (fix)
- [ ] Upload README + notebook to the bucket, run the publish cell, submit the bucket link
