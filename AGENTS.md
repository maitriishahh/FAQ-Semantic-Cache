# AGENTS.md

## Overview
Semantic cache for an e-commerce FAQ chatbot. A new question is embedded; if its nearest cached question is similar enough and safe to match, return the cached answer; otherwise call the LLM, answer, and (if eligible) cache the pair. Three-day portfolio project: a working, measurable result beats extra features. Interface is a terminal CLI. No frontend.

Must demonstrate with real measured numbers: cache hit rate, false-hit count, LLM call reduction, latency reduction, reliability via negation guard and TTL.

Not implemented yet. Root already has `pyproject.toml`, `uv.lock`, `.env.example`, `.gitignore`. Do not scaffold empty modules, data, tests, or docs until the step that needs them.

## Goals and non-goals
Non-goals: Redis/RedisVL, LangChain, LlamaIndex, GPTCache, agents, GraphRAG, per-tenant caches, fine-tuning, web UI (Streamlit only after the core works and only if asked).

## Stack
Locked. Do not change silently. Propose any change in Decision log; never apply it unasked.

- Python ≥3.13 (`requires-python` in `pyproject.toml`). The brief said ≥3.11. Do not downgrade.
- uv. Already declared: `sentence-transformers`, `faiss-cpu`, `numpy`, `google-genai`, `python-dotenv`. Dev: `pytest`, `ruff`.
- Embedder: `paraphrase-multilingual-MiniLM-L12-v2`.
- faiss-cpu `IndexFlatIP` on L2-normalized vectors, so score = cosine.
- SQLite (stdlib). Put the DB under `storage/` (already gitignored). Do not create that directory until the store exists.
- One provider behind `LLMClient`, config via env. Already chosen: Google Gemini (`google-genai`). `GEMINI_API_KEY`, `GEMINI_MODEL` (`.env.example` sets `gemini-3.5-flash-lite`). Do not swap providers unasked.

## Architecture
One module per responsibility, no extra abstraction layers.

- embed: load model once, return normalized float32 vectors.
- faqs: load and validate 30 FAQs.
- store: SQLite is the source of truth (cache entries, expiry, request log). The FAISS index is rebuilt from non-expired rows at startup and on cleanup.
- guards: negation guard, cache-eligibility (PII) check. Pure functions.
- cache: embed → nearest → threshold → guards → TTL → hit/miss decision. Takes the threshold from config.
- llm: `LLMClient.generate(question) -> (text, usage | None)`; never raises out of the request flow.
- bot: the single `answer()` request flow; writes one log row per request.
- eval: sweep and report. Never imports `llm`.
- cli: thin argparse entry point (`ask`, `sweep`, `bench`, `demo`).

## Directory structure
Target only. Do not create these until implementation needs them. Never overwrite existing files.

```
src/faqcache/  config.py embed.py faqs.py store.py guards.py cache.py llm.py bot.py eval.py cli.py
data/          faqs.json  eval_set.json
results/       sweep_*.csv  bench.csv  caught_examples.md
tests/
docs/          STATUS.md
AGENTS.md  README.md  .env.example  pyproject.toml  uv.lock
```

Already present: `pyproject.toml`, `uv.lock`, `.env.example`, `.gitignore`, `.env` (gitignored). `storage/` is gitignored for the SQLite file.

## Data schemas
faqs.json: `{id, category, question, answer, ttl_days}`. 30 entries covering refunds, returns, payments, delivery, cancellation, orders, account support. Unique ids (`faq_01`…). `ttl_days` per category (short for price/policy topics).

eval_set.json: exactly 100 rows `{id, question, expected_faq_id | null, language: en|hi|hinglish, kind: paraphrase|near_miss|out_of_domain, split: dev|test}`.

