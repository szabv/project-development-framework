# Designing an Agentic Research Assistant for High-Quality Technical Research

Date: 2025-11-22  
Owner: Research / Agentic Systems  
Status: Draft

This report summarizes design options and recommendations for building an LLM-based research agent (or small multi-agent system) that can perform high-quality, outcome-focused technical research, aligned with the methodology described in `reasearch/research-methodology.md`.

---

## How the Research Was Conducted

- **Anchor on current state**
  - Started from the internal research methodology in `reasearch/research-methodology.md`, which defines how human reports are structured (current state, goals, recommendations, evaluation, cost/operations).
  - Treated this as the target output pattern for an automated research agent.

- **Clarify goals and constraints**
  - Goal: design a research agent that can produce high-quality, evidence-backed, outcome-focused reports with minimal hallucination and maximum traceability.
  - Constraints: works with web and domain-specific sources; needs transparent reasoning, citations, and explicit uncertainty; should be adaptable to different research topics.

- **Survey domain practices and options**
  - Searched for:
    - LLM-based research agents and multi-agent systems.
    - Systematic literature review methodologies in software engineering.
    - LLM-assisted web research practices.
    - Fact-checking and RAG factuality pipelines.
  - Considered scientific and practitioner sources (papers, surveys, frameworks, blogs, GitHub repos).

- **Map general guidance to this system**
  - Mapped systematic-review methodology (Kitchenham-style) to an agentic workflow: clarify → plan → search → screen → extract → synthesize → evaluate.
  - Mapped multi-agent LLM patterns (e.g., Agent Laboratory, AgentReview) to roles: planner, searcher, reader/extractor, synthesizer, critic/fact-checker, optional experimenter.
  - Integrated fact-checking and RAG evaluation patterns (e.g., OpenFactCheck-style) into a dedicated critique phase.

- **Plan evaluation and observability**
  - Proposed logging for queries, sources, and decisions so runs are auditable.
  - Suggested a human and/or multi-agent peer-review loop (AgentReview-style) to assess factuality, coverage, and usefulness over time.

---

## Executive Summary

- An effective research agent should be built as a **methodology-driven, multi-agent system**, not as a single prompt that “does research”.
- The agent’s workflow should follow a **lightweight systematic-review pattern**: clarify questions → plan → systematically search → screen sources → extract structured evidence → synthesize → fact-check → finalize.
- **Role specialization** improves quality and robustness: planner, search agent, reader/extractor, synthesizer, critic/fact-checker, and (optionally) experimenter, coordinated by a supervisor.
- The system must be **evidence-first and citation-centric**: every major claim links to one or more sources and includes a confidence level and evidence strength tag.
- **Systematic search and coverage** are critical for quality: the agent should use multiple queries, multiple source types (papers, standards, official docs, high-quality blogs), and explicit inclusion/exclusion rules.
- A dedicated **critique and fact-checking phase** should challenge the draft report, search for counter-evidence, and correct unsupported or weakly supported statements.
- The entire process must be **outcome-focused**: the planner and synthesizer work backwards from the decision the user is trying to make, and the report converges on prioritized, actionable recommendations.
- **Transparency and reproducibility** require detailed logging of search queries, selected sources, and reasoning checkpoints; long-term quality requires periodic human or agentic peer review of outputs.

---

## Current State / Ground Truth

### Internal methodology

- The existing methodology in `reasearch/research-methodology.md` defines:
  - A structured way to capture current state (with repo/file references).
  - Explicit goals and constraints before exploring options.
  - Recommendations grouped by area (Architecture, Data, Models, Tooling, Evaluation, Observability, Cost/Operations).
  - A consistent output pattern: title/context → executive summary → current state → goals → recommendations → evaluation → cost/operations.
- This is a good target shape for reports produced by a research agent. The automation problem becomes:
  - How to make an agent reliably fill these sections with **correct, well-grounded content**.

### State of research agents and multi-agent systems

From recent surveys and frameworks:

