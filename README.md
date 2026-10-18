# Autonomous Multi Agent Financial Analysis System

This project presents an agentic financial research system that combines:
* **Hierarchical Orchestration** (Investment Research Supervisor, Planner, and Router)
* **Domain Specialist Agents** (Market, News, and Earnings Analysts)
* **Self Reflective Quality Control** (Evaluator and Optimizer Loop)
* **Persistent Cross Run Memory** (LangGraph StateGraph, SQLite Checkpointer, and InMemoryStore)

The goal is to automate institutional grade equity research by dynamically planning investigations, routing tasks to specialized data tools, enforcing strict factual grounding against live market data, and retaining structural company insights across independent research runs.

## Project Overview

The complete implementation is available in the project notebook (`Project_Notebook_team_v5.ipynb`).

The system follows a multi stage agentic workflow:
1. **Persistent Memory Retrieval:** Recalls enduring analytical takeaways from prior runs on the target ticker.
2. **Research Planning:** Generates an ordered, step by step investigation plan tailored to the user query.
3. **Dynamic Tool Routing:** Selects only the relevant specialist agents from the tool registry based on query intent.
4. **Specialist Evidence Collection:** Executes live data retrieval and domain analysis across Market, News, and Earnings pipelines.
5. **Draft Synthesis:** Aggregates multi source specialist evidence into a unified research report.
6. **Evaluator Optimizer Quality Gate:** Audits the draft across five quantitative criteria and refines the output if quality thresholds are not met.
7. **Memory Consolidation:** Extracts new long term structural insights and commits them to the persistent store.

## Agent & Workflow Architecture

**Supervisor: Investment Research Agent**
* **Planner (`plan_research_steps`):** Inspects registered tool descriptions and outputs a structured JSON array of investigation steps.
* **Router (`select_research_tools`):** Evaluates query scope and dispatches tasks only to necessary specialists (`market`, `news`, `earnings`) alongside an explicit routing rationale.

**Specialist 1: Market Agent (`market_agent`)**
* Retrieves 1 year daily OHLCV history and company fundamentals via `yfinance`.
* **Computed Outputs:**
  * `current_price`, `daily_change`, and `daily_change_pct`
  * Multi horizon returns (`return_1m_pct`, `return_3m_pct`, `return_6m_pct`)
  * `annualized_volatility_pct` and 52 week high/low distance
  * `volume_ratio` (latest volume vs. 1 year average) and valuation multiples (`pe_ratio`, `forward_pe`, `market_cap`, `eps`)

**Specialist 2: News Agent (`news_agent` / 5 Stage Prompt Chain)**
* Processes real time articles from `NewsAPI` through a strict sequential prompt chain where each stage's output feeds the next:
  1. `ingest_news` -> Fetches top 5 recent English headlines and publisher metadata.
  2. `preprocess_news` -> Normalizes raw JSON into clean `Publisher | Headline` strings.
  3. `classify_news` -> Tags stories as *Earnings*, *Market Trends*, *Product Updates*, or *Macroeconomics* with *Bullish*, *Bearish*, or *Neutral* sentiment.
  4. `extract_information` -> Isolates concrete financial figures, dates, and regulatory events.
  5. `summarize_news` -> Produces an executive brief (*Executive Overview*, *Key Catalysts and Drivers*, *Immediate Market Implications*).

**Specialist 3: Earnings Agent (`earnings_agent`)**
* Extracts quarterly and annual reported income statements from `Yahoo Finance`.
* **Generated Features:**
  * Up to 5 periods of `Total Revenue`, `Diluted/Basic EPS`, and `Net Income`
  * `latest_quarter_vs_same_quarter_last_year` percentage growth
  * `latest_fiscal_year_vs_prior_fiscal_year` percentage growth
  * Explicit qualification when forward management guidance is unavailable

**Quality Gate: Evaluator & Optimizer Agents**
* **Evaluator (`evaluate_research_report`):** Audits the synthesized draft strictly against raw specialist JSON evidence across five dimensions (0 to 5 scale):
  * `factual_grounding`, `completeness`, `source_attribution`, `financial_reasoning`, and `clarity`
  * Outputs `overall_score`, `critical_issues`, `issues`, and `revision_instructions`.
* **Optimizer (`optimize_research_report`):** Triggered automatically if `overall_score < 4.0`, any individual criterion scores `< 3.0`, or any `critical_issues` exist. Rewrites the draft and re evaluates to ensure quality improvement.

**Memory Layer: LangGraph Persistence**
* **Short Term State (`SqliteSaver`):** Checkpoints graph execution threads (`ResearchState`) to a local SQLite database (`research_memory.db`).
* **Long Term Store (`InMemoryStore`):** Saves and retrieves enduring ticker insights under the `("memory", ticker)` namespace across separate sessions.

## Pipeline Flow

`User Query + Ticker` $\rightarrow$ `Memory Retrieval (InMemoryStore)` $\rightarrow$ `Planner` $\rightarrow$ `Dynamic Router` $\rightarrow$ `Specialist Agents (Market / News / Earnings)` $\rightarrow$ `Synthesis Agent` $\rightarrow$ `Evaluator Agent` $\rightarrow$ `Optimizer Agent (If Needed)` $\rightarrow$ `Final Report & Memory Commit`

## Evaluation & Validation Summary (`MSFT` Target Run)

