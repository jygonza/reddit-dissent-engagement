# Reddit Dissent — Replication Guide

This repository contains the four-notebook pipeline for **"Does Dissent Accelerate Engagement in Reddit Discussions?"** — a study of whether substantive disagreement in Reddit comment threads increases downstream thread activity.

All notebooks are designed to run in **Google Colab** with data stored on **Google Drive**.

---

## Repository Structure

```
0.scraper.ipynb          # Stage 0 — scrape raw posts and comments from Reddit
1.merging.ipynb          # Stage 1 — build (parent comment, reply) pairs
2_Dissent_Label.ipynb    # Stage 2 — classify each reply's stance with Gemini
3.Engagment.ipynb        # Stage 3 — compute engagement metrics and test hypotheses
```

Run the notebooks **in order**. Each stage produces the files that the next one expects.

---

## Prerequisites

### Google Drive folders

The notebooks read and write to two folders on your Google Drive. Create both before starting:

```
MyDrive/dissent_project/        ← Stage 0 scraper output
MyDrive/My_Dissent_project/     ← Stages 1–3 input and output
```

All intermediate and final outputs will be written here automatically.

### Notebook 1 — Hugging Face account

The raw Reddit data is downloaded from Hugging Face Hub. You need a (free) Hugging Face account and a read-access token.

