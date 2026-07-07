
# Data Governance-Aware GenAI Agent 🛡️🤖

This repository contains the infrastructure, governance automation scripts, and application code for building a **Governance-Aware GenAI Agent**. 

The project demonstrates how to use **Google Cloud Dataplex Metadata** as a strict context boundary for LLMs, ensuring that an AI agent only recommends trusted, certified data assets and avoids hallucinations.

---

## 🚀 Overview

In a messy data lake, GenAI agents often struggle to distinguish between production-grade tables and ad-hoc sandboxes. This project solves that by:
1.  **Defining Governance via Dataplex:** Using custom Aspect Types to tag tables with metadata (e.g., `is_certified`, `product_tier`).
2.  **Strict Reasoning Algorithm:** Training the LLM to perform "Metadata Discovery" before searching for data.
3.  **Model Context Protocol (MCP):** Exposing Dataplex tools to the agent through a standard, secure interface.
4.  **Agent Development Kit (ADK):** Orchestrating multiple specialized agents (Researcher & Formatter) to handle the governance loop.

---

## 🛠️ Project Structure

```text
.
├── terraform/                # Infrastructure as Code (BigQuery & Dataplex)
├── aspect_payloads/          # Generated metadata payloads (YAML)
├── mcp_server/               # Production Agent (Python ADK + MCP)
│   ├── agent.py              # Multi-agent orchestration logic
│   └── tools.yaml            # Tool definitions for MCP
├── generate_payloads.sh      # Script to create Dataplex Aspect YAMLs
├── apply_governance.sh       # Script to attach Aspects to BQ tables
├── GEMINI.md                 # System instructions for local prototyping
└── README.md                 # This file
```

---

## 🚦 Getting Started

### 1. Prerequisites
- A Google Cloud Project with Billing enabled.
- [Google Cloud SDK](https://cloud.google.com/sdk/docs/install) installed and authenticated.
- [Terraform](https://developer.hashicorp.com/terraform/downloads) installed.

### 2. Infrastructure Setup
Deploy the BigQuery data lake and Dataplex Aspect Type:
```bash
cd terraform
terraform init
terraform apply -var="project_id=$(gcloud config get-value project)" -var="region=us-central1"
```

### 3. Apply Governance Metadata
Generate and apply the "Official Data Product" tags to your tables:
```bash
chmod +x *.sh
./generate_payloads.sh
./apply_governance.sh
```

### 4. Local Prototyping (Gemini CLI)
If you have the `gemini` CLI installed with the `knowledge-catalog` extension:
```bash
# Set your project environment variable
export DATAPLEX_PROJECT=$(gcloud config get-value project)

# Start a session using the provided system instructions
gemini --system-instruction-file GEMINI.md
```

---

## 🧠 The Governance Algorithm

The agent is instructed to follow a strict **3-Phase Execution Workflow**:

1.  **Phase 1: Metadata Discovery:** Look up the `official-data-product-spec` Aspect Type to understand the valid Enum values (e.g., `GOLD_CRITICAL`, `FINANCE`).
2.  **Phase 2: Precise Search:** Construct a Dataplex search query using the discovered schema.
    - *Example Query:* `build-with-ai.us-central1.official-data-product-spec.is_certified=true`
3.  **Phase 3: Verification:** Use `lookup_entry` to confirm table details and explain the recommendation to the user based on its governance status.

---

## 🏢 Enterprise Deployment (MCP & ADK)

For production, the project uses the **Google Agent Development Kit (ADK)** and an **MCP Server**:

- **Researcher Agent:** Specifically tuned to interpret metadata and perform strict Dataplex searches.
- **Formatter Agent:** Takes JSON metadata and translates it into a compliant, user-friendly response.

To set up the MCP environment:
1. Copy `mcp_server/env.temp` to `mcp_server/.env`.
2. Populate the required variables (`GOOGLE_CLOUD_PROJECT`, `MCP_SERVER_URL`).
3. Install dependencies: `pip install -r mcp_server/requirements.txt`.

---

## 📚 Codelab Series

This repository is part of a two-part developer series:
*   **[Part 1: Build the Data Foundation](https://codelabs.developers.google.com/governance-context-part1)**
*   **[Part 2: Deploy an Enterprise Agent with MCP](https://codelabs.developers.google.com/governance-context-part2)**

---

## ⚖️ License
This repository is licensed under the Apache 2.0 License. See `LICENSE` (if provided) for details.