- 60 positives (`kind=paraphrase`): genuine paraphrases of the 30 FAQs, a mix of English, Hindi (Devanagari or romanized) and Hinglish, with typos/short forms.
- 40 negatives (`expected_faq_id=null`): at least 20 `near_miss` (same topic, different intent or negation flipped, e.g. "refund milega" vs "refund nahi milega"), the rest `out_of_domain`.
- Fixed 50/50 stratified dev/test split stored in the file.
- Generated data is a draft. A human reviews all 100 rows. Never edit labels or questions to improve a metric.

## Invariants
- Evaluation uses a read-only, in-memory snapshot of the original 30 FAQs. Eval never calls the LLM and never writes new entries to the cache.
- Expired entries are never returned.
- A cache hit requires: score ≥ threshold AND negation status equal AND entry not expired.
- Never cache: errors, empty answers, answers or questions containing order IDs, names, phone numbers, emails (conservative heuristics; limits are in README).
- LLM failure returns a readable string and is never cached.
- Hits log zero LLM tokens. Use provider-reported usage; if unavailable store NULL. Never estimate or fabricate tokens or metrics.
- Threshold lives only in config.
- Secrets only in env; `.env.example` committed, `.env` gitignored.

## Evaluation
Definitions (guards on unless stated):

- correct_hit: positive query returns its expected FAQ.
- wrong_faq_hit: positive query returns a different FAQ.
- false_hit: negative query returns any FAQ.
- false_miss: positive query not served from cache.
- unsafe hits = wrong_faq_hit + false_hit.
- positive hit rate = correct_hit / positives; overall accuracy = (correct_hit + correct negatives) / total.
- LLM calls = number of non-hits (simulated; no real calls in sweep).
- Latency: `perf_counter` around embed+search, model warmed up first. Report avg lookup latency and avg hit latency.

Process: sweep thresholds 0.75, 0.80, 0.85, 0.90, 0.95, then refine in 0.01 steps around the best region. Run with the negation guard off and on to show its effect. Choose the threshold on **dev**: highest correct_hit with unsafe hits = 0 (at most 1, with the pair explained). Report final numbers on **test**. Export CSV per run. Write a failure analysis: one line per unsafe hit and per false miss at the chosen threshold.

## Conventions
Type hints, short docstrings, config separate from logic, no unnecessary dependencies, handle missing model/API failures gracefully, simple over clever, update README when behaviour changes, record results only from actual runs. `ruff` is already the linter; do not add another.

## Testing
pytest. Required: negation guard (English + Hinglish: not, no, never, nahi, mat), TTL expiry, eligibility check, eval snapshot isolation (eval run leaves the cache unchanged), data validation (counts, unique ids, label ids exist). No tests that call a real LLM.

## Roadmap
- Day 1: setup, FAQ data, eval set (human-reviewed), embeddings, FAISS lookup, hit/miss flow with a stub LLM.
- Day 2: sweep and metrics, negation guard, TTL, eligibility, tests.
- Day 3: real LLM client, request logging checks, CLI demo, before/after benchmark (cache off vs on), README, demo recording.

README results line (fill only from real runs): "Cache hit rate X% on N test questions at threshold T with Y false hits; LLM calls down Z%; avg latency A ms → B ms."

## Future work
Redis/RedisVL, per-tenant cache, exact-match layer, cross-encoder rerank, embedder fine-tuning, sampled hit grading, Streamlit.

## Decision log
- Python floor stays `>=3.13` as already set in `pyproject.toml`. Not a downgrade to 3.11.
- LLM provider stays Gemini via `google-genai` and `GEMINI_API_KEY` / `GEMINI_MODEL`. Interface remains `LLMClient`.

## Collaboration mode
The owner is using this project to learn. Default to explaining and scaffolding, not writing core logic. Do not write `embed.py`, the FAISS search, the hit/miss decision in `cache.py`, the sweep metrics in `eval.py`, or `guards.py` negation logic unless explicitly asked. You may write boilerplate on request (pyproject, CLI parsing, SQLite plumbing, CSV output, README formatting). Before writing any non-trivial code, state in 2 to 3 lines what it does and why. Do not create folders or files that are not needed for the current step; the owner builds them while implementing.
