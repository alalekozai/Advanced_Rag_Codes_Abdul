# 12 · RAG Guardrails

Privacy and security layer for the RAG pipeline from `10_rag_evaluation`.
It wraps the retriever and the LLM with checks at three points: the user's
question, the retrieved documents, and the model's answer. Every request
writes one line to an audit log.

```
question → [input guard] → retriever → [chunk guard] → LLM → [output guard] → answer
               │                           │                      │
               └──────────── audit_log.jsonl (no raw text) ───────┘
```

## What it does

| Check | Where | How |
|---|---|---|
| PII redaction | input, retrieved chunks, output | Regex for email, phone, SSN, IP; credit cards verified with the Luhn checksum. Replaced with `[EMAIL]`, `[SSN]`, etc. |
| Prompt injection / jailbreak | user input | Weighted rules (instruction override, system-prompt requests, role hijack, fake `<system>` tags, exfiltration, encoded payloads). Text is normalized first so full-width letters and zero-width spaces don't slip through. |
| Indirect prompt injection | retrieved chunks | Same scanner on every chunk. Poisoned chunks are dropped before they reach the prompt. |
| System prompt leak | output | A random canary token sits in the system prompt. If it shows up in an answer, the answer is blocked. |
| Audit trail | every request | JSONL: request id, timestamp, SHA-256 of the question, outcome, and which checks fired. No raw text, so the log itself holds no PII. |
| Policy-as-code | `policy.yaml` | Thresholds and on/off switches live in config, not code. |

## OWASP Top 10 for LLM Applications (2025) coverage

| Risk | Covered by |
|---|---|
| LLM01 Prompt Injection | input scan + retrieved-chunk scan |
| LLM02 Sensitive Information Disclosure | PII redaction at all three stages; hashed audit log |
| LLM07 System Prompt Leakage | canary token check on output |
| LLM08 Vector and Embedding Weaknesses | poisoned-document detection at retrieval |

Not covered here: excessive agency (LLM06), supply chain (LLM03), and
unbounded consumption (LLM10). Those belong to the agent and serving layers.

## Run it

```bash
uv sync
uv run pytest            # 28 tests, no API key needed (fake retriever + LLM)
uv run main.py           # real PDF + OpenAI, needs OPENAI_API_KEY in .env
```

The same tests run on GitHub Actions for every push and pull request that
touches this folder (`.github/workflows/guardrails-tests.yml`). A red check
on a PR means a guardrail broke or a red-team case stopped being blocked.

## Red-team set

`tests/redteam.yaml` holds attacks that must be blocked and normal questions
that must not be. The benign list matters as much as the attacks: questions
like *"What should governments ignore when planning budgets?"* make sure the
filter doesn't block real users. When you find a bypass, add it to the file
and fix the rule.

## Known limits (honest list)

- Regex PII misses names and street addresses. Next step: swap in Microsoft
  Presidio (NER-based) behind the same `detect()` / `redact()` interface.
- Rule-based injection detection can be reworded around. Next step: add a
  classifier (e.g. Llama Prompt Guard) as a second layer and keep the rules
  as a cheap first pass.
- The red-team set is small. Next step: run promptfoo's red-team plugins
  against `main.py` for broader coverage.
