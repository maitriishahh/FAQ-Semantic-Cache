# Status

Last updated: 2026-11-19. Update this file after each working session. Do not put progress notes in `README.md`.

## Done

- `pyproject.toml`, `uv.lock`, `.env.example`, `.gitignore` (uv project, Python 3.13, Gemini env vars).
- `AGENTS.md` with the working rules, invariants, eval definitions, and collaboration mode.
- `README.md` without status notes.
- `data/faqs.json`: 30 FAQs, numeric ids `1`–`30`. Review fixes applied (FAQ 12 moved to delivery, FAQ 25 reworded).
- `data/eval_set.json`: 100 rows. Review fixes applied. 60 positives (30 en, 24 hinglish, 6 hi), 20 near-miss, 20 out-of-domain, 50/50 dev/test.

## Not started

- `src/faqcache/` modules: config, embed, faqs, store, guards, cache, llm, bot, eval, cli.
- Tests. Data validator.
- Real threshold sweep and results files.
- README results line.

## Next

1. Freeze both JSON files. Further edits only to fix a real mistake, never to move a metric.
2. Start `config.py` and `faqs.py`, then the data validator test.
3. `embed.py` and the FAISS lookup with a stub LLM.

## Known eval notes

Do not "fix" these. Cite them in the failure analysis if they show up.

- Negation guard cost, keep as-is: `eval_015`, `eval_016`, `eval_033`, `eval_034` are correct positives whose FAQ questions have no negation word. With the guard on they become false misses. That cost is the point of the before/after comparison.
- Negation hard negatives added so the guard can show a benefit: `eval_063`, `eval_065`, `eval_071`, `eval_079`.
- Likely false hits to explain, not relabel: `eval_064` (sale items vs "are all items returnable"), `eval_062` (delivery-fee refund vs refused-delivery wording), `eval_061` (refund without return vs "after we receive your return").
- Slight wording gap, label stays: `eval_035` says same-day delivery; FAQ 18 says same-day dispatch.
- Close but correctly null, metrics unaffected: `eval_080`, `eval_094`, and out-of-domain `eval_085`, `eval_092`, `eval_093`, `eval_094`, `eval_099`, `eval_100`.

## Decisions

- Numeric FAQ ids `1`–`30`, not `faq_01`.
- Eval ids `eval_001`–`eval_100`.
- Positives: 30 English, 24 Hinglish, 6 Hindi (Devanagari). One paraphrase per FAQ in dev and one in test. 3 Hindi positives in each split.
- TTL days: 3 for refunds, returns, payments; 7 for delivery, cancellation; 14 for orders, account support. FAQ 12 (delivery fee) is delivery, ttl 7.
- Python floor stays 3.13.
- LLM stays Gemini via `google-genai`. Interface remains `LLMClient`.

## Open questions

- PII heuristic patterns are not designed yet.