1. Create an account at [huggingface.co](https://huggingface.co)
2. Go to [huggingface.co/settings/tokens](https://huggingface.co/settings/tokens) and create a token with **Read** permissions
3. Store the token as a **Colab Secret** named `HF_TOKEN`:
   - In Colab, open the left sidebar → **Secrets** (key icon) → **+ Add new secret**
   - Name: `HF_TOKEN`, Value: your token

### Notebook 2 — Google Cloud project

The labeling notebook submits batch jobs to **Vertex AI** and stores intermediate data in **Google Cloud Storage**.

1. Create or use an existing [Google Cloud project](https://console.cloud.google.com)
2. Enable the following APIs in your project:
   - Vertex AI API
   - Cloud Storage API
3. Note your **Project ID** — you will paste it into the notebook
4. Ensure your Google account has **Vertex AI User** and **Storage Admin** roles on that project

---

## Running the Pipeline

### Stage 0 — `0.scraper.ipynb`

**What it does:** Collects raw Reddit posts and comments for each subreddit using the [Arctic Shift API](https://arctic-shift.photon-reddit.com) — a public, no-auth-required archive of Reddit data. Data is downloaded in year-bounded slices (parallelized by year), then merged and filtered into clean per-subreddit CSVs.

**Steps:**

1. Open `0.scraper.ipynb` in [Google Colab](https://colab.research.google.com)
2. Mount your Google Drive (`/content/drive`)
3. No API key is required — the Arctic Shift API is publicly accessible
4. Configure the target subreddits and year ranges in the designated config cells:
   ```python
   SUBREDDITS = ["amitheasshole"]        # subreddits to scrape
   YEAR_RANGE_CHUNKS = [(2022, 2026)]    # year ranges to fetch
   MIN_COMMENTS = 5                      # minimum comments per post
   ```
5. Run all cells in order. The notebook will:
   - Fetch posts and comments in paginated batches, advancing via `created_utc` cursor
   - Write per-year chunk CSVs to `MyDrive/dissent_project/chunks/`
   - Filter out deleted/removed posts and comments, Automoderator entries, and posts below `MIN_COMMENTS`
   - Merge all chunks into final per-subreddit files
   - Log scrape metadata to `scrape_log.json`

> **Note:** Large subreddits across multi-year ranges can take several hours. The notebook supports **resumable runs** — progress files track the last fetched timestamp so interrupted scrapes can pick up where they left off.

6. **Output:** per-subreddit files written to `MyDrive/dissent_project/`:
   - `{subreddit}_posts.csv` — filtered submissions
   - `{subreddit}_comments.csv` — filtered comments with `post_id` and `post_author` attached
   - `{subreddit}_scrape_log.json` — scrape metadata and row counts

---

### Stage 1 — `1.merging.ipynb`

**What it does:** Downloads raw Reddit comment data from Hugging Face Hub and constructs `(parent comment, reply comment)` pairs for five subreddits.

**Steps:**

1. Open `1.merging.ipynb` in [Google Colab](https://colab.research.google.com)
2. When prompted, mount your Google Drive (`/content/drive`)
3. The first cell installs dependencies:
   ```
   datasets  huggingface_hub  hf_transfer
   ```
4. The notebook reads your `HF_TOKEN` from Colab Secrets — no manual token entry needed
5. Run all cells in order
6. **Output:** five CSV files written to `MyDrive/My_Dissent_project/merged_subreddits/`:
   - `politicalopinions.csv`
   - `the10thdentist.csv`
   - `unpopularopinion.csv`
   - `changemyview.csv`
   - `amitheasshole.csv`

---

### Stage 2 — `2_Dissent_Label.ipynb`

**What it does:** Classifies each reply against its parent comment using Gemini 2.5 Flash Lite via Vertex AI batch prediction. Labels each row as `substantive_dissent`, `agreement`, `neutral`, or `social_disagreement`.

**Steps:**

1. Open `2_Dissent_Label.ipynb` in Google Colab
2. Mount your Google Drive
3. The first setup cell installs dependencies:
   ```
   google-genai  nest_asyncio  google-cloud-storage
   ```
4. **Set your Project ID:** find the cell containing `PROJECT_ID = ""  # REPLACE` and fill in your Google Cloud project ID:
   ```python
   PROJECT_ID = "your-project-id-here"
   ```
5. Run the authentication cell — Colab will prompt you to sign in with your Google account
6. Run all cells in order. The notebook will:
   - Chunk the merged CSVs into ~50,000-row files
   - Upload batch requests to Google Cloud Storage
   - Submit Vertex AI batch prediction jobs
   - Harvest results and write labeled CSVs
   - Automatically retry any rows that returned without a label

> **Note:** Vertex AI batch jobs can take several hours depending on data volume. The notebook includes monitoring cells to check job status before harvesting. Do not run the harvest cells until jobs show as `JOB_STATE_SUCCEEDED`.

7. **Output:** labeled CSVs written to `MyDrive/My_Dissent_project/labeled_chunks/<SUB>_labeled/`
   - Each file contains the original columns plus `label`, `label_name`, and `confidence`

---

### Stage 3 — `3.Engagment.ipynb`

**What it does:** Reconstructs thread structure, engineers engagement and dissent-treatment variables, and tests three hypotheses about whether dissent predicts thread engagement.

**Steps:**

1. Open `3.Engagment.ipynb` in Google Colab
2. Mount your Google Drive
3. No additional package installation is required — the notebook uses libraries available in Colab by default (`pandas`, `scipy`, `statsmodels`, etc.)
4. Run all cells in order. The notebook will:
   - Reassemble labeled chunks into complete per-subreddit CSVs (Cell 1.5 handles this automatically if needed)
   - Rebuild thread trees and compute reply depth
   - Compute full-thread and post-dissent engagement metrics
   - Build composite engagement scores (z-score averages)
   - Run Spearman correlations and OLS regressions for H1, H2, and H3
5. **Output:** two analysis files written to `MyDrive/My_Dissent_project/`:
   - `posts_with_engagement.csv` — thread-level features and engagement scores
   - `comments_with_features.csv` — comment-level features with dissent flags

---

## Expected Google Drive Layout

After running all four stages, your Drive folders should look like this:

```
MyDrive/dissent_project/               ← Stage 0 output
├── chunks/
│   └── {subreddit}_{posts|comments}_{year}.csv
├── {subreddit}_posts.csv
├── {subreddit}_comments.csv
└── {subreddit}_scrape_log.json

MyDrive/My_Dissent_project/            ← Stages 1–3
├── merged_subreddits/
│   ├── politicalopinions.csv
│   ├── the10thdentist.csv
│   ├── unpopularopinion.csv
│   ├── changemyview.csv
│   └── amitheasshole.csv
├── chunks/
│   └── <SUB>_chunks/
│       └── <SUB>_chunk_NNN.csv
├── labeled_chunks/
│   └── <SUB>_labeled/
│       └── <SUB>_chunk_NNN_labeled.csv
├── labeled_complete/          ← assembled by Stage 3 if missing
│   └── <SUB>_labeled.csv
├── batch_jobs.json
├── batch_jobs_retry.json
├── posts_with_engagement.csv
└── comments_with_features.csv
```

---

## Secrets and Credentials

**Never hardcode tokens or project IDs in notebook cells.** Use the following safe alternatives:

| Secret | Safe approach |
|---|---|
| Hugging Face token | Store as Colab Secret `HF_TOKEN`; read with `userdata.get("HF_TOKEN")` |
| Google Cloud Project ID | Paste only into the designated `PROJECT_ID = ""` cell; do not commit |

If you fork this repo, confirm that no tokens or project IDs appear in notebook cells before pushing.

---

## Citation

If you use this pipeline or its outputs, please cite this repository.