- **Agent Laboratory: Using LLMs as Research Assistants**
  - Demonstrates multi-agent LLM systems that can carry out research tasks end-to-end (question → search → analysis → writeup) using specialized agents.
  - Emphasizes tool use (web search, code, data) and orchestration logic as key to performance.

- **AgentReview: Exploring Academic Peer Review with LLM Agents**
  - Uses multiple LLM agents to simulate peer review.
  - Shows that structured roles (reviewer, meta-reviewer) and explicit rubrics improve evaluation quality.

- **LLM-based Multi-Agent Systems for Software Engineering (survey + roadmap)**
  - Reviews many LLM agent architectures and notes:
    - Specialization into roles (planner, executor, critic) tends to outperform single-agent setups.
    - Tool integration and clear task decomposition are major drivers of success.
    - Evaluation remains challenging but critical.

Overall, current systems suggest that:

- Single “monolithic” research prompts tend to hallucinate, miss relevant sources, and lack traceability.
- Multi-agent, tool-using systems with clear roles, structured workflows, and evaluation loops achieve better reliability and depth.

### Systematic literature reviews and evidence quality

Key points from software engineering systematic-review literature (Kitchenham and follow-ups):

- A **systematic literature review (SLR)** aims to:
  - Answer a well-defined research question.
  - Search the literature in a reproducible, comprehensive way.
  - Apply explicit inclusion/exclusion criteria.
  - Extract and synthesize evidence to form defensible conclusions.
- Common steps:
  1. Define research questions.
  2. Define search strategy (databases, keywords, date ranges).
  3. Execute search and collect candidate papers.
  4. Screen based on title/abstract/full text (with criteria).
  5. Extract data using a standard form.
  6. Synthesize results qualitatively (and quantitatively when possible).
- Lessons:
  - Ad-hoc search is prone to bias and omission.
  - Structured extraction and synthesis reduce misinterpretation.
  - Transparent protocols and data extraction forms make reviews more trustworthy.

These insights motivate a **structured, SLR-inspired workflow** for the research agent.

### LLM-assisted web research practices

Guidance from practitioner articles (e.g., LessWrong’s “A Guide for LLM-Assisted Web Research”) and tools (e.g., `langchain-ai/local-deep-researcher`) includes:

- LLMs are strong at:
  - Generating search queries and synonyms.
  - Summarizing and comparing documents.
  - Highlighting patterns and disagreements.
- LLMs are weak at:
  - Knowing what is **not** in the data they saw.
  - Reliable recall of facts without grounding.
  - Distinguishing authoritative from low-quality sources.
- Best practices:
  - Use LLMs as **controllers and summarizers**, not as oracles.
  - Integrate tool calls for search, retrieval, and document loading.
  - Keep explicit notes and citations; avoid relying on internal model “memory”.

### Factuality and fact-checking

From factuality and fact-checking work:

- Fact-checking pipelines typically:
  - Identify claims (e.g., via a claim extractor).
  - Retrieve evidence passages from trusted corpora.
  - Evaluate support/contradiction using models or structured rules.
  - Produce a verdict per claim, often with rationale.
- OpenFactCheck-style frameworks and RAG evaluation work show:
  - LLMs can help evaluate factuality when grounded in retrieved evidence.
  - Multi-step pipelines (claim extraction → retrieval → verification) outperform single-pass generation.
  - Factuality scores can be attached to outputs for monitoring.

These patterns motivate a dedicated **critic/fact-checking agent** and a structured verification phase for the research agent.

---

## Goals and Constraints for the Research Agent

- **Primary goals**
  - Produce **high-quality, evidence-backed research reports** that follow the internal methodology.
  - Maintain an **outcome-focused orientation**: reports converge on prioritized, actionable recommendations.
  - Minimize hallucinations via **tool use, systematic search, and explicit fact-checking**.
  - Ensure **traceability and transparency**: clear mapping from claims to sources and reasoning.

