
# Technical Requirements Document (TRD): Project Argus

## 1. System Objective & Problem Statement
High-throughput telemetry platforms stream millions of vehicle records into cloud-based data lakehouses. Internal analysts, engineers, and developers require continuous SQL access to this historical data (via AWS Athena, Trino, or ClickHouse) to extract operational business metrics. This model introduces critical security vectors:
1. **The Static Privilege Trade-Off:** Traditional cloud access controls (AWS IAM) are binary and rigid. Security teams must either grant full access to tables or block users entirely. There is no native mechanism to parse conversational intent or dynamically filter records at the column or row level based on user identity.
2. **Personally Identifiable Information (PII) Leakage:** Telemetry files contain sensitive parameters including Vehicle Identification Numbers (VINs), precise floating-point GPS coordinate strings, and driving histories that can track driver home locations.
3. **The Static Context Gap:** Large Language Models (LLMs) used for conversational text-to-SQL workflows have zero awareness of real-time infrastructure topologies, active IAM access policies, or dynamic enterprise security compliance files.

**Project Argus** solves this by engineering an autonomous, stateful **AI Governance & Security Agent Proxy** utilizing the **Model Context Protocol (MCP)**. The agent intercepts conversational data intents, actively inspects live AWS IAM/catalog configurations via JSON-RPC, evaluates risk parameters via semantic Vector RAG, and programmatically enforces dynamic zero-trust masking barriers before query results escape the platform boundary.

---

## 2. Functional & Architecture Targets
To satisfy enterprise data platform compliance metrics, the system must enforce the following performance and security constraints:
* **Stateful Execution Control:** All interaction logic must run inside an iterative, cyclic agent loop capable of tool validation, self-correction, and conditional routing.
* **Network-Isolated Tooling:** The agent must communicate with underlying cloud services exclusively using the **Model Context Protocol (RPC over standard I/O or JSON-RPC streams)**.
* **Low-Inference Cost Profile:** Must enforce an intelligent Semantic LLM Cache boundary to completely avoid calling the LLM for repetitive or highly similar user intents.
* **Absolute Zero-Trust Posture:** The agent never retains or utilizes long-lived master database or AWS administration credentials. It must provision temporary, short-lived tokens per verified transaction.

---

## 3. Technology Stack Choice & Rationale

| Infrastructure Layer | Component Selection | Strategic Engineering Rationale |
| :--- | :--- | :--- |
| **Agentic Loop Engine** | **LangGraph** | Chosen over basic sequential scripts. Allows for stateful, cyclic graphs where the agent can invoke tools via MCP, evaluate the raw output, catch security flags, and conditionally re-route its execution flow before returning data. |
| **Context Protocol** | **Model Context Protocol (MCP)** | Open standard designed by Anthropic. Standardizes how an LLM securely connects to tools and data sources. Allows the model to safely read system states via decoupled, structured local/remote MCP servers using standardized JSON-RPC protocols. |
| **Semantic Guardrails** | **Qdrant / PGVector (RAG)** | High-performance vector database. Encodes the company's data classification schedules and privacy regulations into vector spaces. The agent queries this cluster via vector similarity to map dynamic security compliance definitions to targeted columns. |
| **Performance Boundary** | **Semantic LLM Cache (Redis/Qdrant)**| Intercepts user requests before they hit the LLM. Converts the prompt to an embedding; if it matches a previously validated request within a defined threshold, it returns the secure, cached result instantly—saving up to **80% in API token costs**. |
| **Authorization Layer** | **AWS Security Token Service (STS)** | Instead of hardcoded database keys, the agent interacts with AWS STS to generate short-lived, highly restricted IAM Session Tokens. These tokens are dynamically scoped to authorize access *only* to the specific Parquet files matching that single query. |
| **Target Storage Engines** | **AWS Athena / ClickHouse** | The underlying data layout engineered in Project Chronos (ClickHouse columnar matrices for real-time data + S3 Parquet tables managed via Apache Iceberg schemas). |

---

## 4. System Mechanics & The Core Agentic Loop

When an internal user or analyst interfaces with Project Argus, the LangGraph engine coordinates execution across five stateful phases:

```text
[ User Prompt ] ──► (1. Semantic Cache Check) ──► [ Match Found ] ──► Return Cached JSON
                          │
                   [ Cache Miss ]
                          ▼
                (2. Evaluation Node) ◄──► [ Vector DB RAG (Privacy Rules) ]
                          │
                          ▼
                (3. Inspection Node) ◄──► [ AWS Cloud MCP Server (Live IAM Checks) ]
                          │               [ Data Catalog MCP Server (Schema Maps) ]
                          ▼
                (4. Query Rewriter)  ──► Generates PII-Masked SQL Script
                          │
                          ▼
                (5. Token Execution) ──► Calls AWS STS -> Executes Query -> Delivers Output
```

1. **The Semantic Cache Ingress:** The prompt is tokenized and embedded. The engine scans the semantic cache. If a high-similarity match is found, the secure database result is served instantly from memory, bypassing downstream nodes.
2. **The Policy Evaluation Node:** On a cache miss, the agent executes a vector similarity search (RAG) against the internal compliance policies. If the model determines that a Junior Analyst is attempting to view restricted columns (like `latitude`, `longitude`, or `vin`), it marks those components for mandatory masking.
3. **The Infrastructure Inspection Node:** The agent invokes its custom **MCP Servers** via JSON-RPC. The model actively queries the cloud infrastructure to verify the caller's actual identity, checking active AWS IAM bucket restrictions and retrieving the current Apache Iceberg table schemas.
4. **The Safe Transformation Node:** The agent intelligently re-writes the SQL query or modifies the execution plan. It applies runtime transformation functions:
    * **Spatial Truncation:** Truncates precise float coordinates (e.g., `12.971573, 77.594563` becomes `12.97, 77.59`), obscuring specific house addresses while preserving broad geographic region analytics.
    * **Cryptographic Hashing:** Obscures hardware strings (VINs) via secure salted hashes.
5. **The Zero-Trust Token Execution:** The agent passes the sanitized query to AWS STS, retrieves a temporary execution token valid for **15 minutes**, calls AWS Athena or ClickHouse to fetch the masked data stream, and presents the clean dataset to the user.

---

## 5. Architectural Guarantees & Scope Boundaries

### 🛡️ System Guarantees
* **Dynamic Content Masking:** Protects the cloud network boundary from accidental PII data leakage without requiring duplicate row or table mirroring in storage.
* **Deterministic Security Overrides:** Hardcoded programmatic validation checks run at the end of the agent loop; if the LLM attempts to output unmasked PII due to prompt injection, the system intercepts and drops the packet deterministically.
* **Complete Infrastructure Isolation:** The LLM never communicates with your core production networks directly. It operates strictly through the metadata abstraction layers exposed by the JSON-RPC MCP servers.

### ❌ Out of Scope
* **Public Internet Gateways:** This agent is strictly scoped as an internal enterprise data governance tool. It does not interface with public user authentication or multi-tenant billing engines.
* **Database Write Capabilities:** The agent is restricted to an immutable, read-only paradigm. It cannot execute mutations, inserts, deletes, or table drops.