**Specialist Data Extraction (`MSFT`)**
* **Market Agent:** Captured current price ($535.07, +2.38% daily), 3 month return (+39.27%), 6 month return (+39.77%), annualized volatility (32.82%), trailing P/E (29.11), and forward P/E (22.60).
* **Earnings Agent:** Computed Q2 YoY growth across Revenue (+17.75% to $90.01B), EPS (+31.78% to $4.81), and Net Income (+31.33% to $35.77B), alongside annual YoY Revenue growth (+17.79% to $331.84B).
* **News Agent:** Identified AI capex trends ($2.6T industry spend) alongside regulatory headwinds (US H1B green card scrutiny and Italian gaming probe).

**Evaluator Agent Quality Audit**
* **Factual Grounding:** `5.0 / 5.0`
* **Completeness:** `5.0 / 5.0`
* **Source Attribution:** `5.0 / 5.0`
* **Financial Reasoning:** `4.0 / 5.0`
* **Clarity:** `5.0 / 5.0`
* **Overall Score:** `4.8 / 5.0` (`critical_issues: 0`, `quality_gate_passed: True`)

**Cross Run Memory Verification**
* **Run 1 (Initial Earnings Query):** Started with zero prior context (`No historical notes available`), completed full multi agent synthesis, and stored two enduring insights in the `MSFT` namespace.
* **Run 2 (Follow Up Market Implications Query):** Automatically recalled the two stored takeaways from Run 1 (**Rising Valuation Concerns** and **Increasing Regulatory Risks**) and integrated them into the immediate market analysis.

## Key Insights

* **Prompt Chaining Eliminates News Noise:** Breaking news analysis into five distinct cognitive steps prevents the local LLM from conflating raw headlines with verified financial metrics.
* **Selective Routing Saves Compute:** Dynamic tool selection ensures focused queries only trigger relevant specialists while broad queries engage the full specialist suite.
* **Deterministic Schema Enforcement:** Combining `temperature: 0.0` with strict JSON validation prevents structural hallucinations during routing and evaluation.
* **Context Augmented Memory:** Storing distilled structural takeaways in `InMemoryStore` enables cross run continuity without retraining or modifying underlying LLM weights.

## Hallucination & Factual Grounding Controls

To ensure institutional reliability when using a local 8B parameter model:
* Strict prompt constraints prohibit inventing prices, financial ratios, or management guidance.
* Missing metrics (such as forward guidance in Yahoo Finance statements) are explicitly flagged as unavailable.
* Arithmetic mean scores and YoY percentage deltas are calculated deterministically in Python rather than trusting LLM math.
* The Evaluator Optimizer controller penalizes any critical issue by `1.5x` when ranking candidate drafts.

## Limitations

* **Income Statement Granularity:** Yahoo Finance provides reported actuals but lacks qualitative earnings call transcripts and official forward guidance.
* **Headline Scope:** The News Agent evaluates the top 5 most recent articles per run to keep local inference latency manageable.
* **Sequential Local Inference Latency:** Running the planner, router, 5 stage news chain, specialist analyses, synthesis, evaluation, and memory extraction requires 10 to 12 sequential LLM calls per run on local hardware.

## Future Work

* **SEC EDGAR RAG Pipeline:** Integrate 10 K and 10 Q filings using vector embeddings to extract official management guidance and risk factors.
* **Parallel Graph Execution:** Refactor the LangGraph orchestrator so `market_agent`, `news_agent`, and `earnings_agent` execute asynchronously in parallel branches.
* **Macroeconomic Tooling:** Add Federal Reserve Economic Data (`FRED`) tools for interest rate and sector benchmark comparisons.
* **Multi Ticker Portfolio Comparison:** Expand state management to compare valuation multiples across peer groups simultaneously.

## Tech Stack

* **Language:** Python 3
* **Agent Orchestration & Memory:** LangGraph (`StateGraph`, `SqliteSaver`, `InMemoryStore`)
* **Local LLM Engine:** Ollama (`llama3.1:8b`)
* **Financial & News Data:** `yfinance` (Yahoo Finance API), `NewsAPI`, `requests`
* **Data Manipulation:** `pandas`, `numpy`, `sqlite3`

## How to Run

1. **Clone the repository**
   `git clone <your-repository-url>`
2. **Install Python dependencies**
   `pip install -r requirements.txt`
3. **Start the local Ollama server and pull the model**
   `ollama serve`
   `ollama pull llama3.1:8b`
4. **Configure API Keys**
   Set your `NEWS_API_KEY` environment variable (or enter it via the interactive `getpass` prompt when executing the notebook).
5. **Run the Notebook**
   Open `Project_Notebook_team_v5.ipynb` in VS Code or Jupyter Lab and execute all cells sequentially.

## Authors

* **Kunal Gurtoo**
* **Kalyan Brata Majumder**
* **Karthik Mekala**

*Curriculum: MS in Applied Artificial Intelligence (AAI 520)*

## Conclusion

This project demonstrates how combining supervisor orchestration, domain specialist tooling, multi stage prompt chaining, quantitative self reflection, and persistent memory creates a transparent and reliable financial research system. By grounding every stage in verifiable data and auditing outputs through an Evaluator Optimizer gate, the system delivers structured equity analysis while preventing unsupported LLM fabrications.