- **Secondary goals**
  - Support both **broad exploratory research** and **deep dives**.
  - Allow **human steering** at key checkpoints (e.g., clarifying research questions, adjusting inclusion criteria, prioritizing recommendations).
  - Be modular enough to integrate new tools (search APIs, corpora, models) over time.

- **Constraints**
  - Must work under **token and latency budgets**; long exploratory loops should be controllable.
  - Must handle varying **source quality** on the web and distinguish higher-quality evidence where possible.
  - Must be **reproducible**: logs and intermediate artifacts are needed for audit and re-use.

---

## Design Criteria for a Research Agent

### 1. Methodology-driven, not prompt-only

- The agent should implement an explicit **phase structure**, aligned with `reasearch/research-methodology.md` and SLR practice:
  1. Clarify brief and goals.
  2. Plan research questions and strategy.
  3. Systematic search and source collection.
  4. Screening and selection.
  5. Reading and structured extraction.
  6. Synthesis and drafting.
  7. Critique and fact-checking.
  8. Finalization and logging.
- Each phase has defined **inputs, outputs, and success checks**, so it can be traced and improved independently.

### 2. Role specialization (multi-agent design)

Recommended roles:

- **Supervisor / Orchestrator**
  - Owns the overall workflow.
  - Ensures phases run in order and that required artifacts exist before moving on.

- **Planner Agent**
  - Translates the user brief into concrete research questions and sub-questions.
  - Defines information needs, source types, and evidence requirements.

- **Search Agent**
  - Designs and executes search strategies across web and domain-specific corpora.
  - Performs query expansion and multi-pass search, collecting a candidate pool of sources.

- **Reader / Extractor Agent**
  - Reads selected sources and extracts structured notes (claims, methods, metrics, caveats).
  - Normalizes terminology and attaches metadata (evidence strength, date, domain).

- **Synthesizer Agent**
  - Aggregates notes, identifies consensus and disagreements, and maps findings to the user’s context.
  - Drafts a report in the internal pattern, with citations.

- **Critic / Fact-Checker Agent**
  - Challenges the draft, verifies high-impact claims, searches for counter-evidence, and flags weaknesses.
  - Applies a structured checklist for factuality, coherence, and coverage.

- **(Optional) Experimenter Agent**
  - When the research involves algorithms/configs you control, designs small experiments and feeds results back into the synthesis.

This design follows patterns observed in Agent Laboratory and multi-agent surveys, which consistently find specialized agents with clear interfaces more reliable than monolithic prompts.

### 3. Evidence-first and citation-centric

- Every significant claim in the report should:
  - Be backed by one or more **explicit sources** (URL or document identifier).
  - Include an **evidence type tag**:
    - Peer-reviewed paper.
    - Standards/specification.
    - Official documentation.
    - Practitioner blog/post.
    - Internal report or code/docs.
  - Optionally include a **confidence level** (e.g., high/medium/low) reflecting both evidence quality and model certainty.
- The system should encourage the agent to **avoid uncited claims**, or clearly mark them as speculative.

### 4. Systematic search and coverage

- The search phase should:
  - Use multiple query formulations and synonyms for each sub-question.
  - Target multiple source types and domains (e.g., arXiv, ACM, publisher sites, official docs, high-signal blogs).
  - Apply simple but explicit inclusion/exclusion criteria:
    - Date range.
    - Domain relevance.
    - Minimal quality/authority heuristics (e.g., recognize obvious SEO spam).
  - Track:
    - Which queries were used.
    - Which sources were retrieved and why they were included or excluded.
- This makes the search process **reproducible and improvable**.

### 5. Quality and factuality safeguards

- Introduce a dedicated **fact-checking phase** inspired by fact-checking pipelines:
  - Extract high-impact claims from the draft.
  - For each claim, retrieve additional evidence and test whether the evidence supports, contradicts, or is neutral.
  - If evidence is conflicting or weak, downgrade confidence and/or add caveats.
