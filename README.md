<h1 align="center">🏢 Business Entity Resolution</h1>

<h3 align="center">Matching the same business across three noisy data sources: US, India and France</h3>

<p align="center">
  Blocking with 9 keys → a pruner that keeps the top 15 candidates → two LightGBM models → one-owner rule.
</p>

###

<p align="center">
  <img src="https://img.shields.io/badge/Amazon%20ML%20Challenge-2026-FF9900?style=for-the-badge&logo=amazon&logoColor=white" alt="challenge" />
  <img src="https://img.shields.io/badge/metric-F0.5%20(macro)-232f3e?style=for-the-badge" alt="metric" />
  <img src="https://img.shields.io/badge/external%20data-none-10b981?style=for-the-badge" alt="no external data" />
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Python-3.10-3776AB?style=for-the-badge&logo=python&logoColor=white" alt="python" />
  <img src="https://img.shields.io/badge/LightGBM-4.7.0-9ACD32?style=for-the-badge" alt="lightgbm" />
  <img src="https://img.shields.io/badge/pandas-2.3.3-150458?style=for-the-badge&logo=pandas&logoColor=white" alt="pandas" />
  <img src="https://img.shields.io/badge/scikit--learn-1.7.2-F7931E?style=for-the-badge&logo=scikitlearn&logoColor=white" alt="sklearn" />
  <img src="https://img.shields.io/badge/RapidFuzz-3.14.5-6A5ACD?style=for-the-badge" alt="rapidfuzz" />
  <img src="https://img.shields.io/badge/Amazon%20EC2-ED7100?style=for-the-badge&logo=amazonec2&logoColor=white" alt="ec2" />
</p>

###

## 🚀 Quick start: reproduce the submission

```bash
# 1. go to this folder
cd code/business_entity_resolution

# 2. create an environment (Python 3.10) and install the pinned packages
python -m venv venv
source venv/bin/activate            # Windows: venv\Scripts\activate
pip install -r requirements.txt

# 3. run the full pipeline (train + test, end to end)
python src/pipeline.py \
    --train-dir /path/to/dataset/train \
    --test-dir  /path/to/dataset/test \
    --out-dir   output
```

When it finishes, the two submission files are in `output/`:

| File | Content |
|---|---|
| `output/matching_results.tsv` | final matches: one row per test S1 record, matched S2/S3 ids comma-separated (empty = no match) |
| `output/candidate_pairs.tsv` | the candidates the matching model scored (top 15 per S1 after blocking and pruning) |

Optional: check the format with the official validator (run from `student_resource/`):

```bash
python3 utils/validate_submission.py \
  --matching  <path>/output/matching_results.tsv \
  --candidate <path>/output/candidate_pairs.tsv \
  --test-dir  dataset/test
```

###

## 📦 What you need

**Data** (the competition files, unchanged):

```
dataset/train/   train_source1.tsv  train_source2.tsv  train_source3.tsv  train_ground_truth.tsv
dataset/test/    test_source1.tsv   test_source2.tsv   test_source3.tsv
```

The pipeline **trains every model from scratch** on the training data, then predicts the test data, so both folders are needed.

**Everything else is inside this folder:**
- `src/pipeline.py`: the full pipeline in one file
- `src/dictionaries/`: 6 small rule files read by the cleaning step (they must stay **next to** `pipeline.py`)
- `requirements.txt`: pinned package versions, **including `symspellpy`** (the run stops with an error if it is missing, instead of silently giving different results)

**No hard-coded paths:** the data folders are passed on the command line, and the dictionaries are found relative to `pipeline.py`, so it runs from any folder on Windows or Linux.

**Hardware:** the run processes tens of millions of candidate pairs, so it needs a machine with plenty of RAM and several hours. The submitted results were produced on an **AWS EC2** instance.

###

## 🧠 How it works

```
 raw S1 / S2 / S3 (train + test)
          │
          ▼
 STEP 1   clean ── anyascii (9 Indian scripts + French accents) → dictionaries → leet fix → legal-form split
 STEP 1b  spell-correct rare typos ("tecnology" → "technology")
          │
          ▼
 STEP 2-3 blocking ── 9 keys (A B C D E F G AC FC), per country, with bucket limits   (instead of ~22 trillion pairs)
          │
          ▼
 STEP 4   3 quick scores → pruner (LightGBM) → keep the top 15 candidates per S1     → candidate_pairs.tsv
          │
          ▼
 STEP 5   32 pair features (name, address, numbers, legal form, TF-IDF, sound skeleton, rare words)
          │
          ▼
 STEP 6   model 1 (32 features) → sibling features → model 2 (35 features); threshold chosen on F0.5
          │
          ▼
 STEP 7-8 predict test → threshold + one-owner rule                                  → matching_results.tsv
```

