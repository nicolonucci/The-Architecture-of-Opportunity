# 🎯 The Architecture of Opportunity
### A Deterministic Data Pipeline for Market-Aware Content Ideation

> *"Can subjective content ideation be replaced by deterministic data pipelines?"*

**Authors:**  Nicolò Nucci · Francesco Colombini 
**Course:** Data Management Project

---

## Abstract

The creator economy faces a fundamental information asymmetry: platforms provide abundant retrospective analytics but negligible predictive support. This project engineers a three-tier local architecture that synthesizes search intent signals (**Demand**) from Google Trends with empirically validated performance metrics (**Supply**) from YouTube. Through rigorous ELT enforcement and statistical quality constraints, creative ideation is treated as a data engineering challenge — where generative outputs are constrained by statistical evidence rather than plausible speculation.

---

## System Architecture
```
┌─────────────────────────────────────────────────────┐
│              User Interface & Orchestration          │
│                  (User_interface.ipynb)              │
└──────────────────────┬──────────────────────────────┘
                       │
       ┌───────────────┼───────────────┐
       ▼               ▼               ▼
┌─────────────┐ ┌─────────────┐ ┌──────────────┐
│  Acquisition │ │  Data Quality│ │  LLM Reasoning│
│  & Storage  │ │  & Analysis  │ │  (GPT-4.1)   │
│  (.ipynb)   │ │  (.ipynb)    │ │  (.ipynb)    │
└──────┬──────┘ └──────┬───────┘ └──────┬───────┘
       │               │                │
       └───────────────┼────────────────┘
                       ▼
              ┌─────────────────┐
              │   SQL Storage   │
              │  (SQLite .db)   │
              └─────────────────┘
```

The pipeline follows an **ELT** (Extract, Load, Transform) pattern with 4 functional layers:

| Layer | Notebook | Role |
|-------|----------|------|
| Data Ingestion & Staging | `Acquisition_and_storage.ipynb` | YouTube API + Google Trends CSV → SQL |
| Data Quality & Warehousing | `Data_quality_and_analysis.ipynb` | Cleaning, validation, feature engineering |
| LLM Reasoning | `LLM_process.ipynb` | Constrained content ideation via GPT-4.1-mini |
| Orchestration | `User_interface.ipynb` | Single entry point, parameter management |

---

## Quick Start

### 1. Prerequisites
```bash
pip install openai pandas sqlite3 scipy matplotlib seaborn jupyter
```

### 2. Setup API Keys

Run **`Key_creator.ipynb`** to store your YouTube Data API keys:
```python
youtube_keys = ["YOUR_KEY_1", "YOUR_KEY_2", ...]
# ~10 keys recommended for quota resilience
```

Run **`Open_ai_key.ipynb`** to store your OpenAI API key:
```python
OpenAI = ["YOUR_KEY"]
```

### 3. Download Google Trends Data

