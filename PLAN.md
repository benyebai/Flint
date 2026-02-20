Full End-to-End Plan: Firecracker-Based Semantic PR Review Harness

  1. Product Goal

  1. Build a PR review harness that keeps GitHub’s file-by-file review experience, but adds semantic context and behavior-level change summaries per file.
  2. Make review faster and safer by showing intent, impact, risk, and evidence with exact path:line anchors.
  3. Ensure high trust by enforcing: no semantic claim without supporting evidence.

  2. V1 Scope

  1. Input: GitHub pull requests.
  2. Output: per-file semantic panel for each changed file.
  3. Runtime: ephemeral Firecracker microVMs, read-only repo analysis in V1.
  4. Languages: start with TypeScript/JavaScript.
  5. Review UI: file list + semantic panel + evidence links.
  6. Non-goals in V1: auto-fixing code, auto-commits, full polyglot deep semantics.

  3. Success Criteria

  1. Unsupported-claim rate under 2%.
  2. Evidence coverage over 95% of claims.
  3. P90 run latency under 3 minutes for PRs with 30 changed files or fewer.
  4. Reviewer satisfaction score over 4/5 in beta.
  5. Panel open and render under 500ms P95.

  4. System Architecture

  GitHub Webhook
    -> Control Plane API
    -> Orchestrator + Queue
    -> Firecracker Host Pool
    -> Ephemeral MicroVM Worker
    -> Model Gateway + Retrieval Tools
    -> Postgres + Object Storage
    -> Reviewer UI API
    -> Web UI / PR App Surface

  5. Core Components

  1. GitHub App Integration.
  2. Control Plane API.
  3. Orchestrator and Job Queue.
  4. Firecracker Host Manager.
  5. VM Worker Runtime.
  6. Semantic Context Engine.
  7. Agentic Analysis Pipeline.
  8. Persistence Layer.
  9. UI API.
  10. Reviewer UI.
  11. Observability and Audit Pipeline.

  6. Firecracker Runtime Plan

  1. Use one ephemeral microVM per PR run in V1.
  2. Use read-only root filesystem image.
  3. Mount one writable scratch disk for temp repo and intermediate artifacts.
  4. Boot from snapshot to reduce startup latency.
  5. Use jailer with cgroups, namespaces, seccomp.
  6. Disable inbound networking.
  7. Enforce outbound allowlist only to GitHub API, model gateway, object store.
  8. Inject short-lived credentials at boot through vsock.
  9. Stream logs and traces out via serial/vsock.
  10. Hard-kill VM on timeout, memory breach, or policy violation.
  11. Destroy VM and wipe scratch volume after run completion.

  7. Security Model

  1. V1 token scopes: contents:read, pull_requests:read, issues:read as needed.
  2. No persistent credentials inside VM images.
  3. Credential TTL set to short duration, refreshed only by control plane.
  4. Artifact redaction for secrets and sensitive literals.
  5. Prompt and output policy scan before persistence.
  6. Signed run manifests for tamper-evident auditability.
  7. Full per-run provenance: commit SHAs, model versions, prompt hashes, tool traces.
  8. Tenant-level isolation if multi-org support is added.

  8. PR Ingestion and Orchestration Flow

  1. Receive webhook for PR open/sync/review-requested.
  2. Validate signature and event eligibility.
  3. Resolve repo, PR number, base SHA, head SHA.
  4. Create run record with idempotency key.
  5. Enqueue run job.
  6. Scheduler assigns run to Firecracker host.
  7. Worker VM clones repo shallowly, fetches base and head.
  8. Worker computes file diff and hunk map.
  9. Worker runs semantic context indexing.
  10. Worker runs analysis and verification pipeline.
  11. Worker persists panels, evidence, metrics, logs.
  12. Control plane marks run as completed or failed.
  13. UI/API serves results.

  9. Semantic Context Engine

  1. Parse changed files into AST using tree-sitter.
  2. Build symbol table for changed files and nearby dependencies.
  3. Resolve imports and local dependency graph for first-hop context.
  4. Resolve definitions and references through LSP where available.
  5. Map tests to changed code through import/reference heuristics.
  6. Pull ownership metadata (CODEOWNERS, git blame hotspots).
  7. Retrieve surrounding code windows with token-aware chunking.
  8. Rank candidate context snippets with weighted scoring.
  9. Store evidence candidates with source coordinates and confidence.

  10. Retrieval Ranking Strategy

  1. Rank by symbol overlap between changed hunks and candidate snippets.
  2. Rank by call graph distance where available.
  3. Rank by import graph proximity.
  4. Rank by test linkage relevance.
  5. Rank by historical co-change signal.
  6. Penalize stale or duplicate snippets.
  7. Cap final per-file evidence pack by strict token budget.

  11. Agentic Pipeline Design

  1. Explorer stage gathers evidence only and returns cited snippets.
  2. Analyzer stage writes semantic interpretation in strict JSON schema.
  3. Verifier stage validates every claim has citation support.
  4. Repair loop reruns analyzer with verifier feedback up to 2 attempts.
  5. If still weak, output uncertain sections instead of fabricated claims.
  6. Final panel is accepted only after schema validation and citation checks.

  12. Output Contract (Per File Panel)

  {
    "file_path": "src/foo.ts",
    "intent": "What this change is trying to do",
    "semantic_delta": "Behavior before vs behavior after",
    "impacted_surfaces": [
      "public_api",
      "internal_callers",
      "data_contracts",
      "tests"
    ],
    "risk_assessment": [
      {
        "type": "regression",
        "level": "medium",
        "reason": "explanation",
        "evidence_ids": ["ev_12", "ev_18"]
      }
    ],
    "review_checklist": [
      "specific thing reviewer should verify"
    ],
    "confidence": 0.0,
    "uncertainties": [
      "what could not be confirmed"
    ],
    "evidence": [
      {
        "id": "ev_12",
        "path": "src/foo.ts",
        "line_start": 42,
        "line_end": 57,
        "snippet": "short excerpt",
        "why_relevant": "supports claim X"
      }
    ]
  }

  13. Data Model (Postgres)

  1. repositories(id, github_owner, github_repo, install_id, created_at).
  2. pull_requests(id, repository_id, pr_number, base_sha, head_sha, title, state, updated_at).
  3. runs(id, repository_id, pull_request_id, status, trigger, started_at, finished_at, error_code, error_text).
  4. run_files(id, run_id, file_path, change_type, additions, deletions, status).
  5. evidence(id, run_file_id, path, line_start, line_end, snippet, source_type, score, created_at).
  6. panels(id, run_file_id, schema_version, json_payload, confidence, verified, created_at).
  7. model_calls(id, run_id, stage, model, prompt_tokens, completion_tokens, latency_ms, cost_usd, prompt_hash).
  8. tool_calls(id, run_id, stage, tool_name, input_hash, latency_ms, outcome, error_text).
  9. artifacts(id, run_id, kind, object_uri, checksum, size_bytes, created_at).
  10. metrics(id, run_id, key, value, created_at).

  14. API Surface

  1. POST /webhooks/github.
  2. GET /runs/:run_id.
  3. GET /runs/:run_id/files.
  4. GET /runs/:run_id/files/:file_id/panel.
  5. GET /runs/:run_id/files/:file_id/evidence.
  6. POST /runs/:run_id/rerun.
  7. POST /runs/:run_id/cancel.
  8. GET /repos/:repo_id/prs/:pr_number/latest-run.
  9. GET /health.
  10. GET /metrics.

  15. Reviewer UI Plan

  1. Left pane mirrors GitHub changed files list.
  2. Main pane shows semantic panel for selected file.
  3. “Intent” section first.
  4. “Behavior delta” section second.
  5. “Impact map” section third.
  6. “Risk assessment” section with severity badges.
  7. “Reviewer checklist” section with actionable checks.
  8. “Evidence” section with clickable path:line anchors.
  9. Confidence and uncertainty banner always visible.
  10. Fast keyboard navigation between files and evidence anchors.

  16. Queueing and Reliability

  1. Use durable queue with visibility timeout.
  2. Enforce idempotency per (repo, pr, head_sha, pipeline_version).
  3. Retry transient failures with exponential backoff.
  4. Separate terminal failures from retryable failures.
  5. Persist intermediate stage checkpoints.
  6. Support cancellation at run and file granularity.
  7. Add dead-letter queue with replay tooling.

  17. Cost and Performance Controls

  1. Cache semantic index by (repo, commit_sha, language_pack_version).
  2. Cache retrieval neighborhoods per file and commit.
  3. Use small model for exploration and larger model for analysis only.
  4. Cap per-file token budget and evidence count.
  5. Skip generated/minified/binary files.
  6. Batch low-risk files when possible.
  7. Apply strict timeout per stage and per run.

  18. Observability and Audit

  1. Centralized logs with run_id, file_id, stage.
  2. Traces across orchestrator, VM lifecycle, model calls, tool calls.
  3. Metrics for latency, token use, cost, error classes, citation coverage.
  4. Dashboard by repo, team, and model version.
  5. Audit records for every persisted panel and evidence anchor.
  6. Diff view for panel output changes across reruns.

  19. Evaluation Plan

  1. Build gold dataset of 50 real PRs with human-labeled semantic summaries and risks.
  2. Offline eval metrics: claim support precision, citation recall, usefulness score.
  3. Online eval metrics: reviewer accept rate, time-to-approve delta, comment reduction.
  4. Red-team eval for hallucination and prompt injection.
  5. Regression suite runs on every prompt or pipeline change.

  20. Rollout Plan

  1. Phase 0: internal dry-run on archived PRs.
  2. Phase 1: shadow mode on selected repos, no user-visible output.
  3. Phase 2: opt-in beta with visible panels.
  4. Phase 3: default-on with feature flag fallback.
  5. Phase 4: add extra language packs and advanced graph features.

  21. Implementation Timeline (12 Weeks)

  1. Week 1: requirements freeze, schema freeze, API contracts.
  2. Week 2: GitHub app integration and run orchestration baseline.
  3. Week 3: Firecracker host setup, image build, snapshot pipeline.
  4. Week 4: VM worker with repo checkout and diff/hunk extraction.
  5. Week 5: semantic index MVP for TS/JS.
  6. Week 6: retrieval ranking and evidence pack generation.
  7. Week 7: explorer/analyzer/verifier pipeline with schema enforcement.
  8. Week 8: panel persistence, API endpoints, evidence anchors.
  9. Week 9: reviewer UI implementation.
  10. Week 10: observability, audit, cost controls.
  11. Week 11: evaluation harness and red-team hardening.
  12. Week 12: beta rollout and feedback loop.

  22. Team and Ownership

  1. Infra owner: Firecracker host pool, networking, isolation, runtime policies.
  2. Backend owner: orchestration, APIs, data model, retries, idempotency.
  3. ML owner: prompt contracts, stage logic, verifier quality.
  4. Frontend owner: panel UX and evidence navigation.
  5. Security owner: token policy, sandbox hardening, audit controls.
  6. Product owner: success metrics, rollout policy, user feedback loop.

  23. Risk Register and Mitigations

  1. Hallucinated semantics.
  2. Mitigation: verifier gate + mandatory citations + uncertainty fallback.
  3. Very large PRs causing latency spikes.
  4. Mitigation: sharding by file priority + timeboxing + partial results.
  5. Polyglot repos reducing accuracy.
  6. Mitigation: language-specific plugins and “unsupported language” fallback mode.
  7. Secret leakage in logs/artifacts.
  8. Mitigation: redaction pipeline and strict artifact policy.
  9. Firecracker host saturation.
  10. Mitigation: autoscaling host pool and queue backpressure.
  11. External API outages.
  12. Mitigation: retries, cached metadata, graceful degradation.

  24. Definition of Done (V1 GA)

  1. Meets all success thresholds in Section 3 for 30 consecutive days.
  2. No critical security findings in runtime and credential handling.
  3. Reviewer panel available for 99.9% of eligible PR runs.
  4. Full audit trace available for every run.
  5. Runbook and on-call alerts validated by game-day drills.

  25. Immediate Next Steps

  1. Lock V1 schema and API contracts.
  2. Finalize Firecracker image and sandbox policy spec.
  3. Build a 10-PR pilot dataset and baseline metrics.
  4. Start Week 1 implementation with explicit acceptance tests per milestone.

  If you want, I can now convert this into a concrete build spec with:

  1. exact service repo structure,
  2. migration SQL,
  3. API OpenAPI spec,
  4. Firecracker host config templates,
  5. prompt files for explorer/analyzer/verifier.
