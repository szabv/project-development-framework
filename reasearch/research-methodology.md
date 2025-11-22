# Research Methodology and Output Pattern

This document describes a general approach for conducting and writing up technical research so that reports are consistent, deep, and easy to act on, regardless of the specific subject.

## How the Research Was Conducted

- **Anchor on current state**
  - Start by reading the relevant code, configuration, and architecture diagrams to build a “ground truth” section.
  - Identify concrete components (APIs, services, workers, databases, queues, UIs) and reference them with file paths and line numbers where possible.
  - Note bottlenecks, constraints, and opportunities directly from the implementation and runtime behavior.

- **Clarify goals and constraints**
  - Capture explicit targets and desired outcomes (for example, quality or accuracy metrics, latency goals, usability improvements, cost ceilings, scale assumptions).
  - If the user request does not clearly state the point of the research or the goals, pause and ask for clarification before going deep.
  - Record any relevant platform, technology, or process preferences and constraints.

- **Survey domain practices and options**
  - For each major area of the topic (e.g., architecture, data, models, UX, operations), gather current best practices and commonly recommended patterns.
  - Compare alternatives with an emphasis on fit for the stated constraints rather than generic “best” rankings.

- **Map general guidance to the specific system**
  - Translate general recommendations into concrete changes for the system under study (which functions to update, what data or metadata to add, which configuration or infra elements to adjust).
  - Use measurable effects where possible (for example, expected changes in latency, error rates, accuracy, or cost informed by benchmarks or prior experience).
  - Prioritize by ROI: low-effort, high-impact changes first; defer complex or risky changes unless clearly justified.

- **Plan evaluation and observability**
  - Design a lightweight evaluation harness tied to the main workflows or critical paths.
  - Define metrics that reflect the goals (for example, latency breakdown, throughput, accuracy, reliability, usability) and where results should be written in the repo.
  - Specify logging, metrics, and tracing improvements needed to verify that changes actually meet the goals.

## Output Format Pattern

- **Title and context**
  - Start with a descriptive title and date.
  - Briefly state what the report reviews and the main performance/cost targets.

- **Executive summary**
  - 4–7 bullets capturing the most important recommendations, trade-offs, and expected impact.
  - Emphasize actionable direction (what to do next) rather than restating the current state.

- **Current implementation / ground truth**
  - Describe the existing pipeline or system end-to-end.
  - Include code and schema references using `path/to/file.ext:line` so readers can jump into the repo.
  - Note observed constraints and risks surfaced from the code and infra.

- **Goals and constraints**
  - Call out quality, latency, scale, and budget targets.
  - Include any model/vendor requirements and compliance or operational constraints.

- **Recommendations by area**
  - Group recommendations under clear area headings appropriate to the topic (for example, Architecture, Data & Storage, Algorithms & Models, Tooling & DX, Testing & Evaluation, Observability, Cost & Operations).
  - Within each recommendation, follow a consistent mini-structure:
    - **What**: the concrete change to apply.
    - **Why**: the rationale and how it helps given the goals/constraints.
    - **Impact**: expected directional impact and, when possible, rough quantitative ranges.
    - **How**: specific implementation notes, including code paths, configuration or infra changes, and any rollout or change-management steps.

- **Evaluation and change planning**
  - Dedicate sections to evaluation strategy and change or rollout plans when significant implementation or infrastructure changes are proposed.
  - Spell out dataset size (if applicable), metrics, where reports will live in the repo, and how to keep runs within budget or resource limits.

- **Cost and operational guidance**
  - Summarize cost and operational implications for major options and tie them back to the stated constraints.
  - Highlight where simpler or more operationally robust options are preferable at current scale, even if they are not the most “advanced” choice.

- **Style and cross-linking**
  - Use concise bullets and short paragraphs; avoid long, unstructured prose.
  - Cross-link to repo files and scripts for every important recommendation so the report doubles as a navigation guide.
  - Keep headings stable and ordered from current state → goals → recommendations → evaluation → cost, so readers can skim quickly.

## Quick Research Checklist

- Capture current state with concrete code/config references.
- Write down explicit goals, targeted outcomes, and constraints before surveying options.
- If goals or success criteria are unclear or contradictory, ask the user to clarify.
- Collect domain practices and alternatives for each major area.
- Map options to this system with clear What/Why/Impact/How.
- Define evaluation metrics, datasets (if any), and where results are stored.
- Call out cost and operational implications of major choices.
- Structure the final report: title/context → executive summary → current state → goals → recommendations by area → evaluation → cost/operations.