| Step | What it does | Why |
|---|---|---|
| **1. Clean** | `anyascii` converts any script to English letters **first**, then lowercase, symbols, abbreviation dictionaries, leet fix (`5urat` → `surat`), and the legal form (`pvt ltd`, `llc`, `sarl` …) is split from the name | names appear in 9 Indian scripts and with accents; legal words are shared by thousands of unrelated companies |
| **1b. Spell fix** | a word seen **once** that is 1–2 letters from a word seen **5,000+** times is corrected | typos were the largest group of lost matches; strict settings so genuine names are not changed |
| **2–3. Blocking** | records share a bucket if they share a key; buckets above a size limit are skipped; done separately per country | comparing everything is impossible; 100% of true train pairs are in the same country |
| **4. Pruner** | 3 fast similarity scores + a small model keep the 15 best candidates per S1 | removes ~80% of pairs while keeping 99.7%+ of the true pairs found by blocking |
| **5. Features** | 32 similarity features per pair | each catches a different kind of difference (typos, word order, joined words, transliteration, house numbers …) |
| **6. Models** | model 1 on the 32 features; model 2 adds **sibling features** (similarity to the S1's confident matches) | a business often appears several times in S2/S3 |
| **8. Decide** | threshold chosen on the competition F0.5, then **one-owner rule**: each S2/S3 record keeps only its best S1 | every S2/S3 record belongs to exactly one S1 in the training data; F0.5 punishes wrong matches most |

**Blocking keys**

| Key | Built from | Limit |
|---|---|---:|
| A | Soundex code of each name word, sorted | 200 |
| B | 3 rarest name words (S1 uses 2) | 50 |
| C | 3 rarest address words | 100 |
| D | house number + rare address word | 50 |
| E | rare name word + rare address word | 50 |
| F | name without spaces, first 10 letters | 100 |
| G | sound skeleton of the name (`private`, `praivet` → `prbt`) | 200 |
| AC / FC | key A or F + rare address word | 50 |

###

## 📊 Results measured during development (training data)

| Measure | Value |
|---|---:|
| Blocking pair recall (keys A–F) | 94.82% |
| F0.5 ceiling after blocking (perfect model) | 98.02% |
| True pairs kept by the top-15 shortlist | 99.77% (India) / 99.94% (US) |
| Model 2 with sibling features vs. model 1 | 93.28% → 93.74% F0.5 |
| 5 new features (TF-IDF, skeleton, rare words) | +0.70 points F0.5 |
| One-owner rule | +0.10 points F0.5 |

These numbers come from the research notebooks (earlier pipeline versions, held-out training S1 records). The final pipeline prints its own validation F0.5 for each threshold in step 6.

###

## 📁 Repository structure

```
business_entity_resolution/
├── src/
│   ├── pipeline.py                   # the full pipeline (steps 1-8), used for the submitted results
│   ├── dictionaries/                 # rule files read in step 1
│   │   ├── name_abbreviations.txt    # limited -> ltd, private -> pvt, compagnie -> cie ...
│   │   ├── address_abbreviations.txt # street -> st, road -> rd, maharashtra -> mh, av -> ave ...
│   │   ├── address_phrases.txt       # west bengal -> wb, tamil nadu -> tn ...
│   │   ├── legal_words.txt           # llc, pvt, ltd, inc, sarl, sas, eurl ...
│   │   ├── indic_words.txt           # 533 transliterated Indian-script words -> English
│   │   └── hindi_words.txt           # older Hindi-only map (only used if indic_words.txt is missing)
│   └── notebooks/                    # research behind each step (not needed to run)
│       ├── explore_data.ipynb        # cleaning rules and dictionaries
│       ├── create_bucket.ipynb       # blocking keys and bucket limits
│       ├── feature.ipynb             # pruner, shortlist, features, Indian-script word map
│       └── model.ipynb               # models, threshold, one-owner rule, error analysis
├── README.md
├── requirements.txt
├── dictionary_evidence.md            # evidence for every dictionary rule
└── research_to_pipeline.md           # every research idea and whether it is in the pipeline
```

While running, the pipeline writes its intermediate files to `v2/` (in the folder it is run from) and the two output files to `--out-dir`.

###

## 📚 Dictionaries: how they were made

| File | How it was created |
|---|---|
| `name_abbreviations.txt`, `address_abbreviations.txt`, `address_phrases.txt` | abbreviation candidates **mined from 345,968 matched training pairs** (e.g. `street → st` seen 19,466 times), then reviewed by hand to remove typos and false matches; French rules from word counts of the unlabeled test addresses. Evidence: `dictionary_evidence.md`, `explore_data.ipynb` |
| `legal_words.txt` | most frequent last words of company names (US, India) and French legal forms (`explore_data.ipynb`) |
| `indic_words.txt` | **learned automatically** from 551,240 matched training pairs between English names and names in 9 Indian scripts; on held-out businesses, name similarity rose from 87.5 to 99.2 (`feature.ipynb`, cells L1–L2) |

###

## 🔁 Reproducibility notes

- **Fixed random seeds** everywhere (42, 7, 3, 1), so the same data splits are used on every run.
- **Do not change** `use_frac` (must stay `1.0`), the model settings, `TOP_K`, or the bucket limits: the submitted results used these values.
- `run_all` **skips steps whose files already exist** in `v2/`, so a stopped run can be continued. To start from scratch, delete `v2/` (or call `run_all(..., fresh=True)`).
- Package versions are pinned to the development environment (Python 3.10). The original EC2 environment was not preserved; `symspellpy` is pinned to the latest release at the time of the final run.
- The models are retrained on every run. Small differences in a few matches are possible on other machines (package builds, number of CPU threads in LightGBM). **The files in the submission's `output/` folder are the exact files uploaded to the leaderboard.**
- The notebooks contain local Windows paths from development; they are research records and are **not** run by the pipeline.

###

## ✅ Competition rules

- **No external data, APIs or services:** all rules and models come from the provided training data. The test set's text (never labels) is used only to learn vocabulary: French abbreviations, word rareness and TF-IDF letter patterns.
- **Model:** LightGBM (MIT license), gradient-boosted trees, far below the 8B-parameter limit. No pretrained or language models are used.
- **Outputs** follow the required format: one row per test S1 record, S2/S3 ids only, no duplicates; `matching_results` ⊆ `candidate_pairs`.

###

<p align="center">
  <br>
  Built for the <b>Amazon ML Challenge 2026</b> by <b>Ayushi Pandey</b>
  <br><br>
</p>