- The critic agent should:
  - Check for internal contradictions within the report.
  - Ensure that metrics and numbers are accurately transferred from sources.
  - Ensure that caveats and limitations are not omitted.
  - Flag sections that rely heavily on low-quality or single sources.

### 6. Outcome-focused structuring

- From the planning phase, the agent should orient around:
  - The **primary decision** the research is meant to support.
  - The **success metrics** that matter (e.g., accuracy, latency, maintainability, cost).
- The synthesizer and critic should evaluate content with questions like:
  - “Does this help decide between options A/B/C?”
  - “Are trade-offs between alternatives clearly laid out?”
  - “Are evaluation and rollout implications spelled out?”

### 7. Transparency, logging, and reproducibility

- Log:
  - All search queries and tool calls with timestamps.
  - Candidate and selected sources with metadata and reasons.
  - Structured extraction notes.
  - Intermediate drafts and critiques.
- Store logs in a way that:
  - Allows **re-running** a research task with the same or modified parameters.
  - Supports offline analysis of **agent performance** over time.

### 8. Human-in-the-loop control

- Provide **checkpoints** where a human can:
  - Adjust research questions and scope.
  - Change inclusion/exclusion criteria.
  - Add or remove high-priority sources.
  - Re-order or filter recommendations.
- This allows the system to combine the strengths of automation with human judgment in ambiguous or high-stakes situations.

### 9. Modular tooling

- Implement adapters for:
  - Web search APIs.
  - Vertical search (arXiv, PubMed, ACM, GitHub, internal repositories).
  - A vector store for internal code, docs, and previous research outputs.
  - PDF/HTML/text loaders and parsers.
- Agents should call tools via a **stable interface** so that underlying providers can be swapped without redoing prompts and logic.

---

## Proposed Research Agent Architecture

Conceptual architecture:

- **Supervisor / Orchestrator**
  - Manages the high-level workflow.
  - Accepts user brief and returns final report plus logs.

- **Planner Agent**
  - Produces a structured research plan and sub-questions.
  - Suggests source types, search strategies, and evidence requirements.

- **Search Agent**
  - Executes the plan across multiple search providers.
  - Outputs ranked candidate sources with metadata and notes.

- **Reader / Extractor Agent**
  - Reads selected sources and produces structured notes.

- **Synthesizer Agent**
  - Uses notes and goals to draft the report in the internal pattern.

- **Critic / Fact-Checker Agent**
  - Evaluates factuality, coverage, and coherence.
  - Triggers additional targeted search when needed.

- **(Optional) Experimenter Agent**
  - Runs small computations or experiments when relevant and safe.

- **Memory / Knowledge Store**
  - Stores prior research outputs and notes.
  - Provides retrieval for planner and search agent to reuse prior work.

---

## Proposed End-to-End Workflow

Below is the recommended agentic workflow, phase by phase.

### Phase 1: Clarify brief and goals

- Input: User prompt / brief.
- Supervisor + Planner:
  - Ask clarifying questions if scope or success criteria are unclear.
  - Produce a **research charter** stating:
    - Primary decision or outcome.
    - Key constraints (quality, latency, cost, domain).
    - Non-goals or out-of-scope aspects.

### Phase 2: Plan research questions and strategy

- Planner:
  - Breaks the charter into sub-questions aligned with report sections (e.g., architecture, evaluation, operations).
  - For each sub-question, defines:
    - Information needs.
    - Target source types and search strategies.
    - Minimal evidence requirements.
- Output: Research plan object (sub-questions, strategies, requirements).

### Phase 3: Systematic search

- Search Agent:
  - For each sub-question:
    - Generates multiple queries (keywords, natural language, synonyms).
    - Runs them through web and vertical search tools.
  - Collects a candidate pool of sources, de-duplicates, and scores them on relevance, recency, and authority.
- Output: Candidate source lists per sub-question.

### Phase 4: Screening and selection

- Supervisor + optional human:
  - Review candidate lists, adjust filters (e.g., year range, emphasis on practitioner content).
