# Status

Last updated: 2026-11-18. Update this file after each working session. Do not put progress notes in `README.md`.

## Done

- `pyproject.toml`, `uv.lock`, `.env.example`, `.gitignore` (uv project, Python 3.13, Gemini env vars).
- `AGENTS.md` with the working rules, invariants, eval definitions, and collaboration mode.
- `README.md` without status notes.
- `data/faqs.json`: 30 FAQs, numeric ids `1`–`30`, TTL per category.
- `data/eval_set.json`: 100 rows, 60 positives (30 en / 30 hinglish), 20 near-miss, 20 out-of-domain, 50/50 dev/test.

## Not started

- `src/faqcache/` modules: config, embed, faqs, store, guards, cache, llm, bot, eval, cli.
- Tests. Data validator.
- Real threshold sweep and results files.
- README results line.

## Next

1. Human review of all 30 FAQ rows and all 100 eval rows. Fix labels now; never later to move a metric.
2. Validate the data: counts, unique ids, labels resolve, split is 50/50 and stratified.
3. Freeze both JSON files.
4. Start `config.py`, `faqs.py`, `embed.py` with a stub LLM, then the FAISS lookup and hit/miss flow.

## Decisions

- Numeric FAQ ids `1`–`30`, not `faq_01`.
- Eval ids `eval_001`–`eval_100`.
- Positives: 30 English, 30 Hinglish. One paraphrase per FAQ in dev and one in test, so every FAQ appears in both splits.
- TTL days: 3 for refunds, returns, payments; 7 for delivery, cancellation; 14 for orders, account support.
- Python floor stays 3.13.
- LLM stays Gemini via `google-genai`. Interface remains `LLMClient`.

## Open questions

- Confirm near-miss and out-of-domain rows after the human review pass.
- PII heuristic patterns are not designed yet.