Go to [https://trends.google.com/trends/](https://trends.google.com/trends/) and download CSV files for your topic:

| Type | Time Windows |
|------|-------------|
| Rising queries | 7, 14, 30, 60 days |
| Top queries | 7, 14 days |

Place all CSV files in the project folder. Files **must** follow this naming convention:
```
searched_with_(top|rising)-queries_{CC}_{YYYYMMDD-HHMM}_{YYYYMMDD-HHMM}.csv
```

### 4. Run the Pipeline

Open **`User_interface.ipynb`** and provide:

| Parameter | Description | Default |
|-----------|-------------|---------|
| `Search topic` | Topic/keyword to analyze (**required**) | — |
| `YouTube API key file` | Path to keys file | `youtube_keys.txt` |
| `YouTube end date` | Last date for video collection (dd/mm/yyyy) | Today |
| `YouTube lookback days` | Days to look back | `14` |
| `YouTube window days` | Temporal resolution (1 = daily) | `1` |
| `Google Trends CSV folder` | Path to CSV files | Current dir |
| `OpenAI key file` | Path to OpenAI key | `Open_AI_key.txt` |
| `Output directory` | Where to save results | `./output/` |

The pipeline then runs **fully automatically**.

---

## Project Structure
```
project-root/
├── User_interface.ipynb          ← START HERE
├── Acquisition_and_storage.ipynb
├── Data_quality_and_analysis.ipynb
├── LLM_process.ipynb
├── Key_creator.ipynb
├── Open_ai_key.ipynb
├── youtube_keys.txt
├── Open_AI_key.txt
├── searched_with_rising-queries_*.csv  (×4 windows)
├── searched_with_top-queries_*.csv     (×2 windows)
│
└── {topic}_Outputs/
    ├── storage/
    │   └── {topic}_engine.db              ← SQLite database
    ├── exports/
    │   ├── {topic}_discovery.ndjson
    │   ├── {topic}_stats.ndjson
    │   ├── {topic}_google_trends_rising.csv
    │   ├── {topic}_google_trends_top.csv
    │   └── {topic}_llm_result.txt         ← Final output
    └── analysis/
        ├── complete/
        │   ├── Master_COMPLETE_Dashboard.png
        │   └── {topic}_complete_eda_report.txt
        └── interesting/
            ├── Master_INTERESTING_Dashboard.png
            └── {topic}_interesting_eda_report.txt
```

---

## Youtube Core Metrics & Scoring

### Feature Engineering

| Metric | Formula | Meaning |
|--------|---------|---------|
| Exposure Efficiency (EE) | `Views / Subscribers` | Algorithmic reach beyond base audience |
| Engagement Rate (ER) | `(Likes + Comments) / Views` | Audience interaction intensity |
| View Percentile | `PERCENT_RANK() OVER (ORDER BY views)` | Relative visibility position |

> All metrics are **percentile-normalized** to handle power-law distributions.

### Composite Scoring

**Best Content Score** — proven high-performers:
```
BCS = 0.4 × ER_pct + 0.3 × EE_pct + 0.3 × View_pct
```

**Opportunity Score** — underexposed hidden gems:
```
OS = 0.4 × ER_pct + 0.4 × (1 − EE_pct) + 0.2 × (1 − View_pct)
```

The top 20% by each score are unioned into the **"Interesting Videos"** set used by the LLM.

##  Google Trends Signal Processing

The pipeline treats **Rising** and **Top** queries as fundamentally different demand signals and processes them through independent validation and ranking workflows.

---

###  Rising Queries — Emerging Trend Detection

Rising queries capture anomalous or accelerating search growth. Each query is evaluated across **4 temporal windows** (7, 14, 30, 60 days) and assigned a **4-digit binary fingerprint** encoding whether a breakout signal was detected in each window.
```
Pattern "1011" → breakout detected in 60d ✓ | 30d ✗ | 14d ✓ | 7d ✓
Pattern "0001" → brand-new spike, only in 7d window (highest priority)
Pattern "1111" → persistent trend across all windows
```

A **log-weighted recency score** is computed before aggregating across windows:
```
row_score = window_weight × log(1 + growth_pct)
```

| Window | Weight | Rationale |
|--------|--------|-----------|
| 7 days | **8×** | Most actionable, highest recency |
| 14 days | **4×** | Short-term confirmation |
| 30 days | **2×** | Medium-term context |
| 60 days | **1×** | Baseline reference |

Queries are ranked **first by temporal pattern** (structural trend shape), then by recency score within each group. The **top 30%** are selected as canonical rising demand signals.

---

###  Top Queries — Stable High-Demand Detection

Top queries represent established, high-volume search interest rather than sudden growth. Analysis is restricted to **short-term windows only** (7 and 14 days), reflecting their role as indicators of current sustained demand.

Each query receives a **2-digit binary fingerprint**:
```
Pattern "11" → present in both 7d and 14d windows  (highest priority)
Pattern "01" → present only in 7d window
Pattern "10" → present only in 14d window
```

When a query appears in both windows, the **7-day signal takes priority** as the more recent and actionable source. A composite score is then computed:
```
final_score = 0.7 × search_interest + 0.3 × growth_pct
```

> The 70/30 weighting deliberately prioritizes **absolute search volume over growth rate** — Top queries represent established demand, not emerging spikes.

Queries are ranked hierarchically by pattern (`11 > 01 > 10`), then by composite score within each group. The **top 20%** are selected as canonical stable demand signals.

---

###  How the Two Signals Combine

| Dimension | Rising Queries | Top Queries |
|-----------|---------------|-------------|
| **Signal type** | Emerging / accelerating trends | Stable, high-volume demand |
| **Time windows** | 7, 14, 30, 60 days | 7, 14 days only |
| **Fingerprint** | 4-digit binary (`1011`) | 2-digit binary (`11`) |
| **Scoring priority** | Recency-weighted growth | Absolute search interest |
| **Selection threshold** | Top 30% | Top 20% |
| **LLM role** | Secondary angle, CTR boost | Core SEO foundation |

In the LLM prompt, **Top queries anchor the content strategy** (titles, primary keywords), while **Rising queries add topical freshness** as secondary angles — without becoming the main subject unless also supported by Top query data.

---

## 🤖 LLM Output

For each run, GPT-4.1-mini generates **3 long-form video concepts** and **3 short-form reel concepts**, each including:

- Proposed **title**
- Recommended **tags**
- Search-aligned **keywords**
- Optimal **publication time**
- A **content type decision** (Videos vs Reels) backed by Welch's t-test results

The LLM operates **only on pre-validated, curated context** — never on raw data. Every suggestion is traceable back to a specific scored video or ranked query.

---

##  Limitations

- **YouTube API quota**: ~10 API keys needed for meaningful coverage. Production use requires a quota extension via the YouTube API Compliance Audit.
- **Google Trends**: Manual CSV export required (no official API). Enterprise DaaS (e.g. DataForSEO) can automate this.
- **Static scoring**: Fixed weights across executions. Future work includes adaptive weighting.
- **No feedback loop**: Post-publication performance is not re-ingested. Future work targets semi-adaptive scoring.

---

##  References

- [YouTube Data API v3](https://developers.google.com/youtube/v3)
- [Google Trends Terms](https://policies.google.com/terms)
- [OpenAI API](https://openai.com/api/)

---
##  License

This project was developed as an academic Data Management project.  
All example results and images are from a pipeline run executed on **February 6th, 2026**, using *"data science"* as the topic.


