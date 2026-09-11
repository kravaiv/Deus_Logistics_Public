# Project Datasheet: Deus Logistics
**Version**: 2.1.0-clean  
**Architecture Style**: Modular Monolith Architecture (Clean Layers)  
**Target Environment**: Windows Server / Windows 10/11 (Python 3.12)  

Deus Logistics is an **enterprise-grade logistics intelligence, foreign trade analytics, and automated lead generation platform** combining a Telegram operations interface with local deterministic data processing and secure cloud LLM integration.

---

## 1. Codebase Statistics & Metrics

- **Primary Language**: **Python 3.12** (100% of business logic and automation).
- **Presentation Engine**: `aiogram 3.x` (Async FSM & Router Architecture).
- **Storage**: **SQLite 3** (local high-performance analytical databases with compound indexing and WAL mode).
- **Browser Automation**: `Playwright Chromium` (headless/headed async worker).
- **AI Integration**: Google Gemini API via Deterministic Data Masking proxy & sqlglot AST guardrails.
- **Test Suite**: 37 comprehensive unit & integration tests (`pytest`), 100% pass rate.
- **Architectural Modularity**: Strict unidirectional dependency between UI handlers, domain calculators, ETL, and infrastructure.

---

## 2. Component Architecture & Modular Layout

```text
deus_logistics/
├── bot/                         # Presentation Layer (aiogram 3.x)
│   ├── handlers/                # Business Domain Routers
│   │   ├── operational.py       # Company KPIs, dispatcher metrics, calendar picker
│   │   ├── customs.py           # EDRPOU customs verification & dossier delivery
│   │   ├── management.py        # Executive HTML dashboard generation
│   │   ├── ai_analytics.py      # Natural language Text-to-SQL + HITL feedback loop
│   │   └── outreach.py          # LinkedIn worker triggers & status checks
│   ├── middlewares/             # Security & Access Control
│   │   └── auth.py              # Role-based Telegram ID whitelist filter
│   ├── keyboards/               # Dynamic inline & reply keyboards
│   ├── states/                  # FSM finite state definitions
│   └── main.py                  # Application factory, dispatcher & lifecycle manager
│
├── services/                    # Business Domain & Application Services Layer
│   ├── reports/                 # Financial & Trade Calculation Engines
│   │   ├── management_report.py # KPI aggregations, multi-currency conversion, HTML generator
│   │   └── customs_report.py    # 32.8 GB customs declarations analytical engine
│   ├── ai/                      # AI Governance & Obfuscation
│   │   └── gemini_analyzer/     # Text-to-SQL, AST validator, entity & salt obfuscator
│   ├── data_processing/         # Pure Normalization Functions
│   │   └── cleaning.py          # Homoglyph transliteration, currency parser, date normalizer
│   ├── outreach/                # Autonomous Lead Generation Engine
│   │   └── linkedin_bridge.py   # Async Playwright process supervisor & scraper
│   └── path_manager.py          # Multi-tier database & cache path resolver
│
├── data/                        # Persistent Storage & Fixtures
│   ├── database/                # SQLite databases (logistics.db, sample trade DB)
│   └── cache/                   # Exchange rate cache (NBU API) & feedback logs
│
├── tests/                       # Automated Test Suite (pytest)
│   ├── test_cleaning.py         # Normalization & parsing unit tests
│   ├── test_financial_math.py   # Margin, yield, ROI calculation verification
│   ├── test_obfuscator.py       # Obfuscation & salt scaling roundtrip tests
│   ├── test_customs_report.py   # Customs aggregation & HTML generation tests
│   └── test_sql_guardrails.py   # AST validation & SQL security guardrails tests
│
├── config/                      # Configuration Management
│   ├── config.example.json      # Sanitized environment configuration template
│   └── .env.example             # Token & API credentials template
│
├── build_logistic_center.py     # Safe Atomic ETL Pipeline (Google Sheets / Excel -> SQLite)
└── logistic_bot.py              # Lightweight Entrypoint Facade (30 lines)
```

---

## 3. Databases and Data Volumes

| Database Identifier | File Reference | Storage Engine | Scope & Data Volume | Indexing Strategy |
| :--- | :--- | :--- | :--- | :--- |
| **Operational Logistics DB** | `logistics.db` | SQLite 3 | Active operational shipments (2024–2026), carriers, drivers, gross rates, client debts. | Compound indices on `(date_deal, logistic_clean)`, `(client_clean, date_deal)`. |
| **Customs Declarations DB** | `trade_data.db` | SQLite 3 | **32.8 GB** dataset encompassing all export/import declarations of Ukraine for 2025–2026 (~millions of records). | B-Tree index on `(edrpou, direction, date)`. Sub-second query latency. |
| **Customs Demo Sample DB** | `trade_data_sample.db` | SQLite 3 | 94 KB portable fixture with normalized sample declarations for staging and development. | Matches production schema identically. |
| **Currency Cache DB** | `nbu_rates_cache.json` | JSON Key-Value | Official National Bank exchange rates indexed by ISO date string (`YYYYMMDD`). | In-memory lookup with persistent disk cache. |

---

## 4. Key Architectural Trade-Offs & Decisions

1. **Local SQLite with WAL Mode vs. Remote PostgreSQL/MySQL**:
   - *Decision*: Adopted local SQLite instances with Write-Ahead Logging (`PRAGMA journal_mode = WAL;`) and engine-level read-only connections (`mode=ro`, `PRAGMA query_only = ON;`).
   - *Rationale*: Sub-second analytical query performance on multi-gigabyte local storage (32.8 GB) without network roundtrip latency or external database infrastructure maintenance. Read-heavy analytical workload with atomic, isolated ETL rebuilds.
2. **Deterministic Data Masking vs. Local 70B LLM (Ollama/vLLM)**:
   - *Decision*: Paired cloud Google Gemini API with deterministic session-scoped data masking (salt scaling on tabular aggregates and token mapping on entities) rather than hosting heavy 70B parameter models on local GPUs.
   - *Rationale*: Eliminates requirement for expensive GPU hardware while guaranteeing that sensitive financial margins and counterparty identities never leave the local environment.
3. **Decoupled Playwright Worker vs. In-Process Scraping**:
   - *Decision*: Isolated LinkedIn browser automation into an asynchronous background subprocess decoupled from the main event loop.
   - *Rationale*: Prevents Chromium memory consumption from blocking or impacting the Telegram bot's async event loop, enabling dedicated process lifecycle control.
