# Vendor to PIM product matcher

Matches the product names in a vendor's file to the product names in your PIM.
Confident matches are accepted automatically; uncertain ones go to a review queue.

## Quick start

```bash
pip install -r requirements.txt
streamlit run app.py
```

Then open the app, upload the vendor file and the PIM export (CSV or Excel), check
the detected columns, and press **Run matching**. Try it with the files in `sample_data/`.

### Command line (no UI)

```bash
python -m pim_matcher.cli --vendor vendor.xlsx --pim pim.xlsx --out matches.xlsx
# optional flags: --vendor-name COL --pim-name COL --auto 90 --review 70 --llm
```

## How matching works

1. **Clean-up**: lower-casing, punctuation, Arabic letter variants and digits, abbreviations.
2. **Size parsing**: `1 LTR`, `1000 ml`, `1L` all become 1000 ml; `5x70g` becomes 350 g.
3. **Candidates**: TF-IDF over character n-grams finds the 40 most similar PIM names per vendor row,
   and an exact barcode (GTIN/EAN) match is always added when both files have barcodes.
4. **Scoring** (0 to 100): fuzzy name similarity, plus a bonus when size and brand agree and a penalty
   when they don't, or when model numbers differ (S23 vs S24).
5. **Status**
   - `auto`: score >= 90 and at least 3 points ahead of the runner-up, or a barcode match
   - `review`: score >= 70
   - `no_match`: everything else

All thresholds and penalties are sliders in the app (and fields of `MatchConfig` in code).

## Optional: Claude as a second opinion

Tick "Use Claude on rows that need review" in the app (or pass `--llm` on the CLI) and provide
`ANTHROPIC_API_KEY`. Claude chooses only among the candidates already found, and its pick is
pre-selected in the review queue. A human still confirms it. Model name is configurable
(`PIM_MATCHER_MODEL`).

## Output columns

`final_status` is one of `auto`, `confirmed`, `manual`, `no_match`, `no_match (reviewed)`, `unreviewed`,
with `final_pim_id` and `final_pim_name`. The file also keeps the score, the reason notes
(for example "size mismatch (500 g vs 1000 g)"), and the original vendor row number.

## Where to customise

| Want to change | File |
|---|---|
| Abbreviations (`choc` to `chocolate`), stop-words, units | `pim_matcher/normalize.py` |
| Scoring rules, thresholds, statuses | `pim_matcher/engine.py` |
| LLM prompt | `pim_matcher/llm.py` |
| Column auto-detection, export format | `pim_matcher/io.py` |
| Look and feel | `.streamlit/config.toml`, `app.py` |

## Known limits

- Matching is within one language. An English vendor name will not match an Arabic PIM name
  (the next upgrade would be multilingual embeddings in the candidate step).
- Very large files (500k+ rows) will work but take minutes; run the CLI for those.

## Tests

```bash
python -m pytest
```
