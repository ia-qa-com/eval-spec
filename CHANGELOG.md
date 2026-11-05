# Changelog

## 1.1.1

- **A contract that names a model and no provider now uses that model's provider.** `model: gemini-3.5-flash-lite` used to go to Groq and read `PARTIAL` with a message asking for a Groq key. A contract that names neither still runs on Groq.
- **On Google, `system_prompt` is sent as Gemini's system instruction.** It used to be pasted into the user message, so a guardrail tested on Gemini was not the one an application ships. Verdicts on such contracts can change.

## 1.1.0

Until this version, a contract could pass without its assertions having been checked. If your CI turns red after this release, it is very likely reporting something that was already wrong.

### Verdicts that change

- **Assertions are enforced whatever the evaluators and the score.** With an `evaluators` list, `contains`, `not_contains`, `regex`, `json_output`, `max_length`, `no_toxicity` and `no_hallucination` were ignored unless a matching evaluator existed. Without one, they were averaged: a forbidden string could be present in a passing scenario.
- **`hallucination_check`, `toxicity_filter` and `response_quality` run on the hosted endpoint.** They failed on every request there, and their result was left out of the score without a word.
- **`hallucination_check` fails on a specific the source does not back** ("60 days" against "30 days"), and a failed check counts 0 in the score. A contradiction used to keep its full score.
- **A check that could not decide is no longer a pass.** Those scenarios read `passed: null`, and the run `PARTIAL`. So does a scenario with nothing to check, which used to score 100, and a run where no scenario ran, which used to read `PASS`.
- **An evaluator's `config.strings` adds to `expectations`.** It used to replace them, so `strings: []` turned the contract's `contains` off.
- `contains_check` and `not_contains_check` with no strings are skipped and not scored, instead of scoring 100.

### Contracts now refused

Each of these was accepted and did nothing. Fix them as the error message says.

- **Unknown fields**, at every level, including inside `expectations` and an evaluator's `config`. Length bounds are `min_length` and `max_length`, not `min` and `max`.
- **`tone`**: it was never evaluated, and no deterministic check can decide it. Use an `llm_as_judge` evaluator with a rubric.
- **`language`** outside the 15 languages the check can tell apart.
- **`llm_as_judge` `config.apiKey`**: a key has no place in a committed file. It comes from `api_keys`.

### Added

- `language` and `min_length` are real assertions. `language` was documented and never run; `min_length` was used in published templates and never run.
- Per-scenario `evaluators`, replacing the contract's.
- `failed_assertions` and `not_evaluated` on each scenario result.
- `overrides.min_score`, and `deepseek`, `xai`, `mistral` in the providers documented.
- `provider: hf` keeps working as an alias of `huggingface`.

## 1.0.0

Initial specification.
