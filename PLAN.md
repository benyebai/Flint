Plan Draft

  1. Define the v1 contract (2 days).

  - Output target: per-file semantic panel with intent, semantic_delta, impacted_surfaces, risk_assessment, review_checklist, evidence.
  - Hard rule: every claim must include path:line evidence or be marked uncertain.
  - Non-goals for v1: auto-fixing code, full-repo deep graph reasoning, multi-language support beyond first language.

  2. Build PR ingestion + run orchestration (3 days).

  - Trigger on PR events, fetch base/head, compute unified diff hunks.
  - Create a run record and per-file work items.
  - Store artifacts so runs are reproducible.

  3. Build semantic context engine (1 week).

  - Parse changed files and build symbol index for head.
  - Add lightweight relationship graph: imports, definitions/references, callers/callees, touched tests.
  - For each hunk, retrieve top-N context snippets with ranking and token budget.
  - Persist retrieved snippets as evidence candidates.

  4. Build multi-agent analysis pipeline (1 week).

  - Explorer agent: gathers relevant snippets only, no conclusions.
  - Analyzer agent: writes structured semantic panel JSON for each file.
  - Verifier agent: checks citation coverage and rejects unsupported claims.
  - Add deterministic validation: schema checks, citation existence checks, confidence scoring.

  5. Build reviewer UI with file panels (1 week).

  - Left pane: changed files like GitHub PR.
  - Right pane: semantic panel for selected file.
  - Clickable evidence anchors jump to exact lines/diff hunk.
  - Show confidence and “unknown/needs human check” states clearly.

  6. Add quality gates + evaluation (1 week).

  - Golden dataset of 30–50 PRs across difficulty levels.
  - Metrics: citation coverage, unsupported-claim rate, reviewer usefulness score, latency per file.
  - Release gate: block rollout if unsupported-claim rate exceeds threshold.

  7. Rollout strategy (3 stages).

  - Stage 1: shadow mode (internal only, no user-facing output).
  - Stage 2: opt-in beta on selected repos.
  - Stage 3: default-on with feature flag fallback.

  Technical baseline

  1. Backend: worker queue + API + DB (runs, file_panels, evidence, metrics).
  2. Model flow: tool-driven iterative retrieval first, synthesis second.
  3. Caching: cache parsed symbols and retrieved context by repo@commit.
  4. Failure policy: degrade to “diff-only mode” when context retrieval fails.

  Review Decisions Needed

  1. V1 language scope: TS/JS only or TS/JS + Python.
  2. Latency target: prioritize speed (quick panels) or depth (more evidence).
  3. Confidence policy: hide low-confidence claims or show with warning.
  4. Hosting model: GitHub App in your infra vs customer-hosted runner.
