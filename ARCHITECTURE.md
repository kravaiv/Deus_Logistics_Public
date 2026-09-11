# Deus Logistics — System Architecture Specification

This document provides a comprehensive technical breakdown of the architecture, design patterns, data flows, and reliability mechanisms implemented in the Deus Logistics platform.

---

## 1. Architectural Style & Layering

The system is structured following **Clean Architecture / Layered Architecture** principles, enforcing a strict unidirectional dependency rule:

```text
┌─────────────────────────────────────────────────────────────┐
│                    Presentation Layer                       │
│  Telegram Bot (bot/handlers, bot/keyboards, bot/main.py)    │
└──────────────────────────────┬──────────────────────────────┘
                               │ depends on
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                      Service Layer                          │
│  services/reports, services/ai, services/outreach, ETL      │
└──────────────────────────────┬──────────────────────────────┘
                               │ depends on
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                 Data Processing & Normalization             │
│  services/data_processing/cleaning.py, path_manager.py      │
└──────────────────────────────┬──────────────────────────────┘
                               │ depends on
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                   Infrastructure & Storage                  │
│  SQLite (logistics.db, trade_data.db), NBU Cache, Filesystem│
└─────────────────────────────────────────────────────────────┘
```

### Layer Responsibilities:
1. **Presentation Layer (`bot/`)**:
   - Manages user sessions and Telegram Bot API events via `aiogram 3.x`.
   - Contains zero raw SQL queries or business calculation logic; delegates all operations to the service layer.
   - Enforces role-based access control via `AuthMiddleware`.
2. **Service Layer (`services/`)**:
   - Contains high-level business capabilities: financial yield calculations (`management_report.py`), customs dossier aggregations (`customs_report.py`), AI interaction pipelines, and browser-based lead generation.
3. **Data Processing Layer (`services/data_processing/`)**:
   - Pure domain helper utilities for string sanitation, homoglyph replacement, rate extraction, and multi-format date parsing. Independent of external frameworks.
4. **Infrastructure Layer (`data/`, `config/`)**:
   - SQLite databases, caches, and filesystem path resolution via `path_manager.py`.

---

## 2. Sequence Diagrams

### 2.1. Zero-Data-Leak AI Natural Language BI Query
Demonstrates how user natural language questions are converted into answers without exposing commercial secrets to the cloud LLM:

```mermaid
sequenceDiagram
    autonumber
    actor User as Telegram User
    participant Bot as bot/handlers/ai_analytics.py
    participant Obf as services/ai/gemini_analyzer/obfuscator.py
    participant SQLGen as services/ai/gemini_analyzer/text_to_sql.py
    participant DB as SQLite (logistics.db)
    participant Gemini as Google Gemini API

    User->>Bot: "What was Petrov's margin in March 2026 in EUR?"
    Bot->>SQLGen: Generate SQL for user question (Schema only)
    SQLGen->>Gemini: Prompt with DB Schema & question
    Gemini-->>SQLGen: SELECT COUNT(*), SUM(rate_client)... WHERE logistic_clean='Petrov'
    SQLGen->>SQLGen: AST validation via sqlglot (Verify exp.Select / exp.Union)
    SQLGen->>DB: Execute via mode=ro & PRAGMA query_only = ON
    DB-->>SQLGen: Raw Database Rows [(15, 45000.0, 32000.0, 'EUR')]
    SQLGen->>Obf: Obfuscate rows (Mask Names + Salt Rates on Aggregates)
    Note over Obf: "Petrov" -> "Dispatcher_1"<br/>Rate * 1.42x Salt
    Obf-->>Bot: Obfuscated Data [(15, 63900.0, 45440.0, 'EUR')]
    Bot->>Gemini: Request Structured JSON (synthesis + scaled_metrics)
    Gemini-->>Bot: JSON {"synthesis": "Dispatcher_1 delivered 15 trips...", "scaled_metrics": {...}}
    Bot->>Obf: Hybrid Deobfuscation
    Note over Obf: synthesis.replace("Dispatcher_1", "Petrov")<br/>scaled_metrics / 1.42x Salt
    Obf-->>Bot: Restored human-readable answer & exact metrics
    Bot->>User: Formatted analytical report with feedback buttons (Accurate / Correction)
```

---

### 2.2. Safe Atomic ETL Database Rebuilding
Guarantees uninterrupted operation of the live bot during data syncs:

