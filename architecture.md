# Project Architecture: Governance-Aware GenAI Agent 🛡️🏗️

This document provides a detailed technical deep-dive into the architecture of the Governance-Aware GenAI Agent. It explains how the system integrates BigQuery, Dataplex, and the Model Context Protocol (MCP) to create a secure, metadata-driven reasoning loop.

---

## 1. High-Level System Architecture

The system is organized into three distinct layers, each serving a specific role in the governance-aware lifecycle:

### A. The Data Lake Layer (BigQuery)
This layer simulates a realistic enterprise data environment.
*   **Production/Mart Datasets (`finance_mart`, `marketing_prod`):** These contain high-quality, production-ready tables.
*   **Ad-hoc/Sandbox Datasets (`analyst_sandbox`):** Contains raw or temporary data that has not passed governance checks.
*   **Terraform Provisioning:** The infrastructure is defined in `terraform/main.tf`, which not only creates the tables but also uses BigQuery DML jobs to populate them with sample data, ensuring a "ready-to-query" environment.

### B. The Governance Layer (Dataplex Catalog)
Dataplex acts as the **"Governance Brain"**. It allows the agent to distinguish between "trusted" and "untrusted" data without ever seeing the raw rows.
*   **Aspect Type (`official-data-product-spec`):** A custom metadata schema that defines:
    *   `product_tier`: Enum (GOLD_CRITICAL, SILVER_STANDARD, BRONZE_ADHOC).
    *   `data_domain`: Enum (FINANCE, MARKETING, LOGISTICS).
    *   `is_certified`: Boolean (The primary "Safety Switch").
*   **Automation:** `generate_payloads.sh` and `apply_governance.sh` simulate a CI/CD pipeline that automatically tags tables with these aspects based on their location and dataset.

### C. The Agentic Layer (ADK & MCP)
This is the intelligence layer that orchestrates the governance logic.
*   **MCP Server (Model Context Protocol):** Configured in `mcp_server/tools.yaml`, it standardizes the exposure of Dataplex APIs (Search, Lookup) as tools that the agent can execute.
*   **Google Agent Development Kit (ADK):** Used in `mcp_server/agent.py` to build a multi-agent system.

---

## 2. Detailed Workflows

### Workflow 1: Governance Provisioning (Build Time)
This workflow establishes the "Truth" that the agent will later rely on.

1.  **Infrastructure as Code:** Terraform deploys the Dataplex Aspect Type and BigQuery resources.
2.  **Policy Definition:** `generate_payloads.sh` reads the Project ID and generates specific YAML configurations for each table scenario (e.g., Financial Reports are GOLD/Certified).
3.  **Governance Enforcement:** `apply_governance.sh` updates the Dataplex Catalog. This step is critical because it moves governance from "documentation" to "machine-readable metadata".

### Workflow 2: Multi-Agent Reasoning Loop (Run Time)
When a user asks: *"I need to see certified financial performance for last month,"* the agent initiates a sequential workflow:

#### Phase 1: Context Initialization (Root Agent)
*   The **Greeter Agent** captures the user's intent and saves it to the **Agent State** (using `add_prompt_to_state`). This ensures that subsequent agents in the chain have access to the original request.

#### Phase 2: Metadata Discovery (Researcher Agent)
*   The agent **cannot** search for data until it knows the "rules". It calls `search_aspect_types` for `official-data-product-spec`.
*   It parses the returned schema to identify that it needs to filter by `data_domain=FINANCE` and `is_certified=true`.

#### Phase 3: Strict Aspect Search (Researcher Agent)
*   The agent constructs a complex Dataplex search query. 
*   **Query Example:** `[project].[location].official-data-product-spec.data_domain=FINANCE [project].[location].official-data-product-spec.is_certified=true`
*   This search returns only the entries that strictly match the governance criteria.

#### Phase 4: Compliance Formatting (Formatter Agent)
*   The **Compliance Formatter Agent** receives the raw JSON results.
*   It translates the technical metadata (e.g., "Table: fin_monthly_closing") into a helpful recommendation, explaining *why* it chose that table (e.g., "This table is Gold-tier and officially certified for internal use").

---

## 3. Component Interaction Diagram

```text
[ USER ] 
   |
   v
[ ADK ROOT AGENT ] <---> [ AGENT STATE (PROMPT) ]
   |
   +---(Sequential Invoke)---> [ GOVERNANCE WORKFLOW ]
                                   |
                                   +---[ RESEARCHER AGENT ] 
                                   |      |
                                   |      +-- (MCP/JSON-RPC) --> [ MCP SERVER ]
                                   |                                |
                                   |                                +-- (API) --> [ DATAPLEX ]
                                   |
                                   +---[ COMPLIANCE FORMATTER ]
                                          |
                                          v
[ FINAL RESPONSE ] <----------------------+
```

---

## 4. Security & Safety Boundaries

*   **Prompt Injection Mitigation:** By forcing the agent to query Dataplex for "rules" first, the system prevents users from "tricking" the agent into using unauthorized tables. The agent relies on the Dataplex schema, not the user's phrasing.
*   **Zero-Trust Data Access:** The agent **never** generates or executes SQL. It only identifies the *correct* table. In a real-world scenario, the output of this agent would be passed to a separate, restricted query agent.
*   **Strict Search Syntax:** The Researcher Agent is system-prompted with a specific syntax for `search_entries` that ignores standard table names and focuses entirely on governance Aspects.

---

## 5. Technology Stack

| Component | Technology | Description |
| :--- | :--- | :--- |
| **Cloud Provider** | Google Cloud Platform | Hosting and Managed Services |
| **Data Warehouse** | BigQuery | Storage and Sample Data |
| **Data Governance** | Dataplex Catalog | Metadata Management & Search |
| **Agent Framework** | Google ADK | Multi-agent orchestration and State management |
| **Tool Interface** | MCP (Model Context Protocol) | Standardized Tool access for LLMs |
| **IaC** | Terraform | Infrastructure provisioning |
| **Language** | Python / Bash | App logic and Automation scripts |
