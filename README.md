<div align="center">

# 🩺 Medical Symptom Triage Prototype

**An explainable, retrieval-grounded symptom-triage engine — not a black box, and not a diagnosis.**

![Python](https://img.shields.io/badge/Python-3.9%2B-3776AB?style=for-the-badge&logo=python&logoColor=white)
![Dependencies](https://img.shields.io/badge/core%20deps-none-brightgreen?style=for-the-badge)
![Tests](https://img.shields.io/badge/tests-17%20passing-success?style=for-the-badge)
![Status](https://img.shields.io/badge/status-working%20prototype-orange?style=for-the-badge)
![License](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)

</div>

---

> ### ⚠️ Read this first
> This is a **research and learning prototype**. It is **not a medical device**, it has
> **not been clinically validated**, and it **does not provide a diagnosis**. It cannot
> examine you or know your history. If you are unwell, see a qualified healthcare
> professional. **In an emergency in India, call 112 (ambulance 108).**

---

## What this is

Give it a set of symptoms; it returns a ranked, explainable shortlist of possible
conditions, an honest confidence band, a static ICD-10 reference code, and next-step
advice that is **retrieved from a curated knowledge base — never generated freely**.

It was built to answer one question: *can a small, transparent triage system be both
useful and honest about what it does not know?* Every design choice leans toward
explainability and safety over a flashy but opaque answer.

## How it works

```
 free-text symptoms
        │
        ▼
 ┌──────────────────┐   normalise "temperature" → "fever",
 │  Knowledge Base  │   "loose motions" → "diarrhoea", ...
 │  (74 symptoms,   │
 │   22 conditions) │
 └────────┬─────────┘
          ▼
 ┌──────────────────┐   Naive Bayes over symptom likelihoods,
 │   Classifier     │   temperature-calibrated so it does NOT
 │                  │   output 100% / 0% for everything
 └────────┬─────────┘
          ▼
 ┌──────────────────┐   Tree-of-Thought:
 │  ToT Orchestrator│   1 GENERATE  top-N candidates
 │                  │   2 EVALUATE  re-rank by explanation power
 │                  │   3 DISCRIMINATE  best next question
 │                  │   4 CONCLUDE  ranked shortlist + confidence band
 └────────┬─────────┘
          ▼
 ┌──────────────────┐   static ICD-10 lookup
 │  ICD-10 Mapper   │
 └────────┬─────────┘
          ▼
 ┌──────────────────┐   retrieve advice from the KB,
 │  Advice Engine   │   constrain output to that text only,
 │  + Red Flags     │   flag emergency symptoms
 └────────┬─────────┘
          ▼
   explainable report + reasoning trace
```

## Quick start

The core engine and CLI use **only the Python standard library** — no `pip install` needed.

```bash
git clone https://github.com/NishantDas0079/Medical-Symptom-Triage-Prototype.git
cd Medical-Symptom-Triage-Prototype

# run the built-in demo
python -m src.cli --demo

# or describe your own symptoms
python -m src.cli --symptoms "fever, dry cough, loss of smell"

# see the recognised symptom vocabulary
python -m src.cli --list
```

Optional web UI (needs Streamlit):

```bash
pip install -r requirements.txt
streamlit run app.py
```

Run the tests:

```bash
python -m unittest discover -s tests      # or: pytest
```

## Example output

```
Reported symptoms: High fever, Severe headache, Pain behind the eyes, Joint pain, Skin rash

Confidence: HIGH

Most likely possibilities (model ranking):
  1. Dengue fever  (ICD-10 A90)
     probability 84% · explains 100% of your symptoms
  2. Migraine  (ICD-10 G43.9)
     probability 2% · explains 20% of your symptoms

Suggested next question: Do you have nausea?

About: Dengue fever  (ICD-10 A90)
Self-care / next steps:
  • Dengue needs prompt medical review and blood tests.
  • Drink fluids; use ONLY paracetamol for fever — avoid aspirin/ibuprofen (bleeding risk).
  • Seek emergency care for bleeding, severe abdominal pain, or persistent vomiting.
Who to see: General physician / emergency
```

## The knowledge base

`data/conditions.json` holds **22 curated conditions** and **74 symptoms** (with
free-text aliases), each condition carrying:

- a symptom likelihood table,
- a static ICD-10 code,
- grounded self-care advice,
- condition-specific red flags,
- a suggested specialist.

**Honesty note:** the likelihoods are **expert-estimated relative weights, not fitted
to patient data**, and the priors are illustrative. This is the single biggest
simplification in the project — see *Limitations*.

## Calibration (why it doesn't say "100%")

Raw Naive Bayes multiplies many likelihoods and collapses to ~100% / ~0%. This project
applies two standard corrections before the softmax:

1. **Normalisation** — divide the log-score by the number of symptoms observed
   (a geometric-mean likelihood), so a condition doesn't win by accumulating terms.
2. **Temperature scaling** — divide by `T` (default `0.7`) to soften the result.

The headline output is a **confidence band** (`low` / `medium` / `high`) derived from
the top score *and* the margin to the runner-up *and* how many symptoms were given —
because a bare percentage would over-claim precision. The percentage is a relative
model score, not a validated clinical probability.

## Safety design

- **Red-flag layer.** Emergency symptoms (chest pain, breathing difficulty, fainting,
  confusion, bleeding) trigger an urgent-care warning regardless of the model's ranking.
- **Retrieve-then-constrain.** Advice is *retrieved* from the KB and shown verbatim.
  An optional LLM step may rephrase it but is constrained to invent nothing. With no
  LLM (the default), output is guaranteed grounded.
- **No free generation.** The system never fabricates medical guidance.

## Project structure

```
Medical-Symptom-Triage-Prototype/
├── app.py                     # optional Streamlit UI
├── data/
│   └── conditions.json        # curated knowledge base
├── src/
│   ├── knowledge_base.py      # loading + free-text symptom normalisation
│   ├── classifier.py          # Naive Bayes + calibration
│   ├── triage.py              # Tree-of-Thought orchestrator
│   ├── icd10.py               # static ICD-10 mapping
│   ├── advice.py              # retrieve-then-constrain advice
│   ├── engine.py              # composes everything into a report
│   └── cli.py                 # command-line interface
├── tests/
│   └── test_triage.py         # 17 tests
├── requirements.txt
└── LICENSE
```

## Limitations (stated plainly)

- The KB has **22 conditions**; real triage needs hundreds. The `load_ddxplus()` hook in
  `knowledge_base.py` is a **documented extension point, not implemented**.
- Likelihoods and priors are **not fitted to data** — they encode expert intuition.
- **Not clinically validated.** No real-patient evaluation has been run.
- Single-turn: it does not yet follow up on its own discriminating question.
- ICD-10 codes are static references; real coding is context-dependent and should be verified.

## Roadmap

- [ ] **Ingest the public DDXPlus dataset** to scale from 22 curated conditions to 49, with fitted likelihoods.
- [ ] Fit the temperature on held-out data (proper calibration) instead of hand-tuning it.
- [ ] Multi-turn mode: ask the discriminating question and update beliefs.
- [ ] Add a held-out evaluation with top-k accuracy and calibration curves.
- [ ] Optional constrained-LLM rephrasing of retrieved advice.

## Tech stack

Python 3.9+ (standard library for the core) · optional Streamlit UI · zero core dependencies.

## License

MIT — see [LICENSE](LICENSE).

<div align="center"><sub>Built by Nishant Das · a learning project, kept honest on purpose.</sub></div>
