# ICLR review experiments

A research pipeline that generates ICLR-style reviews and compares them with human reviews from OpenReview. It supports single-paper and batch runs with structured output and comparison reports.

In the OpenReview workflow, the model receives the paper title and abstract. TF-IDF overlap measures textual similarity; it is not a measure of review quality.

## Setup

- Install Python 3.11 (or compatible)
- In the project root:

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Create a local `.env` file with `ANTHROPIC_API_KEY` set to your own key. Keep it outside version control. `ANTHROPIC_MODEL` can be set to select a model available to your account.

## Data: OpenReview ICLR (get the raw data first)

Fetch ICLR submissions + reviews from OpenReview and write JSONL files to `data/raw/iclr/`:

```bash
python -m data.ingest.openreview_iclr --years 2024 --out-dir data/raw/iclr
```

This writes `data/raw/iclr/iclr_{year}.jsonl`. Each record is one **(paper, review)** pair and includes paper fields (e.g. `paper_id`, `year`, `title`, `abstract`) plus review fields (e.g. `review_id`, `review_text`, `ratings`, and when available `strengths`, `weaknesses`, `questions`).

Optional dedupe + normalization (no repeated title/abstract):

```bash
python -m data.ingest.dedupe_iclr --input-dir data/raw/iclr --normalize
```

## Running the pipeline on a single paper

Once you have `data/raw/iclr/iclr_2024.jsonl`, run the pipeline on a single (paper, review) row:

```bash
python run.py --from-iclr data/raw/iclr/iclr_2024.jsonl --line 0 --out-dir ./out
```

- `--from-iclr` points to the ICLR JSONL (one row per (paper, review))
- `--line` selects which row to use (0-based index)

This writes:

- `out/generated_review.txt` — LLM-generated review (structured like an ICLR review)
- `out/generated_review.json` — same review, split into sections (Review, Overall score, Strengths, Weaknesses, etc.)
- `out/metadata.json` — paper_id, review_id, year, and the full human review text + ratings
- `out/metrics.json` — topical overlap score between generated and human review

You can also run on your own JSON files:

```bash
python run.py --paper path/to/paper.json --reviewer path/to/reviewer.json --out-dir ./out
python run.py --paper path/to/paper.json --reviewer path/to/reviewer.json --human-review path/to/human_review.txt --out-dir ./out
```

Paper JSON can contain `title` + `text`, or `content`, or a `path` to a text file. Reviewer JSON can contain `bio` and/or `expertise`. Human review is a plain text file.

## Batch runs and evaluation report

To run on many papers at once and get an evaluation report:

```bash
python run_batch.py --from-iclr data/raw/iclr/iclr_2024.jsonl --num-papers 50 --out-dir out/batch_50
```

This creates:

- `out/batch_50/paper_*/` — one folder per paper with generated review, metadata, and metrics
- `out/batch_50/summary.json` — JSON summary by human verdict category
- `out/batch_50/evaluation_report.md` — markdown report with overall mean overlap, per-category stats, and a per-paper table

You can also rebuild the summary and report from existing `paper_*` folders (no new API calls):

```bash
python run_batch.py --out-dir out/batch_50 --build-from-existing
```

In this repo, `out/batch_212/` contains a run on 212 ICLR 2024 papers and an `evaluation_report.md` that summarizes how close the generated reviews are to the real ones.

## Project layout

- `run.py` — single-paper pipeline (build prompt, call Claude, parse, compute metrics)
- `run_batch.py` — batch runner + evaluation report
- `data/ingest/openreview_iclr.py` — fetch ICLR data from OpenReview
- `data/ingest/dedupe_iclr.py` — dedupe and (optionally) normalize to papers_*.jsonl + reviews_*.jsonl
- `pipeline/prompt/templates/` — `system.txt` and `review.txt` templates used to prompt the LLM

## Interpretation

The generated review and human review may have different information available to them. The topical-overlap metric captures shared vocabulary, with optional comparisons of strengths and weaknesses. Use it as an exploratory comparison rather than evidence that a model can replace a reviewer. Existing batch outputs are recorded artifacts; rerunning generation makes external API requests.
