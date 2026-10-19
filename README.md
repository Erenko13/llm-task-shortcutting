# llm-task-shortcutting
Eren Alp Fall 2026 Fatima Fellowship Application Project

Everything runs in one notebook, [`task_shortcutting_eval.ipynb`](task_shortcutting_eval.ipynb): dataset
construction, grader self-test, generation with Qwen3.5-4B (thinking on and off), grading, analysis, and
publishing to a Hugging Face Bucket. Open it in Google Colab on a GPU runtime and run it top to bottom.

## Dataset

Every item is a **minimal pair**: a familiar (canonical) task and a perturbed version that differs in
one key detail. Each pair is asked under four conditions:

| condition | prompt |
|---|---|
| `canonical` | the familiar task (reference) |
| `perturbed` | the changed task, no warning |
| `nudge` | the changed task plus a generic "read carefully" warning |
| `explicit` | the changed task with the changed detail pointed out |

On non-canonical rows, `shortcut` holds the answer to the canonical task. A wrong answer that equals
`shortcut` counts as **substitution** (the model solved the easier, familiar task) rather than an
arbitrary error. A gain from `perturbed` to `nudge`/`explicit` shows that the model could solve the
changed task but did not by default.

| category | pairs | perturbation |
|---|---|---|
| modified puzzles | 17 | classic riddles with one detail changed (Monty Hall, bat-and-ball, river crossing, ...) |
| counterfactual arithmetic | 40 | addition in base 8/9 instead of base 10 |
| misspelled counting | 40 | count a letter in a familiar word with one letter added or removed |
| verbatim copy | 40 | copy a famous sentence that has one swapped word or one typo |
| list length | 20 | "list N items" with N other than 5 or 10 |
| code spec | 10 | textbook function with a changed spec, graded by running tests |

167 pairs × 4 conditions = 668 prompts. Row schema (`data/shortcut_pairs.jsonl`):
`id, pair_id, category, condition, prompt, gold, shortcut, checker, meta`.