- Search Agent:
  - Applies updated criteria and selects a final set of sources to read.
  - Tags each with evidence type and priority.
- Output: Selected source sets per sub-question.

### Phase 5: Reading and structured extraction

- Reader / Extractor:
  - Loads selected sources (HTML, PDF, docs).
  - Fills a standard extraction schema for each:
    - Problem addressed.
    - Methods/approach.
    - Key findings and metrics.
    - Recommendations or patterns.
    - Limitations and caveats.
    - Context (domain, scale, assumptions).
  - Stores notes in a shared data structure accessible to other agents.

### Phase 6: Synthesis and drafting

- Synthesizer:
  - Clusters notes by sub-question and topic.
  - Identifies consensus, disagreements, and gaps.
  - Maps findings to the user’s goals and constraints.
  - Drafts a report that follows the internal pattern:
    - Title and context.
    - Executive summary.
    - Current state / ground truth (if applicable).
    - Goals and constraints.
    - Recommendations by area.
    - Evaluation and change planning.
    - Cost and operational guidance.
  - Attaches citations to all substantial claims.

### Phase 7: Critique and fact-checking

- Critic / Fact-Checker:
  - Reads the draft and extraction notes.
  - Identifies high-impact claims (recommendations, metrics, strong statements).
  - For each:
    - Runs targeted searches or retrieval to confirm or refute.
    - Checks that the cited sources actually support the claim.
    - Looks for missing opposing evidence.
  - Ensures:
    - No obvious contradictions within the report.
    - Caveats are correctly represented.
    - Claims are appropriately labelled (evidence-backed vs speculative).
  - Produces a set of suggested edits and risk flags, plus section-level robustness scores.
- Synthesizer:
  - Incorporates required changes and notes uncertainties in the final text.

### Phase 8: Finalization and logging

- Supervisor:
  - Ensures that:
    - The final report clearly answers the original decision question.
    - Recommendations are prioritized and trade-offs are explicit.
    - Evaluation and rollout plans are included where relevant.
  - Packages:
    - Final report (in your standard format).
    - Logs of queries, sources, extraction notes, and critiques.
- Optional human review:
  - Adjusts prioritization based on organization-specific constraints.

---

## Evaluation and Future Improvements

### Evaluation strategy

- **Human expert review**
  - Periodically have humans rate reports on:
    - Factual correctness.
    - Coverage and depth.
    - Practical usefulness of recommendations.
    - Clarity and structure.

- **Agent-based peer review**
  - Use an AgentReview-style setup where multiple independent critic agents review a report, then a meta-critic aggregates their findings.

- **Factuality metrics**
  - Use an OpenFactCheck-like pipeline to:
    - Extract claims from reports.
    - Retrieve evidence from trusted corpora.
    - Score claims on supported/contradicted/uncertain.

- **Task-based evaluation**
  - For certain topics with known “best-practice” answers, create benchmarks.
  - Compare the agent’s recommendations against target answers and measure alignment.

### Future improvements

- Integrate domain-specific corpora (internal documentation, code, prior research) more deeply into search and retrieval.
- Add more sophisticated **debate-style** mechanisms between agents for contentious questions.
- Incorporate lightweight **experimentation capabilities** (e.g., running small simulations or benchmarks) where relevant.
- Learn from usage logs which query strategies, tools, and prompts correlate with high-quality reports, and adapt automatically.

---

## Summary

The research suggests that a high-quality, outcome-focused research agent should be:

- **Methodology-driven**, following a structured research and reporting process.
- **Multi-agent and tool-using**, with specialized roles for planning, search, reading, synthesis, and critique.
- **Evidence-first and transparent**, with strong citation practices and explicit uncertainty.
- **Outcome-focused**, always working backwards from the decision the user needs to make.
- **Evaluated and improved over time**, via human review, agentic peer review, and factuality metrics.

This document can serve as the high-level design reference for implementing a concrete research-agent orchestration in this repository.

