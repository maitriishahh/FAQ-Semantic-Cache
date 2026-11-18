# FAQ Semantic Cache

Terminal semantic cache for an e-commerce FAQ chatbot. Embed the question, return a cached answer when the nearest FAQ is similar enough and safe to match, otherwise call the LLM and cache the pair only if it is eligible.

No web UI. A measured result matters more than extra features.

## Status

Day-0 setup only: `pyproject.toml`, `uv.lock`, `.env.example`. The CLI, data, cache, and eval are not built. Do not treat this README as a claim that those pieces exist.

Results (fill only after a real run; do not invent numbers):

> Cache hit rate X% on N test questions at threshold T with Y false hits; LLM calls down Z%; avg latency A ms → B ms.

## Stack

Locked in `pyproject.toml` and `.env.example`. Do not change it silently.

- Python ≥3.13, uv
- `sentence-transformers` model `paraphrase-multilingual-MiniLM-L12-v2`
- `faiss-cpu` `IndexFlatIP` on L2-normalized vectors (score = cosine)
- SQLite (stdlib), file under gitignored `storage/` once the store exists
- Gemini via `google-genai`, behind a small `LLMClient`
- pytest, ruff

## Setup

```bash
uv sync
copy .env.example .env
```

Put a real key in `.env`. Never commit it.

| Variable | Role |
| --- | --- |
| `GEMINI_API_KEY` | Gemini API key |
| `GEMINI_MODEL` | Model id. Example file sets `gemini-3.5-flash-lite` |

## Planned CLI

Not implemented. Entry points, when they exist: `ask`, `sweep`, `bench`, `demo`.

`sweep` must not call the LLM. It scores a fixed eval set against the original 30 FAQs.

## Data

Not in the repo yet.

- `data/faqs.json`: 30 FAQs, `{id, category, question, answer, ttl_days}`, ids `faq_01`…. Short TTL on price and policy topics.
- `data/eval_set.json`: 100 rows, `{id, question, expected_faq_id | null, language, kind, split}`. 60 paraphrases, 40 negatives (at least 20 near-misses). Fixed 50/50 dev/test split. A human reviews every row. Do not edit labels or questions to improve a metric.

## What counts as a hit

A hit needs score ≥ threshold, the same negation status, and a non-expired row. Threshold lives only in config. Eval uses a read-only snapshot of the original 30 FAQs and must not write cache rows or call the LLM.

Choose the threshold on dev: highest correct-hit count with zero unsafe hits (at most one, and that pair must be explained). Report test numbers only after that choice. Unsafe hit = wrong FAQ on a positive, or any FAQ on a negative.

## What is never cached

Errors, empty answers, and questions or answers that look like they contain an order id, a person's name, a phone number, or an email.

Those checks are conservative string heuristics, not a PII model. Limits:

- They miss names, ids, and phones that do not match the patterns, including many Hindi or romanized names and odd order-id formats.
- They can block harmless text that merely looks like a phone, email, or id.
- They are not redaction and not a compliance control. A miss means the pair is not cached; it does not mean the text was scrubbed.

LLM failures return a readable string and are not cached. Hits log zero LLM tokens. If the provider does not report usage, store NULL. Never estimate tokens or metrics.

## Layout

Target, not created yet: `src/faqcache/`, `data/`, `results/`, `tests/`, `docs/STATUS.md`. Agent rules are in `AGENTS.md`.
