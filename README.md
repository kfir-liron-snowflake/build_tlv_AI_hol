# Getting Started with Cortex Agent Evaluations

A hands-on lab that walks you through building a full **Marketing Campaigns Analytics System** using Snowflake Cortex Agents — and then **evaluating and optimizing** that agent's performance.

By the end of this lab you will have:
- A populated marketing campaigns database with structured and unstructured data
- A **Cortex Analyst** semantic view for text-to-SQL queries
- A **Cortex Search** service for full-text content discovery
- A **stored procedure** that generates HTML campaign reports
- A **Cortex Agent** that orchestrates all three tools
- Baseline and optimized **agent evaluation runs** to measure improvement

---

## Table of Contents

1. [Before You Begin](#before-you-begin)
   - [Step 1 — Create a Snowflake Trial Account](#step-1--create-a-snowflake-trial-account)
   - [Step 2 — Register for the Agent Evaluations Feature](#step-2--register-for-the-agent-evaluations-feature)
2. [Prerequisites](#prerequisites)
3. [Architecture Overview](#architecture-overview)
4. [Part 1 — SQL Setup (Sections 1–10)](#part-1--sql-setup-sections-110)
   - [Section 1 — Database and Schema Creation](#section-1--database-and-schema-creation)
   - [Section 2 — Role Configuration](#section-2--role-configuration)
   - [Section 3 — GitHub Integration](#section-3--github-integration)
   - [Section 4 — Create and Populate Tables](#section-4--create-and-populate-tables)
   - [Section 5 — Validate Data](#section-5--validate-data)
   - [Section 6 — Create Semantic View](#section-6--create-semantic-view)
   - [Section 7 — Create Cortex Search Service](#section-7--create-cortex-search-service)
   - [Section 8 — Create Report Generation Stored Procedure](#section-8--create-report-generation-stored-procedure)
   - [Section 9 — Test the Services](#section-9--test-the-services)
   - [Section 10 — Create the Cortex Agent](#section-10--create-the-cortex-agent)
5. [Part 2 — UI Steps (Sections 11–12)](#part-2--ui-steps-sections-1112)
   - [Section 11 — Run the Baseline Evaluation](#section-11--run-the-baseline-evaluation-snowsight-ui)
   - [Section 12 — Optimize the Agent](#section-12--optimize-the-agent-snowsight-ui)
   - [Section 13 — Conclusion](#section-13--conclusion)
6. [Troubleshooting](#troubleshooting)
7. [Cleanup](#cleanup)
8. [Resources](#resources)

---

## Before You Begin

Complete these two steps **before** running any SQL in the lab.

### Step 1 — Create a Snowflake Trial Account

1. Go to **[https://signup.snowflake.com/](https://signup.snowflake.com/)**
2. Fill in your details and, when prompted for edition and cloud region, select:
   - **Edition:** Enterprise
   - **Cloud provider:** AWS
   - **Region:** US West (Oregon)

> **Important:** The Agent Evaluations feature used in this lab is only available on Enterprise edition accounts in **AWS US West (Oregon)**. Choosing a different edition or region will prevent certain steps from working.

### Step 2 — Register for the Agent Evaluations Feature

Agent Evaluations is a gated feature. Once your trial account is active, register for access using the form below and provide your account locator.

**[Fill out the Agent Evaluations Feature Registration form](https://docs.google.com/forms/d/1Xbmu8qjZ3WDYGQZONeOSPlPA4X5_l6vPODMD8VpLDIk/edit)**

To find your **Account Locator**:
1. Log in to Snowsight
2. Click your name/initials in the bottom-left corner
3. Go to **Account** → **View account details**
4. Copy the value shown as **Account Locator**

---

## Prerequisites

| Requirement | Details |
|---|---|
| Snowflake edition | Enterprise |
| Cloud / Region | AWS US West (Oregon) |
| Snowflake role | `ACCOUNTADMIN` (or a role with `CREATE DATABASE` privileges) |
| Warehouse | `COMPUTE_WH` (created automatically if it does not exist) |
| Database role | `SNOWFLAKE.CORTEX_USER` — granted automatically by the script |
| Estimated runtime | 5–10 minutes |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                  MARKETING_CAMPAIGNS_DB                  │
│                      schema: AGENTS                      │
│                                                          │
│  Tables                  Services                        │
│  ──────                  ────────                        │
│  CAMPAIGNS               Semantic View                   │
│  CAMPAIGN_PERFORMANCE    └─ MARKETING_PERFORMANCE_ANALYST│
│  CAMPAIGN_CONTENT                                        │
│  CAMPAIGN_FEEDBACK       Cortex Search Service           │
│  EVALS_TABLE             └─ MARKETING_CAMPAIGNS_SEARCH   │
│                                                          │
│                          Stored Procedure                │
│                          └─ GENERATE_CAMPAIGN_REPORT_HTML│
│                                                          │
│                          Cortex Agent                    │
│                          └─ MARKETING_CAMPAIGNS_AGENT    │
└─────────────────────────────────────────────────────────┘
```

The agent exposes three tools to end users:

| Tool | Type | Purpose |
|---|---|---|
| `query_performance_metrics` | Cortex Analyst (text-to-SQL) | Quantitative analysis — revenue, ROI, clicks, conversions |
| `search_campaign_content` | Cortex Search | Qualitative discovery — copy, feedback, A/B test notes |
| `generate_campaign_report` | Generic (stored procedure) | Generate a full HTML report for a given campaign |

---

## Part 1 — SQL Setup (Sections 1–10)

> Open `SETUP.sql`, paste the full contents into a new Snowsight worksheet, and run each section top to bottom.

---

### Section 1 — Database and Schema Creation

Runs as `ACCOUNTADMIN`. Creates `MARKETING_CAMPAIGNS_DB`, the `AGENTS` schema, and a `COMPUTE_WH` warehouse (Small size) that are used throughout the lab.

---

### Section 2 — Role Configuration

Creates a least-privilege role (`AGENT_EVAL_ROLE`) and grants it all permissions needed to build and run the lab objects: tables, stages, semantic views, search services, procedures, agents, and evaluation tasks. `AGENT_EVAL_ROLE` is also set as the **default role** for your user so it is active automatically on next login.

Key grants include:
- `SNOWFLAKE.CORTEX_USER` database role for Cortex API access
- `SNOWFLAKE.AI_OBSERVABILITY_EVENTS_LOOKUP` application role for evaluation observability
- `IMPERSONATE` on the current user so the evaluation task can execute agent calls on your behalf
- `CREATE AGENT`, `CREATE SEMANTIC VIEW`, `CREATE CORTEX SEARCH SERVICE` on the schema

---

### Section 3 — GitHub Integration

Creates a Snowflake Git Repository integration pointing to the public Snowflake Labs repository. All CSV data files are loaded directly from this repository — no manual file upload required. After cloning, the script verifies the repository is accessible by listing the `/data` folder.

---

### Section 4 — Create and Populate Tables

Creates a CSV file format, then creates and loads four tables by reading CSV files directly from the cloned repository stage.

| Table | Records | Description |
|---|---|---|
| `CAMPAIGNS` | 25 | Campaign metadata — name, type, channel, budget, dates, status |
| `CAMPAIGN_PERFORMANCE` | 1,578 | Daily performance metrics — impressions, clicks, conversions, revenue, ROI |
| `CAMPAIGN_CONTENT` | 25 | Unstructured content — descriptions, marketing copy, A/B test notes |
| `CAMPAIGN_FEEDBACK` | 23 | Customer feedback — satisfaction scores, comments, recommendations |
| `EVALS_TABLE` | varies | Evaluation dataset — sample queries with expected tool selections |

The `EVALS_TABLE` is post-processed to parse the `EXPECTED_TOOLS` column from a JSON string into a `VARIANT` column used as ground truth during evaluations.

---

### Section 5 — Validate Data

Runs a `SELECT *` on each table to confirm all rows loaded correctly before proceeding.

---

### Section 6 — Create Semantic View

Creates `MARKETING_PERFORMANCE_ANALYST`, a **Semantic View** that Cortex Analyst uses to generate SQL from natural language. It defines:

- **Dimensions** — campaign name, type, channel, audience, status, dates
- **Metrics** — pre-aggregated measures (total revenue, total impressions, average ROI, etc.)
- **Relationships** — join between `CAMPAIGNS` and `CAMPAIGN_PERFORMANCE` on `campaign_id`

---

### Section 7 — Create Cortex Search Service

Creates `MARKETING_CAMPAIGNS_SEARCH`, a **Cortex Search Service** that indexes all unstructured text from `CAMPAIGN_CONTENT` and `CAMPAIGN_FEEDBACK`. A `UNION ALL` query combines both sources into a single `combined_text` column for semantic search, refreshed on a 1-hour lag.

---

### Section 8 — Create Report Generation Stored Procedure

Creates `GENERATE_CAMPAIGN_REPORT_HTML(campaign_id NUMBER)`, a stored procedure that:

1. Queries `CAMPAIGNS`, `CAMPAIGN_PERFORMANCE`, and `CAMPAIGN_FEEDBACK` for the given campaign
2. Assembles a styled HTML document with a metrics table and feedback cards
3. Writes the HTML file to the `@CAMPAIGN_REPORTS` internal stage
4. Returns a direct Snowsight link to the uploaded file

The procedure is registered as the `generate_campaign_report` tool in the agent so users can trigger it via natural language (e.g. _"Generate a report for the Spring Fashion campaign"_).

---

### Section 9 — Test the Services

Runs validation queries against each service independently before wiring them into the agent:
- A `SEMANTIC_VIEW()` query to test Cortex Analyst text-to-SQL
- A `SEARCH_PREVIEW()` call to test the Cortex Search service
- A `CALL GENERATE_CAMPAIGN_REPORT_HTML(1)` to test the stored procedure

---

### Section 10 — Create the Cortex Agent

Creates `MARKETING_CAMPAIGNS_AGENT` and wires all three tools to their backing services:

| Tool | Backing service |
|---|---|
| `query_performance_metrics` | `MARKETING_PERFORMANCE_ANALYST` semantic view |
| `search_campaign_content` | `MARKETING_CAMPAIGNS_SEARCH` Cortex Search service |
| `generate_campaign_report` | `GENERATE_CAMPAIGN_REPORT_HTML` stored procedure |

Once created, navigate to the agent in Snowsight and try these sample questions:
- _"What campaigns have the highest ROI?"_
- _"What feedback did customers give about email campaigns?"_
- _"Generate a report for campaign ID 1"_

---

## Part 2 — UI Steps (Sections 11–12)

> These sections are performed entirely in the Snowsight UI — no SQL required.

---

### Section 11 — Run the Baseline Evaluation _(Snowsight UI)_

1. Navigate to your agent (`MARKETING_CAMPAIGNS_AGENT`) in Snowsight
2. Click the **Evaluations** tab → **New Evaluation Run**

![Evaluations tab showing existing runs](images/evaluation-runs.png)

3. Name the run (e.g. `baseline_eval`) and optionally add a description → **Next**
4. Select **Create New Dataset**
   - Input table: `MARKETING_CAMPAIGNS_DB.AGENTS.EVALS_TABLE`
   - New dataset name: `MARKETING_CAMPAIGNS_DB.AGENTS.QUICKSTART_EVALSET`
   - → **Next**

![Select dataset screen](images/select-dataset.png)

5. Column mapping:
   - **Query Text column**: `INPUT_QUERY`
   - Enable all available metrics
   - For **Tool Selection Accuracy**, **Tool Execution Accuracy**, and **Answer Correctness** → reference the `GROUND_TRUTH_DATA` column

![Select metrics screen](images/select-metrics.png)

6. Click **Create Evaluation**

![Evaluation run in progress](images/run-in-progress.png)

Results populate in approximately 3–5 minutes. Note your baseline scores before proceeding to optimization.

![Tool Selection Accuracy record detail](images/tsa.png)

---

### Section 12 — Optimize the Agent _(Snowsight UI)_

`SETUP.sql` contains two blocks of optimized instructions (Section 12). Copy them into the agent UI to improve routing consistency and response quality.

#### Step 1 — Add Orchestration Instructions

1. Open the agent in Snowsight → **Edit** → **Orchestration** tab
2. Paste the **Orchestration Instructions** block from `SETUP.sql` (Section 12, first `$$` block)

![Agent orchestration instructions](images/Agent%20instructions.png)

The instructions define four strict routing rules:
- **Rule 1** — Use `query_performance_metrics` for all numerical/quantitative queries
- **Rule 3** — Use `search_campaign_content` for all qualitative/text queries  
- **Rule 4a** — Use `generate_campaign_report` when a report is explicitly requested (always resolve `campaign_id` via `query_performance_metrics` first)
- **Rule 4b** — For mixed queries, call `query_performance_metrics` first, then `search_campaign_content`

#### Step 2 — Add Response Instructions

1. Navigate to the **Response** tab in the same Edit dialog
2. Paste the **Response Instructions** block from `SETUP.sql` (Section 12, second `$$` block)

The instructions enforce consistent formatting rules: leading direct answer, tabular metric presentation, consistent number formatting, source citations, and professional tone.

#### Step 3 — Re-run the Evaluation

Repeat **Section 11** with a new run name (e.g. `optimized_eval`) and select **Existing Dataset** to reuse the dataset created earlier. Compare the metrics side-by-side with your baseline run to quantify the improvement.

---

### Section 13 — Conclusion

---

## Troubleshooting

| Symptom | Resolution |
|---|---|
| You get an error running the script, or the Evaluations tab is not visible on your agent | Log out of Snowsight and log back in. The feature flag is applied at session start, so a fresh login is required after access is granted. |
| Evaluation metrics do not improve after adding optimization instructions | This is expected and not a problem. Prompt sensitivity varies by query; partial improvement or no change on some metrics is a normal outcome. Focus on overall trends across the full eval set. |
| Red triangle alert icon on an evaluation run | This is expected on trial accounts and reflects a known limitation of the trial environment. It does not affect the evaluation results or metrics — proceed normally. |

---

## Cleanup

To remove all objects created by this lab:

```sql
USE ROLE ACCOUNTADMIN;
DROP DATABASE IF EXISTS MARKETING_CAMPAIGNS_DB;
DROP ROLE IF EXISTS AGENT_EVAL_ROLE;
DROP API INTEGRATION IF EXISTS GIT_API_INTEGRATION_AGENT_EVAL_QUICKSTART;
```

---

## Resources

- [Snowflake Cortex Agents documentation](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents)
- [Cortex Analyst — Semantic Views](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-analyst/semantic-view)
- [Cortex Search documentation](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-search/cortex-search-overview)
- [Agent Evaluations documentation](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents/agent-evaluation)
- [Source repository](https://github.com/Snowflake-Labs/sfguide-getting-started-with-cortex-agent-evaluations)
