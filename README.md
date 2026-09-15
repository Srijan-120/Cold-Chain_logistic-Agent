# 🧊 Autonomous Cold-Chain Logistics Agent (Enterprise FDE Platform)

An end-to-end, enterprise-grade AI decision-support platform for cold-chain fleet monitoring, breach mitigation, and dynamic transit rerouting. Built with **LangGraph**, **Microsoft SQL Server**, **Pinecone**, **Voyage AI**, and **Streamlit**, deployed on a secure **2-Tier AWS EC2 Cloud Architecture**.

---

## 📌 Executive Summary & Problem Statement

In cold-chain logistics, maintaining strict temperature ranges (e.g., $0^\circ\text{C}$ to $4^\circ\text{C}$ for fresh perishables) is critical. A delay or cooling unit failure directly leads to catastrophic cargo spoilage, regulatory violations, and tens of thousands of dollars in lost inventory.

However, operational data in enterprise logistics is heavily fragmented:
1. **Obscure Legacy Telemetry**: Stored in 2000s-era database schemas with cryptically abbreviated column names (e.g., `IOT_TEMP_VAL_C`, `PRT_CNG_LVL`, `CGO_COND_CD`).
2. **Disconnected Compliance SOPs**: Buried in unstructured PDF, Markdown, and text policy manuals.
3. **External Transit Hazards**: Live corridor weather, winds, and port congestion residing in third-party REST APIs.

This platform bridges all three silos using an **autonomous agentic state machine** that correlates real-time telemetry with live corridor conditions, retrieves compliance policies, and recommends auditable operational resolutions.

---

## 🏛️ System Architecture

The application is engineered as a decoupled, **2-Tier Production Cloud Deployment** hosted on AWS:

```
                                  [ INTERNET USERS / DISPATCHERS ]
                                                 │
                                                 │ HTTPS / Port 8501
                                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│  AWS EC2 INSTANCE #1: APPLICATION SERVER (Streamlit + LangGraph)                                │
│                                                                                                 │
│   ┌─────────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                STREAMLIT DISPATCH CONSOLE                               │   │
│   │            - Live Telemetry Querying       - Real-time Trace Expanders                  │   │
│   │            - Operational Resolutions       - Admin-Gated Audit Trail                    │   │
│   └────────────────────────────────────────────┬────────────────────────────────────────────┘   │
│                                                ▼                                                │
│   ┌─────────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                         LANGGRAPH AGENT ORCHESTRATION ENGINE                            │   │
│   │                         Reasoner: Groq (LPU) / DeepSeek V3                              │   │
│   │                                                                                         │   │
│   │   [ Tool 1: Telemetry DB ]     [ Tool 2: Corridor REST API ]   [ Tool 3: Vector RAG ]   │   │
│   └─────────────────┬────────────────────────────┬────────────────────────────┬─────────────┘   │
└─────────────────────┼────────────────────────────┼────────────────────────────┼─────────────────┘
                      │                            │                            │
                      │ AWS Private VPC            │ HTTPS (REST)               │ HTTPS (Vector)
                      │ Port 1433                  │                            │
                      ▼                            ▼                            ▼
┌───────────────────────────────────────┐  ┌──────────────┐             ┌─────────────────────────┐
│ AWS EC2 INSTANCE #2: DATABASE SERVER  │  │  OPEN-METEO  │             │   PINECONE VECTOR DB    │
│                                       │  │   REST API   │             │   (Serverless 1024-D)   │
│  ┌─────────────────────────────────┐  │  │              │             │                         │
│  │ Containerized MSSQL (Docker)    │  │  │ Live Weather │             │ Embedded via Voyage AI  │
│  │ Persistent EBS: /var/opt/mssql  │  │  │  Wind & Port │             │ Hierarchical SOP Chunks │
│  ├─────────────────────────────────┤  │  │  Congestion  │             └─────────────────────────┘
│  │ Schema: FDE_VIEWS               │  │  └──────────────┘
│  │  ├─ View: VW_ACTIVE_FLEET       │  │
│  │  └─ Table: AgentAuditLog        │  │
│  ├─────────────────────────────────┤  │
│  │ Least-Privilege Role:           │  │
│  │  └─ USR_FDE_RO (Read-Only)      │  │
│  └─────────────────────────────────┘  │
└───────────────────────────────────────┘
```

---

## 🔒 Enterprise Data Governance & Security Layer

Allowing an LLM to query production databases carries inherent risks of hallucinations, SQL injection, or accidental schema mutations. This system enforces **Defense-in-Depth**:

1. **Semantic Abstraction Layer (`FDE_VIEWS.VW_ACTIVE_FLEET`)**:
   - Rather than exposing the cryptic raw table (`dbo.TBL_SC_FLEET_HIST_RAW`), a semantic view translates enterprise codes into clean, contextual business fields (`Current_Temperature_C`, `Delay_Probability`, `Port_Congestion_Level`, `Route_Risk_Index`).
2. **Database-Enforced RBAC (`USR_FDE_RO`)**:
   - The AI Agent connects exclusively using a dedicated, least-privilege user account.
   - `GRANT SELECT ON FDE_VIEWS.VW_ACTIVE_FLEET TO USR_FDE_RO`
   - `DENY SELECT ON dbo.TBL_SC_FLEET_HIST_RAW TO USR_FDE_RO`
   - `DENY INSERT, UPDATE, DELETE, ALTER ON SCHEMA::dbo TO USR_FDE_RO`
   - Even if an LLM is tricked into generating `DROP TABLE` or `DELETE`, the Microsoft SQL Server engine strictly terminates the query at the kernel level.
