# Project Architecture: Governance-Aware GenAI Agent 🛡️🏗️

This document provides a detailed technical deep-dive into the architecture of the Governance-Aware GenAI Agent. It explains how the system integrates BigQuery, Dataplex, and the Model Context Protocol (MCP) to create a secure, metadata-driven reasoning loop.

---

## 1. High-Level System Architecture

The system is organized into three distinct layers, each serving a specific role in the governance-aware lifecycle:

### A. The Data Lake Layer (BigQuery)
This is where the actual data resides. It simulates a realistic enterprise environment with multiple datasets:
*   **Production/Mart Datasets:** Highly governed, clean data (e.g., `finance_mart`).
*   **Ad-hoc/Sandbox Datasets:** Raw, unverified, or temporary data (e.g., `analyst_sandbox`).

### B. The Governance Layer (Dataplex Catalog)
Dataplex acts as the **"Brain" of the project**. Instead of the LLM guessing what tables exist, it queries Dataplex to find metadata "Aspects" that define the table's quality and purpose.
*   **Aspect Type (`official-data-product-spec`):** A schema that defines governance fields like `is_certified` (Boolean) and `product_tier` (Enum).
*   **Entries:** Metadata records in Dataplex that represent the BigQuery tables.

### C. The Agentic Layer (ADK & MCP)
This is the interface between the user and the data.
*   **MCP Server (GenAI Toolbox):** Standardizes the way tools (Dataplex Search/Lookup) are exposed to the AI.
*   **Sequential Agents (ADK):** A multi-agent system where a "Researcher" finds the data and a "Formatter" explains it.

---

## 2. Detailed Workflows

### Workflow 1: Governance Provisioning (Build Time)
This workflow prepares the "governance context" before any user asks a question.

1.  **Terraform Deployment:** Creates the BQ datasets/tables and the Dataplex Aspect Type.
2.  **Payload Generation:** `generate_payloads.sh` creates YAML files representing governance states (e.g., Table A is GOLD, Table B is BRONZE).
3.  **Metadata Attachment:** `apply_governance.sh` uses `gcloud dataplex` to attach these YAML aspects to the Dataplex Entries.

### Workflow 2: Agent Reasoning Loop (Run Time)
When a user asks: *"Show me certified finance data,"* the agent follows this sequence:

1.  **Phase 1: Metadata Discovery:** The agent uses `search_aspect_types` to understand the valid Enum values and Boolean flags (e.g., what does "Certified" mean in the metadata?).
2.  **Phase 2: Precise Search:** The agent constructs a search query using the strict Dataplex aspect syntax: `project.location.aspect.field=value`.
3.  **Phase 3: Verification & Response:** The agent uses `lookup_entry` to confirm the table details and then explains *why* it recommended that specific asset.

---

## 3. Data Flow Diagram

```text
[ USER QUERY ] 
      |
      v
[ ADK ROOT AGENT ] (Greeter/State Manager)
      |
      +---> [ GOVERNANCE WORKFLOW ]
               |
               +---> [ RESEARCHER AGENT ] <---(MCP)--- [ DATAPLEX CATALOG ]
               |         (Phase 1: Discover Rules)           |
               |         (Phase 2: Search Assets)            |
               |         (Phase 3: Verify Details)           |
               |                                             |
               +---> [ COMPLIANCE FORMATTER ]                |
                         (Translates JSON to English)        |
      |                                                      |
[ FINAL ANSWER ] <-------------------------------------------+
```

---

## 4. Key Security Features

*   **No Direct SQL Access:** The agent **never** generates SQL. It only retrieves metadata. This prevents data exfiltration and prompt injection attacks targeting the database schema.
*   **Metadata Boundaries:** The agent is system-prompted to *only* recommend assets that have the `is_certified=true` flag.
*   **MCP Isolation:** The tools exposed via `tools.yaml` are strictly read-only, ensuring the agent cannot modify governance policies or data.

---

## 5. Technology Stack

| Component | Technology |
| :--- | :--- |
| **Cloud Provider** | Google Cloud Platform (GCP) |
| **Data Warehouse** | BigQuery |
| **Data Governance** | Dataplex (Catalog & Aspects) |
| **Agent Framework** | Google Agent Development Kit (ADK) |
| **Tool Protocol** | Model Context Protocol (MCP) |
| **Infrastructure** | Terraform |
| **Language** | Python / Shell |
