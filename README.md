# ia-eval spec

> The open specification for `.ia-eval.yaml` — a CI/CD-ready evaluation contract format for LLM applications.

**Backed by [IA-QA](https://ia-qa.com) · MIT License**

---

## What is `.ia-eval.yaml`?

A `.ia-eval.yaml` file describes a suite of tests for an LLM prompt, RAG pipeline, or AI feature. It runs against real model APIs and produces a pass/fail verdict suitable for CI/CD gates.

```yaml
metadata:
  name: My RAG Pipeline
  version: "1.0"
  model: gpt-4o-mini
  provider: openai

expectations:
  min_score: 75
  no_hallucination: true

scenarios:
  - id: basic-grounding
    input: What is the refund policy?
    ground_truth: Refunds are accepted within 30 days.
    expectations:
      contains: ["30 days"]
```

Run it in CI with the [ia-qa-com/eval-action](https://github.com/ia-qa-com/eval-action):

```yaml
- uses: ia-qa-com/eval-action@v1
  with:
    contract: tests/my-pipeline.ia-eval.yaml
    api_key_openai: ${{ secrets.OPENAI_API_KEY }}
```

---

## File naming

Files must end with `.ia-eval.yaml` or `.ia-eval.yml`.

Conventional locations:
```
your-project/
  evals/
    rag-pipeline.ia-eval.yaml
    guardrails.ia-eval.yaml
  tests/
    prompt-stability.ia-eval.yaml
```

---

## Editor autocomplete

Add this comment as the **first line** of any `.ia-eval.yaml` file to get live
validation, field autocompletion, and inline errors in your editor (VS Code with
the [Red Hat YAML extension](https://marketplace.visualstudio.com/items?itemName=redhat.vscode-yaml),
JetBrains, Neovim, etc.):

```yaml
# yaml-language-server: $schema=https://www.ia-qa.com/api/eval/json-schema
metadata:
  name: My Pipeline
  version: "1.0"
```

The schema is served live and always matches the runtime validator. All
[examples](examples/) include this line.

---

## Schema reference

Full JSON Schema: [`schema/ia-eval.schema.json`](schema/ia-eval.schema.json)

### `metadata` (required)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | ✅ | Human-readable contract name |
| `version` | string | ✅ | SemVer (e.g. `1.0`) |
| `description` | string | | What this contract tests |
| `model` | string | | Default model (`gpt-4o-mini`, `claude-3-5-haiku-20241022`…) |
| `provider` | string | | `openai` \| `anthropic` \| `groq` \| `google` \| `hf` |
| `temperature` | number | | 0–2 |
| `max_tokens` | integer | | 1–100000 |
| `system_prompt` | string | | Default system prompt for all scenarios |
| `tags` | string[] | | Freeform labels |
| `author` | string | | |

### `scenarios` (required, 1–500 items)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | ✅ | Unique identifier (`[a-z0-9_-]`) |
| `input` | string | ✅ | User prompt sent to the model |
| `ground_truth` | string | | Expected correct answer |
| `description` | string | | Human description of the test |
| `system_prompt` | string | | Overrides `metadata.system_prompt` |
| `model` | string | | Overrides `metadata.model` |
| `provider` | string | | Overrides `metadata.provider` |
| `temperature` | number | | Overrides `metadata.temperature` |
| `skip` | boolean | | Skip this scenario |
| `expectations` | object | | Per-scenario expectation overrides |

### `expectations` (optional, global)

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `min_score` | number | `70` | Minimum score to PASS (0–100) |
| `no_hallucination` | boolean | `false` | Fail if hallucination detected |
| `no_toxicity` | boolean | `false` | Fail if toxic content detected |
| `json_output` | boolean | `false` | Output must be valid JSON |
| `contains` | string[] | `[]` | Strings that must appear in output |
| `not_contains` | string[] | `[]` | Strings that must NOT appear |
| `max_length` | integer | | Max output length in characters |
| `regex` | string | | Output must match this regex |
| `tone` | string | | `professional` \| `friendly` \| `neutral` \| `technical` |
| `language` | string | | Expected ISO 639-1 language code |

### `evaluators` (optional)

Ordered list of scoring functions. Each has a `weight` (0–1) and optional `config`.

| Name | Type | Description |
|------|------|-------------|
| `exact_match` | deterministic | Exact string match against `ground_truth` |
| `contains_check` | deterministic | Required strings present in output |
| `not_contains_check` | deterministic | Forbidden strings absent from output |
| `json_format` | deterministic | Output is valid JSON |
| `regex_match` | deterministic | Output matches regex pattern |
| `length_check` | deterministic | Character count within bounds |
| `cosine_similarity` | deterministic | TF-IDF cosine vs `ground_truth` (no API needed) |
| `hallucination_check` | network | Grounding score via ia-qa |
| `toxicity_filter` | network | Toxicity risk level via ia-qa |
| `response_quality` | network | Quality score via ia-qa |
| `llm_as_judge` | llm | Rubric-based scoring (requires `GROQ_API_KEY`) |

---

## Examples

| File | Use case |
|------|----------|
| [`examples/rag-pipeline.ia-eval.yaml`](examples/rag-pipeline.ia-eval.yaml) | RAG grounding + hallucination |
| [`examples/prompt-stability.ia-eval.yaml`](examples/prompt-stability.ia-eval.yaml) | Format consistency across runs |
| [`examples/guardrails.ia-eval.yaml`](examples/guardrails.ia-eval.yaml) | Safety, injection resistance, PII |

---

## Running locally

Via the [IA-QA web app](https://ia-qa.com/devtools/eval-framework) — paste your YAML and run directly in the browser.

Via the API (BYOK):
```bash
curl -X POST https://www.ia-qa.com/api/eval/run \
  -H "Content-Type: application/json" \
  -d '{
    "inline_contract_yaml": "'"$(cat my-test.ia-eval.yaml)"'",
    "api_keys": { "openai": "sk-..." }
  }'
```

Response shape:
```json
{
  "summary": {
    "total": 3,
    "passed": 3,
    "failed": 0,
    "avg_score": 87,
    "verdict": "PASS"
  },
  "scenario_results": [...]
}
```

---

## Contributing

The spec is open. If you have a use case the format doesn't cover, open an issue.

**Do not** open PRs that add vendor-specific fields — the format is provider-agnostic by design.

---

## License

MIT — use freely in open source and commercial projects.