3. **Application Guardrails**:
   - Pre-flight regex and T-SQL validation ensures only deterministic `SELECT` statements are executed.
   - Result truncation (`fetchmany(10)`) guards against context window overflow.

---

## 🛠️ Technology Stack

| Layer | Technology | Key Details |
| :--- | :--- | :--- |
| **Agent Orchestration** | LangGraph / LangChain | Cyclic state graph with `ToolNode` and conditional edges |
| **LLM Reasoning Engine**| Groq (LPU) / DeepSeek | High-speed inference with native function/tool-calling |
| **Relational Database** | Microsoft SQL Server 2022 | Dockerized on Ubuntu EC2 with persistent host EBS volume |
| **Database Driver** | PyODBC / SQLAlchemy 2.0 | Native Microsoft ODBC Driver 18 with connection pooling |
| **Vector Database** | Pinecone Serverless | Cosine distance metric with metadata filtering |
| **Embeddings Model** | Voyage AI (`voyage-3`) | 1024-dimensional semantic dense vectors |
| **Document Ingestion** | LangChain Text Splitters | Polymorphic parser for `.md`, `.pdf`, `.csv`, `.xlsx`, `.txt` |
| **User Interface** | Streamlit | Dark-mode dispatch dashboard with real-time trace inspection |
| **Cloud Infrastructure**| AWS EC2 (`c7i-flex.large`) | 2-tier architecture isolated with AWS VPC Security Groups |

---

## 📁 Repository Structure

```
Cold-Chain_logistic-Agent/
├── data/
│   ├── policy/                     # Regulatory SOPs & incident manuals (Markdown/PDF)
│   └── raw/                        # Historical logistics telemetry datasets (CSV)
├── scripts/
│   ├── ingest_legacy_data.py       # Loads raw CSV telemetry into Dockerized SQL Server
│   ├── ingest_sop_document.py      # Polymorphic parser, Voyage embedder & Pinecone indexer
│   └── setup_secuirty_and_view.sql # DDL script for schemas, views, RBAC & audit tables
├── src/
│   ├── agent_tools.py              # SQL execution, Open-Meteo REST API & Pinecone retriever
│   ├── orchestrator.py             # LangGraph state machine, checkpointing & LLM factory
│   ├── ui.py                       # Streamlit multi-tab dispatch and audit console
│   └── prompts/
│       └── system_prompt.txt       # Executive persona, reasoning rules & output format
├── requirements.txt                # Production Python dependencies
└── README.md                       # System documentation
```

---

## ⚙️ Setup & Local Quickstart

### 1. Clone & Environment Setup
```bash
git clone https://github.com/Srijan-120/Cold-Chain_logistic-Agent.git
cd Cold-Chain_logistic-Agent

# Create and activate virtual environment
python -m venv coldenv
# Windows:
coldenv\Scripts\activate
# Linux/macOS:
source coldenv/bin/activate

pip install -r requirements.txt
```

### 2. Configure Environment Variables (`.env`)
Create a `.env` file in the root directory:

```ini
# Vector Database & Embeddings
PINECONE_API_KEY=pcsk_...
VOYAGE_API_KEY=pa-...
Embedding_model=VOYAGE

# LLM Reasoner (Choose GROQ or DEEPSEEK)
Agent_llm=GROQ
GROQ_API_KEY=gsk_...
DEEPSEEK_API_KEY=sk-...

# Database Configuration (sa admin credentials)
SQL_ADMIN_USER=sa
SQL_ADMIN_PASSWORD=FdeEnterprisePass123!

# Database Configuration (agent read-only credentials)
SQL_AGENT_USER=USR_FDE_RO
SQL_AGENT_PASSWORD=AgentPassword2026!

# Database Host (localhost for local Docker, or EC2 Private/Public IP)
SQL_SERVER_HOST=localhost
SQL_SERVER_PORT=1433
```

### 3. Spin Up Persistent MSSQL Docker Container
```bash
# Ensure persistent directory exists
docker run \
  -e "ACCEPT_EULA=Y" \
  -e "MSSQL_SA_PASSWORD=FdeEnterprisePass123!" \
  -p 1433:1433 \
  --name mssql-db \
  --restart always \
  -v /var/opt/mssql/data:/var/opt/mssql/data \
  -d mcr.microsoft.com/mssql/server:2022-latest
```

### 4. Ingest Telemetry & SOP Knowledge
```bash
# 1. Ingest telemetry CSV into MSSQL
python scripts/ingest_legacy_data.py

# 2. Execute scripts/setup_secuirty_and_view.sql in your SQL client (as sa)

# 3. Ingest SOPs into Pinecone Vector DB
python scripts/ingest_sop_document.py
```

### 5. Launch the Streamlit Console
```bash
streamlit run src/ui.py
```
Open `http://localhost:8501` in your browser.

---

## 🛡️ Enterprise Auditability & Traceability

Every interaction in the dispatch console produces a dual-trace record:
1. **Interactive UI Traces**: Expandable cards reveal exact SQL queries executed, raw JSON returned by corridor APIs, and relevant clauses retrieved from vector memory.
2. **Immutable SQL Audit Trail (`FDE_VIEWS.AgentAuditLog`)**:
   - `SessionID`: Unique UUID tracking the dispatcher's conversation thread.
   - `NodeExecuted`: Graph node trace (`reasoner`, `tools`, `reasoner_final`).
   - `ToolName`: Exact subsystem invoked.
   - `Content`: Full input parameter or output payload.
   - Protected behind an **Admin Authentication Gate** requiring administrative credentials to inspect.

---

## 👨‍💻 Author & Contributions
Engineered and maintained by **Srijan** ([@Srijan-120](https://github.com/Srijan-120)).
Open to contributions, issue reports, and architecture enhancements!
