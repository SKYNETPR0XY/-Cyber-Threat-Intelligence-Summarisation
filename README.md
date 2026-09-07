# -Cyber-Threat-Intelligence-Summarisation
Evaluating Large Language Models for Automated Cyber Threat Intelligence Summarisation


 ctieval — Evaluating LLMs for Automated CTI Summarisation

A reproducible framework for benchmarking Large Language Models on cyber threat
intelligence (CTI) summarisation, and for measuring the hallucination they
introduce.

MSc Cyber Security and Forensics, University of Westminster · W18850154.

---

## What it does

`ctieval` runs a six-stage pipeline over three CTI source types (NVD CVE
records, AlienVault OTX pulses, APT/vendor report sections) and three models
(Claude Sonnet 4.6 via the Anthropic API; LLaMA 3 8B and Mistral 7B served
locally through LM Studio). It scores every summary with ROUGE and BERTScore,
classifies every hallucination against an adaptation of the Ji et al. (2023)
taxonomy using three complementary detectors, runs a blinded human evaluation,
and emits publication-ready tables and figures plus a drop-in Chapter 6
`results_pack.md`.

```
collect -> annotate -> run -> score -> detect -> report
                                    \-> humaneval -/
```

## Install

```bash
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
export PYTHONPATH=src                                   # or: pip install -e .
```

`bert-score` pulls in PyTorch. If it is not installed, or if its backbone
weights cannot be downloaded, the pipeline records a clearly-labelled TF-IDF
proxy instead and warns loudly — that proxy is **not** BERTScore and must be
resolved before producing results for submission.

## Credentials

Never put keys in `config.yaml`. Copy `.env.example` to `.env`, fill it in, and
export it:

```bash
cp .env.example .env      # then edit
set -a; source .env; set +a
```

Needed: `ANTHROPIC_API_KEY` (required for Claude), `NVD_API_KEY` (optional, lifts
the NVD rate limit), `OTX_API_KEY` (required for OTX collection).

## Set up the local models (LM Studio)

1. Install LM Studio and download `Meta-Llama-3-8B-Instruct` and
   `Mistral-7B-Instruct` in a 4-bit GGUF quant (Q4_K_M is a good default).
2. Load a model, open the **Developer** tab, and **Start Server** (default
   `http://localhost:1234`).
3. Copy the exact model identifier LM Studio shows into `config.yaml`
   (`models[].model_id`).
4. Confirm everything is reachable:

```bash
python -m ctieval preflight
```

LM Studio serves one model at a time by default. Either run the two open-source
models in separate passes (`run --models "LLaMA 3 8B"` then
`run --models "Mistral 7B"`), or enable multi-model serving and load both.

## Reproduce the experiment

```bash
# 1. Build the corpus (put report PDFs in data/apt_pdfs/ first)
python -m ctieval collect --source all --pdf-dir data/apt_pdfs

# 2. Author the OTX (and any unreferenced APT) summaries
python -m ctieval annotate --export                     # writes a worksheet CSV
#    ... fill in the reference_summary column, second-rater review ...
python -m ctieval annotate --import data/reference_summaries.csv

# 3. Generate summaries (resumable — safe to Ctrl-C and restart)
python -m ctieval run

# 4. Automated metrics + statistics
python -m ctieval score

# 5. Hallucination detection (add --judge "Claude Sonnet 4.6" for the LLM judge)
python -m ctieval detect

# 6. Adjudicate the detector's candidates. Detection produces CANDIDATES; until
#    a person has confirmed or rejected each one the only defensible figure is a
#    detector-flagged rate, not a hallucination rate.
python -m ctieval adjudicate --export results/adjudication.csv
#    ... set 'confirmed' to TRUE/FALSE on each row ...
python -m ctieval adjudicate --import results/adjudication.csv

# 7. Human evaluation. --n counts corpus ITEMS per source type; every model's
#    output for each is rated, so rows = n x source types x models.
python -m ctieval humaneval build --n 10                # 10 items/type -> 90 rows
#    ... two raters complete the workbook ...
python -m ctieval humaneval analyse --scores rater1=r1.csv --scores rater2=r2.csv

# 8. Figures, tables, and the Chapter 6 results pack
python -m ctieval report
open results/report/results_pack.md
```

### BERTScore is required, and `score` enforces it

`score` fails with a non-zero exit if BERTScore cannot be computed, rather than
quietly recording the TF-IDF proxy in its place — a results table must not carry
a column that is not the metric its caption names. Install `bert-score` and make
sure the backbone can be fetched from the Hugging Face hub before the real run:

```bash
pip install bert-score
python -c "import bert_score, transformers; print('ok')"
```

Pass `--allow-semantic-fallback` only for plumbing checks. It records
`semantic_sim_tfidf`, which is **not** BERTScore and must never be reported as
such.

## Offline dry run (no network, no GPU, no keys)

Validates the whole pipeline end to end against a synthetic corpus with mock
models. Every output is watermarked and cannot be reported as a result.

```bash
python -m ctieval demo
```

The dry run switches itself to `config/config.demo.yaml` if the active config
names real backends, and refuses to overwrite a corpus that holds non-synthetic
items, so it can never destroy collected data or spend API budget. Output lands
under `results/demo/`.

## Tests

```bash
python -m pytest tests -v
```

31 tests cover the schema, providers, metrics, statistics, the hallucination
detectors (including the CVSS-vector regression and a detector-validity check
against seeded errors), the human-eval statistics, the collectors, and the
reporting integrity guard.

## Integrity guarantees

- **No fabricated results.** The reporting stage refuses to build a results pack
  from a synthetic corpus unless `--allow-synthetic` is passed, and watermarks
  every such figure.
- **No leaked BERTScore.** The TF-IDF fallback is stored under a different column
  name and never labelled as BERTScore.
- **No leaked credentials.** Keys are read only from the environment; `.gitignore`
  excludes `.env` and all results.
- **Validated instrument.** The grounding detector's precision/recall against
  seeded errors is reported, so hallucination rates can be read with the
  detector's own accuracy in view.

## Layout

```
src/ctieval/
  schema.py         data structures + JSONL I/O
  config.py         YAML config; env-only credentials
  prompts.py        unified prompt template (+ mitigation variants)
  providers/        anthropic - lmstudio - mock, behind one interface
  collectors/       nvd - otx - apt_pdf - offline
  metrics.py        ROUGE - BERTScore(+fallback) - Friedman/Wilcoxon/Holm
  hallucination.py  grounding - llm_judge - selfcheck - taxonomy
  humaneval.py      blinded sheet - weighted kappa - ICC(2,k)
  report.py         figures - tables - results_pack.md (integrity-guarded)
  runner.py         resumable experiment runner
  cli.py            command-line interface
config/             config.yaml - config.demo.yaml
tests/              31-test suite
docs/               dissertation source + architecture figure
```

## Licence

Intended for release under the MIT Licence. Report PDFs are not redistributed;
the corpus records only content hashes and page references so a third party can
reproduce it from their own copies.
