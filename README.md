# Julia Tamagno — AI/ML Engineer

![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)
![PyTorch](https://img.shields.io/badge/PyTorch-EE4C2C?style=flat&logo=pytorch&logoColor=white)
![scikit-learn](https://img.shields.io/badge/scikit--learn-F7931E?style=flat&logo=scikitlearn&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=flat&logo=fastapi&logoColor=white)
![Apache Kafka](https://img.shields.io/badge/Kafka-231F20?style=flat&logo=apachekafka&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=flat&logo=redis&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![Next.js](https://img.shields.io/badge/Next.js-000000?style=flat&logo=nextdotjs&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=flat&logo=postgresql&logoColor=white)
![DuckDB](https://img.shields.io/badge/DuckDB-FFF000?style=flat&logo=duckdb&logoColor=black)
![AWS](https://img.shields.io/badge/AWS-232F3E?style=flat&logo=amazonaws&logoColor=white)
![MongoDB](https://img.shields.io/badge/MongoDB-47A248?style=flat&logo=mongodb&logoColor=white)
![Elasticsearch](https://img.shields.io/badge/Elasticsearch-005571?style=flat&logo=elasticsearch&logoColor=white)

I learn by building. Each project here is an attempt to understand a layer of the ML stack from the inside: how retrieval and ranking actually work, what a streaming feature store needs to handle drift, how causal uplift changes a bidding strategy, what makes a deployment pipeline safe to automate.

These aren't production systems — they're built to the spec of one. The goal is to understand the design decisions well enough to make them myself.

---

## The ecosystem

```
┌─────────────────────────────────────────────────────────────────┐
│                         ml-platform                             │
│      model registry · shadow/canary deploy · PSI skew          │
│               automated rollback · lifecycle API                │
└──────────────────────┬──────────────────────────────────────────┘
                       │ manages models trained in ↓
┌──────────────────────▼──────────────────────────────────────────┐
│                       recsys-adtech                             │
│  two-tower retrieval → LambdaRank → DeepFM CTR/CVR reranking    │
│  Vickrey auction · uplift bids · PID budget pacing              │
│  cold-start (LinUCB) · fatigue decay · multi-objective          │
└────┬──────────┬──────────┬──────────┬───────────────────────────┘
     │          │          │          │
     ▼          ▼          ▼          ▼
 rankflow  uplift-engine  driftguard  eventsourced-ml
 semantic  causal uplift  streaming   event store +
 ranking   T/X-Learner    features    data contracts
 PT + EN   Qini / RTB     + drift     point-in-time
     │          │          │          │
     └──────────┴──────────┴──────────┘
                       │
              ┌────────▼────────┐
              │   personaflow   │
              │  API gateway    │
              │  50ms p99 SLA   │
              └─────────────────┘

┌──────────────────────┐     ┌──────────────────────────────┐
│    governa-flow      │     │      data-lineage-cli        │
│  data governance     │─────│  lineage graph · trust       │
│  semantic drift      │     │  propagation · CLI viewer    │
│  LLM metadata KG     │     └──────────────────────────────┘
└──────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                          trustline                              │
│  credit origination audit · BCB 538/2025 · LGPD compliance     │
│  LLM fraud detection · correspondent risk scoring              │
│  Bedrock · MongoDB · Kafka · ElasticSearch · Airflow DAGs      │
└─────────────────────────────────────────────────────────────────┘

┌──────────────────────┐     ┌──────────────────────────────┐
│     askmydata        │     │       watch-with-me          │
│  NL → SQL over any   │     │  RAG over video/audio with   │
│  CSV/Parquet file    │     │  precise timestamp answers   │
└──────────────────────┘     └──────────────────────────────┘
```

---

## Projects

### Recommendation + AdTech core

**[recsys-adtech](https://github.com/jutamagno/recsys-adtech)** — Full recommendation and advertising stack, end-to-end.

Eight independent modules that chain together into a production pipeline:

| Module | What it implements |
|---|---|
| Two-tower retrieval | FAISS IVFFlat, dot-product ANN, hard negative mining |
| Ranking | LambdaRank with position-bias correction (IPS), pairwise training |
| CTR/CVR | DeepFM with field-aware factorization, Platt calibration |
| Auction | Vickrey second-price, GSP multi-slot, revenue analysis |
| Budget pacing | PID controller, spend rate tracking, bid multiplier |
| Cold-start | LinUCB contextual bandit, Sherman-Morrison rank-1 updates |
| Fatigue | Exponential decay, frequency caps, diversity injection |
| Serving API | FastAPI, multi-stage pipeline, latency breakdown per stage |

Each module is independent and runnable — building them in sequence is how I worked through the full retrieval → ranking → reranking → serving stack.

---

### Component services

**[rankflow](https://github.com/jutamagno/rankflow)** — Semantic ranking with Portuguese/English support.

Teacher-student distillation: Claude Haiku generates LLM quality labels offline, which train a lightweight `paraphrase-multilingual-mpnet-base-v2` + LTR reranker that runs at inference speed. Handles structural differences between PT and EN user intent — not just translation.

**[uplift-engine](https://github.com/jutamagno/uplift-engine)** — Causal uplift modeling for ad bid optimization.

Implements T-Learner and X-Learner meta-learners, four-quadrant user segmentation (Persuadable / Sure Thing / Lost Cause / Sleeping Dog), and Qini coefficient evaluation. Simulated RTB environment shows X-Learner achieving 51.2% better ROI than a CTR-only bidding policy.

**[driftguard](https://github.com/jutamagno/driftguard)** — Streaming feature store with online learning and drift detection.

Kafka consumer → River `LogisticRegression` online model → Redis feature cache (TTL 7200s). ADWIN drift detector (Bifet & Gavalda 2007) publishes alerts to a `drift:alerts` pub/sub channel. Page-Hinkley for gradual drift. Designed to feed personalization features to any downstream ranker.

**[eventsourced-ml](https://github.com/jutamagno/eventsourced-ml)** — Reproducible event processing with data contracts.

Append-only Parquet event store (partitioned by date), point-in-time correct feature joins via bisect for training data that doesn't leak the future, Avro schema registry with backward-compatibility enforcement, and data contract validation at ingestion. No Kafka required to run locally.

---

### Integration layer

**[personaflow](https://github.com/jutamagno/personaflow)** — API gateway that assembles all four component services.

`POST /rank` fans out to rankflow and driftguard in parallel (`asyncio.gather`) to stay within a 50ms p99 budget. `POST /bid` proxies to uplift-engine with graceful degradation (feature service timeout returns `{}` rather than failing the request). The scenario is a PT/EN e-commerce platform where user intent in Portuguese isn't just "English translated" — it's structurally different.

---

### MLOps

**[ml-platform](https://github.com/jutamagno/ml-platform)** — Model lifecycle management with automated deployment and rollback.

Shadow → canary → full deployment pipeline. PSI-based training data skew detection. Automated rollback triggered independently on three conditions: AUC drop, error rate spike, or p99 latency breach. Model registry with versioning. Integrates with recsys-adtech via `run_recsys_cycle.py` — generates auction events, trains LightGBM + DeepFM, and routes the resulting artifacts through the deployment pipeline.

---

### Data governance

**[trustline](https://github.com/jutamagno/trustline)** — Intelligent audit platform for credit origination data in banking.

Built around a real problem: in 2025, banking correspondents faked customer consent to issue fraudulent consignado loans. Resolução BCB 538/2025 mandates compliance by December 2026. Trustline audits every origination event before credit is issued — three LLM-powered analyzers (inconsistency detection, consent chain verification, correspondent risk scoring), four Airflow DAGs (daily eval, daily risk scoring, daily BCB 538 report, weekly LGPD audit), and a continuous eval framework that tracks false negative rate, PII leakage, and Bedrock cost across a golden dataset of fraud scenarios. Stack: FastAPI · AWS Bedrock (Claude Haiku) · MongoDB · Kafka · ElasticSearch · PostgreSQL · Airflow · LocalStack · Helm (EKS).

**[governa-flow](https://github.com/jutamagno/governa-flow)** — Data governance platform with LLM-based metadata reasoning.

Semantic drift detection across dataset columns, trust propagation through a lineage graph (PII signals and quality scores flow downstream), and a KG-Agent pattern where an Ollama-backed LLM reasons over the governance knowledge graph. Integrates with OpenLineage and Airflow for pipeline-level lineage.

**[data-lineage-cli](https://github.com/jutamagno/data-lineage-cli)** — Lightweight CLI for exploring data lineage graphs.

Reads lineage metadata and visualizes dependency graphs in the terminal. Companion tool to governa-flow for local inspection without a running platform.

---

### Applied LLM / RAG

**[askmydata](https://github.com/jutamagno/askmydata)** — Natural language interface for tabular data.

Upload any CSV or Parquet file, ask questions in plain English, get SQL + results back. Stack: Next.js 14 frontend, FastAPI backend, DuckDB for in-process query execution, Ollama (local LLM) for NL→SQL generation. Streaming answers via SSE. No cloud API keys required.

**[watch-with-me](https://github.com/jutamagno/watch-with-me)** — RAG over video and audio with timestamp-aware answers.

Upload a video or audio file, ask questions, get answers with precise `[HH:MM:SS → HH:MM:SS]` references. faster-whisper transcribes into 30-second chunks, nomic-embed-text embeds them into ChromaDB, and a local Ollama LLM generates the answer from top-k retrieved chunks. Everything runs locally.

---

## How to navigate

**If you want to see RecSys fundamentals** → start with [`recsys-adtech`](https://github.com/jutamagno/recsys-adtech). Each of the 8 modules (`01-retrieval` through `08-serving`) is self-contained and runnable independently.

**If you want everything running together** → clone the five sibling repos and run `make up` in [`personaflow`](https://github.com/jutamagno/personaflow). The gateway, ranking service, feature service, and uplift service all start together.

**If you want the MLOps layer** → [`ml-platform`](https://github.com/jutamagno/ml-platform) stands alone and also has a `run_recsys_cycle.py` that exercises the full recsys-adtech pipeline end-to-end.

**If you want the LLM/RAG projects** → [`watch-with-me`](https://github.com/jutamagno/watch-with-me) and [`askmydata`](https://github.com/jutamagno/askmydata) each run with `docker compose up --build`, no GPU required.

**If you want the banking data governance project** → [`trustline`](https://github.com/jutamagno/trustline): `make up && make seed && make demo`. Runs the full stack locally via LocalStack (no real AWS needed).

---

## Stack

| Area | Technologies |
|---|---|
| ML | PyTorch, scikit-learn, LightGBM, River (online learning), FAISS |
| LLMs | Ollama (local inference), Claude API (offline label generation), AWS Bedrock (Claude Haiku) |
| Data | Kafka, Redis, MongoDB, DuckDB, Parquet, Avro, ChromaDB, ElasticSearch |
| Serving | FastAPI, Next.js 14, Docker Compose, SSE streaming |
| MLOps | GitHub Actions, pytest, ruff, codecov, custom model registry |
| Transcription | faster-whisper (CTranslate2 backend), yt-dlp |

---

## Contact

[LinkedIn](https://linkedin.com/in/jutamagno)