```mermaid
sequenceDiagram
    autonumber
    participant Cron as Scheduler / Manual Invocation
    participant ETL as build_logistic_center.py
    participant Cloud as Google Sheets API / Excel Backup
    participant TempDB as Temporary Database (logistics.db.tmp)
    participant Backup as Backup Storage (logistics.db.bak)
    participant LiveDB as Live Database (logistics.db)

    Cron->>ETL: Trigger sync process
    ETL->>Cloud: Fetch latest active & archived shipment sheets
    Cloud-->>ETL: Raw data records
    ETL->>TempDB: Initialize schema & stream normalized records
    Note over ETL,TempDB: Clean homoglyphs, parse dates,<br/>classify transport directions
    ETL->>TempDB: Verify integrity (SELECT COUNT(*) > 0)
    alt Data valid and non-empty
        ETL->>Backup: Copy current LiveDB -> logistics.db.bak
        ETL->>LiveDB: Atomic replacement (os.replace TempDB -> LiveDB)
        Note over LiveDB: Live database updated atomically with zero downtime
    else Data validation failed
        ETL->>TempDB: Delete temporary file
        Note over LiveDB: Live database remains untouched and fully operational
    end
```

---

## 3. Finite State Machine (FSM) Design

The Telegram bot uses `aiogram.fsm` to isolate user dialogue contexts:

```mermaid
stateDiagram-v2
    [*] --> MainMenu: /start

    MainMenu --> GeneralMode: "General Analytics"
    GeneralMode --> GeneralYearSelected: Select Year (2024-2026)
    GeneralYearSelected --> GeneralPeriodMode: Select Quarter / Range / Calendar
    GeneralPeriodMode --> GeneralYearSelected: "Back to Year Selection"

    MainMenu --> LogistsMenu: "Logistics Managers"
    LogistsMenu --> LogistSelected: Pick Manager
    LogistSelected --> LogistPeriodMode: "Select Period"
    LogistPeriodMode --> LogistSelected: "Back to Manager"

    MainMenu --> CarriersMenu: "Carriers"
    CarriersMenu --> CarrierSelected: Pick Fleet / Subcontractor
    CarrierSelected --> CarrierPeriodMode: "Select Period"

    MainMenu --> CustomsCheck: "Verify Counterparty"
    CustomsCheck --> CustomsResult: Valid 8-digit EDRPOU
    CustomsResult --> CustomsCheck: "New Query"

    MainMenu --> AIChatMode: "Ask AI"
    AIChatMode --> AIFeedbackWait: Negative Feedback (Correction)
    AIFeedbackWait --> AIChatMode: User Correction Stored

    MainMenu --> ManagementPeriod: "Executive Dashboard"
    ManagementPeriod --> ManagementYear: Pick Scope (Year/Quarter/Month)
    ManagementYear --> [*]: HTML Dashboard Download

    GeneralMode --> MainMenu: "Return to Main Menu"
    LogistsMenu --> MainMenu: "Return to Main Menu"
    CarriersMenu --> MainMenu: "Return to Main Menu"
    CustomsCheck --> MainMenu: "Return to Main Menu"
    AIChatMode --> MainMenu: "Return to Main Menu"
```

---

## 4. Multi-Tier Path & Database Fallback Architecture

To ensure portable local development without transferring multi-gigabyte files across networks, the system implements a multi-tier database resolver (`services/path_manager.py`):

1. **Customs Database Resolution**:
   - *Tier 1 (Production)*: Searches `ТБ_26/trade_data_2026.db` (32.8 GB SQLite declarations database).
   - *Tier 2 (Development/Demo)*: Falls back transparently to `data/database/trade_data_sample.db` (~94 KB synthetic fixture matching schema).
2. **Configuration Resolution**:
   - *Tier 1*: `config/config.json`.
   - *Tier 2*: Root fallback `config.json`.
3. **Rates & Currency Cache Resolution**:
   - *Tier 1*: `data/cache/nbu_rates_cache.json`.
   - *Tier 2*: Root fallback `nbu_rates_cache.json`.

---

## 5. Runtime Execution Topology

```text
Host Environment (Windows / Windows Server)
 └── Python 3.12 Virtual Environment (`venv`)
      └── Service: Telegram Bot (`logistic_bot.py`)
           ├── Core Engine: aiogram 3 Event Loop
           ├── Subprocess Worker: Playwright Chromium (LinkedIn Outreach)
           ├── Local Storage (Direct I/O):
           │    ├── data/database/logistics.db (SQLite WAL)
           │    ├── trade_data_2026.db (32.8 GB Customs Dataset)
           │    └── data/cache/ (NBU Currency Cache)
           ├── Configuration: config/config.json & .env
           └── External Cloud Integrations:
                ├── Google Sheets API (ETL Ingestion)
                ├── Google Gemini API (Text-to-SQL Analytics)
                └── Telegram Bot API
```
