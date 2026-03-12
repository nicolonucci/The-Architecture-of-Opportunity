# 🎯 The Architecture of Opportunity
### A Deterministic Data Pipeline for Market-Aware Content Ideation

> *"Can subjective content ideation be replaced by deterministic data pipelines?"*

**Authors:** Francesco Colombini · Nicolò Nucci  
**Course:** Data Management Project

---

## 📖 Abstract

The creator economy faces a fundamental information asymmetry: platforms provide abundant retrospective analytics but negligible predictive support. This project engineers a three-tier local architecture that synthesizes search intent signals (**Demand**) from Google Trends with empirically validated performance metrics (**Supply**) from YouTube. Through rigorous ELT enforcement and statistical quality constraints, creative ideation is treated as a data engineering challenge — where generative outputs are constrained by statistical evidence rather than plausible speculation.

---

## 🏗️ System Architecture
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

## 🚀 Quick Start

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
OpenAI = ["sk-proj-YOUR_OPENAI_KEY"]
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

## 📁 Project Structure
```
📦 project-root/
├── 📓 User_interface.ipynb          ← START HERE
├── 📓 Acquisition_and_storage.ipynb
├── 📓 Data_quality_and_analysis.ipynb
├── 📓 LLM_process.ipynb
├── 📓 Key_creator.ipynb
├── 📓 Open_ai_key.ipynb
├── 🔑 youtube_keys.txt
├── 🔑 Open_AI_key.txt
├── 📄 searched_with_rising-queries_*.csv  (×4 windows)
├── 📄 searched_with_top-queries_*.csv     (×2 windows)
│
└── 📂 {topic}_Outputs/
    ├── 📂 storage/
    │   └── {topic}_engine.db              ← SQLite database
    ├── 📂 exports/
    │   ├── {topic}_discovery.ndjson
    │   ├── {topic}_stats.ndjson
    │   ├── {topic}_google_trends_rising.csv
    │   ├── {topic}_google_trends_top.csv
    │   └── {topic}_llm_result.txt         ← Final output
    └── 📂 analysis/
        ├── 📂 complete/
        │   ├── Master_COMPLETE_Dashboard.png
        │   └── {topic}_complete_eda_report.txt
        └── 📂 interesting/
            ├── Master_INTERESTING_Dashboard.png
            └── {topic}_interesting_eda_report.txt
```

---

## 🧮 Core Metrics & Scoring

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

---

## 🤖 LLM Output

For each run, GPT-4.1-mini generates **3 long-form video concepts** and **3 short-form reel concepts**, each including:
- Proposed **title**
- Recommended **tags**
- Search-aligned **keywords**
- Optimal **publication time**
- A **content type decision** (Videos vs Reels) backed by Welch's t-test results

The LLM operates **only on pre-validated, curated context** — never on raw data.

---

## ⚙️ Google Trends Temporal Fingerprinting

Each rising query receives a **4-digit binary fingerprint** encoding breakout presence across time windows:
```
Pattern "1011" → breakout in 60d, 14d, 7d windows (absent in 30d)
```

Window weights for recency scoring:
| Window | Weight |
|--------|--------|
| 7 days | 8× |
| 14 days | 4× |
| 30 days | 2× |
| 60 days | 1× |

---

## ⚠️ Limitations

- **YouTube API quota**: ~10 API keys needed for meaningful coverage. Production use requires a quota extension via the YouTube API Compliance Audit.
- **Google Trends**: Manual CSV export required (no official API). Enterprise DaaS (e.g. DataForSEO) can automate this.
- **Static scoring**: Fixed weights across executions. Future work includes adaptive weighting.
- **No feedback loop**: Post-publication performance is not re-ingested. Future work targets semi-adaptive scoring.

---

## 📚 References

- [YouTube Data API v3](https://developers.google.com/youtube/v3)
- [Google Trends Terms](https://policies.google.com/terms)
- [OpenAI API](https://openai.com/api/)

---

## 📜 License

This project was developed as an academic Data Management project. All results and images are from a run executed on February 6th, 2026, using "data science" as the topic.
```
