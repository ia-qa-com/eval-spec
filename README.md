# ia-eval spec

> The open specification for `.ia-eval.yaml`: an evaluation contract for LLM applications that gates CI on what the model actually answers.

**Backed by [IA-QA](https://www.ia-qa.com) · MIT License · [Changelog](CHANGELOG.md)**

---

## What is `.ia-eval.yaml`?

A `.ia-eval.yaml` file describes a suite of tests for a prompt, a RAG pipeline or an AI feature. Each scenario is sent to a real model, and the answer is judged against what the contract states.

```yaml
metadata:
  name: My RAG Pipeline
  version: "1.0"
  model: openai/gpt-oss-20b
  provider: groq

expectations:
  no_toxicity: true

scenarios:
  - id: refund-window
    input: How long do I have to return an item?
    system_prompt: 'Answer from this documentation: "Returns are accepted within 30 days."'
    expectations:
      contains: ["30 days"]
      not_contains: ["60 days"]
```

Run it in CI with [ia-qa-com/eval-action](https://github.com/ia-qa-com/eval-action):

```yaml
- uses: ia-qa-com/eval-action@v1
  with:
    contract: evals/my-pipeline.ia-eval.yaml
    api_key_groq: ${{ secrets.GROQ_API_KEY }}
```

---

## How a scenario is judged

Two different things are checked, and only one of them is a score.

- **Assertions**, in `expectations`. Every assertion a contract states is enforced: one that fails, fails the scenario, whatever the score.
- **Evaluators**, in `evaluators`. A weighted score from 0 to 100, compared to `min_score`. Without evaluators, the score is the average of the assertions.

Each scenario ends in one of three states, and only `true` counts as passed:

| `passed` | Meaning |
|---|---|
| `true` | Every assertion holds and the score reaches `min_score` |
| `false` | An assertion failed (named in `failed_assertions`), or the score is under `min_score` |
| `null` | Nothing could be judged: no answer from the model (`error`), a check that could not decide (named in `not_evaluated`), or nothing to check at all |

The verdict of the run:

| Verdict | Meaning |
|---|---|
| `PASS` | At least one scenario ran, and every one passed |
| `FAIL` | At least one scenario has `passed: false` |
| `PARTIAL` | None failed, but at least one has `passed: null`, or none ran |

**A check that could not decide is never a pass.** `language` on a one-word answer, `no_hallucination` without a `ground_truth`, and a judge with no key all read `passed: null`.

An evaluator with nothing to work on (`exact_match` on a scenario with no `ground_truth`, `contains_check` with no strings) is skipped and not scored. It does not lower the score, and it does not raise it.

---

## File naming

Files must end with `.ia-eval.yaml` or `.ia-eval.yml`.

```
your-project/
  evals/
    rag-pipeline.ia-eval.yaml
    guardrails.ia-eval.yaml
```

---

## Editor autocomplete

Add this comment as the **first line** of a contract to get validation, autocompletion and inline errors (VS Code with the [Red Hat YAML extension](https://marketplace.visualstudio.com/items?itemName=redhat.vscode-yaml), JetBrains, Neovim…):

```yaml
# yaml-language-server: $schema=https://www.ia-qa.com/api/eval/json-schema
```

That schema is derived from the validator the runner uses, so what your editor refuses is what the runner refuses. **Unknown fields are refused**: a typo such as `min_scroe` is an error, never a default in silence.

---

## Schema reference

Full JSON Schema: [`schema/ia-eval.schema.json`](schema/ia-eval.schema.json)

### `metadata` (required)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | ✅ | Human-readable contract name |
| `version` | string | ✅ | Contract version (e.g. `1.0`) |
| `description` | string | | What this contract tests |
| `model` | string | | Default model (default: `openai/gpt-oss-20b`) |
| `provider` | string | | `openai` \| `anthropic` \| `google` \| `groq` \| `deepseek` \| `xai` \| `mistral` \| `huggingface` (default: `groq`) |
| `temperature` | number | | 0–2 (default: 0.7). Use 0 for a stable gate |
| `max_tokens` | integer | | 1–100000 (default: 1024) |
| `system_prompt` | string | | Default system prompt for all scenarios |
| `tags` | string[] | | Freeform labels |
| `author` | string | | |
| `created_at` | string | | ISO date |

### `scenarios` (required, 1–500 items)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | ✅ | Letters, digits, `_` and `-` |
| `input` | string | ✅ | User prompt sent to the model |
| `ground_truth` | string | | Expected correct answer |
| `description` | string | | Human description of the test |
| `system_prompt` | string | | Overrides `metadata.system_prompt` |
| `model` | string | | Overrides `metadata.model` |
| `provider` | string | | Overrides `metadata.provider` |
| `temperature` | number | | Overrides `metadata.temperature` |
| `tags` | string[] | | Freeform labels |
| `skip` | boolean | | Skip this scenario |
| `expectations` | object | | Overrides the global `expectations` field by field: a `contains` here replaces the global `contains` |
| `evaluators` | array | | Replaces the contract's `evaluators` for this scenario, e.g. a judge with its own rubric |

### `expectations` (optional, global)

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `min_score` | number | `70` | Minimum score to pass (0–100) |
| `contains` | string[] | `[]` | Assertion: every string appears (case-insensitive) |
| `not_contains` | string[] | `[]` | Assertion: no string appears (case-insensitive) |
| `regex` | string | | Assertion: the output matches (case-insensitive) |
| `json_output` | boolean | `false` | Assertion: the output is valid JSON |
| `min_length` | integer | | Assertion: at least this many characters |
| `max_length` | integer | | Assertion: at most this many characters |
| `language` | string | | Assertion: the output is in this language. `en`, `fr`, `es`, `de`, `it`, `pt`, `nl`, `ru`, `zh`, `ja`, `ko`, `ar`, `pl`, `tr`, `sv` |
| `no_toxicity` | boolean | `false` | Assertion: not toxic, by a pattern scan |
| `no_hallucination` | boolean | `false` | Assertion: lexically grounded in `ground_truth`, which is required for it to pass |

### `evaluators` (optional)

Each has a `name`, a `weight` (0–1, default 1) and an optional `config` limited to the keys below.

| Name | Kind | `config` keys | Description |
|------|------|---------------|-------------|
| `exact_match` | deterministic | | Equals `ground_truth` (case-insensitive, trimmed) |
| `contains_check` | deterministic | `strings` | Share of required strings present. Adds to `expectations.contains` |
| `not_contains_check` | deterministic | `strings` | No forbidden string. Adds to `expectations.not_contains` |
| `json_format` | deterministic | | Output is valid JSON |
| `regex_match` | deterministic | `pattern`, `flags` | Output matches `pattern` (default: `expectations.regex`) |
| `length_check` | deterministic | `min_length`, `max_length` | Character count within bounds |
| `cosine_similarity` | deterministic | `min_similarity` | Bag-of-words cosine against `ground_truth`. Lexical: a paraphrase scores low |
| `hallucination_check` | deterministic | `context`, `min_grounding` | Lexical grounding in `ground_truth` or `context`, numbers and polarity included |
| `toxicity_filter` | deterministic | | Pattern scan for toxic content |
| `response_quality` | deterministic | `expected_keywords`, `max_length`, `min_quality` | Keyword coverage and length. Skipped when neither is given |
| `llm_as_judge` | model call | `rubric`, `model`, `provider`, `min_score` | A model scores the answer against a rubric. Not reproducible: calibrate it against human labels before you gate on it |

### What the checks do not do

`no_hallucination`, `hallucination_check`, `no_toxicity` and `toxicity_filter` match words, numbers and known patterns. They catch "60 days" against "30 days"; they do not catch a false claim worded differently from its source. A pass means nothing was found, not that the answer is verified true. For open-ended qualities (tone, helpfulness), use `llm_as_judge`, calibrated.

---

## Examples

Each runs as is on the free Groq tier, and each is run against outputs that must fail by the tests of the repository that maintains this spec.

| File | Use case |
|------|----------|
| [`examples/rag-pipeline.ia-eval.yaml`](examples/rag-pipeline.ia-eval.yaml) | Grounding, and no invented figure when the documentation is silent |
| [`examples/prompt-stability.ia-eval.yaml`](examples/prompt-stability.ia-eval.yaml) | Format and length across runs |
| [`examples/guardrails.ia-eval.yaml`](examples/guardrails.ia-eval.yaml) | Injection resistance, secrets, PII, and a request that must still work |

---

## Running a contract

In the [LLM Test Runner](https://www.ia-qa.com/qatools/eval-contract): paste your YAML and run it.

Through the API, with your own key:

```bash
jq -n --rawfile yaml my-test.ia-eval.yaml \
  '{ inline_contract_yaml: $yaml, api_keys: { groq: env.GROQ_API_KEY } }' \
| curl -sS -X POST https://www.ia-qa.com/api/eval/run \
    -H "Content-Type: application/json" --data-binary @-
```

The key comes from your environment and travels on stdin, so it stays out of your shell history and process list.

`api_keys` accepts `openai`, `anthropic`, `google`, `groq`, `deepseek`, `xai`, `mistral`, `hf`. `overrides` accepts `model`, `provider`, `temperature`, `system_prompt`, `min_score`.

Response shape (the `summary` fields are always present):

```json
{
  "summary": { "total": 3, "passed": 2, "failed": 1, "skipped": 0, "errors": 0, "avg_score": 87, "duration_ms": 1412, "verdict": "FAIL" },
  "scenario_results": [
    {
      "scenario_id": "refund-window",
      "passed": false,
      "total_score": 100,
      "min_score": 70,
      "failed_assertions": ["not_contains"],
      "not_evaluated": [],
      "evaluator_results": [],
      "output": "…",
      "error": null
    }
  ],
  "warnings": []
}
```

### What leaves your machine

Running a contract is not local. Your contract, your API keys and the model's answers go through `www.ia-qa.com`, and your prompts and keys go on to the provider you chose. The keys are used for the duration of the request; this endpoint's code writes neither the keys, the contract nor the answers anywhere. It increments an anonymous counter of runs, which the header `X-IAQA-No-Track: 1` turns off.

### Limits of the hosted endpoint

- 20 runs per minute per IP.
- Without a key, Groq runs on a shared free key that gets rate limited: scenarios then read `error`, and the verdict `PARTIAL`. **In CI, use your own key.**
- Scenarios run one after the other. A long run keeps the connection open with leading whitespace, so a client that reads the whole body gets valid JSON.

---

## Contributing

The spec is open. If you have a use case the format does not cover, open an issue.

**Do not** open PRs that add vendor-specific fields: the format is provider-agnostic by design.

---

## License

[MIT](LICENSE): use freely in open source and commercial projects.
